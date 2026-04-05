# Contributing to GTM Wizard

Thank you for your interest in contributing. GTM Wizard is a Claude Code skill — all logic lives in markdown files that Claude interprets at runtime. No build step required.

## How to Add a New Platform Module

Create a single `.md` file in `.claude/commands/gtm-wizard/platforms/`. Name it after the platform: `tiktok.md`, `linkedin.md`, etc.

### Required Frontmatter

Every platform module must start with YAML frontmatter. All fields are required:

```yaml
---
platform: your_platform        # machine-readable identifier, snake_case
version: 1.0.0                 # semver
mode: full                     # "full" (diagnose+fix+build) or "research_only"
detect_types:                  # GTM tag type strings (if platform uses native tag types)
  - tag_type_string
detect_html:                   # HTML content substrings (if platform uses Custom HTML tags)
  - "platformSdkFunction("
events:                        # standard event names this platform tracks
  - PageView
  - Lead
---
```

At least one of `detect_types` or `detect_html` is required. Include both if the platform can be detected either way.

**Detection signatures:**

- `detect_types` — exact GTM tag type values (e.g. `gaawc` for GA4 Config, `awct` for Google Ads). Find these in GTM export JSON under `tag[].type`.
- `detect_html` — substrings to look for in Custom HTML tag content (e.g. `fbq(` for Meta Pixel, `ttq.track(` for TikTok). The platform-detector will match these against `tag[].parameter[key=html].value`.

### Required Sections

Your module must include all four sections:

1. **`## Platform-Specific Checks`** — Define each check with a unique ID (`PLATFORM-N`), severity, what triggers it, and the auto-fix to apply (or mark as manual).

2. **`## Event Mapping`** — Table mapping dataLayer event names (e.g. `generate_lead`) to platform event names (e.g. `Lead`). Used by scaffold-generator to match events.

3. **`## Scaffold Templates`** — JSON-like templates for tags, triggers, and variables the wizard adds when building a new implementation. Use `REPLACE_WITH_PLATFORM_ID` style placeholders for values the user must supply.

4. **`## Best Practice Notes`** — Key recommendations used to populate the "Best Practices Review" section of the report. Keep this focused: two or three actionable items per section is better than an exhaustive list.

## Testing Your Module

1. Run `/gtm-wizard` and point it at `examples/sample-web-export.json`.
2. When asked which platforms to add, choose your new platform.
3. Verify the output report shows correct scaffolding and no errors.
4. If your platform should also detect issues in an existing container, add relevant tags to `examples/sample-web-export.json` (synthetic data only) and verify your checks fire.

## PR Checklist

Before opening a pull request:

- [ ] No real credentials, pixel IDs, conversion labels, or domain names in any file
- [ ] Frontmatter includes all required fields
- [ ] All four required sections are present in the module
- [ ] Module version bumped if adding new checks to an existing module
- [ ] `examples/sample-report.md` updated if your changes produce new findings in the sample export
- [ ] Tested against `examples/sample-web-export.json` and output looks correct

## Questions

Open an issue describing what platform or check you want to add and we'll help you get started.
