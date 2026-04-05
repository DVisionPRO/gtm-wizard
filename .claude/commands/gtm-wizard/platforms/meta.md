---
platform: meta
version: 1.0.0
mode: full
detect_html: ['fbq(', 'connect.facebook.net', 'facebook.com/tr']
events: [PageView, Lead, CompleteRegistration, Purchase, AddToCart, ViewContent, InitiateCheckout, Contact, Schedule]
---

# Meta Platform Module

## Platform-Specific Checks

### META-1: Missing CAPI Parameters (event_id)
- **Severity:** HIGH
- **Condition:** `fbq()` conversion event calls exist in the container's custom HTML tags but the calls do not include an `eventID` parameter in the options object (third argument to `fbq('track', ...)`).
- **Explanation:** Without a matching `event_id` on both the browser pixel event and the server-side CAPI event, Meta cannot deduplicate the two signals. Both events are counted as separate conversions, inflating reported conversion numbers and corrupting campaign optimization signals.
- **Fix:** Generate a unique `event_id` per event using `Date.now().toString() + Math.random().toString(36).substr(2,9)`, pass it as `{eventID: event_id}` in the fbq options object, AND send the same `event_id` to the server-side CAPI tag. Push the `event_id` to the dataLayer for the sGTM CAPI tag to read.

### META-2: No Server-Side CAPI Event
- **Severity:** MEDIUM
- **Condition:** A browser-side Meta pixel exists (fbq() calls detected) but no server-side Conversions API (CAPI) tag is present in the sGTM container (or no sGTM container is in use).
- **Explanation:** Browser-only pixels lose 20-40% of conversions due to iOS ITP, ad blockers, and browser privacy restrictions. CAPI sends conversion data server-to-server, bypassing these restrictions and improving measurement completeness.
- **Fix:** Set up Meta Conversions API via a server-side GTM container. Route browser pixel events through the sGTM client, then forward to both the browser and the CAPI tag. Requires a Meta CAPI access token.

### META-3: PageView Not Firing
- **Severity:** WARN
- **Condition:** A Meta base pixel tag exists (contains `fbq('init', ...)`) but no `fbq('track', 'PageView'` substring is found in the same tag or any other tag on All Pages. Use substring matching for `fbq('track', 'PageView'` (without the closing parenthesis) to correctly match calls that include additional arguments like eventID options.
- **Explanation:** The base pixel only initializes the pixel and sets cookies. Without a `PageView` event call, no PageView is reported to Meta — this breaks retargeting audiences and inflates CPM by reducing pixel signal quality.
- **Fix:** Ensure the base pixel code includes `fbq('track', 'PageView')` immediately after `fbq('init', ...)`. The standard Meta base code already includes this — verify it has not been stripped out.

## Event Mapping

| dataLayer event | Meta pixel event | Notes |
|---|---|---|
| page_view | PageView | Fired by base pixel tag on All Pages |
| generate_lead | Lead | Standard event. Use for form submissions, quote requests. |
| purchase | Purchase | Standard event. Pass `value`, `currency`, `content_ids`. |
| add_to_cart | AddToCart | Standard event. Pass `content_ids`, `value`, `currency`. |
| begin_checkout | InitiateCheckout | Standard event |
| view_item | ViewContent | Standard event. Pass `content_ids`, `content_type`. |
| sign_up | CompleteRegistration | Standard event. Pass `status: true`. |
| contact | Contact | Standard event |
| schedule | Schedule | Standard event |
| *(unmapped)* | Custom event | Use `fbq('trackCustom', '{event_name}', {...})`. Custom events cannot be used for conversion optimization — only for audiences. |

## Scaffold Templates

### 1. Meta Base Pixel Tag (Custom HTML) — fires on All Pages

```json
{
  "type": "html",
  "name": "[GTM Wizard] Meta - Base Pixel",
  "parameter": [
    {
      "type": "template",
      "key": "html",
      "value": "<script>\n!function(f,b,e,v,n,t,s)\n{if(f.fbq)return;n=f.fbq=function(){n.callMethod?\nn.callMethod.apply(n,arguments):n.queue.push(arguments)};\nif(!f._fbq)f._fbq=n;n.push=n;n.loaded=!0;n.version='2.0';\nn.queue=[];t=b.createElement(e);t.async=!0;\nt.src=v;s=b.getElementsByTagName(e)[0];\ns.parentNode.insertBefore(t,s)}(window, document,'script',\n'https://connect.facebook.net/en_US/fbevents.js');\n\nfbq('init', '{{[GTM Wizard] DLV - Meta Pixel ID}}');\nfbq('track', 'PageView');\n</script>\n<noscript>\n  <img height=\"1\" width=\"1\" style=\"display:none\"\n    src=\"https://www.facebook.com/tr?id={{[GTM Wizard] DLV - Meta Pixel ID}}&ev=PageView&noscript=1\"\n  />\n</noscript>"
    },
    {
      "type": "boolean",
      "key": "supportDocumentWrite",
      "value": "false"
    }
  ],
  "firingTriggerId": ["2147479573"],
  "tagFiringOption": "oncePerLoad"
}
```

### 2. Meta Conversion Event Tag (Custom HTML) — per conversion event

```json
{
  "type": "html",
  "name": "[GTM Wizard] Meta - {MetaEventName}",
  "parameter": [
    {
      "type": "template",
      "key": "html",
      "value": "<script>\n(function() {\n  var event_id = Date.now().toString() + Math.random().toString(36).substr(2, 9);\n\n  // Push event_id to dataLayer for sGTM CAPI tag deduplication\n  window.dataLayer = window.dataLayer || [];\n  window.dataLayer.push({\n    'event': 'meta_event_id_ready',\n    'meta_event_id': event_id,\n    'meta_event_name': '{MetaEventName}'\n  });\n\n  fbq('track', '{MetaEventName}', {\n    // Add event parameters here, e.g.:\n    // value: {{dlv - value}},\n    // currency: {{dlv - currency}},\n    // content_ids: [{{dlv - content_id}}]\n  }, {\n    eventID: event_id\n  });\n})();\n</script>"
    },
    {
      "type": "boolean",
      "key": "supportDocumentWrite",
      "value": "false"
    }
  ],
  "firingTriggerId": ["{TRIGGER_ID_FOR_EVENT}"]
}
```

**Note:** Replace `{MetaEventName}` with the Meta event name (e.g., `Lead`, `Purchase`). The `event_id` is pushed to the dataLayer so the server-side CAPI tag can read it and send the same ID to the Conversions API for deduplication.

### 3. Variable — Meta Pixel ID

```json
{
  "type": "c",
  "name": "[GTM Wizard] DLV - Meta Pixel ID",
  "parameter": [
    {
      "type": "template",
      "key": "value",
      "value": "REPLACE_WITH_YOUR_PIXEL_ID"
    }
  ]
}
```

## Best Practice Notes

- **Always include event_id for deduplication:** When running both browser pixel and server CAPI, every conversion event must carry a matching `event_id`. Without it, Meta counts both signals as separate conversions. Generate `event_id` on the browser and send the same value to CAPI.
- **Hash PII for CAPI:** Customer data sent via Conversions API (email, phone number, first name, last name, date of birth, city, state, zip, country) must be SHA-256 hashed before sending. Most sGTM CAPI tags handle this automatically — verify hashing is enabled in the tag configuration.
- **EMQ target 6.0+:** Event Match Quality (EMQ) score reflects how well Meta can match conversion events to user accounts. Aim for 6.0 or higher. Improve by sending more customer information parameters (em, ph, fn, ln, ct, st, zp, country, external_id) with each event.
- **Test via Meta Test Events:** Before going live, use Meta Events Manager > Test Events to verify pixel fires and confirm the event_id appears in both the browser and CAPI hits. Look for the "Deduplicated" label — it confirms deduplication is working.
- **sGTM for iOS resilience:** Routing the Meta pixel through a server-side GTM container makes the pixel request first-party, bypassing Safari ITP restrictions. This significantly improves conversion measurement completeness on iOS devices.
- **Multiple pixels:** If a site has multiple Meta pixel IDs (e.g., different business accounts), initialize each with a separate `fbq('init', 'PIXEL_ID')` call. All subsequent `fbq('track', ...)` calls fire to all initialized pixels. Use `fbq('trackSingle', 'PIXEL_ID', ...)` to fire to a specific pixel only.
