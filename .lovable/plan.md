

# Truncation Issues in Build Agent and Other Edge Functions

## Findings

There are **multiple active truncation points** across the codebase that cut off user-provided content. Here is a complete inventory:

### 1. **coding-agent-orchestrator** (Build Agent) -- `supabase/functions/coding-agent-orchestrator/index.ts`

| Content Type | Truncation | Lines |
|---|---|---|
| Artifacts | Only first 5 shown, content sliced to 160 chars each | 646-654 |
| Requirements | Only first 10 shown, content sliced to 160 chars each | 658-667 |
| Canvas nodes | Only first 20 shown | 724-734 |
| Canvas edges | Only first 20 shown | 738-743 |
| Sample DB data | Only first 3 rows shown | 777 |

**Not truncated** (full content passed): Standards, Tech Stacks, Repository Files, Database definitions.

### 2. **orchestrate-agents** (Canvas AI Agent) -- `supabase/functions/orchestrate-agents/index.ts`

| Content Type | Truncation | Line |
|---|---|---|
| Repository files | Content sliced to 500 chars each | 855 |

### 3. **decompose-requirements** -- `supabase/functions/decompose-requirements/index.ts`

| Content Type | Truncation | Line |
|---|---|---|
| Artifact content | Sliced to 500 chars each | 510 |
| File content | Sliced to 300 chars each | 537 |

### 4. **Chat edge functions** (gemini, anthropic, xai)

No truncation -- full `attachedContext` JSON is passed through.

## Proposed Fix

Remove all truncation from user-attached context. If the user explicitly attaches content via the ProjectSelector, the full content should be provided to the LLM.

### File 1: `supabase/functions/coding-agent-orchestrator/index.ts`

**Artifacts** (lines 644-655): Remove `.slice(0, 5)` and replace 160-char summary with full content:
```
Artifacts (N attached by user - FULL CONTENT):
### ARTIFACT: {title}
{full content}
```

**Requirements** (lines 657-668): Remove `.slice(0, 10)` and replace 160-char snippet with full content:
```
Requirements (N attached by user - FULL CONTENT):
### REQ: {code} {title}
{full content}
```

**Canvas nodes** (lines 723-735): Remove `.slice(0, 20)`, include all nodes with full data.

**Canvas edges** (lines 737-744): Remove `.slice(0, 20)`, include all edges.

**DB sample data** (line 777): Remove `.slice(0, 3)`, include all sample rows.

### File 2: `supabase/functions/orchestrate-agents/index.ts`

**Repository files** (line 855): Remove the 500-char truncation, include full file content.

### File 3: `supabase/functions/decompose-requirements/index.ts`

**Artifact content** (line 510): Remove 500-char truncation, include full content.

**File content** (line 537): Remove 300-char truncation, include full content.

## Summary

Seven truncation points across three edge functions need to be updated. All three functions will need redeployment.

