# GTM Container Parser Module

## Purpose
Parse a GTM container export JSON file into a normalized container model. Called by the orchestrator — do not invoke directly.

## Input
- `container_path`: absolute or relative path to the GTM export JSON file

## Output
A normalized container model object held in memory (not written to disk). Structure:

```json
{
  "containerId": "GTM-XXXXXXX",
  "containerType": "web|server",
  "containerName": "...",
  "tags": [{ "id": "N", "name": "...", "type": "...", "status": "active|paused", "firingTriggerId": [...], "parameter": [...], "rawHtml": "..." }],
  "triggers": [{ "id": "N", "name": "...", "type": "...", "isBuiltIn": true|false, "filter": [...] }],
  "variables": [{ "id": "N", "name": "...", "type": "...", "parameter": [...] }],
  "clients": [{ "id": "N", "name": "...", "type": "...", "parameter": [...] }],
  "maxId": N
}
```

## Steps

1. Use the Read tool to read the file at `container_path`.
2. Parse as JSON. The root structure is `{ exportFormatVersion, exportTime, containerVersion: { ... } }`.
3. Extract from `containerVersion`:
   - `container.publicId` → `containerId`
   - `container.name` → `containerName`
   - `container.usageContext` → if includes `"SERVER"` then `containerType = "server"`, else `"web"`
   - `tag[]` → `tags` (normalize each, see below)
   - `trigger[]` → `triggers` (normalize each, see below)
   - `variable[]` → `variables` (normalize each, see below)
   - `client[]` → `clients` (normalize each, see below; server containers only — web containers have no `client[]` array, set `clients: []`)

### Tag normalization
For each entry in `tag[]`:
- `id`: value of `tag.tagId` (as string)
- `name`: `tag.name`
- `type`: `tag.type`
- `status`: if `tag.paused === true` → `"paused"`, else `"active"`
- `firingTriggerId`: `tag.firingTriggerId || []` (array of string IDs)
- `parameter`: `tag.parameter || []` (keep raw — other modules read specific params)
- `rawHtml`: if `tag.type === "html"`, find `parameter` where `key === "html"` and return its `value`; else `null`

### Trigger normalization
Built-in GTM trigger IDs (these exist in every container without being explicitly defined):
- `"2147479573"` = All Pages (Page View)
- `"2147479574"` = All Elements (Click)
- `"2147479575"` = Window Loaded
- `"2147479576"` = DOM Ready
- `"2147479553"` = Consent Initialization
- `"2147479572"` = Initialization

For each entry in `trigger[]`:
- `id`: `trigger.triggerId` (as string)
- `name`: `trigger.name`
- `type`: `trigger.type` (e.g., `PAGE_VIEW`, `CUSTOM_EVENT`, `ALWAYS`)
- `isBuiltIn`: false (explicit triggers are never built-in)
- `filter`: `trigger.filter || []`

For built-in triggers referenced in `firingTriggerId` but not in the `trigger[]` array, synthesize entries with `isBuiltIn: true` and the names above.

### Variable normalization
For each entry in `variable[]`:
- `id`: `variable.variableId` (as string)
- `name`: `variable.name`
- `type`: `variable.type`
- `parameter`: `variable.parameter || []`

### Client normalization (server containers only)
For each entry in `client[]` (present only in server container exports):
- `id`: `client.clientId` (as string)
- `name`: `client.name`
- `type`: `client.type`
- `parameter`: `client.parameter || []`

For web containers, set `clients: []`.

### maxId computation
`maxId = max of all numeric IDs across tags, triggers, and variables`

4. Return the normalized model. Report: "Parsed container `{containerId}` ({containerType}): {N} tags, {M} triggers, {P} variables. Max ID: {maxId}."
