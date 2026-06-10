# Deeplink Enrollment Reference

Use deeplinks when a plan has `deeplink_enrollment: true` but `api_enrollment: false`. The deeplink redirects the user to HealthSherpa's hosted enrollment UI with fields prefilled.

## Enrollment Routing

After quoting, check the flags on each plan:

```
api_enrollment == true        → EnrollConnect API (POST /api/v1/applications)
deeplink_enrollment == true   → Deeplink (POST /public/ichra/off_ex)
both true                     → your choice (API gives full control; deeplink is simpler)
both false                    → not enrollable through HealthSherpa
```

NEVER attempt EnrollConnect for a plan with `api_enrollment: false`.

## Deeplink Endpoint

```
POST /public/ichra/off_ex
```

| | EnrollConnect | Deeplink |
|---|---|---|
| Endpoint | `POST /api/v1/applications` | `POST /public/ichra/off_ex` |
| Request body | Canonical schema (nested) | Flat schema (see mapping below) |
| Auth | `x-api-key` (required) | `x-api-key` (optional in prod); Basic Auth in staging |
| Response | `201` JSON with `application_id` | `302` redirect to HealthSherpa UI |
| Lifecycle control | Full (update, submit, cancel, terminate) | None — user completes in HealthSherpa UI |
| Status tracking | API polling + webhooks | Webhooks only |

### Required Fields

- `_agent_id` — required, HealthSherpa-assigned agent slug
- `plan_hios_id` — the HIOS plan ID
- `zip_code` and `fip_code` — note: `fip_code` (not `fips_code`) in the deeplink schema
- Either `email` or `phone_number`

### Flat Schema Mapping

The deeplink uses a flat schema that differs from the canonical EnrollConnect schema:

| EnrollConnect (Canonical) | Deeplink (Flat) |
|---|---|
| `residential_address.street_address_1` | `street_address` (top-level) |
| `residential_address.street_address_2` | `street_address_unit_number` (top-level) |
| `residential_address.city` | `city` (top-level) |
| `residential_address.state` | `state` (top-level) |
| `residential_address.zip_code` | `zip_code` (top-level) |
| `residential_address.fips_code` | `fip_code` (top-level, note: no 's') |
| `hra` (top-level object) | `applicants.primary.hra` (nested under primary) |
| `agent_of_record.national_producer_number` | `agent_of_record_npn` (flat, top-level) |
| `special_enrollment_period.event_type` | `sep_reason` (top-level) |
| `special_enrollment_period.event_date` | `sep_reason_date` (top-level) |
| `applicants.dependents[].relationship` | Separate `spouse`, `domestic_partner`, `dependents` keys |

### Response Handling

The deeplink returns a **302 redirect**, not JSON. Your backend must:

1. Capture the `Location` header — do NOT follow the redirect server-side
2. Redirect the user's browser to that URL
3. The user completes enrollment in HealthSherpa's UI

```typescript
const response = await fetch(deeplinkUrl, {
  method: "POST",
  headers: { "Content-Type": "application/json", "x-api-key": apiKey },
  body: JSON.stringify(payload),
  redirect: "manual",
});

if (response.status === 302) {
  const enrollmentUrl = response.headers.get("Location");
  // Send this URL to the user's browser
}
```

### Environments

| Environment | Base URL | Auth |
|---|---|---|
| Staging | `https://staging.healthsherpa.com` | Basic Auth (credentials provided during onboarding) |
| Production | `https://www.healthsherpa.com` | `x-api-key` (optional, recommended for agent allow-listing) |

### Post-Deeplink Tracking

After the user completes enrollment in the HealthSherpa UI:

1. **Submission Confirmation Webhook** — fires when submitted. Contains `application_id` and `external_id` (if set).
2. **Policy Status Webhook** — fires on status transitions (`pending_effectuation`, `effectuated`, `cancelled`, `terminated`).

You cannot poll `GET /api/v1/applications/:id` for deeplink-initiated applications unless the application was created with your API key. Webhooks are the primary tracking mechanism.
