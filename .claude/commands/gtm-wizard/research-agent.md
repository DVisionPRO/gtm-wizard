# GTM Wizard — Research Agent Module

## Purpose
For each platform in play, check a local research cache. If the cache entry is missing or stale, run WebSearch to fetch current best practices and save results to cache. Return cached or fresh research content per platform for use by downstream modules.

## Input
- `all`: the full platform list from platform-detector output
- `mode`: `"diagnose"` or `"build"` (from orchestrator)
- `cache_dir`: path to the research cache directory (default: `.gtm-wizard-research/` relative to cwd)

## Output
A map of platform → research content (markdown string), sourced from cache or freshly fetched.

---

## Cache Structure

```
.gtm-wizard-research/
├── google-ga4.md
├── google-ads.md
├── meta.md
├── tiktok.md
├── linkedin.md
└── .cache-meta.json
```

### .cache-meta.json format
```json
{
  "google-ga4": { "fetched": "2026-03-18T14:00:00Z", "ttl_days": 7 },
  "google-ads": { "fetched": "2026-03-18T14:00:00Z", "ttl_days": 7 },
  "meta": { "fetched": "2026-03-18T14:00:00Z", "ttl_days": 7 },
  "tiktok": { "fetched": "2026-03-18T14:00:00Z", "ttl_days": 7 },
  "linkedin": { "fetched": "2026-03-18T14:00:00Z", "ttl_days": 7 }
}
```

---

## Platform → Filename Mapping

| Platform | Cache filename |
|---|---|
| GA4 | google-ga4.md |
| Google Ads | google-ads.md |
| Meta | meta.md |
| TikTok | tiktok.md |
| LinkedIn | linkedin.md |
| Twitter/X | twitter-x.md |
| Snapchat | snapchat.md |
| Pinterest | pinterest.md |
| Microsoft/Bing Ads | microsoft-bing.md |

---

## TTL Check

1. Load `.cache-meta.json` (if it exists; if missing, treat all entries as absent).
2. For each platform in `all`:
   - If no entry in `.cache-meta.json` → cache is **missing**.
   - If entry exists: compute `age_days = (now - fetched) / 86400`. If `age_days > ttl_days` → cache is **stale**.
   - Otherwise → cache is **fresh**.
3. Default TTL: 7 days.

---

## On Stale Cache

Ask the user before refreshing:

> "Research cache for {Platform} is {N} days old (fetched {fetched_date}). Refresh? [y/N]"

- If user answers `y` or `yes` → fetch fresh data.
- If user answers `n`, `N`, or presses Enter → use the stale cached content as-is.
- If cache is **missing** → fetch without asking.

---

## Search Queries per Platform

| Platform | Mode | Queries |
|---|---|---|
| GA4 | diagnose | "GA4 GTM server-side tagging best practices {year}", "GA4 common GTM configuration mistakes" |
| GA4 | build | "GA4 GTM implementation guide {year}", "GA4 measurement protocol recommended events" |
| Google Ads | diagnose | "Google Ads conversion tracking GTM best practices {year}" |
| Google Ads | build | "Google Ads conversion tracking GTM setup guide {year}" |
| Meta | diagnose | "Meta Pixel CAPI GTM server-side best practices {year}", "Meta event match quality GTM improvements" |
| Meta | build | "Meta Pixel CAPI GTM implementation guide {year}", "Meta Conversions API event deduplication setup" |
| TikTok | diagnose | "TikTok pixel Events API GTM best practices {year}" |
| TikTok | build | "TikTok pixel GTM implementation guide {year}", "TikTok Events API server-side tagging setup" |
| LinkedIn | any | "LinkedIn Insight Tag GTM best practices {year}", "LinkedIn conversion tracking GTM setup" |
| Twitter/X | any | "X Ads pixel GTM best practices {year}", "Twitter Ads conversion tracking GTM setup" |
| Snapchat | any | "Snapchat Pixel GTM best practices {year}", "Snapchat Conversions API GTM setup" |
| Pinterest | any | "Pinterest Tag GTM best practices {year}", "Pinterest Conversions API GTM setup" |
| Microsoft/Bing Ads | any | "Microsoft Advertising UET tag GTM best practices {year}", "Bing Ads conversion tracking GTM setup" |

Substitute `{year}` with the current year (from today's date).

---

## Synthesized Output Format per Platform

Each cached file must follow this structure:

```markdown
# {Platform} Best Practices — Research Cache
**Fetched:** {ISO date} | **Mode:** diagnose|build

## Official Documentation Summary
(key points extracted from official documentation sources)

## Common Issues & Fixes
(top issues with their remediation steps)

## Implementation Recommendations
(best-practice setup steps for this platform in GTM)

## Sources
(list of URLs from search results used to build this summary)
```

---

## Steps

1. Use ToolSearch to fetch the WebSearch tool definition before running any searches.
2. Load `.cache-meta.json` from `cache_dir`. If the file does not exist, initialize an empty object.
3. For each platform in the `all` list (process sequentially):
   a. Resolve the cache filename using the mapping table above.
   b. Check TTL status (missing / stale / fresh).
   c. If **fresh**: read the cached `.md` file using the Read tool. Use its content.
   d. If **stale**: ask the user. If user declines refresh, read cached content. If user approves → proceed to fetch.
   e. If **missing** or fetch approved:
      - Select the queries for this platform and mode from the table above.
      - Run WebSearch for each query. Collect all results (titles, URLs, snippets).
      - Synthesize results into the structured markdown format above.
      - Write the synthesized content to `cache_dir/{filename}` using the Write tool.
      - Update `.cache-meta.json` entry: set `fetched` to current ISO timestamp, `ttl_days` to 7.
      - Write the updated `.cache-meta.json` using the Write tool.
4. After processing all platforms, write the final `.cache-meta.json` once more to ensure all updates are persisted.
5. Return a map of `{ platform: content_string }` for each platform in `all`.
6. Report: "Research ready for: {all.join(', ')}." Note any platforms served from cache vs. freshly fetched.
