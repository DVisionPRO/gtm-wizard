# GTM Wizard — Patch Generator Module

## Purpose
Apply structural check fixes to existing tags/triggers, merge scaffold additions, write patched JSON files. Maintain a changelist for the report.

## Input
- Original raw GTM export JSON file(s) — the file paths from the orchestrator
- Normalized container model(s) — from container-parser (held in memory)
- Findings list from structural-checks (and any platform module checks)
- Scaffolds object from scaffold-generator (may be empty if mode is `"diagnose"`)

## Output
- Patched GTM export JSON file(s) written to disk
- Changelist array returned to the orchestrator for the report-generator

---

## Safety Rules (CRITICAL — enforce throughout all phases)

- **Never delete a tag.** If a fix would remove a tag, pause it instead (`"paused": true` / `tag.paused = true`).
- **Never remove a required parameter from a vendor tag.** GTM validates vendor template schemas on import (e.g., `gaawe` requires `eventName`, `awct` requires `conversionId`). Removing a required parameter causes "The value must not be empty" import errors. If a fix would leave a required parameter empty or missing, pause the tag instead.
- **Never leave a tag with an empty `firingTriggerId`.** If a fix would empty the array, pause the tag instead and record a MANUAL entry in the changelist.
- **Always validate JSON** before writing. The output file must be parseable GTM-format JSON with no trailing commas or syntax errors.
- Every modification — including pauses and skipped auto-fixes — must be annotated in the changelist.

---

## Phase A — Apply fixes from findings

Iterate the findings list. For each finding, look up its `id` and apply the corresponding fix to the normalized container model in memory:

### DUP-1 — Config + page_view overlap
Pause the affected `gaawe` tag (set `"paused": true`). Do NOT remove the `eventName` parameter — GTM requires `eventName` on `gaawe` tags and will reject the import with "vendorTemplate.parameter.eventName: The value must not be empty" if it is missing.

### DUP-2 — Multi-trigger double-fire
Remove the duplicate `firingTriggerId` entry from the affected tag. Keep the more specific trigger (prefer `CUSTOM_EVENT` over `PAGE_VIEW`; prefer explicit over built-in). If removal would empty the array → pause the tag instead.

### DUP-3 — Catch-all re-fire
Remove the All Pages built-in ID `"2147479573"` from the tag's `firingTriggerId` array. If removal would empty the array → pause the tag instead.

### EFF-1 — Redundant config tags
Within each group of `gaawc` tags sharing the same `measurementId`:
- Keep the first tag (lowest numeric ID) unchanged.
- Set `paused = true` on all others.

### EFF-2 — Overly broad triggers
Mark as **MANUAL** in the changelist. Do not modify the tag. The correct replacement trigger depends on site-specific page structure that cannot be inferred automatically.

### EFF-3 — Paused tags
No automatic change. Record a `"manual"` changelist entry prompting the user to review. Do not unpause or delete.

### EFF-4 — Asymmetric trigger coverage
Mark as **MANUAL** in the changelist. Asymmetric trigger sets may be intentional; human judgment is required before equalizing them.

### EFF-5 — Firing option mismatch
For all tags in the affected family, normalize `tagFiringOption` to `"ONCE_PER_EVENT"`:
- In each tag's `parameter[]`, find the entry where `key === "tagFiringOption"` and update its `value` to `"oncePerEvent"`.
- If no such parameter entry exists, add one: `{ "type": "TEMPLATE", "key": "tagFiringOption", "value": "oncePerEvent" }`.

### Platform-specific fixes (META-1, TT-1, etc.)
Apply parameter additions or corrections as specified in the platform module's fix guidance. Each platform module defines the exact parameter key, type, and value to add or update.

---

## Phase B — Merge scaffolds

If the scaffolds object is non-empty, process each platform:

1. Append all items from `scaffolds[platform].tags[]` to the container model's `tags[]`.
2. Append all items from `scaffolds[platform].triggers[]` to the container model's `triggers[]`.
3. Append all items from `scaffolds[platform].variables[]` to the container model's `variables[]`.

IDs are already assigned sequentially above `maxId` by scaffold-generator — no reassignment needed. Cross-references (`firingTriggerId` values in new tags) are already correct.

For each appended item, add a changelist entry of type `"add"` with `"scaffold": true`.

---

## Phase C — Write output

1. Take the **original raw GTM export JSON** (as read from disk, before any in-memory modifications).
2. **Denormalize** the modified container model arrays back to raw GTM export format before writing:
   - Tags: `id` → `tagId`, `status === "paused"` → `paused: true` (omit `paused` field if active), remove `rawHtml` (it is derived from `parameter[]`, not a GTM field), remove `status` field
   - Triggers: `id` → `triggerId`, remove `isBuiltIn` field (built-in triggers are not stored in the export)
   - Variables: `id` → `variableId`
   - Preserve all other fields on each object (e.g., `name`, `type`, `parameter`, `firingTriggerId`, `tagFiringOption`, `fingerprint`)
3. Replace the following arrays inside `containerVersion` with the denormalized versions:
   - `tag[]` ← denormalized modified + appended tags
   - `trigger[]` ← denormalized modified + appended triggers (excluding built-in triggers)
   - `variable[]` ← denormalized modified + appended variables
4. Preserve all other fields in the export JSON exactly as-is (`exportFormatVersion`, `exportTime`, `containerVersion.container`, `containerVersion.fingerprint`, etc.).
5. Construct the output filename: strip the extension from the original filename and append `_patched.json`.
   - Example: `GTM-ABC123_web.json` → `GTM-ABC123_web_patched.json`
   - Write to the same directory as the original file.
6. Write the reconstructed JSON to disk using the Write tool with 2-space indentation.
7. Confirm: "Wrote patched container to `{output_path}`."

---

## Changelist Format

Every modification must produce one changelist entry. Collect all entries and return the array.

```json
[
  { "type": "pause",  "target": "tag:42", "check": "DUP-1", "description": "Paused duplicate page_view event tag (Config tag already fires page_view automatically)" },
  { "type": "modify", "target": "tag:17", "check": "DUP-3", "description": "Removed All Pages trigger from firingTriggerId (kept specific trigger 88)" },
  { "type": "pause",  "target": "tag:99", "check": "EFF-1", "description": "Duplicate GA4 Config tag paused (keeping tag:14 for measurement ID G-XXXXXXX)" },
  { "type": "modify", "target": "tag:33", "check": "EFF-5", "description": "Normalized tagFiringOption to ONCE_PER_EVENT" },
  { "type": "add",    "target": "tag:101",  "scaffold": true, "platform": "Meta", "description": "[GTM Wizard] Meta - Base" },
  { "type": "add",    "target": "tag:102",  "scaffold": true, "platform": "Meta", "description": "[GTM Wizard] Meta - Lead" },
  { "type": "add",    "target": "trigger:103", "scaffold": true, "platform": "Meta", "description": "[GTM Wizard] Custom Event - generate_lead" },
  { "type": "add",    "target": "variable:104", "scaffold": true, "platform": "Meta", "description": "[GTM Wizard] DLV - Meta Pixel ID" },
  { "type": "manual", "check": "EFF-2", "target": "tag:55", "description": "Overly broad trigger on conversion tag — replace All Pages with a specific conversion trigger (manual action required)" },
  { "type": "manual", "check": "EFF-4", "target": "tag:60,tag:61", "description": "Asymmetric trigger coverage in 'purchase' family — review and align triggers manually" }
]
```

### Changelist entry fields

| Field | Required | Description |
|---|---|---|
| `type` | yes | `"modify"`, `"pause"`, `"add"`, or `"manual"` |
| `target` | yes | `"tag:{id}"`, `"trigger:{id}"`, `"variable:{id}"`, or comma-separated list |
| `check` | when applicable | The finding ID that caused this change (e.g., `"DUP-1"`) |
| `scaffold` | for `add` type | `true` if this item came from scaffold-generator |
| `platform` | for scaffold adds | Platform name |
| `description` | yes | Human-readable explanation of what was changed and why |

---

## Steps

1. Use the Read tool to load the original GTM export JSON file from disk. Store as `originalJson`.
2. Take the normalized container model from memory (produced by container-parser earlier in the session).
3. **Phase A:** Iterate the findings list. For each finding, apply the fix described above. Record each action in the changelist.
4. **Phase B:** For each platform in the scaffolds object, append tags, triggers, and variables to the container model. Record each addition in the changelist.
5. **Phase C:** Reconstruct the export JSON and write the patched file using the Write tool.
6. Return the completed changelist to the orchestrator.
