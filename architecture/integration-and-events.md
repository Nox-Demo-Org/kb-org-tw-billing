---
type: Page
title: Integration, Contracts & Event Backbone
description: This document details the communication patterns, message topologies, and synchronous REST contracts connecting Tidewell Mutual services.
resource: https://github.com/Nox-Demo-Org/kb-org-tw-billing/blob/main/architecture/integration-and-events.md
tags:
- org-tw-billing
- architecture
generated:
  by: nox/gemini-3.7-flash
  at: '2026-10-05T15:53:34Z'
---

# Integration, Contracts & Event Backbone

This document details the communication patterns, message topologies, and synchronous REST contracts connecting Tidewell Mutual services.

---

## Synchronous REST Interfaces

| Producer Service | Endpoint | Consumer Service | Purpose |
| :--- | :--- | :--- | :--- |
| `billing-service` | `GET /v1/plans/{policyId}` | `customer-portal` | View plan details, schedules, and instalment fees. |
| `billing-service` | `POST /v1/plans` | Internal / Admin | Synchronously create a payment plan. |
| `payments-gateway` | `POST /v1/collections` | `billing-service` | Execute card token or direct debit premium collection. |
| `payments-gateway` | `POST /v1/payouts` | `claims-management` | Dispatch claim settlement via Faster Payments. |
| `customer-identity`| `GET /v1/customers/{id}` | `billing-service` | Retrieve customer snapshot details at plan creation. |

---

## Asynchronous Event Catalog

```
+--------------------------------------------------------------------------------------+
|                              Event Backbone Matrix                                   |
|                                                                                      |
|  Event Name                     Publisher           Subscribers                      |
|  -----------------------------  ------------------  -------------------------------  |
|  policy.policy.issued           policy-admin        billing-service                  |
|  policy.policy.cancelled        policy-admin        billing-service                  |
|  billing.instalment.due         billing-service     payments-gateway,                |
|                                                     notifications-hub                |
|  billing.payment.missed         billing-service     policy-admin,                    |
|                                                     notifications-hub                |
|  payments.collection.succeeded  payments-gateway    billing-service                  |
|  payments.payout.sent           payments-gateway    claims-management,               |
|                                                     notifications-hub,               |
|                                                     customer-portal                  |
+--------------------------------------------------------------------------------------+
```

### Event Specifications

#### `billing.instalment.due`
- **Publisher:** `billing-service`
- **Payload:**
```json
{
  "plan_id": "string",
  "policy_id": "string",
  "customer_id": "string",
  "instalment": 1,
  "amount_pence": 5250,
  "fee_pence": 250,
  "due_date": "2025-04-01"
}
```

#### `billing.payment.missed`
- **Publisher:** `billing-service`
- **Payload:**
```json
{
  "plan_id": "string",
  "policy_id": "string",
  "instalment": 1,
  "attempts": 2
}
```

#### `payments.collection.succeeded`
- **Publisher:** `payments-gateway`
- **Payload:**
```json
{
  "plan_id": "string",
  "instalment": 1,
  "amount_pence": 5250,
  "collected_at": "2025-04-01T08:30:00Z"
}
```

#### `payments.payout.sent`
- **Publisher:** `payments-gateway`
- **Payload:**
```json
{
  "claim_id": "string",
  "amount_pence": 125000,
  "sent_at": "2025-04-01T09:15:00Z"
}
```

---

## Infrastructure & Protocol Notes

- **Message Broker:** Services interact via event streams (Google Cloud Pub/Sub / Kafka message bus). Shared topic naming conventions follow the domain standard `<domain>.<entity>.<action>`.
- **Idempotency Standard:** All mutation REST endpoints (`POST /v1/collections`, `POST /v1/payouts`) require an `idempotency_id` header or body parameter. Reprocessing requests with identical keys guarantees exactly-once financial execution.
- **Data Snapshotting:** `billing-service` deliberately does not subscribe to `customer.profile.updated`; it captures contact and address data at plan inception to preserve historical fidelity for arrears notices.