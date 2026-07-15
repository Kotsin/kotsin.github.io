# Error Codes

---

## Error response format

```json
{
  "timestamp": "2026-06-11T10:30:00.000Z",
  "path": "/api/v1/merchant/orders/deposit",
  "method": "POST",
  "errorCode": "ORD_001",
  "statusCode": 400,
  "success": false,
  "message": "Invalid order input",
  "details": { "fields": [ ] }
}
```

| Field | Description |
|-------|------------|
| `errorCode` | Logical error code for programmatic handling |
| `statusCode` | HTTP status code |
| `message` | Brief user-facing description |
| `details` | Additional context (e.g. validation `fields`) |

**Recommendation:** use `errorCode` for programmatic handling, not `message`.

---

## Common error codes

| HTTP | `errorCode` | Cause | Action |
|------|-------------|-------|--------|
| 400 | `ORD_001` | Invalid order input (amount, missing fields, unknown profile) | Fix request body |
| 400 | `API_KEY_015` | Invalid HMAC signature | Review [HMAC signing](authentication.md#request-signing-hmac) |
| 403 | `AUTH_600` | Access denied (permission, IP allowlist, invalid key) | Check key permissions and IP |
| 403 | `ORD_006` | Merchant outbound capability disabled | Contact support |
| 404 | `ORD_002` | Order or match not found | Check the ID |
| 409 | `ORD_005` | Duplicate `externalOrderId` on open withdrawal | Use a different ID |
| 429 | `AUTH_001` | Rate limit exceeded | Back off with jitter |
| 500 | — | Internal platform error | Retry later |

---

## Additional error codes

| HTTP | Code | Description |
|------|------|-------------|
| 400 | `INVALID_AMOUNT` | Invalid amount |
| 400 | `INVALID_CURRENCY` | Unsupported currency |
| 400 | `ORDER_ALREADY_MATCHED` | Cannot cancel a matched order |
| 402 | `INSUFFICIENT_DEPOSIT` | Insufficient USDT balance |
| 403 | `MERCHANT_BLOCKED` | Merchant account blocked |
| 422 | `ORD_004` | Match not in correct status for operation |

---

## Ledger error codes

| HTTP | Code | Description |
|------|------|-------------|
| 400 | `LDG_004` | Insufficient USDT balance for withdrawal |
| 409 | `LDG_007` | Idempotency conflict |
| 500 | `LDG_008` | Payout creation failed |
| 404 | `LDG_001` | Payout not found or belongs to another merchant |

---

## Retry strategy

For `429` and `5xx`, apply **exponential backoff with jitter**:

```
delay = min(baseDelay * 2^attempt, maxDelay)
jitter = random(0, delay * 0.1)
sleep(delay + jitter)
```

| Parameter | Value |
|-----------|-------|
| baseDelay | 1 second |
| maxDelay | 60 seconds |
| maxAttempts | 5 |
