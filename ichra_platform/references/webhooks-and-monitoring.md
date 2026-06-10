# Webhooks & Monitoring Reference

## Webhook Setup

Webhook endpoints are configured during onboarding. Contact your HealthSherpa account manager to register or update your webhook URL. There is no self-service webhook registration API.

Two webhook types are available:
- **Submission Confirmation** — fires when an application is submitted to the carrier
- **Policy Status** — fires on status transitions (`pending_effectuation`, `effectuated`, `cancelled`, `terminated`)

Both webhook types fire for EnrollConnect and Deeplink submissions.

## Signature Verification

Every webhook request includes an `X-Webhook-Signature` header. Always verify this header before processing the payload to ensure the request is authentic. The signing secret is provided during onboarding alongside your webhook URL. Contact your HealthSherpa account manager for verification details specific to your integration.

## Webhook Rules

- Respond 2xx within 10 seconds. Process the event asynchronously.
- Verify `X-Webhook-Signature` header on every request.
- Deduplicate by `application_id` + `policy_status` + `timestamp`.
- Delivery is at-least-once. Up to 3 retries: 20 seconds, 10 minutes, 1 hour.
- Failed events can be replayed on request — contact your account manager for time-range replay.

## Payload

```json
{
  "event_type": "policy_status_changed",
  "application_id": "HSA000000001",
  "external_id": "your-tracking-id",
  "policy_status": "effectuated",
  "policy": {
    "policy_id": "HSP000000001",
    "effective_date": "2026-07-01",
    "status": "effectuated",
    "plan_hios_id": "53901AZ1490005",
    "gross_premium": 750.25,
    "members": [
      {
        "member_id": "HSM000000001",
        "effective_date": "2026-07-01",
        "removed_date": null
      }
    ]
  },
  "timestamp": "2026-07-02T08:15:00Z"
}
```

## Event Actions

| `policy_status` | Action |
|---|---|
| `submission_failed` | Carrier submission failed. Check `errors`. Alert for manual review or retry. |
| `sep_docs_required` | SEP documentation needed. Prompt user to upload via `/supporting_documentation`. |
| `sep_docs_under_review` | Docs uploaded, carrier reviewing. No action needed — wait for next transition. |
| `pending_effectuation` | Record submission. Coverage pending carrier confirmation. |
| `effectuated` | Coverage active. Start HRA reimbursements. |
| `cancelled` | Stop future reimbursements. Prospective cancellation. |
| `terminated` | Reconcile retroactive changes. May require reimbursement clawback. |

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

## Reconciliation

| Cadence | Action |
|---|---|
| Daily | `GET /applications?updated_since=<yesterday>` — catch any missed webhooks |
| Weekly | Investigate `pending_effectuation` applications older than 7 days |
| Monthly | Compare local policy records against `GET /applications?policy_status=effectuated` |

## Alerting

Set up alerts for:

- `pending_effectuation` status persisting > 48 hours
- Webhook endpoint returning non-2xx for > 15 minutes
- Spike in 422 errors on create/submit (may indicate payload issues or carrier-side changes)
- Increasing 429 rate limit responses (may need rate limit upgrade)
