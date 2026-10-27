# Subscription & Billing Service: Test Strategy and Approach

> Status: **approach only (pre-implementation)**. Full solution to follow in this PR.
> Location: `submission/karishma-k/` | Branch: `solution/karishma-k` | Stack: TypeScript, Vitest (or Jest), Supertest, better-sqlite3

## 1. My understanding of the task

The goal is not to test CRUD endpoints. It is to prove a **stateful, billed workflow** stays correct when events are duplicated, reordered, forged, or partially failed. The system has two inputs, API calls and signed webhooks from a payment provider, and three places where truth must agree: the **API response**, the **database**, and the **provider mock's recorded calls**. Most scenarios end with a three-way agreement check across those.

## 2. System under test (Option 3: Hybrid)

No service is provided, so I will build a **small Express + SQLite fixture** (deliberately thin) and spend the effort on the test framework.

| Component | Real or test double | Why |
|---|---|---|
| HTTP API, state machine, webhook handler | **Real** (my fixture) | This is the behaviour under test |
| SQLite (in-memory, fresh per test) | **Real** | Persistence correctness is a graded layer; no DB mocking |
| `PaymentProvider` client | **Fake** (`FakePaymentProvider`) | External dependency; configurable success / decline / timeout; records calls |
| Inbound webhooks | **Simulated** by the suite | Signed with HMAC-SHA256, plus deliberately forged/malformed variants |
| Clock | **Injectable** | Lets tests trigger trial-end and retry transitions deterministically |

### Assumptions I am making (to be confirmed or documented)
- **Plans:** `basic` = 900 USD-cents, no trial, charged immediately. `pro` = 4900, 14-day trial.
- **Time-driven transitions** (trial end, recurring charge, retries exhausted) are triggered via a test-only `BillingEngine.runDue(now)` with an injected clock, not real timers. Max 3 retries.
- **Declined charge at creation** (immediate-charge plan): `402`, no subscription persisted. The lifecycle diagram has no `-> past_due` edge from creation.
- **Provider timeout:** outcome is *unknown*, so the subscription never advances to `active` on a timeout. The invoice stays `pending` until a webhook or retry resolves it. The provider call carries an idempotency key (`invoice_id:attempt`).
- **`payment.refunded`:** recorded on the invoice and audit log with no status change (not in the lifecycle diagram).
- **Out-of-order events:** processed event IDs are always recorded, but events that do not apply to the current state are marked `ignored`.

## 3. Test levels and scope

1. **API contract:** status codes, payload shape, validation errors (bad plan, missing fields, unknown customer), webhook signature and malformed-payload handling.
2. **State machine:** every valid transition from the diagram exercised at least once; invalid transitions proven impossible, driven by both API and webhooks.
3. **Database:** `subscriptions`, `invoices`, `webhook_events`, `audit_log` are checked through a repository, never inline SQL in tests.
4. **Provider interaction:** call count, arguments (amount, customer, payment method, idempotency key), behaviour under success / decline / timeout.
5. **End-to-end flows:** API/webhook -> service -> DB -> provider, with a three-way consistency check.

**Out of scope:** UI, real provider, performance, proration/tax/multi-currency. **Bonus if time allows:** parallel webhook race for the same subscription.

## 4. Design patterns (each tied to a concrete problem)

| Pattern | Where | Problem it solves |
|---|---|---|
| **State machine (explicit transition table)** | `SubscriptionStateMachine` in the fixture; the same table drives the test matrix | Status is only changed via `transition(from, event)`; illegal moves throw. Tests are generated from the table so valid and invalid coverage is exhaustive. |
| **Builder** | `SubscriptionBuilder`, `WebhookPayloadBuilder`, `CustomerBuilder` | Scenarios read as intent (`aSubscription().onPlan('pro').pastDue()`), with no repeated object literals. |
| **Adapter / DI seam** | `PaymentProvider` interface, with `FakePaymentProvider` injected in tests | Swaps the network call for a configurable, recording test double. |
| **Repository** | `SubscriptionRepo`, `InvoiceRepo`, `WebhookEventRepo` | Keeps SQL out of tests and gives a typed verification layer. |
| **Strategy** (plan pricing/trial) | `PlanPolicy` per tier | Avoids plan branching scattered across the service. |

I will only list a pattern in the final PR description if it is actually present in the code.

## 5. Planned structure

```
submission/karishma-k/
  src/                        # minimal service fixture
    app.ts  stateMachine.ts  billingEngine.ts  webhookHandler.ts
    providers/PaymentProvider.ts  repos/*  db/schema.sql
  tests/
    framework/
      ApiClient.ts            # typed HTTP client
      FakePaymentProvider.ts  # outcomes + call recording
      WebhookSimulator.ts     # sign / forge / malform
      builders/*              # Builders
      assertions/*            # DbAssertions, ProviderAssertions, ConsistencyAssertions
      TestEnvironment.ts      # fresh DB + injected fakes per test
    specs/
      api.contract.spec.ts
      lifecycle.spec.ts       # table-driven transitions
      webhooks.idempotency.spec.ts
      provider.interaction.spec.ts
      persistence.invariants.spec.ts
      e2e.flows.spec.ts
  docs/ TEST_STRATEGY.md
```

## 6. Webhook and idempotency strategy

- Signature: HMAC-SHA256 over the **raw body**, constant-time compare; invalid -> `401`, nothing persisted, no side effects.
- `webhook_events.event_id` is `UNIQUE`. Event insert, state transition, and invoice write happen in **one transaction**, so a duplicate cannot partially apply.
- Duplicate delivery: `200` no-op, exactly one invoice/transition, **zero** additional provider calls, and the duplicate is still visible in the event log.
- Out of order (`failed` after `succeeded` for the same invoice): must not regress `active`.
- Cancelled subscription: absorbs all later webhooks without any transition.

## 7. Invariants asserted (after every scenario via `assertConsistent(subId)`)

1. `active` implies at least one persisted successful payment.
2. `canceled` never transitions again.
3. Duplicate events never create duplicate invoices, payments, or provider charges.
4. Provider is called at most once per real billing attempt and not at all for rejected or replayed requests.
5. Persisted status equals the API-observed status.
6. Plan price and trial rules are identical at creation and at billing.
7. No contradictory records (for example `active` with only a failed invoice).

## 8. Isolation and false-positive control

A fresh in-memory SQLite database and a fresh `FakePaymentProvider` are created per test. Event IDs and invoice IDs are generated per test. Assertions always query by IDs created in that test, never "latest row" or counts across the table.

## 9. Delivery plan (Red -> Blue -> Green)

1. This document and a placeholder PR.
2. Red: transition-table and idempotency specs, written failing first.
3. Blue: minimum service fixture and framework to pass them.
4. Green: refactor, then add API contract, provider interaction, and e2e specs.
5. Bonus: concurrent webhook test. Final README and PR description in the required 7-section format.

## 10. Known limitations (initial)

SQLite is single-writer, so the concurrency bonus tests application-level guarantees (unique constraint plus transaction), not a production multi-node database. The provider is a fake, so real network and retry semantics are modelled but not exercised.

## 11. Responsible AI usage

To be filled in candidly in the final PR: where AI helped, and what I personally reviewed, ran, and corrected.
