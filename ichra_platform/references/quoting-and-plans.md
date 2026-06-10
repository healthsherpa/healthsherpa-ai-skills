# Quoting & Plan Selection Reference

## POST /quotes

ALWAYS use POST (not GET). ALWAYS include the `hra` object for ICHRA quoting.

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
  ],
  "hra": {
    "offered_hra": true,
    "type": "ichra",
    "amount": 500,
    "contribution_covers": "premium",
    "start": "2026-01-01"
  }
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
| `issuer_hios_ids` | array of strings | Filter results to specific carriers by issuer HIOS ID prefix (e.g., `["12515"]` for HCSC). |
| `filter` | string | Comma-separated list of field names to include in the response (e.g., `"name,premium,hios_id,metal_level"`). Reduces payload size. |

### Response Fields (Per Plan)

| Field | Type | Description |
|---|---|---|
| `hios_id` | string | 14-char plan ID. Store this — it's the key to everything. Also returned as `plan_hios_id` in some contexts. |
| `name` | string | Display name |
| `issuer` | object | `{name, hios_id, state, logo_url, payment_phone, customer_service_phone}` — carrier info. Use `logo_url` for carrier branding. See issuer Object section below for details. |
| `premium` | number | Monthly net premium (after ICHRA if applicable) |
| `gross_premium` | number | Monthly premium before any HRA/subsidy |
| `metal_level` | enum | `Bronze`, `Silver`, `Gold`, `Platinum`, `Catastrophic` |
| `plan_type` | enum | `HMO`, `PPO`, `EPO`, `POS` |
| `api_enrollment` | boolean | Plan supports EnrollConnect API enrollment |
| `deeplink_enrollment` | boolean | Plan supports deeplink enrollment |
| `cost_sharing` | object | See cost_sharing fields below |
| `benefits` | object | Benefit coverage details |
| `adult_dental` | boolean | Plan covers adult dental (relevant for dental plan grouping) |
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
both true                     → your choice (API gives full control; deeplink is simpler)
both false                    → not enrollable through HealthSherpa
```

NEVER attempt EnrollConnect for a plan with `api_enrollment: false`. The API will reject the request with a 422. A plan may have `deeplink_enrollment: true` but `api_enrollment: false` — always check both flags per plan.

## GET /plans/:hios_id

Look up a specific plan's details and enrollment flags without running a full quote.

```
GET /api/v1/plans/53901AZ1490005?plan_year=2026
```

Returns `api_enrollment` and `deeplink_enrollment` flags plus full plan metadata. Use this to confirm enrollment path when you already have a plan selected.

### Attestation Content

Add `?include=enrollment_requirements` to get carrier-specific requirements and attestation text:

```
GET /api/v1/plans/53901AZ1490005?plan_year=2026&include=enrollment_requirements
```

The response includes `enrollment_requirements.attestations`:

```json
{
  "enrollment_requirements": {
    "attestations": {
      "agrees_issuer_attestations": {
        "content": "I acknowledge that I have read..."
      },
      "electronic_signature_consent": {
        "content": "%{signature_name}, type your full name to sign electronically."
      },
      "broker_signature_attestation": {
        "content": "I attest that I have obtained written consent..."
      },
      "pediatric_dental": {
        "content": "Individuals must purchase pediatric dental benefits...",
        "options": ["purchased_separately", "not_applicable"]
      },
      "state_supplement_primary_signature": {
        "content": ["I acknowledge that I have read all sections...", "I understand that my answers are the basis..."]
      }
    }
  }
}
```

**Key rules:**
- Keys that are absent or null are not required for that carrier/state.
- `electronic_signature_consent.content` may contain `%{signature_name}` — replace with the applicant's full legal name before displaying. All other placeholders are resolved server-side.
- `pediatric_dental.options` lists the valid values for the `pediatric_dental` field on the application.
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

No bulk endpoint. One `POST /quotes` per household.

For a 500-person group at Standard rate limits (3,000 quotes/min, burst 300): approximately 10 seconds with 50 concurrent requests.

- Cache results per household composition — premiums don't change within a plan year for the same inputs.
- Implement exponential backoff on 429 responses.
- Use `retry_after` header value from 429 responses.
