# Webhooks & Monitoring Reference

## Webhook Setup

Webhook endpoints are configured during onboarding. Contact your HealthSherpa account manager to register or update your webhook URL. There is no self-service webhook registration API.

Two webhook types are available:
- **Submission Confirmation** — fires when an application is submitted to the carrier
- **Policy Status** — fires on status transitions (`pending_effectuation`, `effectuated`, `cancelled`, `terminated`)

Both webhook types fire for EnrollConnect and Deeplink submissions.

## Authentication

Webhook authentication is configured per integration during onboarding — HealthSherpa supports a variety of methods, and you specify the approach and credentials when you register your endpoint. Requests carry headers consistent with the method you chose, plus a `Content-Type: application/json` header. Verify every request using that configured method before processing the payload. Contact your HealthSherpa account manager for the details specific to your integration.

## Webhook Rules

- Respond with `HTTP 200 OK` promptly and process the event asynchronously.
- Authenticate every request using the method you configured during onboarding.
- Make your handler idempotent — deduplicate by `transaction_id` (the unique event identifier); the same event may arrive more than once.
- Contact your account manager if you need help re-delivering missed events.

## Payload

Both webhook types share one schema. `event_type` is `submission` (Submission Confirmation) or `sync` (Policy Status). Dates in the payload use `MM/DD/YYYY`. `policies` is an array. Off-exchange example:

```json
{
  "transaction_id": 123456789,
  "application_id": "HSA000000000",
  "policy_status": "effectuated",
  "event_type": "sync",
  "event_timestamp": "07/01/2025",
  "external_id": "your-tracking-id",
  "policy_aor_npn": "1234567890",
  "submitter_npn": "17169718",
  "npn_used_at_submission": "17169718",
  "issuer_hios_id": "12345",
  "members": [
    { "member_id": "HSM000000000", "first_name": "John", "last_name": "Doe", "date_of_birth": "05/10/1980" }
  ],
  "policies": [
    {
      "policy_id": "HSP000000000",
      "effective_date": "08/01/2025",
      "expiration_date": "12/31/2025",
      "status": "effectuated",
      "plan_hios_id": "12345LA0123456",
      "gross_premium": 750.25,
      "members": [
        { "member_id": "HSM000000000", "effective_date": "08/01/2025", "removed_date": null }
      ],
      "payment": {
        "payment_status": "paid",
        "payment_status_updated_date": "06/15/2025",
        "paid_through_date": "07/31/2025",
        "current_member_responsibility_balance_due": "50.50",
        "autopay_indicator": true
      }
    }
  ]
}
```

## Event Actions

The Policy Status webhook (`event_type: "sync"`) delivers the lifecycle `policy_status` values:

| `policy_status` | Action |
|---|---|
| `pending_effectuation` | Record submission. Coverage pending carrier confirmation (may await binder payment or account setup, even for $0 plans). |
| `effectuated` | Coverage active and binder payment received. Start HRA reimbursements. |
| `cancelled` | Policy never effectuated (typically non-payment). Stop future reimbursements. |
| `terminated` | Policy was active and later ended. Reconcile retroactive changes; may require reimbursement clawback. |

The following `policy_status` values are not delivered by the Policy Status webhook — you observe them on the application record via `GET /api/v1/applications/:id` during the create / submit / document flow:

Use `sep_docs_*` rows only when those values are returned as `policy_status`
for a currently SEP-suspended application. For other document workflows,
monitor `document_status` instead.

| `policy_status` | Action |
|---|---|
| `submission_failed` | Carrier submission failed. Check `errors`. Alert for manual review or retry. |
| `sep_docs_required` | SEP documentation needed. Prompt user to upload via `/supporting_documentation`. |
| `sep_docs_under_review` | Docs uploaded and under review. Continue bounded polling; alert when review exceeds the integration's documented service threshold. |
| `sep_docs_denied` | Submitted proof was denied. Stop passive polling. Use returned actions or errors when present; otherwise prompt for corrected documentation or escalate manually. |

## Polling Fallback

Use polling as a supplement to webhooks, not a replacement.

```
GET /api/v1/applications?updated_since=2026-07-01T00:00:00Z
```

### Recommended Polling Intervals

| Phase | Interval |
|---|---|
| First hour after submit | Every 5 minutes |
| Awaiting effectuation | Every 30 minutes |
| Steady state | Every 4 hours (or rely on webhooks) |

Carrier processing is asynchronous. Applications may stay in `pending_effectuation` for minutes to hours depending on the carrier. Do not treat slow transitions as errors.

Define a separate stale-review threshold for `sep_docs_under_review` based on
the supported carrier process. Polling must produce an alert when that boundary
is exceeded rather than continue indefinitely.

## Reconciliation

| Cadence | Action |
|---|---|
| Daily | `GET /applications?updated_since=<yesterday>` — catch any missed webhooks |
| Weekly | Investigate `pending_effectuation` applications older than 7 days |
| Monthly | Compare local policy records against `GET /applications?policy_status=effectuated` |

## Alerting

Set up alerts for:

- `pending_effectuation` status persisting > 48 hours
- `sep_docs_under_review` persisting beyond the configured review threshold
- Webhook endpoint returning non-2xx for > 15 minutes
- Spike in 422 errors on create/submit (may indicate payload issues or carrier-side changes)
- Increasing 429 rate limit responses (may need rate limit upgrade)
