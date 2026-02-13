

# Download DDL Definitions from Database Explorer

## Overview

Add two new right-click context menu options:
1. **Single table**: "Download DDL (.sql)" - downloads the CREATE TABLE statement as a `.sql` file
2. **Schema node**: "Download Schema DDL (.sql)" - downloads all table definitions in that schema as a single `.sql` file

## Changes

### 1. DatabaseTreeContextMenu.tsx

Add two new callback props and menu items:

- **Props**: `onDownloadTableDDL` and `onDownloadSchemaDDL`
- **Table menu**: Add a "Download DDL (.sql)" item with the Download icon, after the existing "Get CREATE TABLE" item
- **Schema menu**: Add a "Download Schema DDL (.sql)" item with the Download icon, before the existing "Drop Schema CASCADE" item

### 2. DatabaseSchemaTree.tsx

Thread the two new props through the component tree:

- Add `onDownloadTableDDL` and `onDownloadSchemaDDL` to the `DatabaseSchemaTreeProps` interface
- Pass them into the `contextMenuProps` object

### 3. DatabaseExplorer.tsx

Implement the two handler functions:

**`handleDownloadTableDDL(schema, tableName)`**:
- Calls the existing `manage-database` edge function with action `get_table_definition`
- Takes the returned DDL string and triggers a browser download as `{schema}.{tableName}.sql`
- Shows a toast on success/failure

**`handleDownloadSchemaDDL(schemaName, schemaInfo)`**:
- Iterates over all tables in `schemaInfo.tables`
- For each table, calls the `manage-database` edge function with `get_table_definition`
- Concatenates all DDL results separated by `\n\n` with comment headers (`-- Table: tablename`)
- Triggers a browser download as `{schemaName}_schema.sql`
- Shows progress toast ("Fetching 1/N...") during the process

Both handlers use a simple utility to trigger the download:

```typescript
function downloadSqlFile(filename: string, content: string) {
  const blob = new Blob([content], { type: 'application/sql' });
  const url = URL.createObjectURL(blob);
  const a = document.createElement('a');
  a.href = url;
  a.download = filename;
  a.click();
  URL.revokeObjectURL(url);
}
```

## Technical Details

### File Changes Summary

| File | Change |
|------|--------|
| `src/components/deploy/DatabaseTreeContextMenu.tsx` | Add `onDownloadTableDDL` and `onDownloadSchemaDDL` props; add menu items for table and schema types |
| `src/components/deploy/DatabaseSchemaTree.tsx` | Thread new props through to context menu |
| `src/components/deploy/DatabaseExplorer.tsx` | Implement `handleDownloadTableDDL` and `handleDownloadSchemaDDL` handlers; pass them to the tree |

### Context Menu Additions

**Table node** (after "Get CREATE TABLE"):
```
[Download icon] Download DDL (.sql)
```

**Schema node** (before "Drop Schema CASCADE"):
```
[Download icon] Download Schema DDL (.sql)
---
[Trash icon] Drop Schema CASCADE
```

