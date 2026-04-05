# GTM Wizard Report
**Site:** https://www.example.com | **Date:** 2026-03-18 | **Wizard version:** 1.0.0

---

## Platform Coverage

| Platform | Module Version | Mode | Checks Run | Issues Found |
|---|---|---|---|---|
| GA4 | 1.0.0 | Diagnose + Fix | 4 | 2 |
| Google Ads | 1.0.0 | Diagnose + Fix | 2 | 1 |
| Meta | 1.0.0 | Diagnose + Fix | 3 | 1 |

---

## Summary

**7 issues found** — 2 HIGH, 2 MEDIUM, 1 WARNING, 1 WARN, 1 INFO
**3 fixes applied** in patched JSON | **0 tags scaffolded** for new platforms

---

## Structural Issues

### Event Duplication

**[HIGH]** GA4 is sending the page view event twice. The Configuration tag already sends `page_view` automatically when it loads, but a separate "GA4 - Page View Event" tag also fires it on every page. This double-counts page views in your GA4 reports and inflates session counts.
_Affected: "GA4 - Configuration", "GA4 - Page View Event"_
Fixed automatically in patched container. The `eventName` parameter was removed from "GA4 - Page View Event" so it no longer sends a duplicate page_view.
_Reference: DUP-1_

### Tag Efficiency

**[HIGH]** There are two identical GA4 Configuration tags sending data to the same Measurement ID (G-EXAMPLEXX1). This initializes the GA4 tracker twice on every page, causing duplicated events across all your GA4 reporting.
_Affected: "GA4 - Configuration", "GA4 - Configuration (duplicate)"_
Fixed automatically in patched container. "GA4 - Configuration (duplicate)" has been paused.
_Reference: EFF-1_

**[MEDIUM]** The Google Ads conversion tag fires on every page instead of only when a lead is generated. Conversion tags should fire on specific conversion events — not on All Pages — otherwise Google Ads cannot accurately attribute leads to ad clicks.
_Affected: "Google Ads - Lead Conversion"_
**Manual fix required:**
1. In GTM, open the "Google Ads - Lead Conversion" tag.
2. Remove the "All Pages" trigger.
3. Add the "Custom Event - generate\_lead" trigger (or whichever trigger fires when a lead form is submitted).
4. Save and publish.
_Reference: EFF-2_

**[WARN]** One paused tag was found: "Old Facebook Pixel - Deprecated". Paused tags remain in the container and can cause confusion. If this pixel has been replaced, consider deleting the tag entirely to keep the container clean.
_Affected: "Old Facebook Pixel - Deprecated"_
No fix applied — deletion requires manual review to confirm the tag is no longer needed.
_Reference: EFF-3_

### Security / Auditability

**[WARN]** The "Suspicious Tracker" tag contains potentially obfuscated code. It uses `String.fromCharCode()` to build a URL string, which is a common technique for hiding the true destination of data collection. Review this tag manually to verify it behaves as expected and is authorized.
_Affected: "Suspicious Tracker"_
**Manual fix required:** Inspect the tag's JavaScript to determine what URL it constructs and where it sends data. If the tag is not recognized or authorized, remove it immediately.
_Reference: SEC-1_

**[INFO]** The "Vendor Analytics SDK" tag contains minified JavaScript (672 characters on a single line) that is difficult to audit. The code is likely a third-party vendor SDK, but its behavior cannot be verified without de-minification.
_Affected: "Vendor Analytics SDK"_
No fix applied — review the vendor documentation to confirm the tag's purpose and data collection scope.
_Reference: SEC-2_

---

## Platform: Meta — Existing Setup

### Issues Found

**[HIGH]** The Meta Lead event is missing an `event_id` parameter. Without a stable `event_id`, Meta cannot deduplicate browser pixel events against Conversions API (CAPI) events. This leads to double-counted leads in Meta Ads Manager and lowers your Event Match Quality (EMQ) score, which reduces targeting accuracy.
_Affected: "Meta Pixel - Lead"_
Fixed automatically in patched container. An `eventID` parameter has been added to the `fbq('track', 'Lead', ...)` call using a timestamp-based unique ID. If you are also sending this event via CAPI, ensure the same `event_id` value is forwarded server-side.
_Reference: META-1_

### Best Practices Review

**What's configured correctly:**
- Meta Base Pixel tag is present and fires on All Pages, initializing the pixel and tracking PageView.
- The Lead event tag fires on a specific custom event trigger (`generate_lead`) rather than All Pages — correct scoping.
- The pixel ID is consistent between the Base tag and Lead tag.

**What could be improved:**
- Add `advanced_matching` parameters (hashed email, phone) to the `fbq('init', ...)` call in "Meta Pixel - Base" to improve EMQ.
- Consider moving pixel calls to a server-side GTM setup (CAPI) to improve reliability as browser-based tracking is affected by ad blockers and iOS privacy restrictions.
- Add a Content Security Policy (CSP) nonce or move to a first-party domain proxy for the fbevents.js script.

---

## Platform: GA4 — Existing Setup

### Issues Found

Issues are listed in the Structural Issues section above (DUP-1, EFF-1).

### Best Practices Review

**What's configured correctly:**
- GA4 Configuration tag is present and correctly references the Measurement ID.
- `sendPageView` is enabled on the Configuration tag, which is the recommended approach for page view tracking.

**What could be improved:**
- Add a Consent Mode v2 initialization block before the GA4 Configuration tag to comply with EU consent requirements.
- Consider enabling Google Signals in the GA4 property settings for cross-device reporting.

---

## Platform: Google Ads — Existing Setup

### Issues Found

Issues are listed in the Structural Issues section above (EFF-2).

### Best Practices Review

**What's configured correctly:**
- The Google Ads conversion tag includes both `conversionId` and `conversionLabel`, which are required fields.

**What could be improved:**
- After fixing the trigger to fire only on `generate_lead`, add a conversion value and currency to the tag parameters for value-based bidding.
- Enable "Enhanced Conversions" in Google Ads and pass hashed first-party data (email or phone) from the form submission to improve match rates.

---

## All Changes Applied

| # | Container | Type | Target | Check | Description |
|---|---|---|---|---|---|
| 1 | GTM-EXAMPLE | pause | GA4 - Page View Event | DUP-1 | Paused duplicate page_view event tag (Config tag already fires page_view automatically) |
| 2 | GTM-EXAMPLE | pause | GA4 - Configuration (duplicate) | EFF-1 | Paused redundant GA4 Configuration tag with duplicate Measurement ID |
| 3 | GTM-EXAMPLE | modify | Meta Pixel - Lead | META-1 | Added `eventID` timestamp parameter to fbq Lead call for CAPI deduplication |

---

## Next Steps

- [ ] Import `GTM-EXAMPLE-patched.json` into GTM (Admin → Import Container → Merge)
- [ ] Verify changes in GTM Preview mode
- [ ] Apply manual fix for EFF-2: narrow the "Google Ads - Lead Conversion" trigger to `generate_lead` only (see instructions above)
- [ ] Review "Old Facebook Pixel - Deprecated" and delete it if no longer needed
- [ ] Inspect "Suspicious Tracker" tag — verify it is authorized and understand where it sends data
- [ ] Review "Vendor Analytics SDK" tag — confirm the vendor and its data collection scope
- [ ] Test conversion flows end-to-end — submit a test lead form and confirm GA4, Google Ads, and Meta each record exactly one event
- [ ] Publish GTM container
