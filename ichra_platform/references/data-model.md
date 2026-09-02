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
| `dental_plan_hios_id` | string | Cond | HIOS ID of a qualified stand-alone dental plan. Only for carriers that offer qualified dental (currently HCSC). Exactly one of `dental_plan_hios_id` or `attestations.pediatric_dental` must be present — not both. Must be the same carrier as the medical plan and available in the member's service area. See "Qualified Dental (HCSC)" in SKILL.md. |
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
| `gender` | enum | Required. Common values are `male` and `female`; the 2026 AmeriHealth Caritas Next contract also accepts `x`. NOT `sex`. |
| `ssn` | string | 9 digits, no dashes. Masked in responses. BCBS Michigan requires either SSN or ITIN for every applicant. For AmeriHealth Caritas Next, SSN is required when the applicant is age one or older on the request date. |
| `itin` | string | Individual Taxpayer ID (alternative to SSN). BCBS Michigan accepts it in place of SSN. |
| `uses_tobacco` | boolean | NOT `tobacco_use` |
| `us_citizen` | boolean | |
| `resides_in_state` | boolean | |
| `currently_incarcerated` | boolean | Carrier-specific. Include for Anthem and Wellpoint. |
| `marital_status` | enum | Carrier-specific. Anthem and Wellpoint use `single`, `married`, and `domestic_partner`. The 2026 AmeriHealth Caritas Next values are `married`, `unmarried`, `divorced`, and `widowed`. |
| `race_ethnicity` | enum | `white`, `black_or_african_american`, `asian_indian`, `chinese`, `filipino`, `japanese`, `korean`, `vietnamese`, `native_hawaiian`, `guamanian_or_chamorro`, `samoan`, `american_indian_or_alaskan_native`, `decline_to_answer` |
| `hispanic_origin` | enum | `yes`, `no`, `decline_to_answer` |
| `hispanic_origin_description` | enum | Only when `hispanic_origin: "yes"`. Values: `cuban`, `mexican_mexican_american_or_chicanx`, `puerto_rican`, `other_hispanic_latino_or_spanish_origin`, `decline_to_answer` |
| `hra` | object | Send for the primary and every non-child dependent when `enrollment_requirements.hra.per_applicant_hra` is true. Do not send for children. |
| `existing_coverage` | object | `{has_existing_coverage, plan_replaces_existing_coverage, type, insurer, policy_id, policyholder_name, start_date, term_date, will_continue}`. `type` is `"issuer"` or `"government"`. |
| `signature` | string | Typed full legal name. Required on primary for submission. |

### Primary-Only Fields

| Field | Type | Notes |
|---|---|---|
| `external_id` | string | Optional. Unique within application. |
| `email` | string | |
| `phone` | string | |
| `phone_type` | enum | `cell`, `home`, `work`. The 2026 AmeriHealth Caritas Next values are `cell` and `work`; `cell` maps to the carrier's Personal option. |
| `language_spoken`, `language_written` | enum | `english`, `spanish`, `arabic`, `chinese`, `french_creole`, `french`, `german`, `gujarati`, `hindi`, `korean`, `polish`, `portuguese`, `russian`, `tagalog`, `urdu`, `vietnamese`, `other`. Both fields are required for the 2026 AmeriHealth Caritas Next primary applicant. |
| `has_communication_impairment` | boolean | Required when returned in `enrollment_requirements.applicants.primary`. Both `true` and `false` are valid answers. |
| `communication_impairment_format` | enum | Required when `has_communication_impairment` is true. `braille`, `large_print`, `audio`, `other`. |
| `communication_impairment_format_other` | enum | Required when format is `other`. `encrypted_audio_cd`, `encrypted_data_cd`. |
| `guardian` | object | See Guardian |
| `responsible_party` | object | See Responsible Party Sub-Object below. |
| `translator` | object | `{first_name, middle_name, last_name, reason}` |
| `children_live_with_primary` | boolean | |
| `has_pediatric_dental_coverage` | boolean | |
| `previously_applied` | boolean | |
| `previously_applied_member_id` | string | |


### Dependent-Only Fields

| Field | Type | Notes |
|---|---|---|
| `relationship` | enum | Required. Use the options returned for the selected plan. BCBS Michigan and 2026 AmeriHealth Caritas Next medical plans accept `spouse` and `child`. Other values are carrier- and state-specific. |

### Applicant Matching on PUT

- Include `member_id` (HealthSherpa-assigned) → updates that applicant
- Omit `member_id` on a dependent → creates new dependent
- Omit existing dependent from array → removes them
- `external_id` is metadata only — never used for matching
- Treat `member_id` as opaque and match it to your own applicant record. Never construct it or assign IDs by dependent array position.

For a submitted application whose carrier supports changes, preserve all identity fields in the full PUT. Changing both first and last name together is rejected. Changing date of birth together with a name is also rejected. Make one supported identity correction at a time.

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

For a primary applicant under 18, include the required guardian identity and
contact fields. Set `applicants.primary.signature` to the guardian's exact full
name; a different name is rejected. Application reads return the accepted
value under `applicants.primary.guardian.signature`.

### Responsible Party Sub-Object

Plan requirements can return
`applicants.primary.responsible_party` with the section `question`,
`instruction`, relationship options, and field-level requiredness. An
`applies: true` value means the carrier supports the section; use its question
and conditional rules to determine whether the consumer must complete it. A
requirements response can use this shape:

```json
{
  "responsible_party": {
    "applies": true,
    "question": "Is someone else responsible for payment?",
    "instruction": "Carrier-defined responsible-party instruction.",
    "uses_minor_dependent_enrollment_gate": false,
    "relationship": {
      "required_when_section_applies": true,
      "options": [
        {"value": "parent", "label": "Parent"},
        {"value": "grandparent", "label": "Grandparent"},
        {"value": "legal_guardian", "label": "Legal guardian"},
        {"value": "other", "label": "Other"}
      ]
    },
    "email": {"required_when_section_applies": false},
    "applicant_can_release_dependent_info": {
      "required_when_section_applies": false
    },
    "dependent_same_address": {
      "required_when_section_applies": false
    },
    "dependent_address": {
      "required_when_dependent_same_address_false": false
    }
  }
}
```

Display the exact returned question, `instruction`, labels, options, and
conditions. When the consumer identifies another person as responsible for
payment, send the object under `applicants.primary`. Applicant identity uses
`gender`; this sub-object uses `sex`.

```json
{
  "responsible_party": {
    "first_name": "Alex",
    "middle_name": "J",
    "last_name": "Doe",
    "sex": "female",
    "date_of_birth": "1985-02-10",
    "relationship": "parent",
    "phone": "3025550103",
    "email": "alex.doe@example.com",
    "street_address_1": "123 Main St",
    "street_address_2": "",
    "city": "Wilmington",
    "state": "DE",
    "zip_code": "19801"
  }
}
```

The flat address fields belong to the responsible party. Apply requirement
conditions as follows:

- When `uses_minor_dependent_enrollment_gate` is true, collect
  `enrollment_includes_minor_or_dependent`. If the consumer answers false,
  omit the remaining responsible-party fields.
- Once the section applies, require each field whose
  `required_when_section_applies` value is true.
- Send `dependent_same_address` when that field is required. If the answer is
  false and `required_when_dependent_same_address_false` is true, send the
  dependent's address under the nested `dependent_address` object.
- If the `uses_minor_dependent_enrollment_gate` configuration value is false,
  omit only `enrollment_includes_minor_or_dependent`. Use the returned question
  and household context to decide whether to send the remaining
  `responsible_party` fields.

For 2026 AmeriHealth Caritas Next plans, this section applies to child-only
applications when someone other than the application contact is responsible
for payment. If no additional responsible party applies, omit
`applicants.primary.responsible_party`. Do not place HRA employer information
in this object.

## Agent of Record

Strongly recommended. Without agent of record, the enrollment may not be attributed to the correct agent/broker. Include at minimum `first_name`, `last_name`, `national_producer_number`, `email`, and `phone`.

```json
{
  "agent_of_record": {
    "first_name": "Jane",
    "last_name": "Agent",
    "national_producer_number": "12345678",
    "carrier_producer_code": "KJNNKJSJUY",
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

`carrier_producer_code` is carrier-specific. For Anthem and Wellpoint broker-assisted enrollments, it is the carrier-issued encrypted agent TIN, not an agency identifier. It must contain exactly 10 uppercase letters and end in `Y` or `Z`.

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

When `plan.enrollment_requirements.hra.per_applicant_hra` is true, also send
`hra` under `applicants.primary` and every non-child dependent relationship
supported by the selected carrier. Do not send per-applicant HRA answers for
children. Use the returned
`contribution_covers.ichra` and `contribution_covers.qsehra` values. If the
QSEHRA household condition applies, send the answer at top-level
`qsehra_both_employers_claim_reimbursement`.

BCBS Michigan medical requires at least the top-level `hra.offered_hra` answer and supports complete ICHRA and QSEHRA blocks, including employer data.

## Special Enrollment Period

```json
{
  "special_enrollment_period": {
    "event_type": "offered_ichra",
    "event_date": "2026-06-01"
  }
}
```

Each carrier supports a different subset. Read
`plan.enrollment_requirements.special_enrollment_period.event_types` and send
only a returned key. Each value states how far the event date may be before or
after today and whether documentation is required. HealthSherpa maps the
canonical key to the carrier's internal format.

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
    "agent_submitted_application": true,
    "agent_provided_consumer_marketing_materials": true,
    "agent_advised_consumer_of_product_features": true,
    "agent_retained_signed_application_copy": true,
    "consumer_working_with_agent": true
  }
}
```

Most fields are booleans. `pediatric_dental` is a string enum. When
`enrollment_requirements.attestations.pediatric_dental.required` is true,
present `attestation_text`, display the labels from its `{value: label}`
`options` map, and submit the selected key. HCSC qualified-dental applications
still require exactly one of this attestation or `dental_plan_hios_id` when the
returned `required` flag is false.

For carriers that offer qualified dental (currently HCSC), `attestations.pediatric_dental` is the **attestation** path for satisfying the ACA pediatric dental requirement. The alternative is buying a stand-alone dental plan via the top-level `dental_plan_hios_id`. Send exactly one of the two — both, or neither, returns a `422` (`Invalid dental selection`). See "Qualified Dental (HCSC)" in SKILL.md.

Create and detail responses echo the saved path at
`response.application.dental_plan_hios_id` and
`response.application.pediatric_dental`. Use those fields to confirm which
qualified-dental path was saved.

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
    "agrees_hsa_contact_opt_in": true
  }
}
```

When plan lookup returns
`enrollment_requirements.communication_preferences.email_contact_consent.required: true`,
display the returned question and content, then send an explicit boolean at
`communication_preferences.email_contact_consent`. A false answer is valid and must not be treated as missing.

When plan lookup returns
`enrollment_requirements.communication_preferences.agrees_hsa_contact_opt_in`,
display its exact `content` and boolean option labels. The carrier content
defines whether the answer represents HSA enrollment, account-opening help,
contact consent, or data-sharing consent. Do not infer the meaning from the API
field name.

| Requirement state | Form and payload behavior |
|---|---|
| Parent or field absent or null | Do not render HSA content or options, and omit the payload field. |
| Object with `required: false` | Render the returned `content` and option labels as optional. Omission is valid; preserve an answered `true` or `false`. |
| Object with `required: true` | Require `true` or `false` before submit. Omission or null is invalid. |

Create and full PUT requests send the answer at
`communication_preferences.agrees_hsa_contact_opt_in`. Both booleans satisfy
requiredness, and false does not block enrollment in the HSA-eligible health
plan. Non-boolean values are invalid. When a required answer is missing, block
submit locally and use the draft's `missing_required_field` error at
`communication_preferences.agrees_hsa_contact_opt_in` for correction. If a
submit is attempted anyway, the API returns `422`.

Application reads preserve both boolean values, including an explicit false.
Do not use the similarly named `hsa_contact_opt_in`.

## Response-Only Fields

| Field | Description |
|---|---|
| `application_id` | HealthSherpa ID (e.g., `HSA000000001`) |
| `policy_status` | See Policy Status Values below |
| `document_status` | `none_needed`, `required`, `uploaded`, `verified`, `denied` |
| `sep_reason` | Echoed SEP reason (may differ from input — see API Behavior Notes in SKILL.md) |
| `policies` | Array of policy objects (populated post-submission) |
| `payment_instructions` | Carrier payment configuration. When `payment_required_with_submission` is `true`, provide the payment method before submit (see [payment-and-documents.md](payment-and-documents.md)). |
| `payment` | Carrier-reported payment data (may be null for days/weeks) |
| `errors` | Validation/prerequisite errors — empty array means ready to submit |
| `next_actions` | HATEOAS links: `[{rel, href, method}]`. Use these to drive UI. Includes `payment_method` when in-flow payment is available. |
| `supports_changes` | Whether the carrier supports post-enrollment changes for this application |
| `can_change_plan` | Whether a plan change is currently allowed (requires Open Enrollment + carrier support) |
| `can_report_change` | Whether demographic changes can be submitted |
| `created_at`, `updated_at`, `submitted_at` | ISO 8601 timestamps |

### Action Validation

For application `HSA000000001`, validate action links before calling them:

| Action | Required `rel` | Method | Exact path |
|---|---|---|---|
| Full update | `update` | `PUT` | `/api/v1/applications/HSA000000001` |
| Submit | `submit` | `POST` | `/api/v1/applications/HSA000000001/submit` |
| Cancel | `cancel` | `POST` | `/api/v1/applications/HSA000000001/cancel` |
| Terminate | `terminate` | `POST` | `/api/v1/applications/HSA000000001/terminate` |

Require `supports_changes` and `can_report_change` before a demographic
update. Require `supports_changes`, `can_change_plan`, and a valid `update`
action before changing the plan through the full PUT payload. Never construct
or call an action that is not returned. A submitted update for a carrier with
`supports_changes: false` returns `422`; direct the member to the carrier
instead.

### Payment Method Response

`PUT /api/v1/applications/:id/payment_method` returns masked payment details. The current public integration supports ACH bank accounts.

| Field | Type | Description |
|---|---|---|
| `payment_method_type` | enum | `"bank_account"` |
| `payment_type` | enum | `"initial"`, `"both"`, or `"recurring"`. Current Anthem and Wellpoint flows use `"initial"` or `"both"`; `"recurring"` without an initial payment returns `422`. |
| `masked_account` | string | Account number masked except for the last four digits |
| `status` | enum | `"active"` or `"inactive"` |
| `is_recurring` | boolean | Whether the saved method covers recurring premiums |
| `withdraw_day` | integer or null | Recurring withdrawal day from 1 through 28; null for binder-only payments |

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
| `sep_docs_denied` | Carrier denied the submitted SEP proof; use returned actions or errors when present, otherwise escalate manually | No |
| `pending_effectuation` | Submitted to carrier, awaiting confirmation | No |
| `effectuated` | Carrier confirmed, coverage active | Yes (but can be cancelled/terminated) |
| `cancelled` | Policy never took effect (never effectuated), typically non-payment | Yes |
| `terminated` | Policy was active and later ended | Yes |

The `sep_docs_*` policy statuses are derived only while an application is
currently SEP-suspended. For other document workflows, use `document_status`
and do not infer a `sep_docs_*` policy status.

For a response that uses SEP suspension:

- `sep_docs_required`: prompt for the required document and upload it through
  `POST /applications/:id/supporting_documentation`.
- `sep_docs_under_review`: continue bounded GET polling or scheduled GET
  reconciliation. Alert when review exceeds the integration's documented
  service threshold; policy-status webhooks do not deliver this state.
- `sep_docs_denied`: stop passive polling and prompt for corrected
  documentation or manual escalation. Use a supporting-documentation action
  when one is present.

After document verification, continue monitoring for the later carrier and
policy status. No `sep_docs_*` state confirms effectuation.

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
      "status": "draft",
      "policy_status": "draft",
      "plan_year": 2026,
      "plan_hios_id": "13877AZ0070072",
      "issuer_hios_id": "13877",
      "policy_effective_date": null,
      "state": "AZ",
      "created_at": "2026-04-28T13:56:56.119Z",
      "updated_at": "2026-04-28T13:56:56.838Z"
    }
  ],
  "pagination": { "total_count": 61, "limit": 25, "offset": 0 }
}
```
- Uses `primary_applicant` (flat) not `applicants.primary` (nested)
- Uses `pagination.total_count` not `pagination.total`
- Use `application_id` to link to the detail view
- `status`, `policy_status`, `plan_year`, `policy_effective_date`,
  `created_at`, and `updated_at` are separate list-item fields.
- `plan_hios_id` and `issuer_hios_id` can be null; fetch detail when complete
  plan data is required.

List filters:

| Parameter | Description |
|---|---|
| `policy_status` | Filter by lifecycle status |
| `external_id` | Filter by the platform's tracking ID |
| `plan_year` | Filter by plan year |
| `issuer_hios_id`, `plan_hios_id` | Filter by issuer or plan |
| `employer_external_id` | Filter by employer |
| `updated_since` | Return applications updated after an ISO 8601 timestamp |
| `limit`, `offset` | Paginate results; limit defaults to 25 and has a maximum of 100 |
