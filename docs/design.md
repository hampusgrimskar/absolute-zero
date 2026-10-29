# AbsoluteZero — Fintech Portfolio Project (Draft)

**Name:** AbsoluteZero
**Repo:** `absolute-zero-ledger`
**Tagline:** A double-entry wallet service where every transaction sums to zero.

## 1. Goal

Build a small **wallet / payments service** that demonstrates fintech engineering discipline:
money is never lost, duplicated, or miscalculated, even when things fail.

Focus is on the **backend**. No mobile app. UI is optional (Swagger UI is enough).

> This is a simulation. No real money, no real bank.

---

## 2. Tech Stack

| Area | Choice |
|---|---|
| Language | Java 21 |
| Framework | Spring Boot 3 |
| Database | PostgreSQL |
| Migrations | Flyway |
| Messaging | Kafka |
| Testing | JUnit 5 + Testcontainers |
| Local run | Docker Compose |
| API docs | OpenAPI / Swagger UI |
| Metrics | Micrometer counters via Actuator (`/actuator/prometheus`) |
| Logging | Structured logs with transaction/trace ID |
| CI (optional, last) | GitHub Actions |

**Out of scope:** Grafana/Prometheus servers, mobile app, real bank integration.

---

## 3. Architecture

```
Client / Swagger UI
        │
        ▼
 ┌──────────────────┐        ┌──────────────┐
 │  Wallet Service  │◄──────►│  Mock Bank   │
 │  (Spring Boot)   │ webhooks│ (separate    │
 │                  │        │  service)    │
 └───────┬──────────┘        └──────────────┘
         │
   ┌─────┴──────┐
   ▼            ▼
PostgreSQL     Kafka
(ledger,       (domain events,
 audit,         notifications)
 idempotency)
```

### Components
- **Wallet Service**: accounts, deposits, withdrawals, transfers, ledger, audit.
- **Mock Bank**: simulates authorize / capture / refund / transfer, with injectable failures.
- **PostgreSQL**: source of truth (ACID, row locking).
- **Kafka**: publishes events after commit (e.g. `TransferCompleted`).

---

## 4. Core Features (priority order)

| # | Feature | What it proves |
|---|---|---|
| 1 | Double-entry ledger | Debits = credits; financial correctness |
| 2 | Idempotency keys | Retrying a request never moves money twice |
| 3 | Exact money handling | Integers (minor units) or `BigDecimal`, explicit currency, no floats |
| 4 | Concurrency safety | Parallel transfers cannot create wrong/negative balances |
| 5 | Atomic transactions | Crash mid-transfer leaves no half-done state |
| 6 | Immutable audit trail | Every change traceable; no updates/deletes of history |
| 7 | Failure handling with Mock Bank | Timeouts, declines, duplicate/late webhooks handled |
| 8 | Reconciliation job | Ledger vs. balances vs. bank statement verified |

### Transfer flow

```
POST /transfers   (Idempotency-Key: abc123)
   │
   ├─ key already seen? ─► return stored result
   │
   └─ new ─► BEGIN TX
              lock accounts (consistent order)
              debit  A  -100
              credit B  +100
              write audit entry
              store idempotency result
            COMMIT
            publish TransferCompleted event
```

---

## 5. Mock Bank — Failure Scenarios

Injectable behaviors (via config or request header):
- Timeout / slow response
- Decline (insufficient funds, risk)
- Duplicate webhook delivery
- Delayed settlement
- Webhook out of order
- Service down

The Wallet Service must stay consistent in every case.

---

## 6. Metrics (evidence)

Micrometer counters, assertable in tests:

```
transfers_completed_total
transfers_rejected_total{reason="insufficient_funds"}
idempotent_replays_total
bank_call_failures_total
reconciliation_mismatches_total      # must stay 0
```

Structured logs include a transaction ID on every line.

---

## 7. Proof (tests)

Tests are the main evidence of competence.

- [ ] **Concurrency:** 1000 parallel transfers → total money unchanged, no negative balance
- [ ] **Idempotency:** same request sent 10× → exactly one transfer, `idempotent_replays_total` = 9
- [ ] **Atomicity:** failure mid-transaction → no partial ledger entries
- [ ] **Ledger invariant:** sum of all ledger entries = 0
- [ ] **Mock bank failures:** timeout, decline, duplicate webhook → consistent state
- [ ] **Reconciliation:** detects an injected mismatch

All integration tests run with **Testcontainers** (PostgreSQL, Kafka).

---

## 8. Optional / Later

- Auth (JWT/OAuth2) and role-based access
- Rate limiting
- Basic fraud rules (limits, velocity checks)
- Stripe test mode as a second payment provider
- **Usage-based charging module** (reserve → rate → deduct), tying in telecom charging experience
- Minimal web UI (balances, history, transfer form, "simulate bank failure" button)
- GitHub Actions CI with README badge

---

## 9. Milestones

1. **Skeleton**: project setup, Docker Compose (Postgres, Kafka), Flyway baseline
2. **Ledger core**: accounts, double-entry, exact money type, invariants
3. **Transfers**: atomic transactions, locking, idempotency
4. **Concurrency + idempotency tests**
5. **Mock Bank** + failure scenarios + webhooks
6. **Kafka events** (outbox pattern if time allows)
7. **Reconciliation job**
8. **Metrics + structured logs**
9. **README**: architecture diagram, design decisions, how to run
10. **Optional**: CI, usage-based charging, UI

---

## 10. README Checklist

- [ ] One-paragraph summary + "simulation, no real money" notice
- [ ] Architecture diagram
- [ ] One-command start (`docker compose up`)
- [ ] **Design decisions** section (why ledger, idempotency, locking strategy, trade-offs)
- [ ] How to run tests
- [ ] Short GIF / screen recording of a transfer + simulated failure

---

## 11. Open Questions

- Build tool: Maven or Gradle?
- Single module or multi-module (wallet-service + mock-bank)?
- Locking strategy: pessimistic row locks vs. optimistic versioning?
- Outbox pattern for Kafka publishing, or direct publish after commit?
- Money representation: `long` minor units vs. `BigDecimal`?
