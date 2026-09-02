# Quoting & Plan Selection Reference

## POST /quotes

ALWAYS use POST (not GET). Required fields are `zip_code`, `fip_code`, and `applicants`; include `off_ex: true` and `plan_year` for ICHRA quoting.

HRA/ICHRA data is part of the enrollment application (`POST /api/v1/applications`). To show an employee's net cost from a quote, subtract the monthly ICHRA contribution from each plan's `premium` client-side.

### Request

```json
{
  "zip_code": "85001",
  "fip_code": "04013",
  "state": "AZ",
  "plan_year": 2026,
  "off_ex": true,
  "applicants": [
    {"age": 35, "smoker": false, "relationship": "primary"},
    {"age": 33, "smoker": false, "relationship": "spouse"}
  ]
}
```

**CRITICAL: Always include `off_ex: true`.** The API only supports off-exchange enrollment. Without this flag, the API returns on-exchange plans which are not supported.

**IMPORTANT field name differences between QuoteConnect and EnrollConnect:**

| QuoteConnect (Quoting) | EnrollConnect (Enrollment) |
|---|---|
| `fip_code` | `fips_code` |
| `applicants[].age` (integer) | `applicants.primary.date_of_birth` (ISO date) |
| `applicants[].smoker` (boolean) | `applicants.primary.uses_tobacco` (boolean) |
| `applicants[].relationship`: `primary`, `spouse`, `dependent` | `applicants.dependents[].relationship`: `spouse`, `child`, etc. |
| Flat array of applicants | Nested object: `{primary: {...}, dependents: [...]}` |

### Response Structure

```json
{
  "plans": [ /* array of Plan objects */ ],
  "meta": {
    "result_count": 42
  }
}
```

### Optional Request Parameters

| Parameter | Type | Description |
|---|---|---|
| `dental_search` | boolean | Set `true` to query standalone dental plans instead of medical. Used for dental plan discovery (e.g., HCSC). |
| `issuer_hios_ids` | array of strings | Filter results to specific carriers by issuer HIOS ID prefix (e.g., `["36096"]` for Blue Cross and Blue Shield of Illinois / HCSC). Note: does not reliably filter `dental_search` results. |
| `filter` | string | Comma-separated list of field names to include in the response (e.g., `"name,premium,hios_id,metal_level"`). Reduces payload size. |

### Response Fields (Per Plan)

| Field | Type | Description |
|---|---|---|
| `hios_id` | string | 14-char plan ID. Store this — it's the key to everything. Also returned as `plan_hios_id` in some contexts. |
| `name` | string | Display name |
| `issuer` | object | `{name, hios_id, state, logo_url, payment_phone, customer_service_phone}` — carrier info. Use `logo_url` for carrier branding. See issuer Object section below for details. |
| `premium` | number | Final monthly premium after subsidies. With `off_ex: true` subsidies are ignored, so this equals `gross_premium`. Net the ICHRA amount client-side for the employee's cost. |
| `gross_premium` | number | Monthly premium before subsidies |
| `metal_level` | enum | `Bronze`, `Silver`, `Gold`, `Platinum`, `Catastrophic` |
| `plan_type` | enum | `HMO`, `PPO`, `EPO`, `POS` |
| `api_enrollment` | boolean | Plan supports EnrollConnect API enrollment |
| `deeplink_enrollment` | boolean | Plan supports deeplink enrollment |
| `cost_sharing` | object | See cost_sharing fields below |
| `benefits` | object | Benefit coverage details |
| `adult_dental` | boolean | Plan covers adult dental (relevant for dental plan grouping) |
| `dental_only` | boolean | Whether the result is dental-only. Require `false` when selecting medical coverage. |
| `hsa_eligible` | boolean | HSA-eligible plan |

### cost_sharing Field Names

The `cost_sharing` object uses abbreviated medical/drug field names — NOT human-readable names:

| Field | Type | Description |
|---|---|---|
| `medical_ded_ind` | string | Medical deductible, individual |
| `medical_ded_fam` | string or null | Medical deductible, family |
| `drug_ded_ind` | string or null | Drug deductible, individual |
| `drug_ded_fam` | string or null | Drug deductible, family |
| `medical_moop_ind` | string | Medical max out-of-pocket, individual |
| `medical_moop_fam` | string | Medical max out-of-pocket, family |
| `drug_moop_ind` | string or null | Drug max out-of-pocket, individual |
| `drug_moop_fam` | string or null | Drug max out-of-pocket, family |
| `medical_coins` | string or null | Medical coinsurance |
| `drug_coins` | string or null | Drug coinsurance |
| `network_tier` | string | Network tier label |
| `csr_type` | string | Cost-sharing reduction type |

Do NOT use names like `deductible_individual` or `moop_individual` — these do not exist in the response.

### issuer Object

The `issuer` object includes more fields than just `name` and `hios_id`:

| Field | Type | Description |
|---|---|---|
| `name` | string | Carrier display name |
| `hios_id` | string | Issuer HIOS ID prefix (e.g., `"68445"`) |
| `state` | string | State the issuer operates in |
| `logo_url` | string | Carrier logo image URL |
| `payment_phone` | string | Payment phone number (may be empty) |
| `customer_service_phone` | string or null | Customer service phone |

### Enrollment Routing

After quoting, route enrollment based on the flags on each plan:

```
api_enrollment == true        → POST /api/v1/applications (EnrollConnect)
deeplink_enrollment == true   → POST /public/ichra/off_ex (Deeplink)
both true                     → prefer EnrollConnect; choose Deeplink only before create
both false                    → not enrollable through HealthSherpa
```

NEVER attempt EnrollConnect for a plan with `api_enrollment: false`. The API will reject the request with a 422. A plan may have `deeplink_enrollment: true` but `api_enrollment: false` — always check both flags per plan.

Do not use Deeplink as an automatic retry after an EnrollConnect timeout or
ambiguous create response. Retry only when transport-level evidence proves the
request was never sent. Otherwise use bounded reconciliation and manual
intervention; a negative application lookup is not proof that an asynchronous
create did not occur.

### BCBS Michigan medical plans

BCBS Michigan uses two issuer families: `15560` for BCBSM PPO and `98185` for BCN HMO. Include those values in `issuer_hios_ids` when narrowing a Michigan medical quote, but still validate each returned plan:

- `off_ex` was requested as `true`
- `api_enrollment` is `true`
- `dental_only` is `false`
- `plan_type` is the product expected by the user

Use `hsa_eligible` to distinguish HSA plans. Do not infer product or enrollment support from the issuer ID alone, and do not hardcode the returned plan HIOS IDs across plan years.

### AmeriHealth Caritas Next medical plans

AmeriHealth Caritas Next issuer IDs are `72760` in Delaware, `67926` in
Florida, `38246` in Louisiana, `17414` in North Carolina, and `73107` in South
Carolina. South Carolina uses the First Choice Next consumer brand. Quote
current inventory and require `api_enrollment: true` and `dental_only: false`;
issuer presence does not guarantee plan availability for a specific ZIP.

For plan year 2026, `agrees_hsa_contact_opt_in` is absent from AmeriHealth
Caritas Next plan requirements even when `hsa_eligible` is true. Applicability
comes from `plan.enrollment_requirements.communication_preferences`, not
`hsa_eligible` or carrier identity. When the field is absent, do not render or
send it; if a selected plan returns it, follow the returned content, options,
and requiredness.

## Dental Plan Quoting (HCSC / qualified-dental carriers)

Some carriers (currently **HCSC** — the Blue Cross and Blue Shield plans in IL, MT, NM, OK, and TX) offer qualified dental and require the ACA pediatric dental essential health benefit on every enrollment. When the member wants to buy a stand-alone dental plan alongside the medical plan, you quote for dental separately and pass the selected dental plan into the enrollment as `dental_plan_hios_id`.

**Discover dental plans** with a ZIP/FIPS-scoped dental quote: a `POST /quotes` call that sets `dental_search: true` and includes the member's `zip_code`, `fip_code`, and `state`. Because it is scoped to the member's location, it returns only plans in their service area, which is what create enforces, so it is the reliable way to pick a plan that will pass:

```
POST /api/v1/quotes
{ "off_ex": true, "plan_year": 2026, "state": "IL", "zip_code": "60601",
  "fip_code": "17031", "dental_search": true,
  "applicants": [{ "age": 40, "smoker": false, "relationship": "primary" }] }
```

Results come back with `dental_only: true`. A `dental_search` quote can also return dental plans from **other carriers**, and `issuer_hios_ids` does not reliably restrict dental results, so pick a plan whose **issuer matches the medical plan's issuer** (HCSC's stand-alone dental is "BlueCare Dental", same issuer prefix as the medical plan, e.g. `36096IL0830001` for BCBS IL / `33602...` in TX). Passing a different-carrier plan, or one outside the member's service area, returns `422 No plan found for dental_plan_hios_id: <id>` at create.

The `GET /api/v1/plans?state=..&plan_year=..&dental_only=true&off_ex=true` list is useful for browsing or caching a dental catalog, but it is **state-level and not filtered by the member's service area**, so a plan taken directly from it can still fail create with a `422`. Validate service area with the dental quote before enrolling.

**End-to-end flow for a qualified-dental carrier:**

1. `POST /quotes` for the **medical** plan (as usual) — member selects a medical plan with `api_enrollment: true`.
2. Satisfy pediatric dental exactly one way:
   - **Attestation path** — no second quote needed. Send `attestations.pediatric_dental` (`purchased_separately` or `not_applicable`) on the application.
   - **Dental plan path** — discover dental plans via a ZIP/FIPS-scoped `dental_search` quote (service-area accurate), member selects a dental plan **whose issuer matches the medical plan's issuer**, send its `hios_id` as `dental_plan_hios_id` on the application (and omit `attestations.pediatric_dental`).
3. `POST /api/v1/applications` with exactly one of the two. Both, or neither, returns a `422` (`Invalid dental selection`).

The dental plan must be the **same carrier** as the medical plan, off-exchange eligible for the plan year, and in the member's service area, or the create returns a `422`. See "Qualified Dental (HCSC)" in SKILL.md and the HCSC entry in [carrier-examples.md](carrier-examples.md) for full payloads.

## GET /plans (list)

Pull plan and benefit metadata in bulk for a state and plan year, without quoting. Use this to cache plan data locally (benefits, cost-sharing, networks, metal level) so quoting and plan-comparison front-ends can render instantly from your own store instead of round-tripping for every interaction.

```
GET /api/v1/plans?state=AZ&plan_year=2026
```

Required: `state`, `plan_year`. Optional: `off_ex` (off-exchange plans), `dental_only`, `filter` (comma-separated field allowlist; `hios_id`, `name`, and `year` are always included), `page`, `per_page` (default 20, max 500 — values above the cap are silently capped).

Response is a lighter per-plan payload plus `meta.result_count` (total across all pages):

```json
{
  "plans": [
    {
      "hios_id": "53901AZ1490005",
      "name": "Blue Portfolio HSA Gold Statewide PPO",
      "year": 2026,
      "state": "AZ",
      "metal_level": "Gold",
      "plan_type": "PPO",
      "ichra_only": false,
      "dental_only": false,
      "hsa_eligible": true,
      "issuer_name": "Blue Cross Blue Shield",
      "issuer": { },
      "benefits": { },
      "cost_sharing": { },
      "cost_sharing_tiers": [ ],
      "urls": { }
    }
  ],
  "meta": { "result_count": 214 }
}
```

**What this endpoint does NOT return** (by design): premiums, `api_enrollment`/`deeplink_enrollment` flags, and `enrollment_requirements`. These are member/household- or selection-specific — fetch them in real time:
- Premiums and enrollment flags → `POST /quotes` (priced, per household).
- Enrollment flags + attestation content for a single selected plan → `GET /plans/:hios_id`.

Recommended pattern: cache the `/plans` payload per `state`/`plan_year` (benefit data is stable within a plan year), then layer live `POST /quotes` pricing on top for the actual quote.

## GET /plans/:hios_id

Look up a specific plan's details and enrollment flags without running a full quote.

```
GET /api/v1/plans/53901AZ1490005?plan_year=2026
```

Returns `api_enrollment` and `deeplink_enrollment` flags plus full plan metadata. Use this to confirm enrollment path when you already have a plan selected.

Plan-detail response shape:

- `estimated_rate` is a boolean indicating whether an ancillary plan uses estimated rates; it is not a premium.
- `benefits` is an object mapping benefit keys to formatted cost-sharing text.
- `benefits_with_tier_2` is an object mapping benefit keys to booleans.
- `benefits_pretty` is an object mapping benefit keys to `{original, before, after}` display values.
- `benefits_explanations` is an object mapping benefit keys to explanatory text.
- `cost_sharing` is an object whose values can be formatted strings, integer visit counts, or null.

### Enrollment Requirements

Add `?include=enrollment_requirements` to get carrier-specific enrollment metadata:

```
GET /api/v1/plans/35107NV0010017?plan_year=2026&include=enrollment_requirements
```

The response nests requirements under `plan.enrollment_requirements`. The example below shows representative groups; keys that do not apply to the selected plan are omitted.

```json
{
  "plan": {
    "enrollment_requirements": {
      "attestations": {
        "agrees_issuer_attestations": {
          "content": "I acknowledge that I have read..."
        },
        "electronic_signature_consent": {
          "content": "%{signature_name}, type your full name to sign electronically."
        },
        "pediatric_dental": {
          "required": false,
          "type": "attestation",
          "attestation_text": "Pediatric dental disclosure text...",
          "options": {
            "purchased_separately": "Consumer-facing attestation label...",
            "not_applicable": "There are no children under 19..."
          }
        }
      },
      "special_enrollment_period": {
        "event_types": {
          "offered_ichra": {
            "event_date_days_before": 60,
            "event_date_days_after": 60,
            "documentation_required": false
          }
        }
      },
      "communication_preferences": {
        "agrees_hsa_contact_opt_in": {
          "required": true,
          "content": "Would you like to enroll in a HealthEquity HSA?",
          "options": [
            {"value": true, "label": "Yes"},
            {"value": false, "label": "No"}
          ]
        }
      },
      "applicants": {
        "primary": {
          "marital_status": {
            "required": false,
            "options": ["single", "married"]
          }
        }
      },
      "proof_of_residency_required": false
    }
  }
}
```

The HSA content and option labels above illustrate the response shape only. Display the values returned for the selected plan, not the example text.

**Key rules:**
- Keys that are absent or null are not required for that carrier/state.
- `electronic_signature_consent.content` may contain `%{signature_name}` — replace with the applicant's full legal name before displaying. All other placeholders are resolved server-side.
- `pediatric_dental.options` is a `{value: label}` map. Display the label and submit the selected key as `attestations.pediatric_dental`.
- HCSC qualified-dental applications require exactly one of `attestations.pediatric_dental` or `dental_plan_hios_id`, even when the returned pediatric-dental `required` flag is false.
- `special_enrollment_period.event_types` is an object keyed by the request enum. Use only returned keys, enforce the returned date window, and follow `documentation_required`.
- `event_date_days_before` is how far the event date may be in the past relative to today; `event_date_days_after` is how far it may be in the future.
- Process returned applicant, communication-preference, HRA, and proof-of-residency requirements in addition to attestations.
- When `communication_preferences.agrees_hsa_contact_opt_in` is an object, render its exact carrier-defined `content` and option labels. Require an explicit boolean when `required` is true; treat it as optional when `required` is false. False is a valid answer and does not block enrollment in the health plan.
- If `communication_preferences` or `agrees_hsa_contact_opt_in` is absent or null, the HSA question does not apply to the selected plan. Do not render it or send the payload field, and do not infer applicability from the carrier name.
- For plan year 2026, BCBS Michigan medical metadata requires race/ethnicity for primary and dependents, limits dependent relationships to `spouse` and `child`, and marks every returned SEP reason as documentation-required.
- `state_supplement_*` content may be an array of paragraphs.
- Call once when the user selects a plan. Cache the result.

## FIPS Code Resolution

Many zip codes span multiple counties. The quoting API requires a `fip_code` (FIPS county code). Resolve zip codes using the **public CMS Marketplace API** (no API key required):

```
GET https://marketplace-int.api.healthcare.gov/api/v1/counties/by/zip/{zipcode}
```

Example:
```
GET https://marketplace-int.api.healthcare.gov/api/v1/counties/by/zip/42003
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

If one county is returned, auto-select it. If multiple, prompt the user to choose. Pass the `fips` value as `fip_code` in the quoting request. The `state` value can also be used to populate the `state` field.

## Large Group Quoting

There is no bulk **quoting** (pricing) endpoint — premiums are household-specific, so run one `POST /quotes` per household. For bulk **plan/benefit metadata**, use `GET /api/v1/plans` (see above) and cache it locally.

For a 500-person group at Standard rate limits (3,000 quotes/min, burst 300): approximately 10 seconds with 50 concurrent requests.

- Cache `GET /api/v1/plans` results per `state`/`plan_year` for benefit data; cache `POST /quotes` results per household composition — premiums don't change within a plan year for the same inputs.
- Implement exponential backoff on 429 responses.
- Use `retry_after` header value from 429 responses.
