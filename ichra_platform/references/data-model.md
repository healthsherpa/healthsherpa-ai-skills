# Application Data Model Reference

## Complete Request Example

A full `POST /api/v1/applications` request with all required and recommended fields:

```json
{
  "plan_hios_id": "53901AZ1490005",
  "plan_year": 2026,
  "applicants": {
    "primary": {
      "first_name": "Jane",
      "last_name": "Smith",
      "date_of_birth": "1990-05-15",
      "gender": "female",
      "email": "jane.smith@example.com",
      "phone": "2125550100",
      "phone_type": "cell",
      "ssn": "123456789",
      "us_citizen": true,
      "resides_in_state": true,
      "uses_tobacco": false,
      "race_ethnicity": "decline_to_answer",
      "hispanic_origin": "decline_to_answer",
      "language_spoken": "english",
      "language_written": "english",
      "signature": "Jane Smith"
    }
  },
  "residential_address": {
    "street_address_1": "123 Main St",
    "street_address_2": "Apt 4B",
    "city": "Phoenix",
    "state": "AZ",
    "zip_code": "85001",
    "fips_code": "04013"
  },
  "agent_of_record": {
    "first_name": "James",
    "last_name": "Bond",
    "national_producer_number": "12345678",
    "email": "agent@example.com",
    "phone": "2125550101"
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
      "phone": "2125550100",
      "address": {
        "street_address_1": "789 Corporate Blvd",
        "city": "Phoenix",
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

Note: `applicants.primary.signature` (the typed name) is on the applicant object. `signatures.signature_date` (the signing date) is a separate top-level object. Both are required for submission.

## Top-Level Request Fields

| Field | Type | Req | Notes |
|---|---|---|---|
| `plan_hios_id` | string | Yes | 14-char HIOS plan ID (e.g., `53901AZ1490005`) |
| `plan_year` | int | Yes | e.g., 2026 |
| `applicants` | object | Yes | `{primary: {…}, dependents: [{…}]}` |
| `residential_address` | address | Yes | See Address Object |
| `external_id` | string | No | Your tracking ID. Optional everywhere. |
| `tpa_slug` | string | No | HealthSherpa-assigned TPA identifier |
| `_agent_id` | string | No | HealthSherpa-assigned agent slug |
| `desired_effective_date` | date | No | Optional. When omitted, carrier auto-determines the date. When provided, must match one of the carrier's valid dates for the SEP reason — some carriers return no valid dates at all, rejecting any value. Omit unless you have a specific reason to override. |
| `mailing_address` | address+ | No | Address fields plus `different_from_home_address` (bool), `billing_use_only` (bool) |
| `american_indian_or_alaskan_native_in_household` | boolean | No | |
| `analytics` | object | No | `{utm_source, utm_campaign, utm_medium, utm_content, utm_term}` |

## Address Object

Reused for `residential_address`, `mailing_address`, `guardian.mailing_address`.

```json
{
  "street_address_1": "123 Main St",
  "street_address_2": "Apt 4B",
  "city": "Phoenix",
  "state": "AZ",
  "zip_code": "85001",
  "fips_code": "04013"
}
```

Use `street_address_1` / `street_address_2`. NOT `street_line_1` or `address_line_1`.

## Applicants

```json
{
  "applicants": {
    "primary": { /* fields */ },
    "dependents": [{ /* fields */ }]
  }
}
```

### All Applicant Fields (primary + dependents)

| Field | Type | Notes |
|---|---|---|
| `member_id` | string | HealthSherpa-assigned. Read-only on create. Required on PUT for matching. |
| `first_name`, `last_name` | string | Required |
| `middle_name`, `suffix` | string | Optional |
| `date_of_birth` | date | Required. ISO 8601. |
| `gender` | enum | Required. `male`, `female`. NOT `sex`. |
| `ssn` | string | 9 digits, no dashes. Masked in responses. |
| `itin` | string | Individual Taxpayer ID (alternative to SSN) |
| `uses_tobacco` | boolean | NOT `tobacco_use` |
| `us_citizen` | boolean | |
| `resides_in_state` | boolean | |
| `race_ethnicity` | enum | `white`, `black_or_african_american`, `asian_indian`, `chinese`, `filipino`, `japanese`, `korean`, `vietnamese`, `native_hawaiian`, `guamanian_or_chamorro`, `samoan`, `american_indian_or_alaskan_native`, `decline_to_answer` |
| `hispanic_origin` | enum | `yes`, `no`, `decline_to_answer` |
| `hispanic_origin_description` | enum | Only when `hispanic_origin: "yes"`. Values: `cuban`, `mexican_mexican_american_or_chicanx`, `puerto_rican`, `other_hispanic_latino_or_spanish_origin`, `decline_to_answer` |
| `existing_coverage` | object | `{has_existing_coverage, plan_replaces_existing_coverage, type, insurer, policy_id, policyholder_name, start_date, term_date, will_continue}`. `type` is `"issuer"` or `"government"`. |
| `signature` | string | Typed full legal name. Required on primary for submission. |

### Primary-Only Fields

| Field | Type | Notes |
|---|---|---|
| `external_id` | string | Optional. Unique within application. |
| `email` | string | |
| `phone` | string | |
| `phone_type` | enum | `cell`, `home`, `work` |
| `language_spoken`, `language_written` | enum | `english`, `spanish`, `arabic`, `chinese`, `french_creole`, `french`, `german`, `gujarati`, `hindi`, `korean`, `polish`, `portuguese`, `russian`, `tagalog`, `urdu`, `vietnamese`, `other` |
| `guardian` | object | See Guardian |
| `responsible_party` | object | See EnrollConnect YML spec for full fields |
| `translator` | object | `{first_name, middle_name, last_name, reason}` |
| `children_live_with_primary` | boolean | |
| `has_pediatric_dental_coverage` | boolean | |
| `previously_applied` | boolean | |
| `previously_applied_member_id` | string | |


### Dependent-Only Fields

| Field | Type | Notes |
|---|---|---|
| `relationship` | enum | Required. `spouse`, `child`, `domestic_partner`, `stepchild`, etc. |

### Applicant Matching on PUT

- Include `member_id` (HealthSherpa-assigned) → updates that applicant
- Omit `member_id` on a dependent → creates new dependent
- Omit existing dependent from array → removes them
- `external_id` is metadata only — never used for matching

### Guardian Sub-Object

```json
{
  "guardian": {
    "first_name": "John",
    "last_name": "Doe",
    "gender": "male",
    "relationship": "parent",
    "date_of_birth": "1970-01-15",
    "ssn_last_four": "1234",
    "email": "guardian@example.com",
    "home_phone": "2125550100",
    "mailing_address": {
      "street_address_1": "123 Main St",
      "city": "Phoenix",
      "state": "AZ",
      "zip_code": "85001"
    }
  }
}
```

## Agent of Record

Strongly recommended. Without agent of record, the enrollment may not be attributed to the correct agent/broker. Include at minimum `first_name`, `last_name`, `national_producer_number`, `email`, and `phone`.

```json
{
  "agent_of_record": {
    "first_name": "Jane",
    "last_name": "Agent",
    "national_producer_number": "12345678",
    "carrier_producer_code": "ABC123",
    "state_license_number": "SL456",
    "email": "agent@example.com",
    "phone": "2125550100",
    "address": {
      "street_address_1": "100 Agent Blvd",
      "city": "Phoenix",
      "state": "AZ",
      "zip_code": "85001"
    },
    "signature": "Jane Agent"
  }
}
```

## HRA & Employer

**Expected for all ICHRA enrollments.** The `hra` object is essential for HRA tracking, employer contribution calculations, and reimbursement workflows. The `employer` sub-object — specifically `name`, `fein`, and `address` — allows HealthSherpa to associate the enrollment with the correct employer group. Without these fields, employer-level reporting, group management, and carrier coordination will not function correctly. Always include the full `hra` object with employer details on every ICHRA enrollment.

```json
{
  "hra": {
    "offered_hra": true,
    "type": "ichra",
    "amount": 500,
    "contribution_covers": "premium",
    "start": "2026-01-01",
    "premium_payer": "employer",
    "household_size": 2,
    "annual_household_income": 75000,
    "annual_household_income_determination": "self_reported",
    "employer": {
      "name": "Acme Corp",
      "external_id": "emp_123",
      "phone": "2125550100",
      "fein": "123456789",
      "address": {
        "street_address_1": "789 Corporate Blvd",
        "city": "Phoenix",
        "state": "AZ",
        "zip_code": "85001",
        "fips_code": "04013"
      }
    }
  }
}
```

Employer address is nested under an `address` sub-object within the employer, using the same address schema as `residential_address`.

## Special Enrollment Period

```json
{
  "special_enrollment_period": {
    "event_type": "offered_ichra",
    "event_date": "2026-06-01"
  }
}
```

Common values: `offered_ichra`, `offered_qsehra`, `loss_of_mec`, `birth`, `adoption`, `marriage`, `relocation`, `domestic_partnership`, `released_from_incarceration`, `lost_aptc`.

Each carrier supports a different subset. Always send the canonical value — HealthSherpa maps it to the carrier's internal format. If a carrier doesn't support a given SEP reason, the API returns a 422 with a descriptive error.

## Attestations

```json
{
  "attestations": {
    "electronic_signature_consent": true,
    "agrees_issuer_attestations": true,
    "broker_signature_attestation": true,
    "disclosure_statement_accepted": true,
    "coverage_replacement_attestation_accepted": false,
    "pediatric_dental": "purchased_separately",
    "pediatric_dental_attestation": true,
    "agent_submitted_application": true,
    "agent_provided_consumer_marketing_materials": true,
    "agent_advised_consumer_of_product_features": true,
    "agent_retained_signed_application_copy": true
  }
}
```

Most fields are booleans. `pediatric_dental` is a string enum (`"purchased_separately"` or `"not_applicable"`) — get the legal text and valid options from `GET /plans/:hios_id?plan_year=2026&include=enrollment_requirements`. Present legal text to the consumer before setting values.

## Signatures

```json
{
  "signatures": {
    "signature_date": "2026-06-15",
    "pediatric_dental_signature": "Jane Smith",
    "pediatric_dental_signature_date": "2026-06-15",
    "translator_signature_date": "2026-06-15",
    "state_supplement_primary_signature": "Jane Smith",
    "state_supplement_spouse_signature": "John Smith",
    "state_supplement_disclosures_signature": "Jane Smith"
  }
}
```

`signature_date` is the application signing date. Required for submission.

The primary applicant's typed signature goes under `applicants.primary.signature` — NOT under the `signatures` object. This split is intentional and both are required for a successful submit.

State supplements by state: CO (`primary`, `disclosures`), UT (`primary`, `spouse`), NJ (`primary`).

## Communication Preferences

```json
{
  "communication_preferences": {
    "application_notification_email": true,
    "application_notification_call": false,
    "application_notification_text": true,
    "email_contact_consent": true,
    "marketing_contact_consent": false,
    "decline_marketing_contact": false,
    "preferred_communication_method": "email",
    "agrees_hsa_contact_opt_in": false
  }
}
```

## Response-Only Fields

| Field | Description |
|---|---|
| `application_id` | HealthSherpa ID (e.g., `HSA000000001`) |
| `policy_status` | See Policy Status Values below |
| `document_status` | `none_needed`, `required`, `uploaded`, `verified`, `denied` |
| `sep_reason` | Echoed SEP reason (may differ from input — see API Behavior Notes in SKILL.md) |
| `policies` | Array of policy objects (populated post-submission) |
| `payment_instructions` | Carrier payment configuration (see [payment-and-documents.md](payment-and-documents.md)) |
| `payment` | Carrier-reported payment data (may be null for days/weeks) |
| `errors` | Validation/prerequisite errors — empty array means ready to submit |
| `next_actions` | HATEOAS links: `[{rel, href, method}]` — use these to drive UI. Includes `change_plan` when plan changes are available. |
| `supports_changes` | Whether the carrier supports post-enrollment changes for this application |
| `can_change_plan` | Whether a plan change is currently allowed (requires Open Enrollment + carrier support) |
| `can_report_change` | Whether demographic changes can be submitted |
| `created_at`, `updated_at`, `submitted_at` | ISO 8601 timestamps |

### Events Timeline

The `GET /applications/:id` response includes an `events` array that provides a chronological audit trail of all significant changes to the application. Use `include_events=false` to omit the array for performance.

| Field | Type | Description |
|---|---|---|
| `type` | enum | `submitted`, `changed`, `document_status_changed`, `cancelled`, `submission_failed`, `policy_status_updated` |
| `occurred_at` | datetime | ISO 8601 timestamp |
| `target` | string | `"application"` or `"applicant"` — which record was affected |
| `member_id` | string | For applicant-level `changed` events: the affected applicant's member_id |
| `changes` | array | For `changed` events: `[{field, from, to}]`. Sensitive fields (SSN, ITIN) are redacted. |
| `response_code` | string | For `submitted` events: carrier response code |
| `old_status`, `new_status` | string | For `document_status_changed` and `policy_status_updated` events |

```json
{
  "events": [
    {
      "type": "submitted",
      "occurred_at": "2026-05-01T14:30:00Z",
      "response_code": "success"
    },
    {
      "type": "changed",
      "occurred_at": "2026-05-02T10:15:00Z",
      "target": "applicant",
      "member_id": "HSM001145562",
      "changes": [
        {"field": "email", "from": "old@example.com", "to": "new@example.com"}
      ]
    },
    {
      "type": "policy_status_updated",
      "occurred_at": "2026-05-03T09:00:00Z",
      "old_status": "pending_effectuation",
      "new_status": "effectuated"
    }
  ]
}
```

### Policy Status Values

| Status | Meaning | Terminal? |
|---|---|---|
| `draft` | Created, not yet submitted | No |
| `submission_failed` | Async carrier submission failed. Check `errors` for details. | No (can re-submit after fixing) |
| `sep_docs_required` | SEP documentation must be uploaded before carrier processes | No |
| `sep_docs_under_review` | SEP docs uploaded, carrier reviewing | No |
| `pending_effectuation` | Submitted to carrier, awaiting confirmation | No |
| `effectuated` | Carrier confirmed, coverage active | Yes (but can be cancelled/terminated) |
| `cancelled` | Prospective cancellation | Yes |
| `terminated` | Retroactive or immediate termination | Yes |

### Response Normalization

Some response values are normalized differently from what was sent. Code defensively:

```
Input: phone_type: "cell"         → Response may return: "Cell"
Input: hispanic_origin: "decline_to_answer" → Response may return: "decline"
Input: event_type: "offered_ichra" → Response may return: "ichra_qsehra"
```

Always normalize response values before comparing to stored input. Store your original values locally as the source of truth.

### List vs. Detail Response Structures

The `GET /applications` (list) and `GET /applications/:id` (detail) responses have **different structures**. Code must handle both:

**Detail response** (also `POST /applications`, `PUT /applications/:id`):
```json
{
  "application_id": "HSA000739499",
  "application": {
    "applicants": { "primary": { "first_name": "Jane", ... }, "dependents": [...] },
    "plan_hios_id": "13877AZ0070072",
    "policy_status": "draft",
    ...
  },
  "errors": [...],
  "next_actions": [...],
  "payment_instructions": { ... }
}
```
- `application_id` is at the **top level** (NOT inside `application`)
- App data (applicants, plan, address, etc.) is **nested under `application`**
- You must flatten these together for use

**List response** (`GET /applications`):
```json
{
  "applications": [
    {
      "application_id": "HSA000739499",
      "primary_applicant": { "member_id": "HSM001145562", "first_name": "Jane", "last_name": "Doe", ... },
      "external_id": "your-id",
      "policy_status": "draft",
      "state": "AZ",
      ...
    }
  ],
  "pagination": { "total_count": 61, "limit": 25, "offset": 0 }
}
```
- Uses `primary_applicant` (flat) not `applicants.primary` (nested)
- Uses `pagination.total_count` not `pagination.total`
- Use `application_id` to link to the detail view
