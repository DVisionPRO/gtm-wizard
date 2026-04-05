---
platform: tiktok
version: 0.9.0
mode: full
detect_html: ['ttq.track', 'analytics.tiktok.com', 'tiktok-pixel']
events: [Pageview, ViewContent, AddToCart, InitiateCheckout, CompletePayment, PlaceAnOrder, SubmitForm, Subscribe, Contact, Search]
---

# TikTok Platform Module

> **Beta** — Diagnose, fix, and build scaffolding are supported. Validation against TikTok Events Manager is manual — no automated verification endpoint is available.

## Platform-Specific Checks

### TT-1: Non-Standard Event Names
- **Severity:** MEDIUM
- **Condition:** `ttq.track()` calls detected in custom HTML tags using event names that are not in the TikTok gallery events list (e.g., `ttq.track('form_submit')` instead of `ttq.track('SubmitForm')`).
- **Explanation:** TikTok only uses gallery events (standard event names) for campaign conversion optimization. Custom event names are recorded but cannot be selected as campaign objectives or optimization goals. Campaigns targeting a custom event name simply will not optimize.
- **Fix:** Map non-standard event names to the closest gallery event using the Event Mapping table below. If no close match exists, use the gallery event that most accurately represents the action (e.g., `SubmitForm` for any lead or form submission).

### TT-2: No Events API Server-Side Tag
- **Severity:** MEDIUM
- **Condition:** A client-side TikTok pixel (`ttq.track`) is detected but no TikTok Events API server-side tag is present in the sGTM container (or no sGTM container is in use).
- **Explanation:** Like Meta, TikTok's browser pixel is subject to iOS ITP and ad blocker interference. The TikTok Events API (server-to-server) bypasses these restrictions and can also deduplicate browser + server signals using `event_id`.
- **Fix:** Set up the TikTok Events API via a server-side GTM container using a TikTok Events API access token. Ensure matching `event_id` values are sent from both browser and server to enable deduplication.

## Event Mapping

| dataLayer event | TikTok gallery event | Notes |
|---|---|---|
| page_view | Pageview | Fired by base pixel tag on All Pages |
| view_item | ViewContent | Pass `content_id`, `content_type`, `value`, `currency` |
| add_to_cart | AddToCart | Pass `content_id`, `value`, `currency` |
| begin_checkout | InitiateCheckout | Pass `value`, `currency` |
| purchase | CompletePayment | Pass `value`, `currency`, `content_id`. This is TikTok's primary purchase event for ROAS optimization. |
| generate_lead | SubmitForm | Use for all form submissions, quote requests, contact forms |
| sign_up | Subscribe | Use for newsletter signup, account creation |
| contact | Contact | Use for contact page interactions, phone clicks |
| search | Search | Pass `query` parameter |
| *(no match)* | PlaceAnOrder | Use for order confirmation when `CompletePayment` fires later in a different session |

## Scaffold Templates

### 1. TikTok Base Pixel Tag (Custom HTML) — fires on All Pages

```json
{
  "type": "html",
  "name": "[GTM Wizard] TikTok - Base Pixel",
  "parameter": [
    {
      "type": "template",
      "key": "html",
      "value": "<script>\n!function (w, d, t) {\n  w.TiktokAnalyticsObject=t;var ttq=w[t]=w[t]||[];\n  ttq.methods=[\"page\",\"track\",\"identify\",\"instances\",\"debug\",\"on\",\"off\",\"once\",\"ready\",\"alias\",\"group\",\"enableCookie\",\"disableCookie\"];\n  ttq.setAndDefer=function(t,e){t[e]=function(){t.push([e].concat(Array.prototype.slice.call(arguments,0)))};\n  };for(var i=0;i<ttq.methods.length;i++)ttq.setAndDefer(ttq,ttq.methods[i]);\n  ttq.instance=function(t){for(var e=ttq._i[t]||[],n=0;n<ttq.methods.length;n++)ttq.setAndDefer(e,ttq.methods[n]);return e};\n  ttq.load=function(e,n){var i=\"https://analytics.tiktok.com/i18n/pixel/events.js\";\n    ttq._i=ttq._i||{},ttq._i[e]=[],ttq._i[e]._u=i,ttq._t=ttq._t||{},ttq._t[e]=+new Date,ttq._o=ttq._o||{};\n    ttq._o[e]=n||{};var o=document.createElement(\"script\");o.type=\"text/javascript\",o.async=!0,o.src=i+\"?sdkid=\"+e+\"&lib=\"+t;\n    var a=document.getElementsByTagName(\"script\")[0];a.parentNode.insertBefore(o,a)\n  };\n\n  ttq.load('{{[GTM Wizard] DLV - TikTok Pixel ID}}');\n  ttq.page();\n}(window, document, 'ttq');\n</script>"
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

**Note:** `ttq.page()` fires the `Pageview` event. This is equivalent to `fbq('track', 'PageView')` in the Meta pixel — it must be called after `ttq.load()`.

### 2. TikTok Event Tag (Custom HTML) — per conversion event

```json
{
  "type": "html",
  "name": "[GTM Wizard] TikTok - {TikTokEventName}",
  "parameter": [
    {
      "type": "template",
      "key": "html",
      "value": "<script>\nttq.track('{TikTokEventName}', {\n  // Add event parameters here, e.g.:\n  // value: {{dlv - value}},\n  // currency: {{dlv - currency}},\n  // content_id: {{dlv - content_id}}\n});\n</script>"
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

**Note:** Replace `{TikTokEventName}` with the gallery event name (e.g., `SubmitForm`, `CompletePayment`). Only gallery events can be used for campaign optimization.

### 3. Variable — TikTok Pixel ID

```json
{
  "type": "c",
  "name": "[GTM Wizard] DLV - TikTok Pixel ID",
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

- **Gallery events only for optimization:** Only TikTok gallery events (standard event names) can be selected as conversion goals in TikTok Ads Manager campaigns. Custom events recorded via `ttq.track('my_custom_event')` appear in Events Manager but cannot be used for campaign optimization or bidding. Always use the closest gallery event name.
- **Events API improves iOS tracking:** TikTok's Events API (server-to-server) is particularly important for iOS 14.5+ users where browser-side pixel signals are significantly degraded. Events API events bypass ITP and can be deduplicated with browser pixel events using a shared `event_id`.
- **Verify in TikTok Events Manager:** After implementation, use TikTok Events Manager > Test Events to confirm pixel fires and event names. Pay special attention to the event name column — a misspelled or non-gallery event name will show up but will silently fail to qualify for optimization.
- **SubmitForm for leads:** TikTok's gallery event for lead form submissions is `SubmitForm` (not `Lead`, which is a Meta convention). Using `Lead` or other non-gallery names is the most common TikTok tagging mistake.
- **CompletePayment vs PlaceAnOrder:** Use `CompletePayment` for confirmed purchases (payment processed). Use `PlaceAnOrder` only when the order is placed but payment is confirmed asynchronously in a separate session. Most e-commerce sites should use `CompletePayment`.
- **`ttq.identify()` for logged-in users:** If users are logged in, call `ttq.identify({email: 'hashed_email', phone_number: 'hashed_phone'})` to improve match rates. Values must be SHA-256 hashed.
