---
type: Page
title: Billing & Payments Domain Architecture
description: The Billing and Payments domain coordinates policy financial lifecycles, premium collections, arrears handling, and claims disbursements across Tidewell Mutual.
resource: https://github.com/Nox-Demo-Org/kb-org-tw-billing/blob/main/architecture/billing-and-payments.md
tags:
- org-tw-billing
- architecture
generated:
  by: nox/gemini-3.7-flash
  at: '2026-10-05T15:53:34Z'
---

# Billing & Payments Domain Architecture

The Billing and Payments domain coordinates policy financial lifecycles, premium collections, arrears handling, and claims disbursements across Tidewell Mutual.

---

## Domain Services

### 1. `billing-service`
- **Primary Role:** Financial plan lifecycle management for issued insurance policies.
- **Core Responsibilities:**
  - Generates payment schedules on `policy.policy.issued` (yearly single pay or monthly 12 instalments).
  - Computes monthly instalment fees (6% of annual premium, capped at £38/year, front-loaded remainder).
  - Dispatches collection notices 5 days before due dates (`billing.instalment.due`).
  - Handles arrears and missed payment escalation (`billing.payment.missed`).
  - Calculates daily pro-rata refunds upon mid-term cancellations (`policy.policy.cancelled`).

### 2. `payments-gateway`
- **Primary Role:** Edge provider for external financial networks (PSP, Bacs Direct Debit, UK Faster Payments).
- **Core Responsibilities:**
  - Premium collections via card token charges (immediate settlement) or Direct Debit (3 working days).
  - Claims disbursements via Faster Payments.
  - Tokenization boundary ensuring no raw PAN/card data is handled by core systems.
  - Enforces secondary approvals on high-value disbursements (>£25,000).

---

## End-to-End Workflows

### 1. Policy Inception & Premium Collection
1. Policy is issued by `policy-admin`, emitting `policy.policy.issued`.
2. `billing-service` consumes the event, creates a `PaymentPlan`, and calculates instalment schedules.
3. 5 days prior to an instalment due date, `billing-service` emits `billing.instalment.due`.
4. `billing-service` invokes `POST /v1/collections` on `payments-gateway` with an `idempotency_id`.
5. `payments-gateway` charges the customer via the PSP and emits `payments.collection.succeeded`.
6. `billing-service` consumes `payments.collection.succeeded` and marks the instalment as paid.

### 2. Arrears & Failure Flow
1. If a collection attempt fails, `payments-gateway` retries after 3 working days.
2. If the second attempt fails, `billing-service` publishes `billing.payment.missed`.
3. `policy-admin` initiates statutory 14-day cancellation letter procedures.
4. `notifications-hub` sends SMS alerts to the policyholder.

### 3. Claim Settlement Disbursement
1. `claims-management` finalizes a claim settlement.
2. If amount > £25,000, secondary management approval is enforced.
3. `claims-management` calls `POST /v1/payouts` on `payments-gateway`.
4. `payments-gateway` executes the Faster Payment transfer and emits `payments.payout.sent`.
5. `notifications-hub` alerts the customer and `customer-portal` updates the claim tracking status.