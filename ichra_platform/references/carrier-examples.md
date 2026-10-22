# EnrollConnect API — Example Create & Submit Requests by Carrier

_Reference examples for building successful off-exchange ICHRA enrollments through the EnrollConnect API. Each example illustrates the target carrier's specific validations; section-specific notes state the staging validation date and boundary._

> **Synthetic data only.** Every value below (names, SSN, FEIN, phone, email, agent NPN) is a placeholder. Replace all applicant, employer, and agent values with real data before sending. Never commit real PII/PHI to source control or logs.

## Before you start

> **Baseline create validation, PY2026, 2026-06-10.** The create payloads below were validated against HealthSherpa staging on this date. Later section-specific notes identify newer validation and the observed status boundary. A 202 submit response only confirms the background job was queued. Continue monitoring through intermediate states until a lifecycle terminal status. If the application reaches `submission_failed`, stop the current polling attempt, remediate the reported errors, and resubmit. Plan IDs, ZIP, and FIPS are plan-year- and inventory-specific and rotate each year. Treat them as illustrative and always quote for current inventory. Carrier documentation requirements are a point-in-time snapshot; the create response is the source of truth.

- **Server-side only.** Call EnrollConnect from a backend you control. Never embed the `x-api-key` in browser or mobile code.
- **Base URL (production):** `https://api.ichra.healthsherpa.com`  •  **Staging:** `https://api.ichra-staging.healthsherpa.com`
- **Auth:** send `x-api-key: <your key>` and `Content-Type: application/json` on every request.
- **Quote first.** The plan IDs, ZIP, and FIPS below are illustrative 2026 examples. In production, call `POST /api/v1/quotes` (with `off_ex: true`) to get a current `plan_hios_id` for the member's county, and confirm `api_enrollment: true` on the plan before creating an application.
- **Fetch requirements.** Call `GET /api/v1/plans/{hios_id}?plan_year=2026&include=enrollment_requirements` to get the carrier's exact attestation text and SEP rules. Render the returned legal text to the consumer — do not use generic labels.
- **`external_id` is not de-duplicated.** Submitting two creates with the same `external_id` produces two separate applications. Store the `application_id` returned by create and use it for every follow-up call. After an ambiguous create response, retry only when transport-level evidence proves the request was never sent; otherwise use bounded reconciliation and manual intervention.
- **Use real, current dates.** For the selected SEP reason, the event date may be up to `event_date_days_before` days before today or `event_date_days_after` days after today. The dates in these examples are illustrative — replace them with the member's actual event and signature dates.

## Conventions used in every example (best practices)

- **Request enums are lowercase snake_case**; dates are ISO 8601 (`YYYY-MM-DD`); premium fields in responses and HRA amount and income fields in requests are JSON numbers.
- **Field names:** `street_address_1`/`street_address_2` (not `street_line_*`), applicant `gender` (not `sex`), and `uses_tobacco` (not `tobacco_use`). The `responsible_party` sub-object is the exception and uses `sex`.
- **Signatures are split** — `applicants.primary.signature` (typed legal name) **and** `signatures.signature_date`. Both are required to submit.
- **Always include `hra`** with employer `name`, `fein`, and nested `address`, plus an `agent_of_record` (`first_name`, `last_name`, `national_producer_number`).
- **SEP:** these examples use `event_type: "offered_ichra"` (the ICHRA offer), the canonical reason for ICHRA enrollment. Send the canonical value; HealthSherpa maps it to the carrier's format.
- **`desired_effective_date` is omitted** so the carrier auto-determines the effective date (recommended).
- **`external_id` is optional and appears in two places** — at the top level (your application tracking id) and on each applicant (your member/person id). Both are correlation metadata echoed back on reads; neither is used to match or de-duplicate records. Use them to map HealthSherpa applications and members back to your own system.
- **SSN is carrier-specific.** Some carriers require the primary's SSN to submit; others do not. Each carrier section below states which. When required, send a valid SSN (or ITIN where the carrier accepts one).
- **These payloads carry PII/PHI.** SSN, DOB, signatures, addresses, and the other identity fields below are sensitive data — securing, storing, and redacting them in your own systems is your responsibility, not HealthSherpa's. See **Security & Credentials** in `SKILL.md`.

## Quote for a current plan

Quote first to get a `hios_id` valid for the member's county and to confirm `api_enrollment: true`. The quote applicant shape differs from the application payload: it uses `age`, `smoker`, and `relationship` (and the location keys are `zip_code` + `fip_code`).

```bash
curl -sS -X POST 'https://api.ichra.healthsherpa.com/api/v1/quotes' \
  -H "x-api-key: $HS_API_KEY" -H 'Content-Type: application/json' \
  -d '{
    "off_ex": true,
    "plan_year": 2026,
    "zip_code": "85001",
    "fip_code": "04013",
    "applicants": [
      { "age": 36, "smoker": false, "relationship": "primary" }
    ]
  }'
```

Each returned plan includes `hios_id` plus `api_enrollment` and `deeplink_enrollment` flags. Use the plan's `hios_id` as `plan_hios_id` when you create the application. Enroll through the API only when `api_enrollment: true`; when it is `false`, route the enrollment through Deeplink.

## The enrollment flow

**1. Create:** `POST /api/v1/applications` with the carrier payload below. A `201` creates the draft, but it can include prerequisites in `errors`. An empty array means it is ready for submit. Route `supporting_documentation_required` to the document step and `missing_required_field` for `payment_method` or the required HSA contact answer to its correction step. Stop for other errors.

```bash
curl -sS -X POST 'https://api.ichra.healthsherpa.com/api/v1/applications' \
  -H "x-api-key: $HS_API_KEY" -H 'Content-Type: application/json' \
  -d @create_payload.json
```

**2. Upload available documentation** — `POST /api/v1/applications/{application_id}/supporting_documentation`. Use `document_type: "sep"` for qualifying-event proof or `"proof_of_residency"` when plan requirements indicate residency proof is required. When the selected event returns `documentation_required: true`, upload before submit. When it returns `false`, submission is not blocked, but that does not mean proof will never be needed. Encourage upload when evidence is available and follow later document actions or carrier requests. **Maximum file size is carrier-dependent** — most carriers allow ~2 MB, but some are higher (currently Oscar 50 MB, HCSC 10 MB) and limits change over time. Keep uploads as small as possible; if a file is rejected for size, check the current limit for that carrier with your account manager.

```bash
curl -sS -X POST 'https://api.ichra.healthsherpa.com/api/v1/applications/{application_id}/supporting_documentation' \
  -H "x-api-key: $HS_API_KEY" -H 'Content-Type: application/json' \
  -d '{"file":{"filename":"sep_proof.pdf","content_type":"application/pdf","content_base64":"<base64 of the file>"},"document_type":"sep"}'
```

**Before submit, check payment instructions.** Read `payment_instructions` and `next_actions` from the create or GET response. When `payment_required_with_submission` is `true`, require the `payment_method` next action. Its method must be `PUT`, and its relative `href` must match the current application. Send the validated `href` through the PCI-compliant proxy URL provided by HealthSherpa and require `200 OK` before submit. See [payment-and-documents.md](payment-and-documents.md).

**3. Submit:** `POST /api/v1/applications/{application_id}/submit` (no body). Call this only after any required document and payment method have been provided. A `202 Accepted` response contains `application_id` and means the background submission job was queued.

```bash
curl -sS -X POST 'https://api.ichra.healthsherpa.com/api/v1/applications/{application_id}/submit' \
  -H "x-api-key: $HS_API_KEY" -H 'Content-Type: application/json'
```

**4. Track:** poll `GET /api/v1/applications/{application_id}` with backoff or use webhooks. Treat `effectuated` as success even if no poll observed `pending_effectuation`. `sep_docs_under_review` and `pending_effectuation` are intermediate states. If the application reaches `submission_failed`, stop the current polling attempt, read `errors`, correct the application or payment data, and resubmit. An immediate `draft` after 202 is transient while the background job runs; if it remains `draft` for more than one hour, alert and reconcile instead of resubmitting automatically.

---

## Carrier examples

The carrier sections below provide either a complete create payload or the exact carrier-specific fields to apply to the complete request shape in [data-model.md](data-model.md). Fetch `enrollment_requirements` for the selected plan, require `api_enrollment: true`, and then follow the document, payment, and submit steps above.

### Avera Health Plans

> **Validated in HealthSherpa staging on September 17, 2026.** The current examples are PY2026 South Dakota plans under issuer `60536`. Quote for current inventory and confirm the plan's enrollment flags rather than hardcoding these IDs or assuming another Avera market is available.

- **Non-HSA example plan:** `60536SD0060034`, Sioux Falls ZIP `57104`, FIPS `46099`.
- **HSA example plan:** `60536SD0060021`.
- **Enrollment routing:** Both tested plans returned `api_enrollment: true` and `deeplink_enrollment: true`. Prefer EnrollConnect unless Deeplink is selected before create.
- **Identity:** Every applicant age one or older needs either `ssn` or `itin`.
- **Applicant answers:** Send `uses_tobacco` and `race_ethnicity` for every applicant. Send `language_spoken` and `language_written` for the primary. Citizenship and state-residency answers are not collected.
- **Contact and household:** Use a physical residential address, not a PO box. `phone_type` accepts `home`, `work`, or `cell`. `marital_status` is optional with `single` and `married` as current values.
- **Dependents:** Current relationships are `spouse`, `domestic_partner`, and `child`. Every dependent needs `has_disability`; a child also needs `has_qualified_medical_child_support_order`.
- **Age exception:** A non-disabled child dependent can enroll through age 25. A non-disabled full-time student can enroll through age 28 by sending `full_time_student: true`; Avera does not require `graduation_date` for this exception.
- **HRA:** Always send the complete top-level HRA, employer FEIN and address, `premium_payer`, `household_size`, `annual_household_income`, and `annual_household_income_determination`. Use `w2_wages`, `rate_of_pay_hourly`, or `fpl` for the determination. When `premium_payer` is `tpa`, send the HealthSherpa-assigned identifier at application top-level `tpa_slug` and any direct contact values under `hra` using `tpa_name`, `tpa_phone_number`, `tpa_street_address`, `tpa_street_address_unit_number`, `tpa_city`, `tpa_state`, `tpa_zip_code`, and `tpa_county`. Current plan requirements return `hra: null`, so do not interpret absent HRA metadata as permission to omit these fields.
- **SEP documents:** Current returned reasons use `documentation_required: false`. This means proof does not block initial submission. Encourage applicants to upload available evidence, display the returned optional-upload notice, and offer the supporting-documentation action until 30 days after application submission.
- **HSA question:** The tested HSA plan returned a required Fidelity contact-consent question. When this requirement is returned, send the explicit boolean at `communication_preferences.agrees_hsa_contact_opt_in`; `false` is a valid Opt-out and does not block enrollment.
- **Payment:** Read `payment_instructions` before submit. Current responses do not require a payment method before submit and provide payment redirect and pay-by-phone afterward. Follow a future response if that changes.
- **Agent:** Send the standard `agent_of_record` with NPN. Do not add a carrier producer code or broker license ID.
- **Lifecycle:** Current applications advertise update, supporting-documentation, payment-redirect, and cancellation actions after submission. Use the current response before every change, cancellation, or termination.

The standard non-HSA subscriber request is:

```json
{
  "external_id": "your-tracking-id-001",
  "plan_hios_id": "60536SD0060034",
  "plan_year": 2026,
  "residential_address": {
    "street_address_1": "100 Test Avenue",
    "city": "Sioux Falls",
    "state": "SD",
    "zip_code": "57104",
    "fips_code": "46099"
  },
  "mailing_address": {
    "street_address_1": "100 Test Avenue",
    "city": "Sioux Falls",
    "state": "SD",
    "zip_code": "57104",
    "fips_code": "46099",
    "different_from_home_address": false,
    "billing_use_only": false
  },
  "applicants": {
    "primary": {
      "external_id": "member-001",
      "first_name": "Jordan",
      "last_name": "Example",
      "date_of_birth": "1988-01-15",
      "gender": "male",
      "itin": "<9-digit-itin>",
      "email": "jordan.example@example.com",
      "phone": "6055550100",
      "phone_type": "cell",
      "marital_status": "single",
      "uses_tobacco": false,
      "race_ethnicity": "white",
      "language_spoken": "english",
      "language_written": "english",
      "existing_coverage": {
        "has_existing_coverage": false
      },
      "signature": "Jordan Example"
    },
    "dependents": []
  },
  "agent_of_record": {
    "first_name": "Pat",
    "last_name": "Broker",
    "national_producer_number": "<agent-npn>",
    "email": "agent@example.com",
    "phone": "6055550101"
  },
  "hra": {
    "offered_hra": true,
    "type": "ichra",
    "amount": 500,
    "contribution_covers": "premium",
    "start": "2026-10-01",
    "employer": {
      "name": "Example Employer LLC",
      "fein": "<employer-fein>",
      "phone": "6055550199",
      "address": {
        "street_address_1": "200 Test Street",
        "city": "Sioux Falls",
        "state": "SD",
        "zip_code": "57104",
        "fips_code": "46099"
      }
    },
    "premium_payer": "employer",
    "household_size": 1,
    "annual_household_income": 60000,
    "annual_household_income_determination": "w2_wages"
  },
  "special_enrollment_period": {
    "event_type": "offered_ichra",
    "event_date": "2026-09-17"
  },
  "attestations": {
    "agrees_issuer_attestations": true,
    "electronic_signature_consent": true,
    "broker_signature_attestation": true,
    "agent_advised_consumer_of_product_features": true
  },
  "signatures": {
    "signature_date": "2026-09-17"
  },
  "communication_preferences": {}
}
```

When a current quoted Avera HSA plan returns `agrees_hsa_contact_opt_in`, replace the plan ID and add the returned required answer:

```json
{
  "communication_preferences": {
    "agrees_hsa_contact_opt_in": false
  }
}
```

For a child dependent, merge the current required dependent fields. Use a unique SSN or ITIN for each applicant:

```json
{
  "relationship": "child",
  "first_name": "Sam",
  "last_name": "Example",
  "date_of_birth": "2015-03-10",
  "gender": "female",
  "ssn": "<9-digit-ssn>",
  "uses_tobacco": false,
  "race_ethnicity": "decline_to_answer",
  "has_disability": false,
  "has_qualified_medical_child_support_order": false
}
```

For a non-disabled child age 26 through 28, also send `full_time_student: true`. Do not send `graduation_date` for the Avera student exception. A non-disabled non-student in that range and any non-disabled child age 29 or older are ineligible.

For child-only coverage, render `enrollment_requirements.applicants.primary.responsible_party`. If another person is responsible for payment, send that object with one of the returned relationships, currently `parent_guardian` or `sibling`. Do not place HRA employer data in `responsible_party`.

The September 17 smoke tests created standard and HSA applications without errors, returned `202` from submit, and reached `pending_effectuation`. The HSA Opt-out persisted as `false`. Continue monitoring beyond that intermediate state. Payment redirect returned an HTTPS POST with opaque SAML data; pass it through using the payment safety rules and do not log it. A September 15 carrier run passed 27 applicable scenarios with no failures, including dependents, updates, cancellation, termination, optional SEP upload, and payment discovery.

### Blue Cross and Blue Shield of Arizona

- **Plan (example):** `53901AZ1420107` — AZ · ZIP `85001` · FIPS `04013` · plan year 2026
- **Required attestations:** `electronic_signature_consent`, `agrees_issuer_attestations`, `broker_signature_attestation`
- **SSN:** Not required — omit unless you collect it.
- **SEP documentation:** **Required** — upload before submit
- **Notes:** Requires SEP supporting documentation for `offered_ichra`. Create promotes to `sep_docs_required`; upload a doc, then submit -> `sep_docs_under_review`.

```json
{
  "external_id": "your-tracking-id-001",
  "plan_hios_id": "53901AZ1420107",
  "plan_year": 2026,
  "residential_address": {
    "street_address_1": "123 Main St",
    "city": "Anytown",
    "state": "AZ",
    "zip_code": "85001",
    "fips_code": "04013"
  },
  "applicants": {
    "primary": {
      "external_id": "member-001",
      "first_name": "Jane",
      "last_name": "Doe",
      "date_of_birth": "1990-05-15",
      "gender": "female",
      "email": "jane.doe@example.com",
      "phone": "5555550100",
      "phone_type": "cell",
      "us_citizen": true,
      "resides_in_state": true,
      "uses_tobacco": false,
      "race_ethnicity": "decline_to_answer",
      "hispanic_origin": "decline_to_answer",
      "language_spoken": "english",
      "language_written": "english",
      "signature": "Jane Doe"
    }
  },
  "agent_of_record": {
    "first_name": "Pat",
    "last_name": "Broker",
    "national_producer_number": "98765432",
    "email": "agent@example.com",
    "phone": "5555550101"
  },
  "hra": {
    "offered_hra": true,
    "type": "ichra",
    "amount": 500,
    "contribution_covers": "premium",
    "start": "2026-01-01",
    "employer": {
      "name": "Acme Corp",
      "fein": "123456789",
      "phone": "5555550102",
      "address": {
        "street_address_1": "789 Corporate Blvd",
        "city": "Anytown",
        "state": "AZ",
        "zip_code": "85001"
      }
    }
  },
  "special_enrollment_period": {
    "event_type": "offered_ichra",
    "event_date": "2026-06-01"
  },
  "attestations": {
    "electronic_signature_consent": true,
    "agrees_issuer_attestations": true,
    "broker_signature_attestation": true,
    "agent_advised_consumer_of_product_features": true
  },
  "signatures": {
    "signature_date": "2026-06-15"
  }
}
```

> After create, run **step 2 (upload SEP documentation)** above, then **step 3 (submit)**.

### Blue Cross Blue Shield of Michigan and Blue Care Network

> **Plan year 2026 individual medical plans only.** Quote current inventory. Example HIOS IDs are not stable across plan years.

- **Issuer identification:** `15560` is Blue Cross Blue Shield of Michigan PPO. `98185` is Blue Care Network HMO.
- **Example plan:** `15560MI0350010`, BCBSM PPO, ZIP `48201`, FIPS `26163`, plan year 2026.
- **Applicant identity:** Every applicant needs either `ssn` or `itin`.
- **Required applicant answers:** Every applicant needs `race_ethnicity`, `us_citizen`, `resides_in_state`, and `uses_tobacco`. The primary also needs `phone_type` with `home`, `cell`, or `work`.
- **Relationships:** Use the selected plan's returned options. BCBSM and BCN medical plans accept `spouse` and `child`; `domestic_partner` is rejected.
- **SEP documentation:** Required for every SEP type returned by the selected medical plan. Upload before submit.
- **HSA plan:** When plan requirements return an `agrees_hsa_contact_opt_in` object, display its exact `content` and option labels and follow its `required` value. BCBSM's content asks whether the consumer wants to enroll in a HealthEquity HSA. Send an answered boolean at `communication_preferences.agrees_hsa_contact_opt_in`.
- **Payment:** Submit first, then offer payment redirect and pay by phone. Always follow `payment_instructions`.
- **Lifecycle:** Under the 2026 contract, BCBSM and BCN do not return post-submit update, cancel, or terminate actions. Always follow the current response.
- **Dental boundary:** This section is for Michigan medical plans. Do not apply HCSC qualified-dental rules.

```json
{
  "external_id": "your-tracking-id-001",
  "plan_hios_id": "15560MI0350010",
  "plan_year": 2026,
  "residential_address": {
    "street_address_1": "123 Main St",
    "city": "Detroit",
    "state": "MI",
    "zip_code": "48201",
    "fips_code": "26163"
  },
  "applicants": {
    "primary": {
      "external_id": "member-001",
      "first_name": "Jane",
      "last_name": "Doe",
      "date_of_birth": "1990-05-15",
      "gender": "female",
      "ssn": "317201410",
      "email": "jane.doe@example.com",
      "phone": "3135550100",
      "phone_type": "cell",
      "us_citizen": true,
      "resides_in_state": true,
      "uses_tobacco": false,
      "race_ethnicity": "white",
      "signature": "Jane Doe"
    },
    "dependents": [
      {
        "external_id": "member-002",
        "relationship": "spouse",
        "first_name": "Alex",
        "last_name": "Doe",
        "date_of_birth": "1991-07-22",
        "gender": "male",
        "itin": "900701234",
        "us_citizen": true,
        "resides_in_state": true,
        "uses_tobacco": false,
        "race_ethnicity": "white",
        "signature": "Alex Doe"
      }
    ]
  },
  "agent_of_record": {
    "first_name": "Pat",
    "last_name": "Broker",
    "national_producer_number": "98765432",
    "email": "agent@example.com",
    "phone": "3135550101"
  },
  "hra": {
    "offered_hra": true,
    "type": "ichra",
    "amount": 500,
    "contribution_covers": "premium",
    "start": "2026-09-01",
    "employer": {
      "name": "Acme Corp",
      "fein": "123456789",
      "phone": "3135550102",
      "address": {
        "street_address_1": "789 Corporate Blvd",
        "city": "Detroit",
        "state": "MI",
        "zip_code": "48201"
      }
    }
  },
  "special_enrollment_period": {
    "event_type": "offered_ichra",
    "event_date": "2026-08-15"
  },
  "attestations": {
    "electronic_signature_consent": true,
    "agrees_issuer_attestations": true,
    "broker_signature_attestation": true,
    "agent_advised_consumer_of_product_features": true
  },
  "signatures": {
    "signature_date": "2026-08-31"
  }
}
```

When BCBSM or BCN plan requirements return the HSA question, add the explicit answer:

```json
{
  "communication_preferences": {
    "agrees_hsa_contact_opt_in": false
  }
}
```

Both `true` and `false` are valid. False declines the carrier-defined HSA action but does not block enrollment in the HSA-eligible health plan. When `required` is true, omission or null leaves the draft incomplete; block submit until the answer is corrected. When `required` is false, the answer is optional. If the requirement or its parent is absent or null, omit the field and do not show the question.

Create returns `201` even when its `errors` array contains blocking items. For the illustrated SEP, upload `document_type: "sep"` before submit. A valid submit returns `202`; follow the tracking rules above through intermediate `draft` and `pending_effectuation` states until a lifecycle terminal state or monitoring boundary.

After create, store each returned applicant `member_id`. It is an opaque HealthSherpa identifier. On a full draft PUT, return the exact ID with the matching applicant; do not derive it, use `external_id` for matching, or assign it by array position.

BCBSM and BCN support post-submit payment redirect and pay by phone, with no payment method required before submission. The redirect is an HTTPS `POST` with opaque SAML form data. HTML-escape each field while preserving its decoded form value, and do not inspect or log those values. When change capabilities are false, do not update or change plans. Call cancel or terminate only when its matching `next_actions` entry is present. Otherwise direct the member to the carrier.

### AmeriHealth Caritas Next

- **Plan (example):** `72760DE0010001`, Delaware HMO, ZIP `19801`, FIPS `10003`, plan year 2026.
- **Issuer IDs:** `72760` in Delaware, `67926` in Florida, `38246` in Louisiana, `17414` in North Carolina, and `73107` in South Carolina.
- **Brand:** South Carolina plans are presented as First Choice Next.
- **Identity:** Under the 2026 application contract, SSN is required for each applicant age one or older on the request date. An applicant younger than one may omit it. The example uses the HC.gov test SSN `317201410`; use it only in a sandbox.
- **Primary applicant:** The 2026 fields include `marital_status`, `language_spoken`, and `language_written`. Follow returned options; accepted marital statuses are `married`, `unmarried`, `divorced`, and `widowed`. Gender also accepts `x`. Use `cell` or `work` for `phone_type`.
- **Eligibility:** Send explicit `us_citizen` and `resides_in_state` answers. False makes the applicant ineligible.
- **Relationships:** Use the returned options. The 2026 medical values are `spouse` and `child`.
- **Responsible party:** For a child-only application, render the returned question. When the consumer identifies another person as responsible for payment, send the object using its returned conditional fields; otherwise omit it.
- **SEP documentation:** Follow each returned event's `documentation_required` value. The 2026 events require SEP proof.
- **SEP suspension:** For `sep_docs_required`, prompt for upload. For `sep_docs_under_review`, use bounded monitoring and alert if review becomes stale. For `sep_docs_denied`, stop passive polling and prompt for corrected documents or escalation. None of these statuses means coverage is effectuated.
- **HSA question:** Follow `enrollment_requirements`. For 2026 AmeriHealth Caritas Next plans, `agrees_hsa_contact_opt_in` is absent even when `hsa_eligible` is true, so do not render or send it.
- **State content:** Display the returned Louisiana out-of-network disclosure. For broker-assisted North Carolina applications, display and collect the returned NC licensed-agent attestation.
- **Dental boundary:** Do not apply HCSC qualified-dental rules. Follow the returned pediatric-dental requirement.

```json
{
  "external_id": "your-tracking-id-001",
  "plan_hios_id": "72760DE0010001",
  "plan_year": 2026,
  "residential_address": {
    "street_address_1": "123 Main St",
    "city": "Wilmington",
    "state": "DE",
    "zip_code": "19801",
    "fips_code": "10003"
  },
  "applicants": {
    "primary": {
      "external_id": "member-001",
      "first_name": "Jane",
      "last_name": "Doe",
      "date_of_birth": "1990-05-15",
      "gender": "female",
      "ssn": "317201410",
      "email": "jane.doe@example.com",
      "phone": "3025550100",
      "phone_type": "cell",
      "marital_status": "unmarried",
      "language_spoken": "english",
      "language_written": "english",
      "us_citizen": true,
      "resides_in_state": true,
      "uses_tobacco": false,
      "signature": "Jane Doe"
    }
  },
  "agent_of_record": {
    "first_name": "Pat",
    "last_name": "Broker",
    "national_producer_number": "<agent-npn>",
    "email": "agent@example.com",
    "phone": "3025550101"
  },
  "hra": {
    "offered_hra": true,
    "type": "ichra",
    "amount": 500,
    "contribution_covers": "premium",
    "start": "2026-09-01",
    "employer": {
      "name": "Acme Corp",
      "fein": "<employer-fein>",
      "phone": "3025550102",
      "address": {
        "street_address_1": "789 Corporate Blvd",
        "city": "Wilmington",
        "state": "DE",
        "zip_code": "19801"
      }
    }
  },
  "special_enrollment_period": {
    "event_type": "offered_ichra",
    "event_date": "2026-08-15"
  },
  "attestations": {
    "electronic_signature_consent": true,
    "agrees_issuer_attestations": true,
    "broker_signature_attestation": true,
    "agent_advised_consumer_of_product_features": true
  },
  "signatures": {
    "signature_date": "2026-08-31"
  }
}
```

Upload SEP proof with `document_type: "sep"` when the selected event returns `documentation_required: true`. A successful submit can move the application to `sep_docs_under_review`. After verification, continue monitoring until the policy reaches its later lifecycle status.

For a child-only application where the consumer identifies another person as responsible for payment, merge a responsible-party object into the primary applicant:

```json
{
  "applicants": {
    "primary": {
      "responsible_party": {
        "first_name": "Alex",
        "last_name": "Doe",
        "relationship": "parent",
        "phone": "3025550103",
        "email": "alex.doe@example.com",
        "street_address_1": "123 Main St",
        "city": "Wilmington",
        "state": "DE",
        "zip_code": "19801"
      }
    }
  }
}
```

Use the `question`, `instruction`, relationship options, and field conditions returned by plan requirements. Omit `responsible_party` when the section does not apply.

For a minor primary applicant, include the guardian fields and set `applicants.primary.signature` to the guardian's exact full name. Do not sign with the minor's name.

### UnitedHealthcare

- **Plan (example):** `40702AZ0060051` — AZ · ZIP `85001` · FIPS `04013` · plan year 2026
- **Required attestations:** `electronic_signature_consent`, `agrees_issuer_attestations`, `broker_signature_attestation`, `pediatric_dental`
- **SSN:** Not required — omit unless you collect it.
- **SEP documentation:** Not required before initial submission for `offered_ichra`. Encourage upload when evidence is available and follow later carrier requests.
- **Notes:** Straight-through to `pending_effectuation` without a pre-submission document. Enroll any UnitedHealthcare plan that returns `api_enrollment: true` for the member's county — swap `plan_hios_id` + address per state (quote first). Some states add a state-supplement signature (see "State-specific requirements" below); the plan's `enrollment_requirements` lists exactly which keys to send.

```json
{
  "external_id": "your-tracking-id-001",
  "plan_hios_id": "40702AZ0060051",
  "plan_year": 2026,
  "residential_address": {
    "street_address_1": "123 Main St",
    "city": "Anytown",
    "state": "AZ",
    "zip_code": "85001",
    "fips_code": "04013"
  },
  "applicants": {
    "primary": {
      "external_id": "member-001",
      "first_name": "Jane",
      "last_name": "Doe",
      "date_of_birth": "1990-05-15",
      "gender": "female",
      "email": "jane.doe@example.com",
      "phone": "5555550100",
      "phone_type": "cell",
      "us_citizen": true,
      "resides_in_state": true,
      "uses_tobacco": false,
      "race_ethnicity": "decline_to_answer",
      "hispanic_origin": "decline_to_answer",
      "language_spoken": "english",
      "language_written": "english",
      "signature": "Jane Doe"
    }
  },
  "agent_of_record": {
    "first_name": "Pat",
    "last_name": "Broker",
    "national_producer_number": "98765432",
    "email": "agent@example.com",
    "phone": "5555550101"
  },
  "hra": {
    "offered_hra": true,
    "type": "ichra",
    "amount": 500,
    "contribution_covers": "premium",
    "start": "2026-01-01",
    "employer": {
      "name": "Acme Corp",
      "fein": "123456789",
      "phone": "5555550102",
      "address": {
        "street_address_1": "789 Corporate Blvd",
        "city": "Anytown",
        "state": "AZ",
        "zip_code": "85001"
      }
    }
  },
  "special_enrollment_period": {
    "event_type": "offered_ichra",
    "event_date": "2026-06-01"
  },
  "attestations": {
    "electronic_signature_consent": true,
    "agrees_issuer_attestations": true,
    "broker_signature_attestation": true,
    "agent_advised_consumer_of_product_features": true,
    "pediatric_dental": "not_applicable"
  },
  "signatures": {
    "signature_date": "2026-06-15"
  }
}
```

### Anthem and Wellpoint

- **Plan (example):** `32753MO0980022`, MO · ZIP `63101` · FIPS `29510` · plan year 2026. Always quote current inventory and confirm `api_enrollment: true`.
- **Required attestations:** Fetch `enrollment_requirements` for the selected plan. Broker-assisted test flows include `consumer_working_with_agent: true`.
- **Primary applicant:** Include `marital_status` and `currently_incarcerated`. Citizenship and incarceration answers can make the applicant ineligible.
- **Carrier producer code:** Broker-assisted enrollments include `agent_of_record.carrier_producer_code`. For Anthem and Wellpoint, this is the carrier-issued encrypted agent TIN, not an agency identifier. It must contain exactly 10 uppercase letters and end in `Y` or `Z`.
- **SSN:** The example uses the designated fake SSN `317201410` (`317-20-1410`). Use it only in staging.
- **SEP documentation:** Required for the illustrated Missouri flow. Other carrier and state combinations may differ; follow the create response.
- **Payment:** When `payment_instructions.payment_required_with_submission` is `true`, send an ACH bank account through `PUT /payment_method` before submit.
- **State supplements:** Render and send every state supplement returned in `enrollment_requirements`. For example, Colorado can require primary and disclosures signatures.
- **Anthem electronic communications:** When plan requirements return `communication_preferences.email_contact_consent`, display the returned carrier content and send the consumer's explicit boolean answer. Both Yes and No are valid.
- **Wellpoint Texas accessible materials:** When plan requirements return the primary communication-impairment questions, send the explicit Yes or No answer and any required format fields. Braille uses `communication_impairment_format: "braille"`.
- **Anthem Statement of Accountability:** In CA, CO, and NV, send the two primary-applicant Statement of Accountability booleans. When either is `true`, include the translator details. See [data-model.md](data-model.md).

> **Staging test data only.** The SSN below is the designated fake value. The payment example uses the personal savings fixture provided for Elevance staging and verified with HealthKeepers Virginia on July 27, 2026 and Wellpoint Texas on July 29, 2026. Never use these values in production.

```json
{
  "external_id": "your-tracking-id-001",
  "plan_hios_id": "32753MO0980022",
  "plan_year": 2026,
  "residential_address": {
    "street_address_1": "123 Main St",
    "city": "St. Louis",
    "state": "MO",
    "zip_code": "63101",
    "fips_code": "29510"
  },
  "applicants": {
    "primary": {
      "external_id": "member-001",
      "first_name": "Jane",
      "last_name": "Doe",
      "date_of_birth": "1990-05-15",
      "gender": "female",
      "marital_status": "single",
      "currently_incarcerated": false,
      "email": "jane.doe@example.com",
      "phone": "5555550100",
      "phone_type": "cell",
      "ssn": "317201410",
      "us_citizen": true,
      "resides_in_state": true,
      "uses_tobacco": false,
      "race_ethnicity": "decline_to_answer",
      "hispanic_origin": "decline_to_answer",
      "language_spoken": "english",
      "language_written": "english",
      "signature": "Jane Doe"
    }
  },
  "agent_of_record": {
    "first_name": "Pat",
    "last_name": "Broker",
    "national_producer_number": "98765432",
    "carrier_producer_code": "KJNNKJSJUY",
    "email": "agent@example.com",
    "phone": "5555550101"
  },
  "hra": {
    "offered_hra": true,
    "type": "ichra",
    "amount": 500,
    "contribution_covers": "premium",
    "start": "2026-01-01",
    "employer": {
      "name": "Acme Corp",
      "fein": "123456789",
      "phone": "5555550102",
      "address": {
        "street_address_1": "789 Corporate Blvd",
        "city": "St. Louis",
        "state": "MO",
        "zip_code": "63101"
      }
    }
  },
  "special_enrollment_period": {
    "event_type": "offered_ichra",
    "event_date": "2026-07-08"
  },
  "attestations": {
    "electronic_signature_consent": true,
    "agrees_issuer_attestations": true,
    "broker_signature_attestation": true,
    "agent_advised_consumer_of_product_features": true,
    "consumer_working_with_agent": true
  },
  "signatures": {
    "signature_date": "2026-07-18"
  }
}
```

For Anthem, add the electronic communications answer when returned by plan requirements:

```json
{
  "communication_preferences": {
    "email_contact_consent": false
  }
}
```

For Wellpoint Texas, merge the accessible-materials fields into `applicants.primary`. If the Texas HMO consumer-choice disclosure is returned, also accept it under `attestations`:

```json
{
  "applicants": {
    "primary": {
      "has_communication_impairment": true,
      "communication_impairment_format": "braille"
    }
  },
  "attestations": {
    "disclosure_statement_accepted": true
  }
}
```

When `has_communication_impairment` is false, omit both format fields. When the format is `other`, also send `communication_impairment_format_other` with `encrypted_audio_cd` or `encrypted_data_cd`.

After create:

1. Save `application_id`.
2. Check `errors` for supporting documents and upload them when required.
3. Check `payment_instructions.payment_required_with_submission`.
4. Require the `next_actions` entry with `rel: "payment_method"`. Its method must be `PUT`, and its relative `href` must match the current application. If it is absent or invalid, stop and contact HealthSherpa.
5. Build the request at runtime from the consumer's billing and bank details. The JSON below is a staging fixture, not a production request template.
6. Send the request through the PCI-compliant proxy URL provided by HealthSherpa.
7. Call `/submit` only after the payment method response is `200 OK`.
8. After submit returns 202, poll the application with backoff and continue through `sep_docs_under_review` and `pending_effectuation`, because both are intermediate states. Stop on lifecycle terminal states `effectuated`, `cancelled`, or `terminated`. Stop the current polling attempt at `submission_failed`, then remediate and resubmit. Treat `effectuated` as immediate success. Treat `draft` as transient for up to one hour; after that, alert and reconcile instead of resubmitting automatically.
9. If submission fails, read `errors`, correct the application or payment data, and resubmit.

```json
{
  "payment_method_type": "bank_account",
  "first_name": "Test",
  "last_name": "Applicant",
  "address": {
    "street_address_1": "123 Main St",
    "street_address_2": "",
    "city": "St. Louis",
    "state": "MO",
    "zip_code": "63101"
  },
  "eft_routing": "071205850",
  "eft_number": "23487289374982",
  "eft_type": "savings",
  "eft_level": "personal",
  "bank_name": "Test Bank",
  "payment_type": "both",
  "withdraw_day": 12
}
```

`PERSONALSAVINGS` maps to `eft_type: "savings"` and `eft_level: "personal"`. Synthetic account-holder names are accepted for this staging fixture, but the API requires both `first_name` and `last_name`.

Use `payment_type: "initial"` for the first payment only. Use `"both"` for the first payment and recurring monthly premiums. The endpoint enum also defines `"recurring"`, but Anthem and Wellpoint do not currently support it without an initial payment. `withdraw_day` accepts 1 through 28 and defaults to `1` when omitted. HealthSherpa transmits this selection to the carrier, and the carrier initiates the debits. Verified Anthem Virginia and Wellpoint Texas flows returned `200` from `/payment_method`, `202` from `/submit`, and remained `pending_effectuation` without carrier errors through 90 seconds of polling. This is not final payment or effectuation confirmation. See [payment-and-documents.md](payment-and-documents.md) for transport and response details.

### Oscar

- **Plan (example):** `13877AZ0070072` — AZ · ZIP `85001` · FIPS `04013` · plan year 2026
- **Required attestations:** `electronic_signature_consent`, `agrees_issuer_attestations`, `broker_signature_attestation`, `pediatric_dental`
- **SSN:** Not required — omit unless you collect it.
- **SEP documentation:** **Required** — upload before submit
- **Notes:** Requires SEP supporting documentation for `offered_ichra`. Create stays `draft` with a `supporting_documentation_required` error; upload a doc, then submit -> `pending_effectuation`.

```json
{
  "external_id": "your-tracking-id-001",
  "plan_hios_id": "13877AZ0070072",
  "plan_year": 2026,
  "residential_address": {
    "street_address_1": "123 Main St",
    "city": "Anytown",
    "state": "AZ",
    "zip_code": "85001",
    "fips_code": "04013"
  },
  "applicants": {
    "primary": {
      "external_id": "member-001",
      "first_name": "Jane",
      "last_name": "Doe",
      "date_of_birth": "1990-05-15",
      "gender": "female",
      "email": "jane.doe@example.com",
      "phone": "5555550100",
      "phone_type": "cell",
      "us_citizen": true,
      "resides_in_state": true,
      "uses_tobacco": false,
      "race_ethnicity": "decline_to_answer",
      "hispanic_origin": "decline_to_answer",
      "language_spoken": "english",
      "language_written": "english",
      "signature": "Jane Doe"
    }
  },
  "agent_of_record": {
    "first_name": "Pat",
    "last_name": "Broker",
    "national_producer_number": "98765432",
    "email": "agent@example.com",
    "phone": "5555550101"
  },
  "hra": {
    "offered_hra": true,
    "type": "ichra",
    "amount": 500,
    "contribution_covers": "premium",
    "start": "2026-01-01",
    "employer": {
      "name": "Acme Corp",
      "fein": "123456789",
      "phone": "5555550102",
      "address": {
        "street_address_1": "789 Corporate Blvd",
        "city": "Anytown",
        "state": "AZ",
        "zip_code": "85001"
      }
    }
  },
  "special_enrollment_period": {
    "event_type": "offered_ichra",
    "event_date": "2026-06-01"
  },
  "attestations": {
    "electronic_signature_consent": true,
    "agrees_issuer_attestations": true,
    "broker_signature_attestation": true,
    "agent_advised_consumer_of_product_features": true,
    "pediatric_dental": "not_applicable"
  },
  "signatures": {
    "signature_date": "2026-06-15"
  }
}
```

> After create, run **step 2 (upload SEP documentation)** above, then **step 3 (submit)**.

### Ambetter

- **Plan (example):** `12613AZ0010001` — AZ · ZIP `85001` · FIPS `04013` · plan year 2026
- **Required attestations:** `electronic_signature_consent`, `agrees_issuer_attestations`, `broker_signature_attestation`, `pediatric_dental`
- **SSN:** Not required — omit unless you collect it.
- **SEP documentation:** **Required** — upload before submit
- **Notes:** Requires SEP supporting documentation for `offered_ichra`. Upload a doc after create, then submit.
- **California Statement of Accountability:** For an Ambetter California plan, send the two primary-applicant Statement of Accountability booleans. When either is `true`, include the translator details. See [data-model.md](data-model.md).

```json
{
  "external_id": "your-tracking-id-001",
  "plan_hios_id": "12613AZ0010001",
  "plan_year": 2026,
  "residential_address": {
    "street_address_1": "123 Main St",
    "city": "Anytown",
    "state": "AZ",
    "zip_code": "85001",
    "fips_code": "04013"
  },
  "applicants": {
    "primary": {
      "external_id": "member-001",
      "first_name": "Jane",
      "last_name": "Doe",
      "date_of_birth": "1990-05-15",
      "gender": "female",
      "email": "jane.doe@example.com",
      "phone": "5555550100",
      "phone_type": "cell",
      "us_citizen": true,
      "resides_in_state": true,
      "uses_tobacco": false,
      "race_ethnicity": "decline_to_answer",
      "hispanic_origin": "decline_to_answer",
      "language_spoken": "english",
      "language_written": "english",
      "signature": "Jane Doe"
    }
  },
  "agent_of_record": {
    "first_name": "Pat",
    "last_name": "Broker",
    "national_producer_number": "98765432",
    "email": "agent@example.com",
    "phone": "5555550101"
  },
  "hra": {
    "offered_hra": true,
    "type": "ichra",
    "amount": 500,
    "contribution_covers": "premium",
    "start": "2026-01-01",
    "employer": {
      "name": "Acme Corp",
      "fein": "123456789",
      "phone": "5555550102",
      "address": {
        "street_address_1": "789 Corporate Blvd",
        "city": "Anytown",
        "state": "AZ",
        "zip_code": "85001"
      }
    }
  },
  "special_enrollment_period": {
    "event_type": "offered_ichra",
    "event_date": "2026-06-01"
  },
  "attestations": {
    "electronic_signature_consent": true,
    "agrees_issuer_attestations": true,
    "broker_signature_attestation": true,
    "agent_advised_consumer_of_product_features": true,
    "pediatric_dental": "not_applicable"
  },
  "signatures": {
    "signature_date": "2026-06-15"
  }
}
```

> After create, run **step 2 (upload SEP documentation)** above, then **step 3 (submit)**.

### Molina

- **Plan (example):** `54172FL0010019` — FL · ZIP `33101` · FIPS `12086` · plan year 2026
- **Required attestations:** `electronic_signature_consent`, `agrees_issuer_attestations`, `broker_signature_attestation`, `pediatric_dental`
- **SSN:** Required — include a valid SSN (or ITIN where the carrier accepts one).
- **SEP documentation:** **Required** — upload before submit
- **Notes:** Requires SEP supporting documentation for `offered_ichra`. Upload a doc after create, then submit.

```json
{
  "external_id": "your-tracking-id-001",
  "plan_hios_id": "54172FL0010019",
  "plan_year": 2026,
  "residential_address": {
    "street_address_1": "123 Main St",
    "city": "Anytown",
    "state": "FL",
    "zip_code": "33101",
    "fips_code": "12086"
  },
  "applicants": {
    "primary": {
      "external_id": "member-001",
      "first_name": "Jane",
      "last_name": "Doe",
      "date_of_birth": "1990-05-15",
      "gender": "female",
      "email": "jane.doe@example.com",
      "phone": "5555550100",
      "phone_type": "cell",
      "ssn": "123456789",
      "us_citizen": true,
      "resides_in_state": true,
      "uses_tobacco": false,
      "race_ethnicity": "decline_to_answer",
      "hispanic_origin": "decline_to_answer",
      "language_spoken": "english",
      "language_written": "english",
      "signature": "Jane Doe"
    }
  },
  "agent_of_record": {
    "first_name": "Pat",
    "last_name": "Broker",
    "national_producer_number": "98765432",
    "email": "agent@example.com",
    "phone": "5555550101"
  },
  "hra": {
    "offered_hra": true,
    "type": "ichra",
    "amount": 500,
    "contribution_covers": "premium",
    "start": "2026-01-01",
    "employer": {
      "name": "Acme Corp",
      "fein": "123456789",
      "phone": "5555550102",
      "address": {
        "street_address_1": "789 Corporate Blvd",
        "city": "Anytown",
        "state": "FL",
        "zip_code": "33101"
      }
    }
  },
  "special_enrollment_period": {
    "event_type": "offered_ichra",
    "event_date": "2026-06-01"
  },
  "attestations": {
    "electronic_signature_consent": true,
    "agrees_issuer_attestations": true,
    "broker_signature_attestation": true,
    "agent_advised_consumer_of_product_features": true,
    "pediatric_dental": "not_applicable"
  },
  "signatures": {
    "signature_date": "2026-06-15"
  }
}
```

> After create, run **step 2 (upload SEP documentation)** above, then **step 3 (submit)**.

### Blue KC (Blue Cross Blue Shield Kansas City)

- **Plan (example):** `34762MO0590026` — MO · ZIP `64101` · FIPS `29095` · plan year 2026
- **Required attestations:** `electronic_signature_consent`, `agrees_issuer_attestations`, `broker_signature_attestation`, `pediatric_dental`
- **SSN:** Required — include a valid SSN (or ITIN where the carrier accepts one).
- **SEP documentation:** Not required before initial submission for `offered_ichra`. Encourage upload when evidence is available and follow later carrier requests.
- **Notes:** Straight-through to `pending_effectuation` without a pre-submission document.

```json
{
  "external_id": "your-tracking-id-001",
  "plan_hios_id": "34762MO0590026",
  "plan_year": 2026,
  "residential_address": {
    "street_address_1": "123 Main St",
    "city": "Anytown",
    "state": "MO",
    "zip_code": "64101",
    "fips_code": "29095"
  },
  "applicants": {
    "primary": {
      "external_id": "member-001",
      "first_name": "Jane",
      "last_name": "Doe",
      "date_of_birth": "1990-05-15",
      "gender": "female",
      "email": "jane.doe@example.com",
      "phone": "5555550100",
      "phone_type": "cell",
      "ssn": "123456789",
      "us_citizen": true,
      "resides_in_state": true,
      "uses_tobacco": false,
      "race_ethnicity": "decline_to_answer",
      "hispanic_origin": "decline_to_answer",
      "language_spoken": "english",
      "language_written": "english",
      "signature": "Jane Doe"
    }
  },
  "agent_of_record": {
    "first_name": "Pat",
    "last_name": "Broker",
    "national_producer_number": "98765432",
    "email": "agent@example.com",
    "phone": "5555550101"
  },
  "hra": {
    "offered_hra": true,
    "type": "ichra",
    "amount": 500,
    "contribution_covers": "premium",
    "start": "2026-01-01",
    "employer": {
      "name": "Acme Corp",
      "fein": "123456789",
      "phone": "5555550102",
      "address": {
        "street_address_1": "789 Corporate Blvd",
        "city": "Anytown",
        "state": "MO",
        "zip_code": "64101"
      }
    }
  },
  "special_enrollment_period": {
    "event_type": "offered_ichra",
    "event_date": "2026-06-01"
  },
  "attestations": {
    "electronic_signature_consent": true,
    "agrees_issuer_attestations": true,
    "broker_signature_attestation": true,
    "agent_advised_consumer_of_product_features": true,
    "pediatric_dental": "not_applicable"
  },
  "signatures": {
    "signature_date": "2026-06-15"
  }
}
```

### Blue Cross and Blue Shield of South Carolina

- **Plan (example):** `26065SC0720010` — SC · ZIP `29201` · FIPS `45079` · plan year 2026
- **Required attestations:** `electronic_signature_consent`, `agrees_issuer_attestations`, `broker_signature_attestation`, `pediatric_dental`
- **SSN:** Required — include a valid SSN (or ITIN where the carrier accepts one).
- **SEP documentation:** **Required** — upload before submit
- **Notes:** Requires SEP supporting documentation for `offered_ichra`. Upload a doc after create, then submit.

```json
{
  "external_id": "your-tracking-id-001",
  "plan_hios_id": "26065SC0720010",
  "plan_year": 2026,
  "residential_address": {
    "street_address_1": "123 Main St",
    "city": "Anytown",
    "state": "SC",
    "zip_code": "29201",
    "fips_code": "45079"
  },
  "applicants": {
    "primary": {
      "external_id": "member-001",
      "first_name": "Jane",
      "last_name": "Doe",
      "date_of_birth": "1990-05-15",
      "gender": "female",
      "email": "jane.doe@example.com",
      "phone": "5555550100",
      "phone_type": "cell",
      "ssn": "123456789",
      "us_citizen": true,
      "resides_in_state": true,
      "uses_tobacco": false,
      "race_ethnicity": "decline_to_answer",
      "hispanic_origin": "decline_to_answer",
      "language_spoken": "english",
      "language_written": "english",
      "signature": "Jane Doe"
    }
  },
  "agent_of_record": {
    "first_name": "Pat",
    "last_name": "Broker",
    "national_producer_number": "98765432",
    "email": "agent@example.com",
    "phone": "5555550101"
  },
  "hra": {
    "offered_hra": true,
    "type": "ichra",
    "amount": 500,
    "contribution_covers": "premium",
    "start": "2026-01-01",
    "employer": {
      "name": "Acme Corp",
      "fein": "123456789",
      "phone": "5555550102",
      "address": {
        "street_address_1": "789 Corporate Blvd",
        "city": "Anytown",
        "state": "SC",
        "zip_code": "29201"
      }
    }
  },
  "special_enrollment_period": {
    "event_type": "offered_ichra",
    "event_date": "2026-06-01"
  },
  "attestations": {
    "electronic_signature_consent": true,
    "agrees_issuer_attestations": true,
    "broker_signature_attestation": true,
    "agent_advised_consumer_of_product_features": true,
    "pediatric_dental": "not_applicable"
  },
  "signatures": {
    "signature_date": "2026-06-15"
  }
}
```

> After create, run **step 2 (upload SEP documentation)** above, then **step 3 (submit)**.

### CareSource

- **Plan (example):** `77552OH0010222` — OH · ZIP `43215` · FIPS `39049` · plan year 2026
- **Required attestations:** `electronic_signature_consent`, `agrees_issuer_attestations`, `broker_signature_attestation`, `pediatric_dental`
- **SSN:** Not required — omit unless you collect it.
- **SEP documentation:** Not required before initial submission for `offered_ichra`. Encourage upload when evidence is available and follow later carrier requests.
- **Notes:** Straight-through to `pending_effectuation` without a pre-submission document.

```json
{
  "external_id": "your-tracking-id-001",
  "plan_hios_id": "77552OH0010222",
  "plan_year": 2026,
  "residential_address": {
    "street_address_1": "123 Main St",
    "city": "Anytown",
    "state": "OH",
    "zip_code": "43215",
    "fips_code": "39049"
  },
  "applicants": {
    "primary": {
      "external_id": "member-001",
      "first_name": "Jane",
      "last_name": "Doe",
      "date_of_birth": "1990-05-15",
      "gender": "female",
      "email": "jane.doe@example.com",
      "phone": "5555550100",
      "phone_type": "cell",
      "us_citizen": true,
      "resides_in_state": true,
      "uses_tobacco": false,
      "race_ethnicity": "decline_to_answer",
      "hispanic_origin": "decline_to_answer",
      "language_spoken": "english",
      "language_written": "english",
      "signature": "Jane Doe"
    }
  },
  "agent_of_record": {
    "first_name": "Pat",
    "last_name": "Broker",
    "national_producer_number": "98765432",
    "email": "agent@example.com",
    "phone": "5555550101"
  },
  "hra": {
    "offered_hra": true,
    "type": "ichra",
    "amount": 500,
    "contribution_covers": "premium",
    "start": "2026-01-01",
    "employer": {
      "name": "Acme Corp",
      "fein": "123456789",
      "phone": "5555550102",
      "address": {
        "street_address_1": "789 Corporate Blvd",
        "city": "Anytown",
        "state": "OH",
        "zip_code": "43215"
      }
    }
  },
  "special_enrollment_period": {
    "event_type": "offered_ichra",
    "event_date": "2026-06-01"
  },
  "attestations": {
    "electronic_signature_consent": true,
    "agrees_issuer_attestations": true,
    "broker_signature_attestation": true,
    "agent_advised_consumer_of_product_features": true,
    "pediatric_dental": "not_applicable"
  },
  "signatures": {
    "signature_date": "2026-06-15"
  }
}
```

### CoxHealth

Quote current Missouri inventory under issuer `96384` and require `api_enrollment: true`; do not hardcode a plan ID. Apply these CoxHealth fields to the complete request shape:

- Require an SSN, `uses_tobacco`, and `race_ethnicity` for every applicant.
- Require the primary's `marital_status`; current values use the standard single and married options.
- Do not send citizenship or state-residency answers unless current requirements return them.
- Use only the returned relationships. CoxHealth currently supports `spouse` and `child`.
- Send the complete HRA and employer block. CoxHealth collects affordability, TPA, and employer FEIN information and requires the HRA offer answer. Current `contribution_covers` values are `premium` and `premium_oop`.
- Current SEP events return `documentation_required: false`. This means proof does not block initial submission. Encourage upload when evidence is available and use the current supporting-documentation action during the returned post-submit window.
- CoxHealth does not present an HSA contact question or pediatric-dental attestation through the current carrier configuration. Continue to follow live requirements.
- Read payment instructions and lifecycle actions from the application response.

### Christus

- **Plan (example):** `98780LA0220003` — LA · ZIP `71101` · FIPS `22017` · plan year 2026
- **Required attestations:** `electronic_signature_consent`, `agrees_issuer_attestations`, `broker_signature_attestation`, `pediatric_dental`
- **SSN:** Not required — omit unless you collect it.
- **SEP documentation:** Not required before initial submission for `offered_ichra`. Encourage upload when evidence is available and follow later carrier requests.
- **Notes:** Straight-through to `pending_effectuation` without a pre-submission document.

```json
{
  "external_id": "your-tracking-id-001",
  "plan_hios_id": "98780LA0220003",
  "plan_year": 2026,
  "residential_address": {
    "street_address_1": "123 Main St",
    "city": "Anytown",
    "state": "LA",
    "zip_code": "71101",
    "fips_code": "22017"
  },
  "applicants": {
    "primary": {
      "external_id": "member-001",
      "first_name": "Jane",
      "last_name": "Doe",
      "date_of_birth": "1990-05-15",
      "gender": "female",
      "email": "jane.doe@example.com",
      "phone": "5555550100",
      "phone_type": "cell",
      "us_citizen": true,
      "resides_in_state": true,
      "uses_tobacco": false,
      "race_ethnicity": "decline_to_answer",
      "hispanic_origin": "decline_to_answer",
      "language_spoken": "english",
      "language_written": "english",
      "signature": "Jane Doe"
    }
  },
  "agent_of_record": {
    "first_name": "Pat",
    "last_name": "Broker",
    "national_producer_number": "98765432",
    "email": "agent@example.com",
    "phone": "5555550101"
  },
  "hra": {
    "offered_hra": true,
    "type": "ichra",
    "amount": 500,
    "contribution_covers": "premium",
    "start": "2026-01-01",
    "employer": {
      "name": "Acme Corp",
      "fein": "123456789",
      "phone": "5555550102",
      "address": {
        "street_address_1": "789 Corporate Blvd",
        "city": "Anytown",
        "state": "LA",
        "zip_code": "71101"
      }
    }
  },
  "special_enrollment_period": {
    "event_type": "offered_ichra",
    "event_date": "2026-06-01"
  },
  "attestations": {
    "electronic_signature_consent": true,
    "agrees_issuer_attestations": true,
    "broker_signature_attestation": true,
    "agent_advised_consumer_of_product_features": true,
    "pediatric_dental": "not_applicable"
  },
  "signatures": {
    "signature_date": "2026-06-15"
  }
}
```

### Health First

- **Plan (example):** `36194FL0490001` — FL · ZIP `32901` · FIPS `12009` · plan year 2026
- **Required attestations:** `electronic_signature_consent`, `agrees_issuer_attestations`, `broker_signature_attestation`, `pediatric_dental`
- **SSN:** Not required — omit unless you collect it.
- **SEP documentation:** **Required** — upload before submit
- **Notes:** Requires SEP supporting documentation for `offered_ichra`. Upload a doc after create, then submit.

```json
{
  "external_id": "your-tracking-id-001",
  "plan_hios_id": "36194FL0490001",
  "plan_year": 2026,
  "residential_address": {
    "street_address_1": "123 Main St",
    "city": "Anytown",
    "state": "FL",
    "zip_code": "32901",
    "fips_code": "12009"
  },
  "applicants": {
    "primary": {
      "external_id": "member-001",
      "first_name": "Jane",
      "last_name": "Doe",
      "date_of_birth": "1990-05-15",
      "gender": "female",
      "email": "jane.doe@example.com",
      "phone": "5555550100",
      "phone_type": "cell",
      "us_citizen": true,
      "resides_in_state": true,
      "uses_tobacco": false,
      "race_ethnicity": "decline_to_answer",
      "hispanic_origin": "decline_to_answer",
      "language_spoken": "english",
      "language_written": "english",
      "signature": "Jane Doe"
    }
  },
  "agent_of_record": {
    "first_name": "Pat",
    "last_name": "Broker",
    "national_producer_number": "98765432",
    "email": "agent@example.com",
    "phone": "5555550101"
  },
  "hra": {
    "offered_hra": true,
    "type": "ichra",
    "amount": 500,
    "contribution_covers": "premium",
    "start": "2026-01-01",
    "employer": {
      "name": "Acme Corp",
      "fein": "123456789",
      "phone": "5555550102",
      "address": {
        "street_address_1": "789 Corporate Blvd",
        "city": "Anytown",
        "state": "FL",
        "zip_code": "32901"
      }
    }
  },
  "special_enrollment_period": {
    "event_type": "offered_ichra",
    "event_date": "2026-06-01"
  },
  "attestations": {
    "electronic_signature_consent": true,
    "agrees_issuer_attestations": true,
    "broker_signature_attestation": true,
    "agent_advised_consumer_of_product_features": true,
    "pediatric_dental": "not_applicable"
  },
  "signatures": {
    "signature_date": "2026-06-15"
  }
}
```

> After create, run **step 2 (upload SEP documentation)** above, then **step 3 (submit)**.

### Group Health Cooperative

> **Validated in HealthSherpa staging on September 30, 2026.** The tested PY2026 Wisconsin plans used issuer `94529` in Madison, ZIP 53703, FIPS 55025. Quote current inventory instead of hardcoding those plans.

- Every applicant requires an SSN and race or ethnicity.
- Send the primary's `phone_type`, `language_spoken`, and `language_written`.
- Current gender options include `male`, `female`, and `x`.
- Current dependent relationships are `spouse`, `domestic_partner`, and `child`. Follow the returned adult-signature metadata.
- When an HSA plan returns the required Fidelity question, display the returned content and send `communication_preferences.agrees_hsa_contact_opt_in`. An explicit `false` is complete and does not block enrollment.
- Plan requirements return `hra.explicit_dependent_hra_offer_supported: true`. When a dependent answers separately, send only `offered_hra: true` or `false`, or `offered_hra_unknown: true`, under `applicants.dependents[].hra`. Omit the dependent HRA block, or send `offered_hra: null`, to inherit the application-level answer. A dependent can answer `offered_hra: true` only when the application-level `hra.offered_hra` is also `true`. Keep employer and contribution details in the complete top-level HRA. This offer-only behavior is separate from `per_applicant_hra`.
- Current returned SEP events require proof before submission. Upload `document_type: "sep"` before submit.
- Current responses support application updates, dependent addition and removal, cancellation before effectuation, payment discovery, and application search. Follow live capabilities and actions.

The September 30 run passed 25 applicable scenarios with no failures. Submitted applications reached `pending_effectuation`; the future-effective cancellation reached `cancelled`. These results do not prove carrier effectuation or downstream file delivery.

### Jefferson Health Plans

> **Validated in HealthSherpa staging on September 29, 2026.** The tested PY2026 Pennsylvania plan used issuer `93909`. Quote current inventory and require `api_enrollment: true`.

- Every applicant requires an SSN and race or ethnicity.
- Send the primary's `marital_status`, `phone_type`, `language_spoken`, and `language_written`.
- Do not send citizenship or state-residency answers unless current requirements return them.
- Use the full dependent relationship option set returned by the selected plan. Do not replace it with a generic spouse and child list.
- Jefferson asks tobacco questions of dependents rather than the primary and collects the child medical-support-order answer where applicable.
- For a child-only application, render the returned responsible-party question. When another person is responsible for payment, send the responsible-party object using a returned relationship and keep HRA employer data out of it.
- Send the complete top-level HRA and an explicit HRA offer answer.
- Follow returned dependent-signature metadata; the primary signature and application signature date are always separate.
- Current returned SEP events require proof before submission. Upload `document_type: "sep"` before submit.
- The September 29 recreation submitted five applications, delivered their SEP documents, and successfully queued two future-effective cancellations. No HSA contact requirement was returned for the selected plan.

### Antidote

- **Plan (example):** `68445AZ0600010` — AZ · ZIP `85001` · FIPS `04013` · plan year 2026
- **Required attestations:** `electronic_signature_consent`, `agrees_issuer_attestations`, `broker_signature_attestation`, `pediatric_dental`
- **SSN:** Required — include a valid SSN (or ITIN where the carrier accepts one).
- **SEP documentation:** Not required before initial submission for `offered_ichra`. Encourage upload when evidence is available and follow later carrier requests.
- **Notes:** Straight-through to `pending_effectuation` without a pre-submission document.

```json
{
  "external_id": "your-tracking-id-001",
  "plan_hios_id": "68445AZ0600010",
  "plan_year": 2026,
  "residential_address": {
    "street_address_1": "123 Main St",
    "city": "Anytown",
    "state": "AZ",
    "zip_code": "85001",
    "fips_code": "04013"
  },
  "applicants": {
    "primary": {
      "external_id": "member-001",
      "first_name": "Jane",
      "last_name": "Doe",
      "date_of_birth": "1990-05-15",
      "gender": "female",
      "email": "jane.doe@example.com",
      "phone": "5555550100",
      "phone_type": "cell",
      "ssn": "123456789",
      "us_citizen": true,
      "resides_in_state": true,
      "uses_tobacco": false,
      "race_ethnicity": "decline_to_answer",
      "hispanic_origin": "decline_to_answer",
      "language_spoken": "english",
      "language_written": "english",
      "signature": "Jane Doe"
    }
  },
  "agent_of_record": {
    "first_name": "Pat",
    "last_name": "Broker",
    "national_producer_number": "98765432",
    "email": "agent@example.com",
    "phone": "5555550101"
  },
  "hra": {
    "offered_hra": true,
    "type": "ichra",
    "amount": 500,
    "contribution_covers": "premium",
    "start": "2026-01-01",
    "employer": {
      "name": "Acme Corp",
      "fein": "123456789",
      "phone": "5555550102",
      "address": {
        "street_address_1": "789 Corporate Blvd",
        "city": "Anytown",
        "state": "AZ",
        "zip_code": "85001"
      }
    }
  },
  "special_enrollment_period": {
    "event_type": "offered_ichra",
    "event_date": "2026-06-01"
  },
  "attestations": {
    "electronic_signature_consent": true,
    "agrees_issuer_attestations": true,
    "broker_signature_attestation": true,
    "agent_advised_consumer_of_product_features": true,
    "pediatric_dental": "not_applicable"
  },
  "signatures": {
    "signature_date": "2026-06-15"
  }
}
```

### Mountain Health CO-OP

- **Plan (example):** `38128ID0100004` — ID · ZIP `83701` · FIPS `16001` · plan year 2026
- **Required attestations:** `electronic_signature_consent`, `agrees_issuer_attestations`, `broker_signature_attestation`, `pediatric_dental`
- **SSN:** Required — include a valid SSN (or ITIN where the carrier accepts one).
- **SEP documentation:** **Required** — upload before submit
- **Notes:** Requires SEP supporting documentation for `offered_ichra`. Upload a doc after create, then submit.

```json
{
  "external_id": "your-tracking-id-001",
  "plan_hios_id": "38128ID0100004",
  "plan_year": 2026,
  "residential_address": {
    "street_address_1": "123 Main St",
    "city": "Anytown",
    "state": "ID",
    "zip_code": "83701",
    "fips_code": "16001"
  },
  "applicants": {
    "primary": {
      "external_id": "member-001",
      "first_name": "Jane",
      "last_name": "Doe",
      "date_of_birth": "1990-05-15",
      "gender": "female",
      "email": "jane.doe@example.com",
      "phone": "5555550100",
      "phone_type": "cell",
      "ssn": "123456789",
      "us_citizen": true,
      "resides_in_state": true,
      "uses_tobacco": false,
      "race_ethnicity": "decline_to_answer",
      "hispanic_origin": "decline_to_answer",
      "language_spoken": "english",
      "language_written": "english",
      "signature": "Jane Doe"
    }
  },
  "agent_of_record": {
    "first_name": "Pat",
    "last_name": "Broker",
    "national_producer_number": "98765432",
    "email": "agent@example.com",
    "phone": "5555550101"
  },
  "hra": {
    "offered_hra": true,
    "type": "ichra",
    "amount": 500,
    "contribution_covers": "premium",
    "start": "2026-01-01",
    "employer": {
      "name": "Acme Corp",
      "fein": "123456789",
      "phone": "5555550102",
      "address": {
        "street_address_1": "789 Corporate Blvd",
        "city": "Anytown",
        "state": "ID",
        "zip_code": "83701"
      }
    }
  },
  "special_enrollment_period": {
    "event_type": "offered_ichra",
    "event_date": "2026-06-01"
  },
  "attestations": {
    "electronic_signature_consent": true,
    "agrees_issuer_attestations": true,
    "broker_signature_attestation": true,
    "agent_advised_consumer_of_product_features": true,
    "pediatric_dental": "not_applicable"
  },
  "signatures": {
    "signature_date": "2026-06-15"
  }
}
```

> After create, run **step 2 (upload SEP documentation)** above, then **step 3 (submit)**.

### Medical Mutual of Ohio (MedMutual)

- **Plan (example):** `99969OH0090359` — OH · ZIP `43215` · FIPS `39049` · plan year 2026
- **Required attestations:** `electronic_signature_consent`, `agrees_issuer_attestations`, `broker_signature_attestation`, `pediatric_dental`
- **SSN:** Required — include a valid SSN (or ITIN where the carrier accepts one).
- **SEP documentation:** **Required** — upload before submit
- **Notes:** Requires SEP supporting documentation for `offered_ichra`. Upload a doc after create, then submit.

```json
{
  "external_id": "your-tracking-id-001",
  "plan_hios_id": "99969OH0090359",
  "plan_year": 2026,
  "residential_address": {
    "street_address_1": "123 Main St",
    "city": "Anytown",
    "state": "OH",
    "zip_code": "43215",
    "fips_code": "39049"
  },
  "applicants": {
    "primary": {
      "external_id": "member-001",
      "first_name": "Jane",
      "last_name": "Doe",
      "date_of_birth": "1990-05-15",
      "gender": "female",
      "email": "jane.doe@example.com",
      "phone": "5555550100",
      "phone_type": "cell",
      "ssn": "123456789",
      "us_citizen": true,
      "resides_in_state": true,
      "uses_tobacco": false,
      "race_ethnicity": "decline_to_answer",
      "hispanic_origin": "decline_to_answer",
      "language_spoken": "english",
      "language_written": "english",
      "signature": "Jane Doe"
    }
  },
  "agent_of_record": {
    "first_name": "Pat",
    "last_name": "Broker",
    "national_producer_number": "98765432",
    "email": "agent@example.com",
    "phone": "5555550101"
  },
  "hra": {
    "offered_hra": true,
    "type": "ichra",
    "amount": 500,
    "contribution_covers": "premium",
    "start": "2026-01-01",
    "employer": {
      "name": "Acme Corp",
      "fein": "123456789",
      "phone": "5555550102",
      "address": {
        "street_address_1": "789 Corporate Blvd",
        "city": "Anytown",
        "state": "OH",
        "zip_code": "43215"
      }
    }
  },
  "special_enrollment_period": {
    "event_type": "offered_ichra",
    "event_date": "2026-06-01"
  },
  "attestations": {
    "electronic_signature_consent": true,
    "agrees_issuer_attestations": true,
    "broker_signature_attestation": true,
    "agent_advised_consumer_of_product_features": true,
    "pediatric_dental": "not_applicable"
  },
  "signatures": {
    "signature_date": "2026-06-15"
  }
}
```

> After create, run **step 2 (upload SEP documentation)** above, then **step 3 (submit)**.

### Quartz

> **Validated in HealthSherpa staging on September 30, 2026.** The tested PY2026 plans used issuer `37833` in Wisconsin and issuer `70373` in Minnesota. Quote current inventory rather than hardcoding those plans.

- Every applicant requires an SSN and race or ethnicity.
- Ask tobacco only for applicants age 21 or older.
- Send the primary's spoken and written language values using the exact options returned for the selected plan. Current spoken options include `"American Sign Language"`; written language excludes that value.
- Current dependent relationships include `spouse`, `domestic_partner`, `child`, `stepchild`, `grandchild`, `adopted_child`, and `ward`.
- Follow returned adult-signature metadata.
- For a minor primary applicant, render the returned responsible-party section and send the applicable parent or guardian information.
- Quartz Wisconsin and Minnesota collect the Statement of Accountability. Send the required primary booleans and conditional translator fields.
- Current returned SEP events require proof before submission. Upload `document_type: "sep"`. Quartz also supports later document replacement during its returned post-submission window.
- Current tested HSA plans did not return an HSA contact question. Do not infer one from `hsa_eligible`.
- Read payment instructions by application. The Wisconsin test returned a payment path; the Minnesota test did not expose one.

The Wisconsin run passed 24 applicable scenarios with no failures. The Minnesota run passed 23, had no failures, and marked payment as not verifiable because no payment instructions were returned.

### HCSC (Blue Cross and Blue Shield — IL / MT / NM / OK / TX)

HCSC is the carrier behind Blue Cross and Blue Shield of Illinois, Montana, New Mexico, Oklahoma, and Texas. It is the one carrier here that **offers qualified dental**, so every application must satisfy the ACA pediatric dental requirement exactly one way (see below).

- **Plan (example):** `36096IL0810080` — Blue Cross and Blue Shield of Illinois (issuer `36096`) · IL · ZIP `60601` · FIPS `17031` · plan year 2026 (illustrative — quote for current inventory)
- **Required attestations:** `electronic_signature_consent`, `agrees_issuer_attestations`, `broker_signature_attestation`
- **Pediatric dental:** **Required, exactly one of** `attestations.pediatric_dental` **or** top-level `dental_plan_hios_id` — never both, never neither (else `422 Invalid dental selection`).
- **SSN:** Not required — omit unless you collect it. (FEIN is also not collected by HCSC; including it is harmless.)
- **SEP documentation:** **Required for every SEP reason, including `offered_ichra`** — upload before submit (10 MB limit).
- **Notes:** No post-enrollment changes, cancellations, or renewals (`supports_changes: false`). `phone_type` is required. Create promotes to `sep_docs_required`; upload a doc, then submit.

**Branch A — pediatric dental attestation** (no children under 19, or member already has stand-alone dental):

```json
{
  "external_id": "your-tracking-id-001",
  "plan_hios_id": "36096IL0810080",
  "plan_year": 2026,
  "residential_address": {
    "street_address_1": "123 Main St",
    "city": "Chicago",
    "state": "IL",
    "zip_code": "60601",
    "fips_code": "17031"
  },
  "applicants": {
    "primary": {
      "external_id": "member-001",
      "first_name": "Jane",
      "last_name": "Doe",
      "date_of_birth": "1990-05-15",
      "gender": "female",
      "email": "jane.doe@example.com",
      "phone": "5555550100",
      "phone_type": "cell",
      "us_citizen": true,
      "resides_in_state": true,
      "uses_tobacco": false,
      "race_ethnicity": "decline_to_answer",
      "hispanic_origin": "decline_to_answer",
      "language_spoken": "english",
      "language_written": "english",
      "signature": "Jane Doe"
    }
  },
  "agent_of_record": {
    "first_name": "Pat",
    "last_name": "Broker",
    "national_producer_number": "98765432",
    "email": "agent@example.com",
    "phone": "5555550101"
  },
  "hra": {
    "offered_hra": true,
    "type": "ichra",
    "amount": 500,
    "contribution_covers": "premium",
    "start": "2026-01-01",
    "employer": {
      "name": "Acme Corp",
      "fein": "123456789",
      "phone": "5555550102",
      "address": {
        "street_address_1": "789 Corporate Blvd",
        "city": "Chicago",
        "state": "IL",
        "zip_code": "60601"
      }
    }
  },
  "special_enrollment_period": {
    "event_type": "offered_ichra",
    "event_date": "2026-06-01"
  },
  "attestations": {
    "electronic_signature_consent": true,
    "agrees_issuer_attestations": true,
    "broker_signature_attestation": true,
    "agent_advised_consumer_of_product_features": true,
    "pediatric_dental": "not_applicable"
  },
  "signatures": {
    "signature_date": "2026-06-15"
  }
}
```

**Branch B — stand-alone dental plan.** Identical to Branch A, except you **remove** `attestations.pediatric_dental` and add a top-level `dental_plan_hios_id`. The dental plan **must be the same carrier as the medical plan** (BlueCare Dental for HCSC — same issuer prefix, e.g. `36096` in IL) and available in the member's service area. Discover eligible dental plans with a ZIP/FIPS-scoped dental quote (`POST /quotes` with `dental_search: true` and the member's `zip_code`/`fip_code`/`state`); because it is scoped to the member's location it returns only plans in their service area. Pick one whose issuer matches the medical plan's issuer (see [quoting-and-plans.md](quoting-and-plans.md)). A different-carrier dental plan, or one outside the member's service area, returns `422 No plan found for dental_plan_hios_id: <id>`. The `GET /plans?...&dental_only=true` list is state-level and not service-area filtered, so validate service area with the dental quote before enrolling. The example below uses the verified BCBS IL dental plan `36096IL0830001` ("BlueCare Dental"); quote for current inventory rather than hardcoding.

```json
{
  "plan_hios_id": "36096IL0810080",
  "dental_plan_hios_id": "36096IL0830001",
  "attestations": {
    "electronic_signature_consent": true,
    "agrees_issuer_attestations": true,
    "broker_signature_attestation": true,
    "agent_advised_consumer_of_product_features": true
  }
}
```

> After create, run **step 2 (upload SEP documentation)** above, then **step 3 (submit)**.

---

## Multi-member households

Add dependents under `applicants.dependents` (an array). Each dependent needs a `relationship` value returned for the selected plan and the same identity fields as the primary. Do not reduce the returned options to a generic spouse, child, and domestic-partner list; carriers such as Quartz return additional relationships.

- `has_disability` and `full_time_student` (booleans) on every dependent when required. When `full_time_student` is `true`, also send `graduation_date` unless the carrier explicitly omits it, as Avera does for its student age exception.
- A typed dependent `signature` when the selected plan's returned signature metadata requires one. For current Avera plans, an adult domestic partner signs and a spouse does not.
- `ssn` only when the carrier requires it (see each carrier above); dependents may otherwise omit it.

Always include the top-level `hra` object with employer details. When plan requirements return `hra.per_applicant_hra: true`, also send `hra` under the primary and every non-child dependent relationship supported by the selected carrier, using the returned `contribution_covers` values. Do not send these per-applicant HRA details for children. When `hra.explicit_dependent_hra_offer_supported: true`, a dependent can instead send only `offered_hra` or `offered_hra_unknown` under its own `hra` block; omission or `offered_hra: null` inherits the application-level answer, and a dependent cannot answer `true` unless the application-level answer is also `true`. If neither requirement is returned, do not duplicate the household HRA answer per applicant. The example below (primary + spouse + child) was verified end-to-end for a carrier that does not require per-applicant HRA answers.

```json
{
  "applicants": {
    "primary": { "...": "as shown in the carrier examples above" },
    "dependents": [
      {
        "relationship": "spouse",
        "external_id": "member-002",
        "first_name": "Alex",
        "last_name": "Doe",
        "date_of_birth": "1989-07-22",
        "gender": "male",
        "uses_tobacco": false,
        "us_citizen": true,
        "resides_in_state": true,
        "has_disability": false,
        "full_time_student": false,
        "race_ethnicity": "decline_to_answer",
        "hispanic_origin": "decline_to_answer",
        "signature": "Alex Doe"
      },
      {
        "relationship": "child",
        "external_id": "member-003",
        "first_name": "Sam",
        "last_name": "Doe",
        "date_of_birth": "2015-03-10",
        "gender": "male",
        "uses_tobacco": false,
        "us_citizen": true,
        "resides_in_state": true,
        "has_disability": false,
        "full_time_student": false,
        "race_ethnicity": "decline_to_answer",
        "hispanic_origin": "decline_to_answer"
      }
    ]
  }
}
```

## State-specific requirements

Some states require an additional **state-supplement signature** alongside the primary signature. These are not needed in most states, so the base payloads above omit them. When you enroll in one of these states, add the matching field(s) under the `signatures` object:

| State | Add to `signatures` |
|---|---|
| Colorado (CO) | `state_supplement_primary_signature`, `state_supplement_disclosures_signature` |

Texas HMO plans can return
`enrollment_requirements.attestations.tx_hmo_consumer_choice_disclosure`.
Display its `content`. When
`content_data.disclosure_statement_checkbox_label` is present, render that
checkbox and send `attestations.disclosure_statement_accepted: true`. Do not
require acceptance when the disclosure does not include a checkbox label.

Example (`signatures` block for a Colorado enrollment):

```json
{
  "signatures": {
    "signature_date": "2026-06-15",
    "state_supplement_primary_signature": "Jane Doe",
    "state_supplement_disclosures_signature": "Jane Doe"
  }
}
```

**Forward-compatible rule (do this and you never need an updated example):** before create, call `GET /api/v1/plans/{hios_id}?plan_year=2026&include=enrollment_requirements` and inspect `enrollment_requirements.attestations`. For every `state_supplement_*` key present, send the corresponding `signatures.state_supplement_*` value (typed legal name). Carrier/state combinations that are not yet enabled today will surface their exact requirements through this response the moment they go live, so a payload built this way will succeed without any changes on your side.

## Common 422s and fixes

| Error references | Cause | Fix |
|---|---|---|
| `applicants.primary.signature` / `signatures.signature_date` | Signature split not honored | Send both — typed name on the applicant, date in `signatures` |
| `agrees_issuer_attestations` / `electronic_signature_consent` | Required attestation omitted | Include all attestations listed for that carrier |
| `hra` | HRA answer missing | Include the top-level `hra` block. If plan requirements return `hra.per_applicant_hra: true`, also send HRA answers for the primary and every non-child dependent, plus any conditionally required top-level QSEHRA household follow-up |
| `applicants[n].has_disability` / `full_time_student` / `signature` | A carrier-required dependent field is omitted | Follow the selected plan and carrier rules. Avera requires `has_disability` for every dependent, `full_time_student` only for its age exception, and a signature for the currently returned adult relationships. |
| `ssn` / `itin` (`invalid_field_value` on submit) | Carrier identity requirement is not satisfied | Follow the carrier rule. BCBS Michigan accepts either value for every applicant. Avera accepts either value for applicants age one or older. Other carriers can require SSN specifically. |
| `communication_preferences.agrees_hsa_contact_opt_in` | The selected plan returned a required HSA question and the answer is omitted or null | Display the returned content and boolean options, then send an explicit `true` or `false`; false is valid |
| `supporting_documentation_required` | Carrier requires SEP proof before initial submission (e.g., BCBS AZ) | Upload via `/supporting_documentation` then submit |
| Proof of residency required | Plan requirements return `proof_of_residency_required: true` | Display `proof_of_residency_text` and upload with `document_type: "proof_of_residency"` before submit |
| `payment_method` (`missing_required_field`) | `payment_required_with_submission` is true and no payment method has been saved | Use the `payment_method` next action to send ACH through the HealthSherpa-provided PCI proxy, require 200 OK, then submit |
| `Invalid dental selection` | Qualified-dental carrier (HCSC) got both `dental_plan_hios_id` and `attestations.pediatric_dental`, or neither | Send exactly one. On update, `null` the field you are clearing |
| `No plan found for dental_plan_hios_id: <id>` / `dental_plan_hios_id has a carrier mismatch` | Dental plan is a different carrier than the medical plan, or not eligible/in service area | Discover dental with a ZIP/FIPS-scoped dental quote (`POST /quotes` with `dental_search: true` and the member's `zip_code`/`fip_code`/`state`) so results are service-area accurate, and pick a plan whose **issuer matches the medical plan's issuer** — HCSC's "BlueCare Dental" shares the medical issuer prefix (e.g. `36096`). The `GET /plans?...&dental_only=true` list is state-level and not service-area filtered, so confirm service area with the dental quote before enrolling. A different-carrier or out-of-area dental plan is rejected |
| `not eligible for API enrollment` | Plan's `api_enrollment` is false for that carrier/state | Re-quote; route to Deeplink when `api_enrollment:false` |
| SEP date outside window | `event_date` is too far in the past or future | Use the selected reason's `event_date_days_before` and `event_date_days_after` offsets relative to today |

_Generated from HealthSherpa staging `enrollment_requirements` and validated against staging application flows. Plan IDs are 2026 examples; always quote for current inventory and poll asynchronous submission status._
