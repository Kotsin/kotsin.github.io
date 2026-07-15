# API Reference

## Authentication

| Method | Path | Auth |
|--------|------|------|
| `POST` | `/v1/auth/signup` | Public |
| `POST` | `/v1/auth/signup/confirm` | Public |
| `POST` | `/v1/auth/signup/resend-code` | Public |
| `POST` | `/v1/auth/signin` | Public |
| `POST` | `/v1/auth/tokens/refresh` | Public |
| `POST` | `/v1/auth/logout` | Bearer JWT |

---

## Profile

### Get profile

```
GET /v1/merchant/profile
```

```json
{
  "success": true,
  "message": "OK",
  "merchant": {
    "id": "uuid",
    "name": "acme",
    "displayName": "Acme Corp",
    "email": "owner@test.local",
    "additionalData": { "siteUrl": "https://example.com", "logoUrl": null },
    "status": "ACTIVE",
    "ownerUserId": "uuid",
    "createdAt": "...",
    "updatedAt": "..."
  }
}
```

### Update profile

```
PATCH /v1/merchant/profile
```

Owner only. Fields: `name`, `displayName`, `additionalData` (`siteUrl`, `logoUrl`).

---

## Merchant Settings (read-only)

```
GET /v1/merchant/settings
```

```json
{
  "success": true,
  "message": "Merchant settings",
  "settings": {
    "merchantId": "uuid",
    "depositCommissionRate": "3.00",
    "settlementStrategy": "event",
    "settlementTimeIntervalMinutes": null,
    "settlementAmountThresholdUsdt": null,
    "minOrderAmountRub": null,
    "maxOrderAmountRub": null,
    "balanceWarningThresholdUsdt": null,
    "orderCloseCommitmentSeconds": 900,
    "defaultWithdrawalProfileId": "uuid-or-null",
    "inboundEnabled": true,
    "outboundEnabled": true
  }
}
```

---

## API Keys

```
GET    /v1/api-keys                         — list
POST   /v1/api-keys                         — create
GET    /v1/api-keys/:id                     — get by ID
PATCH  /v1/api-keys/:id                     — update
DELETE /v1/api-keys/:id                     — delete
POST   /v1/api-keys/:id/reissue             — reissue
GET    /v1/api-keys/allowed-permissions     — available scopes
```

### Create key

```json
{
  "type": "write",
  "label": "Production",
  "permissions": ["<permission-uuid>"],
  "ttl": "30d"
}
```

| Field | Required | Description |
|-------|----------|-------------|
| `type` | Yes | `read` / `write` / `admin` |
| `ttl` | Yes | `1d`, `7d`, `14d`, `30d`, `60d`, `90d` |
| `label`, `permissions` (UUIDs), `allowedIps`, `twoFaCodes` | No | See Swagger |

---

## Balances

```
GET /v1/merchant/balances
```

```json
{
  "balances": [
    { "currency": "USDT", "available": "45000.00", "reserved": "5000.00" }
  ]
}
```

- `available` — USDT for settlement / free balance
- `reserved` — USDT reserved for active withdrawal holds

---

## Deposit Wallet (USDT Custody)

```
GET    /v1/merchant/deposit/summary              — balance + wallets
GET    /v1/merchant/wallets                       — active deposit wallets
GET    /v1/merchant/deposit/movements             — ledger movements
GET    /v1/merchant/deposit/withdrawal-networks   — supported payout networks
POST   /v1/merchant/deposit/withdraw              — withdraw USDT
GET    /v1/merchant/deposit/withdrawals/:payoutId — payout status
POST   /v1/merchant/create-deposit               — create wallet
```

---

## Orders (Matching)

```
POST /v1/merchant/orders/deposit
POST /v1/merchant/orders/withdrawal
POST /v1/merchant/orders/withdrawals/batch
GET  /v1/merchant/orders/:id
GET  /v1/merchant/orders
```

### Create deposit

```
POST /v1/merchant/orders/deposit
Headers: x-api-key, x-signature, x-timestamp
```

```json
{
  "amount": "1500.00",
  "currency": "RUB",
  "externalOrderId": "order-123",
  "idempotencyKey": "optional-key"
}
```

```json
{
  "depositStatus": "READY",
  "depositOrderId": "uuid",
  "withdrawalOrderId": "uuid",
  "matchId": "uuid",
  "paymentDeadlineAt": "2026-05-22T12:00:00.000Z",
  "payerWidgetUrl": "https://app.example.com/widget/payer-token-abc",
  "paymentDetails": { }
}
```

| `depositStatus` | Meaning |
|-----------------|---------|
| `READY` | Match found; redirect to `payerWidgetUrl` |
| `NO_LIQUIDITY` | No counterpart; order not created |
| `PROCESSING_UNAVAILABLE` | Temporary failure; retry |

### Create withdrawal

```
POST /v1/merchant/orders/withdrawal
Headers: x-api-key, x-signature, x-timestamp
```

```json
{
  "amount": "10000.00",
  "currency": "RUB",
  "externalOrderId": "bl-order-001",
  "profileCode": "SPLIT_2H",
  "paymentDetails": {
    "requisites": "4444 4444 4444 4444",
    "holder": "Ivan I.",
    "bankCode": "sberbank",
    "paymentOption": "TO_CARD"
  },
  "ttlExpiresAt": "2026-06-06T18:00:00.000Z",
  "idempotencyKey": "optional-key"
}
```

| Field | Description |
|-------|------------|
| `profileCode` | Withdrawal profile code; default profile if omitted |
| `paymentDetails` | Recipient details snapshot (stored on the order) |
| `ttlExpiresAt` | Order book TTL; defaults to `first_deposit_wait_seconds` from profile |

```json
{ "orderId": "f47ac10b-58cc-4372-a567-0e02b2c3d479", "message": null }
```

### Batch withdrawal

```
POST /v1/merchant/orders/withdrawals/batch
```

Up to **50 orders** per request. Partial success supported.

```json
{
  "idempotencyKey": "batch-2026-06-06-001",
  "orders": [
    {
      "amount": "5000.00",
      "currency": "RUB",
      "externalOrderId": "bl-order-002",
      "profileCode": "MONO_1H",
      "paymentDetails": {
        "requisites": "+79001234567",
        "holder": "Petr P.",
        "bankCode": "tinkoff",
        "paymentOption": "SBP"
      }
    }
  ]
}
```

```json
{
  "accepted": 1,
  "rejected": 0,
  "results": [
    { "externalOrderId": "bl-order-002", "orderId": "550e8400-..." }
  ]
}
```

### List orders

```
GET /v1/merchant/orders?page=1&limit=20&type=DEPOSIT&status=...
```

Supports `search`, `dateFrom`, `dateTo`, `amountFrom`, `amountTo`.

---

## Matches

```
GET  /v1/merchant/matches                   — list
GET  /v1/merchant/matches/:id               — detail
POST /v1/merchant/matches/:id/confirm       — confirm
POST /v1/merchant/matches/:id/reject        — reject
POST /v1/merchant/matches/:id/dispute       — dispute
POST /v1/merchant/matches/:id/receipt       — upload receipt
```

### List matches

```
GET /v1/merchant/matches?status=AWAITING_RECIPIENT&page=1&limit=20
```

`displayStatus`: PENDING, AWAITING_RECIPIENT, CONFIRMED, DISPUTED, EXPIRED.

### Confirm / Reject / Dispute

Auth: JWT (UI) or API key + HMAC (integration).

```
POST /v1/merchant/matches/:id/confirm   {}
POST /v1/merchant/matches/:id/reject    { "reason": "..." }
POST /v1/merchant/matches/:id/dispute   { "reason": "..." }
```

### Upload receipt (headless deposit)

```
POST /v1/merchant/matches/:id/receipt
Content-Type: multipart/form-data
```

Field: `file` (PDF, max 3 MB). Auth: API key + HMAC.

Response `202`:
```json
{ "status": "CHECKING", "matchId": "uuid" }
```

HMAC: sign multipart body as `{}` (empty JSON object).

---

## Settlements

```
GET /v1/merchant/settlements                — list batches
GET /v1/merchant/settlements/:batchId       — batch detail
```

---

## Documents (Receipts)

```
GET /v1/merchant/documents                  — list
GET /v1/merchant/documents/:id/download     — download
```

Download returns a pre-signed `downloadUrl` with expiry.

---

## Webhooks

```
POST   /v1/webhooks                         — register
GET    /v1/webhooks                         — list
PATCH  /v1/webhooks/:id                     — update
DELETE /v1/webhooks/:id                     — delete
POST   /v1/webhooks/:id/status              — activate/deactivate
GET    /v1/webhook-events                   — delivery log
POST   /v1/webhook-events/:deliveryId/resend — resend
```

---

## Dashboard

```
GET /v1/merchant/dashboard/summary
```

```json
{
  "balances": {
    "totalUsdt": "48320",
    "availableUsdt": "31080",
    "reservedUsdt": "17240"
  },
  "alertThresholdUsdt": "5000",
  "activeDepositsCount": 12,
  "activeWithdrawalsCount": 7,
  "pendingMatchesCount": 4,
  "tailsCount": 2
}
```

---

## Market Data

```
GET /v1/market-data/rates/rub-usdt
```

```json
{
  "data": {
    "baseCurrency": "RUB",
    "targetCurrency": "USDT",
    "rate": 95.42,
    "lastUpdated": "2026-06-11T00:00:00.000Z",
    "outdatedMinutes": 0,
    "source": "coingecko"
  }
}
```
