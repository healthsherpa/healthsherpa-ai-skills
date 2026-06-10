# Payment & Document Upload Reference

## Payment Decision Tree

Read `payment_instructions` from the application response and follow this logic:

```
payment_instructions is null
  → no plan set yet (application still in draft without a plan)

payment_required_with_submission == true
  → GET /payment_redirect BEFORE calling /submit

payment_redirect_supported == true
  → submit FIRST, then GET /payment_redirect to send user to carrier payment

pay_by_phone_supported == true
  → show payment_phone_number to user

none of the above
  → carrier handles payment outside this flow
```

## payment_instructions Object

```json
{
  "payment_instructions": {
    "payment_required_with_submission": false,
    "payment_redirect_supported": true,
    "pay_by_phone_supported": true,
    "payment_phone_number": "8005550100"
  }
}
```

NEVER hardcode payment behavior per carrier. Always read from `payment_instructions`.

## GET /payment_redirect

```
GET /api/v1/applications/:id/payment_redirect
```

| Status | Meaning |
|---|---|
| 200 | Returns `{endpoint, method: "POST", fields: [{name, value}]}` |
| 404 | Carrier doesn't support redirect |
| 409 | Application not yet submitted |

Build a hidden HTML form from the response and auto-submit it to redirect the user to the carrier's payment page. Validate that `endpoint` begins with `https://` and HTML-escape every interpolated value (`endpoint`, each `field.name`, each `field.value`) before insertion. Carrier-supplied field values may legitimately contain characters that break unescaped HTML, and defense-in-depth requires escaping even though the response originates from HealthSherpa.

```html
<!-- Pseudocode. htmlEscape() must escape &, <, >, ", ' -->
<!-- Validate endpoint.startsWith("https://") before rendering. -->
<form id="payment-form" method="POST" action="${htmlEscape(endpoint)}">
  <!-- For each field in fields[]: -->
  <input type="hidden"
         name="${htmlEscape(field.name)}"
         value="${htmlEscape(field.value)}" />
</form>
<script>document.getElementById('payment-form').submit();</script>
```

The carrier handles payment collection. HealthSherpa does not process payments.

## Carrier-Reported Payment Status

Available in the application response under `payment`. Populated from carrier data feeds — may be null for days or weeks after submission.

```json
{
  "payment": {
    "payment_status": "paid",
    "payment_status_updated_date": "2026-06-15",
    "paid_through_date": "2026-07-31",
    "past_due_member_responsibility_balance_due": "0.00",
    "current_member_responsibility_balance_due": "50.50",
    "autopay_indicator": true
  }
}
```

NEVER rely on this for real-time payment confirmation. It is an asynchronous carrier feed.

## Document Upload

Required for some SEP types. After creating an application, check the `errors` array for a `supporting_documentation_required` entry.

```
POST /api/v1/applications/:id/supporting_documentation
```

### JSON Upload

```json
{
  "file": {
    "filename": "ichra_offering.pdf",
    "content_type": "application/pdf",
    "content_base64": "<base64-encoded-content>"
  },
  "document_type": "sep"
}
```

### Multipart Upload

```
Content-Type: multipart/form-data
```

Fields:
- `file` — the binary file
- `document_type` — `"sep"` (required)

### Rules

- `document_type` is required. The API rejects uploads without it. The only accepted value is `"sep"`.
- Max file size is **carrier-dependent**. Most carriers allow ~2 MB, but some are higher (currently Oscar 50 MB, HCSC 10 MB) and limits change over time. Do not hardcode a fixed cap; if an upload is rejected for size, check the carrier's current limit with your account manager.
- Supported formats: PDF, PNG, JPG.
- Upload BEFORE calling `/submit`.
- After upload, `document_status` on the application transitions from `required` to `uploaded`.
