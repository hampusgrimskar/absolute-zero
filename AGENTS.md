# AGENTS.md — AbsoluteZero

Instructions for AI agents working in this repository.
Each section covers one topic, so you can jump to the behaviour you need.

## Table of Contents

1. Project Overview
2. Scope and Editing Discipline
3. Tech Stack
4. Repository Layout
5. Build, Run and Test Commands
6. Money Handling Rules
7. Ledger Rules
8. Idempotency Rules
9. Concurrency and Transactions
10. Holds (Reserve / Capture / Release)
11. Mock Bank
12. Database and Migrations
13. Kafka and Events
14. Testing Rules
15. Metrics and Logging
16. Security and Secrets
17. Code Style
18. Git and Commits
19. Documentation
20. Definition of Done
21. Communication Style

---

## 1. Project Overview

AbsoluteZero is a **simulated wallet / payments backend** built as a fintech portfolio project.
Its purpose is to demonstrate that money is never lost, duplicated or miscalculated, even when things fail.

- **Core invariant:** every transaction sums to zero (double-entry ledger).
- **Focus:** backend correctness. No mobile app. UI is optional.
- **Simulation only:** no real money, no real bank.

Design document: `docs/design.md`

## 2. Scope and Editing Discipline

- Do **only** what was explicitly asked.
- Make the **smallest possible change**. Prefer targeted edits over rewriting files.
- Do not refactor, rename, reformat or "clean up" code that was not part of the request.
- Do not add features, dependencies or configuration that were not requested.
- If extra changes seem necessary, **stop and ask first**.
- Never revert, reset or check out files without explicit instruction. Uncommitted work may exist.

## 3. Tech Stack

| Area | Choice |
|---|---|
| Language | Java 21 |
| Framework | Spring Boot 3 |
| Build tool | Maven |
| Database | PostgreSQL |
| Migrations | Flyway |
| Messaging | Kafka |
| Testing | JUnit 5 + Testcontainers |
| Local run | Docker Compose |
| API docs | OpenAPI / Swagger UI |
| Metrics | Micrometer via Actuator |

- Do not introduce a new library or framework without asking.
- When adding a dependency, pin an exact version.

## 4. Repository Layout

Planned structure (adjust only when asked):

```
absolute-zero/
├── wallet-service/     Spring Boot app: accounts, ledger, transfers, holds
├── mock-bank/          Separate service simulating a bank, with injectable failures
├── docker-compose.yml  Postgres, Kafka, services
├── docs/               Design notes, diagrams
├── AGENTS.md
└── README.md
```

Update this section when the structure changes.

## 5. Build, Run and Test Commands

> Fill in or correct these once the project skeleton exists.

```
Build:         mvn clean verify
Unit tests:    mvn test
Integration:   mvn verify
Start infra:   docker compose up -d
Stop infra:    docker compose down
```

- Run the relevant tests before reporting work as done.
- If a command fails, report the actual error. Do not claim success.

## 6. Money Handling Rules

- **Never use `float` or `double` for money.**
- Use integer **minor units** (`long`) or `BigDecimal`. Pick one and use it consistently.
- Every amount carries an explicit **currency**.
- Never mix currencies in one operation without explicit conversion.
- Rounding rules must be explicit and tested.
- Amounts must be validated: no negative or zero amounts where not allowed.

## 7. Ledger Rules

- The ledger is **double-entry**: every transaction has debit and credit entries that sum to **zero**.
- Ledger entries are **immutable**. Never update or delete them. Corrections are new entries.
- Balances are derived from, or verified against, ledger entries.
- A transaction must be recorded **atomically**: all entries or none.
- Every ledger change must produce an audit trail entry.

## 8. Idempotency Rules

- Every state-changing endpoint accepts an `Idempotency-Key` header.
- Same key + same request → return the **stored original result**, do not execute again.
- Same key + different request → reject with a clear error.
- Idempotency records are stored in the same database transaction as the money movement.
- Idempotent replays must be counted by a metric (see section 15).

## 9. Concurrency and Transactions

- Money movements run inside a **single database transaction**.
- Lock accounts in a **consistent order** to avoid deadlocks.
- Choose the locking strategy deliberately (pessimistic row locks or optimistic versioning) and document the reason in `docs/`.
- No balance may go negative unless the account type explicitly allows it.
- Never call an external system (mock bank, Kafka) while holding database locks longer than necessary.

## 10. Holds (Reserve / Capture / Release)

- A **hold** reduces the available balance without deducting it.
  `available = total - active holds`
- Operations: **hold → capture** (deduct actual amount) or **hold → release** (cancel).
- Capture amount must not exceed the held amount.
- Holds have an **expiry**. Expired holds are released automatically.
- Hold, capture and release must all be **idempotent**.
- Holds are used for withdrawals to the bank and authorize/capture flows.

## 11. Mock Bank

- The mock bank is a **separate service**, not a class inside the wallet service.
- It must support injectable failures: timeout, decline, duplicate webhook, delayed settlement, out-of-order webhook, service down.
- Failure injection is controlled by configuration or request header, never by random behaviour that makes tests flaky.
- The wallet service must stay **consistent** under every failure scenario.
- Clearly mark the mock bank as a simulation in code comments and README.

## 12. Database and Migrations

- All schema changes go through **Flyway** migrations in versioned files.
- **Never edit an applied migration.** Add a new one.
- Use database constraints (NOT NULL, foreign keys, unique, check) to enforce invariants where possible.
- No manual schema changes outside migrations.

## 13. Kafka and Events

- Events describe facts that already happened (e.g. `TransferCompleted`).
- Publish events **after** the database commit, or use the outbox pattern. Never publish before commit.
- Consumers must tolerate **duplicate** and **out-of-order** delivery.
- Event schemas are versioned. Do not change published event fields without a version bump.

## 14. Testing Rules

- Use **JUnit 5** and **Testcontainers** (real PostgreSQL and Kafka). Do not mock the database for integration tests.
- Every core guarantee needs a test that **proves** it:
  - Concurrency: many parallel transfers keep total money constant
  - Idempotency: repeated request causes exactly one transfer
  - Atomicity: mid-transaction failure leaves no partial entries
  - Ledger invariant: sum of all entries is zero
  - Mock bank failures: timeout, decline, duplicate webhook leave consistent state
  - Reconciliation: detects an injected mismatch
- Tests must be **deterministic**. No sleeps as synchronization, no reliance on timing luck.
- Add or update tests together with the code change.
- Do not delete or weaken a failing test to make a build pass. Fix the cause or report it.

## 15. Metrics and Logging

Micrometer counters exposed via Actuator (`/actuator/prometheus`):

```
transfers_completed_total
transfers_rejected_total{reason="..."}
idempotent_replays_total
bank_call_failures_total
reconciliation_mismatches_total      # must stay 0
```

- Tests may assert on these counters.
- Logs are **structured** and include a transaction/trace ID.
- **Never log** secrets, tokens or full personal data.

## 16. Security and Secrets

- Never commit secrets, tokens, passwords or `.env` files.
- Read configuration from environment variables or ignored local config.
- If a file may contain secrets, do not print its values. Refer to keys by name.
- Validate all external input (API requests, webhooks, events).
- Flag any endpoint without authentication in the summary of your work.

## 17. Code Style

- Match the existing code style and conventions in the repository.
- Prefer clear names and small methods over comments.
- Comments explain **why**, not what.
- Domain logic stays free of framework code where practical.
- Use `final`, records and immutable types where they fit.
- No unused code, no commented-out code.

## 18. Git and Commits

- Default branch is `master`. Solo project: committing directly to `master` is acceptable.
- **Only commit when explicitly asked.**
- Stage specific files. Avoid `git add .`.
- Commit messages: short imperative summary, e.g. `Add idempotency key handling to transfers`.
- One logical change per commit.
- Never force-push, rewrite history or skip hooks without explicit permission.
- Never commit build output (`target/`) or secrets.

## 19. Documentation

- Keep `README.md` up to date: summary, architecture diagram, how to run, how to test.
- Record non-obvious design decisions in `docs/` with the trade-offs considered.
- The README must state that this is a **simulation** and not for real money.
- Update this file when conventions change.

## 20. Definition of Done

A task is done only when:

- [ ] The requested change is implemented, with nothing extra
- [ ] The project builds
- [ ] Relevant tests were run and pass (cite the actual result)
- [ ] New behaviour has a test proving it
- [ ] No secrets or build output were added
- [ ] Docs were updated if behaviour or structure changed

If something could not be verified, say so explicitly.

## 21. Communication Style

- Be short, direct and factual.
- Report what was done and what was verified. Do not speculate.
- Explain concepts simply and visually when asked. Assume the reader does not have the full mental model.
- Ask before making decisions that change scope.
