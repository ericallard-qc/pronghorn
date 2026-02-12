

# Fix: Pronghorn Runner Sync Loop Prevention

## Root Cause

The bidirectional sync loop occurs because:

1. **Cloud writes trigger the local watcher**: When `syncStagingToLocal()` writes a file to disk, chokidar sees it as a local change and calls `pushLocalChangeToCloud()`
2. **Push triggers a broadcast**: The `staging-operations` edge function always broadcasts `staging_refresh` after any stage operation
3. **Broadcast triggers another sync**: The runner receives the broadcast and calls `syncStagingToLocal()` again
4. **No origin tracking**: The runner has no way to distinguish "I wrote this file" from "the user edited this file"

This is especially bad during rapid Claude Code editing because dozens of files change within seconds, creating cascading loops.

## Proposed Solution: Three-Layer Defense

### Layer 1: Write Origin Tracking (Primary Fix)

Track files that the runner itself writes to disk. When chokidar fires for those files, skip the push.

```text
// New state tracking
let cloudWrittenFiles = new Map();  // path → { hash, timestamp }
const CLOUD_WRITE_COOLDOWN_MS = 2000;

// In syncStagingToLocal(), BEFORE writing:
cloudWrittenFiles.set(relativePath, { 
  hash: hashContent(newContent), 
  timestamp: Date.now() 
});

// In chokidar handlers (add/change), BEFORE pushing:
const tracked = cloudWrittenFiles.get(relativePath);
if (tracked) {
  const localHash = hashContent(content);
  const elapsed = Date.now() - tracked.timestamp;
  if (localHash === tracked.hash || elapsed < CLOUD_WRITE_COOLDOWN_MS) {
    // This write came from us, skip push
    cloudWrittenFiles.delete(relativePath);
    return;
  }
}
```

This is the single most impactful change. It prevents the echo where cloud-written files bounce back to the cloud.

### Layer 2: Increase Debounce for Rapid Editing

Change the staging sync debounce from 150ms to 1500ms (1.5 seconds) during burst activity. This collapses rapid Claude Code edits into a single sync pass.

```text
const STAGING_DEBOUNCE_MS = 150;     // Normal single-file edits
const STAGING_BURST_DEBOUNCE_MS = 1500; // When multiple events arrive rapidly

let stagingEventCount = 0;
let stagingBurstTimer = null;

function scheduleStagingSync() {
  stagingEventCount++;
  clearTimeout(stagingSyncTimer);
  
  // If we've received 3+ events in quick succession, use longer debounce
  const debounceMs = stagingEventCount > 3 
    ? STAGING_BURST_DEBOUNCE_MS 
    : STAGING_DEBOUNCE_MS;
  
  stagingSyncTimer = setTimeout(async () => {
    stagingEventCount = 0;
    await syncStagingToLocal();
  }, debounceMs);
}
```

### Layer 3: Manual Sync Mode (New Feature)

Add a keyboard-driven sync mode where the user controls when syncs happen. This is critical for Claude Code sessions where dozens of files change rapidly.

```text
// New .run config option
SYNC_MODE=auto         // 'auto' (current behavior) or 'manual'
MANUAL_SYNC_PROMPT=true  // Show 'Press Y to sync' prompts

// In manual mode:
// - Cloud → Local: queues changes, shows "5 files pending, press Y to pull"
// - Local → Cloud: queues changes, shows "3 files pending, press P to push"  
// - Press S for full bidirectional sync pass
// - Press A to switch back to auto mode
```

Implementation uses Node.js `readline` to capture keypresses:

```text
const readline = require('readline');

function setupManualSyncMode() {
  readline.emitKeypressEvents(process.stdin);
  if (process.stdin.isTTY) process.stdin.setRawMode(true);
  
  let pendingCloudChanges = [];
  let pendingLocalChanges = [];
  
  process.stdin.on('keypress', async (str, key) => {
    if (key.name === 'y') {
      // Pull cloud → local
      await syncStagingToLocal();
      console.log('[Pronghorn] Manual pull complete');
    }
    if (key.name === 'p') {
      // Push local → cloud (flush queue)
      for (const change of pendingLocalChanges) {
        await pushLocalChangeToCloud(change.path, change.op, change.content);
      }
      pendingLocalChanges = [];
    }
    if (key.name === 's') {
      // Full bidirectional sync
      await syncStagingToLocal();
      for (const change of pendingLocalChanges) {
        await pushLocalChangeToCloud(change.path, change.op, change.content);
      }
      pendingLocalChanges = [];
    }
    if (key.name === 'a') {
      // Toggle auto/manual
      CONFIG.syncMode = CONFIG.syncMode === 'auto' ? 'manual' : 'auto';
      console.log(`[Pronghorn] Sync mode: ${CONFIG.syncMode}`);
    }
    if (key.ctrl && key.name === 'c') {
      process.exit(0);
    }
  });
}
```

---

## Changes Summary

All changes are in a single file: `supabase/functions/generate-local-package/index.ts` (the generated runner script).

| Change | Layer | Impact |
|--------|-------|--------|
| `cloudWrittenFiles` tracking map | 1 - Origin tracking | Prevents echo loops entirely |
| Hash comparison before push | 1 - Origin tracking | Skips identical content pushes |
| Cooldown window (2s) after cloud writes | 1 - Origin tracking | Safety net for hash collisions |
| Burst detection with adaptive debounce | 2 - Debounce | Collapses rapid edits into single sync |
| `SYNC_MODE=manual` option | 3 - Manual mode | Full user control during Claude sessions |
| Keypress handlers (Y/P/S/A) | 3 - Manual mode | Interactive sync control |
| `SYNC_MODE` config in .run file | 3 - Manual mode | Persistent preference |
| Status line showing pending changes | 3 - Manual mode | Visibility into queue state |

## New .run Configuration Options

```text
# SYNC SETTINGS (updated)
REBUILD_ON_STAGING=true
REBUILD_ON_FILES=true
PUSH_LOCAL_CHANGES=true

# NEW: Sync mode - 'auto' (realtime) or 'manual' (press keys to sync)
SYNC_MODE=auto
```

## Expected Behavior After Fix

**Auto mode (default)**:
- Cloud writes a file locally → chokidar fires → hash matches `cloudWrittenFiles` → push skipped (no loop)
- User edits a file locally → chokidar fires → hash does NOT match → push proceeds normally
- Claude Code edits 20 files rapidly → burst debounce collapses to single sync after 1.5s quiet period

**Manual mode (for Claude Code sessions)**:
- Cloud changes arrive → queued, status line shows "5 files pending pull"
- User presses Y → all pending cloud changes written to disk at once
- Local changes detected → queued, status shows "3 files pending push"
- User presses P → all pending local changes pushed to cloud
- User presses S → full bidirectional sync in one pass
- User presses A → toggle back to auto mode

