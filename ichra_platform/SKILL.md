---
name: ichra-platform-integration
description: Build ICHRA platforms that integrate with HealthSherpa APIs for plan quoting, enrollment, payment, and policy management. Use when building or debugging an ICHRA platform integration, working with HealthSherpa enrollment or quoting APIs, or when the user mentions ICHRA, HRA, enrollment, quoting, plan selection, carrier submission, or policy status.
metadata:
  author: healthsherpa
  version: "1.1"
---

# ICHRA Platform Integration with HealthSherpa

## Environments

| Environment | Base URL | Notes |
|---|---|---|
| Production | `https://api.ichra.healthsherpa.com` | Primary. Use for all live integrations. |
| Staging | `https://api.ichra-staging.healthsherpa.com` | For testing only. May have intermittent backend issues. |

All EnrollConnect and Quoting endpoints use these base URLs. Deeplinks use different hosts (see [deeplink-enrollment.md](references/deeplink-enrollment.md)).

## Authentication

QuoteConnect, EnrollConnect, and Deeplink requests require an `x-api-key` provided during onboarding.

```
x-api-key: your_api_key_here
Content-Type: application/json
```

Deeplink staging requests also require Basic Auth credentials provided during onboarding. The API key associates the hosted application with the platform so it can later be accessed through EnrollConnect. All POST and PUT requests must set `Content-Type: application/json` except multipart file uploads. These APIs do not use OAuth, session authentication, or HMAC request signing.

## Security & Credentials

Keep API keys, Basic Auth credentials, webhook credentials, and trusted proxy or redirect configuration in server-side secret/configuration storage. Fail closed during startup or request preflight when required credentials, payment-proxy configuration, or payment-redirect origins are missing. Rotate credentials after suspected compromise and contact the HealthSherpa account manager for replacements. An API key authenticates the platform, not its end user. Before create or Deeplink, authorize the current user for the tenant, employer, household, selected plan, and enrollment action; bind the returned `application_id` to that scope. Before later calls, authorize the user for the specific application and action. Treat application, HRA, payment, and document payloads as sensitive. Never log full bodies; redact identity, signature, address, employer, income, and bank fields. Authenticate webhooks before processing and deduplicate by `transaction_id`. Require HTTPS and TLS verification. Integrators must determine and meet the HIPAA, state privacy, and CMS obligations applicable to their role, data, and workflow. Validate and escape payment redirects as described in [payment-and-documents.md](references/payment-and-documents.md). Do not retry deterministic 4xx responses; honor `retry_after` for 429 responses.

## Critical Rules

- ALWAYS use `x-api-key` for QuoteConnect, EnrollConnect, and Deeplink. Staging Deeplink requests also require Basic Auth.
- ALWAYS call HealthSherpa endpoints from server-side code. NEVER embed the API key, webhook secret, or staging Basic Auth credentials in browser, mobile, or any client-distributed code.
- ALWAYS load the API key from an environment variable or secret manager. NEVER hardcode it in source code or commit it to version control.
- Before EnrollConnect create or Deeplink, authorize the current user for the source tenant, employer, household, selected plan, and action. Bind any returned `application_id` to that authorized scope.
- ALWAYS authorize the current platform user for the specific application and requested action before an application-scoped API call. A partner API key alone is not end-user authorization.
- ALWAYS use HTTPS. NEVER disable TLS certificate verification to work around connection errors.
- For Deeplink, require a 302 response, parse `Location`, require HTTPS and the exact HealthSherpa origin for the environment, reject userinfo and unexpected ports, and reject every other status or destination.
- Treat all response fields as untrusted at their destination. Parameterize SQL, avoid shell interpolation by passing values as separate arguments, allowlist file-path components, and escape values before rendering HTML.
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
- Use `gender` for applicant identity and `uses_tobacco` for tobacco status. The responsible-party object uses its documented `sex` field.
- ALWAYS use `state_supplement_*` for state signature fields (not `addendum_*`).
- In the request, `pediatric_dental` (string enum: `"purchased_separately"` / `"not_applicable"`) goes under `attestations`. Read the saved value from `response.application.pediatric_dental`. `pediatric_dental_signature` goes under `signatures`.
- For carriers that offer qualified dental (currently HCSC — Blue Cross and Blue Shield of IL, MT, NM, OK, and TX), the ACA pediatric dental requirement must be satisfied **exactly one way** per application: send `attestations.pediatric_dental` OR a top-level `dental_plan_hios_id`, never both and never neither. Sending both, or neither, returns a 422 (`Invalid dental selection`). See "Qualified Dental (HCSC)".
- ALWAYS include top-level `hra` with employer `name`, `fein`, and `address` on every ICHRA enrollment. When `enrollment_requirements.hra.per_applicant_hra` is true, also collect and send `hra` under the primary and every non-child dependent relationship supported by the selected carrier, using the returned allowed values. Do not collect per-applicant HRA answers from children. Send the household follow-up at top-level `qsehra_both_employers_claim_reimbursement` when its returned condition applies.
- NEVER require `external_id`. It is optional everywhere.
- NEVER assume real-time payment confirmation. Most carriers report asynchronously via feeds.
- ALWAYS fetch `enrollment_requirements` via `GET /plans/:hios_id?plan_year=YYYY&include=enrollment_requirements`. Process returned `attestations`, `special_enrollment_period.event_types`, applicant questions, communication preferences, HRA requirements, and proof-of-residency instructions. Use the returned questions, content, labels, options, date windows, and conditional requiredness. Omitted keys do not apply to the selected plan.
- When `communication_preferences.email_contact_consent.required` is true, show the returned carrier content and collect an explicit Yes or No. Send the boolean at `communication_preferences.email_contact_consent`; both `true` and `false` satisfy requiredness.
- When `communication_preferences.agrees_hsa_contact_opt_in` is an object, show its returned carrier question, content, and options. If `required` is true, require an explicit `true` or `false`; omission or null is a blocking error. If `required` is false, the answer is optional. Send any answer at `communication_preferences.agrees_hsa_contact_opt_in`. A false answer satisfies requiredness and does not block enrollment in the health plan. If the requirement or its parent is absent or null, do not show or send the field. Reject non-boolean answers.
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

Use when `deeplink_enrollment: true` and the platform selects Deeplink before any EnrollConnect create request. When both flags are true, prefer EnrollConnect. Deeplink redirects the user to HealthSherpa UI.

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

The API expects the full application shape on create. Build the complete payload before calling `POST /applications`. A structurally invalid request can return `422`; a `201` means the draft exists but can still include blocking errors and is not ready to submit until they are resolved.

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

**If Deeplink is selected before create and `deeplink_enrollment: true`:**

5. `POST /public/ichra/off_ex` — flat schema with `_agent_id` (required) and applicant data
6. Capture `Location` header from 302 response — redirect user's browser (do NOT follow server-side)
7. User completes enrollment in HealthSherpa UI
8. Track via webhooks. Use the submission webhook's `application_id` with the same platform API key for later EnrollConnect reads.

## Anthem and Wellpoint In-Flow ACH

Use in-flow ACH only when `payment_instructions.payment_required_with_submission` is true and a valid `payment_method` action is returned. See [payment-and-documents.md](references/payment-and-documents.md) for request validation, payment types, staging fixtures, and response handling. See [carrier-examples.md](references/carrier-examples.md) for Anthem and Wellpoint enrollment fields.

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

HCSC medical applications in IL, MT, NM, OK, and TX must include exactly one of `attestations.pediatric_dental` or top-level `dental_plan_hios_id`. Both or neither returns `422`. A selected dental plan must share the medical carrier and be available in the member's service area. Confirm the saved path from `response.application.pediatric_dental` and `response.application.dental_plan_hios_id`. When switching paths on PUT, send `null` for the field being cleared. See [data-model.md](references/data-model.md), [quoting-and-plans.md](references/quoting-and-plans.md), and [carrier-examples.md](references/carrier-examples.md).

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

Create and detail responses put application data under `application`, while `application_id`, `errors`, `next_actions`, and `payment_instructions` remain top-level. Flatten both levels in the client. See [data-model.md](references/data-model.md).

## Submit Response

`POST /applications/:id/submit` returns **202 Accepted** with:

```json
{
  "application_id": "HSA000000001"
}
```

The 202 means the submission job was queued. It does not confirm carrier acceptance and does not return `policy_status`. An immediate `GET /applications/:id` can still return `draft` while the background job runs.

Poll `GET /applications/:id` with backoff or use webhooks after submission:

- `effectuated`: coverage is already active. Treat this as immediate success even if no poll observed `pending_effectuation`.
- `sep_docs_under_review`: carrier review is in progress. Continue monitoring.
- `pending_effectuation`: carrier submission succeeded. `submitted_at` is populated.
- `submission_failed`: carrier submission failed. Read `errors`, correct the application or payment data, and resubmit.

Do not treat a transient `draft` immediately after 202 as success or failure. Bound this state with a submission grace period. If the application remains `draft` for more than one hour, alert and reconcile the queued job before considering another submit.

API responses establish only the reported API operation and status. A 202 confirms queueing, and `sep_docs_under_review` or `pending_effectuation` is not payment, carrier acceptance, or effectuation confirmation. Do not claim an 834, SFTP transfer, application PDF, or carrier acknowledgement unless that downstream event is observed separately.

## Listing Applications

`GET /api/v1/applications` supports status, external ID, plan, issuer, employer, update time, limit, and offset filters. List items use `primary_applicant`, not `applicants.primary`; pagination uses `total_count`. Use each `application_id` to fetch detail. See [data-model.md](references/data-model.md).

## Applicant Matching on PUT

- Include `member_id` (HealthSherpa-assigned) to update that applicant
- Omit `member_id` on a dependent to create a new dependent
- Omit an existing dependent from the array to remove them
- `external_id` can be set, changed, or cleared freely — it is never used for matching

## Post-Enrollment Changes

For a draft, require the validated `update` action and send the full PUT payload; do not submit unless submission was requested. For a submitted demographic change, require `supports_changes`, `can_report_change`, and a `next_actions` entry with `rel: "update"`, method `PUT`, and href `/api/v1/applications/{application_id}`. For a plan change, require `supports_changes`, `can_change_plan`, and the same validated update action. Send the full payload with all identity fields, then require the returned submit action. Call cancel or terminate only through its matching action. If support is false, direct the member to the carrier. See [data-model.md](references/data-model.md).

## Error Format

All endpoints return errors in a single format:

```json
{"errors": [{"code": "missing_required_field", "field": "applicants.primary.date_of_birth", "message": "Date of birth is required"}]}
```

Create (201) and GET (200) can include prerequisites or blocking validation errors. A 201 means the draft exists, not that it is ready to submit. Retain its `application_id`, route `supporting_documentation_required` to document upload, and correct other errors on that draft with a full PUT. Do not retry create.

## HATEOAS

Every application response includes `next_actions`. Use it to determine available operations:

```json
"next_actions": [
  {"rel": "_self", "href": "/api/v1/applications/HSA000000001", "method": "GET"},
  {"rel": "submit", "href": "/api/v1/applications/HSA000000001/submit", "method": "POST"}
]
```

The array is state-aware. After submission, `submit` disappears; `cancel` or `terminate` appears only when the carrier and application state support that action. Use `rel` values to drive your UI rather than hardcoding status-to-action mappings.

## Enrollment Decision Path

`api_enrollment` and `deeplink_enrollment` are **internal routing flags** — they determine how your backend processes enrollment. **NEVER expose these labels to end users.** Users should see a single "Enroll" or "Select Plan" button. Your backend uses the flags to decide _how_ to enroll.

| `api_enrollment` | `deeplink_enrollment` | Backend Routing |
|---|---|---|
| `true` | `true` | Prefer EnrollConnect. Choose Deeplink only before creating an EnrollConnect application. |
| `true` | `false` | EnrollConnect only. |
| `false` | `true` | Deeplink only. Redirect user to HealthSherpa UI. |
| `false` | `false` | Not enrollable through HealthSherpa. Hide or disable the enroll button. |

These flags are determined per carrier and state. They appear on:
- Each plan in the `POST /quotes` response
- The `GET /plans/:hios_id` response

Always check these flags before routing. If `api_enrollment` is `false` for a plan, `POST /applications` will reject the request with a 422.

**IMPORTANT: You MUST include `off_ex: true` in every quoting request.** The API only supports off-exchange enrollment. Without this parameter, the API returns on-exchange plans which are not supported.

Never retry create or switch to Deeplink automatically after a timeout or ambiguous EnrollConnect create response. If transport-level evidence proves the request was never sent, a new attempt is safe. Otherwise use a bounded reconciliation period and manual intervention; a negative lookup is not proof that an asynchronous create did not occur.

## Carrier-Specific Behavior

- Some carriers reject specific SEP reasons — the API returns a 422 with a descriptive error
- Some carriers do not support cancellations — `cancel` will not appear in `next_actions`
- Carrier-specific fields may cause 422 on create if missing

Always check the `errors` array on create and the `next_actions` on GET to understand what's available.

### BCBS Michigan Individual Medical

Use issuer `15560` for Blue Cross Blue Shield of Michigan PPO and issuer `98185` for Blue Care Network HMO. Quote current inventory and check each plan's enrollment flags; do not hardcode example plan IDs.

- Fetch plan requirements. For plan year 2026, the returned options allow `spouse` and `child`, require race/ethnicity for every applicant, and require SEP documentation for every returned SEP reason.
- Send one valid `ssn` or `itin` for every applicant. Send explicit `us_citizen`, `resides_in_state`, and `uses_tobacco` booleans. The primary also needs `phone_type`: `home`, `cell`, or `work`.
- Send top-level `hra.offered_hra` and the complete HRA and employer data. Both ICHRA and QSEHRA are supported.
- When plan requirements return `communication_preferences.agrees_hsa_contact_opt_in`, display the exact returned BCBSM or BCN question, content, and options and follow its `required` value. BCBSM asks whether the consumer wants to enroll in a HealthEquity HSA. Preserve either boolean answer at `communication_preferences.agrees_hsa_contact_opt_in`; do not use `hsa_contact_opt_in` or interpret false as missing.
- Treat applicant `member_id` as opaque. Store and return the exact value on draft PUT; never construct one or match applicants by array position.
- Follow `payment_instructions`. BCBSM and BCN use post-submit payment redirect and phone payment, with no payment method required before submit.
- Under the 2026 contract, BCBSM and BCN do not return post-submit update, cancel, or terminate actions. Always check current response capabilities and direct members to the carrier when an action is absent.
- BCBS Michigan is not HCSC. Do not apply HCSC qualified-dental rules to these Michigan medical plans.

See [carrier-examples.md](references/carrier-examples.md) for a complete request and [payment-and-documents.md](references/payment-and-documents.md) for the redirect flow.

### AmeriHealth Caritas Next

Quote current inventory rather than hardcoding plans. AmeriHealth Caritas Next issuer IDs are `72760` in Delaware, `67926` in Florida, `38246` in Louisiana, `17414` in North Carolina, and `73107` in South Carolina. South Carolina plans use the First Choice Next consumer brand.

- Under the 2026 application contract, require an SSN for each applicant age one or older on the request date. An applicant younger than one may omit it.
- Send explicit citizenship and state-residency answers. A false answer makes the applicant ineligible for the plan.
- The 2026 primary-applicant fields include `marital_status`, `language_spoken`, and `language_written`. Follow returned options; the marital-status values are `married`, `unmarried`, `divorced`, and `widowed`. Use `cell` or `work` for `phone_type`; `home` is not accepted.
- Use the selected plan's returned relationship options. The 2026 medical options are `spouse` and `child`.
- Render the returned responsible-party question for a child-only application. When the consumer identifies another person as responsible for payment, send `applicants.primary.responsible_party` using the returned field-level conditions and relationship options. Otherwise omit the object. For a minor primary applicant, include guardian details and set `applicants.primary.signature` to the guardian's exact full name.
- Follow each SEP event's `documentation_required` value. For 2026 AmeriHealth Caritas Next plans, the returned events require `document_type: "sep"` before submit.
- Drive the HSA question only from `enrollment_requirements`. For 2026 AmeriHealth Caritas Next plans, `agrees_hsa_contact_opt_in` is absent even when `hsa_eligible` is true, so do not render or send it.
- In Louisiana, render the returned out-of-network disclosure. For a broker-assisted North Carolina application, render and collect the returned NC licensed-agent attestation.
- Do not apply HCSC qualified-dental rules. Follow the returned pediatric-dental requirement.

See [carrier-examples.md](references/carrier-examples.md) for a complete request.

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
