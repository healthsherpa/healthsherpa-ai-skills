# Payment & Document Upload Reference

## Payment Decision Tree

Read `payment_instructions` from the application response and follow this logic:

```
if payment_instructions is null:
  The application does not have a plan yet.
  Stop payment routing until a plan has been selected.

if payment_required_with_submission is true:
  Require rel "payment_method" in next_actions.
  Require method PUT and a relative href matching the current application.
  Send the supported payment method through the HealthSherpa-provided PCI proxy.
  Require 200 OK before calling /submit.
  If the action is absent, stop and contact HealthSherpa.

otherwise:
  Submit the application first.
  If payment_redirect_supported is true, use GET /payment_redirect.
  If pay_by_phone_supported is true, show payment_phone_number.
  If neither is true, the carrier handles payment outside this flow.
```

## payment_instructions Object

```json
{
  "payment_instructions": {
    "payment_required_with_submission": true,
    "payment_redirect_supported": false,
    "pay_by_phone_supported": true,
    "payment_phone_number": "8557481808"
  }
}
```

NEVER hardcode payment behavior per carrier. Always read from `payment_instructions`.

### BCBS Michigan medical

BCBSM PPO and BCN HMO applications return `payment_required_with_submission: false`, `payment_redirect_supported: true`, and `pay_by_phone_supported: true`. Submit first. The payment redirect is an HTTPS `POST` form with an opaque `SAMLResponse` field. HTML-escape every returned field while preserving its decoded form value, and never log it. Read the instructions, endpoint, fields, and phone number from each application response instead of hardcoding them.

## PUT /payment_method

Use this endpoint when `payment_required_with_submission` is `true`. Anthem and Wellpoint currently use this in-flow ACH path. The create or GET response may include a `missing_required_field` error for `payment_method` until this call succeeds.

Require the `next_actions` entry with `rel: "payment_method"`:

```json
{
  "rel": "payment_method",
  "href": "/api/v1/applications/HSA000000001/payment_method",
  "method": "PUT"
}
```

If `payment_required_with_submission` is `true` but this action is absent, stop and contact HealthSherpa. Do not construct the endpoint URL or submit the application.

Validate the action before joining its `href` to the PCI proxy base URL:

- `method` must equal `"PUT"`.
- `href` must be a relative path, not an absolute URL.
- `href` must equal `/api/v1/applications/{application_id}/payment_method` for the current application.
- Reject unexpected methods, hosts, paths, and application ID mismatches.

Send the request from your backend through the PCI-compliant proxy URL provided by HealthSherpa. The proxy tokenizes the bank account number in transit. Reject every HTTP redirect so credentials and bank data cannot leave the pinned origin. Do not send this request from browser code, and do not log `eft_routing`, `eft_number`, or the full request body.

Build the payment object at runtime from the consumer's billing and bank information. Do not save a reusable request file containing bank details or hardcode test fixtures in production code. Pin the exact payment-proxy origin supplied during onboarding in server-side deployment configuration; never accept it from a request.

```javascript
const apiKey = process.env.HS_API_KEY;
if (!apiKey || !paymentProxyBaseUrl || !pinnedPaymentProxyOrigin) {
  throw new Error("HealthSherpa payment configuration is unavailable");
}

const paymentProxyUrl = new URL(paymentProxyBaseUrl);
if (
  paymentProxyUrl.protocol !== "https:" ||
  paymentProxyUrl.origin !== pinnedPaymentProxyOrigin ||
  paymentProxyUrl.username ||
  paymentProxyUrl.password
) {
  throw new Error("Invalid HealthSherpa payment proxy origin");
}

const expectedPath =
  `/api/v1/applications/${encodeURIComponent(applicationId)}/payment_method`;
const paymentAction = nextActions.find(
  (action) => action.rel === "payment_method"
);

if (
  paymentAction?.method !== "PUT" ||
  paymentAction.href !== expectedPath ||
  !paymentAction.href.startsWith("/")
) {
  throw new Error("Invalid payment_method action");
}

const effectiveWithdrawDay =
  paymentType === "both" ? (selectedWithdrawDay ?? 1) : undefined;

if (
  paymentType === "both" &&
  (
    !Number.isInteger(effectiveWithdrawDay) ||
    effectiveWithdrawDay < 1 ||
    effectiveWithdrawDay > 28
  )
) {
  throw new Error("withdraw_day must be an integer from 1 through 28");
}

const paymentMethod = {
  payment_method_type: "bank_account",
  first_name: billingAccount.firstName,
  last_name: billingAccount.lastName,
  address: {
    street_address_1: billingAccount.address.street1,
    street_address_2: billingAccount.address.street2,
    city: billingAccount.address.city,
    state: billingAccount.address.state,
    zip_code: billingAccount.address.zipCode,
  },
  eft_routing: bankAccount.routingNumber,
  eft_number: bankAccount.accountNumber,
  eft_type: bankAccount.accountType,
  eft_level: bankAccount.accountLevel,
  bank_name: bankAccount.bankName,
  payment_type: paymentType,
  ...(paymentType === "both" && { withdraw_day: effectiveWithdrawDay }),
};

const response = await fetch(
  `${paymentProxyUrl.origin}${paymentAction.href}`,
  {
    method: "PUT",
    redirect: "error",
    headers: {
      "x-api-key": apiKey,
      "Content-Type": "application/json",
    },
    body: JSON.stringify(paymentMethod),
  }
);

if (response.status !== 200) {
  throw new Error(`Payment method failed with HTTP ${response.status}`);
}
```

`paymentProxyBaseUrl` is the PCI proxy URL provided during onboarding. Keep the API key and proxy URL in server-side secret/configuration storage.

### Staging Test Fixture

Anthem and Wellpoint provided the following personal savings account for Elevance staging. Use these values only in staging tests and never in production. Build production requests from the consumer's actual bank and billing information.

```json
{
  "first_name": "Test",
  "last_name": "Applicant",
  "eft_routing": "071205850",
  "eft_number": "23487289374982",
  "eft_type": "savings",
  "eft_level": "personal",
  "bank_name": "Test Bank"
}
```

The carrier's `PERSONALSAVINGS` account type maps to `eft_type: "savings"` plus `eft_level: "personal"`. The API still requires separate `first_name` and `last_name` fields; synthetic account-holder values are accepted for this staging fixture.

For this fixture, a successful `PUT /payment_method` returns `200` with a masked account. A later `202` from `/submit` confirms only that HealthSherpa queued the submission. Neither response confirms carrier acknowledgement, payment collection, 834 or SFTP delivery, or final effectuation.

Required fields:

- `payment_method_type`: use `"bank_account"` for the current public EnrollConnect integration.
- `first_name`, `last_name`: names on the bank account.
- `address`: billing address with `street_address_1`, `city`, `state`, and `zip_code`.
- `eft_routing`: nine-digit ABA routing number.
- `eft_number`: bank account number.
- `eft_type`: `"checking"` or `"savings"`.
- `eft_level`: `"personal"` or `"business"`.
- `bank_name`: financial institution name.
- `payment_type`: the endpoint enum is `"initial"`, `"both"`, or `"recurring"`. Use `"initial"` for binder only or `"both"` for binder plus recurring premiums. There is no current Anthem or Wellpoint use case for `"recurring"` without an initial payment; that request returns `422`.

Optional fields:

- `withdraw_day`: recurring withdrawal day from 1 through 28. Applies with `payment_type: "both"` and defaults to `1` when omitted.
- `address.street_address_2`: second billing address line.

### Payment Roles

HealthSherpa securely transmits the bank and payment-selection data from the platform to Anthem or Wellpoint but does not initiate withdrawals.

- With `payment_type: "initial"`, the carrier debits only the first payment.
- With `payment_type: "both"`, the carrier debits the first payment and recurring monthly premiums using `withdraw_day` for the monthly schedule.

### Success Response

```json
{
  "payment_method_type": "bank_account",
  "payment_type": "both",
  "masked_account": "****4982",
  "status": "active",
  "is_recurring": true,
  "withdraw_day": 12
}
```

A `200` response confirms that the payment method was saved. It does not confirm that the carrier collected payment or effectuated coverage. Only call `/submit` after this request succeeds.

`POST /submit` returns 202 with `application_id` when the background submission job is queued. It does not return `policy_status`. Poll the application with backoff and continue through `sep_docs_under_review` and `pending_effectuation`, because both are intermediate states. Stop on lifecycle terminal states `effectuated`, `cancelled`, or `terminated`. Stop the current polling attempt at `submission_failed`, then read `errors`, remediate, and resubmit. Treat `effectuated` as success even if no poll observed either intermediate state. An immediate `draft` response is transient; if it remains `draft` for more than one hour, alert and reconcile instead of resubmitting automatically.

| Status | Meaning |
|---|---|
| 200 | Payment method saved; response contains masked account details |
| 401 | API key is missing or invalid |
| 404 | Application was not found |
| 422 | Payment is unavailable for the application or a field is invalid |

## GET /payment_redirect

```
GET /api/v1/applications/:id/payment_redirect
```

| Status | Meaning |
|---|---|
| 200 | Returns `{endpoint, method: "POST", fields: [{name, value}]}` |
| 404 | Application was not found, or the carrier does not support payment redirect |
| 422 | Application has not been submitted or another redirect prerequisite is invalid |

Build a hidden HTML form only from a fresh, authenticated HealthSherpa response for the current authorized application. Never accept the endpoint or fields from a browser parameter or persisted client value. Require a non-empty exact-origin allowlist supplied during onboarding. Parse the endpoint, require HTTPS, reject userinfo, and require its origin to appear in that allowlist. Require the returned method to be `POST`. HTML-escape every interpolated value (`endpoint`, `method`, each `field.name`, and each `field.value`) while preserving its decoded form value. Carrier-supplied field values may contain characters that break unescaped HTML.

```javascript
// endpoint, method, and fields come from the fresh HealthSherpa response.
const paymentUrl = new URL(endpoint);
if (!paymentRedirectOriginAllowlist?.size) {
  throw new Error("Payment redirect origins are not configured");
}
if (
  paymentUrl.protocol !== "https:" ||
  paymentUrl.username ||
  paymentUrl.password ||
  !paymentRedirectOriginAllowlist.has(paymentUrl.origin)
) {
  throw new Error("Payment redirect origin is not allowed");
}
if (method.toUpperCase() !== "POST") {
  throw new Error("Payment redirect must use POST");
}

// htmlEscape() escapes &, <, >, ", and ' while preserving form values.
const hiddenInputs = fields.map(
  ({ name, value }) =>
    `<input type="hidden" name="${htmlEscape(name)}" value="${htmlEscape(value)}">`
).join("");
const formHtml =
  `<form id="payment-form" method="${htmlEscape(method)}" ` +
  `action="${htmlEscape(paymentUrl.toString())}">${hiddenInputs}</form>`;
document.body.insertAdjacentHTML("beforeend", formHtml);
document.getElementById("payment-form").submit();
```

For redirect flows, the carrier page handles payment collection. For in-flow ACH, HealthSherpa securely accepts and forwards the payment method as part of carrier submission.

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
- `document_type` — `"sep"` for qualifying-event proof or `"proof_of_residency"` for required residency proof

### Rules

- `document_type` is required. The API rejects uploads without it.
- Max file size is **carrier-dependent**. Most carriers allow ~2 MB, but some are higher (currently Oscar 50 MB, HCSC 10 MB) and limits change over time. Do not hardcode a fixed cap; if an upload is rejected for size, check the carrier's current limit with your account manager.
- Supported formats: PDF, PNG, JPG.
- Upload BEFORE calling `/submit`.
- After upload, `document_status` on the application transitions from `required` to `uploaded`.
- For plan year 2026, BCBS Michigan medical requirements mark documentation as required for every returned SEP type, including `offered_ichra` and `offered_qsehra`.
