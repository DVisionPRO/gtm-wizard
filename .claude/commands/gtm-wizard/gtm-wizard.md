# GTM Wizard

Your GTM container expert — diagnose, fix, and build.

## Overview

GTM Wizard analyzes your Google Tag Manager container exports, finds structural issues, suggests best-practice improvements, and can scaffold new platform implementations — all locally, without browser access or credentials. Only WebSearch queries for best practices are sent externally.

**Version:** 1.0.0

---

## Step 0: Explain the Workflow

Before any questions, print this introduction:

> **GTM Wizard v1.0.0**
>
> Here's what I'm going to do:
>
> 1. **Ask a few questions** — which container files to analyze, your site URL, and which platforms/events matter to you (about 1 minute)
> 2. **Analyze your container** — structural checks, platform-specific checks, and best-practice research
> 3. **Generate a report** — plain-language findings with clear next steps (`gtm-wizard-report.md`)
> 4. **Produce a patched container** — a fixed JSON file you can import back into GTM
>
> Your container data stays local — nothing is uploaded. The only external calls are web searches for current platform best practices.
>
> Let's get started.

---

## Step 1: Load Configuration

Use the Read tool to read `.gtm-wizard-config.json` from the current working directory.

- If the file exists and is valid JSON, load its values as defaults for the questions below.
- If it does not exist or cannot be parsed, start fresh with no defaults.

Config schema:
```json
{
  "containers": [],
  "site_url": "",
  "platform_overrides": "none",
  "conversion_events": [],
  "last_run": ""
}
```

---

## Step 2: Interactive Input

Ask the user the following questions one at a time, in order.

**Default handling:** When a question has a recommended default, present it as a numbered option (e.g., the first or last option in the list). This way the user can select the default by typing its number rather than having to type the full value. Always make it clear which option is the default.

**Handling arbitrary answers:** Users may respond in natural language instead of the expected format (e.g., "just the first one" instead of "1", or "my site is example.com" instead of just the URL). Interpret the intent behind any response — extract the relevant information and proceed. Only ask for clarification if the answer is genuinely ambiguous.

### Q1: Container Files

Use the Glob tool with pattern `*.json` to find JSON files in the current working directory. Exclude files ending in `_patched.json` — these are output artifacts from previous runs, not input files. For each remaining file found, use the Read tool to check whether it contains `"exportFormatVersion"` — this field confirms it is a GTM export.

For each confirmed GTM export file, also detect the container type:
- If the parsed JSON contains a container with `usageContext` including `"SERVER"` → type is `Server`
- Otherwise → type is `Web`

Present confirmed files using bullet points (not numbered — numbers are reserved for the selection options below). Then ask with numbered selection options. Output **exactly** this format (do not rephrase or add parenthetical hints):

```
GTM export files found:
  • GTM-ABC123.json  [Web]
  • GTM-SRV456.json  [Server]

Which files should I analyze?
  1. All of the above (default)
  2. Let me pick specific files
```

If the user picks option 2, ask them to enter filenames or paths.

If no GTM export files are found in the current directory, ask the user to enter file paths directly.

### Q2: Website URL

Ask:
> "What is your website URL?"
> Default: [{config.site_url} or none]

This URL is used to validate server container client domains and to focus research queries. Accept any string; do not validate format here.

### Q3: Parse and Detect Server-Side Tagging

Run the container-parser module (`commands/gtm-wizard/container-parser.md`) on all selected container files. Merge the parsed models into a combined view.

**Server container detection:** If all selected containers are `web` type (no `server` container was included), scan the parsed web container(s) for server-side tagging signals:

1. **`transport_url` parameter** — in `googtag` or `gaawc` tags, look for a `configSettingsTable` or `configSettingsVariable` map entry where `parameter === "transport_url"`. The value will be a server container endpoint URL (e.g., `https://gtm-XXXXXXX-xxxxx.uc.r.appspot.com`).
2. **`serverContainerUrl` parameter** — in any tag's `parameter[]`, look for `key === "serverContainerUrl"`.
3. **CAPI / Events API tag names** — tags whose names contain `CONVERSIONS_API`, `CAPI`, `Events_API`, or `SERVER` suggest server-side integration.

If any signals are found, report to the user:

> "Your web container is configured to send data to a server container (detected via {signal description, e.g., 'transport_url pointing to gtm-p768vqt-xxxxx.uc.r.appspot.com'}). For a complete analysis, I need the server container export too — without it, I can't verify server-side tag configuration, CAPI setup, or event routing."
>
> "Do you have the server container JSON export? (enter file path, or 'skip' to continue without it)"

If the user provides a file path:
- Parse it with container-parser
- Verify its `containerType === "server"`
- Add it to the combined model
- Add it to the selected containers list

If the user says `skip`:
- Continue without the server container
- Note in the report that server-side analysis was skipped despite detected signals

### Q4: Platform Confirmation

Run the platform-detector module (`commands/gtm-wizard/platform-detector.md`) on the combined parsed models. This produces `detected_platforms[]`.

Report to the user:
> "I found these platforms in your container(s): {comma-separated list of detected platform display names}"

If no platforms were detected, say so.

Ask:
> "Any platforms to skip or add? (e.g. 'skip LinkedIn, add TikTok') Enter 'none' to continue."
> Default: [{config.platform_overrides} or none]

Parse the response:
- Extract `skip {platform}` instructions → remove matching platforms from `detected_platforms`
- Extract `add {platform}` instructions → add to `added_platforms[]`
- `"none"` or empty → no changes

Maintain two lists going forward:
- `detected_platforms[]` — existing platforms to diagnose and fix
- `added_platforms[]` — new platforms to scaffold (Build mode)

### Q5: Conversion Events

From the combined container model, extract event names used in CUSTOM_EVENT triggers. Look at the `customEventFilter` parameter for each trigger — the `value` field within a condition of type `EQUALS` or `MATCHES_REGEX` is the event name.

Present detected events as a bulleted list, then offer numbered selection options:

```
Detected conversion events:
  • recibido_visible
  • purchase

Which conversion events matter to you? These drive deduplication checks and tag scaffolding.
  1. All of the above (default)
  2. Let me pick specific events
```

If the user picks option 2, ask them to enter event names (comma-separated). The user may also add event names that are not currently in the container.

### Q6: Research Cache

Check whether a `.gtm-wizard-research/` directory exists in the current working directory.

If it exists, attempt to read `.gtm-wizard-research/.cache-meta.json`. For each platform in the active list (`detected_platforms` union `added_platforms`), check `cache_entries[platform].cached_at` against the current date.

Default TTL: 7 days. If an entry is older than 7 days, it is stale.

If any platforms have stale cache entries, ask:
> "Research cache for {stale platform list} is {N} days old. Refresh? [y/N]"
> Default: N (use cached data)

If the user enters `y` or `yes`, mark those platforms for a fresh research run. Otherwise, use the cached summaries.

---

## Step 3: Save Configuration

Write the user's answers back to `.gtm-wizard-config.json` in the current working directory using the Write tool:

```json
{
  "containers": ["./GTM-EXAMPLE.json"],
  "site_url": "https://www.example.com",
  "platform_overrides": "none",
  "conversion_events": ["generate_lead", "purchase"],
  "last_run": "{current ISO timestamp}"
}
```

---

## Step 4: Run Analysis Pipeline

Execute the following modules in order. Print a one-line progress report after each step completes.

### 4a: Parse Containers

For each selected container file, follow `commands/gtm-wizard/container-parser.md`.

Progress report (after each file):
> "Parsed {containerId} ({containerType}): {N} tags, {M} triggers, {P} variables"

### 4b: Structural Checks

Follow `commands/gtm-wizard/structural-checks.md` on the combined parsed models.

Pass in:
- The combined container model (web + server if both present)
- `conversion_events[]`
- `site_url`

Collect all findings into `findings[]`.

Progress report:
> "Structural checks complete: {N} issues found"

### 4c: Platform Checks

For each platform in `detected_platforms`:

1. Load the platform module from `commands/gtm-wizard/platforms/{platform-slug}.md`.
2. Run the platform-specific checks defined in that module.
3. Append findings to `findings[]`.

Progress report (after each platform):
> "{Platform display name} checks complete: {N} issues found"

If a platform module file does not exist for a given platform, skip checks for that platform and note: `"No platform module found for {platform} — skipping checks."`

### 4d: Research

Follow `commands/gtm-wizard/research-agent.md` for each platform in the active list (`detected_platforms` union `added_platforms`), respecting cache state determined in Q5.

Collect results into `research_summaries{}` keyed by platform slug.

Progress report:
> "Research complete for {platforms}"

### 4e: Scaffold (Build Mode)

For each platform in `added_platforms`:

1. Load the platform module from `commands/gtm-wizard/platforms/{platform-slug}.md`.
2. Follow `commands/gtm-wizard/scaffold-generator.md` using the platform module templates and `conversion_events[]`.
3. Collect the generated tags, triggers, and variables.

Progress report (after each platform):
> "Scaffolded {N} tags, {M} triggers, {P} variables for {platform display name}"

### 4f: Review Findings and Approve Patches

Before generating any patched files, present the findings to the user for approval.

**Present a numbered summary of all auto-fixable findings**, grouped by importance:

```
## Proposed Fixes

Critical:
  1. [DUP-1] GA4 Config + Event tag both fire page_view for G-XXXXXXX
     → Pause duplicate event tag

Recommended:
  2. [EFF-5] Tags in 'purchase' family have different firing options
     → Normalize to ONCE_PER_EVENT

Informational (no auto-fix — manual action needed):
  3. [EFF-3] Tag "Old Pixel" is paused
  4. [EFF-2] Conversion tag fires on every page view

Which fixes should I apply? (enter numbers, 'all', or 'none')
```

- If the user selects specific numbers → only apply those fixes
- If the user says `all` → apply all auto-fixable findings
- If the user says `none` → skip patching entirely, still generate the report

Store the user's selections as `approved_findings[]`.

### 4g: Patch

Follow `commands/gtm-wizard/patch-generator.md`.

Pass in:
- Original container model(s)
- `approved_findings[]` (only the findings the user approved — NOT the full findings list)
- Scaffold output from Step 4e

The patch-generator writes patched JSON files and returns `changelist[]` and `patched_files[]`.

If the user said `none` and there are no scaffolds, skip this step entirely and set `changelist = []` and `patched_files = []`.

Progress report:
> "Patched container written to {filename}"

### 4h: Report

Follow `commands/gtm-wizard/report-generator.md`.

Pass in:
- `findings[]`
- `research_summaries{}`
- `changelist[]`
- `detected_platforms[]`
- `added_platforms[]`
- `platform_module_versions{}` (version strings from each loaded platform module)
- `site_url`
- `container_ids[]`
- `patched_files[]`

The report-generator writes `gtm-wizard-report.md`.

Progress report:
> "Report written to gtm-wizard-report.md"

---

## Step 5: Complete

Print:

```
GTM Wizard complete!

Report:             gtm-wizard-report.md
Patched container:  {patched filename(s), one per line if multiple}

Next steps:
  1. Import the patched JSON into GTM:  Admin → Import Container → Merge
     (choose "Merge" and "Rename conflicting tags/triggers/variables")
  2. Test with Google Tag Assistant:
     - Install the Tag Assistant Companion extension if you don't have it
     - Open GTM → Preview → enter your site URL
     - Walk through your conversion flows (form submissions, purchases, etc.)
     - Verify each tag fires on the correct events with the correct data
  3. Check platform dashboards:
     - GA4: DebugView (Realtime → DebugView) — confirm events arrive
     - Google Ads: Conversions page — check for "Recording conversions" status
     - Meta: Events Manager → Test Events — verify pixel and CAPI events
     - TikTok: Events Manager → Test Events — verify pixel and Events API
  4. Publish when satisfied

Full details and action items are in gtm-wizard-report.md.
```
