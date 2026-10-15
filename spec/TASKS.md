---
id: TASKS
title: Orderflow Engine — Task plan
status: accepted
lang: en
pair: none
last-reviewed: 2026-10-14
---

# Orderflow Engine — Task plan

**Goal:** deliver exactly what the assignment asks: the pricing engine (A1), the idempotent order confirmation (A2), one ADR of one page (A3), and the delivery items (single command, README, readable history). The optional order-summary screen comes after the cut line. Everything is governed by [SPEC.md](SPEC.md).

**Architecture:** a hexagonal modular monolith (SPEC §12), spread across sibling folders that will become repositories (SPEC §15). `orderflow-engine-api` is the deliverable.

**Stack:** Java 21 · Spring Boot · Maven Wrapper · PostgreSQL · Liquibase · JUnit 5 · AssertJ · Testcontainers · ArchUnit · JaCoCo · PIT · Angular · ngx-translate · Docker Compose · Node and Python (tools).

## Global constraints

| # | Constraint |
|---|-----------|
| GC-1 | SPEC.md and this file are the source of truth. A behavior change updates SPEC first |
| GC-2 | Each task ends green: `./mvnw verify` (API), `node orderflow-engine-tools/docs-check/check-docs.mjs` and `node orderflow-engine-tools/integrity/check-integrity.mjs`. From T-01 on, `orderflow-engine-infra/scripts/test-all` runs all three |
| GC-3 | Each task enters through one or more pull requests into `dev` (into `main` in `orderflow-engine-docs`), squash-merged, then the branch is deleted. A task too large to review in one sitting is split into thematic pull requests ([pull-request-rules](../governance/pull-request-rules.md), [golden rule](../governance/golden-rule.md)). Each title is a one-line commit of at most 60 characters ending with the task tag ([commit-conventions](../governance/commit-conventions.md)) |
| GC-4 | Stage only the task's own files. Never `git add -A`. Code repositories keep `dev` and `main`. `orderflow-engine-docs` keeps only `main` ([branches-and-environments](../governance/branches-and-environments.md)) |
| GC-5 | No `TODO`, placeholders or dead code. A smaller scope never means a lower standard |
| GC-6 | Documentation is updated in the same task that changes behavior. Delivery documents are always written in en and es |
| GC-7 | English for code, comments, logs, tests and commits. End-user text only through i18n bundles |
| GC-8 | At the end of every task, the State table in `handoff/HANDOFF.md` is updated with the output of the verification commands |
| GC-9 | Nothing beyond the assignment is added without updating SPEC §1 first |

## Task cycle

Every task follows the same cycle:
1. Write the failing test for the task's ACs and run it. **It must fail for the expected reason.**
2. Implement the minimum that makes it pass, then run it green.
3. **Prove that the test bites.** Break the guard once (flip a rule, remove a lock, skip a rounding), confirm the test fails, then restore the guard.
4. Run the full verification (GC-2).
5. Open the task's pull request with the message listed for the task as its title, paste the verification output, squash-merge it into its base and delete the branch ([pull-request-rules](../governance/pull-request-rules.md)).

## Review focus

These are the risky edges. A reviewer should check each one in the task that owns it.

| # | Edge | Task |
|---|------|------|
| RF-1 | Marginal brackets are rounded once per line, not once per segment | T-04 |
| RF-2 | Combo-consumed units never receive a tier discount, and a non-beneficial combo consumes nothing | T-05 |
| RF-3 | Combo allocation sums exactly (largest remainder, ties by SKU) | T-05 |
| RF-4 | Gift tax policy and the identity `total = Σ net + taxes` | T-06 |
| RF-5 | Totals are never re-rounded. COP scale 0 does not come from `java.util.Currency` | T-02, T-07 |
| RF-6 | Lock order idempotency → quote → credit, and the conditional credit update | T-11 |
| RF-7 | The 201 is serialized after commit, and failures are not cached | T-11, T-12 |
| RF-8 | The outbox relay never holds a database transaction during a remote call | T-14 |
| RF-9 | No hard-coded user text, and every error code has es and en messages | T-09, T-17 |

## Day plan and cut line

| Day | Tasks |
|-----|-------|
| D1 | T-00 … T-03 |
| D2 | T-04 … T-07 |
| D3 | T-08 … T-10 |
| D4 | T-11 … T-14 |
| D5 | T-15 … T-17 |

**MVP cut line after T-16.** T-17 is the optional screen of the assignment. At the end of D4, progress is checked against this plan. Any further cut is agreed explicitly and recorded under "Not included" in the API README.

---

## Tasks

### T-00: Specification, documentation governance and delivery documents
- **Covers:** SPEC §15, AC-36, AC-39, AC-40 (draft), GC-1, GC-6.
- **Files (`orderflow-engine-docs`):**
  - `README(.es).md` (map and glossary) and `LICENSE`
  - `handoff/HANDOFF.md`
  - `spec/{constitution, SPEC, plan, TASKS, api-contract, data-model, architecture}.md`
  - `diagrams/01…09-*.svg`
  - `assignment/`
  - `governance/`
  - generated results: `diagrams/`, `handoff/progress.svg`, `deliverables/`
- **Files (`orderflow-engine-api`):** `README(.es).md` with the required headings, `docs/adr/0001-order-pricing-and-confirmation(.es).md` (status `proposed`), `LICENSE`.
- **Files (`orderflow-engine-tools`):** `docs-check/`, `integrity/`, `progress/`, `diagrams/`, `render/`, `consistency/`, `requirements.txt`, `Dockerfile`, `.dockerignore`, `README(.es).md`, `LICENSE`.
- **Files (others):** `README(.es).md` and `LICENSE` in `orderflow-engine-web`, `orderflow-engine-infra` and `orderflow-engine-app`.
- **Files (every repository):** `.gitignore` and `.gitattributes`, as required by [repository-integrity](../governance/repository-integrity.md).
- **Verify:** `node orderflow-engine-tools/docs-check/check-docs.mjs` → OK, and `node orderflow-engine-tools/integrity/check-integrity.mjs` → OK.
- **Done when:** every Markdown file is indexed, paired and linked; the API README and the ADR carry their required headings; the ADR fits in 500 words per language; every repository passes the integrity check.
- **Delivery:** the preparation phase. It enters each repository through thematic pull requests (listed in the HANDOFF), and closes with this merge commit on the `main` of each code repository.
- **Commit:** `chore: close preparation phase (T-00)`

### T-01: Scaffold the API, its containers and the workspace test command
- **Covers:** AC-38, AC-47 (domain purity part).
- **Files (`orderflow-engine-api`):**
  - `pom.xml` and the Maven Wrapper: Java 21, Spring Boot, JDBC, Liquibase, Actuator, Testcontainers, ArchUnit, JaCoCo (gate ≥ 90% on `pricing`), and PIT in the `mutation` profile
  - `Application.java`, `application.yml` (every connection value from environment variables), `db/changelog/db.changelog-master.yaml`
  - `Dockerfile`: a multi-stage build with the Maven Wrapper on JDK 21, run on a Java 21 runtime image as a non-root user
  - `compose.yaml`: the service and PostgreSQL with a healthcheck. The service starts once the database is healthy. Values come from `.env` when it exists, with the `.env.example` values as defaults, so a fresh clone starts without one
  - `.env.example`: the rows of the configuration table of [plan](plan.md) §2 that T-01 wires (database connection and port), one comment per key, no real secret. Later tasks add their own keys
  - `.dockerignore`
  - `DomainPurityArchTest`, `HealthIT`
- **Files (`orderflow-engine-infra`):**
  - `compose.yaml`: the full stack, built from the sibling repositories cloned side by side (database and API; the web screen joins in T-17)
  - `.env.example`: every key with its non-secret local default and `change-me` for the password, one comment per key ([repository-integrity](../governance/repository-integrity.md) §2)
  - `scripts/test-all.sh`, `scripts/test-all.ps1`: API verify, documentation check and integrity check
- **Verify:** `./mvnw verify` → green, and `node orderflow-engine-tools/integrity/check-integrity.mjs` → OK.
- **Done when:**
  - `/actuator/health` returns UP against Testcontainers PostgreSQL (`HealthIT`)
  - `docker compose up --build` in `orderflow-engine-api`, with no other repository present and no `.env`, starts the service and its database, and `/actuator/health` returns UP (the README's How to run)
  - `docker compose up --build` in `orderflow-engine-infra` starts the same stack from the side-by-side clones
  - the ArchUnit rule fails if `pricing.domain` imports Spring (proven once)
- **Commit:** `chore(api): scaffold service and test command (T-01)`

### T-02: Money and currency policy
- **Covers:** AC-08, RF-5.
- **Files:** `pricing/domain/Money.java`, `CurrencyPolicy.java`, `MoneyTest`, `CurrencyPolicyTest`, and the no-float rule in `DomainPurityArchTest`.
- **Tests:**
  - mixing currencies → exception
  - a scale above the policy → rejected
  - HALF_UP at scale 0 and at scale 2
  - `compareTo` ignores trailing zeros
- **Commit:** `feat(pricing): add money value object (T-02)`

### T-03: Pricing pipeline, normalization and golden harness
- **Covers:** AC-02, AC-03 (skeleton), AC-13, BR-00, BR-05.
- **Files:**
  - `pricing/domain/{PricingEngine, PricingSnapshot, Cart, CartLine, Breakdown, LineBreakdown, Totals, CartNormalizer}.java`
  - `GoldenPricingTest`: `@Tag("golden")`, loads `golden/<case>/{request,snapshot,expected}.json`
  - `CartNormalizationTest`
  - `PricingPipelineOrderTest`: the steps run in the BR-01 order, extended as each lever lands (T-04 … T-06)
- **Interfaces:** produces `Breakdown PricingEngine.price(Cart, PricingSnapshot)`, which is pure and deterministic.
- **Commit:** `feat(pricing): add pricing pipeline and goldens (T-03)`

### T-04: Tier discounts
- **Covers:** AC-04, AC-15, BR-02, RF-1.
- **Files:** `pricing/domain/lever/{TierRule, TierDiscountStep}.java`, `golden/gt01-two-tier-brackets/*`, `TierPrecedenceTest`.
- **Commit:** `feat(pricing): apply marginal tier discounts (T-04)`

### T-05: Combos and allocation
- **Covers:** AC-05, AC-07, BR-03, RF-2, RF-3.
- **Files:** `pricing/domain/lever/{ComboRule, ComboStep, LargestRemainderAllocator}.java`, `golden/gt02-combo-no-loose-discount/*`, `golden/gt04-combo-with-taxes/*`.
- **Tests:**
  - GT-02 seeds a tier rule on the combo SKUs, and that rule must not fire
  - GT-04 uses the numbers of SPEC BR-03
  - a non-beneficial combo consumes nothing
  - the allocations sum exactly to the combo discount
- **Commit:** `feat(pricing): apply combos before tiers (T-05)`

### T-06: Gifts and gift tax policy
- **Covers:** AC-06, AC-12, BR-04, RF-4.
- **Files:** `pricing/domain/lever/{GiftRule, GiftStep, GiftTaxPolicy}.java`, `golden/gt03-gift/*`, `golden/gift-policy-{exempt,taxed}/*`.
- **Commit:** `feat(pricing): add gifts with tax policy (T-06)`

### T-07: Rounding and multi-country
- **Covers:** AC-09, AC-11, RF-5.
- **Files:** `RoundingPolicyTest`, `golden/rounding-half-cases/*`, `golden/multi-country-co-vs-ec/*`.
- **Commit:** `test(pricing): cover rounding and countries (T-07)`

### T-08: Schema and seed data
- **Covers:** AC-10, SPEC §5.
- **Files:** `db/changelog/2026/2026-10-NN-00{1…5}-*.yaml` (tenancy and catalog, levers, credit, quotes and orders, idempotency and outbox), `…-006-seed-co-and-ec.yaml`, each with a rollback and a `logicalFilePath` and included explicitly by the master changelog; `PriceScaleValidationIT`, `SchemaConstraintsIT`, `LiquibaseRollbackIT`.
- **Tests (`SchemaConstraintsIT`):**
  - composite tenant FKs reject cross-tenant references
  - `UNIQUE (tenant_id, quote_id)` on orders
  - the order FK pins `customer_id` to the quote
  - the credit `CHECK` holds
  - a confirmed quote cannot be deleted
- **Tests (`LiquibaseRollbackIT`):** update, roll back to an empty schema, update again, on Testcontainers PostgreSQL. Proven once with a changeset whose rollback is wrong (DEC-20).
- **Commit:** `feat(db): add normalized schema and seeds (T-08)`

### T-09: Error handling and API i18n
- **Covers:** AC-18, AC-19, AC-37, RF-9.
- **Files:** `adapters/in/web/{ProblemDetailsAdvice, ErrorCode, LocaleConfig, CorrelationIdFilter}.java`, `i18n/messages{,_es,_en}.properties`, `ErrorCatalogI18nTest`, `ProblemDetailContractIT`, `LocalizedErrorIT`.
- **Commit:** `feat(api): add localized problem details (T-09)`

### T-10: Quote use case and endpoints (A1)
- **Covers:** AC-01, AC-14, AC-16, AC-17 (quote side), AC-20, BR-06, BR-07.
- **Files:** `quoting/{CreateQuoteUseCase, GetQuoteUseCase, QuoteRepository, SnapshotProvider, CreditAdvisor}.java`, the JDBC repositories, `QuoteController` and its DTOs, `golden/gt05-credit-exceeded/*` with `GoldenQuoteTest` (`@Tag("golden")`, the quote use case over fixture-backed ports), `QuoteContractIT`, `QuoteValidationIT`, `QuotePersistenceIT`, `CreditAdvisoryTest`.
- **Commit:** `feat(quoting): add sealed quotes (T-10)`

### T-11: Order confirmation (A2)
- **Covers:** AC-23, AC-24, AC-25, AC-28, AC-32, AC-47, RF-6, RF-7.
- **Files:** `ordering/{ConfirmOrderUseCase, OrderRepository, CreditLedger, IdempotencyStore, OutboxWriter}.java`, `OrderController`, `IdempotencyContractIT`, `IdempotencyReplayIT`, `QuoteAlreadyConfirmedIT`, `OutboxWriteIT`, `TransactionBoundaryArchTest`.
- **Commit:** `feat(ordering): confirm orders atomically (T-11)`

### T-12: Prove the invariants under concurrency and failure
- **Covers:** AC-17 (confirm side), AC-21, AC-26, AC-27, AC-29, AC-30.
- **Files:** `QuoteConfirmRegressionIT` (GT-06, `@Tag("golden")`), `ConfirmOrderConcurrencyIT`, `ConfirmOrderFailureInjectionIT`, `ConfirmOrderCommitOrderingIT`, `CreditLimitIT`.
- **Commit:** `test(ordering): prove confirmation invariants (T-12)`

### T-13: Order history
- **Covers:** AC-31.
- **Files:** `ordering/{GetOrderUseCase, ListOrdersUseCase}.java`, the history endpoints (newest first, `?size=` ≤ 100), `OrderReadYourWritesIT`.
- **Commit:** `feat(ordering): add order history endpoints (T-13)`

### T-14: Outbox relay
- **Covers:** AC-33, AC-34, AC-35, RF-8.
- **Files:**
  - `outbox/{OutboxRelay, OutboxRepository, BackoffPolicy, PushNotificationPort, ErpPublisherPort}.java`
  - the fake gateways, which deduplicate by `event_id` and can be told to fail
  - the `outbox_pending` and `outbox_dead` gauges
  - `OutboxRelayIT`
- **Commit:** `feat(outbox): relay order events reliably (T-14)`

### T-15: Retention housekeeping
- **Covers:** AC-48.
- **Files:** `outbox/HousekeepingJob.java`, `HousekeepingIT`.
- **Commit:** `feat(outbox): purge expired data by retention (T-15)`

### T-16: Final documentation and verification
- **Covers:** AC-36, AC-39, AC-40, AC-41.
- **Steps:**
  - set ADR-0001 to `accepted`
  - finalize the API README, including "Not included" with the real time spent
  - run `./mvnw verify`, `./mvnw -Pmutation verify` (PIT ≥ 80%) and `test-all`
  - record the commands, coverage and mutation score in the HANDOFF
- **Commit:** `docs: accept ADR and record evidence (T-16)`

**— MVP cut line —**

### T-17: Order-summary screen (optional plus, at most 1 hour)
- **Covers:** AC-42 … AC-46, RF-9.
- **Files (`orderflow-engine-web`):**
  - Angular workspace
  - `ui-text`, `ui-action-bar`, `ui-product-image`
  - the `order-summary` page
  - `public/i18n/{es,en}.json`
  - the fixture `order-summary.fixture.json`
  - `order-summary.spec.ts`
  - the README section "Prevention at component level"
  - `Dockerfile` (build with Node, serve the static screen with nginx as a non-root user), `.dockerignore`, `compose.yaml` and `.env.example`
- **Files (`orderflow-engine-infra`):** the web service added to `compose.yaml`, and `test-all` extended with the web tests.
- **Done when:** the test passes, `docker compose up --build` serves the screen, and the real time spent is recorded.
- **Commit:** `feat(web): add order summary screen (T-17)`
