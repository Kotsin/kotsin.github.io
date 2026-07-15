# Webhooks

---

## Overview

All webhooks are delivered via `POST` to the registered merchant URL.

### Request headers

```
Content-Type: application/json
x-webhook-signature: <base64>
x-timestamp: 1713178800
x-delivery-id: <delivery-uuid>
x-webhook-event: order.matched
```

### Base payload structure

```json
{
  "event": "order.matched",
  "deliveryId": "uuid",
  "timestamp": "2024-01-01T00:05:00Z",
  "merchantId": "uuid",
  "data": { }
}
```

The HTTP body sent to your endpoint is the full JSON object above.

---

## Setup and Management

Webhook management uses JWT (dashboard) or API key + HMAC. New webhooks are created in an **inactive** state — activate after configuring your endpoint.

### Register a webhook

```
POST {BASE_URL}/api/v1/webhooks
Authorization: Bearer {jwt}
Content-Type: application/json
```

```json
{
  "label": "Main events",
  "url": "https://your-server.com/platform-webhook",
  "events": [
    "order.matched",
    "order.completed",
    "order.expired",
    "match.rejected",
    "order.created",
    "payment.uploaded",
    "payment.disputed",
    "order.tail_created",
    "order.tail_closed"
  ]
}
```

The response returns the webhook `id` and **`hmacSecret`** (prefix `whsec_...`) for verifying incoming deliveries.

### Managing webhooks

| Action | Endpoint |
|--------|----------|
| List webhooks | `GET /api/v1/webhooks` |
| Get by ID | `GET /api/v1/webhooks/{id}` |
| Update (`label`, `events`) | `PATCH /api/v1/webhooks/{id}` |
| Activate / deactivate | `POST /api/v1/webhooks/{id}/status` with `{ "isActive": true }` |
| Delete | `DELETE /api/v1/webhooks/{id}` |

### Delivery history

```
GET {BASE_URL}/api/v1/webhook-events?webhookId={id}&limit=20
Authorization: Bearer {jwt}
```

### Resend a delivery

```
POST {BASE_URL}/api/v1/webhook-events/{deliveryId}/resend
Authorization: Bearer {jwt}
```

---

## Signature Verification

### Algorithm

1. Read the raw request body as a string (`rawBody`) **before** JSON parsing.
2. Read `x-timestamp` (Unix time in **seconds**).
3. Build canonical string: `` `${timestamp}\n${rawBody}` ``
4. Compute:

```
payloadHash = SHA256(canonicalString)
expected    = Base64( HMAC-SHA256(webhookSecret, payloadHash) )
```

5. Compare `expected` to `x-webhook-signature` using a timing-safe comparison.

### Example (Node.js)

```js
const crypto = require('crypto');

function verifyWebhook({ rawBody, signatureHeader, timestampHeader, webhookSecret }) {
  const canonical = `${timestampHeader}\n${rawBody}`;
  const payloadHash = crypto.createHash('sha256').update(canonical).digest();
  const expected = crypto
    .createHmac('sha256', webhookSecret)
    .update(payloadHash)
    .digest('base64');

  return crypto.timingSafeEqual(
    Buffer.from(signatureHeader),
    Buffer.from(expected),
  );
}
```

### Recommendations

- Respond `200 OK` quickly; offload heavy processing to a queue
- Ensure idempotency — use `deliveryId` from the payload; retries may duplicate deliveries
- Never trust the payload without verifying the signature

---

## Retry Policy

Webhook deliveries retry on non-2xx responses or transport errors, with exponential backoff:

| Setting | Value |
|---------|-------|
| Max attempts | 7 |
| Base delay | 5 minutes |
| Multiplier | 3x per attempt |
| Max delay cap | 2 days |

Approximate schedule: immediately, ~5 min, ~15 min, ~45 min, ~2.25 h, ~6.75 h, ~20 h, then cap at 2 days. After all attempts the delivery is marked failed.

---

## Full Event Reference

### Order events

#### `order.created`

WITHDRAWAL order created and placed in the book (push API).

```json
{
  "event": "order.created",
  "deliveryId": "order-created:f47ac10b-58cc-4372-a567-0e02b2c3d479",
  "timestamp": "2026-06-06T10:00:00.000Z",
  "merchantId": "uuid",
  "data": {
    "orderId": "f47ac10b-58cc-4372-a567-0e02b2c3d479",
    "externalOrderId": "withdrawal-001",
    "type": "WITHDRAWAL",
    "currency": "RUB",
    "amount": "10000.00",
    "status": "PENDING",
    "ingestionSource": "PUSH",
    "createdAt": "2026-06-06T10:00:00.000Z"
  }
}
```

#### `order.matched`

Order matched with a counterpart (both merchant types).

```json
{
  "event": "order.matched",
  "data": {
    "orderId": "uuid-order",
    "externalOrderId": "your-order-123",
    "type": "DEPOSIT",
    "currency": "RUB",
    "totalAmount": "1000.00",
    "matchedAmount": "1000.00",
    "match": {
      "matchId": "uuid-match",
      "amount": "1000.00",
      "payerWidgetUrl": "https://platform/widget/payer-token-abc",
      "recipientWidgetUrl": "",
      "recipientDetails": {
        "bank": "Tinkoff",
        "accountNumber": "40817810...",
        "recipientName": "Petrov P.P."
      },
      "expiresAt": "2024-01-01T00:15:00Z"
    }
  }
}
```

- Merchant A (DEPOSIT): receives `payerWidgetUrl` for redirecting the end user
- Merchant B (WITHDRAWAL): `recipientWidgetUrl` is empty in MVP; confirmation via LK or auto-confirm

#### `order.completed`

All matches confirmed. Settlement initiated.

```json
{
  "event": "order.completed",
  "data": {
    "orderId": "uuid-order",
    "externalOrderId": "your-order-123",
    "type": "DEPOSIT",
    "currency": "RUB",
    "totalAmount": "1000.00",
    "netAmountRub": "970.00",
    "commissionAmountRub": "30.00",
    "commissionRate": "3.00",
    "completedAt": "2024-01-01T00:12:00Z"
  }
}
```

**Action for deposit merchant:** credit `netAmountRub` to the user.

#### `order.cancelled`

Order explicitly cancelled (merchant, operator, or dispute resolution).

```json
{
  "event": "order.cancelled",
  "data": {
    "orderId": "uuid-order",
    "externalOrderId": "your-order-123",
    "reason": "MANUAL",
    "cancelledAt": "2024-01-01T00:10:00Z"
  }
}
```

`reason` values: `MANUAL` | `INSUFFICIENT_DEPOSIT` | `DISPUTE_RESOLVED` | `OPERATOR_FORCED`

#### `order.expired`

Order expired by TTL or payer timeout.

```json
{
  "event": "order.expired",
  "data": {
    "orderId": "uuid-order",
    "externalOrderId": "your-order-123",
    "type": "DEPOSIT",
    "amountMatched": "0.00",
    "expiredAt": "2024-01-01T00:20:00Z",
    "reason": "PAYER_TIMEOUT"
  }
}
```

| `type` | When |
|--------|------|
| `DEPOSIT` | Payer did not pay before `paymentDeadlineAt` |
| `WITHDRAWAL` | TTL expired in order book without matches |

#### `order.tail_created`

WITHDRAWAL order partially matched — remainder became a tail.

```json
{
  "event": "order.tail_created",
  "data": {
    "orderId": "uuid-order",
    "externalOrderId": "your-order-123",
    "originalAmount": "1000.00",
    "matchedAmount": "600.00",
    "tailAmount": "400.00",
    "currency": "RUB",
    "tailCreatedAt": "2024-01-01T01:00:00Z"
  }
}
```

#### `order.tail_closed`

Tail closed by operator.

```json
{
  "event": "order.tail_closed",
  "data": {
    "orderId": "uuid-order",
    "tailAmount": "400.00",
    "closedBy": "OPERATOR",
    "closedAt": "2024-01-01T02:00:00Z"
  }
}
```

### Match events

#### `match.rejected`

Match rejected before completion. Sent to **both** merchants.

```json
{
  "event": "match.rejected",
  "data": {
    "matchId": "uuid-match",
    "depositOrderId": "uuid-deposit",
    "withdrawalOrderId": "uuid-withdrawal",
    "reason": "PAYER_TIMEOUT",
    "rejectedAt": "2024-01-01T00:20:00Z"
  }
}
```

`reason` values: `PAYER_TIMEOUT` | `PAYER_CANCELLED` | `OUTBOUND_REJECT` | `OPERATOR_CANCEL` | `CHECKER_INVALID`

### Payment events

#### `payment.uploaded`

Payer uploaded receipt, ScamChecker passed, match → `CONFIRMING`. Sent after async checker.

```json
{
  "event": "payment.uploaded",
  "deliveryId": "payment-uploaded:a1b2c3d4-...",
  "timestamp": "2026-06-06T12:11:30.000Z",
  "merchantId": "outbound-merchant-uuid",
  "data": {
    "matchId": "a1b2c3d4-...",
    "orderId": "withdrawal-order-uuid",
    "externalOrderId": "bl-order-001",
    "amount": "5000.00",
    "currency": "RUB",
    "recipientWidgetUrl": "",
    "confirmTimeoutAt": "2026-06-06T12:26:30.000Z"
  }
}
```

| Field | Note |
|-------|------|
| `confirmTimeoutAt` | `now + outbound_auto_confirm_seconds` from the withdrawal profile |

**Target:** outbound merchant. Notify the recipient to verify the incoming transfer. Confirm via API before `confirmTimeoutAt`.

#### `payment.confirmed`

Recipient confirmed funds (or auto-confirm fired).

```json
{
  "event": "payment.confirmed",
  "data": {
    "matchId": "uuid-match",
    "orderId": "uuid-order",
    "amount": "1000.00",
    "currency": "RUB",
    "confirmedBy": "USER",
    "confirmedAt": "2024-01-01T00:10:00Z"
  }
}
```

`confirmedBy`: `USER` | `AUTO_TIMEOUT`

#### `payment.disputed`

Match disputed. Settlement frozen.

```json
{
  "event": "payment.disputed",
  "data": {
    "matchId": "uuid-match",
    "orderId": "uuid-order",
    "amount": "1000.00",
    "currency": "RUB",
    "disputedBy": "RECIPIENT",
    "disputeReason": "Funds not received",
    "disputedAt": "2024-01-01T00:09:00Z"
  }
}
```

`disputedBy`: `PAYER` | `RECIPIENT`

### Settlement events

#### `settlement.batched`

Settlement batch created (for TIME or AMOUNT strategies).

```json
{
  "event": "settlement.batched",
  "data": {
    "batchId": "uuid-batch",
    "strategy": "TIME",
    "entriesCount": 12,
    "totalAmountRub": "48500.00",
    "createdAt": "2024-01-01T06:00:00Z"
  }
}
```

#### `settlement.completed`

Settlement applied, balances adjusted.

```json
{
  "event": "settlement.completed",
  "data": {
    "batchId": "uuid-batch",
    "strategy": "EVENT",
    "entriesCount": 1,
    "totalAmountRub": "1000.00",
    "totalAmountUsdt": "11.111",
    "exchangeRate": "90.00",
    "rateSource": "binance",
    "merchantDeltaUsdt": "-11.111",
    "newBalanceUsdt": "48888.889",
    "completedAt": "2024-01-01T00:13:00Z"
  }
}
```

### Custody events

#### `deposit.confirmed`

USDT deposit credited from crypto processing.

```json
{
  "event": "deposit.confirmed",
  "data": {
    "amountUsdt": "10000.00",
    "txHash": "0xabc...",
    "newBalanceUsdt": "60000.00",
    "confirmedAt": "2024-01-01T00:00:00Z"
  }
}
```

#### `withdrawal.payout_completed`

Custody payout completed successfully.

```json
{
  "event": "withdrawal.payout_completed",
  "data": {
    "payoutId": "uuid",
    "amountUsdt": "100.50",
    "destinationAddress": "0xAbC...",
    "network": "POLYGON",
    "txHash": "0xdead...",
    "completedAt": "2024-01-01T00:00:00Z"
  }
}
```

#### `withdrawal.payout_failed`

Custody payout failed (reserve returned to available).

```json
{
  "event": "withdrawal.payout_failed",
  "data": {
    "payoutId": "uuid",
    "amountUsdt": "100.50",
    "destinationAddress": "0xAbC...",
    "network": "POLYGON",
    "errorMessage": "Address blacklisted",
    "failedAt": "2024-01-01T00:00:00Z"
  }
}
```

### Wallet events (when KYT is enabled)

#### `merchant.wallet_blocked`

KYT blocked a deposit address.

```json
{
  "event": "merchant.wallet_blocked",
  "data": {
    "blockedAddress": "TXYZabc...",
    "reason": "KYT_RISK_SCORE_HIGH",
    "blockedAt": "2024-01-01T00:00:00Z"
  }
}
```

#### `merchant.wallet_reissued`

Deposit address reissued after block or by request.

```json
{
  "event": "merchant.wallet_reissued",
  "data": {
    "previousAddress": "TXYZabc...",
    "newAddress": "TNEW123...",
    "reason": "KYT_RISK_SCORE_HIGH",
    "reissuedAt": "2024-01-01T00:01:00Z"
  }
}
```

---

## Full Event List for Subscription

```
order.created          — WITHDRAWAL order created and placed in the book (push API)
order.matched          — order matched with a counterpart (both merchant types)
order.completed        — order settled after match confirmation
order.cancelled        — order explicitly cancelled
order.expired          — order expired (payer timeout or withdrawal TTL)
order.tail_created     — WITHDRAWAL remainder became a tail at TTL
order.tail_closed      — tail closed by the operator
match.rejected         — match rejected before completion
payment.uploaded       — receipt verified; outbound merchant should confirm
payment.confirmed      — recipient confirmed funds
payment.disputed       — dispute opened on a match
settlement.batched     — settlement batch created
settlement.completed   — settlement applied
deposit.confirmed      — USDT deposit confirmed
merchant.wallet_blocked   — KYT blocked a wallet
merchant.wallet_reissued  — wallet reissued
withdrawal.payout_completed — custody payout completed
withdrawal.payout_failed    — custody payout failed
```
