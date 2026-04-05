---
platform: linkedin
version: 1.0.0
mode: research_only
detect_html: ['lintrk(', 'snap.licdn.com']
events: [Lead, Purchase, AddToCart, ViewContent, SignUp, CompleteRegistration, Download]
---

# LinkedIn Platform Module

> **Research Only** — Detection of LinkedIn Insight Tag and conversion tags, the LI-1 structural check, and best-practice guidance via research agent. No scaffold JSON patch output is generated.

## Platform-Specific Checks

### LI-1: Broad Insight Tag Without Conversion Tracking
- **Severity:** WARN
- **Condition:** An Insight Tag (`snap.licdn.com/scdn/n/en_US/fc/ads/scripts/retarget.js` or inline `lintrk` initialization code) fires on the All Pages trigger (2147479573) AND no event-specific conversion tags exist (no `lintrk('track', {conversion_id: ...})` calls found in any custom HTML tag in the container).
- **Explanation:** The LinkedIn Insight Tag firing on all pages is correct behavior — this is by design and enables retargeting audiences, demographic reporting, and website analytics in Campaign Manager. This check does NOT flag the Insight Tag as misconfigured. It flags the **absence of event-specific conversion tracking** alongside the Insight Tag. Without conversion tags, Campaign Manager cannot report on specific actions (leads, purchases) — only traffic and retargeting pools are available.
- **Fix:** Add event-specific conversion tags using `lintrk('track', {conversion_id: XXXXXXX})` for each conversion action. Conversion IDs are created in LinkedIn Campaign Manager > Analyze > Conversion Tracking. Each conversion action in Campaign Manager generates a unique numeric ID.

## Event Mapping

LinkedIn conversion event names are defined in Campaign Manager — they are not a fixed enum like Meta or TikTok. The tag call itself is always `lintrk('track', {conversion_id: XXXXXXX})` where the ID references the Campaign Manager conversion definition.

| dataLayer event | LinkedIn conversion type (Campaign Manager) | Notes |
|---|---|---|
| generate_lead | Lead | Most common B2B conversion |
| purchase | Purchase | Pass value if available (not supported natively — use insight tag URL rules instead) |
| add_to_cart | Add to Cart | Less common on LinkedIn — typically B2B |
| view_item | View Content | |
| sign_up | Sign Up | |
| sign_up (post-verification) | Complete Registration | Distinguish from initial sign_up if two-step registration |
| file download event | Download | Track whitepapers, case studies, etc. |

## Best Practice Notes

- **Insight Tag on all pages is correct:** Do not remove or restrict the LinkedIn Insight Tag to specific pages. It must fire on all pages to build retargeting audiences, populate the website demographics report, and enable LinkedIn's conversion attribution window. A check that flags the Insight Tag as firing on All Pages as an error is incorrect.
- **Add event-specific conversion tags for actionable data:** The Insight Tag alone only gives you traffic. To see which campaigns generate leads or purchases, you need `lintrk('track', {conversion_id: XXXXXXX})` tags on conversion confirmation pages or dataLayer events. Create conversion actions in Campaign Manager first, then implement the tag.
- **LinkedIn CAPI (Conversions API):** LinkedIn offers a server-side Conversions API for sending conversion events server-to-server. Available via direct API or via partner integrations. Recommended for improving measurement completeness on ITP-affected Safari/iOS traffic. As of v1.0, LinkedIn CAPI setup requires the research agent — no automated scaffold is available.
- **Insight Tag enables retargeting even without conversions:** Even if no conversion tags are implemented, the Insight Tag enables audience creation for retargeting (website visitors, page visitors, etc.) and provides website demographic data (job title, company, industry). This has standalone value independent of conversion tracking.
- **Verify in Campaign Manager:** After implementing conversion tags, verify firing in LinkedIn Campaign Manager > Analyze > Insight Tag. Use the "Pixel Helper" Chrome extension (LinkedIn Insight Tag Helper) for real-time tag verification during QA.
- **Conversion attribution windows:** LinkedIn default attribution is 30-day post-click and 7-day post-view. For long B2B sales cycles, consider extending the post-click window to 90 days in Campaign Manager conversion settings.
- **URL-based conversions as alternative:** For simple thank-you page conversions, LinkedIn Campaign Manager supports URL-based conversion rules (no additional tag required beyond the Insight Tag). Consider this approach when GTM is not available or custom HTML tags are restricted.
