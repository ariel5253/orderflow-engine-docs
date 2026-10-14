---
id: CONSTITUTION
title: Orderflow Engine — Constitution
status: accepted
lang: en
pair: none
last-reviewed: 2026-10-14
---

# Orderflow Engine — Constitution

These are the non-negotiable principles. A principle is never waived to save time. If a principle must change, a new ADR supersedes it first. Each article states how it is verified, and a principle without a verification is a wish, not an article.

| Art. | Principle | Why | Verified by |
|------|-----------|-----|-------------|
| P-01 | **Money is never a binary floating-point number.** Every amount is a `Money` value object holding a `BigDecimal` and a currency policy | Floating point cannot represent decimal cents exactly, and a cent drift between quote and order breaks trust | ArchUnit rule `DomainPurityArchTest` bans `float`/`double` in the domain. `MoneyTest` |
| P-02 | **The quoted total is the confirmed total.** The order references the sealed quote and stores no amount: nothing is copied or recomputed | Prices and levers can change between the two moments | GT-06 `QuoteConfirmRegressionIT` |
| P-03 | **A 201 is sent only after the commit** of the transaction that created the order | A number returned for an order that does not exist is the defect this system exists to prevent | `ConfirmOrderCommitOrderingIT`, `OrderReadYourWritesIT` |
| P-04 | **Every external effect goes through the outbox,** written in the same transaction as the order | An effect sent before commit can describe an order that never existed. An effect sent after commit can be lost | `OutboxWriteIT`, `OutboxRelayIT` |
| P-05 | **No remote call inside a database transaction** | It holds locks for the duration of someone else's latency and couples availability to theirs | `TransactionBoundaryArchTest`. Review of `OutboxRelay` |
| P-06 | **The domain is pure:** no Spring, JDBC, HTTP or clock access inside `pricing.domain` | A pure engine is deterministic, so it can be replayed and golden-tested | `DomainPurityArchTest`, the golden tests |
| P-07 | **No country data in code.** Currency, scale, rounding mode, tax rates, gift policy and TTL are data | The platform serves several countries and brands | `GoldenPricingTest#multiCountry`. Seeds in Liquibase |
| P-08 | **Every business rule has a test that has been seen failing.** A test is merged only after the guard it protects was broken once and the test caught it | A test that cannot fail proves nothing | TASKS cycle step 3. PIT mutation score ≥ 80% on `pricing.domain` |
| P-09 | **English for engineering, Spanish for users.** Code, identifiers, comments, logs, tests, commits and technical documents are in English. End-user text comes only from i18n resources, with Spanish as default and English complete | One language for the team. The user's language must be switchable without code changes | `ErrorCatalogI18nTest`, the web no-literal-text test, the `check-docs` key-parity check |
| P-10 | **Delivery documents exist in English and Spanish** with equivalent content | Both the team and the business must be able to read them | `check-docs` (pairs and heading parity) |
| P-11 | **Documentation is governed.** Every document is indexed, answers one question, and lives in its assigned folder | Loose files rot and contradict each other | `check-docs` |
| P-12 | **No claim without evidence.** Status, coverage and test results are stated only with the command that produced them | Documentation that promises unverified checks misleads the reader | [HANDOFF](../handoff/HANDOFF.md) §State |
| P-13 | **Comments explain why, never what** | The code already says what. The reason is what gets lost | Code review |
| P-14 | **Each repository README answers five questions:** what it is, how to run it, how to test it, where its configuration and data live, and where to read more | A repository must make sense when cloned alone | Review at T-16 |
| P-15 | **A smaller scope never lowers the standard.** No `TODO`, placeholder or dead code is delivered | An unfinished piece presented as finished costs more than a missing one | `check-docs`. Review. The API README lists what is not included and why |
