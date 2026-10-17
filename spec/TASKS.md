---
id: TASKS
title: Orderflow Engine — Task plan
status: accepted
lang: en
pair: none
last-reviewed: 2026-10-15
---

# Orderflow Engine — Task plan

**Goal:** deliver exactly what the assignment asks: the pricing engine (A1), the idempotent order confirmation (A2), one ADR of one page (A3), and the delivery items (single command, README, readable history). The optional web client comes after the cut line: the order summary the assignment offers as a plus, inside the full flow the product owner added (SPEC §10, §14.3). Everything is governed by [SPEC.md](SPEC.md).

**Architecture:** a hexagonal modular monolith (SPEC §12), spread across sibling folders that will become repositories (SPEC §15). `orderflow-engine-api` is the deliverable.

**Stack:** Java 21 · Spring Boot · Maven Wrapper · PostgreSQL · Liquibase · JUnit 5 · AssertJ · Testcontainers · ArchUnit · JaCoCo · PIT · Angular · ngx-translate · Vitest with Playwright Chromium (web specs) · nginx (web container and `/api/` proxy) · Docker Compose · Node and Python (tools).

## Global constraints

| # | Constraint |
|---|-----------|
| GC-1 | SPEC.md and this file are the source of truth. A behavior change updates SPEC first |
| GC-2 | Each task ends green: `./mvnw verify` (API), `node orderflow-engine-tools/docs-check/check-docs.mjs` and `node orderflow-engine-tools/integrity/check-integrity.mjs`. From T-01 on, `orderflow-engine-infra/scripts/test-all` runs all three. From T-17 on, it also runs the web tests in their container (`docker build --target test` in `orderflow-engine-web`), so the host needs no Node |
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
| D5 | T-15, T-18, T-16, then T-17 slice by slice, then T-19 and T-20, then the delivery set T-21 |

**MVP cut line after T-16.** T-17 is the optional plus of the assignment, extended by the product owner to the full web flow (SPEC §14.3). It is estimated at about 13.5 h, more than D5 holds after T-16, and its slices are cut-safe (T-17, Delivery). Pull request 1 of T-17 (the specification) lands first, as GC-9 requires; every other T-17 pull request starts only after T-16 is merged, and T-17 never delays T-12…T-16. At the end of D4, progress is checked against this plan (HANDOFF OD-02). Any further cut, including the T-17 slices not merged at delivery, is agreed explicitly and recorded under "Not included" in the API README.

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
- **Files:** `adapters/in/web/{ProblemDetailsAdvice, ErrorCode, ApiException, LocaleConfig, CorrelationIdFilter}.java`, the test-scoped `ProblemProbeController`, `i18n/messages{,_es,_en}.properties`, `ErrorCatalogI18nTest`, `ProblemDetailContractIT`, `LocalizedErrorIT`.
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
- **Files:** `ordering/{GetOrderUseCase, ListOrdersUseCase}.java`, the history endpoints (newest first, `?size=` ≤ 100, `currency` once per E-05 body, 404 `CUSTOMER_NOT_FOUND` when `customerId` is not the caller), `OrderReadYourWritesIT`, `OrderHistoryContractIT`.
- **Tests (`OrderHistoryContractIT`):**
  - every E-04 and E-05 error row of the API contract, with its status and body
  - a `customerId` other than `X-Customer-Id` → 404 `CUSTOMER_NOT_FOUND` (BD-01), whether or not that customer exists
  - `currency` once per body, also when `items` is empty, and every `total` a string at that currency's scale (no decimals for COP, two for USD)
  - items newest first, and `size` above 100 returns at most 100
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

### T-18: Wire the levers into quotes
- **Covers:** AC-01, AC-04, AC-05, AC-06, AC-07, AC-17 (quote side).
- **Why:** T-10 was built before T-04 … T-06, so its quote path loads prices and tax rates only and seals no discount component. The engine applies the levers, but E-01 cannot until the snapshot carries them.
- **Files:** `adapters/out/persistence/JdbcSnapshotProvider` (the tenant's active tier rules and brackets, combo rules and items, and gift rules into `PricingSnapshot`), `quoting/CreateQuoteUseCase` (seals the TIER and COMBO components, the combo allocations and the gift lines with their origin and reference value), `QuoteLeversIT`, and the lever leg of `QuoteConfirmRegressionIT` (GT-06) through `POST /api/v1/quotes`.
- **Tests:**
  - through E-01 against the seeds: the GT-01 totals (tiers), the GT-04 totals and allocation (combo with taxes), and a gift line from the seeded gift rule
  - a lever deactivated after a quote no longer applies to a fresh quote, while the sealed quote and its confirmation keep it (GT-06)
- **Runs:** after T-04 … T-07 and T-10 are on `dev`, and before T-16 is finalized.
- **Commit:** `feat(quoting): price quotes with seeded levers (T-18)`

### T-16: Final documentation and verification
- **Covers:** AC-36, AC-39, AC-40, AC-41.
- **Steps:**
  - set ADR-0001 to `accepted`
  - finalize the API README, including "Not included" with the real time spent
  - run `./mvnw verify`, `./mvnw -Pmutation verify` (PIT ≥ 80%) and `test-all`
  - record the commands, coverage and mutation score in the HANDOFF
- **Commit:** `docs: accept ADR and record evidence (T-16)`

**— MVP cut line —**

### T-17: Web client: order summary and full flow (optional plus)
- **Covers:** AC-42 … AC-46, AC-49 … AC-53, RF-9.
- **Scope:** SPEC §10 and its §14.3 rows. The order summary is the assignment's plus (slice T-17a). The cart, the confirmation, the order, My orders, the store picker and E-07 are the product owner's extension beyond the assignment's one-hour screen (SPEC §1, Deviation).
- **Starts:** pull request 1 (the specification) lands first, as GC-9 requires; it was delivered inside `docs(spec): align spec with built API and web plan (T-16)`. Every other T-17 pull request starts only after T-16 is merged. T-17 never delays T-12 … T-16: when time runs short, slices are cut, never the delivery date.
- **Depends on:** T-10 (E-01, E-02) for T-17a. T-11 (E-03), T-13 (E-04) and E-07 for T-17b. T-13 (E-05) for T-17c.
- **Delivery:** the thematic pull requests below, in that order (GC-3): docs → T-17a → infra → E-07 → T-17b → T-17c → tools → time record. Slice labels (T-17a … T-17c) name pull requests, not tasks: every commit ends in `(T-17)` and the HANDOFF tracks T-17 as one row, because the tools read task ids as `T-NN` (HANDOFF TR-10). E-07 is cut with the cart: if T-17b is not merged at delivery, the E-07 pull request is not merged either. After each web slice the client builds, runs in Docker and passes its tests, so a cut after any slice leaves no route, key or endpoint half-wired. The slices not merged at delivery are listed under "Not included" in the API README.
- **Web test runner:** `npm test` runs `scripts/no-literal-text.mjs`, then `ng test --watch=false`. It runs inside a container, by the owner's decision of 2026-10-15: the `test` stage of the web `Dockerfile` starts from the official Playwright image whose tag matches the pinned `playwright` version exactly (`mcr.microsoft.com/playwright:v<version>-noble`, which holds Node and Chromium), runs `npm ci`, then `npm test`; `docker build --target test .` is the web test command, so the host needs only Docker. `npm test` on the host stays possible, with a Node version the pinned Angular accepts and `npx playwright install chromium` once. The Angular version is the latest stable release when T-17a starts, pinned to an exact version in `package.json`. Specs run in headless Chromium, through the Angular CLI's Vitest runner in browser mode with Playwright, because jsdom has no layout engine and a 768 px wrapping check passes there for any layout (HANDOFF TR-11). Every spec that reaches the API uses a simulated backend (`HttpTestingController`) and asserts the exact request: method, path, headers and body. No spec opens a socket.

| # | Pull request | Repository | Commit | Estimate |
|---|--------------|------------|--------|----------|
| 1 | Specification: the full flow, E-07 and AC-49 … AC-53 | `orderflow-engine-docs` | `docs(spec): adopt full web flow (T-17)` | 1 h |
| 2 | T-17a: the assignment's screen and its container | `orderflow-engine-web` | `feat(web): add order summary screen (T-17)` | 3 h |
| 3 | The web client in the full stack and in `test-all` | `orderflow-engine-infra` | `feat(infra): serve web through api proxy (T-17)` | 0.5 h |
| 4 | Product list (E-07), right before T-17b | `orderflow-engine-api` | `feat(quoting): list tenant products (T-17)` | 1.5 h |
| 5 | T-17b: cart, confirmation, order, store picker and navigation | `orderflow-engine-web` | `feat(web): add cart and idempotent confirmation (T-17)` | 4.5 h |
| 6 | T-17c: My orders and the full-flow spec | `orderflow-engine-web` | `feat(web): add order history screen (T-17)` | 2 h |
| 7 | Figures 01 and 02 show the web client, and figure 03 gains "products" | `orderflow-engine-tools` | `docs(tools): relabel web and products in figures (T-17)` | 0.5 h |
| 8 | Time and scope record | `orderflow-engine-api` | `docs(api): record web time and scope (T-17)` | 0.5 h |

**Estimate:** about 13.5 h (11 to 16 h), against the assignment's one-hour screen. The assignment's screen alone (T-17a) takes about 3 h once its tests and its container are counted, so the plus exceeds its hour even without the extension. These are estimates, never measurements. The measured time of each pull request is recorded in the HANDOFF when it merges, and in the API README by pull request 8, in three parts: the assignment's screen, the container and test wiring, and the extension.

#### T-17a: the assignment's screen and its container, pull request 2
The order summary over E-02 and what it needs, nothing more, so its time can be measured on its own. The interceptor sends the first entry of `public/config/stores.json`; the store picker and the navigation join in T-17b. "Confirm order" joins in T-17b with E-03, and each §10.5 action that opens another screen joins with that screen.
- **Covers:** AC-42 … AC-46, AC-51 (the notice), RF-9.
- **Files (`orderflow-engine-web`):**
  - the Angular workspace (standalone components, router, `HttpClient`) with the scripts `start`, `build` and `test`
  - `src/app/core/`: `store-context.ts`, `api-headers.interceptor.ts` (`X-Tenant-Id`, `X-Customer-Id`, `Accept-Language`), `quote-api.ts` (E-02), `money.pipe.ts` (`Intl.NumberFormat` over the API string, with `app.formatLocale`), `format.ts` (rate, count and date pipes over `Intl`), `language.ts` (`?lang=`, the dictionaries), `problem.ts` (the three failure kinds: a problem, no answer, and an answer the API never sends; and which of them retry)
  - `src/app/ui/`: `ui-text`, `ui-action-bar`, `ui-product-image`, `ui-problem-notice`
  - `src/app/pages/order-summary/` (route `/quotes/:quoteId`; `order-summary-keys.ts` builds the status, origin and line-count keys, the count form through `Intl.PluralRules`)
  - `public/i18n/{es,en}.json` (`app.formatLocale`: `es-419` and `en`; `problem.network.title` and `problem.network.detail`, which `ui-problem-notice` needs for a request with no answer), `public/config/stores.json` (the seeded demo stores, A-06), `public/products/<sku>.svg` and `placeholder.svg`
  - `src/testing/`: `translations.ts`, `screen-checks.ts` (the layout measures, among them `actionBarObstructions`), `text-imports.d.ts`, `order-summary.fixture.json` (an E-02 body, with `discounts[]` sorted by origin as the API sends them) and `long-names.json`
  - `scripts/no-literal-text.mjs`
  - `proxy.conf.mjs`, `nginx/default.conf.template`, `Dockerfile` (a `test` stage that runs `npm ci` and `npm test` on the Playwright image; a build stage with Node; served by unprivileged nginx on port 8080; exact image tags), `.dockerignore`, `compose.yaml`, `.env.example` (`WEB_PORT`, `API_UPSTREAM`)
  - `README(.es).md`: Scope, How to run, Prevention at component level, listing only what this slice delivers
- **Files (`orderflow-engine-docs`, in the same change):** the web README's required headings in `governance/docs-governance.json`, for `README.md` (Scope, How to run, Prevention at component level) and for `README.es.md` (their Spanish pairs), so `check-docs` verifies AC-45 from this slice on. It merges right after the web pull request, never before it, so `check-docs` never requires a heading the web README lacks.
- **Tests:**
  - `order-summary.spec.ts` (AC-42 … AC-44): the fixture with long names, flushed through `HttpTestingController`, at 768 × 1024 and at 375 × 812: no horizontal scroll, no `ui-text` wider than its box, no `text-overflow: ellipsis`, every visible text node inside a `ui-text`, the action bar at the bottom of the viewport, all product images equal squares
  - `ui-action-bar.spec.ts` (AC-43): a simulated 300 px keyboard, fed through an injected `visualViewport` port, keeps the bar above it; every control is at least 44 × 44 px at 375 px
  - `ui-product-image.spec.ts` (AC-44): images with four different intrinsic ratios render as identical squares; a missing image falls back to the placeholder
  - `ui-problem-notice.spec.ts` (AC-51): the API's `title` and `detail` as sent, the `code`, the first 8 characters of `correlationId`, and the action it is given; a request with no answer shows `problem.network`
  - `api-headers.interceptor.spec.ts`: every `/api/` request carries the configured store and the active language
  - `money.pipe.spec.ts`: COP at scale 0 and USD at scale 2, in `es-419` and `en`; `"12345678901234567.89"` formats exactly, which a double cannot hold, so no float is involved
  - `i18n-keys.spec.ts` (AC-46): `es` and `en` have the same keys, each sets `app.formatLocale`, every key the code uses exists (also keys built from an enum value), and no key is unused
  - `scripts/no-literal-text.mjs` (RF-9, P-09): no template holds literal text; text comes from `translate` or from data
- **Done when:**
  - `docker build --target test .` passes (the specs in the container), and `check-docs` passes with the web README's required headings
  - `docker compose up --build` in `orderflow-engine-web`, with the API started from its own repository, serves the client on `WEB_PORT`; a quote created with `curl` through `http://localhost:8081/api/v1/quotes`, with the headers of the first `stores.json` entry, renders at its route `/quotes/:quoteId`
  - no response the browser receives carries a CORS header
  - the measured time of the screen and that of the container are recorded apart in the HANDOFF
- **Commit:** `feat(web): add order summary screen (T-17)`

#### The web client in the full stack, pull request 3
- **Files (`orderflow-engine-infra`):** the `web` service in `compose.yaml` (built from `../orderflow-engine-web`, depends on `api`, `API_UPSTREAM: http://api:${API_PORT:-8080}`, port `${WEB_PORT:-8081}:8080`), `WEB_PORT` in `.env.example`, the step "web tests" (`docker build --target test` in `../orderflow-engine-web`) in `scripts/test-all.{sh,ps1}`, `README(.es).md`.
- **Done when:** `docker compose up --build` starts the database, the API and the client; `http://localhost:8081` reaches the API through `/api/`; `test-all` ends with `test-all: OK (API verify, web tests, documentation check, integrity check)`, and a failing web spec fails `test-all` in PowerShell too (proven once).
- **Commit:** `feat(infra): serve web through api proxy (T-17)`

#### Product list (E-07), pull request 4
It merges right before T-17b and is cut with it.
- **Covers:** AC-53.
- **Files (`orderflow-engine-api`):** `quoting/{ListProductsUseCase, ProductCatalog, ProductView}.java`, `adapters/out/persistence/JdbcProductCatalog.java`, `adapters/in/web/{ProductController, ProductListResponse}.java`, the wiring in `config/QuotingConfig.java`, `ProductContractIT`, and in `README(.es).md` the catalog clause of "Not included": the only catalog operation is a read-only product list (E-07), which feeds the optional web cart. It lives in `quoting` because it lists exactly what E-01 can price. No `catalog` module is added, since catalog administration stays out (SPEC §1).
- **Files (`orderflow-engine-docs`, in the same change):** the `quoting` row of architecture §2 (`ListProductsUseCase`, read-only `ProductCatalog` port) and the HANDOFF State.
- **Tests (`ProductContractIT`, on Testcontainers PostgreSQL with the seeds):**
  - `co-alpha` lists its products in SKU order, `currency` `COP` once, `listPrice` as a string at scale 0, `taxRate` as a string
  - `ec-alpha` lists USD prices as strings at scale 2, trailing zeros kept, and never a `co-alpha` product (tenant scoping)
  - headers follow E-01: a missing header, or an `X-Customer-Id` that is not a UUID → 400 `VALIDATION_FAILED`; an unknown tenant → 400 `TENANT_NOT_FOUND`; an unknown customer → 404 `CUSTOMER_NOT_FOUND`
  - a request with a foreign `Origin` gets no `Access-Control-Allow-Origin` (no CORS, architecture §1.1)
- **Done when:** GC-2 is green again after T-16 (`./mvnw verify`, `check-docs`, `check-integrity`), the HANDOFF State carries that evidence, and E-07 answers as its contract card says.
- **Commit:** `feat(quoting): list tenant products (T-17)`

#### T-17b: cart, confirmation, order and store picker, pull request 5
The actions that open My orders join in T-17c.
- **Covers:** AC-42 … AC-44 (cart, order, store picker), AC-49, AC-50, AC-51.
- **Files (`orderflow-engine-web`):**
  - `src/app/core/`: `product-api.ts` (E-07), `order-api.ts` (E-03, E-04), `cart-store.ts` (one cart per store, in browser storage), `idempotency-keys.ts` (one UUID per quote, created at the first "Confirm order" and kept in browser storage); `store-context.ts` follows the picker
  - `src/app/shell/`: brand, navigation ("Shop"), the ES/EN switch and the store picker (a disclosure, never a `<select>`, which cuts long names when closed)
  - `src/app/pages/cart/` (routes `/cart` and `/`): the product grid through `ui-product-image`, a quantity stepper, cash or credit, and the action bar with the units and "Get quote" (E-01)
  - `src/app/pages/order-summary/`: "Confirm order" and "Edit cart"; the expired, confirmed and credit-short states; the §10.5 action of each problem
  - `src/app/pages/order/` (route `/orders/:orderId`): the E-04 body, with "Order confirmed" after a 201 and "This order was already recorded" after a replay
  - the new keys in `public/i18n/{es,en}.json`, among them `cart.actions.review`
- **Tests:**
  - `cart.spec.ts`: the E-07 products in SKU order, with list prices and unit counts and never a subtotal or a total; `POST /api/v1/quotes` carries only lines with a quantity above zero; a 201 opens `/quotes/:quoteId`
  - `confirm-idempotency.spec.ts` (AC-49): two taps send one request (`expectOne`); a network error and a retry send the same `Idempotency-Key` and body; a reload keeps the key; a new quote gets a new key; "Confirm order" is disabled while the request is in flight; `Idempotent-Replayed: true` leads to "This order was already recorded"; the store's cart is emptied after the 201
  - `credit-shortfall.spec.ts` (AC-50): `credit.fits=false` disables "Confirm order" and shows the shortfall, and "Pay cash" sends E-01 with `CASH` and the same lines
  - `problem-actions.spec.ts` (AC-51): every §8 code, and a request with no answer, on the cart, the summary and the order, shows the notice with exactly the §10.5 action, and each action does what the table says
  - `screen-rules.spec.ts` (AC-42 … AC-44): the cart, the order and the open store picker, with long names at 768 × 1024 and 375 × 812, pass the checks of `order-summary.spec.ts`; the action bar stays above a simulated keyboard while a quantity is typed; the order renders E-04 through the same breakdown components as the summary
  - `api-headers.interceptor.spec.ts`, added case: switching the store changes both headers
- **Done when:** `docker build --target test .` passes, and in the full stack a `co-alpha` cart goes by clicks from cart to quote to a confirmed order with its number, with no console error.
- **Commit:** `feat(web): add cart and idempotent confirmation (T-17)`

#### T-17c: My orders and the full-flow spec, pull request 6
- **Covers:** AC-42 (My orders), AC-51 (My orders), AC-52.
- **Files (`orderflow-engine-web`):** `src/app/pages/history/` (route `/orders`, E-05), "My orders" in the navigation, the actions that open My orders ("View my orders" on the order and on a confirmed quote), `src/app/flow.spec.ts`, the new keys in `public/i18n/{es,en}.json`.
- **Tests:**
  - `history-read-your-writes.spec.ts` (AC-52): E-05 is read on every visit, as `GET /api/v1/orders?customerId=` with the selected customer; items newest first, with number, date and total in the body's `currency`; after a 201 the new order number comes first, with the confirmed total; the empty state; an unknown customer (a stale `stores.json`) shows the 404 `CUSTOMER_NOT_FOUND` notice with "Go shopping"
  - `flow.spec.ts` (AC-52): cart → quote → confirm → order → My orders against the simulated backend, asserting each request, and the cart emptied after the 201
  - `screen-rules.spec.ts` and `problem-actions.spec.ts`, added cases: My orders with long names at both viewports, and its default action
- **Done when:** `docker build --target test .` and `test-all` pass, and the full stack runs the whole flow by clicks in es and en.
- **Commit:** `feat(web): add order history screen (T-17)`

#### Figures and time record, pull requests 7 and 8
- **Files (`orderflow-engine-tools`):** in `diagrams/generate_diagrams.py`, the web box, edge label and alt text of figures 01 and 02, and "products" among the controllers of figure 03. The regenerated SVGs enter `orderflow-engine-docs` with the HANDOFF update of that pull request.
- **Files (`orderflow-engine-api`):** in `README(.es).md`, under "Not included", a "Time spent" paragraph with the measured time in four rows (the take-home; the assignment's screen; the container, compose and `test-all` wiring; the extension), listing the slices not merged if T-17 is cut; the "CORS configuration" row of "Not included" (not needed: the web client's nginx forwards `/api/`), placed before the SonarQube row; and the `orderflow-engine-web` row of "Repositories".
- **Done when:** the figures match architecture §1 and §2, both API READMEs carry measured times only, and `test-all` passes.
- **Commits:** `docs(tools): relabel web and products in figures (T-17)`, `docs(api): record web time and scope (T-17)`

### T-19: Fixes after the T-17 preview
The owner walked the full stack after T-17 and ordered these fixes on 2026-10-16. Each is one thematic pull request, in this order (GC-3): docs → api seeds → api quantities → web → infra.
- **Covers:** AC-14, AC-46, AC-52, BR-00, SPEC §9, §10.1, §10.3, §11.
- **Seed names (`orderflow-engine-api`):** `db/changelog/2026/2026-10-16-001-seed-names-in-spanish.yaml` renames the seeded products and customers to the tenants' language, with a rollback to the original names (`LiquibaseRollbackIT` proves it): BEV-COLA-1500 "Gaseosa cola 1,5 l", BEV-COLA-350 "Gaseosa cola 350 ml", BEV-WATER-600 "Agua 600 ml", SNK-CHIPS-45 "Papas fritas 45 g", SNK-PLANTAIN-45 "Chifles 45 g"; customers "Tienda Norte" and "Tienda Sur" (`co-alpha`), "Tienda Quito" and "Tienda Guayaquil" (`ec-alpha`). An applied changeset is never edited. Every test and fixture that reads a seeded name follows. Commit: `feat(db): seed catalog in the tenants' language (T-19)`.
- **Quantities (`orderflow-engine-api`):** BR-00 caps a line quantity at 1,000,000 (also after merging) and a derived gift quantity at the same bound, both 400 `INVALID_QUANTITY`; the gift quantity is derived with overflow-checked arithmetic. Tests: the bound itself and one above it through E-01; a gift rule whose derived quantity passes the bound answers 400, never 500. Commit: `feat(quoting): cap line and gift quantities (T-19)`.
- **Web (`orderflow-engine-web`):** the currency code next to the amounts that need it (SPEC §10.3): "Prices in {currency}" above the cart grid, and the code after each action-bar total and each My orders total; the ES/EN choice kept in browser storage and restored on load, `?lang=` winning and replacing it (SPEC §9); `public/config/stores.json` and the test fixtures use the Spanish seed names; `restart: unless-stopped` in `compose.yaml`. Tests: `language.spec.ts` (reload keeps the choice, `?lang=` replaces it, storage unavailable falls back to `es`), the currency code in `cart.spec.ts` and `screen-rules.spec.ts` for COP and USD. Commit: `feat(web): show currency codes and keep language (T-19)`.
- **Infra (`orderflow-engine-infra`):** `restart: unless-stopped` on every service of the full-stack compose; an end-to-end step in `test-all.{sh,ps1}`, after the web tests: it starts the full stack under its own compose project and ports, runs a Playwright spec on the official image whose tag matches the web's pinned `playwright` against it (cart → quote → confirm → order → My orders, in es and en, with the order number read back from E-05 through the client's origin), and stops the stack with `down -v`, also when the spec fails. The final line becomes `test-all: OK (API verify, web tests, end-to-end, documentation check, integrity check)`. Commits: `build(infra): restart the stack with docker (T-19)`, `test(infra): run the full flow against the stack (T-19)`.
- **Done when:** each repository's own check passes (`./mvnw verify`, `docker build --target test .`), `test-all` ends with the line above, and the full stack shows Spanish names, currency codes and a remembered language by clicks.
- **Commit:** `feat(db): seed catalog in the tenants' language (T-19)`, the first of the five commits named in the bullets above.

### T-20: Seed Guatemala and Peru
The assignment names four countries (Colombia, Ecuador, Guatemala, Peru); the seeds held two. The owner ordered the other two on 2026-10-16, as data only, so they also prove that no country data lives in the code (P-07).
- **Covers:** SPEC §5, data model §1 (seed data), P-07.
- **API (`orderflow-engine-api`):** `db/changelog/2026/2026-10-16-002-seed-gt-and-pe.yaml`, included after `2026-10-16-001`, with a rollback that removes everything it adds: currencies GTQ and PEN (scale 2), countries GT and PE, tenants `gt-alpha` and `pe-alpha`, their families, products, tax rates, levers, customers and credit accounts exactly as data model §1 lists them, names in Spanish, and anything per country the order numbers need. No Java change. Tests: `ProductContractIT` lists GTQ and PEN at scale 2 for the new tenants; a quote through E-01 for each tenant whose totals are computed by hand from the seed values (the GT tier and the PE combo, with their taxes); `LiquibaseRollbackIT` covers the new changeset; the README's seeded tenants sentence names the four. Commit: `feat(db): seed Guatemala and Peru tenants (T-20)`.
- **Web (`orderflow-engine-web`):** `public/config/stores.json` lists the four new stores, grouped by tenant; a spec checks that the picker shows the four tenants and that a GTQ and a PEN catalog format with their currency code. Commit: `feat(web): add Guatemala and Peru stores (T-20)`.
- **Done when:** `./mvnw verify` and `docker build --target test .` pass, and the full stack quotes and confirms an order in GTQ and in PEN by clicks.
- **Commit:** `feat(db): seed Guatemala and Peru tenants (T-20)`, the first of the two commits named above.

### T-21: Delivery documents and public page
The owner ordered the delivery set on 2026-10-17: five documents for readers outside the team, each in English and Spanish, their Word, PDF and PowerPoint versions, and a public page. They describe the delivered system and add no behavior: no code, contract or seed changes, and every number they cite is one HANDOFF §3 holds.
- **Covers:** SPEC §15 (delivery set and `orderflow-engine-page`), GC-6, GC-9, DR-1, DR-2.
- **Files (`orderflow-engine-docs`):**
  - registration, first (GC-9): SPEC §15, the document map in `README(.es).md`, `governance/docs-governance.json` (the `delivery/` location and its language pairs; `orderflow-engine-page` as a scan root with the integrity rules of a content-only repository), this task, and the HANDOFF
  - `delivery/{user-manual, technical-manual, architecture-report, solution-walkthrough, presentation}(.es).md`: each with the front matter of [documentation-rules](../governance/documentation-rules.md) §4 and, under the H1, the one question it answers; both files of a pair have the same number of headings
  - `delivery/screenshots/{en,es}/`: `01-store-picker`, `02-catalog`, `03-cart`, `04-quote-summary`, `05-quote-levers`, `06-order-confirmed`, `07-my-orders`, `08-error-message` and `09-mobile-summary`, as PNG, captured from the full stack
  - `delivery/figures/`: one PNG per SVG in `diagrams/` (`01-runtime-topology` … `09-outbox-lifecycle`), with the same stem
  - `deliverables/delivery/`: the user manual, technical manual, architecture report and solution walkthrough as `.docx` and `.pdf` with the print template (documentation-rules §3a), and `presentation.pptx` and `presentation.es.pptx`
- **Files (`orderflow-engine-page`):** a new folder, a static site for GitHub Pages, Spanish by default with an English switch: `README(.es).md`, `LICENSE`, and the `.gitignore` and `.gitattributes` that [repository-integrity](../governance/repository-integrity.md) requires. No Docker files and no `.env`.
- **Steps:**
  - register the delivery set before writing it (GC-9)
  - capture the screenshots in Spanish and in English, and export each SVG figure to PNG
  - write each document in English and in Spanish, stating only what the repositories show
  - render the four documents to Word and PDF and the presentation to PowerPoint, in both languages
  - build the public page
  - run both checks and record the state in the HANDOFF (GC-8)
- **Verify:** `node orderflow-engine-tools/docs-check/check-docs.mjs` → OK, and `node orderflow-engine-tools/integrity/check-integrity.mjs` → OK with `orderflow-engine-page` among the repositories.
- **Done when:** every delivery document is indexed and paired with the same number of headings; every image a document cites exists; the Word, PDF and PowerPoint files exist in both languages; the page opens in Spanish and switches to English; both checks pass.
- **Commit:** `docs: add delivery documents and public page (T-21)` in this repository, one pull request with the registration and the documents; `feat(tools): render delivery documents and decks (T-21)` in `orderflow-engine-tools`; and `feat(page): add delivery portal (T-21)` in `orderflow-engine-page`, after its bootstrap commit.
