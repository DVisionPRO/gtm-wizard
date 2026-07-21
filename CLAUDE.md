# CLAUDE.md

Conventions for AI contributors working on this repository.

## Structure

- All skill logic lives in `.claude/commands/gtm-wizard/` as markdown files.
- Platform modules live in `.claude/commands/gtm-wizard/platforms/` — one `.md` file per platform.
- The orchestrator entry point is `.claude/commands/gtm-wizard/gtm-wizard.md`.

## Platform Module Conventions

- Every platform module must include YAML frontmatter with the following fields:
  - `platform` — machine-readable platform identifier, snake_case (e.g. `ga4`, `meta`, `google_ads`)
  - `version` — semver string (e.g. `1.0.0`)
  - `mode` — `full` (diagnose + fix + build) or `research_only`
  - `detect_types` — list of GTM tag type strings that identify this platform (e.g. `gaawc`, `gaawe`). Required if the platform uses native GTM tag types.
  - `detect_html` — list of HTML content substrings that identify this platform in Custom HTML tags (e.g. `fbq(`, `gtag(`). Required if the platform uses Custom HTML tags.
  - `events` — list of standard event names this platform tracks
  - At least one of `detect_types` or `detect_html` must be present.

- Required sections in every platform module:
  1. `## Platform-Specific Checks` — findings with IDs, severity, and auto-fix instructions
  2. `## Event Mapping` — dataLayer event names mapped to platform event names
  3. `## Scaffold Templates` — tag/trigger/variable JSON templates for build mode
  4. `## Best Practice Notes` — research reference used for best practices review

## Normalized Container Model

The normalized container model format is defined in `container-parser.md`. All modules must read from and write to this model exclusively — never parse raw GTM export JSON directly inside a platform module.

## Findings Format

All findings must conform to this shape:

```
{
  id: string,           // e.g. "META-1"
  severity: "HIGH" | "MEDIUM" | "WARN" | "INFO",
  platform: string,     // matches frontmatter `platform` field, or "structural"
  affected_tags: string[],
  description: string,
  fix: string | null    // null if manual fix required
}
```

Use this shape consistently across all checks. Do not add extra fields.

## Sample Data

- `examples/sample-web-export.json` is a synthetic GTM export with intentional issues — it contains no real client data.
- `examples/sample-report.md` shows the expected report output when running gtm-wizard against the sample export.
- Test all changes against `examples/sample-web-export.json` before submitting.

## Content Rules

- No client data, credentials, real pixel IDs, real conversion labels, or real domain names in any file.
- The sample export uses only placeholder IDs (GTM-EXAMPLE, G-EXAMPLEXX1, AW-0000000000, 000000000000000).
- Keep it that way.

## Tag Naming Convention

Scaffolded tags must follow this naming pattern:

```
[GTM Wizard] {Platform} - {Event}
```

Examples:
- `[GTM Wizard] Meta - Lead`
- `[GTM Wizard] TikTok - Purchase`
- `[GTM Wizard] LinkedIn - Page View`

Scaffolded triggers use:
```
[GTM Wizard] Custom Event - {event_name}
```

Scaffolded variables use:
```
[GTM Wizard] DLV - {variable_name}
```

## Response style

- **Lead with the action or answer on line 1** -- no preamble, no restating the plan before doing it.
- **Number multi-step work** when the steps are actions -- one bounded action per step, no "and then" compounds.
- **Finish before branching** -- resolve the request fully, then raise any secondary issue once, at the end, as a separate line.
- **End when done** -- no closing pleasantries or "let me know if..." sign-offs.
- **State errors matter-of-factly** -- name the cause and the fix directly; skip softening adverbs.
