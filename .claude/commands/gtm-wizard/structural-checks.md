# GTM Wizard — Structural Checks Module

## Purpose
Run all structural checks against the normalized container model. Detect event duplication, tag inefficiencies, and server container issues. Return a findings list for the report module.

## Input
- Normalized container model (from container-parser)
- Optional: normalized server container model (from container-parser, if sgtm_available)
- `site_url`: the site URL provided by the user (for SRV-1 domain matching)

## Custom HTML Parsing Notes

When extracting information from `rawHtml` in Custom HTML tags, be aware of:
- **Nested objects:** Use non-greedy patterns (`[\s\S]*?`) instead of `[^}]*` to avoid stopping at the first closing brace in nested structures.
- **GTM variable references:** Strings like `{{DLV - event_name}}` are runtime values, not literal strings. Skip these when extracting event names for static analysis.
- **Template literals and concatenation:** Code using backticks (`` ` ``), `${}` interpolation, or string concatenation (`+`) produces dynamic values that cannot be statically resolved. Skip these matches.
- **Obfuscated/minified code:** Some tags contain minified JavaScript. Regex extraction is best-effort — if a pattern cannot be reliably parsed, skip it rather than produce a false positive. Dedicated checks for obfuscation and minification are in SEC-1 and SEC-2.

## Output
A findings list:

```json
[
  {
    "id": "DUP-1",
    "severity": "HIGH",
    "platform": "structural",
    "affected_tags": ["42", "55"],
    "description": "GA4 Config tag (tag:42) and GA4 Event tag (tag:55) both fire page_view for measurement ID G-XXXXXXXXXX",
    "fix": "Pause the duplicate gaawe tag — the Config tag already sends page_view automatically"
  }
]
```

Severity levels: `HIGH`, `MEDIUM`, `WARN`, `INFO`

If no findings, return an empty array `[]` and report "No structural issues found."

---

## Check Definitions

### DUP-1 — Config + page_view overlap
**Severity:** HIGH

**Detection logic:**
1. Find all GA4 configuration tags: type `gaawc` (extract `measurementId` parameter) OR type `googtag` where the `tagId` parameter starts with `G-` (extract `tagId` as the measurement ID). For `googtag` tags, also check if `send_page_view` is set to `false` in `configSettingsTable` — if it is, this config tag does NOT auto-send page_view, so skip it for this check.
2. Find all tags with `type === "gaawe"` (GA4 Event tags) where `parameter[]` contains `key === "eventName"` with `value === "page_view"`. For each, extract its `measurementId` parameter (may be a direct value or a `TAG_REFERENCE` to a config tag).
3. For each (config tag, event tag) pair sharing the same `measurementId`: check if their `firingTriggerId` arrays have any overlap. When comparing trigger IDs, treat custom triggers that are functionally equivalent to built-in triggers as overlapping (e.g., a custom `PAGEVIEW` trigger and the built-in All Pages trigger `2147479573` overlap).
4. If overlap exists AND the config tag sends page_view automatically → flag.

**Finding:**
- `affected_tags`: IDs of the config tag and the event tag
- `description`: "GA4 Config tag (tag:{id}) and GA4 Event tag (tag:{id}) both fire page_view for measurement ID {measurementId}"
- `fix`: "Pause the duplicate gaawe event tag (tag:{event_tag_id}) — the Config tag already sends page_view automatically. OR set `send_page_view: false` on the Config tag if you want manual control."

---

### DUP-2 — Multi-trigger double-fire
**Severity:** MEDIUM

**Detection logic:**
1. For each tag with `firingTriggerId.length > 1`:
2. Resolve each trigger ID to its trigger object (including built-ins).
3. Check if any two triggers in the list fire on the same user action. Heuristic: two triggers of the same `type` (e.g., both `PAGE_VIEW`, or both `CUSTOM_EVENT` with the same event name filter) are considered overlapping.
4. If overlap detected → flag.

**Finding:**
- `affected_tags`: the tag's ID
- `description`: "Tag {name} (tag:{id}) has multiple triggers that may both fire for the same user action: {trigger names}"
- `fix`: "Consolidate into a single trigger or add conditions to make them mutually exclusive"

---

### DUP-3 — Catch-all re-fire
**Severity:** HIGH

**Detection logic:**
1. For each tag, check if `firingTriggerId` contains BOTH:
   - A page-view-on-every-page trigger: either the built-in All Pages ID `"2147479573"`, OR a custom trigger with `type === "PAGEVIEW"` and no restrictive filter conditions
   - At least one other trigger ID
2. If true → flag.

**Finding:**
- `affected_tags`: the tag's ID
- `description`: "Tag {name} (tag:{id}) fires on every page view AND on a specific trigger — it will fire twice on matching pages"
- `fix`: "Remove the page-view trigger from tag:{id} and rely only on the specific trigger"

---

### DUP-4 — Chain reaction double-fire
**Severity:** MEDIUM

**Detection logic:**
1. Collect all Custom HTML tags (`type === "html"`) whose `rawHtml` contains `dataLayer.push`. Extract the pushed event name(s) using regex: `/dataLayer\.push\(\s*\{[\s\S]*?event\s*:\s*['"]([^'"]+)['"]/g` — use `[\s\S]*?` (non-greedy, crosses newlines) instead of `[^}]*` to handle nested objects and multi-line code.
   - **Skip** matches where the event name looks like a GTM variable reference (contains `{{` and `}}`), a JavaScript expression (contains `+` or backticks), or a template literal placeholder (`${`). These are dynamic values that cannot be statically resolved.
   - **Skip** matches inside commented-out code (lines starting with `//` or inside `/* */` blocks).
2. For each extracted event name, find tags that have a `CUSTOM_EVENT` trigger where the trigger's filter matches that event name.
3. Check if those triggered tags also contain `dataLayer.push` in their `rawHtml` (chain).
4. If a chain is found → flag both the originating tag and the triggered tag.

**Finding:**
- `affected_tags`: IDs of the chain-originating tag and triggered tag
- `description`: "Tag {name} (tag:{id}) pushes event '{event}' to dataLayer, which fires tag {name2} (tag:{id2}), creating a potential chain reaction"
- `fix`: "Verify the chain is intentional; if not, remove the dataLayer.push from tag:{id2} or add a guard condition"

---

### EFF-1 — Redundant config tags
**Severity:** HIGH

**Detection logic:**
1. Collect all GA4 configuration tags: type `gaawc` (legacy GA4 Config) OR type `googtag` (Google Tag). For `gaawc`, extract `measurementId` from `parameter[]` where `key === "measurementId"`. For `googtag`, extract `tagId` from `parameter[]` where `key === "tagId"` — only include entries starting with `G-` (GA4 measurement IDs); skip `AW-` entries (Google Ads).
2. Group these tags by their measurement ID value.
3. Any group with more than one **active** (non-paused) tag → flag all active tags in that group. Exclude paused tags from groups — they are typically old versions being replaced.

**Finding:**
- `affected_tags`: all active tag IDs sharing the same measurementId
- `description`: "Multiple GA4 Config tags ({count}) found for measurement ID {measurementId}: tag:{id1}, tag:{id2}, ..."
- `fix`: "Keep only one GA4 Config tag per measurement ID; remove the duplicates"

---

### EFF-2 — Overly broad triggers
**Severity:** MEDIUM

**Detection logic:**
1. Identify conversion/pixel tags: any tag with type `awct` (Google Ads Conversion), `gaawe` (GA4 Event used for conversion), or Custom HTML tags containing `fbq(`, `ttq.track`, `lintrk(`, `twq(`, `snaptr(`, `pintrk(`, `uetq`.
2. For each such tag, resolve every trigger ID in `firingTriggerId` to its trigger object.
3. Check if any resolved trigger is a page-view-on-every-page trigger. A trigger matches if:
   - It is the built-in All Pages trigger (`"2147479573"`), OR
   - It has `type === "PAGEVIEW"` with no filter conditions (or only broad filters that match all pages)
4. If true → flag.

**Finding:**
- `affected_tags`: the tag's ID
- `description`: "Conversion/pixel tag {name} (tag:{id}) fires on every page view — this will over-count conversions"
- `fix`: "Replace the page-view trigger with a specific conversion trigger (e.g., thank-you page URL, custom event, or form submission)"

---

### EFF-3 — Paused tags
**Severity:** WARN

**Detection logic:**
1. Collect all tags where `status === "paused"`.
2. Each paused tag is a separate finding.

**Finding:**
- `affected_tags`: the tag's ID
- `description`: "Tag {name} (tag:{id}) is paused and will not fire"
- `fix`: "If this tag is no longer needed, delete it to keep the container clean; if it should be active, unpause it"

---

### EFF-4 — Asymmetric trigger coverage
**Severity:** MEDIUM

**Detection logic:**
1. Group **active** (non-paused) tags into "event families" — tags that share the same event name or measurement purpose. Exclude paused tags entirely — they are typically old versions being replaced and would cause false positives. Heuristic grouping:
   - GA4 Event tags (`gaawe`) with the same `eventName` parameter value form a family.
   - Meta pixel tags pushing the same event (e.g., `fbq('track', 'Purchase')`) form a family.
2. Within each family, compare `firingTriggerId` sets across all tags.
3. If any two tags in the same family have different `firingTriggerId` sets → flag.

**Finding:**
- `affected_tags`: IDs of the mismatched tags
- `description`: "Tags in the '{event}' family have different trigger sets: tag:{id1} fires on {triggers1}, tag:{id2} fires on {triggers2}"
- `fix`: "Ensure all tags for the same event use identical trigger sets, or explain the intentional difference in a tag note"

---

### EFF-5 — Firing option mismatch
**Severity:** MEDIUM

**Detection logic:**
1. Use the same event family grouping as EFF-4.
2. For each tag, extract `tagFiringOption` from `parameter[]` where `key === "tagFiringOption"`. Default is `"oncePerEvent"` if not set.
3. Within each family, if tags have different `tagFiringOption` values → flag.

**Finding:**
- `affected_tags`: IDs of the mismatched tags
- `description`: "Tags in the '{event}' family have different firing option settings: tag:{id1} uses '{option1}', tag:{id2} uses '{option2}'"
- `fix`: "Align the tagFiringOption across all tags in the same event family to prevent inconsistent firing behavior"

---

### SEC-1 — Obfuscated code in Custom HTML tags
**Severity:** WARN

**Detection logic:**
1. Collect all Custom HTML tags (`type === "html"`) with `rawHtml` content.
2. Test `rawHtml` against these patterns (any match triggers the finding):
   - `eval(` — dynamic code execution
   - `new Function(` — dynamic function construction
   - `atob(` — base64 decoding
   - `String.fromCharCode(` — char-code string building
   - Hex escape sequences: 4+ consecutive `\xNN` patterns (`/(?:\\x[0-9a-fA-F]{2}){4,}/`)
   - Unicode escape sequences: 4+ consecutive `\uNNNN` patterns (`/(?:\\u[0-9a-fA-F]{4}){4,}/`)
   - Bracket notation with string concatenation to hide property names: `/\w+\[['"][a-zA-Z]+['"]\s*\+\s*['"][a-zA-Z]+['"]\]/`
   - `document.write(unescape(` — classic obfuscation wrapper
3. One finding per tag. If multiple patterns match, list all matched pattern names in the description.

**Finding:**
- `affected_tags`: the tag's ID
- `description`: "Tag '{name}' (tag:{id}) contains potentially obfuscated code (matched: {pattern_names}). Review manually to verify this tag's intent."
- `fix`: null

---

### SEC-2 — Minified code in Custom HTML tags
**Severity:** INFO

**Detection logic:**
1. Collect all Custom HTML tags (`type === "html"`) with `rawHtml` content.
2. Skip tags already flagged by SEC-1 — obfuscation is the higher-priority finding.
3. Strip `<script>` / `</script>` wrapper tags and leading/trailing whitespace from `rawHtml`.
4. Split the remaining content by newlines.
5. If any single line exceeds 500 characters of JavaScript → flag. Report the length of the longest line.

**Finding:**
- `affected_tags`: the tag's ID
- `description`: "Tag '{name}' (tag:{id}) contains minified JavaScript ({N} characters on a single line) that is difficult to audit"
- `fix`: null

---

### SRV-1 — Client domain mismatch
**Severity:** HIGH
**Condition:** Only run if server container model is provided.

**Detection logic:**
1. Extract the hostname from `site_url` (e.g., `https://www.example.com` → `www.example.com` and `example.com`).
2. In the server container model, find all tags of type `remm` (GA4 client) or custom client tags. Extract their `hostname` or `requestPath` parameters.
3. If no client tag domain matches the site URL hostname → flag.

**Finding:**
- `affected_tags`: IDs of the mismatched client tags
- `description`: "Server container client(s) are configured for domain(s) {server_domains} but site URL is {site_url}"
- `fix`: "Update the server container client configuration to accept requests from {site_url_hostname}"

---

### SRV-2 — Missing FPID config
**Severity:** MEDIUM
**Condition:** Only run if server container model is provided.

**Detection logic:**
1. In the web container, search for any variable or tag that references `FPID`, `_fbc`, `_fbp`, or first-party cookie configuration.
2. In the server container, search for any variable of type `smm` (cookie variable) reading a first-party ID cookie.
3. If neither is found → flag.

**Finding:**
- `affected_tags`: []
- `description`: "No first-party ID (FPID) cookie configuration found in web or server container"
- `fix`: "Add a first-party cookie variable in GTM server to read and persist a FPID cookie; this improves user identity resolution and event match quality"

---

### SRV-3 — Catch-all trigger filters
**Severity:** MEDIUM
**Condition:** Only run if server container model is provided.

**Detection logic:**
1. In the server container, iterate all triggers.
2. For each trigger where `filter[]` is empty or absent and `isBuiltIn === false` → flag.

**Finding:**
- `affected_tags`: IDs of tags using this trigger
- `description`: "Server container trigger '{trigger_name}' (trigger:{id}) has no filter conditions and will fire for all incoming requests"
- `fix`: "Add filter conditions to restrict this trigger to the intended request type (e.g., filter by event name, client name, or request path)"

---

### SRV-4 — Orphan server tags
**Severity:** WARN
**Condition:** Only run if server container model is provided.

**Detection logic:**
1. In the server container, collect all tag names.
2. In the web container, collect all tag names and Custom HTML content.
3. A server tag is "orphaned" if no web container tag appears to send data to it (heuristic: no web tag references a matching event name or the server container URL in its parameters or rawHtml).
4. Flag each orphaned server tag.

**Finding:**
- `affected_tags`: the server tag's ID
- `description`: "Server container tag '{name}' (tag:{id}) has no corresponding web container tag sending data to it"
- `fix`: "Verify this server tag is receiving data via another path (e.g., direct HTTP requests); if not, it may be unused and can be removed"

---

## Execution Order

Run checks in this order to allow later checks to reference earlier findings:

1. EFF-1 (redundant configs — needed context for DUP-1)
2. DUP-1
3. DUP-2
4. DUP-3
5. DUP-4
6. EFF-2
7. EFF-3
8. EFF-4
9. EFF-5
10. SEC-1
11. SEC-2 (skip tags already flagged by SEC-1)
12. SRV-1 (skip if no server container)
13. SRV-2 (skip if no server container)
14. SRV-3 (skip if no server container)
15. SRV-4 (skip if no server container)

After all checks: report total counts by severity. Example: "Found 2 HIGH, 3 MEDIUM, 1 WARN issues."
