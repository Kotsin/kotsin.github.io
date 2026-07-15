# Withdrawal (Outbound Merchant)

---

## How Withdrawal Processing Works

The outbound merchant places withdrawal orders into the platform order book. The platform matches them with incoming deposits.

### Happy path

1. The merchant creates a withdrawal order (`POST /merchant/orders/withdrawal`) with recipient payment details
2. The platform confirms acceptance → webhook `order.created`
3. The platform finds a counterpart deposit → webhook `order.matched`
4. The inbound user transfers money to the recipient requisites from the order
5. The platform verifies the receipt → webhook `payment.uploaded`
6. The recipient confirms via `POST /merchant/matches/{id}/confirm`, or the match is **auto-confirmed** after `confirmTimeoutAt`
7. Settlement runs → webhook `order.completed`

### Edge cases

| Situation | Behavior |
|-----------|---------|
| No match within TTL | Webhook `order.expired` |
| Partial match when TTL expires | Webhook `order.tail_created` (remainder becomes a tail) |
| Tail closed by operator | Webhook `order.tail_closed` |

---

## Creating a Single Withdrawal Order

```
POST {BASE_URL}/api/v1/merchant/orders/withdrawal
x-api-key: mk_live_...
x-timestamp: 1717651200000
x-signature: <base64>
Content-Type: application/json
```

### Request body

| Field | Type | Required | Description |
|-------|------|:---:|---------|
| `amount` | string (decimal) | Yes | Withdrawal amount in rubles |
| `currency` | string | — | Defaults to `RUB` |
| `externalOrderId` | string | Yes | Your order ID; must be unique among open orders |
| `paymentDetails` | object | Yes | Recipient payment details |
| `profileCode` | string | — | Withdrawal profile code; defaults to the merchant's default profile |
| `ttlExpiresAt` | ISO 8601 | — | Order book TTL; defaults to the value from the profile |
| `idempotencyKey` | string | — | Idempotency key |

### `paymentDetails` fields

| Field | Description |
|-------|------------|
| `requisites` | Card number or phone number |
| `holder` | Recipient name |
| `bankCode` | Bank code, e.g. `sberbank`, `tinkoff`, `alfabank` |
| `paymentOption` | `TO_CARD` (card transfer) or `SBP` (Faster Payments System) |

```json
{
  "amount": "10000.00",
  "currency": "RUB",
  "externalOrderId": "withdrawal-001",
  "profileCode": "SPLIT_2H",
  "paymentDetails": {
    "requisites": "4444 4444 4444 4444",
    "holder": "Ivan Ivanov",
    "bankCode": "sberbank",
    "paymentOption": "TO_CARD"
  },
  "ttlExpiresAt": "2026-06-07T18:00:00.000Z",
  "idempotencyKey": "idem-w-001"
}
```

### Response (201)

```json
{
  "timestamp": 1717651200000,
  "path": "/api/v1/merchant/orders/withdrawal",
  "method": "POST",
  "statusCode": 201,
  "success": true,
  "orderId": "f47ac10b-58cc-4372-a567-0e02b2c3d479",
  "message": null
}
```

### Errors

| HTTP | `errorCode` | Cause |
|------|-------------|-------|
| 400 | `ORD_001` | Invalid input (missing `externalOrderId`, bad amount, unknown `profileCode`) |
| 403 | `ORD_006` | Outbound capability disabled for merchant |
| 409 | `ORD_005` | An open order with this `externalOrderId` already exists |

---

## Batch Withdrawal Orders

Up to **50 orders** per request. Each is processed independently — partial success is allowed.

```
POST {BASE_URL}/api/v1/merchant/orders/withdrawals/batch
x-api-key: mk_live_...
x-timestamp: 1717651200000
x-signature: <base64>
Content-Type: application/json
```

```json
{
  "idempotencyKey": "batch-2026-06-06-001",
  "orders": [
    {
      "amount": "5000.00",
      "currency": "RUB",
      "externalOrderId": "withdrawal-002",
      "profileCode": "MONO_1H",
      "paymentDetails": {
        "requisites": "+79001234567",
        "holder": "Petr Petrov",
        "bankCode": "tinkoff",
        "paymentOption": "SBP"
      }
    }
  ]
}
```

### Response — partial failure (201)

```json
{
  "timestamp": 1717651200000,
  "path": "/api/v1/merchant/orders/withdrawals/batch",
  "method": "POST",
  "statusCode": 201,
  "success": true,
  "accepted": 1,
  "rejected": 1,
  "results": [
    { "externalOrderId": "withdrawal-002", "orderId": "uuid-1" },
    {
      "externalOrderId": "withdrawal-003",
      "error": "Open withdrawal with this externalOrderId already exists",
      "errorCode": "ORD_005"
    }
  ]
}
```

---

## Match Management

After receiving `payment.uploaded`, the outbound merchant can confirm, reject, or dispute a match. All endpoints require API key + HMAC headers.

### Confirm a match

```
POST {BASE_URL}/api/v1/merchant/matches/{matchId}/confirm
x-api-key: mk_live_...
x-timestamp: ...
x-signature: ...
Content-Type: application/json

{}
```

Optional body: `{ "comment": "Funds received" }`

### Reject a match

```
POST {BASE_URL}/api/v1/merchant/matches/{matchId}/reject
x-api-key: mk_live_...
x-timestamp: ...
x-signature: ...
Content-Type: application/json

{ "reason": "Funds not received" }
```

### Open a dispute

```
POST {BASE_URL}/api/v1/merchant/matches/{matchId}/dispute
x-api-key: mk_live_...
x-timestamp: ...
x-signature: ...
Content-Type: application/json

{ "reason": "Receipt appears to be fraudulent" }
```

### Match statuses (outbound)

| `displayStatus` | Meaning | Available actions |
|-----------------|---------|------------------|
| `PENDING` | Awaiting payment from the inbound user | — |
| `AWAITING_RECIPIENT` | Receipt uploaded; awaiting your decision | confirm / reject / dispute |
| `CONFIRMED` | Match confirmed | — |
| `DISPUTED` | Dispute opened | — |
| `EXPIRED` | Expired | — |

If no action is taken by `confirmTimeoutAt` from the `payment.uploaded` webhook, the platform auto-confirms the match.

---

## Withdrawal Webhooks

| Event | When it fires | Action |
|-------|--------------|--------|
| `order.created` | Order accepted and placed in the book | Save `orderId` |
| `order.matched` | Counterpart deposit found | Review `recipientDetails` in match payload |
| `payment.uploaded` | Receipt uploaded and verified | **Notify the recipient to verify the transfer** |
| `payment.disputed` | Dispute opened | Suspend processing |
| `order.completed` | Order settled | — |
| `order.expired` | TTL expired without a full match | Create a new order if needed |
| `order.tail_created` | Order partially matched; remainder is a tail | Notify operations |
| `order.tail_closed` | Tail closed by the operator | — |
| `match.rejected` | Match rejected | — |

See [Webhooks](webhooks.md) for detailed payload structures.
