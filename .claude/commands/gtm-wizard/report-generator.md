# Report Generator

Compile all findings, research summaries, and changelist into `gtm-wizard-report.md`. Written for non-technical users: plain language, no jargon, clear next steps.

## Inputs

- `findings[]` — full findings list from structural-checks and platform module checks
- `research_summaries{}` — per-platform research results from research-agent
- `changelist[]` — all changes from patch-generator
- `detected_platforms[]` — platforms found in the container
- `added_platforms[]` — platforms added by the user for scaffolding
- `platform_module_versions{}` — version string per platform module
- `site_url` — website URL from user input
- `container_ids[]` — GTM container IDs parsed from export files
- `patched_files[]` — filenames written by patch-generator

## Output

A single markdown file `gtm-wizard-report.md` written to the current working directory.

## Steps

### Step 1: Build Header

Write the report header using site URL, current ISO date, and wizard version:

```markdown
# GTM Wizard Report
**Site:** {site_url} | **Date:** {ISO date} | **Wizard version:** 1.0.0
```

If `site_url` is not set, write `(not provided)` in place of the URL.

---

### Step 2: Build Platform Coverage Table

For each platform in `detected_platforms` union `added_platforms`:

- **Module Version:** look up `platform_module_versions[platform]`
- **Mode:**
  - If platform is in both `detected_platforms` and `added_platforms` → `Diagnose + Fix + Build`
  - If platform is in `detected_platforms` only → `Diagnose + Fix`
  - If platform is in `added_platforms` only → `Build only`
  - If the platform has no checks defined (research-only) → `Research only`
- **Checks Run:** count findings where `finding.platform === platform`
- **Issues Found:** count findings where `finding.platform === platform` and `finding.severity` is HIGH, MEDIUM, or WARN

Write:

```markdown
## Platform Coverage
| Platform | Module Version | Mode | Checks Run | Issues Found |
|---|---|---|---|---|
| {platform display name} | {version} | {mode} | {N} | {M} |
```

---

### Step 3: Compute Summary Statistics

From `findings[]`:
- `total_issues` = total count of all findings
- `high` = count where `severity === "HIGH"`
- `medium` = count where `severity === "MEDIUM"`
- `warn` = count where `severity === "WARN"`

From `changelist[]`:
- `fixes_applied` = count of entries where `type` is `"modify"` or `"pause"` — i.e. changes that fix an existing issue
- `tags_scaffolded` = count of entries where `type` is `"add"` and `scaffold === true`

Write:

```markdown
## Summary
**{total_issues} issues found** — {high} HIGH, {medium} MEDIUM, {warn} WARNINGS
**{fixes_applied} fixes applied** in patched JSON | **{tags_scaffolded} tags scaffolded** for new platforms
```

---

### Step 4: Format Structural Issues Section

Group structural findings by category. Only write a category heading if there is at least one finding in that group.

**Categories and plain-language framing:**

| Check ID prefix | Section heading | Plain-language framing |
|---|---|---|
| DUP-* | Event Duplication | Explain that the same event is being tracked more than once, which inflates conversion counts |
| EFF-* | Tag Efficiency | Explain that tags are firing more often than needed, wasting budget or skewing data |
| SEC-* | Security / Auditability | Flag tags with suspicious or hard-to-audit code patterns that should be reviewed manually |
| SRV-* | Server Container | Explain server container configuration issues that may prevent events from routing correctly |

For each finding in a category:

```markdown
### {Section heading}

**[{SEVERITY}]** {plain-language description}
_Affected: {tag names or trigger names}_
{If auto-fixed: "Fixed automatically in patched container."}
{If manual: "**Manual fix required:** {step-by-step instructions}"}
_Reference: {check_id}_
```

**Plain-language description rules:**
- DUP findings: "GA4 is sending the [event name] event twice — once from the Config tag and once from an Event tag. This double-counts the conversion."
- EFF findings: "The [tag name] tag fires on every page, but it only needs to fire on [specific pages]. Consider restricting it to [trigger condition]."
- SEC findings: "The [tag name] tag contains [obfuscated code / minified JavaScript] that is difficult to review. Inspect it manually to confirm it behaves as expected."
- SRV findings: "The server container client is configured for [domain], but your site runs on [site_url]. Events may not route to the server container."

If there are no structural findings, write:

```markdown
## Structural Issues
No structural issues found.
```

---

### Step 5: Format Per-Platform Sections — Existing Platforms

For each platform in `detected_platforms`:

```markdown
## Platform: {Platform display name} — Existing Setup
```

#### 5a: Issues Found

List each finding for this platform:

```markdown
### Issues Found

**[{SEVERITY}]** {plain-language description of the issue}
_Affected: {tag or trigger names}_
{If auto-fixed: "Fixed automatically in patched container."}
{If manual: "**Manual fix required:**"}
{  Step-by-step instructions for manual fixes, numbered list}
_Reference: {check_id}_
```

If no platform-specific issues found, write: `No issues found for this platform.`

#### 5b: Best Practices Review

Use `research_summaries[platform]` to compare current container setup against recommended best practices.

Write:

```markdown
### Best Practices Review

**What's configured correctly:**
{Bulleted list of things the container does that match best practices}

**What could be improved:**
{Bulleted list of gaps or missing recommended configurations, in plain language}
{If nothing to improve: "Everything looks aligned with current best practices."}
```

If no research summary is available, write: `Research summary not available for this platform.`

---

### Step 6: Format Per-Platform Sections — New Platforms (Build)

For each platform in `added_platforms` (that is not also in `detected_platforms`):

```markdown
## Platform: {Platform display name} — New Implementation

> **Important:** Placeholder values were inserted during scaffolding. You must replace them before publishing. See "Complete Your Setup" below.
```

#### 6a: Scaffolded Setup

From `changelist[]`, filter entries where `scaffold === true` and `platform === this platform`.

```markdown
### Scaffolded Setup

The following tags, triggers, and variables were added to your container:

| Type | Name | Purpose |
|---|---|---|
| Tag | {tag name} | {one-line purpose} |
| Trigger | {trigger name} | {one-line purpose} |
| Variable | {variable name} | {one-line purpose} |
```

#### 6b: Event Mapping

If the scaffold-generator produced an event mapping (events from `conversion_events[]` matched to platform events):

```markdown
### Event Mapping Applied

| Your dataLayer Event | {Platform} Event | Tag Created |
|---|---|---|
| {dataLayer event name} | {platform event name} | {tag name} |
```

If no mapping was produced, omit this subsection.

#### 6c: Complete Your Setup

```markdown
### Important: Complete Your Setup

Before publishing, you must:

1. **Replace placeholder values** — open the patched JSON in GTM and search for `REPLACE_WITH_`. Each placeholder has a descriptive name (e.g. `REPLACE_WITH_PIXEL_ID`).
2. **Test in GTM Preview mode** — verify each tag fires on the correct events.
3. **Check consent settings** — confirm tags respect your consent management setup.
4. {Platform-specific instructions from the platform module's `build_notes` field, if present}
```

---

### Step 7: Build Changes Applied Table

From `changelist[]`, write the full list of all changes. Add an **Importance** column that categorizes each change:

- **Critical** — Fixes that affect data accuracy (duplicate conversions, missing attribution, inflated counts). These must be applied before publishing.
- **Recommended** — Fixes that improve efficiency or follow best practices but won't cause incorrect data if skipped.
- **Informational** — Observations that require human judgment (e.g., paused tags, asymmetric triggers).

Map severity to importance: HIGH → Critical, MEDIUM → Recommended, WARN → Informational. Scaffold additions are always Recommended.

```markdown
## All Changes Applied

| # | Importance | Container | Type | Target | Description |
|---|---|---|---|---|---|
| {index} | {importance} | {container_id} | {type} | {target name} | {description} |
```

After the table, add a brief explanation of what the patched container file contains:

```markdown
**What the patched file includes:** All changes marked "modify" or "pause" above have been applied to `{patched_filename}`. Changes marked "manual" require your action in GTM — they are noted in the report but not applied automatically because they depend on your specific setup.
```

If `changelist` is empty, write: `No changes were applied.`

---

### Step 8: Build Next Steps Checklist

Collect placeholder variable names from `changelist[]` — any variable name or value containing `REPLACE_WITH_`.

```markdown
## Next Steps

- [ ] Import `{patched_file}` into GTM (Admin → Import Container → Merge)
{If multiple patched files, one line per file}
- [ ] Verify changes in GTM Preview mode
{If any REPLACE_WITH_ placeholders exist:}
- [ ] Replace placeholder values in GTM:
  - `{REPLACE_WITH_PIXEL_ID}` → your {platform} Pixel ID
  - `{REPLACE_WITH_API_TOKEN}` → your {platform} API access token
  {... one line per unique placeholder found}
- [ ] Test conversion flows end-to-end
- [ ] Publish GTM container
{If any manual fixes were flagged:}
- [ ] Apply manual fixes noted above before publishing
```

---

### Step 9: Write the Report

Use the Write tool to write the complete assembled markdown to `gtm-wizard-report.md` in the current working directory.

Print: `Report written to gtm-wizard-report.md`

## Tone Guidelines

- **Plain language first.** Translate check IDs into sentences a non-technical user can act on. Write "GA4 is sending the page view twice" not "DUP-1: Config + page_view overlap detected".
- **Severity badges** use text markers: `[HIGH]`, `[MEDIUM]`, `[WARN]`. Do not use emoji.
- **Check IDs** appear only in the small `_Reference: DUP-1_` line at the end of each finding block — never as a headline.
- **Manual fixes** are always labeled "Manual fix required" and include numbered steps. Never leave the user guessing what to do.
- **New builds** always include the "Important: Complete Your Setup" callout. Assume the user has never edited a GTM container before.
- **Research comparisons** focus on actionable gaps, not exhaustive lists of features. Two or three concrete improvements are more useful than ten vague ones.
- **Encouraging framing:** if there are no issues in a section, say so explicitly and positively ("No issues found — this looks good.").
