# EnrollConnect API — Example Create & Submit Requests by Carrier

_Reference examples for building successful off-exchange ICHRA enrollments through the EnrollConnect API. Each example passes the target carrier's specific validations and was verified end-to-end against HealthSherpa staging._

> **Synthetic data only.** Every value below (names, SSN, FEIN, phone, email, agent NPN) is a placeholder. Replace all applicant, employer, and agent values with real data before sending. Never commit real PII/PHI to source control or logs.

## Before you start

> **Verified PY2026, 2026-06-10.** Every example below was run end-to-end against HealthSherpa staging (create -> upload doc where required -> submit -> 202). Plan IDs, ZIP, and FIPS are plan-year- and inventory-specific and rotate each year — treat them as illustrative and always quote for current inventory. Carrier documentation requirements are a point-in-time snapshot; the create response is the source of truth (see the runtime rules below).

- **Server-side only.** Call EnrollConnect from a backend you control. Never embed the `x-api-key` in browser or mobile code.
- **Base URL (production):** `https://api.ichra.healthsherpa.com`  •  **Staging:** `https://api.ichra-staging.healthsherpa.com`
- **Auth:** send `x-api-key: <your key>` and `Content-Type: application/json` on every request.
- **Quote first.** The plan IDs, ZIP, and FIPS below are illustrative 2026 examples. In production, call `POST /api/v1/quotes` (with `off_ex: true`) to get a current `plan_hios_id` for the member's county, and confirm `api_enrollment: true` on the plan before creating an application.
- **Fetch requirements.** Call `GET /api/v1/plans/{hios_id}?plan_year=2026&include=enrollment_requirements` to get the carrier's exact attestation text and SEP rules. Render the returned legal text to the consumer — do not use generic labels.
- **`external_id` is not de-duplicated.** Submitting two creates with the same `external_id` produces two separate applications. Store the `application_id` returned by create and use it for every follow-up call; look up by `external_id` before retrying a create.
- **Use real, current dates.** `special_enrollment_period.event_date` must fall within the carrier's SEP window (commonly 60 days before or after the event). The dates in these examples are illustrative — replace them with the member's actual event and signature dates.

## Conventions used in every example (best practices)

- **Enums are lowercase snake_case**; dates are ISO 8601 (`YYYY-MM-DD`); money is a string (`"50.50"`).
- **Field names:** `street_address_1`/`street_address_2` (not `street_line_*`), `gender` (not `sex`), `uses_tobacco` (not `tobacco_use`).
- **Signatures are split** — `applicants.primary.signature` (typed legal name) **and** `signatures.signature_date`. Both are required to submit.
- **Always include `hra`** with employer `name`, `fein`, and nested `address`, plus an `agent_of_record` (`first_name`, `last_name`, `national_producer_number`).
- **SEP:** these examples use `event_type: "offered_ichra"` (the ICHRA offer), the canonical reason for ICHRA enrollment. Send the canonical value; HealthSherpa maps it to the carrier's format.
- **`desired_effective_date` is omitted** so the carrier auto-determines the effective date (recommended).
- **`external_id` is optional and appears in two places** — at the top level (your application tracking id) and on each applicant (your member/person id). Both are correlation metadata echoed back on reads; neither is used to match or de-duplicate records. Use them to map HealthSherpa applications and members back to your own system.
- **SSN is carrier-specific.** Some carriers require the primary's SSN to submit; others do not. Each carrier section below states which. When required, send a valid SSN (or ITIN where the carrier accepts one).
- **These payloads carry PII/PHI.** SSN, DOB, signatures, addresses, and the other identity fields below are sensitive data — securing, storing, and redacting them in your own systems is your responsibility, not HealthSherpa's. See **Security & Credentials → PII / PHI** in `SKILL.md`.

## Quote for a current plan

Quote first to get a `hios_id` valid for the member's county and to confirm `api_enrollment: true`. The quote applicant shape differs from the application payload: it uses `age`, `gender`, `smoker`, and `relationship` (and the location keys are `zip_code` + `fip_code`).

```bash
curl -sS -X POST 'https://api.ichra.healthsherpa.com/api/v1/quotes' \
  -H "x-api-key: $HS_API_KEY" -H 'Content-Type: application/json' \
  -d '{
    "off_ex": true,
    "plan_year": 2026,
    "zip_code": "85001",
    "fip_code": "04013",
    "applicants": [
      { "age": 36, "gender": "female", "smoker": false, "relationship": "primary" }
    ]
  }'
```

Each returned plan includes `hios_id` plus `api_enrollment` and `deeplink_enrollment` flags. Use the plan's `hios_id` as `plan_hios_id` when you create the application. Enroll through the API only when `api_enrollment: true`; when it is `false`, route the enrollment through Deeplink.

## The flow (3 calls; 4 for document-required carriers)

**1. Create** — `POST /api/v1/applications` with the carrier payload below. A `201` with an empty `errors` array means it's ready to submit. If `errors` contains `supporting_documentation_required` (or `policy_status` is `sep_docs_required`), upload a document first.

```bash
curl -sS -X POST 'https://api.ichra.healthsherpa.com/api/v1/applications' \
  -H "x-api-key: $HS_API_KEY" -H 'Content-Type: application/json' \
  -d @create_payload.json
```

**2. (If required) Upload SEP documentation** — `POST /api/v1/applications/{application_id}/supporting_documentation`. `document_type` must be `"sep"` (only accepted value). **Maximum file size is carrier-dependent** — most carriers allow ~2 MB, but some are higher (currently Oscar 50 MB, HCSC 10 MB) and limits change over time. Keep uploads as small as possible; if a file is rejected for size, check the current limit for that carrier with your account manager.

```bash
curl -sS -X POST 'https://api.ichra.healthsherpa.com/api/v1/applications/{application_id}/supporting_documentation' \
  -H "x-api-key: $HS_API_KEY" -H 'Content-Type: application/json' \
  -d '{"file":{"filename":"sep_proof.pdf","content_type":"application/pdf","content_base64":"<base64 of the file>"},"document_type":"sep"}'
```

**3. Submit** — `POST /api/v1/applications/{application_id}/submit` (no body). Returns `202 Accepted`; the submission is processed asynchronously.

```bash
curl -sS -X POST 'https://api.ichra.healthsherpa.com/api/v1/applications/{application_id}/submit' \
  -H "x-api-key: $HS_API_KEY" -H 'Content-Type: application/json'
```

**4. Track** — poll `GET /api/v1/applications/{application_id}` (or use webhooks) for `pending_effectuation` → `effectuated`. For document carriers the path is `sep_docs_required` → `sep_docs_under_review` → `pending_effectuation`.

---

## Carrier examples

All carriers below are `api_enrollment: true`. Every payload includes the attestation set that carrier requires (verified via its `enrollment_requirements`). Send the JSON shown to `POST /api/v1/applications`, then run the **Submit** call above.

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

### UnitedHealthcare

- **Plan (example):** `40702AZ0060051` — AZ · ZIP `85001` · FIPS `04013` · plan year 2026
- **Required attestations:** `electronic_signature_consent`, `agrees_issuer_attestations`, `broker_signature_attestation`, `pediatric_dental`
- **SSN:** Not required — omit unless you collect it.
- **SEP documentation:** Not required for `offered_ichra`
- **Notes:** Straight-through to `pending_effectuation`. Enroll any UnitedHealthcare plan that returns `api_enrollment: true` for the member's county — swap `plan_hios_id` + address per state (quote first). Some states add a state-supplement signature (see "State-specific requirements" below); the plan's `enrollment_requirements` lists exactly which keys to send.

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
- **SEP documentation:** Not required for `offered_ichra`
- **Notes:** Straight-through to `pending_effectuation`.

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
- **SEP documentation:** Not required for `offered_ichra`
- **Notes:** Straight-through to `pending_effectuation`.

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

### Christus

- **Plan (example):** `98780LA0220003` — LA · ZIP `71101` · FIPS `22017` · plan year 2026
- **Required attestations:** `electronic_signature_consent`, `agrees_issuer_attestations`, `broker_signature_attestation`, `pediatric_dental`
- **SSN:** Not required — omit unless you collect it.
- **SEP documentation:** Not required for `offered_ichra`
- **Notes:** Straight-through to `pending_effectuation`.

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

### Antidote

- **Plan (example):** `68445AZ0600010` — AZ · ZIP `85001` · FIPS `04013` · plan year 2026
- **Required attestations:** `electronic_signature_consent`, `agrees_issuer_attestations`, `broker_signature_attestation`, `pediatric_dental`
- **SSN:** Required — include a valid SSN (or ITIN where the carrier accepts one).
- **SEP documentation:** Not required for `offered_ichra`
- **Notes:** Straight-through to `pending_effectuation`.

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

---

## Multi-member households

Add dependents under `applicants.dependents` (an array). Each dependent needs `relationship` (`spouse`, `child`, or `domestic_partner`) and the same identity fields as the primary, plus:

- `has_disability` and `full_time_student` (booleans) on every dependent. When `full_time_student` is `true`, also send `graduation_date`.
- A typed `signature` on each adult dependent (spouse or domestic partner); children do not sign.
- `ssn` only when the carrier requires it (see each carrier above); dependents may otherwise omit it.

The household HRA answer is carried by the top-level `hra` object — do not repeat it per applicant. The example below (primary + spouse + child) was verified end-to-end.

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
| New Jersey (NJ) | `state_supplement_primary_signature` |
| Colorado (CO) | `state_supplement_primary_signature`, `state_supplement_disclosures_signature` |
| Utah (UT) | `state_supplement_primary_signature`, `state_supplement_spouse_signature` (spouse only when a spouse is on the application) |

Example (`signatures` block for a NJ enrollment):

```json
{
  "signatures": {
    "signature_date": "2026-06-15",
    "state_supplement_primary_signature": "Jane Doe"
  }
}
```

**Forward-compatible rule (do this and you never need an updated example):** before create, call `GET /api/v1/plans/{hios_id}?plan_year=2026&include=enrollment_requirements` and inspect `enrollment_requirements.attestations`. For every `state_supplement_*` key present, send the corresponding `signatures.state_supplement_*` value (typed legal name). Carrier/state combinations that are not yet enabled today will surface their exact requirements through this response the moment they go live, so a payload built this way will succeed without any changes on your side.

## Common 422s and fixes

| Error references | Cause | Fix |
|---|---|---|
| `applicants.primary.signature` / `signatures.signature_date` | Signature split not honored | Send both — typed name on the applicant, date in `signatures` |
| `agrees_issuer_attestations` / `electronic_signature_consent` | Required attestation omitted | Include all attestations listed for that carrier |
| `hra` | HRA answer missing | Include the top-level `hra` block — it covers every adult on the application; do not repeat it per applicant |
| `applicants[n].has_disability` / `full_time_student` / `signature` | Required dependent fields omitted | Send `has_disability` and `full_time_student` on every dependent, and a `signature` on adult dependents |
| `ssn` (`invalid_field_value` on submit) | Carrier requires the primary's SSN | Include a valid `ssn` for carriers marked **SSN: Required** |
| `supporting_documentation_required` | Carrier requires SEP proof (e.g., BCBS AZ) | Upload via `/supporting_documentation` then submit |
| `not eligible for API enrollment` | Plan's `api_enrollment` is false for that carrier/state | Re-quote; route to Deeplink when `api_enrollment:false` |
| SEP date outside window | `event_date` too old/new for the SEP reason | Use a date within the carrier's window (typically 60 days before/after) |

_Generated from HealthSherpa staging `enrollment_requirements` and verified end-to-end. Plan IDs are 2026 examples — always quote for current inventory._
