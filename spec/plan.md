---
id: PLAN
title: Orderflow Engine — Technical plan
status: accepted
lang: en
pair: none
last-reviewed: 2026-10-15
---

# Orderflow Engine — Technical plan

> **Question this document answers:** which technical decisions were taken to satisfy the [specification](SPEC.md), and which risks remain?

This plan lists every technical decision with its rejected alternatives, and keeps the risk register. The structural decisions are summarized in the single ADR the assignment asks for, ADR-0001 (`orderflow-engine-api/docs/adr/`).

## 1. Technical decisions

| Code | Decision | Rejected alternatives | Serves | ADR |
|------|----------|----------------------|--------|-----|
| DEC-01 | Java 21 with Spring Boot (servlet stack) | Spring WebFlux: reactive drivers complicate the single blocking transaction that A2 depends on, with no throughput need to justify it. Go and NestJS were dropped by preference and because `BigDecimal` is native | SPEC §7 | 0001 |
| DEC-02 | Maven Wrapper as the build tool | Gradle: no advantage for a single-module service, and the wrapper removes the need for a local install | GC-2 | — |
| DEC-03 | Hexagonal modular monolith with a pure pricing domain | Microservices: one transaction must span order, credit, idempotency and outbox, and splitting them buys a saga for no benefit | SPEC §12, P-06 | 0001 |
| DEC-04 | PostgreSQL with Liquibase changelogs (YAML, one rollback per changeset) | In-memory store: it cannot prove atomicity or row locks. Flyway: no rollback support in the free edition | SPEC §7 | 0001 |
| DEC-05 | Spring `JdbcClient` with explicit SQL | JPA/Hibernate: it hides the locks and the flush order that the A2 invariants depend on | SPEC §7.1 | 0001 |
| DEC-06 | `BigDecimal` money with per-country scale and rounding, rounded per line | `long` minor units: awkward with tax rates. Rounding per tax group or per total was rejected (SPEC §4) | SPEC §4 | 0001 |
| DEC-07 | Insert-only quote lines and discount components. The order references the quote and stores no amount | Copy the breakdown into the order: two copies can diverge. Recompute at confirm and compare: fails whenever prices change. Price locking: blocks catalog updates | SPEC §6 | 0001 |
| DEC-08 | Idempotency keys in the order transaction (`ON CONFLICT DO NOTHING`) | Redis or a cache: not atomic with the order. Client-generated order ids: they do not cover replays of the response | SPEC §7.3 | 0001 |
| DEC-09 | Transactional outbox with a leased polling relay | Publishing after commit: loses events on a crash. Two-phase commit: unavailable for push and ERP. Change data capture: too much infrastructure for this scope | SPEC §7.4 | 0001 |
| DEC-10 | RFC 9457 `problem+json` with a stable `code` | Custom error envelope: reinvents a standard | SPEC §8 | — |
| DEC-11 | Spring `MessageSource` (API) and ngx-translate runtime JSON (web), es default | Angular built-in i18n: one build per locale | SPEC §9 | — |
| DEC-12 | Testcontainers PostgreSQL for integration tests | H2: different locking and SQL dialect, so it would prove nothing about A2 | SPEC §11 | — |
| DEC-13 | Golden tests as JSON fixtures (request, snapshot, expected) | Inline assertions: hard to review numbers, easy to drift | SPEC §11 | — |
| DEC-14 | JUnit 5 runs the tests, JaCoCo measures line and branch coverage (gate ≥ 90% on `pricing`), PIT measures whether the tests detect wrong code (≥ 80% on `pricing.domain`, optional profile), ArchUnit guards the architecture | Coverage alone: says nothing about assertion quality. SonarQube or SonarCloud now: needs a server the evaluator cannot run, and it does not measure coverage itself but imports JaCoCo's report (adoption path in §3). Cobertura: unmaintained. OpenClover: much smaller ecosystem | P-06, P-08 | — |
| DEC-15 | Sibling folders, each a future repository, with a docs folder that owns the specification | Monorepo: one history and one command, but it ties together components that are released, owned and evolved independently (service, screen, infrastructure, documentation) | SPEC §15 | 0001 |
| DEC-16 | Angular standalone components for the web client: summary screen and full flow, SPEC §10 | Flutter: not installed in the build environment, and the assignment accepts either | SPEC §10 | — |
| DEC-18 | Documentation governance enforced by `check-docs.mjs`, with no dependencies, in a separate `orderflow-engine-tools` repository that also generates the figures, the progress chart and the ADR in Word and PDF | Manual review only: rules that are not checked decay | P-11 | — |
| DEC-19 | Relational model in third normal form, no JSON columns. Tenant-scoped composite keys `(tenant_id, id)` and UUIDv7 ids | JSON documents for quotes: not constrainable or queryable per line. Global surrogate keys only: allow cross-tenant references and need re-keying to shard | data-model §7, §9 | 0001 |
| DEC-20 | The Liquibase changelogs live in `orderflow-engine-api`, the only component that talks to PostgreSQL, and are applied when the service starts. Each changeset has a rollback that `LiquibaseRollbackIT` proves | A separate migration job with a DML-only runtime role: needs a second credential and a different lock for the quote (`SELECT … FOR UPDATE` requires UPDATE privilege). A separate schema repository applied by infra or a submodule: a plain clone of the API would have no schema, so `./mvnw verify` would stop being a single command. A schema image or artifact on GitHub Packages: the build would depend on a registry and its access rules. Both separate repositories split every schema change from the SQL that relies on it | SPEC §0, §11, §12, AC-38 | — |

## 2. Configuration

| Key | Meaning | Default |
|-----|---------|---------|
| `app.i18n.default-locale` | Locale used when `Accept-Language` is missing or unsupported | `es` |
| `app.outbox.poll-interval` | Relay polling interval | `1s` |
| `app.outbox.lease` | How long a claimed event stays leased (`locked_until`) | `30s` |
| `app.outbox.max-attempts` | Attempts before an event becomes `DEAD` | `8` |
| `app.outbox.backoff.base` / `.max` | Exponential backoff bounds (full jitter) | `2s` / `5m` |
| `app.idempotency.retention` | How long idempotency keys are kept | `24h` |
| `app.housekeeping.retention` | Retention of expired quotes and `SENT` events | `7d` |
| `spring.datasource.*` | Database connection, from environment variables only | — (no secret defaults) |
| `server.port` | HTTP port of the service | `8080` |

The quote TTL, currency scale, rounding mode and gift tax policy are **country data**, not application configuration (P-07).

`orderflow-engine-api/.env.example` mirrors this table, one comment per key and no real secret ([repository-integrity](../governance/repository-integrity.md) §2). A key enters it in the task that wires it (T-01: database connection and port), so it never lists dead configuration (GC-5).

## 3. Adoption paths

These are not implemented now, because the assignment does not ask for them and the evaluator could not run them. For the first five, the project already produces what each one needs, so adopting them later changes CI configuration, not the code. The last one is a product feature with a ready design.

| Item | Today | How it is adopted later | Project changes | Trigger |
|------|-------|-------------------------|-----------------|---------|
| SonarQube or SonarCloud | Not used | A CI job runs the scanner after `./mvnw verify` and imports `target/site/jacoco/jacoco.xml` | None. The JaCoCo XML report is already produced | The team wants a shared quality dashboard and quality gates on pull requests |
| GitHub Actions | Verification runs locally, and its output is pasted into each pull request | A workflow runs `./mvnw verify` and `check-docs` on every pull request, and the branch protections require it | None. The commands already exist | The team wants verification enforced by the platform instead of by the pull request body |
| Migration job and least-privilege roles | Liquibase runs at startup with the database owner's credential | A one-shot migration job runs Liquibase with an owner role before the service starts. The service connects with a DML-only role, and the quote lock of SPEC §7.1 moves to `pg_advisory_xact_lock` | The datasource keys, a grants changeset, the quote lock, and a test that runs as the DML-only role | A deployed environment, more than one replica, or a security review of database privileges |
| Separate schema repository | The changelogs live with the API, their only owner (DEC-20) | The changelogs move to their own repository, published as a versioned artifact that the API pins and the deployment applies first | A `git mv` of `db/changelog/` (every changelog already has a `logicalFilePath`), a pinned version, and a pipeline step | A second service or team that owns or reads the same schema |
| PIT in CI | Optional profile `./mvnw -Pmutation verify`, run before each delivery, and the score is recorded in the HANDOFF | A nightly CI job, or `scmMutationCoverage` on the files changed in a pull request. `withHistory` makes the runs incremental | None. Only CI configuration | Mutation runs are wanted on every change, but a full run is too slow for each build |
| Catalog in several languages | Product descriptions are tenant data in the tenant's language (SPEC §10.1). The ES/EN switch translates the interface only | A `product_translations (tenant_id, product_id, locale, description)` table. A translation is written when the product is entered or changed, by the merchant or proposed by a translation service and reviewed by a person; it is never produced while a page is read. E-01, E-02 and E-07 return the description for `Accept-Language`, falling back to the tenant's language. A quote line keeps the description it was sealed with, so a confirmed order never changes | A changeset, the snapshot and product queries, the E-01/E-02/E-07 cards, and contract tests per language | The catalog becomes editable (an administration screen or import exists) and a tenant sells to buyers who read another language |

## 4. Risk register

| Code | Risk | Probability | Impact | Mitigation | Owner task |
|------|------|-------------|--------|------------|-----------|
| RISK-01 | Contention on a customer's credit row under bursts of confirmations | Medium | Latency | One short transaction, a `lock_timeout`, and 409 `IDEMPOTENCY_KEY_IN_PROGRESS` instead of waiting forever. Measured in `ConfirmOrderConcurrencyIT` | T-13 |
| RISK-02 | The outbox grows while the ERP is down for a long time | Medium | Storage, delayed ERP | Backoff, `DEAD` after max attempts, `outbox_pending` gauge, and the requeue procedure in the "Operations" section of the API README | T-15 |
| RISK-03 | Prices honored for the quote TTL after a price error | Low | Revenue | Accepted and recorded as the ADR risk. A short TTL (15 min) bounds it. A revocation operation is the documented next step | T-16 |
| RISK-04 | Engine change silently alters prices of new quotes | Medium | Pricing errors | Golden tests fail the build | T-07 |
| RISK-05 | A downstream consumer does not deduplicate by `event_id` | Medium | Duplicate push or ERP order | `event_id` contract, push collapse id, ERP upsert by order number. Stated in the ADR | T-15 |
| RISK-06 | Sibling repositories drift once split (links, contract) | Medium | Broken docs | The contract lives only in `orderflow-engine-docs`, and `check-docs` verifies cross-folder links. Relative links between repositories become absolute URLs when the repositories are published | T-16 |
| RISK-07 | Five-day window | High | Scope | Day plan and cut line in [TASKS](TASKS.md). Cuts are agreed explicitly and recorded in the API README under "Not included" | — |
