---
id: API-CONTRACT
title: Orderflow Engine — API contract
status: accepted
lang: en
pair: none
last-reviewed: 2026-10-14
---

# Orderflow Engine — API contract

> **Question this document answers:** what exactly does each operation accept, return and reject?

This document is the contract. No OpenAPI file is produced, because the assignment does not ask for one (SPEC §1). The rules come from [SPEC.md](SPEC.md) §3–§8, and the contract tests (`*ContractIT`) assert every status and error body listed here.

## 1. Conventions

| Topic | Convention |
|-------|-----------|
| Base path | `/api/v1` |
| Media type | `application/json`. Errors use `application/problem+json` (RFC 9457) |
| Tenant | `X-Tenant-Id` header, required on every business operation |
| Customer | `X-Customer-Id` header, required on every business operation |
| Language | `Accept-Language`, optional. Unsupported or missing → `es` |
| Money | `{"amount": "71103", "currency": "COP"}`. The amount is a string at the currency's scale |
| Identifiers | UUIDs for `quoteId` and `orderId`. The order number is human-readable: `<COUNTRY>-<sequence>`, e.g. `CO-000042` |
| Correlation | `X-Correlation-Id` is echoed or generated, and is included in every error body |

## 2. Operation cards

### E-01 Create a quote — `POST /api/v1/quotes`

Request:

```json
{
  "paymentMethod": "CREDIT",
  "lines": [ { "sku": "BEV-COLA-350", "qty": 25 } ]
}
```

Response `201 Created`, with `Location: /api/v1/quotes/{quoteId}`. This example is GT-01, tenant `co-alpha`:

```json
{
  "quoteId": "7f6c1a52-…",
  "status": "OPEN",
  "expiresAt": "2026-10-14T15:15:00Z",
  "currency": "COP",
  "lines": [{
    "sku": "BEV-COLA-350", "description": "Cola 350 ml", "qty": 25, "isGift": false,
    "unitPrice": "2500", "gross": "62500",
    "discounts": [ { "origin": "TIER:0192f3c4-…", "amount": "2750" } ],
    "comboAllocations": [],
    "net": "59750", "taxableBase": "59750", "taxRate": "0.190000", "tax": "11353"
  }],
  "totals": {
    "grossSubtotal": "62500",
    "discounts": [ { "origin": "TIER:0192f3c4-…", "amount": "2750" } ],
    "gifts": [],
    "taxableBase": "59750",
    "taxes": "11353",
    "total": "71103"
  },
  "credit": { "fits": true, "available": "500000", "shortfall": "0" }
}
```

| Error | Status | When |
|-------|--------|------|
| `VALIDATION_FAILED` | 400 | Malformed body, missing `paymentMethod` |
| `CART_EMPTY` | 400 | `lines` is empty |
| `INVALID_QUANTITY` | 400 | `qty ≤ 0` or not an integer |
| `TENANT_NOT_FOUND` | 400 | Unknown `X-Tenant-Id` |
| `CUSTOMER_NOT_FOUND` | 404 | Unknown customer for the tenant |
| `UNKNOWN_SKU` | 422 | SKU unknown, or not sold by this tenant |
| `CREDIT_NOT_ENABLED` | 422 | `CREDIT` requested by a customer without a credit account |

### E-02 Read a quote — `GET /api/v1/quotes/{quoteId}`

Returns `200` with the same body as E-01, including the current `status`: `OPEN`, `CONFIRMED` or `EXPIRED`, derived when the quote is read. `credit` is recomputed from the customer's current position while the quote is `OPEN`, and omitted once it is `CONFIRMED` or `EXPIRED`, because the advisory check no longer applies.

| Error | Status | When |
|-------|--------|------|
| `QUOTE_NOT_FOUND` | 404 | Unknown id, or the quote belongs to another tenant or customer (BD-01) |

### E-03 Confirm an order — `POST /api/v1/orders`

Headers: `Idempotency-Key` (required, `^[A-Za-z0-9_-]{8,64}$`, a UUID is recommended).

```json
{ "quoteId": "7f6c1a52-…" }
```

Response `201 Created`, with `Location: /api/v1/orders/{orderId}`:

```json
{
  "orderId": "c0a8…", "orderNumber": "CO-000042", "status": "CONFIRMED",
  "quoteId": "7f6c1a52-…", "confirmedAt": "2026-10-14T15:03:12Z",
  "currency": "COP", "lines": [ "… rendered from the quote lines …" ], "totals": { "… rendered from the quote lines …": "" },
  "paymentMethod": "CREDIT"
}
```

A replay of the same key and body returns the stored response byte for byte, with `Idempotent-Replayed: true`.

| Error | Status | When |
|-------|--------|------|
| `IDEMPOTENCY_KEY_MISSING` / `IDEMPOTENCY_KEY_INVALID` | 400 | Header absent or malformed |
| `QUOTE_NOT_FOUND` | 404 | Unknown, or not owned by the caller |
| `QUOTE_EXPIRED` | 409 | `now > expiresAt` |
| `QUOTE_ALREADY_CONFIRMED` | 409 | Confirmed under another key. The body includes `orderNumber` and `orderId` |
| `CREDIT_LIMIT_EXCEEDED` | 409 | The conditional credit update affected 0 rows |
| `IDEMPOTENCY_KEY_IN_PROGRESS` | 409 | The same key is still being processed when `lock_timeout` expires |
| `IDEMPOTENCY_KEY_REUSED` | 422 | Same key, different request hash |

### E-04 Read an order — `GET /api/v1/orders/{orderId}`

Returns `200` with the E-03 body.

| Error | Status | When |
|-------|--------|------|
| `ORDER_NOT_FOUND` | 404 | Unknown, or not owned by the caller |

### E-05 Order history — `GET /api/v1/orders?customerId={id}&size=20`

Returns `200` with `{ "items": [ {orderId, orderNumber, confirmedAt, total} ] }`, newest first. `total` is summed from the quote lines. `size` defaults to 20 and is capped at 100. `customerId` must match `X-Customer-Id` (BD-01).

### E-06 Health — `GET /actuator/health`

Returns `200 {"status":"UP"}` when the database is reachable. It needs no tenant headers.

## 3. Error body

```json
{
  "type": "https://orderflow-engine.example/problems/credit-limit-exceeded",
  "title": "Cupo de crédito insuficiente",
  "status": 409,
  "detail": "El total del pedido supera el cupo disponible.",
  "code": "CREDIT_LIMIT_EXCEEDED",
  "correlationId": "5b1e…"
}
```

`title` and `detail` come from the i18n bundles, resolved with `Accept-Language`. `code` never changes once published. Any operation may also answer 500 `INTERNAL_ERROR`: it is logged with the `correlationId`, and the body never exposes internals.

## 4. Behavior decisions

| Code | Decision | Why |
|------|----------|-----|
| BD-01 | Foreign quotes and orders return 404, not 403 | Does not reveal that the resource exists |
| BD-02 | Failed confirmations are not stored under the idempotency key | The client can fix the cause and retry with the same key |
| BD-03 | `QUOTE_ALREADY_CONFIRMED` carries the existing order number | A client that lost the first response (app reinstall) can still find its order |
| BD-04 | The order body is rendered from its quote's lines. The order itself stores no amount | One copy of every figure: the confirmed total is the quoted total by construction (P-02) |

## 5. Known gaps

| Code | Gap | Reason |
|------|-----|--------|
| GAP-01 | No authentication. Tenant and customer come from headers | Out of scope (A-06). An API gateway would inject them in production |
| GAP-02 | No cancellation endpoint | Out of scope (A-09) |
| GAP-03 | No quote revocation | A wrong price is honored until the quote expires. This is the ADR's recorded risk and mitigation |
