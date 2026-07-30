---
name: ichra-platform-integration
description: Build ICHRA platforms that integrate with HealthSherpa APIs for plan quoting, enrollment, payment, and policy management. Use when building or debugging an ICHRA platform integration, working with HealthSherpa enrollment or quoting APIs, or when the user mentions ICHRA, HRA, enrollment, quoting, plan selection, carrier submission, or policy status.
metadata:
  author: healthsherpa
  version: "1.0"
---

# ICHRA Platform Integration with HealthSherpa

## Environments

| Environment | Base URL | Notes |
|---|---|---|
| Production | `https://api.ichra.healthsherpa.com` | Primary. Use for all live integrations. |
| Staging | `https://api.ichra-staging.healthsherpa.com` | For testing only. May have intermittent backend issues. |

All EnrollConnect and Quoting endpoints use these base URLs. Deeplinks use different hosts (see [deeplink-enrollment.md](references/deeplink-enrollment.md)).

## Authentication

All requests require an `x-api-key` header. API keys are provided during onboarding.

```
x-api-key: your_api_key_here
Content-Type: application/json
```

All POST and PUT requests must set `Content-Type: application/json` (except multipart file uploads). There is no OAuth, no sessions, no HMAC request signing.

## Security & Credentials

The integrator is responsible for the security of their own systems, API credentials, webhook endpoints, and any PII or PHI they process. This skill provides API shape guidance only; it does not exempt integrators from standard secure-development practices.

**Credential handling**
- API keys, webhook signing secrets, and staging Basic Auth credentials are secrets. Treat them like database passwords: store in environment variables or a secret manager, never in source code, never in client-side bundles, never in version control.
- Rotate credentials if compromise is suspected. Contact your HealthSherpa account manager to issue replacements.

**Server-side only**
- All HealthSherpa API calls (Quoting, EnrollConnect, Deeplink, Payment Method, Payment Redirect, Plan Lookup) must be made from a server you control. Frontend code must call your own backend, which then calls HealthSherpa.
- The payment redirect form and deeplink `Location` URL are the only HealthSherpa surfaces a browser interacts with directly. Before redirecting, validate the URL uses `https://` and (for deeplinks) points at a HealthSherpa host.
- Send payment method requests through the PCI-compliant proxy URL provided by HealthSherpa. The proxy tokenizes the bank account number in transit.

**PII / PHI**
- These endpoints transmit SSN, ITIN, DOB, signatures, addresses, employer FEIN, household income, and bank account details. Treat all payloads as sensitive.
- Do not log full request or response bodies. Redact at minimum: `ssn`, `itin`, `date_of_birth`, `signature`, `email`, `phone`, `residential_address`, `mailing_address`, `hra.employer.fein`, `annual_household_income`, `eft_routing`, and `eft_number`. HealthSherpa already redacts SSN/ITIN in the `events` timeline. Apply the same standard to your own storage and logs.
- HIPAA, state privacy laws, and CMS requirements apply to integrators independent of this skill.

**Webhook endpoint**
- Serve over HTTPS only.
- Authenticate every request using the method you configured during onboarding (HealthSherpa supports a variety of methods; requests carry headers consistent with that method). Reject requests that fail authentication before any business logic runs.
- Process events idempotently (dedupe by `transaction_id`) to handle repeat deliveries safely.

**Defensive rendering**
- When building the payment redirect HTML form, require the returned `method` to be `POST`. HTML-escape `endpoint`, `method`, every `field.name`, and every `field.value` before insertion. Validate `endpoint` begins with `https://`.
- Do not interpolate response data into HTML, SQL, shell commands, or file paths without escaping/parameterization.

**TLS**
- Never disable certificate verification when calling HealthSherpa APIs (no `verify=False`, no `rejectUnauthorized: false`).

**Errors and retries**
- Do not retry 4xx responses (401, 403, 422). These are deterministic and indicate a payload or auth issue that retrying will not fix.
- Use the `retry_after` header on 429 responses for backoff.

## Critical Rules

- ALWAYS use `x-api-key` header for authentication. No OAuth, no sessions.
- ALWAYS call HealthSherpa endpoints from server-side code. NEVER embed the API key, webhook secret, or staging Basic Auth credentials in browser, mobile, or any client-distributed code.
- ALWAYS load the API key from an environment variable or secret manager. NEVER hardcode it in source code or commit it to version control.
- ALWAYS use HTTPS. NEVER disable TLS certificate verification to work around connection errors.
- NEVER log full request or response bodies. Applicant fields including `ssn`, `itin`, `date_of_birth`, `signature`, `email`, full `residential_address`, and `hra.employer.fein` must be redacted from logs and error reports.
- ALWAYS authenticate webhook requests using the method configured during onboarding. Reject requests that fail authentication before any business logic runs.
- ALWAYS use `application_id` (not `id`) as the application identifier in responses.
- ALWAYS use HealthSherpa-assigned applicant `member_id` for matching on PUT. `external_id` is optional metadata.
- ALWAYS check `api_enrollment` / `deeplink_enrollment` flags on each plan before routing to enroll.
- ALWAYS read `payment_instructions` from the application response. NEVER hardcode carrier payment behavior.
- When `payment_instructions.payment_required_with_submission` is `true`, require a `next_actions` entry with `rel: "payment_method"`. Require `method: "PUT"` and an `href` equal to `/api/v1/applications/{application_id}/payment_method` for the current application. Reject absolute URLs, unexpected methods, and application ID mismatches. Call the validated action before `/submit` and require a successful response. If the action is absent or invalid, stop and contact HealthSherpa.
- Send `PUT /api/v1/applications/:id/payment_method` through the PCI-compliant proxy URL provided by HealthSherpa. NEVER log or expose `eft_routing` or `eft_number`.
- Build the payment request object at runtime from the consumer's billing, bank, and payment-selection data. Do not store a reusable request file containing bank details or hardcode staging fixtures in production code.
- ALWAYS send full payload on PUT. No PATCH semantics in V1.
- ALWAYS include `document_type` when uploading supporting documentation. Use `"sep"` for qualifying-event proof and `"proof_of_residency"` when plan requirements indicate residency proof is required.
- ALWAYS include `signatures.signature_date` and `applicants.primary.signature` — these are required for submission.
- ALWAYS use `street_address_1`/`street_address_2` (not `street_line_*`).
- ALWAYS include `residential_address.fips_code` on application create and update requests so HealthSherpa can validate plan availability in the applicant's county. Quoting uses `fip_code`; enrollment applications use `fips_code`.
- ALWAYS use `gender` (not `sex`) and `uses_tobacco` (not `tobacco_use`).
- ALWAYS use `state_supplement_*` for state signature fields (not `addendum_*`).
- In the request, `pediatric_dental` (string enum: `"purchased_separately"` / `"not_applicable"`) goes under `attestations`. In the response, it's a top-level field. `pediatric_dental_signature` goes under `signatures`.
- For carriers that offer qualified dental (currently HCSC — Blue Cross and Blue Shield of IL, MT, NM, OK, and TX), the ACA pediatric dental requirement must be satisfied **exactly one way** per application: send `attestations.pediatric_dental` OR a top-level `dental_plan_hios_id`, never both and never neither. Sending both, or neither, returns a 422 (`Invalid dental selection`). See "Qualified Dental (HCSC)".
- ALWAYS include top-level `hra` with employer `name`, `fein`, and `address` on every ICHRA enrollment. When `enrollment_requirements.hra.per_applicant_hra` is true, also collect and send `hra` under the primary and every non-child dependent relationship supported by the selected carrier, using the returned allowed values. Do not collect per-applicant HRA answers from children. Send the household follow-up at top-level `qsehra_both_employers_claim_reimbursement` when its returned condition applies.
- NEVER require `external_id`. It is optional everywhere.
- NEVER assume real-time payment confirmation. Most carriers report asynchronously via feeds.
- ALWAYS fetch `enrollment_requirements` via `GET /plans/:hios_id?plan_year=YYYY&include=enrollment_requirements`. Process returned `attestations`, `special_enrollment_period.event_types`, applicant questions, communication preferences, HRA requirements, and proof-of-residency instructions. Use the returned questions, content, labels, options, date windows, and conditional requiredness. Omitted keys do not apply to the selected plan.
- When `communication_preferences.email_contact_consent.required` is true, show the returned carrier content and collect an explicit Yes or No. Send the boolean at `communication_preferences.email_contact_consent`; both `true` and `false` satisfy requiredness.
- When the plan returns Wellpoint Texas communication-impairment requirements, collect `applicants.primary.has_communication_impairment`. A `true` answer requires `communication_impairment_format`; `other` also requires `communication_impairment_format_other`. A `false` answer must remain an explicit No.
- Interpret each SEP reason's `event_date_days_before` and `event_date_days_after` relative to today. They define how far the qualifying-event date may be in the past or future; they are not enrollment windows measured from the event.
- ALWAYS include `agent_of_record` with at minimum `first_name`, `last_name`, and `national_producer_number` on every enrollment. Without it, the enrollment may not be attributed to the correct agent or broker.
- ALWAYS include `off_ex: true` in quoting requests. The API only supports off-exchange enrollment. Without it, the API returns on-exchange plans which are not supported.
- All enums are snake_case lowercase. Dates in API requests and responses are ISO 8601; dates in webhook payloads are `MM/DD/YYYY` (see webhooks-and-monitoring.md). Money as string (`"50.50"`).

## API Behavior Notes

Some response values are normalized differently from input. Code defensively:

| Behavior | Guidance |
|---|---|
| **Response field casing differs from input** — e.g., send `phone_type: "cell"`, receive `"Cell"`; send `hispanic_origin: "decline_to_answer"`, receive `"decline"` | Normalize response values to lowercase/snake_case before comparing to stored input. Do not rely on exact round-trip equality for: `phone_type`, `hispanic_origin`, `race_ethnicity`, `event_type`, `language_spoken`, `language_written`. |
| **`event_type` lossy mapping** — `offered_ichra` maps internally to `ichra_qsehra` and comes back as that value instead of what you sent | Store the original value you sent locally. Do not rely on the response `event_type` as your source of truth. |
| **`attestations` empty in response** — attestation values are accepted on create but not serialized back on GET | Store attestation values locally after successful create/update. |
| **List response uses `primary_applicant` not `applicants`** — list items have `primary_applicant: {member_id, first_name, last_name, date_of_birth, external_id}` | Use `item.primary_applicant.first_name` (not `item.applicants.primary.first_name`) for list display. |
| **List pagination uses `total_count`** — not `total` | Read `pagination.total_count` (not `pagination.total`). |
| **`plan_hios_id` and `issuer_hios_id` null in list** — these fields may be null in `GET /applications` list responses | Fetch individual application via `GET /applications/:id` for complete data. |
| **Create/GET response nesting** — `POST /applications` and `GET /applications/:id` nest app data under an `application` key, with `application_id`, `errors`, `next_actions`, `payment_instructions` at the top level | Flatten: `{...response.application, application_id: response.application_id, errors: response.errors, next_actions: response.next_actions, payment_instructions: response.payment_instructions}` |
| **Plan lookup response nesting** — `GET /plans/:hios_id` wraps plan data under a `plan` key | Access plan data via `response.plan`, not directly on the response. e.g. `response.plan.enrollment_requirements.attestations` |

## Architecture

HealthSherpa = enrollment infrastructure (carrier submission, quoting, policy management).
Your platform = user experience (employer admin, employee shopping, HRA administration).

```
Your platform → HealthSherpa ICHRA API → Carrier
```

## Endpoints

### Quoting & Plan Lookup

| Method | Path | Purpose |
|---|---|---|
| POST | `/api/v1/quotes` | Quote plans with premiums and enrollment flags |
| GET | `/api/v1/plans?state=AZ&plan_year=2026` | List plans + benefit metadata for a state/year (no premiums) — cache for quoting UIs |
| GET | `/api/v1/plans/:hios_id?plan_year=2026` | Plan details, enrollment flags, and optional attestation content |

Add `?include=enrollment_requirements` to plan lookup to get carrier-specific attestations, SEP reasons and date windows, applicant questions, communication preferences, HRA requirements, and residency instructions. Use `GET /api/v1/plans` to pull benefit data in bulk and cache it locally; it excludes premiums, enrollment flags, and enrollment requirements (fetch those via `POST /quotes` or `GET /plans/:hios_id`).

### EnrollConnect (API Enrollment)

Use when `api_enrollment: true`. Full lifecycle control.

| Method | Path | Purpose |
|---|---|---|
| POST | `/api/v1/applications` | Create enrollment application |
| PUT | `/api/v1/applications/:id` | Update application (full replace) |
| GET | `/api/v1/applications/:id` | Retrieve application with status, errors, and next_actions |
| GET | `/api/v1/applications` | List applications (filters + pagination) |
| POST | `/api/v1/applications/:id/submit` | Submit to carrier |
| POST | `/api/v1/applications/:id/cancel` | Request cancellation (async, 202) |
| POST | `/api/v1/applications/:id/terminate` | Request termination (async, 202) |
| POST | `/api/v1/applications/:id/supporting_documentation` | Upload SEP docs (max size is carrier-dependent — see Document Upload) |
| PUT | `/api/v1/applications/:id/payment_method` | Set required in-flow ACH payment method before submission |
| GET | `/api/v1/applications/:id/payment_redirect` | Get carrier payment page data |

### FIPS Code Lookup (External)

The quoting API requires a `fip_code` (FIPS county code). Resolve zip codes to FIPS codes using the public CMS Marketplace API:

```
GET https://marketplace-int.api.healthcare.gov/api/v1/counties/by/zip/{zipcode}
```

Response:
```json
{
  "counties": [
    {"zipcode": "42003", "name": "McCracken County", "fips": "21145", "state": "KY"},
    {"zipcode": "42003", "name": "Graves County", "fips": "21083", "state": "KY"}
  ]
}
```

If multiple counties are returned, prompt the user to select one. Pass the `fips` value as `fip_code` in the quoting request. No API key required for this endpoint.

### Deeplink Enrollment

Use when `deeplink_enrollment: true` and `api_enrollment: false`. Redirects user to HealthSherpa UI.

| Method | Path | Purpose |
|---|---|---|
| POST | `/public/ichra/off_ex` | Deeplink — prefills HealthSherpa enrollment UI. Returns 302. |

Deeplink uses a flat schema that differs from EnrollConnect. See [deeplink-enrollment.md](references/deeplink-enrollment.md) for the schema mapping and 302 handling.

## Enrollment Lifecycle

```
draft → pending_effectuation → effectuated
  ↓            ↓                    ↓
  ↓     submission_failed    cancelled / terminated
  ↓
sep_docs_required → sep_docs_under_review → pending_effectuation
```

| Status | Meaning |
|---|---|
| `draft` | Created, not yet submitted |
| `submission_failed` | Carrier submission failed (async job error). Check application errors. |
| `sep_docs_required` | SEP documentation must be uploaded before carrier will process |
| `sep_docs_under_review` | SEP docs uploaded, carrier is reviewing |
| `pending_effectuation` | Submitted to carrier, awaiting confirmation |
| `effectuated` | Carrier confirmed, coverage active |
| `cancelled` | Policy never took effect (never effectuated), typically non-payment. Some carriers do not support requesting this. |
| `terminated` | Policy was active and later ended (consumer, carrier, or plan-year expiration) |

## Minimum Viable Payload

The API does not support partial creates. `POST /applications` validates the full payload on create and rejects incomplete requests. Build the complete payload before calling create.

Required fields vary by carrier. Some carriers require SSN, race/ethnicity, hispanic origin, language, and other fields that are optional for others. The safest approach is to always send the full set below.

**Always required (all carriers):**

| Section | Fields |
|---|---|
| Top-level | `plan_hios_id`, `plan_year` |
| `applicants.primary` | `first_name`, `last_name`, `date_of_birth`, `gender`, `email`, `phone`, `phone_type`, `signature` |
| `residential_address` | `street_address_1`, `city`, `state`, `zip_code`, `fips_code` |
| `signatures` | `signature_date` |
| `attestations` | `electronic_signature_consent: true`, `agrees_issuer_attestations: true` |
| `special_enrollment_period` | `event_type`, `event_date` |

**Required by most carriers (send these to avoid carrier-specific 422s):**

| Section | Fields | Notes |
|---|---|---|
| `applicants.primary` | `ssn` | Some carriers require SSN. Others accept ITIN as alternative. Some require at least one of SSN/ITIN. |
| `applicants.primary` | `us_citizen` | Required when carrier asks citizenship question (most do). |
| `applicants.primary` | `resides_in_state` | Required when carrier asks residency question (most do). |
| `applicants.primary` | `uses_tobacco` | Required when carrier asks tobacco question (most do). |
| `applicants.primary` | `race_ethnicity` | Required by some carriers. Use `"decline_to_answer"` if not collected. |
| `applicants.primary` | `hispanic_origin` | Required by some carriers. Use `"decline_to_answer"` if not collected. |
| `applicants.primary` | `language_spoken`, `language_written` | Required for primary by some carriers. Default `"english"` if not collected. |

**Expected for all ICHRA enrollments (omitting these will degrade the enrollment experience):**

| Section | Fields | Why |
|---|---|---|
| `hra` | `offered_hra`, `type`, `amount`, `contribution_covers`, `start` | Core ICHRA data. Without this, HealthSherpa cannot track the HRA offer, calculate employer contributions, or support downstream reimbursement workflows. Always include for ICHRA enrollments. |
| `hra.employer` | `name`, `fein`, `phone`, `address: {street_address_1, city, state, zip_code}` | Employer identification. Without the employer `name`, `fein`, and `address`, HealthSherpa cannot associate the enrollment with the correct employer group. This breaks employer-level reporting, group management, and carrier coordination. Always include all three. |
| `agent_of_record` | `first_name`, `last_name`, `national_producer_number`, `email`, `phone` | Required for agent attribution. Always include on every enrollment — without it, the enrollment will not be properly associated with the agent/broker. |
| `attestations` | `broker_signature_attestation`, `agent_advised_consumer_of_product_features` | Agent compliance attestations. Recommended for all broker-assisted enrollments. |

## Desired Effective Date

`desired_effective_date` is optional. When omitted, the carrier determines the effective date automatically based on the SEP type and event date. This is the recommended approach.

When provided, the date must be one of the carrier's valid effective dates for the given SEP reason and event date. The valid dates are carrier-specific and computed from the carrier's enrollment rules. If the date is not valid, the API returns a 422 with either:
- The list of valid dates to choose from
- `"effective date selection is not available"` — meaning the carrier does not allow date selection for this SEP type (the date is auto-determined)

There is no way to query valid effective dates ahead of time. Omit this field unless your platform has a specific reason to override the carrier's default.

## Happy Path

**A plan must be selected before enrollment begins.** There is no valid "plan-less" enrollment. The enrollment form/page should only be reachable after plan selection and must always have `plan_hios_id` and `plan_year` populated.

1. `POST /quotes` with applicants (age, smoker, relationship) and `off_ex: true` — get plans with premiums and enrollment flags. Net out the ICHRA amount client-side for display.
2. Employee selects a plan (user clicks "Enroll" or "Select" on a specific plan card)
3. `GET /plans/:hios_id?plan_year=2026&include=enrollment_requirements` — get all carrier-specific enrollment requirements
4. Route based on flags (internal — not shown to user):

**If `api_enrollment: true` (EnrollConnect):**

5. `POST /applications` — create with complete payload (see Minimum Viable Payload above)
6. Check the `errors` array and route each prerequisite:
   - If it contains `supporting_documentation_required`, upload docs in step 7.
   - If it contains `missing_required_field` for `payment_method` and `payment_required_with_submission` is `true`, resolve payment in step 8.
   - For other errors, stop and correct the application before submit.
7. `POST /applications/:id/supporting_documentation` with the applicable `document_type` and file payload (if needed)
8. Read `payment_instructions` from the application response to determine payment timing:
   - If `payment_required_with_submission` is `true`, require and validate the `payment_method` action in `next_actions`. Its method must be `PUT`, and its relative `href` must match the current application. Send the validated action through the HealthSherpa-provided PCI proxy. Continue only after a `200` response. If the action is absent or invalid, stop and contact HealthSherpa.
   - Otherwise, proceed to submit first (step 9).
9. `POST /applications/:id/submit` — submit to carrier (returns 202 Accepted)
10. Handle post-submit payment:
    - If `payment_redirect_supported` is `true` → `GET /payment_redirect`, redirect user to carrier payment page
    - If `pay_by_phone_supported` is `true` → display `payment_phone_number` to user
    - If none of the above → carrier handles payment outside this flow (no action needed)
11. Poll `GET /applications/:id` with backoff or use webhooks. Continue monitoring through `sep_docs_under_review` and `pending_effectuation`, because both are intermediate states. Stop on lifecycle terminal states `effectuated`, `cancelled`, or `terminated`. Stop the current polling attempt at `submission_failed`, then remediate and resubmit rather than treating it as lifecycle terminal. A transient `draft` immediately after 202 means the background job is still running. If it remains `draft` beyond a one-hour submission grace period, alert and reconcile instead of resubmitting automatically.
12. Treat `effectuated` as immediate success. After `sep_docs_under_review` or `pending_effectuation`, continue monitoring until `effectuated`.
13. If `submission_failed`, check errors, fix the application or payment data, and re-submit.

**If `deeplink_enrollment: true` and `api_enrollment: false` (Deeplink):**

5. `POST /public/ichra/off_ex` — flat schema with `_agent_id` (required) and applicant data
6. Capture `Location` header from 302 response — redirect user's browser (do NOT follow server-side)
7. User completes enrollment in HealthSherpa UI
8. Track via webhooks (submission confirmation, then policy status changes)

## Anthem and Wellpoint In-Flow ACH

Anthem and Wellpoint applications currently use in-flow ACH when `payment_instructions.payment_required_with_submission` is `true`. Do not select this flow from the carrier name alone. Use the application response and its `next_actions`.

Before creating the application, follow the selected plan's enrollment requirements:

- For Anthem plans that return `communication_preferences.email_contact_consent`, display the full carrier text and submit the consumer's explicit boolean answer.
- For Wellpoint Texas plans that return communication-impairment requirements, submit the primary applicant's Yes or No answer and any conditionally required format fields. Include `attestations.disclosure_statement_accepted: true` when the Texas HMO consumer-choice disclosure is returned.

1. Create the application and retain its `application_id`.
2. Resolve any `supporting_documentation_required` error.
3. Confirm `payment_required_with_submission: true` and require `rel: "payment_method"` in `next_actions`. Require method `PUT` and an `href` matching the current application. If the action is absent or invalid, stop and contact HealthSherpa.
4. Build the ACH request at runtime and send it through the PCI-compliant proxy URL provided by HealthSherpa.
5. Require `200 OK` from `PUT /api/v1/applications/:id/payment_method`.
6. Call `POST /api/v1/applications/:id/submit`.

The current public integration supports `payment_method_type: "bank_account"`. For Anthem and Wellpoint:

- `payment_type: "initial"` covers the binder payment only.
- `payment_type: "both"` covers the binder payment and recurring premiums.
- The endpoint enum includes `payment_type: "recurring"`, but there is no current Anthem or Wellpoint use case for recurring without an initial payment; that request returns `422`. Use `"initial"` or `"both"`.
- `withdraw_day` applies to recurring ACH, accepts integers from 1 through 28, and defaults to `1` when omitted.
- For staging-only tests, the carrier-provided `PERSONALSAVINGS` fixture uses routing `071205850`, account `23487289374982`, `eft_type: "savings"`, and `eft_level: "personal"`. Synthetic account-holder names are accepted, but both `first_name` and `last_name` are required. Never use this fixture in production.

HealthSherpa transmits the bank and payment-selection data from the platform to Anthem or Wellpoint but does not initiate withdrawals. The carrier debits the first payment for `"initial"` and the first plus recurring monthly payments for `"both"`, using `withdraw_day` for the monthly schedule.

The successful response returns masked payment information, including `payment_method_type`, `payment_type`, `masked_account`, `status`, `is_recurring`, and `withdraw_day`. See [payment-and-documents.md](references/payment-and-documents.md) for the complete request and response.

## Enrollment Requirement Content

Call `GET /plans/:hios_id?plan_year=2026&include=enrollment_requirements` to retrieve carrier-specific legal text. **This text must be used as the actual attestation content shown to the user** — do NOT use generic labels like "I agree to the issuer attestations." The carrier-specific text is what was filed with the DOI and must be rendered as presented.

Response includes `enrollment_requirements.attestations` with keys like:
- `agrees_issuer_attestations` — general carrier attestation (render as consent checkbox with the carrier's exact text)
- `electronic_signature_consent` — e-signature consent (render as a consent prompt with the carrier's exact language)
- `broker_signature_attestation` — broker/agent consent
- `pediatric_dental` — pediatric dental attestation with `attestation_text`, `required`, and an `options` map
- `tx_hmo_consumer_choice_disclosure` — Texas HMO disclosure content and optional acceptance-checkbox label
- `state_supplement_primary_signature` — state-specific addendum (CO, UT, NJ)
- `state_supplement_spouse_signature` — spouse state addendum (UT)
- `state_supplement_disclosures_signature` — state disclosure (CO)

Keys that are absent or null are not required for that carrier/state. A present key can still contain `required: false`; follow the returned requiredness instead of relying on key presence alone.

**Implementation pattern:**
1. Fetch enrollment_requirements when user selects a plan (step 3 of Happy Path)
2. Render content-based boolean attestations as checkboxes with the returned `content`
3. When `pediatric_dental.required` is true, display `attestation_text`, render the `{value: label}` options as radio/select choices, and submit the selected key
4. Display a returned Texas HMO disclosure. When its `content_data.disclosure_statement_checkbox_label` is present, render that checkbox and submit `attestations.disclosure_statement_accepted: true`
5. Replace `%{signature_name}` placeholder with the applicant's full legal name
6. Submit only fields required by the returned metadata
7. If enrollment_requirements is unavailable, stop and retry or contact HealthSherpa; do not invent carrier legal text

## Signatures (Common Mistake)

Signatures are split across two locations in the payload:

- `signatures.signature_date` — the date of signing (ISO 8601). Required for submission.
- `applicants.primary.signature` — the primary applicant's typed signature. Required for submission.

These are NOT under the same object. Missing either causes a 422 on submit.

## Qualified Dental (HCSC)

Some carriers offer qualified dental and require the ACA pediatric dental essential health benefit to be satisfied on every application. Currently this applies to **HCSC** — the Blue Cross and Blue Shield plans in IL, MT, NM, OK, and TX. Apply the exact-one rule to HCSC plans even when `enrollment_requirements.attestations.pediatric_dental.required` is false; that flag does not expose the qualified-dental rule. Use the returned `attestation_text` and `options` when the consumer takes the attestation path.

**Satisfy it exactly one way per application — never both, never neither:**

1. **Attestation** — set `attestations.pediatric_dental` to one of the values from `enrollment_requirements.attestations.pediatric_dental.options` (currently `"purchased_separately"` or `"not_applicable"`). Use this when there are no children under 19 on the application, or the member already has stand-alone pediatric dental coverage.
2. **Stand-alone dental plan** — set the top-level `dental_plan_hios_id` to a qualified stand-alone dental plan's HIOS ID (and omit `attestations.pediatric_dental`). Use this when the member wants to buy pediatric dental coverage alongside the medical plan.

Sending both `dental_plan_hios_id` and `attestations.pediatric_dental`, or sending neither, returns a `422` with `Invalid dental selection`.

**Rules for `dental_plan_hios_id`:**

- The dental plan must belong to the **same carrier** as the medical plan, be off-exchange eligible for the plan year, and be available in the member's service area (zip). A mismatched carrier or ineligible/out-of-area plan returns `422` (`No plan found for dental_plan_hios_id: <id>`, or `dental_plan_hios_id has a carrier mismatch to the health plan`).
- Discover eligible dental plans with a **ZIP/FIPS-scoped dental quote**: `POST /quotes` with `dental_search: true` and the member's `zip_code`, `fip_code`, and `state`. Because it is scoped to the member's location, it returns only plans in their service area, so it is the reliable way to pick a plan that will pass create. Then confirm the chosen plan's issuer matches the medical plan's issuer: a `dental_search` quote can also return another carrier's dental plans, and `issuer_hios_ids` does not reliably restrict dental results. HCSC's stand-alone dental is "BlueCare Dental" under the same issuer prefix as the medical plan (e.g. `36096IL0830001` for BCBS IL). The `GET /plans?state=..&plan_year=..&dental_only=true&off_ex=true` list is state-level (useful for browsing or caching a catalog) but is **not** filtered by the member's service area, so a plan taken straight from it can still fail create with a `422` when it is not available at the member's ZIP. Validate service area with the dental quote before enrolling. See [quoting-and-plans.md](references/quoting-and-plans.md).
- On `PUT`, to switch from one path to the other, explicitly send `null` for the field you are clearing (e.g. set `dental_plan_hios_id: null` when moving to the attestation path).
- Both `dental_plan_hios_id` (top-level) and `pediatric_dental` are echoed back on `GET`/create responses, so you can confirm which path was recorded.

HCSC also does **not** support post-enrollment changes, cancellations, or renewals (`supports_changes` is `false`), requires SEP documentation for every SEP reason including `offered_ichra` (10 MB limit), and does not require SSN. See the HCSC entry in [carrier-examples.md](references/carrier-examples.md) for a full worked payload of both dental paths.

## Document Upload

Include `document_type` at the top level alongside the file. The API rejects uploads without it. Use `"sep"` for SEP documentation or `"proof_of_residency"` when the selected plan requires residency proof.

**JSON upload:**
```json
{
  "file": {
    "filename": "ichra_offering.pdf",
    "content_type": "application/pdf",
    "content_base64": "<base64-encoded>"
  },
  "document_type": "sep"
}
```

**Multipart upload:** `Content-Type: multipart/form-data` with `file` field and `document_type` field.

**Maximum file size is carrier-dependent.** Most carriers allow ~2 MB, but some are higher (currently Oscar 50 MB, HCSC 10 MB) and limits change over time. Do not assume a fixed cap — keep uploads small, and if a file is rejected for size, check the current limit for that carrier with your account manager.

## Create/Detail Response Structure

The `POST /applications` and `GET /applications/:id` responses nest the application data under an `application` key. Metadata fields are at the top level:

```json
{
  "application_id": "HSA000739499",
  "application": {
    "external_id": "your-tracking-id",
    "plan_hios_id": "13877AZ0070072",
    "plan_year": 2026,
    "policy_status": "draft",
    "applicants": { "primary": { ... }, "dependents": [...] },
    "residential_address": { ... },
    "attestations": { ... },
    "signatures": { ... },
    "special_enrollment_period": { ... },
    "hra": { ... },
    "policies": [],
    "submitted_at": null
  },
  "errors": [...],
  "next_actions": [...],
  "payment_instructions": { ... }
}
```

**You MUST flatten this.** A correct normalization:

```
const flat = {
  ...response.application,
  application_id: response.application_id,
  errors: response.errors,
  next_actions: response.next_actions,
  payment_instructions: response.payment_instructions,
};
```

## Submit Response

`POST /applications/:id/submit` returns **202 Accepted** with:

```json
{
  "application_id": "HSA000000001"
}
```

The 202 means the submission job was queued. It does not confirm carrier acceptance and does not return `policy_status`. An immediate `GET /applications/:id` can still return `draft` while the background job runs.

Poll `GET /applications/:id` with backoff or use webhooks until the application reaches:

- `effectuated`: coverage is already active. Treat this as immediate success even if no poll observed `pending_effectuation`.
- `sep_docs_under_review`: carrier review is in progress. Continue monitoring.
- `pending_effectuation`: carrier submission succeeded. `submitted_at` is populated.
- `submission_failed`: carrier submission failed. Read `errors`, correct the application or payment data, and resubmit.

Do not treat a transient `draft` immediately after 202 as success or failure. Bound this state with a submission grace period. If the application remains `draft` for more than one hour, alert and reconcile the queued job before considering another submit.

## Listing Applications

```
GET /api/v1/applications?policy_status=effectuated&limit=25&offset=0
```

| Parameter | Type | Description |
|---|---|---|
| `policy_status` | enum | `draft`, `submission_failed`, `sep_docs_required`, `sep_docs_under_review`, `pending_effectuation`, `effectuated`, `cancelled`, `terminated` |
| `external_id` | string | Filter by your tracking ID |
| `plan_year` | integer | Filter by plan year |
| `issuer_hios_id` | string | Filter by carrier |
| `plan_hios_id` | string | Filter by plan |
| `employer_external_id` | string | Filter by employer |
| `updated_since` | datetime | ISO 8601. Returns applications updated after this time. |
| `limit` | integer | Page size (default 25, max 100) |
| `offset` | integer | Offset for pagination (default 0) |

Response structure:

```json
{
  "applications": [
    {
      "application_id": "HSA000739499",
      "external_id": "your-tracking-id",
      "status": "draft",
      "plan_year": 2026,
      "plan_hios_id": "13877AZ0070072",
      "issuer_hios_id": "13877",
      "policy_status": "draft",
      "policy_effective_date": null,
      "state": "AZ",
      "primary_applicant": {
        "external_id": null,
        "member_id": "HSM001145562",
        "first_name": "Jane",
        "last_name": "Doe",
        "date_of_birth": "1990-05-15"
      },
      "created_at": "2026-04-28T13:56:56.119Z",
      "updated_at": "2026-04-28T13:56:56.838Z"
    }
  ],
  "pagination": {"total_count": 61, "limit": 25, "offset": 0}
}
```

Use `application_id` from each list item to link to the detail view (`GET /applications/:id`).

Note: list items use `primary_applicant` (flat object with `member_id`, `first_name`, `last_name`) — not the `applicants.primary` structure used in create/detail responses.

## Applicant Matching on PUT

- Include `member_id` (HealthSherpa-assigned) to update that applicant
- Omit `member_id` on a dependent to create a new dependent
- Omit an existing dependent from the array to remove them
- `external_id` can be set, changed, or cleared freely — it is never used for matching

## Post-Enrollment Changes

After an application has been submitted, partners can make changes to the enrollment via the API if the carrier supports it. This enables demographic corrections (e.g., email, phone, address) and, during Open Enrollment, plan changes.

### Checking Support

The `GET /applications/:id` response includes:

| Field | Type | Description |
|---|---|---|
| `supports_changes` | boolean | Whether the carrier supports any post-enrollment changes for this application |
| `can_change_plan` | boolean | Whether a plan change is currently allowed (requires Open Enrollment and carrier support) |
| `can_report_change` | boolean | Whether demographic changes can be submitted |

Additionally, `next_actions` will include a `change_plan` action when plan changes are available.

### Change Workflow

1. `GET /applications/:id` — confirm `supports_changes` is `true`
2. `PUT /applications/:id` — send the full application payload with changes applied. All identity fields (`first_name`, `last_name`, `date_of_birth`) must be included even if unchanged, as omitted fields are treated as `null`.
3. `POST /applications/:id/submit` — resubmit to apply the changes at the carrier

### Identity Protection

To prevent accidental identity swaps, the API blocks certain simultaneous changes on submitted applications:
- Changing both `first_name` and `last_name` at the same time is rejected
- Changing `date_of_birth` together with a name change is rejected
- Individual corrections (e.g., fixing a typo in `last_name` alone) are allowed

### Carriers Without Change Support

Some carriers do not support post-enrollment changes via the API. For these carriers, `supports_changes` will be `false` and update attempts on submitted applications will return a `422` error. Partners should direct members to contact the carrier directly for changes on these enrollments.

## Error Format

All endpoints return errors in a single format:

```json
{"errors": [{"code": "missing_required_field", "field": "applicants.primary.date_of_birth", "message": "Date of birth is required"}]}
```

Create (201) and GET (200) include `errors` even on success for informational items like `supporting_documentation_required`.

## HATEOAS

Every application response includes `next_actions`. Use it to determine available operations:

```json
"next_actions": [
  {"rel": "self", "href": "/api/v1/applications/HSA000000001", "method": "GET"},
  {"rel": "submit", "href": "/api/v1/applications/HSA000000001/submit", "method": "POST"}
]
```

The array is state-aware. After submission, `submit` disappears and `cancel` appears. Use `rel` values to drive your UI rather than hardcoding status-to-action mappings.

## Enrollment Decision Path

`api_enrollment` and `deeplink_enrollment` are **internal routing flags** — they determine how your backend processes enrollment. **NEVER expose these labels to end users.** Users should see a single "Enroll" or "Select Plan" button. Your backend uses the flags to decide _how_ to enroll.

| `api_enrollment` | `deeplink_enrollment` | Backend Routing |
|---|---|---|
| `true` | `true` | Prefer EnrollConnect (full lifecycle). Fall back to deeplink. |
| `true` | `false` | EnrollConnect only. |
| `false` | `true` | Deeplink only. Redirect user to HealthSherpa UI. |
| `false` | `false` | Not enrollable through HealthSherpa. Hide or disable the enroll button. |

These flags are determined per carrier and state. They appear on:
- Each plan in the `POST /quotes` response
- The `GET /plans/:hios_id` response

Always check these flags before routing. If `api_enrollment` is `false` for a plan, `POST /applications` will reject the request with a 422.

**IMPORTANT: You MUST include `off_ex: true` in every quoting request.** The API only supports off-exchange enrollment. Without this parameter, the API returns on-exchange plans which are not supported.

## Carrier-Specific Behavior

- Some carriers reject specific SEP reasons — the API returns a 422 with a descriptive error
- Some carriers do not support cancellations — `cancel` will not appear in `next_actions`
- Carrier-specific fields may cause 422 on create if missing

Always check the `errors` array on create and the `next_actions` on GET to understand what's available.

## Rate Limits

429 response includes `retry_after` (seconds).

| Category | Limit |
|---|---|
| Quoting (`POST /quotes`) | 3,000/min |
| Mutations (POST/PUT) | 600/min |
| Reads (GET) | 1,000/min |

## Reference Files

Read these as needed for detailed schemas and implementation guidance:

- [data-model.md](references/data-model.md) — Complete request/response schema with all fields, types, attestations, and signatures
- [quoting-and-plans.md](references/quoting-and-plans.md) — Quoting request format, plan response, enrollment routing logic
- [deeplink-enrollment.md](references/deeplink-enrollment.md) — Deeplink schema, 302 handling, field mapping from canonical to flat format
- [payment-and-documents.md](references/payment-and-documents.md) — Payment decision tree, redirect flow, document upload details
- [webhooks-and-monitoring.md](references/webhooks-and-monitoring.md) — Webhook payloads, polling intervals, reconciliation patterns
- [carrier-examples.md](references/carrier-examples.md) — Ready-to-send create + submit example payloads per carrier, the document-required vs. straight-through split, and state-supplement requirements
