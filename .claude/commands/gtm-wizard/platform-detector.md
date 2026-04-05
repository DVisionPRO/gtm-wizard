# GTM Wizard — Platform Detector Module

## Purpose
Detect which advertising platforms are present in the container by inspecting tag types and Custom HTML content. Return the platform list for downstream modules.

## Input
- Normalized container model (from container-parser)
- `added_platforms`: list of platform names the user requested to add (from orchestrator)

## Output
```json
{
  "detected": ["GA4", "Google Ads", "Meta"],
  "added": ["TikTok"],
  "sgtm_available": true,
  "all": ["GA4", "Google Ads", "Meta", "TikTok"]
}
```

## Detection Rules

Iterate over all tags in the container model. For each tag:

**By tag type:**
- `gaawc` or `gaawe` → GA4 detected
- `awct` or `gclidw` → Google Ads detected

**By rawHtml content (for `type === "html"` tags):**
- Contains `fbq(` OR `connect.facebook.net` OR `facebook.com/tr` → Meta detected
- Contains `ttq.track` OR `analytics.tiktok.com` → TikTok detected
- Contains `lintrk(` OR `snap.licdn.com` → LinkedIn detected
- Contains `twq(` OR `static.ads-twitter.com` → Twitter/X detected
- Contains `snaptr(` OR `tr.snapchat.com` → Snapchat detected
- Contains `pintrk(` OR `ct.pinterest.com` → Pinterest detected
- Contains `uetq` OR `bat.bing.com` → Microsoft/Bing Ads detected

**Server-side detection:**
- If a server container model is provided AND its `clients[]` array contains domains → `sgtm_available: true`
- If server container has tags with type matching Meta CAPI or TikTok Events API patterns → note in detection

## Steps

1. Iterate all tags in the container model, apply rules above. Collect unique platform names in `detected[]`.
2. Append `added_platforms` to `added[]` (deduplicate against `detected[]`).
3. Set `sgtm_available` based on server container presence.
4. Compute `all = [...detected, ...added]` (deduplicated).
5. Report to user: "Detected platforms: {detected.join(', ')}."
   If `added` is non-empty: "Platforms to build: {added.join(', ')}."
