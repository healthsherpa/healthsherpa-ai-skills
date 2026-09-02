# Deeplink Enrollment Reference

Use Deeplink when `deeplink_enrollment` is true and the platform selects that route before any EnrollConnect create request. For plans with both flags, prefer EnrollConnect. Deeplink redirects the user to HealthSherpa's hosted enrollment UI with fields prefilled.

## Enrollment Routing

After quoting, check the flags on each plan:

```
api_enrollment == true        → EnrollConnect API (POST /api/v1/applications)
deeplink_enrollment == true   → Deeplink (POST /public/ichra/off_ex)
both true                     → prefer EnrollConnect; choose Deeplink only before create
both false                    → not enrollable through HealthSherpa
```

NEVER attempt EnrollConnect for a plan with `api_enrollment: false`.
Never use Deeplink as a retry after an ambiguous EnrollConnect create response.
If transport-level evidence proves the create request was never sent, a new
attempt is safe. Otherwise use bounded reconciliation and manual intervention;
do not retry or switch routes automatically.

## Deeplink Endpoint

```
POST /public/ichra/off_ex
```

| | EnrollConnect | Deeplink |
|---|---|---|
| Endpoint | `POST /api/v1/applications` | `POST /public/ichra/off_ex` |
| Request body | Canonical schema (nested) | Flat schema (see mapping below) |
| Auth | `x-api-key` (required) | `x-api-key` (required); Basic Auth also required in staging |
| Response | `201` JSON with `application_id` | `302` redirect to HealthSherpa UI |
| Lifecycle control | Full (update, submit, cancel, terminate) | None — user completes in HealthSherpa UI |
| Status tracking | API polling + webhooks | Webhooks, then EnrollConnect reads by `application_id` |

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
2. Parse the URL and require HTTPS
3. Require the environment's exact HealthSherpa origin: `https://staging.healthsherpa.com` or `https://www.healthsherpa.com`
4. Reject a missing location, any other status, or any other destination
5. Redirect the user's browser to the validated URL

```typescript
const expectedDeeplinkOrigin =
  environment === "production"
    ? "https://www.healthsherpa.com"
    : environment === "staging"
      ? "https://staging.healthsherpa.com"
      : null;
if (!expectedDeeplinkOrigin) throw new Error("Unknown HealthSherpa environment");

const deeplinkUrl = new URL(
  "/public/ichra/off_ex",
  expectedDeeplinkOrigin
);
if (
  deeplinkUrl.protocol !== "https:" ||
  deeplinkUrl.origin !== expectedDeeplinkOrigin ||
  deeplinkUrl.username ||
  deeplinkUrl.password ||
  deeplinkUrl.pathname !== "/public/ichra/off_ex"
) {
  throw new Error("Invalid HealthSherpa Deeplink request destination");
}

if (!apiKey) throw new Error("HealthSherpa API key is unavailable");

const headers: Record<string, string> = {
  "Content-Type": "application/json",
  "x-api-key": apiKey,
};

if (environment === "staging") {
  if (!stagingUsername || !stagingPassword) {
    throw new Error("Staging Deeplink credentials are unavailable");
  }
  headers.Authorization =
    `Basic ${Buffer.from(`${stagingUsername}:${stagingPassword}`).toString("base64")}`;
}

const response = await fetch(deeplinkUrl, {
  method: "POST",
  headers,
  body: JSON.stringify(payload),
  redirect: "manual",
});

if (response.status !== 302) {
  throw new Error(`Unexpected deeplink response: ${response.status}`);
}

const location = response.headers.get("Location");
if (!location) throw new Error("Deeplink response is missing Location");

const enrollmentUrl = new URL(location);
if (
  enrollmentUrl.protocol !== "https:" ||
  enrollmentUrl.origin !== expectedDeeplinkOrigin ||
  enrollmentUrl.username ||
  enrollmentUrl.password
) {
  throw new Error("Invalid HealthSherpa deeplink destination");
}

// Return enrollmentUrl.toString() to the browser.
```

### Environments

| Environment | Base URL | Auth |
|---|---|---|
| Staging | `https://staging.healthsherpa.com` | `x-api-key` plus Basic Auth |
| Production | `https://www.healthsherpa.com` | `x-api-key` |

### Post-Deeplink Tracking

After the user completes enrollment in the HealthSherpa UI:

1. **Submission Confirmation Webhook** — fires when submitted. Contains `application_id` and `external_id` (if set).
2. **Policy Status Webhook** — fires on status transitions (`pending_effectuation`, `effectuated`, `cancelled`, `terminated`).

Always include the same platform API key on Deeplink creation. After a webhook supplies the resulting `application_id`, the platform can access that application through EnrollConnect reads. Without the API key, the application is not associated with the platform and cannot be retrieved through its EnrollConnect credentials.
