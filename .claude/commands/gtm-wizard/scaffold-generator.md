# GTM Wizard — Scaffold Generator Module

## Purpose
For each platform in `added[]`, generate a complete set of tags, triggers, and variables to inject into the container. Uses platform module scaffold templates and event mapping tables.

## Input
- Platform modules (from `platforms/` directory) with scaffold templates and event mappings
- User's conversion events (from orchestrator)
- Research agent results (best practices)
- Existing container model (for ID conflict avoidance and variable reuse)

## Output
```json
{
  "meta": {
    "tags": [...],
    "triggers": [...],
    "variables": [...]
  }
}
```

Each key in the output object is a platform name from `added[]`. Tags, triggers, and variables follow GTM export JSON field conventions (same shape as the raw container arrays).

---

## Process (per added platform)

### Step 1 — Load platform module
Read the platform module file at `platforms/{platform-slug}.md`. Extract:
- The **event mapping table**: maps user-facing event names to the platform's native event names (e.g., `generate_lead` → `Lead`)
- The **scaffold templates**: JSON template blocks for the base/config tag, event tag, trigger, and required variables
- The **required parameters**: list of parameters the user must supply (e.g., Pixel ID, Access Token)

Platform slug mapping:

| Platform name | Module file |
|---|---|
| Meta | `platforms/meta.md` |
| TikTok | `platforms/tiktok.md` |
| Google Ads | `platforms/google-ads.md` |
| LinkedIn | `platforms/linkedin.md` |
| Twitter/X | `platforms/twitter-x.md` |
| Snapchat | `platforms/snapchat.md` |
| Pinterest | `platforms/pinterest.md` |
| Microsoft/Bing Ads | `platforms/microsoft-bing.md` |

### Step 2 — Map user events
For each conversion event the user specified:
1. Look up the event name in the platform module's event mapping table.
2. If a mapping exists → use the mapped platform event name.
3. If no mapping exists → use the user's event name as-is (custom event). Record it in the unmapped list.

After mapping, report:
- Mapped events: "'{user_event}' → '{platform_event}'"
- Unmapped events: "'{user_event}' has no standard mapping for {Platform} — will be created as a custom event"

### Step 3 — Resolve ID counter
Set `nextId = container.maxId + 1`. Every new item (tag, trigger, variable) gets `nextId` as its ID, then `nextId` is incremented. This guarantees no collision with existing container IDs.

### Step 4 — Reuse existing triggers
Before generating a new trigger, check the existing container model's `triggers[]` array. If a `CUSTOM_EVENT` trigger already exists whose filter matches the same `datalayer_event_name`, reuse its ID instead of creating a new trigger. Record reused trigger IDs in the scaffold so tags can reference them correctly.

### Step 5 — Generate variables
For each required parameter listed in the platform module:
1. Check if a variable with the exact name `[GTM Wizard] DLV - {Platform} {Parameter}` already exists in the container model's `variables[]`. If it does, reuse it.
2. Otherwise, create a new variable using the template from the platform module:
   - `id`: `nextId++`
   - `name`: `[GTM Wizard] DLV - {Platform} {Parameter}` (e.g., `[GTM Wizard] DLV - Meta Pixel ID`)
   - Fill remaining fields from the platform module's variable template.

### Step 6 — Generate the base/config tag
Create one base/config tag per platform (pixel initialization code):
- `id`: `nextId++`
- `name`: `[GTM Wizard] {Platform} - Base` (e.g., `[GTM Wizard] Meta - Base`)
- Fill all fields from the platform module's base tag template, substituting variable references where the template specifies.
- `firingTriggerId`: `["2147479573"]` (All Pages built-in) unless the platform module specifies otherwise.

### Step 7 — Generate triggers
For each unique `datalayer_event_name` across all mapped events (after Step 4 deduplication):
- If a reusable trigger was identified in Step 4, skip creation.
- Otherwise create a new trigger:
  - `id`: `nextId++`
  - `name`: `[GTM Wizard] Custom Event - {datalayer_event_name}` (e.g., `[GTM Wizard] Custom Event - generate_lead`)
  - Use the trigger template below.

### Step 8 — Generate event tags
For each mapped event:
1. Look up the trigger ID for this event's `datalayer_event_name` (from Step 7 or reused trigger from Step 4).
2. Create one event tag using the platform module's event tag template:
   - `id`: `nextId++`
   - `name`: `[GTM Wizard] {Platform} - {Event}` (e.g., `[GTM Wizard] Meta - Lead`)
   - `firingTriggerId`: `[trigger_id_for_this_event]`
   - Substitute the platform event name and any variable references per the template.

### Step 9 — Server-side tag (conditional)
If `sgtm_available === true` AND the platform module defines a server-side template (CAPI/Events API tag):
1. Create one additional server-side tag per event using the module's server template.
2. Apply the same naming convention: `[GTM Wizard] {Platform} CAPI - {Event}`.
3. Assign IDs from `nextId++` continuing the sequence.
4. Note in the report: "Server-side {Platform} tag added for event '{Event}'."

---

## Naming Conventions

| Item type | Pattern | Example |
|---|---|---|
| Tag (event) | `[GTM Wizard] {Platform} - {Event}` | `[GTM Wizard] Meta - Lead` |
| Tag (base) | `[GTM Wizard] {Platform} - Base` | `[GTM Wizard] Meta - Base` |
| Tag (server) | `[GTM Wizard] {Platform} CAPI - {Event}` | `[GTM Wizard] Meta CAPI - Lead` |
| Variable | `[GTM Wizard] DLV - {Platform} {Parameter}` | `[GTM Wizard] DLV - Meta Pixel ID` |
| Trigger | `[GTM Wizard] Custom Event - {datalayer_event_name}` | `[GTM Wizard] Custom Event - generate_lead` |

---

## Trigger Template

Generated for each unique `datalayer_event_name` that does not already have a reusable trigger:

```json
{
  "id": "{nextId}",
  "name": "[GTM Wizard] Custom Event - {datalayer_event_name}",
  "type": "CUSTOM_EVENT",
  "isBuiltIn": false,
  "filter": [],
  "customEventFilter": [
    {
      "type": "EQUALS",
      "parameter": [
        { "type": "TEMPLATE", "key": "arg0", "value": "{{_event}}" },
        { "type": "TEMPLATE", "key": "arg1", "value": "{datalayer_event_name}" }
      ]
    }
  ]
}
```

---

## Important Rules

- **Never reuse existing IDs.** Always start from `container.maxId + 1` and increment per item.
- **Reuse existing triggers** when a matching `CUSTOM_EVENT` trigger is already present — do not create duplicates.
- **Cross-reference integrity**: after all IDs are assigned, verify every new tag's `firingTriggerId` array contains only IDs that exist in either the existing container triggers or the newly generated triggers list.
- **Platform modules are authoritative** — the exact JSON field names, parameter keys, and tag types come from the platform module templates. This module only fills in event names, IDs, and trigger references.
- **Report unmapped events** in the output summary so the user is aware which events were created as custom events.

---

## Steps

1. For each platform in `added[]` (process one at a time):
   a. Load the platform module from `platforms/{slug}.md` using the Read tool.
   b. Execute Steps 1–9 above for that platform.
   c. Collect the generated tags, triggers, and variables into the platform's output bucket.
2. After all platforms are processed, assemble the final output object keyed by platform name.
3. Report summary per platform:
   - "Built scaffold for {Platform}: {N} tags, {M} triggers, {P} variables."
   - List each generated item by name and assigned ID.
   - List any unmapped events.
4. Return the assembled scaffolds object to the orchestrator.
