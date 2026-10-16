---
id: ARCHITECTURE
title: Orderflow Engine — Architecture
status: accepted
lang: en
pair: none
last-reviewed: 2026-10-15
---

# Orderflow Engine — Architecture

> **Question this document answers:** how is Orderflow Engine shaped (topology, modules, dependency rule, request flows), and where are its risks?

Decisions are in [plan.md](plan.md), and tables are in [data-model.md](data-model.md).

## 1. Topology

![Runtime topology](../diagrams/01-runtime-topology.svg)

| Runtime element | Responsibility | Talks to |
|-----------------|---------------|----------|
| `orderflow-engine-web` | Runs the full flow over HTTP: cart, quote summary, confirmation, order and history (SPEC §10). Its nginx serves the built Angular client and forwards `/api/` to the API, so the browser sees one origin | `orderflow-engine-api` over HTTP, through its own nginx (§1.1) |
| `orderflow-engine-api` | Prices carts, seals quotes, confirms orders, serves history, relays outbox events | PostgreSQL. Push and ERP gateways (fakes in this delivery) |
| PostgreSQL | Single source of truth for catalog, levers, credit, quotes, orders, idempotency keys and outbox | — |
| Push gateway, ERP | External systems that receive `order.confirmed` | Called only by the outbox relay |

The six folders of the workspace and what each one takes from another:

![Workspace and repositories](../diagrams/02-workspace-repositories.svg)

### 1.1 Web to API: one origin

The Angular client calls only relative paths (`/api/v1/…`). Whoever serves the client also forwards `/api/` to the API, so the browser never makes a cross-origin call and the API has no CORS configuration. The API address is runtime configuration of the web container, never compiled into the bundle.

| Where the client runs | Serves the client | Forwards `/api/` to | Set by |
|-----------------------|-------------------|---------------------|--------|
| Full stack (`orderflow-engine-infra`) | nginx in the web container | `http://api:8080`, the API service on the compose network | `API_UPSTREAM` in the infra `compose.yaml` (it follows `API_PORT`) |
| Web alone (`orderflow-engine-web`) | nginx in the web container | `http://host.docker.internal:8080`, an API started by its own compose on the host | `API_UPSTREAM` in the web `.env.example`. `extra_hosts: host-gateway` makes the name resolve on Linux |
| Development (`ng serve`) | Angular dev server | `http://localhost:8080` | `API_UPSTREAM`, read by `proxy.conf.mjs` |

| Rule | Why |
|------|-----|
| nginx renders `API_UPSTREAM` into its configuration when the container starts (`/etc/nginx/templates`). Only defined variables are substituted, so nginx's own `$uri` stays | One image runs against any API. A new address needs a restart, not a rebuild |
| `proxy_pass` carries no URI, so `/api/v1/…` reaches the API unchanged | The API keeps one base path, whoever calls it |
| Request headers (`X-Tenant-Id`, `X-Customer-Id`, `Idempotency-Key`, `Accept-Language`, `X-Correlation-Id`) and response headers (`Location`, `Idempotent-Replayed`) pass through unchanged | The contract is the same through the proxy as against the API directly |
| Any other path falls back to `index.html` | A reload on a client route (`/quotes/:quoteId`, `/orders/:orderId`) still works |
| Only `/api/` is forwarded. `/actuator` is not | The browser has no use for health, and it stays off the public origin |
| The web service depends on the API service in compose | nginx resolves the upstream name when it starts, and exits if the name does not exist yet |

## 2. Hexagon and modules

![Ports and adapters](../diagrams/03-ports-and-adapters.svg)

| Module | Kind | Contains |
|--------|------|----------|
| `pricing.domain` | Domain (pure) | `Money`, `CurrencyPolicy`, `PricingEngine`, lever steps, `Breakdown` |
| `quoting` | Application | `CreateQuoteUseCase`, `GetQuoteUseCase`. Ports: `SnapshotProvider`, `QuoteRepository`, `CreditAdvisor` |
| `ordering` | Application | `ConfirmOrderUseCase`, `GetOrderUseCase`, `ListOrdersUseCase`. Ports: `OrderRepository`, `CreditLedger`, `IdempotencyStore`, `OutboxWriter` |
| `outbox` | Application | `OutboxRelay`, `HousekeepingJob`. Ports: `PushNotificationPort`, `ErpPublisherPort`, `OutboxRepository` |
| `adapters.in.web` | Driving adapter | Controllers, DTOs, `ProblemDetailsAdvice`, locale and correlation filters |
| `adapters.out.persistence` | Driven adapter | `JdbcClient` repositories with explicit SQL and locks |
| `adapters.out.push`, `adapters.out.erp` | Driven adapters | Fake gateways that deduplicate by `event_id` and can be told to fail |
| `config` | Composition root | Wiring of ports to adapters, typed configuration |

### 2.1 Dependency rule

![Dependency rule](../diagrams/04-dependency-rule.svg)

Dependencies point inward: adapters → application → domain. The domain imports only the JDK. Use cases depend on ports, never on adapters. Only `config` knows both sides. `DomainPurityArchTest` and `TransactionBoundaryArchTest` enforce the rule (P-05, P-06).

## 3. Flows

### 3.1 Pricing pipeline

![Pricing pipeline](../diagrams/05-pricing-pipeline.svg)

The engine is a pure function `price(cart, snapshot) → breakdown`. It applies the steps of SPEC BR-01 in order. Rounding happens only at RP-2 and RP-4 (SPEC §4).

### 3.2 Quote

![Quote sequence](../diagrams/06-quote-sequence.svg)

1. The controller validates headers and body, then calls `CreateQuoteUseCase`.
2. The use case normalizes the cart and loads the pricing snapshot (prices, active levers, tax rates, country policy) through `SnapshotProvider`. The snapshot is an in-memory input of the engine. It is not stored.
3. The engine prices the cart. `CreditAdvisor` adds the advisory credit result when the payment is `CREDIT`.
4. `QuoteRepository` inserts the quote, its lines and their discount components, which are never updated afterwards. The controller returns 201.

### 3.3 Confirmation

![Confirmation transaction](../diagrams/07-confirm-transaction.svg)

`ConfirmOrderUseCase` runs SPEC §7.1 in **one** transaction, taking locks in the order idempotency → quote → credit. The 201 is written only after the commit returns (I-201). Nothing in this transaction calls a remote system.

### 3.4 Outbox relay

![Outbox lifecycle](../diagrams/09-outbox-lifecycle.svg)

The relay claims due events with a lease in a short transaction, commits, calls the gateway outside any transaction, and then marks the event `SENT`, or schedules a retry, or marks it `DEAD`. A crash between send and mark causes a re-send after the lease expires. Consumers deduplicate by `event_id`.

### 3.5 Web flow

The client is a thin layer over the contract: it never computes money, never stores an amount, and shows the API's figures and problem texts as they arrive.

| Screen (route) | Calls | What it adds |
|----------------|-------|--------------|
| Every screen | — | An interceptor adds `X-Tenant-Id` and `X-Customer-Id` from the selected store (A-06), and `Accept-Language` from the active language, so problem texts arrive in the screen's language |
| Cart (`/cart`, also `/`) | E-07, then E-01 | List prices and units only. The cart is kept per store in the browser |
| Quote summary (`/quotes/:quoteId`) | E-02, then E-03 | The order summary of SPEC §10. `fits=false` disables "Confirm order" and offers "Pay cash", a new E-01 with `CASH`. An expired quote offers "Quote again", a new quote with the same lines |
| Order (`/orders/:orderId`) | E-04 | "Order confirmed" after a 201, "This order was already recorded" after a replay (`Idempotent-Replayed: true`) |
| My orders (`/orders`) | E-05 | Newest first, in the body's `currency` |

**Client idempotency.** The client creates the `Idempotency-Key` (a UUID) the first time "Confirm order" is tapped for a quote, stores it with the quote in browser storage, and sends the same key on every retry, also after a reload. "Confirm order" is disabled while the request is in flight. A new quote gets a new key. The server's guarantees (SPEC §7.3) never depend on this; the client only avoids asking twice.

**Ways out of a problem.** The notice shows the API's `title` and `detail`, plus the `code` and the start of the `correlationId` for support. The screen adds the action that leaves the error (SPEC §10.5):

| Code | Action |
|------|--------|
| `CREDIT_NOT_ENABLED` (E-01) | "Pay cash" |
| `CREDIT_LIMIT_EXCEEDED` (E-03): another order used the credit after this quote was priced (A-08) | "Pay cash" |
| `QUOTE_EXPIRED` (E-03) | "Quote again": a new quote with the same lines |
| `QUOTE_ALREADY_CONFIRMED` (E-03) | "View order {number}": opens the order the body names |
| `IDEMPOTENCY_KEY_IN_PROGRESS`, `INTERNAL_ERROR`, or no answer (a 5xx without a `problem+json` body counts as no answer) | "Retry": the same request again. E-03 keeps its key |
| Any other | The screen's default way out: "Review cart" on the cart, "Edit cart" on the summary, "View my orders" on the order, "Go shopping" on My orders |

## 4. Cross-cutting concerns

| Concern | Approach |
|---------|----------|
| Errors | Typed domain and application errors carry a stable code. Only `ProblemDetailsAdvice` maps them to HTTP status and localized `problem+json` |
| i18n | API: `MessageSource` bundles. Web: ngx-translate JSON for the client's own text. Problem `title` and `detail` come from the API in the language the client sends in `Accept-Language`, so a notice never mixes languages. Spanish is the default (SPEC §9) |
| Observability | Logs with `correlationId`. Actuator health. `outbox_pending` and `outbox_dead` gauges |
| Configuration | Environment variables. No secret defaults. Country behavior is data (P-07). The web container reads the API address (`API_UPSTREAM`) when it starts, never at build time (§1.1) |
| Time | An injected `Clock`. Tests fix it, which makes goldens and expiry deterministic |
| Browser origin | One origin: the web client's nginx forwards `/api/` to the API. The API has no CORS configuration (§1.1) |
| Client retries | The web client keeps one `Idempotency-Key` per quote and reuses it on every retry (§3.5). The server's guarantees (SPEC §7.3) never depend on it |

## 5. Risks

The risk register (RISK-01…RISK-07) is in [plan.md](plan.md) §4. The two that shape this architecture most:

| Risk | Where it lives | Containment |
|------|----------------|-------------|
| RISK-01 credit-row contention | Step 3 of the confirmation | Short transaction, `lock_timeout`, explicit 409 |
| RISK-05 consumer without dedupe | Outbox relay | `event_id` contract, push collapse id, ERP upsert |
