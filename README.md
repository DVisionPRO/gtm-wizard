# GTM Wizard

**Your Google Tag Manager container expert — diagnose, fix, and build.**

GTM Wizard is a [Claude Code](https://docs.anthropic.com/en/docs/claude-code) plugin that analyzes Google Tag Manager container exports — including paired web + server containers — finds structural issues, applies platform-specific best practices, and can scaffold new platform implementations from scratch. It produces a detailed plain-language report and patched JSON files ready to import back into GTM.

> [!NOTE]
> **All data stays on your machine.** Container JSON is processed locally by Claude Code. Only web searches for current platform best practices leave your terminal.

> [!IMPORTANT]
> **Requires [Claude Code](https://docs.anthropic.com/en/docs/claude-code).** GTM Wizard runs as a slash command (`/gtm-wizard`) inside your terminal — it is not a standalone CLI tool.

---

## How It Works

Run `/gtm-wizard` and point it at your GTM container export(s). Drop in a single web container or a web + server pair — the wizard detects server-side tagging automatically and walks you through a guided flow:

1. **Parses** the container JSON into a normalized model
2. **Detects** which platforms are active (GA4, Google Ads, Meta, TikTok, LinkedIn)
3. **Runs structural checks** — duplicate events, orphaned tags, missing triggers, security issues
4. **Runs platform-specific checks** — configuration and best-practice validation per platform
5. **Researches** current platform documentation for up-to-date recommendations
6. **Scaffolds** tags for any new platforms you want to add
7. **Generates patched JSON** ready to import back into GTM
8. **Produces a plain-language report** summarizing findings, fixes, and next steps

```
GTM Export JSON
  → Container Parser → Platform Detector
    → Structural Checks + Platform Checks
      → Research Agent → Scaffold Generator
        → Patch Generator → Report
```

---

## What You Get

| Output | Description |
|---|---|
| `gtm-wizard-report.md` | Plain-language report with all findings, fixes applied, and recommendations |
| `*_patched.json` | Fixed container JSON — import via GTM's Admin → Import Container → Merge |
| Scaffolded tags | New platform tags, triggers, and variables if you requested build mode |

---

## Key Features

- **Diagnose + Fix + Build in one flow** — spot issues, apply fixes, and add missing platforms in a single session
- **Local and private** — everything runs in your terminal; container data never leaves your machine
- **Multi-platform detection** — automatically identifies which ad and analytics platforms are active in your container
- **Web + Server container analysis** — detects server-side tagging signals and prompts for the server container to analyze both together
- **Research-backed recommendations** — pulls current best practices from platform documentation at runtime
- **Security checks** — flags obfuscated or minified Custom HTML tags that could hide malicious code
- **Patched JSON output** — fixed container files ready to import via GTM's merge workflow

---

## Prerequisites

- **[Claude Code](https://docs.anthropic.com/en/docs/claude-code)** — CLI, desktop app, or IDE extension
- **A GTM container export** — in GTM, go to Admin → Export Container and download the JSON file

---

## Install

```bash
git clone https://github.com/DVisionPRO/gtm-wizard.git
cp -r gtm-wizard/.claude/commands/gtm-wizard ~/.claude/commands/
```

---

## Quick Start

1. Export your GTM container — Admin → Export Container in Google Tag Manager. If you use server-side tagging, export both your web and server containers.
2. Open your terminal in the directory containing the JSON export.
3. Run `/gtm-wizard` — the wizard will guide you through the rest:

```
/gtm-wizard
```

4. Review the generated report (`gtm-wizard-report.md`).
5. Import the patched JSON back into GTM via Admin → Import Container → Merge.

---

## Supported Platforms

| Platform | Module Version | Status |
|---|---|---|
| GA4 | 1.0.0 | Full (Diagnose + Fix + Build) |
| Google Ads | 1.0.0 | Full (Diagnose + Fix + Build) |
| Meta (Facebook / Instagram) | 1.0.0 | Full (Diagnose + Fix + Build) |
| TikTok | 0.9.0 | Beta |
| LinkedIn | 1.0.0 | Research only |

Want to add a platform? See [CONTRIBUTING.md](CONTRIBUTING.md).

---

## Privacy

GTM Wizard runs 100% locally inside Claude Code. Your container data is never uploaded, stored, or transmitted externally.

The only network activity is **web searches** used to fetch current platform best practices (e.g., "GA4 recommended events 2026"). These searches contain generic platform queries — never your container data, tag IDs, or domain names.

---

## Contributing

See [CONTRIBUTING.md](CONTRIBUTING.md) for how to add a new platform module or extend existing checks.

---

## Changelog

See [CHANGELOG.md](CHANGELOG.md) for release history.

---

## License

[MIT](LICENSE)
