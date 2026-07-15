# Crypton Merchant API

**Version:** 1.1 · **Date:** June 2026

---

## Overview

The platform provides P2P fiat payment processing. It works on a matching model: inbound deposit orders are automatically matched with outbound withdrawal orders from another participant.

Two merchant types are supported:

| Type | Role | Key API |
|------|------|---------|
| **Inbound merchant (Deposit)** | Accepts ruble payments from end users | `POST /merchant/orders/deposit` |
| **Outbound merchant (Withdrawal)** | Pays out funds to end users | `POST /merchant/orders/withdrawal` |

A single merchant account may act in both roles simultaneously.

### Base URL

| Environment | Address |
|-------------|---------|
| Sandbox / Test | `https://dvspayment.co` |
| Production | Provided by the platform team upon request |

All API paths use the prefix `/api/v1`. Examples use `{BASE_URL}` as a placeholder.

All amounts are transmitted and returned as **strings** (decimal), for example `"1500.00"`. Do not convert them to floating-point numbers — this leads to precision loss.

### API response envelope

Successful HTTP responses are wrapped by the gateway:

```json
{
  "timestamp": 1717651200000,
  "path": "/api/v1/merchant/orders/deposit",
  "method": "POST",
  "statusCode": 201,
  "success": true,
  "depositStatus": "READY",
  "depositOrderId": "..."
}
```

Business fields (`depositStatus`, `order`, `apiKey`, etc.) are returned alongside the envelope fields.

### Integration checklist

- [ ] Account created, merchant profile filled in the dashboard
- [ ] API key created; `key` and `hmacSecret` saved in secure storage
- [ ] IP allowlist configured on the key
- [ ] HMAC request signing implemented for all API-key calls
- [ ] Webhooks registered and activated
- [ ] Sufficient USDT balance maintained

---

## Glossary

| Term | Definition |
|------|-----------|
| **Merchant** | A platform integrating the API |
| **Inbound merchant** | A merchant accepting top-ups from their users |
| **Outbound merchant** | A merchant paying out funds to their users |
| **End user** | The final client initiating a payment |
| **Order** | A deposit (`DEPOSIT`) or withdrawal (`WITHDRAWAL`) request |
| **Match** | A pairing of a deposit order with a withdrawal order |
| **Tail** | The unmatched remainder of an order after partial matching |
| **Payment widget** | A platform-hosted payment page (`payerWidgetUrl`) for the end user |
| **Settlement** | Mutual settlement between merchants in USDT within the platform |
| **USDT balance** | A merchant's custodial account on the platform |
| **Idempotency** | A repeated request with the same `idempotencyKey` does not create a duplicate |
| **HMAC signature** | A request signature computed on `hmacSecret`, sent in `x-signature` |
| **API key** | An integration credential (`x-api-key`) with an associated permission set |
