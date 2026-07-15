# Authentication

All machine-to-machine API calls must include an **API key** and **HMAC signature**.

---

## API Keys

API keys are created and managed from the merchant dashboard (JWT session) or via the API with a Bearer token.

### Create a key

```
POST {BASE_URL}/api/v1/api-keys
Authorization: Bearer {jwt}
Content-Type: application/json
```

```json
{
  "label": "Integration key",
  "type": "write",
  "allowedIps": ["203.0.113.10"],
  "permissions": ["<permission-uuid-1>", "<permission-uuid-2>"],
  "ttl": "90d"
}
```

| Field | Description |
|-------|-------------|
| `label` | An arbitrary label for the key |
| `type` | `read` / `write` / `admin` |
| `allowedIps` | IP allowlist (IPv4; recommended for production) |
| `permissions` | Array of permission **UUIDs** from `GET /api/v1/api-keys/allowed-permissions` |
| `ttl` | Key lifetime: `1d`, `7d`, `14d`, `30d`, `60d`, `90d` |

### Minimum permission scopes

| Integration | Required scopes |
|-------------|-----------------|
| Inbound (deposit) | `order:deposit:create`, `order:get`, `order:list` |
| Outbound (withdrawal) | `order:withdrawal:create`, `order:withdrawals:batch:create`, `order:get`, `order:list` |
| Match actions (outbound) | `order:merchant:match:confirm`, `order:merchant:match:reject`, `order:merchant:match:dispute`, `order:merchant:matches:get`, `order:merchant:matches:list` |
| Documents | `order:merchant:documents:list`, `order:merchant:documents:get` |
| Webhooks | `webhook:create`, `webhook:list`, `webhook:get:by:id`, `webhook:update`, `webhook:set:status`, `webhook:delete` |
| USDT wallet | `merchant:profile:create-deposit-wallet`, deposit read scopes |

Resolve UUIDs via `GET /api/v1/api-keys/allowed-permissions?fetchAll=true`.

### Create key response

```json
{
  "timestamp": 1717651200000,
  "path": "/api/v1/api-keys",
  "method": "POST",
  "statusCode": 201,
  "success": true,
  "message": "API key created",
  "apiKey": {
    "id": "uuid",
    "label": "Integration key",
    "key": "mk_live_abc...xyz",
    "type": "write",
    "allowedIps": ["203.0.113.10"],
    "permissions": ["order:deposit:create", "order:get"],
    "status": "ACTIVE",
    "hmacSecret": "s3cr3t...long_string",
    "expiredAt": "2026-09-01T00:00:00.000Z",
    "createdAt": "2026-06-01T00:00:00.000Z"
  }
}
```

`key` and `hmacSecret` are shown **only once** at creation (or reissue). Save them immediately. `hmacSecret` cannot be recovered — use reissue to rotate.

### Managing keys

| Action | Endpoint |
|--------|----------|
| List keys | `GET /api/v1/api-keys` |
| Get key by ID | `GET /api/v1/api-keys/{id}` |
| Update (label, IPs, permissions, active flag) | `PATCH /api/v1/api-keys/{id}` |
| Reissue key | `POST /api/v1/api-keys/{id}/reissue` |
| Delete key | `DELETE /api/v1/api-keys/{id}` |
| Available permission scopes | `GET /api/v1/api-keys/allowed-permissions` |

When reissuing with `"isGracePeriod": true`, the old key continues to work for a limited time.

---

## Request Signing (HMAC)

**Every request authenticated with an API key** must include all three headers below — including `GET` and `DELETE`.

| Header | Description |
|--------|-------------|
| `x-api-key` | The `key` value returned when the API key was created |
| `x-timestamp` | Unix time in **milliseconds** (`Date.now()`) |
| `x-signature` | Base64-encoded signature (see algorithm below) |

When an `Authorization: Bearer ...` header is present, the API key is ignored. For machine-to-machine integrations, send **only** the HMAC headers without a Bearer token.

### Signing algorithm

1. Build the signing string (UTF-8):

```
{METHOD}\n{PATH}\n{TIMESTAMP}\n{BODY}
```

- `METHOD` — HTTP method in uppercase (`POST`, `GET`, …)
- `PATH` — request path **without** query string (e.g. `/api/v1/merchant/orders/deposit`)
- `TIMESTAMP` — the value of `x-timestamp` (milliseconds as string)
- `BODY` — raw request body for `POST`/`PATCH`; empty string for `GET`/`DELETE`; for `POST` with JSON use the exact bytes you send

2. Hash and sign:

```
payloadHash = SHA256(signingString)
signature   = Base64( HMAC-SHA256(hmacSecret, payloadHash) )
```

### Example (Node.js)

```js
const crypto = require('crypto');

function signPayload(secret, signingString) {
  const payloadHash = crypto.createHash('sha256').update(signingString).digest();
  return crypto.createHmac('sha256', secret).update(payloadHash).digest('base64');
}

function signRequest({ method, path, body, apiKey, hmacSecret }) {
  const timestamp = Date.now().toString();
  const rawBody = body ?? '';

  const signingString = [method.toUpperCase(), path, timestamp, rawBody].join('\n');
  const signature = signPayload(hmacSecret, signingString);

  return {
    'x-api-key': apiKey,
    'x-timestamp': timestamp,
    'x-signature': signature,
    ...(rawBody ? { 'Content-Type': 'application/json' } : {}),
  };
}
```

### Example (cURL)

```bash
TS=$(date +%s000)
BODY='{"amount":"1500.00","currency":"RUB","externalOrderId":"order-42"}'
PATH="/api/v1/merchant/orders/deposit"
SIGNING_STRING="POST\n${PATH}\n${TS}\n${BODY}"
PAYLOAD_HASH=$(printf '%s' "$SIGNING_STRING" | openssl dgst -sha256 -binary)
SIG=$(printf '%s' "$PAYLOAD_HASH" | openssl dgst -sha256 -hmac "$HMAC_SECRET" -binary | base64)

curl -X POST "${BASE_URL}${PATH}" \
  -H "x-api-key: $API_KEY" \
  -H "x-timestamp: $TS" \
  -H "x-signature: $SIG" \
  -H "Content-Type: application/json" \
  -d "$BODY"
```

### Recommendations

- Keep server time synchronized (NTP). A large discrepancy in `x-timestamp` will cause rejection.
- Never place `hmacSecret` in client-side code, mobile apps, or frontend.
- Sign exactly the body that you send — serialization must match byte-for-byte.
- For `GET` requests with query parameters, include query in the URL but **exclude** it from `PATH` in the signing string.
