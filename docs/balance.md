# Balance & Rate

---

## USDT Balance

To participate in matching and settlement, the merchant maintains a USDT balance on the platform.

### Create a deposit wallet

```
POST {BASE_URL}/api/v1/merchant/create-deposit
x-api-key: mk_live_...
x-timestamp: ...
x-signature: ...
Content-Type: application/json

{ "networkType": "EVM" }
```

Currently **EVM networks only** (e.g. Polygon). The response contains the wallet address for USDT deposits.

### Balance summary

```
GET {BASE_URL}/api/v1/merchant/deposit/summary
x-api-key: mk_live_...
x-timestamp: ...
x-signature: ...
```

```json
{
  "timestamp": 1717651200000,
  "path": "/api/v1/merchant/deposit/summary",
  "method": "GET",
  "statusCode": 200,
  "success": true,
  "balances": {
    "totalUsdt": "48320.00000000",
    "availableUsdt": "31080.00000000",
    "reservedUsdt": "17240.00000000"
  },
  "alertThresholdUsdt": "5000.000000",
  "wallets": [
    {
      "network": "POLYGON",
      "networkType": "EVM",
      "walletId": "uuid",
      "depositAddress": "0x...",
      "walletType": "DEPOSIT",
      "status": "ACTIVE",
      "createdAt": "2026-01-01T00:00:00.000Z"
    }
  ]
}
```

| Field | Description |
|-------|------------|
| `totalUsdt` | Total balance |
| `availableUsdt` | Free balance for settlement and withdrawal |
| `reservedUsdt` | Reserved for active withdrawal holds |
| `alertThresholdUsdt` | Low balance warning threshold |

### Current balances (simplified)

```
GET {BASE_URL}/api/v1/merchant/balances
Authorization: Bearer {jwt}
```

```json
{
  "balances": [
    { "currency": "USDT", "available": "45000.00", "reserved": "5000.00" }
  ]
}
```

### Movement history

```
GET {BASE_URL}/api/v1/merchant/deposit/movements?page=1&limit=20
x-api-key: mk_live_...
x-timestamp: ...
x-signature: ...
```

| `type` | Meaning |
|--------|---------|
| `DEPOSIT` | USDT balance top-up |
| `WITHDRAWAL` | USDT withdrawal |
| `SETTLEMENT_IN` | Settlement credit |
| `SETTLEMENT_OUT` | Settlement debit |

---

## USDT Withdrawal

### Get withdrawal networks

```
GET {BASE_URL}/api/v1/merchant/deposit/withdrawal-networks
x-api-key: mk_live_...
x-timestamp: ...
x-signature: ...
```

```json
{
  "networks": [
    { "name": "POLYGON", "coinSymbol": "USDT" },
    { "name": "ETHEREUM", "coinSymbol": "USDT" },
    { "name": "ARBITRUM", "coinSymbol": "USDT" },
    { "name": "AVALANCHE", "coinSymbol": "USDT" },
    { "name": "BINANCE", "coinSymbol": "USDT" }
  ]
}
```

### Withdraw USDT

```
POST {BASE_URL}/api/v1/merchant/deposit/withdraw
x-api-key: mk_live_...
x-timestamp: ...
x-signature: ...
Content-Type: application/json
```

```json
{
  "address": "0xAbC1230000000000000000000000000000000000",
  "network": "POLYGON",
  "amountUsdt": "100.50",
  "idempotencyKey": "req_abc123"
}
```

| Field | Description |
|-------|------------|
| `address` | Destination address |
| `network` | Network name from `withdrawal-networks` |
| `amountUsdt` | USDT amount, up to 8 decimal places |
| `idempotencyKey` | Optional; server-generated if omitted |

**Response 201:**

```json
{
  "payoutId": "uuid",
  "status": "PROCESSING",
  "amountUsdt": "100.50",
  "network": "POLYGON",
  "destinationAddress": "0xAbC..."
}
```

Errors: `LDG_004` insufficient balance; `LDG_007` idempotency conflict; `LDG_008` payout creation failed.

### Get payout status

```
GET {BASE_URL}/api/v1/merchant/deposit/withdrawals/{payoutId}
x-api-key: mk_live_...
x-timestamp: ...
x-signature: ...
```

```json
{
  "payoutId": "uuid",
  "status": "PROCESSING",
  "amountUsdt": "100.50",
  "network": "POLYGON",
  "destinationAddress": "0xAbC...",
  "txHash": null,
  "errorMessage": null,
  "createdAt": "2026-06-01T10:00:00.000Z",
  "updatedAt": "2026-06-01T10:00:00.000Z"
}
```

| `status` | Meaning |
|----------|---------|
| `PROCESSING` | Awaiting custody; continue polling |
| `COMPLETED` | Success; `txHash` available |
| `FAILED` | Error; check `errorMessage` |

**Polling:** poll every **5 seconds** until `COMPLETED` or `FAILED`.

---

## RUB/USDT Rate

```
GET {BASE_URL}/api/v1/market-data/rates/rub-usdt
x-api-key: mk_live_...
x-timestamp: ...
x-signature: ...
```

```json
{
  "timestamp": 1717651200000,
  "path": "/api/v1/market-data/rates/rub-usdt",
  "method": "GET",
  "statusCode": 200,
  "success": true,
  "data": {
    "baseCurrency": "RUB",
    "targetCurrency": "USDT",
    "rate": 95.42,
    "lastUpdated": "2026-06-11T00:00:00.000Z",
    "outdatedMinutes": 0,
    "source": "coingecko"
  },
  "message": null
}
```

| Field | Description |
|-------|------------|
| `data.rate` | Number of rubles per 1 USDT |
| `data.outdatedMinutes` | Minutes since last update |
| `data.source` | `coingecko` / `cmc` / `dev_static` (sandbox) |

Conversion: `amountUsdt = amountRub / rate`
