---
platform: ga4
version: 1.0.0
mode: full
detect_types: [gaawc, gaawe, googtag]
events: [page_view, generate_lead, purchase, add_to_cart, begin_checkout, view_item, sign_up, login, search]
---

# GA4 Platform Module

## Platform-Specific Checks

### GA-1: Missing Cross-Domain Linker
- **Severity:** HIGH
- **Condition:** Container has GA4 configuration tags (`gaawc` or `googtag` with a `G-` measurement ID) AND custom HTML tags referencing multiple domains AND no `gclidw` (Conversion Linker) tag present.
- **Explanation:** Without a Conversion Linker, cross-domain attribution breaks — gclid parameters are not carried across domains, causing paid traffic to appear as direct.
- **Fix:** Add a Conversion Linker tag (`gclidw`) firing on All Pages trigger (2147479573).

## Event Mapping

GA4 uses the same event names as the dataLayer — no name translation is required.

| dataLayer event | GA4 event | Notes |
|---|---|---|
| page_view | *(automatic)* | Sent automatically by Config tag (`gaawc`) when `sendPageView: true`. Do NOT create a separate `gaawe` tag for page_view. |
| generate_lead | generate_lead | Standard GA4 recommended event |
| purchase | purchase | Standard GA4 recommended event. Requires `transaction_id`, `value`, `currency`. |
| add_to_cart | add_to_cart | Standard GA4 recommended event |
| begin_checkout | begin_checkout | Standard GA4 recommended event |
| view_item | view_item | Standard GA4 recommended event |
| sign_up | sign_up | Standard GA4 recommended event |
| login | login | Standard GA4 recommended event |
| search | search | Standard GA4 recommended event. Use `search_term` parameter. |

**Important:** `page_view` is sent automatically by the Config tag (`gaawc`) when `sendPageView` is `true`. Creating an additional `gaawe` event tag for `page_view` causes duplicate page_view events in GA4 reports.

## Scaffold Templates

### 1. GA4 Config Tag (gaawc)

```json
{
  "type": "gaawc",
  "name": "[GTM Wizard] GA4 - Config",
  "parameter": [
    {
      "type": "template",
      "key": "measurementId",
      "value": "{{[GTM Wizard] DLV - GA4 Measurement ID}}"
    },
    {
      "type": "boolean",
      "key": "sendPageView",
      "value": "true"
    }
  ],
  "firingTriggerId": ["2147479573"],
  "tagFiringOption": "oncePerLoad"
}
```

### 2. GA4 Event Tag (gaawe) — per conversion event

```json
{
  "type": "gaawe",
  "name": "[GTM Wizard] GA4 - {event_name}",
  "parameter": [
    {
      "type": "TAG_REFERENCE",
      "key": "measurementId",
      "value": "[GTM Wizard] GA4 - Config"
    },
    {
      "type": "template",
      "key": "eventName",
      "value": "{event_name}"
    },
    {
      "type": "list",
      "key": "eventParameters",
      "list": []
    }
  ],
  "firingTriggerId": ["{TRIGGER_ID_FOR_EVENT}"]
}
```

**Note:** Replace `{event_name}` with the GA4 event name (e.g., `generate_lead`, `purchase`). The `measurementId` field uses `TAG_REFERENCE` type pointing to the Config tag by name — this links the event tag to the config without hardcoding the measurement ID.

### 3. Variable — GA4 Measurement ID

```json
{
  "type": "c",
  "name": "[GTM Wizard] DLV - GA4 Measurement ID",
  "parameter": [
    {
      "type": "template",
      "key": "value",
      "value": "REPLACE_WITH_G-XXXXXXXXXX"
    }
  ]
}
```

### 4. Conversion Linker Tag (gclidw) — for GA-1 fix

```json
{
  "type": "gclidw",
  "name": "[GTM Wizard] GA4 - Conversion Linker",
  "parameter": [
    {
      "type": "boolean",
      "key": "enableCrossDomain",
      "value": "true"
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

## Best Practice Notes

- **Single Config tag:** Use exactly one `gaawc` Config tag per measurement ID. Multiple Config tags for the same property cause duplicate sessions and inflated pageview counts.
- **No duplicate page_view:** The Config tag sends `page_view` automatically. Do not add a `gaawe` event tag for `page_view` — this is the most common GA4 over-tagging mistake.
- **TAG_REFERENCE for event tags:** Always use `TAG_REFERENCE` type (not a variable) to link event tags to the Config tag. This ensures the measurement ID is resolved at runtime from the Config tag, not hardcoded.
- **sGTM for ITP resilience:** For accurate attribution on Safari/iOS (Intelligent Tracking Prevention), route GA4 hits through a server-side GTM container. This upgrades first-party cookies from JavaScript-set (7-day cap) to server-set (no cap).
- **Enhanced Measurement:** GA4's Enhanced Measurement (scroll, outbound clicks, file downloads, video, form interactions) fires directly from the `gaawc` Config tag. Review which events you actually need — disabling unused Enhanced Measurement events reduces noise in reports.
- **Consent Mode v2:** Ensure `gtag('consent', 'default', {...})` is set before the Config tag fires if operating in the EU. Use `gtag('consent', 'update', {...})` after user consent is captured.
