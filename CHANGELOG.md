# Changelog

All notable changes to GTM Wizard will be documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.1.0/),
and this project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

## [1.0.0] — 2026-04-04

### Added
- Orchestrator entry point (`/gtm-wizard`) with guided Q&A flow
- Container parser with normalized container model
- Platform detector — automatic identification of GA4, Google Ads, Meta, TikTok, LinkedIn
- Structural checks — duplicate events, orphaned tags, missing triggers, paused tag analysis
- Security checks (SEC-1/SEC-2) — flags obfuscated and minified Custom HTML tags
- Platform-specific checks for GA4, Google Ads, and Meta
- Research agent — fetches current best practices from platform documentation
- Scaffold generator — builds new tag/trigger/variable sets for missing platforms
- Patch generator — produces importable `*_patched.json` with user approval before applying
- Report generator — plain-language `gtm-wizard-report.md` with findings, fixes, and recommendations
- Web + Server container analysis — detects server-side tagging and analyzes both containers together
- Platform modules: GA4 (1.0.0), Google Ads (1.0.0), Meta (1.0.0), TikTok (0.9.0 beta), LinkedIn (1.0.0 research-only)
- Sample GTM export (`examples/sample-web-export.json`) with intentional issues for testing
- Sample report output (`examples/sample-report.md`)
