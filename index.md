---
okf_version: '0.2'
title: Tidewell Mutual System Architecture
description: Welcome to the organizational architecture knowledge base for Tidewell Mutual.
generated:
  at: '2026-10-05T15:53:34Z'
---

# Tidewell Mutual System Architecture

Welcome to the organizational architecture knowledge base for Tidewell Mutual. This documentation maps the microservices, domain boundaries, shared infrastructure, and asynchronous integration flows across the enterprise.

---

## Domain Map & Core Services

```
+---------------------------------------------------------------------------------------+
|                                 Digital Channels                                      |
|  +------------------------+                                                           |
|  |    customer-portal     |                                                           |
|  +-----------+------------+                                                           |
+--------------|------------------------------------------------------------------------+
               | (REST /plans, claims)
+--------------v------------------------------------------------------------------------+
|                               Core Business Domains                                   |
|                                                                                       |
|  [ Policy Domain ]                  [ Claims Domain ]                                 |
|  +------------------------+         +-----------------------+                         |
|  |      policy-admin      |         |   claims-management   |                         |
|  +-----------+------------+         +-----------+-----------+                         |
|              |                                  |                                     |
|  ============|==================================|===================================  |
|              | (policy.policy.issued)           | (POST /v1/payouts)                  |
|              v                                  v                                     |
|  [ Billing & Payments Domain ]                                                        |
|  +------------------------+         +-----------------------+                         |
|  |    billing-service     |-------> |   payments-gateway    |                         |
|  +------------------------+ (POST)  +-----------+-----------+                         |
|              |                                  |                                     |
|              | (billing.instalment.due)         | (payments.payout.sent)              |
|  ============|==================================|===================================  |
|              v                                  v                                     |
|  [ Customer & Comms Platform ]                                                        |
|  +------------------------+         +-----------------------+                         |
|  |   customer-identity    |         |   notifications-hub   |                         |
|  +------------------------+         +-----------------------+                         |
+---------------------------------------------------------------------------------------+
```

### 1. Billing & Payments
- [[architecture/billing-and-payments|Billing & Payments Architecture]]: Details the lifecycle of payment plans, fee allocation, direct collections, and claims disbursements.
- `billing-service`: Manages payment plans, instalment schedules, instalment fees, collection workflows, and cancellation refunds.
- `payments-gateway`: Direct interface to Payment Service Providers (PSP) and the Faster Payments network for collections and settled claim payouts.

### 2. Policy & Claims Administration
- `policy-admin`: Policy issuance, cancellation, and statutory notification workflows.
- `claims-management`: First-party claim intake, adjuster review, high-value payout approvals (>£25k), and settlement orchestration.

### 3. Customer Platform & Channels
- `customer-identity`: Single customer view, KYC/profile data, and address management.
- `notifications-hub`: Multi-channel customer messaging (SMS, Email, Letters).
- `customer-portal`: Customer-facing web/mobile applications.

---

## Cross-Cutting Architecture

- [[architecture/integration-and-events|Integration & Event Backbone]]: Comprehensive catalog of REST contracts and asynchronous Pub/Sub / event topics.
- **Idempotency & Safety:** Strict idempotency standards on money movement (`idempotency_id`).
- **Security & Tokenization:** Strict PCI-DSS decoupling via tokenized payment instruments handled exclusively by `payments-gateway`.

<!-- okf:contents -->

## Contents

- [Architecture](/architecture/index.md) — 2 pages.
- [Change log](/log.md) — every generation and sync, newest first.

<!-- /okf:contents -->
