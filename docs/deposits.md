# Deposit (Inbound Merchant)

---

## How Deposit Processing Works

The platform operates on a P2P model: the merchant's end user transfers fiat to a counterparty (the recipient), and the platform guarantees accounting and settlement.

### Participants

| Role | Description |
|------|------------|
| Merchant | Your platform. Creates deposit orders via API |
| End user | The final client topping up their balance |
| Recipient | The counterparty (outbound merchant) who receives the fiat transfer and confirms receipt |
| Platform | Order matching, payment widget, USDT settlement |

### Happy path

1. The end user initiates a top-up on the merchant's platform
2. The merchant calls `POST /merchant/orders/deposit`
3. The platform finds a counterpart order → returns `READY` + `payerWidgetUrl`
4. The merchant redirects the user to `payerWidgetUrl`
5. The user transfers money using the displayed payment details and confirms in the widget
6. The recipient confirms receipt (or auto-confirm fires) → the match completes
7. The platform sends webhook `order.completed` to the merchant
8. The merchant credits `netAmountRub` to the user's account

### What can go wrong

| Situation | Behavior |
|-----------|----------|
| No counterpart liquidity | `depositStatus: NO_LIQUIDITY` — order is not created, no widget |
| Processing temporarily unavailable | `depositStatus: PROCESSING_UNAVAILABLE` |
| User did not pay before the deadline | Match and order expire (`EXPIRED`); webhooks `match.rejected` + `order.expired` |
| Partial match | Order is matched in parts; the remainder becomes a tail (`TAIL`) |
| Payment dispute | Match → `DISPUTED`; settlement is frozen |

---

## Creating a Deposit Order

```
POST {BASE_URL}/api/v1/merchant/orders/deposit
x-api-key: mk_live_...
x-timestamp: 1717651200000
x-signature: <base64>
Content-Type: application/json
```

### Request body

| Field | Type | Required | Description |
|-------|------|:---:|---------|
| `amount` | string (decimal) | Yes | Top-up amount in rubles, e.g. `"1500.00"` (up to 8 fractional digits) |
| `currency` | string | — | Currency; defaults to `RUB` |
| `externalOrderId` | string | — | Your order ID — used to link the platform order to your record |
| `idempotencyKey` | string | — | Idempotency key; if omitted the platform generates one |

```json
{
  "amount": "1500.00",
  "currency": "RUB",
  "externalOrderId": "order-42",
  "idempotencyKey": "req-42"
}
```

On network failures, retry with the same `idempotencyKey` — the platform returns the result of the original order rather than creating a new one.

### Response — match found (201)

```json
{
  "timestamp": 1717651200000,
  "path": "/api/v1/merchant/orders/deposit",
  "method": "POST",
  "statusCode": 201,
  "success": true,
  "depositStatus": "READY",
  "depositOrderId": "f1c2a3b4-0000-0000-0000-000000000001",
  "withdrawalOrderId": "a9b8c7d6-0000-0000-0000-000000000002",
  "matchId": "m-77e6f500-0000-0000-0000-000000000003",
  "paymentDeadlineAt": "2026-06-11T12:34:56.000Z",
  "payerWidgetUrl": "https://dvspayment.co/widget/payer-token-abc",
  "paymentDetails": { },
  "message": "Deposit matched"
}
```

### Response — no liquidity (201)

```json
{
  "timestamp": 1717651200000,
  "path": "/api/v1/merchant/orders/deposit",
  "method": "POST",
  "statusCode": 201,
  "success": true,
  "depositStatus": "NO_LIQUIDITY"
}
```

### `depositStatus` values

| Value | Meaning | Action |
|-------|---------|--------|
| `READY` | Match found; widget is ready | Redirect the user to `payerWidgetUrl` |
| `NO_LIQUIDITY` | No counterpart orders available | Show "try again later"; order is not created |
| `PROCESSING_UNAVAILABLE` | Temporary processing failure (e.g. balance lock) | Retry later |

When retrying with the same `externalOrderId` after a previous order has expired, the platform issues a **new** `depositOrderId`.

---

## Payment Widget

After receiving `payerWidgetUrl`, redirect the end user to that URL:

```js
res.redirect(response.payerWidgetUrl);
```

In the widget the user:

1. Sees the recipient's payment details and the transfer amount
2. Makes a bank transfer
3. Uploads the receipt and confirms payment

The user's confirmation in the widget does not yet mean the top-up is complete. The final status is set only after the recipient confirms receipt of funds (or the auto-confirm timer elapses on the outbound side). Track status via webhooks.

**Payment deadline:** `paymentDeadlineAt` contains the expiry time in ISO 8601. After the deadline, the match and order move to `EXPIRED` — create a new order.

---

## Order and Match Statuses

### Deposit order statuses

| Status | Meaning |
|--------|---------|
| `PENDING` | Created, awaiting a match |
| `PARTIALLY_MATCHED` | Partially matched |
| `TAIL` | Unmatched remainder after partial matching |
| `MATCHED` | Fully matched, awaiting payment |
| `CONFIRMING` | Confirmation in progress |
| `COMPLETED` | Completed; funds accounted |
| `DISPUTED` | A dispute has been opened |
| `EXPIRED` | Payment deadline passed |
| `CANCELLED` | Cancelled (e.g. payer cancelled in widget) |
| `FORCE_CLOSED` | Force-closed by an operator |

### Match statuses

| `status` | `displayStatus` | Meaning |
|----------|-----------------|---------|
| `ACTIVE` | `PENDING` | Match created; awaiting user payment |
| `CONFIRMING` | `AWAITING_RECIPIENT` | User payment received; awaiting recipient confirmation |
| `COMPLETED` | `CONFIRMED` | Recipient confirmed — match is complete |
| `EXPIRED` | `EXPIRED` | Payment deadline passed |
| `DISPUTED` | `DISPUTED` | A dispute has been opened |
| `CANCELLED` | — | Match cancelled |

Use `displayStatus` for integration logic.

### Merchant actions by status

| State | Action |
|-------|--------|
| `MATCHED` / `displayStatus: PENDING` | User must pay in the widget |
| `CONFIRMING` / `AWAITING_RECIPIENT` | Awaiting recipient confirmation; no action needed |
| `COMPLETED` / `CONFIRMED` | Credit `netAmountRub` to the user; fetch the receipt |
| `EXPIRED` | Offer the user a new top-up |
| `DISPUTED` | Suspend crediting until the dispute is resolved |

---

## Tracking Orders

The recommended approach is **webhooks** (see [Webhooks](webhooks.md)). The read API is also available for reconciliation.

### Get order by ID

```
GET {BASE_URL}/api/v1/merchant/orders/{id}
x-api-key: mk_live_...
x-timestamp: ...
x-signature: ...
```

```json
{
  "timestamp": 1717651200000,
  "path": "/api/v1/merchant/orders/f1c2a3b4-...",
  "method": "GET",
  "statusCode": 200,
  "success": true,
  "order": {
    "id": "f1c2a3b4-0000-0000-0000-000000000001",
    "type": "DEPOSIT",
    "status": "CONFIRMING",
    "currency": "RUB",
    "amount": "1500.00",
    "amountMatched": "1500.00",
    "externalOrderId": "order-42",
    "matchIds": ["m-77e6f500-0000-0000-0000-000000000003"],
    "matches": [
      {
        "id": "m-77e6f500-0000-0000-0000-000000000003",
        "amount": "1500.00",
        "status": "CONFIRMING",
        "displayStatus": "AWAITING_RECIPIENT"
      }
    ],
    "tailAmount": null,
    "tailCreatedAt": null,
    "ttlExpiresAt": "2026-06-11T12:34:56.000Z",
    "isTest": false,
    "createdAt": "2026-06-11T12:00:00.000Z",
    "updatedAt": "2026-06-11T12:10:00.000Z"
  },
  "message": null
}
```

### List orders

```
GET {BASE_URL}/api/v1/merchant/orders?type=DEPOSIT&page=1&limit=20
x-api-key: mk_live_...
x-timestamp: ...
x-signature: ...
```

| Parameter | Description |
|-----------|------------|
| `type` | `DEPOSIT` / `WITHDRAWAL` |
| `status` | Filter by order status |
| `search` | Order ID or `externalOrderId` |
| `dateFrom`, `dateTo` | `createdAt` range (ISO 8601) |
| `amountFrom`, `amountTo` | Amount range |
| `page`, `limit` | Pagination (default: 1 / 20) |

### Get match by ID

```
GET {BASE_URL}/api/v1/merchant/matches/{id}
x-api-key: mk_live_...
x-timestamp: ...
x-signature: ...
```

Returns `displayStatus`, `matchedAmount`, `payerConfirmed` / `recipientConfirmed`, `payerConfirmedAt`, `receipts`, `depositOrderId` / `withdrawalOrderId`, `completedAt`, and available `actions` (`canConfirm`, `canReject`, etc.).

### List matches

```
GET {BASE_URL}/api/v1/merchant/matches?status=AWAITING_RECIPIENT&page=1&limit=20
x-api-key: mk_live_...
x-timestamp: ...
x-signature: ...
```

| `status` value | Meaning |
|----------------|---------|
| `PENDING` | Awaiting payer action |
| `AWAITING_RECIPIENT` | Receipt uploaded; awaiting recipient |
| `CONFIRMED` | Match completed |
| `DISPUTED` | Dispute open |
| `EXPIRED` | Expired |

Also supports `search` (match ID or order ID), `dateFrom`, `dateTo`.

---

## Payment Receipts

Upon completed matches the platform generates payment receipts (PDF documents).

### List receipts

```
GET {BASE_URL}/api/v1/merchant/documents?page=1&limit=20
x-api-key: mk_live_...
x-timestamp: ...
x-signature: ...
```

Query parameters: `page`, `limit`, `search`, `dateFrom`, `dateTo`.

### Download a receipt

```
GET {BASE_URL}/api/v1/merchant/documents/{id}/download
x-api-key: mk_live_...
x-timestamp: ...
x-signature: ...
```

```json
{
  "timestamp": 1717651200000,
  "path": "/api/v1/merchant/documents/{id}/download",
  "method": "GET",
  "statusCode": 200,
  "success": true,
  "document": {
    "id": "uuid",
    "matchId": "uuid",
    "orderId": "uuid",
    "type": "application/pdf",
    "uploadedAt": "2026-06-11T12:15:00.000Z"
  },
  "downloadUrl": "https://storage.../receipts/...?token=...",
  "downloadExpiresAt": "2026-06-11T12:30:00.000Z"
}
```

`downloadUrl` is a pre-signed link that expires at `downloadExpiresAt`. Request a fresh link before each download.

---

## Upload Receipt (Headless Deposit)

For headless integrations (no payment widget), use the direct receipt upload endpoint:

```
POST {BASE_URL}/api/v1/merchant/matches/{matchId}/receipt
Content-Type: multipart/form-data
x-api-key: mk_live_...
x-timestamp: ...
x-signature: ...
```

- Field: `file` (PDF, max 3 MB)
- Auth: API key + HMAC
- HMAC: sign the multipart body as `{}` (empty JSON object). `matchId` is in the URL path, which is included in the signature
- Response `202`: `{ "status": "CHECKING", "matchId": "uuid" }`

---

## Deposit Webhooks

| Event | When it fires | Action |
|-------|--------------|--------|
| `order.matched` | Immediately after a successful match (`READY`) | Contains `payerWidgetUrl` |
| `order.completed` | Match settled | **Credit `netAmountRub` to the user** |
| `match.rejected` | Match rejected (timeout, payer cancel, merchant reject) | Notify the user |
| `order.expired` | Deposit order expired (payer timeout) | Offer a new top-up |

See [Webhooks](webhooks.md) for detailed payload structures.
