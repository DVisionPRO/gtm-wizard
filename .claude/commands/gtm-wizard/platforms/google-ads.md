---
platform: google_ads
version: 1.0.0
mode: full
detect_types: [awct, gclidw]
events: [purchase, generate_lead, add_to_cart, begin_checkout, sign_up]
---

# Google Ads Platform Module

## Platform-Specific Checks

### GADS-1: Missing Conversion Linker
- **Severity:** HIGH
- **Condition:** An `awct` (Google Ads Conversion Tracking) tag exists in the container but no `gclidw` (Conversion Linker) tag is present.
- **Explanation:** The Conversion Linker reads the `gclid` parameter from the URL and stores it in a first-party cookie. Without it, conversion tags cannot attribute the conversion back to the originating Google Ads click — all conversions appear unattributed.
- **Fix:** Add a `gclidw` Conversion Linker tag firing on All Pages trigger (2147479573). Must load before any `awct` tags on the same page.

### GADS-2: Conversion Tag Firing on Broad Trigger
- **Severity:** MEDIUM
- **Condition:** An `awct` tag fires on a page-view-on-every-page trigger instead of a specific conversion event trigger. A trigger qualifies as page-view-on-every-page if it is the built-in All Pages trigger (`2147479573`) OR a custom trigger with `type === 'PAGEVIEW'` that has no restrictive filter conditions.
- **Explanation:** Firing a conversion tag on every page view inflates conversion counts and misreports every pageview as a conversion in Google Ads. This corrupts campaign optimization signals.
- **Fix:** Restrict the `awct` tag to fire only on the specific conversion event trigger (e.g., form submit confirmation, thank-you page, purchase dataLayer event). This is the same issue as EFF-2 (broad trigger inefficiency) surfaced with Google Ads-specific impact.

## Event Mapping

| dataLayer event | Google Ads conversion name | Notes |
|---|---|---|
| generate_lead | Lead | Count: One (avoid counting multiple form resubmissions) |
| purchase | Purchase | Count: Every (count each transaction). Pass `value` and `currency`. |
| add_to_cart | Add to Cart | Count: Every |
| begin_checkout | Begin Checkout | Count: Every |
| sign_up | Sign Up | Count: One |

**Conversion ID format:** Google Ads conversion IDs appear as `AW-XXXXXXXXXX/YYYYYYYYYYYY` in the UI. The numeric part after `AW-` is the `conversionId`; the part after `/` is the `conversionLabel`.

## Scaffold Templates

### 1. Conversion Linker Tag (gclidw)

```json
{
  "type": "gclidw",
  "name": "[GTM Wizard] Google Ads - Conversion Linker",
  "parameter": [
    {
      "type": "boolean",
      "key": "enableCrossDomain",
      "value": "false"
    },
    {
      "type": "boolean",
      "key": "rewriteDomains",
      "value": "false"
    }
  ],
  "firingTriggerId": ["2147479573"],
  "tagFiringOption": "oncePerLoad"
}
```

### 2. Conversion Tracking Tag (awct) — per conversion event

```json
{
  "type": "awct",
  "name": "[GTM Wizard] Google Ads - {conversion_name}",
  "parameter": [
    {
      "type": "template",
      "key": "conversionId",
      "value": "{{[GTM Wizard] DLV - Google Ads Conversion ID}}"
    },
    {
      "type": "template",
      "key": "conversionLabel",
      "value": "REPLACE_WITH_CONVERSION_LABEL"
    },
    {
      "type": "template",
      "key": "conversionValue",
      "value": "{{dlv - value}}"
    },
    {
      "type": "template",
      "key": "currencyCode",
      "value": "{{dlv - currency}}"
    },
    {
      "type": "template",
      "key": "conversionCookiePrefix",
      "value": "_gcl"
    },
    {
      "type": "template",
      "key": "rdp",
      "value": "false"
    }
  ],
  "firingTriggerId": ["{TRIGGER_ID_FOR_EVENT}"]
}
```

**Note:** Set `count` to `"ONE"` for leads and `"EVERY"` for purchases by adding the `"conversionCountingType"` parameter. Replace `{conversion_name}` with the conversion name (e.g., `Lead`, `Purchase`).

### 3. Variable — Google Ads Conversion ID

```json
{
  "type": "c",
  "name": "[GTM Wizard] DLV - Google Ads Conversion ID",
  "parameter": [
    {
      "type": "template",
      "key": "value",
      "value": "REPLACE_WITH_AW-XXXXXXXXXX"
    }
  ]
}
```

## Best Practice Notes

- **Always include Conversion Linker:** The Conversion Linker (`gclidw`) is mandatory for any container running Google Ads conversion tags. Deploy it on All Pages so the gclid cookie is set as early as possible in the session.
- **Counting model matters:** Set conversion counting to "One" for lead forms (prevents double-counting if users resubmit) and "Every" for purchases (each transaction is a distinct conversion). Misconfigured counting distorts ROAS calculations.
- **Enhanced Conversions:** Enable Enhanced Conversions to improve match rates when cookies are blocked. This hashes and sends first-party customer data (email, phone, address) alongside the conversion. Requires passing the data through the dataLayer.
- **Consider importing GA4 goals:** If GA4 is already configured and measuring the same conversions, consider importing GA4 conversion events into Google Ads instead of running parallel `awct` tags. This reduces duplication and keeps a single source of truth for conversion data.
- **Consent Mode v2:** Google Ads requires Consent Mode v2 compliance for EU traffic. Without it, conversion modeling is degraded for users who declined consent. Implement `gtag('consent', 'default', {...})` before any tags fire.
- **Conversion window:** Default conversion window is 30 days for clicks. Align with your actual sales cycle — set longer windows for B2B / high-consideration purchases.
