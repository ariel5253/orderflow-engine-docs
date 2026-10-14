---
id: SPEC
title: Orderflow Engine — Specification
status: accepted
lang: en
pair: none
last-reviewed: 2026-10-14
---

# Orderflow Engine — Specification

## 0. Executive summary

Orderflow Engine prices and confirms orders that shopkeepers place with beverage distributors. It runs on one multi-tenant platform spanning several countries and brands. It guarantees four things:

| # | Guarantee | Mechanism |
|---|-----------|-----------|
| G-1 | The **quoted total is the confirmed total**, even if prices or levers change in between | A quote is sealed: its lines and discount components are insert-only. The order references the quote and stores no amount, so every breakdown figure exists exactly once (§6) |
| G-2 | **No floating point for money.** Scale, rounding rule and rounding points are explicit | `BigDecimal` value object. Scale and rounding come from country data. Rounding is per line (§4) |
| G-3 | **A 201 means the order exists and is queryable.** Retries never duplicate orders or consume credit twice | Order, credit, idempotency key and outbox are written in one ACID transaction, and the reply is sent after commit (§7) |
| G-4 | **External effects (push, ERP) are never lost and never applied twice** | Never lost: transactional outbox and a leased relay with retries. Never applied twice: every event carries an `event_id` that consumers deduplicate on (§7) |

Stack: Java 21 · Spring Boot · PostgreSQL · Liquibase · JUnit 5 + Testcontainers · Angular.
Single command for code and tests: `./mvnw verify` in `orderflow-engine-api`, which needs Docker running for Testcontainers. The service runs with `docker compose up --build` in the same repository.

---

## 1. Scope

The scope is exactly the assignment: A1 (pricing engine), A2 (order confirmation), A3 (one ADR), the delivery items, and the optional order-summary screen. Everything else is out.

| In scope | Out of scope (reason) |
|----------|-----------------------|
| `POST /api/v1/quotes`, `GET /api/v1/quotes/{quoteId}` (A1 endpoint) | Authentication and authorization. Tenant and customer come from headers (A-06) |
| `POST /api/v1/orders` with `Idempotency-Key` (A2) | Catalog and lever administration. Data is seeded by migrations |
| `GET /api/v1/orders/{orderId}`, `GET /api/v1/orders?customerId=` (the "history" of A2) | Real push and ERP integrations. Fakes sit behind ports |
| Pricing engine: tiers, combos, gifts, taxes; credit check | Cart persistence. The cart travels in the quote request |
| Outbox relay to push and ERP; retention housekeeping | Payment capture, cancellation, credit release (A-09) |
| i18n (es default, en) for every end-user text | Quote revocation. It is recorded as the ADR's risk and mitigation |
| Angular order-summary screen (at most 1 hour) | Deployment, cloud infrastructure, OpenAPI file. The contract is [api-contract.md](api-contract.md) |
| Local run with Docker: each executable repository has its own Docker files and `.env.example` (§14.3, §15) | |
| | Mobile app. `orderflow-engine-app` is reserved and not built |

## 2. Glossary

| Term | Definition |
|------|------------|
| Tenant | A (country, brand) pair. It owns prices, levers, tax rates and customers |
| Country config | Currency, decimal scale, rounding mode, gift tax policy, quote TTL, default locale |
| SKU / Family | A sellable product, and the group of SKUs it belongs to (e.g. "colas") |
| Cart | A list of `{sku, qty}` plus the payment method, sent by the client |
| Lever | A commercial rule that changes the price: tier discount, combo or gift. Taxes and credit are evaluated alongside |
| Quote | An insert-only record of the priced cart: lines, discount components, taxes. Valid until `expiresAt` |
| Order | The confirmation of a quote: an order number that points at the quote. It stores no amount |
| Credit account | A customer's credit `limit` and `consumed` amount, plus an append-only ledger |
| Outbox event | A row written in the order transaction that records a pending external effect |

## 3. Business rules

Each rule has a stable ID. "Verified by" names the test that proves it (see §14). The principles behind these rules are in [constitution.md](constitution.md).

### BR-00 Cart normalization
- Duplicate SKUs are merged by summing their quantities. Lines are sorted by SKU code. Pricing runs on the normalized cart, so the result never depends on the order the client sent.
- Rejected:
  - `qty ≤ 0` or a non-integer quantity → 400 `INVALID_QUANTITY`
  - an empty cart → 400 `CART_EMPTY`
  - a SKU that is unknown, or that belongs to another tenant → 422 `UNKNOWN_SKU`
- Verified by: `CartNormalizationTest`.

### BR-01 Order of application

| Step | What | Why here |
|------|------|----------|
| 1 | Gift eligibility, evaluated on **paid** quantities and gross subtotal | A gift is triggered by what the customer buys. It never changes what they pay for product |
| 2 | Combos | A combo is a negotiated bundle price. It consumes units first, so that those units cannot also receive a tier discount (no double dipping) |
| 3 | Tier discounts on units **not consumed** by a combo | Tiers are the generic volume incentive. They apply only to what the bundle did not price |
| 4 | Line net = gross − discount components | Commercial discounts must be settled before tax |
| 5 | Taxable base | The assignment requires tax on the base **after** commercial discounts |
| 6 | Tax per line | Tax rates are per SKU. §4 explains the rounding |
| 7 | Total | It needs every previous step |
| 8 | Credit evaluation (application layer) | It needs the final total. It is not a price lever, so it stays outside the pure engine |

Verified by: GT-01…GT-06 and `PricingPipelineOrderTest`.

### BR-02 Tier discount
- A tier rule targets a SKU or a family and defines brackets `{from, percent}`. `from` is inclusive and units are counted from 1.
- **"Cumulative by brackets" is read as marginal (A-01).** Each bracket's percentage applies only to the units that fall inside that bracket. Percentages never stack, so a unit in bracket *k* receives only the percentage of *k*.
- The discount for a line is computed exactly and **rounded once per line** with the country rounding mode, never once per bracket segment.
- A SKU that has its own SKU-scoped rule is removed from any family-scoped rule. It is neither counted nor discounted there (A-02).
- Family units are assigned to brackets in normalized SKU order (A-03).
- Units consumed by a combo, and gift units, do not count toward thresholds and receive no tier discount (A-04).
- Worked example (GT-01): tenant CO, COP scale 0, `HALF_UP`. The SKU is `BEV-COLA-350` at 2,500 with 19% tax. The brackets are 1–9 → 0%, 10–19 → 5%, 20+ → 10%. Quantity: 25.

  | Item | Amount |
  |------|-------:|
  | Gross 25 × 2,500 | 62,500 |
  | Bracket 10–19: 10 × 2,500 × 5% | 1,250 |
  | Bracket 20–25: 6 × 2,500 × 10% | 1,500 |
  | Tier discount (`TIER:<ruleId>`) | 2,750 |
  | Net = taxable base | 59,750 |
  | Tax 19% = 11,352.50 → HALF_UP | 11,353 |
  | **Total** | **71,103** |

### BR-03 Combo
- A combo defines a required set `{sku: minQty}` and a `comboPrice` per application.
- `applications = floor(min(qty_i / minQty_i))`. The consumed units of each SKU are `applications × minQty_i`.
- `comboDiscount = listValue(consumed units) − comboPrice × applications`.
- **If `comboPrice × applications ≥ listValue(consumed units)`, the combo does not apply and consumes nothing.**
- When several combos could apply, they are evaluated in combo-id order, each on the units left by the previous ones.
- **Presentation (golden test 2).** The order-level `discounts[]` holds **exactly one entry per applied combo**, with `origin: COMBO:<id>`. The products that trigger the combo never appear as loose discounts: consumed units never produce `TIER:*` entries. Each participating line exposes `comboAllocations[{comboId, amount}]`. This field exists **only to derive the line's taxable base. It is not a discount entry** (A-07).
- **Allocation.** The combo discount is split across the participating lines in proportion to the gross value of their consumed units, using the largest-remainder method:
  1. Each share is truncated to currency scale.
  2. The leftover minor units go to the largest fractional remainders.
  3. Ties go by SKU-code order.

  The allocations always sum exactly to the combo discount.
- Worked example (GT-04): tenant CO. `BEV-COLA-350` costs 2,500 with 19% tax, and `SNK-CHIPS-45` costs 1,800 with 5% tax. The combo `{COLA ×2, CHIPS ×1}` costs 5,000. The cart is COLA 4, CHIPS 2.

  | Item | COLA | CHIPS | Order |
  |------|-----:|------:|------:|
  | Applications | | | 2 |
  | Consumed list value | 10,000 | 3,600 | 13,600 |
  | Combo discount 13,600 − 10,000 | | | 3,600 |
  | Exact share | 2,647.06 | 952.94 | |
  | Allocated (largest remainder) | 2,647 | 953 | 3,600 |
  | Net = taxable base | 7,353 | 2,647 | 10,000 |
  | Tax | 1,397 (19%) | 132 (5%) | 1,529 |
  | **Total** | | | **11,529** |

### BR-04 Gift
- **Condition.** A minimum paid quantity of a SKU or family, or a minimum gross subtotal computed **before discounts** (A-10). A gift rule may repeat for each full multiple of its condition.
- **Effect.** A separate gift line `{sku, qty, isGift: true, origin: GIFT:<id>}` with `unitPrice = 0`, `gross = 0` and `net = 0`. It never merges with a paid line of the same SKU. Gift units are excluded from tiers and combos.
- **Reference value** = `qty × list price` of the gift SKU.
- **Tax implication.** The country's `giftTaxPolicy` decides it:

  | Policy | Gift taxable base | Gift tax |
  |--------|------------------:|---------:|
  | `EXEMPT` | 0 | 0 |
  | `TAXED` | reference value | gift SKU rate × reference value, rounded per line, and paid by the customer |

### BR-05 Tax and total identity
- Each SKU has its rate as data (`products.tax_rate`, `NUMERIC(7,6)`), captured on every quote line. Prices are **tax-exclusive**.
- `line.tax = round(line.taxableBase × line.taxRate)` with the country scale and rounding mode.
- Identities, asserted by **every** golden test:
  - `grossSubtotal = Σ paid line.gross`
  - `Σ discounts[].amount = Σ line tier discounts + Σ line comboAllocations`
  - `line.net = line.gross − Σ line.discounts − Σ line.comboAllocations`
  - `taxableBase = Σ paid line.net + Σ gift line.taxableBase`
  - `taxes = Σ line.tax`
  - **`total = Σ line.net + taxes`**

### BR-06 Credit
- `paymentMethod` (`CASH` | `CREDIT`) is part of the quote request.
- A quote paid with `CREDIT` returns `credit {fits, available, shortfall}`:
  - `available = limit − consumed`
  - `fits = total ≤ available`, so a total equal to available fits
  - `shortfall = max(0, total − available)`

  At quote time this is **advisory**. It is not part of the sealed price, and nothing is reserved (A-08).
- A customer without a credit account who requests `CREDIT` → 422 `CREDIT_NOT_ENABLED`.
- At confirmation, inside the order transaction (§7):
  1. Run `UPDATE credit_accounts SET consumed = consumed + :total WHERE tenant_id = :tenant AND customer_id = :customer AND consumed + :total <= credit_limit`.
  2. If that updates 0 rows → 409 `CREDIT_LIMIT_EXCEEDED`. The transaction rolls back, so no order row and no ledger row remain.
  3. Otherwise, insert the ledger row `credit_ledger(order_id, amount)` right after the order row, because the ledger references the order.
- **The limit reflects consumption.** After a credit 201, `available` drops by exactly the order total, exactly once, and `consumed = Σ ledger.amount` always holds.

### BR-07 Quote lifecycle

```
OPEN ──confirm──▶ CONFIRMED   (an order exists for the quote)
  └ ──ttl──────▶ EXPIRED     (now > expiresAt, and no order)
```

- The state is derived, never stored, so a quote row is never updated. `expiresAt = createdAt + country.quoteTtl`. The default is 15 minutes (A-05).
- **Catalog and lever changes made after a quote is created never alter an OPEN quote.** The sealed price is honored until `expiresAt` (GT-06).
- A quote belongs to its (tenant, customer). Any other caller receives 404 `QUOTE_NOT_FOUND`, so the quote's existence is not revealed.

## 4. Money and rounding policy

| Aspect | Decision |
|--------|----------|
| Type | `Money(BigDecimal amount, CurrencyPolicy policy)`, an immutable value object. `float` and `double` are banned from the domain by an ArchUnit rule |
| Scale and rounding mode | Read from the `countries` table. COP has scale 0 and USD has scale 2. The default mode is `HALF_UP`. They never come from `java.util.Currency`, which reports COP with scale 2 |
| Storage | Money is `NUMERIC(19,4)` and rates are `NUMERIC(7,6)`. A price whose scale exceeds the country scale is **rejected** when loaded. Values are read back at currency scale |
| Comparison | Always `compareTo`, never `equals`, because `equals` also compares scale |
| API | Money is a JSON **string** with the scale applied, plus an ISO currency code. It is never a JSON number |

**Rounding points: rounding happens per line only.**

| # | Point | Rule |
|---|-------|------|
| RP-1 | Unit price × qty | Exact, because prices are already at currency scale |
| RP-2 | Each discount component per line (tier, combo allocation) | Rounded to currency scale. Combo allocation uses largest remainder, so the sum is exact. A component that rounds to zero is not emitted |
| RP-3 | `net = gross − Σ components` | Exact |
| RP-4 | Tax per line | Rounded to currency scale |
| RP-5 | Order totals | Sums of rounded line values. **Never re-rounded**, so the total always equals the sum of its lines |

**Why per line.** Electronic invoices carry tax per line, so per-line rounding keeps the order consistent with the invoice it produces. Two alternatives were rejected:
- Rounding once per tax-rate group makes the line values disagree with the totals.
- Rounding only the final total hides cent drift and cannot be audited line by line.

## 5. Multi-country and multi-tenant

- `X-Tenant-Id` resolves the tenant, and the tenant resolves its country config. An unknown tenant → 400 `TENANT_NOT_FOUND`.
- Prices, levers, tax rates, customers and credit accounts are tenant-scoped data. No currency, scale, rate or rule is hard-coded in the logic.
- Seeded tenants:
  - `co-alpha`: Colombia, COP, scale 0
  - `ec-alpha`: Ecuador, USD, scale 2

  Two countries with different scales are enough to prove that nothing is hard-coded. The `brand` column supports more brands per country without code changes.

## 6. Quote equals confirmation

**Quote** (`POST /api/v1/quotes`):
1. Normalize the cart (BR-00).
2. Resolve the snapshot at time T: unit prices, active levers, tax rates, country policy and the customer's credit position.
3. Run the pure, deterministic engine on that snapshot.
4. Insert the quote, its lines and their discount components. Every line captures `unit_price` and `tax_rate` as they were at T, and stores the rounded `taxable_base` and `tax`. These rows are **insert-only** ([data-model](data-model.md) §4).

**Confirm** (`POST /api/v1/orders {quoteId}`) **writes no amount.** The order row holds only its number and a reference to the quote. Its breakdown is rendered from the quote lines, so there is one copy of every figure, and the confirmed total is the quoted total by construction. A price or lever change between the two moments cannot reach the order, because nothing is recomputed or copied.

**Determinism.** The engine is never re-run at confirmation. If it were, a deployment that changes the engine would break every open quote. The golden tests pin the engine's output instead, so a change in behavior fails the build, not a customer.

**Trade-off.** A price change applies only to new quotes. Open quotes keep the old price until they expire (TTL 15 minutes). A wrong price is therefore honored for at most one TTL. This is the risk recorded in the ADR, together with its mitigation.

## 7. Order confirmation: idempotency and consistency

### 7.1 Transaction
Locks are always taken in the same order, **idempotency key → quote → credit account**, so concurrent confirmations cannot deadlock. The isolation level is READ COMMITTED with explicit row locks and a `lock_timeout`.

| # | Step | Guarantee |
|---|------|-----------|
| 1 | `INSERT INTO idempotency_keys (tenant_id, customer_id, idem_key, request_hash) … ON CONFLICT DO NOTHING`, then read the row back | One owner per key. A concurrent duplicate blocks on the uncommitted unique entry |
| 2 | `SELECT … FROM quotes WHERE id = ? FOR UPDATE`, then check the owner, that no order exists for it and that it has not expired | One confirmation per quote |
| 3 | Conditional credit update (BR-06), only when paying with `CREDIT` | Credit is never consumed over the limit |
| 4 | Insert the order row (id, quote reference, customer, number from the country sequence, `confirmed_at`), then the credit ledger row when paying with `CREDIT`. No breakdown amount is written. `orders.quote_id` is `UNIQUE` | No order without its quote. No second order for the same quote. Credit consumed at most once per order |
| 5 | Insert one outbox row per destination (`PUSH`, `ERP`) for the order. Its id is the `event_id` | Effects are recorded atomically with the order |
| 6 | Store the order id and the 201 body on the idempotency row | Replays return the same answer |
| 7 | **COMMIT**, then serialize the 201 response | See I-201 |

`request_hash = SHA-256(method + path + customerId + canonical JSON body)`.

### 7.2 The 201 invariant (I-201)
**The 201 body is serialized only after the COMMIT of the transaction that inserted the `orders` row and drew its number. `GET /api/v1/orders/{orderId}` and the history read the same primary tables under READ COMMITTED. Therefore no order number is ever returned without a committed, readable row.**

### 7.3 Retries and replays

| Situation | Response |
|-----------|----------|
| Same key, same `request_hash`, already completed | The stored response, byte-identical, with `Idempotent-Replayed: true` |
| Same key, different `request_hash` | 422 `IDEMPOTENCY_KEY_REUSED` |
| Same key while the first request is still in flight | Waits up to `lock_timeout`, then replays. If the timeout passes first → 409 `IDEMPOTENCY_KEY_IN_PROGRESS` |
| `Idempotency-Key` missing, or not matching `^[A-Za-z0-9_-]{8,64}$` | 400 `IDEMPOTENCY_KEY_MISSING` / `IDEMPOTENCY_KEY_INVALID` |
| The request fails (validation, expired quote, credit exceeded…) | **Not cached.** The transaction rolls back, the key is released, and the client may retry after fixing the cause |
| A new key on an already-confirmed quote (e.g. app reinstall) | 409 `QUOTE_ALREADY_CONFIRMED`, carrying the existing order number |

These semantics follow the IETF draft *The Idempotency-Key HTTP Header Field*.

### 7.4 External effects (outbox relay)
1. **Claim.** In a short transaction, the relay leases due `PENDING` rows with `FOR UPDATE SKIP LOCKED` and sets `locked_until = now + lease`. A leased row is not claimed again until its lease expires. Each order has one event per destination, so there is nothing to reorder.
2. **Send.** The relay calls `PushNotificationPort` or `ErpPublisherPort` **outside any transaction**, passing `event_id` as the downstream idempotency key.
3. **Mark.** On success the row becomes `SENT`. On a retryable failure, `attempts` increases and `next_attempt_at` is set by exponential backoff with full jitter. After `maxAttempts`, or on a non-retryable error such as an ERP 4xx, the row becomes `DEAD`. It is logged and counted in the `outbox_dead` gauge.
4. **Never lost.** If the process crashes between send and mark, the lease expires and the event is sent again.
5. **Never applied twice.** Consumers deduplicate on `event_id`:
   - push uses it as the collapse id
   - the ERP upserts by order number
6. **External system down.** The order stays confirmed. The effect waits in the outbox and is delivered when the system recovers, and `outbox_pending` shows the backlog.

### 7.5 Consistency model

| Scope | Model | Sacrificed |
|-------|-------|------------|
| Order, credit, idempotency, outbox | Strong: one ACID transaction in one database (CP) | Availability while the database is down. Confirmations of the same customer serialize on the credit row. The write path depends on a single database |
| Push and ERP | Eventual, at-least-once delivery, deduplicated by consumers | Push and ERP lag behind the confirmation by the poll interval plus retries |

### 7.6 How "confirmed but missing" is made impossible

The assignment reports that, in three releases, orders were confirmed but later missing.

| Plausible root cause | Design answer | Verified by |
|---------------------|---------------|-------------|
| Response sent before the commit | The reply is serialized after commit (I-201) | `ConfirmOrderCommitOrderingIT` |
| `@Transactional` bypassed by self-invocation | The transaction boundary is a separate use-case bean, enforced by an ArchUnit rule | `TransactionBoundaryArchTest`, `ConfirmOrderFailureInjectionIT` |
| History read from a replica or the ERP | History reads the primary tables synchronously | `OrderReadYourWritesIT` |
| Order written asynchronously after replying | The order row is written in the request transaction. Only the effects are asynchronous | `ConfirmOrderFailureInjectionIT` |
| Lost update or duplicate on a double tap | Unique idempotency key, `quote_id UNIQUE`, conditional credit update | `ConfirmOrderConcurrencyIT` |

## 8. API contract

This section fixes the rules. The operation cards (E-01…E-06), with requests, responses and examples, are in [api-contract.md](api-contract.md), which is the contract. The tables behind it are in [data-model.md](data-model.md).

- Required headers: `X-Tenant-Id` and `X-Customer-Id`. `Idempotency-Key` is also required on `POST /orders`.
- Optional header: `Accept-Language`.
- Errors use RFC 9457 `application/problem+json` with these fields:
  - `type`
  - a localized `title` and `detail`
  - `status`
  - a stable `code`
  - `correlationId`

  Internals such as stack traces and SQL are never exposed.

| Status | Meaning | Codes |
|--------|---------|-------|
| 400 | Malformed request | `VALIDATION_FAILED`, `INVALID_QUANTITY`, `CART_EMPTY`, `TENANT_NOT_FOUND`, `IDEMPOTENCY_KEY_MISSING`, `IDEMPOTENCY_KEY_INVALID` |
| 404 | Not found, or not visible to the caller | `QUOTE_NOT_FOUND`, `ORDER_NOT_FOUND`, `CUSTOMER_NOT_FOUND` |
| 409 | Conflict with the current state | `QUOTE_EXPIRED`, `QUOTE_ALREADY_CONFIRMED`, `CREDIT_LIMIT_EXCEEDED`, `IDEMPOTENCY_KEY_IN_PROGRESS` |
| 422 | Valid request that breaks a business rule | `UNKNOWN_SKU`, `CREDIT_NOT_ENABLED`, `IDEMPOTENCY_KEY_REUSED` |
| 500 | Unexpected. Logged with the correlation id | `INTERNAL_ERROR` |

**Breakdown shape** (the same for quote and order). It covers what the assignment asks for: per line and totals; gross subtotal, discounts with their origin, gifts with quantity, taxable base, taxes and total.

| Level | Fields |
|-------|--------|
| Line | `sku`, `description`, `qty`, `isGift`, `origin` (gift only, `GIFT:<id>`), `unitPrice`, `gross`, `discounts[{origin, amount}]`, `comboAllocations[{comboId, amount}]`, `net`, `referenceValue` (gift only), `taxableBase`, `taxRate`, `tax` |
| Totals | `currency`, `grossSubtotal`, `discounts[{origin, amount}]`, `gifts[{sku, qty, origin}]`, `taxableBase`, `taxes`, `total` |
| Credit | `credit{fits, available, shortfall}`, present when `paymentMethod = CREDIT` |

## 9. Internationalization

| Side | Mechanism | Files | Default and switching |
|------|-----------|-------|-----------------------|
| API | Spring `MessageSource`, UTF-8 | `src/main/resources/i18n/messages.properties` (Spanish base), `messages_es.properties`, `messages_en.properties` | `Accept-Language` picks the locale. Missing or unsupported → `app.i18n.default-locale=es` |
| Web | `@ngx-translate/core`, runtime JSON | `public/i18n/es.json`, `public/i18n/en.json` | Default `es`. The `?lang=en` query parameter switches language without touching code |

- **Domain and application layers carry only error codes and arguments.** The web adapter resolves the text. Logs are in English and never translated.
- Keys are namespaced by feature, e.g. `error.QUOTE_EXPIRED.title` and `orderSummary.totals.taxableBase`.
- Money is formatted with `Intl.NumberFormat`, using the API currency and the active locale.
- **Adding a language** means adding one file per side. No code changes.
- **Guards:**
  - `check-docs` fails when two locales do not have the same key set (API and web)
  - `ErrorCatalogI18nTest` fails when an error code lacks a message in any bundle
- **Rejected:** Angular built-in i18n. It builds one bundle per locale and cannot switch at runtime.

## 10. Frontend (optional plus, at most 1 hour)

One standalone Angular page, `order-summary`. It renders a fixed JSON fixture with the exact shape of `GET /api/v1/quotes/{quoteId}`: lines, discounts with origin, gifts with quantity, taxes and total.

| Defect the assignment names | Component-level answer |
|-----------------------------|------------------------|
| Truncated text | `ui-text` always wraps (`overflow-wrap: anywhere`, `min-width: 0` in flex and grid), never uses ellipsis, and grows vertically. Screens cannot print text another way |
| Controls hidden by the system bar or the keyboard | `ui-action-bar` pads with `env(safe-area-inset-bottom)` (`viewport-fit=cover`) and follows `visualViewport` to stay above the keyboard. Layout uses `100dvh` |
| Grid images scaled inconsistently | `ui-product-image` enforces one `aspect-ratio` with `object-fit: contain`, and is the only way to render a product image in a grid |

One component test renders very long product names at a 768 px viewport and checks the three rules. The real time spent is recorded in the API README under "Not included".

## 11. Testing strategy

| Layer | Scope | Tooling |
|-------|-------|---------|
| Unit | `Money`, normalization, each lever, the pipeline | JUnit 5, AssertJ |
| Golden | Full breakdowns for fixed inputs, with a fixed clock and fixed ids | JSON fixtures under `src/test/resources/golden/<case>/` (`request.json`, `snapshot.json`, `expected.json`). Every golden test is tagged `@Tag("golden")` |
| Integration | Persistence, transactions, concurrency, outbox | Testcontainers PostgreSQL, Spring Boot test |
| Contract | HTTP status, `problem+json` shape, i18n | MockMvc |
| Architecture | Pure domain, no float/double, transaction boundary placement | ArchUnit |
| Web | Component rules with long names | Angular test runner |

**The six golden tests the assignment requires.** Each one also asserts the BR-05 identities.

| ID | Assignment case | Key assertion |
|----|-----------------|---------------|
| GT-01 | An order with a discount in two brackets | Marginal brackets with the worked numbers of BR-02 |
| GT-02 | A combo whose triggering products must not appear as a loose discount | Exactly one `COMBO:<id>` entry and zero `TIER:*` entries for the consumed units, even with a tier rule seeded on the same SKUs |
| GT-03 | An order with a gift | A gift line with its quantity and `GIFT:<id>` origin. Paid lines are unchanged |
| GT-04 | A combo with taxes | Tax is computed on the allocated base and is included in `total` (BR-03 example) |
| GT-05 | A credit order that exceeds the limit | Runs through the quote use case with fixture-backed ports (`GoldenQuoteTest`), because credit is evaluated outside the pure engine. Asserts `fits=false`, `available` and `shortfall`. Companion integration test: confirming it → 409, with no order and no ledger row |
| GT-06 | Quote regression: same cart, quote then confirm, the totals match | Between quote and confirm, a price changes and a lever is deactivated. The confirmed total equals the quoted total, **and** a fresh quote differs, which proves the test can fail |

**Extra goldens:**
- rounding edges (`.5` cases)
- CO scale 0 versus EC scale 2
- both gift tax policies

The unit test `TierPrecedenceTest` proves that a SKU rule beats a family rule.

**Integration tests that demonstrate the A2 invariants:**
- a replay returns a byte-identical response
- N threads with one key produce exactly one order and one credit consumption
- two over-limit confirmations run concurrently, and exactly one succeeds
- a failure injected before commit persists nothing and releases the key
- after a crash after commit, a retry returns the same order number
- a 201 is followed by a successful GET
- the limit reflects consumption
- the outbox lease, retry, `DEAD` and recovery after an outage

**Quality gates:**
- JaCoCo ≥ 90% line and branch coverage on the `pricing` package. It fails `verify`.
- PIT mutation testing on `pricing.domain`, in the optional profile `./mvnw -Pmutation verify` (threshold ≥ 80%). It runs before delivery, and the score is recorded in the HANDOFF.

**Commands:**
- `./mvnw verify` in `orderflow-engine-api` is the single command the assignment asks for.
- `orderflow-engine-infra/scripts/test-all` also runs the web tests and the documentation check.

## 12. Architecture

Topology, flows and figures are in [architecture.md](architecture.md). Technical decisions and their risks are in [plan.md](plan.md).

- A hexagonal, modular monolith. Base package: `com.orderflow.engine`.

  | Module | Contents | Depends on |
  |--------|----------|------------|
  | `pricing.domain` | Pure engine, `Money`, levers, breakdown | JDK only |
  | `quoting` | Quote use cases and ports | `pricing.domain` |
  | `ordering` | Confirm and history use cases and ports | `quoting` ports, `pricing.domain` |
  | `outbox` | Relay, housekeeping, ports | — |
  | `adapters.in.web` | Controllers, DTOs, `problem+json`, locale resolution | application ports |
  | `adapters.out.persistence` | `JdbcClient` repositories, explicit SQL and locks | application ports |
  | `adapters.out.push`, `adapters.out.erp` | Fake gateways that deduplicate by `event_id` and can be told to fail | outbox ports |
  | `config` | Wiring and typed configuration | everything |

- `@Transactional` appears only on application use cases. No remote call is made inside a transaction.
- Liquibase changelogs live in `db/changelog/YYYY/YYYY-MM-DD-NNN-verb-object.yaml`, each with a rollback.
- Observability:
  - logs with `correlationId`
  - actuator `health`
  - Micrometer gauges `outbox_pending` and `outbox_dead`
- Housekeeping: idempotency keys are kept for 24 h. After that, a retry still receives 409 `QUOTE_ALREADY_CONFIRMED` with the order number. Unconfirmed quotes are purged 7 days after `expiresAt`, and `SENT` outbox rows 7 days after creation. A confirmed quote is never purged: it holds the order's lines, and the order's FK protects it.

## 13. Assumptions

| ID | Assumption |
|----|-----------|
| A-01 | "Cumulative by brackets" means marginal brackets: each percentage applies only to the units inside its bracket |
| A-02 | A SKU-scoped tier rule removes that SKU from family-scoped rules |
| A-03 | Family units are assigned to brackets in normalized SKU order |
| A-04 | Gift units and combo-consumed units are excluded from tiers |
| A-05 | The default quote TTL is 15 minutes, configurable per country |
| A-06 | Authentication is out of scope. Tenant and customer come from headers |
| A-07 | A combo appears as one grouped discount entry. Line allocations only derive the tax base |
| A-08 | Credit is not reserved at quote time. The check is advisory until confirmation |
| A-09 | Cancellation and credit release are out of scope |
| A-10 | A gift subtotal condition uses gross before discounts. A gift line is always separate from a paid line |

## 14. Acceptance criteria and traceability

### 14.1 Assignment coverage

| Req | Assignment requirement | Covered by |
|-----|------------------------|------------|
| REQ-A1-1 | An endpoint returns the breakdown per line and totals | AC-01, AC-02 |
| REQ-A1-2 | Levers coexist, and the order is defined and justified. Tiers apply to a SKU or a family | AC-03, AC-15 |
| REQ-A1-3 | A credit payment indicates whether the order fits | AC-16 |
| REQ-A1-4 | Quoted total equals confirmed total, and the mechanism is explained | AC-20, AC-21 |
| REQ-A1-5 | No float. Type, scale, rounding rule and rounding points are explicit | AC-08, AC-09, AC-10 |
| REQ-A1-6 | Currency, scale and rates are not hard-coded | AC-11, AC-12 |
| REQ-A1-T1…T6 | Golden tests 1–6 | AC-04, AC-05, AC-06, AC-07, AC-17, AC-21 |
| REQ-A2-1 | Idempotency key: no duplicate orders, no double credit consumption | AC-23 … AC-28 |
| REQ-A2-2 | 201 ⇒ the order exists and is queryable, and the guarantee is defined | AC-29, AC-30, AC-31 |
| REQ-A2-3 | Push and ERP effects are neither lost nor duplicated, and the behavior with the external system down is defined | AC-32 … AC-35 |
| REQ-A2-4 | The consistency model and what it sacrifices | AC-36 |
| REQ-A3 | ADR of one page at most: decisions and alternatives, what was not implemented, code-review checklist, a risk and its mitigation | AC-40 |
| REQ-D | Single command, README with run, lever order, rounding policy and consistency model, the ADR, readable history. History is made readable by the commit and pull request conventions in the governance folder | AC-38, AC-39, AC-40, AC-41 |
| REQ-F | Front: no truncation, no hidden controls, consistent images, prevention at component level | AC-42 … AC-45 |
| REQ-P | End-user text through i18n, es default, en complete | AC-19, AC-37, AC-46 |

### 14.2 Acceptance criteria

Rows that were removed from scope are deleted. IDs are never renumbered.

| AC | Criterion | Rule | Task | Verified by |
|----|-----------|------|------|-------------|
| AC-01 | A quote returns every line and total field listed in §8 | §8 | T-10 | `QuoteContractIT` |
| AC-02 | Every golden breakdown satisfies all BR-05 identities | BR-05 | T-03 | `GoldenPricingTest#identities` |
| AC-03 | Levers apply in the BR-01 order | BR-01 | T-03…T-06 | `PricingPipelineOrderTest` |
| AC-04 | GT-01 yields a 2,750 discount and a 71,103 total | BR-02 | T-04 | `GoldenPricingTest#gt01` |
| AC-05 | GT-02 yields one COMBO entry and no TIER entry for consumed units | BR-03 | T-05 | `GoldenPricingTest#gt02` |
| AC-06 | GT-03 shows the gift line with its quantity and origin | BR-04 | T-06 | `GoldenPricingTest#gt03` |
| AC-07 | GT-04 taxes the allocated bases, for a total of 11,529 | BR-03, BR-05 | T-05 | `GoldenPricingTest#gt04` |
| AC-08 | No `float`/`double` in the domain | §4 | T-02 | `DomainPurityArchTest` |
| AC-09 | Rounding happens only at RP-2 and RP-4, and totals are never re-rounded | §4 | T-07 | `RoundingPolicyTest` |
| AC-10 | A price with a scale above the country scale is rejected | §4 | T-08 | `PriceScaleValidationIT` |
| AC-11 | The same cart priced in CO (scale 0) and EC (scale 2) follows each country's config | §5 | T-07 | `GoldenPricingTest#multiCountry` |
| AC-12 | Each gift tax policy follows the BR-04 table | BR-04 | T-06 | `GoldenPricingTest#giftPolicies` |
| AC-13 | Duplicate SKUs merge, and the client's line order does not change the breakdown | BR-00 | T-03 | `CartNormalizationTest` |
| AC-14 | `qty ≤ 0` → 400, empty cart → 400, unknown SKU → 422 | BR-00 | T-10 | `QuoteValidationIT` |
| AC-15 | A SKU rule beats a family rule | BR-02 | T-04 | `TierPrecedenceTest` |
| AC-16 | A credit quote returns `fits`, `available` and `shortfall`. A total equal to available fits | BR-06 | T-10 | `CreditAdvisoryTest` |
| AC-17 | GT-05: an over-limit quote reports `fits=false`, and confirming it → 409 with no order and no ledger row | BR-06 | T-10, T-12 | `GoldenQuoteTest#gt05`, `CreditLimitIT` |
| AC-18 | Error responses are `problem+json` with a stable `code` and `correlationId` | §8 | T-09 | `ProblemDetailContractIT` |
| AC-19 | Error text follows `Accept-Language`, and unsupported languages fall back to es | §9 | T-09 | `LocalizedErrorIT` |
| AC-20 | A quote is persisted insert-only, with its lines, discount components and `expiresAt`, and the order stores no amount | §6 | T-10, T-11 | `QuotePersistenceIT`, `SchemaConstraintsIT` |
| AC-21 | GT-06: after price and lever changes, the confirmed total equals the quoted total, and a fresh quote differs | §6 | T-12 | `QuoteConfirmRegressionIT` |
| AC-23 | A missing or invalid `Idempotency-Key` → 400 | §7.3 | T-11 | `IdempotencyContractIT` |
| AC-24 | Same key and same body → a byte-identical replay with `Idempotent-Replayed: true` | §7.3 | T-11 | `IdempotencyReplayIT` |
| AC-25 | Same key and a different body → 422 | §7.3 | T-11 | `IdempotencyReplayIT` |
| AC-26 | N concurrent requests with one key → one order and one ledger row | §7.1 | T-12 | `ConfirmOrderConcurrencyIT` |
| AC-27 | Two concurrent confirmations that together exceed the limit → exactly one succeeds | BR-06 | T-12 | `ConfirmOrderConcurrencyIT` |
| AC-28 | A new key on a confirmed quote → 409 with the existing order number | §7.3 | T-11 | `QuoteAlreadyConfirmedIT` |
| AC-29 | A failure injected before commit leaves no order, no ledger row, no outbox row and no key | §7.2 | T-12 | `ConfirmOrderFailureInjectionIT` |
| AC-30 | After a crash after commit, the retry returns the same order number | §7.2 | T-12 | `ConfirmOrderFailureInjectionIT` |
| AC-31 | Every 201 order is immediately readable by id and in the history | I-201 | T-13 | `OrderReadYourWritesIT` |
| AC-32 | A confirmed order produces exactly one PUSH and one ERP outbox event | §7.4 | T-11 | `OutboxWriteIT` |
| AC-33 | With the ERP down, events are retried with backoff and delivered after recovery | §7.4 | T-14 | `OutboxRelayIT#outageRecovery` |
| AC-34 | A crash between send and mark re-sends, and the consumer deduplicates by `event_id` | §7.4 | T-14 | `OutboxRelayIT#leaseExpiry` |
| AC-35 | Non-retryable errors or exhausted attempts → `DEAD`, counted in `outbox_dead` | §7.4 | T-14 | `OutboxRelayIT#dead` |
| AC-36 | The consistency model and its sacrifices are written in the API README and the ADR | §7.5 | T-00, T-16 | `check-docs` (required headings) |
| AC-37 | Every error code has messages in es and en, and the bundles have identical keys | §9 | T-09 | `ErrorCatalogI18nTest`, `check-docs` |
| AC-38 | One command runs all the API code and tests | §11 | T-01 | `./mvnw verify` |
| AC-39 | The API README contains, inline: how to run, lever order, rounding policy, consistency model | §15 | T-00, T-16 | `check-docs` (required headings) |
| AC-40 | ADR-0001 fits one page (≤ 500 words per language) and covers the four A3 items | §15 | T-00, T-16 | `check-docs` (ADR rules and required headings) |
| AC-41 | The history shows one Conventional Commit per task | TASKS | all | Review of `git log` |
| AC-42 | Long product names wrap without truncation at a 768 px viewport | §10 | T-17 | `order-summary.spec.ts` |
| AC-43 | The action bar respects the safe area and the keyboard | §10 | T-17 | `order-summary.spec.ts` |
| AC-44 | All grid images render through `ui-product-image` with one aspect ratio | §10 | T-17 | `order-summary.spec.ts` |
| AC-45 | The web README explains prevention at component level | §10 | T-17 | `check-docs` (required headings) |
| AC-46 | Web `es` and `en` dictionaries have identical keys | §9 | T-17 | `check-docs` |
| AC-47 | The domain is free of Spring and JDBC, and `@Transactional` appears only in the application layer | §12 | T-01, T-11 | `DomainPurityArchTest`, `TransactionBoundaryArchTest` |
| AC-48 | Housekeeping purges keys, expired quotes and sent events by TTL | §12 | T-15 | `HousekeepingIT` |

### 14.3 Beyond the assignment

Everything the assignment does not ask for is listed here, with what it buys. Nothing else is built. The tag says what the item adds: quality, stability or scalability. "Owner-kept" and "Owner-requested" mark items that the product owner decided to keep or asked for.

| Item | AC | Tag | Benefit | Task | Origin |
|------|----|-----|---------|------|--------|
| Cart normalization (merge duplicates, sort) | AC-13 | Stability | The same cart always prices the same, whatever the client's line order | T-03 | Design |
| Input validation with stable error codes | AC-14 | Stability | Bad input fails fast with 400/422 instead of a 500 or a wrong price | T-10 | Design |
| `problem+json` errors with `code` and `correlationId` | AC-18 | Quality | Clients branch on a stable code. Support traces any error to its log line | T-09 | Design |
| Architecture rules (ArchUnit) | AC-47 | Quality | The pure domain and the transaction boundary cannot erode silently | T-01, T-11 | Design |
| JaCoCo gate ≥ 90% on `pricing` | — | Quality | The money logic cannot lose its tests without failing the build | T-01 | Owner-kept |
| PIT mutation testing (optional profile) | — | Quality | Proves that the pricing tests detect wrong code, not just execute it | T-16 | Owner-kept |
| Retention housekeeping | AC-48 | Scalability | Quotes, events and keys do not grow without bound | T-15 | Owner-kept |
| Outbox gauges `outbox_pending`, `outbox_dead` | — | Stability | A stuck ERP or push integration is visible before customers notice | T-14 | Owner-kept |
| Separate repositories per component | — | Scalability | Service, screen, infrastructure and documentation are released and owned independently. `orderflow-engine-api` still stands alone as the deliverable | T-00 | Owner-kept |
| Tenant-leading composite keys | — | Scalability | Tables can be partitioned or sharded by tenant without re-keying | T-08 | Design |
| Documentation governance (`check-docs`) | — | Quality | Every document is indexed, paired, linked and within its limits, and each repository stands alone | T-00 | Design |
| Branch rulesets (code: `dev` and `main`; docs: `main` only) and deletion of task branches | — | Stability | `main` holds only closed phases and the delivered product, and no stray branch outlives its task | T-00 | Owner-requested |
| Task progress chart generated from the HANDOFF | — | Quality | Progress and schedule slip are visible at a glance, and the chart cannot drift from the task table | T-00 | Owner-requested |
| Separate `orderflow-engine-tools` repository | — | Quality | Generators and checks never mix with the documents they produce | T-00 | Owner-requested |
| ADR in Word and PDF (Arial 12 template, one page) | — | Quality | A printable, formal ADR whose source and hash are recorded, so it cannot drift from the Markdown | T-00, T-16 | Owner-requested |
| `orderflow-engine-app` (reserved, empty) | — | Scalability | A future mobile client has a home. Nothing is built | — | Owner-kept |
| Own Docker files per executable repository (`api`, `web`, `tools`) and a full-stack compose in `infra` | — | Scalability | Each repository builds and runs alone, and the whole stack starts with one command | T-00, T-01, T-17 | Owner-requested |
| Environment contract: `.env.example` per repository, `.env` never versioned | — | Stability | Configuration is explicit, and no secret reaches a repository or an image | T-01, T-17 | Owner-requested |
| Repository integrity check (`check-integrity`): ignore files, line endings, secrets | — | Quality | The integrity policy is enforced on every task, not just written down | T-00 | Owner-requested |

## 15. Deliverables and repositories

The workspace contains sibling folders, and each one is a future repository with its own `README` (en + es) and `LICENSE` (MIT). **`orderflow-engine-api` is the deliverable repository the assignment describes.** On its own it contains the code, the tests, the single command, the README with the four required topics, and the ADR. `orderflow-engine-docs` is delivered alongside, with the full specification.

| Folder | Purpose |
|--------|---------|
| `orderflow-engine-api` | **The deliverable.** Spring Boot service, tests, golden fixtures, Liquibase, README, ADR-0001, and its own `Dockerfile` and `compose.yaml` (service and PostgreSQL) |
| `orderflow-engine-web` | The optional order-summary screen, with its own `Dockerfile` (nginx) and `compose.yaml` |
| `orderflow-engine-infra` | Docker Compose for the full stack, built from the sibling repositories, and `scripts/test-all` |
| `orderflow-engine-docs` | Specification, plan, contract, data model, architecture, governance, and the generated figures, progress chart and ADR in Word and PDF |
| `orderflow-engine-tools` | Generators and checks: figures, progress chart, ADR in Word and PDF, documentation, integrity and consistency checks. Its own `Dockerfile` (Python and Node) |
| `orderflow-engine-app` | Reserved for a future mobile app. Not built in this delivery |

Each executable repository builds and runs with its own Docker files and its own `.env.example`. A `.env` is never versioned. Rules and their check: [repository-integrity.md](../governance/repository-integrity.md).

Delivery documents exist in **English and Spanish**. Internal technical documents are English only. Rules: [documentation-rules.md](../governance/documentation-rules.md). Map: the [docs README](../README.md).

| Document | Location | Languages |
|----------|----------|-----------|
| API README: how to run, lever order, rounding policy, consistency model, golden tests, ADR, not included, repositories | `orderflow-engine-api/README(.es).md` | en, es |
| ADR-0001 order pricing and confirmation: the single, one-page ADR of A3 | `orderflow-engine-api/docs/adr/0001-order-pricing-and-confirmation(.es).md` | en, es |
| Web README: how to run, prevention at component level | `orderflow-engine-web/README(.es).md` | en, es |
| Specification set: constitution, SPEC, plan, TASKS, API contract, data model, architecture | `orderflow-engine-docs/spec/` | en |
| Handoff: verified state, commands, PIT score, time spent | `orderflow-engine-docs/handoff/HANDOFF.md` | en |
