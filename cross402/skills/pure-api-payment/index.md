---
order: 8
---

# Pure API Payment — Agent-Ready HTTP Recipes

Pattern library for AI agents that need to call AgentTech directly over HTTP — no SDK, no language runtime assumptions. Every recipe below is a complete request the agent can reproduce by string-substituting the placeholders.

If you have access to the [JS/TS SDK](../../sdks/js-ts/) or [Go SDK](../../sdks/go/), use those instead — they bundle validation, typed errors, and Permit2 signing. Use this skill only when the SDK is not available (different language, shell-only runtime, sandboxed env without npm/go).

Full prose tutorial: [Pure API (no SDK)](../../sdks/pure-api/).

---

## JSON Schema Definition

```
{
  "name": "pure_api_payment",
  "description": "Execute an AgentTech payment via direct HTTP calls (no SDK). Supports the API-key flow (backend signs) and the payer-signs flow.",
  "input_schema": {
    "type": "object",
    "properties": {
      "auth_mode": {
        "type": "string",
        "enum": ["api_key", "payer_signs"],
        "description": "api_key = use /v2 with Authorization Bearer header; payer_signs = use /api with no auth (payer signs off-chain)"
      },
      "client_id":     { "type": "string", "description": "Required when auth_mode=api_key" },
      "client_secret": { "type": "string", "description": "Required when auth_mode=api_key" },
      "recipient":     { "type": "string", "description": "Recipient wallet address; mutually exclusive with email" },
      "email":         { "type": "string", "description": "Recipient email; mutually exclusive with recipient" },
      "amount":        { "type": "string", "description": "USD amount as string, [0.02, 1000000], up to 6 decimals" },
      "payer_chain":   { "type": "string" },
      "target_chain":  { "type": "string", "description": "Optional; defaults to 'base'" }
    },
    "required": ["auth_mode", "amount", "payer_chain"]
  },
  "output_schema": {
    "type": "object",
    "properties": {
      "intent_id":     { "type": "string" },
      "status":        { "type": "string", "enum": ["TARGET_SETTLED", "EXPIRED", "VERIFICATION_FAILED", "PARTIAL_SETTLEMENT"] },
      "tx_hash":       { "type": "string" },
      "explorer_url":  { "type": "string" }
    },
    "required": ["intent_id", "status"]
  }
}
```

---

## One-liner — create + execute (api\_key mode)

The most common task: pay a merchant with the agent's own wallet. Substitute `<client-id>`, `<client-secret>`, `<merchant-wallet>`, `<amount>`.

```
TOKEN=$(printf '%s:%s' "<client-id>" "<client-secret>" | base64) && \
INTENT_ID=$(curl -sS -X POST https://api-pay.agent.tech/v2/intents \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"recipient":"<merchant-wallet>","amount":"<amount>","payer_chain":"base","target_chain":"ethereum"}' \
  | jq -r '.intent_id') && \
curl -sS -X POST "https://api-pay.agent.tech/v2/intents/$INTENT_ID/execute" \
  -H "Authorization: Bearer $TOKEN"
```

The second call returns the final `status`. If it is not `TARGET_SETTLED`, fall through to polling (next recipe).

---

## Sample request / response — tagged for pattern matching

### Verify key (cheap, do this on cold start)

> Request

```http
GET /v2/me HTTP/1.1
Host: api-pay.agent.tech
Authorization: Bearer <base64(client_id:client_secret)>
```

> 200 OK

```json
{
  "agent_id": "8b2e9c4a-3f7a-4d1b-9e2c-5a6b7c8d9e0f",
  "agent_number": "A-000123",
  "name": "checkout-agent",
  "status": "active",
  "wallet_address": "0x742d35Cc...",
  "solana_wallet_address": "Es9vMFrz..."
}
```

> 401 Unauthorized — hard config error, do not retry

```json
{ "error": "Unauthorized", "message": "invalid api key", "status_code": 401 }
```

### Create intent (api\_key mode)

> Request

```http
POST /v2/intents HTTP/1.1
Host: api-pay.agent.tech
Authorization: Bearer <base64(client_id:client_secret)>
Content-Type: application/json

{
  "email": "merchant@example.com",
  "amount": "100.50",
  "payer_chain": "base",
  "target_chain": "ethereum"
}
```

> 201 Created

```json
{
  "intent_id": "int_abc123xyz",
  "agent_id": "8b2e9c4a-3f7a-4d1b-9e2c-5a6b7c8d9e0f",
  "sending_amount": "100.50",
  "receiving_amount": "98.65",
  "estimated_fee": "1.85",
  "payer_chain": "base",
  "target_chain": "ethereum",
  "status": "AWAITING_PAYMENT",
  "created_at": "2024-01-01T12:00:00Z",
  "expires_at": "2024-01-01T12:10:00Z",
  "payment_requirements": { "...": "x402 typed-data spec" }
}
```

### Execute (settle)

> Request

```http
POST /v2/intents/int_abc123xyz/execute HTTP/1.1
Host: api-pay.agent.tech
Authorization: Bearer <base64(client_id:client_secret)>
```

> 200 OK

```json
{
  "intent_id": "int_abc123xyz",
  "agent_id": "8b2e9c4a-3f7a-4d1b-9e2c-5a6b7c8d9e0f",
  "sending_amount": "100.50",
  "receiving_amount": "98.65",
  "status": "TARGET_SETTLED",
  "created_at": "2024-01-01T12:00:00Z",
  "expires_at": "2024-01-01T12:10:00Z"
}
```

### Poll until terminal

```
while true; do
  STATUS=$(curl -sS "https://api-pay.agent.tech/v2/intents?intent_id=$INTENT_ID" \
    -H "Authorization: Bearer $TOKEN" | jq -r '.status')
  case "$STATUS" in
    TARGET_SETTLED|EXPIRED|VERIFICATION_FAILED|PARTIAL_SETTLEMENT) break ;;
  esac
  sleep 3
done
echo "final status: $STATUS"
```

### Submit proof (payer\_signs mode)

> Request

```http
POST /api/intents/int_pub789 HTTP/1.1
Host: api-pay.agent.tech
Content-Type: application/json

{
  "settle_proof": "<base64-encoded-payment-payload>"
}
```

The `settle_proof` is `base64(JSON(PaymentPayload))` where `PaymentPayload` is the X402 payload built by signing the typed data returned in `payment_requirements.extra` with the payer's private key. See [BSC Signing](../../api/bsc-signing/) for a worked Permit2 example with `viem`.

> 200 OK

```json
{
  "intent_id": "int_pub789",
  "status": "SOURCE_SETTLED",
  "sending_amount": "10.00",
  "receiving_amount": "9.45"
}
```

---

## Failure modes — what to retry, what to surface

The agent runtime should decide based on the HTTP status code and the stable `message` string. The table below covers every state defined by the backend.

| Status | Stable `message` | Retry? | Agent action |
| --- | --- | --- | --- |
| `400` | `invalid payer_chain: ...` / `invalid target_chain: ...` / `invalid amount: ...` / `invalid email format` / `invalid recipient address format` / `either email or recipient is required` / `cannot provide both email and recipient` | **No** | Surface to user — input is wrong. Do not auto-retry, the next attempt will fail identically. |
| `400` | `payment intent has expired` | **No** | Tell user "this took too long, create a new payment" and re-run from `CreateIntent`. |
| `400` | `invalid intent status for this operation` | **No** | Re-read status with `GET /v2/intents?intent_id=...` before deciding next step (often the intent is already settled). |
| `400` | `proof validation failed` | **No** | Re-sign the typed data from the latest `payment_requirements`. If it still fails, the payer wallet did not produce a valid signature — surface to user. |
| `400` | `concurrent update detected: intent status has changed` | **No** | Re-read status and pick a path based on the new value. Do not blindly resubmit. |
| `401` | `missing or invalid api key` / `invalid api key` / `api key revoked` / `agent not active` / `agent not found` | **No** | Hard config error. Surface to user — they must fix or rotate the key. |
| `402` | `agent wallet has insufficient balance on payer chain` | **No** | Surface to user: "your agent wallet needs more USDC on `<payer_chain>`". Include the wallet address from `/v2/me`. |
| `403` | `intent does not belong to this agent` | **No** | Wrong key for this intent. Surface to user. |
| `404` | `payment intent not found` | **No** | The intent doesn't exist **or** is owned by another agent. Verify the intent ID; do not retry. |
| `429` | (any) | **Yes** | Exponential backoff: `sleep 1, 2, 4, 8, 16` seconds. Cap at 5 attempts. |
| `503` | `service temporarily unavailable: payment wallet has insufficient balance` | **Yes** | Backend issue — back off and retry. Cap at 10 minutes total (intents expire then anyway). |
| `5xx` other | sanitized: `an internal error occurred, please try again later` | **Yes** | Treat as transient. Same backoff as `503`. |

### Terminal states for the polling loop

These come back as `status` on `GetIntent` (200 OK). Treat them all as exit conditions — there is no further transition.

| Status | Outcome | Agent action |
| --- | --- | --- |
| `TARGET_SETTLED` | Success | Surface `target_payment.tx_hash` and `target_payment.explorer_url` to user. |
| `VERIFICATION_FAILED` | Permanent failure — signature did not verify | Surface to user; do not retry. |
| `PARTIAL_SETTLEMENT` | Source landed, target failed | Surface to user with "contact support for reconciliation". `error_message` is the generic mapped string; the real reason is on the backend. |
| `EXPIRED` | Not settled inside the 10-minute window | Surface to user; create a new intent if they still want to pay. |

### Soft signals during polling

These are non-terminal. Keep polling.

| Status | Meaning |
| --- | --- |
| `AWAITING_PAYMENT` | Created; execute / submit-proof not yet called |
| `PENDING` | Backend is processing |
| `SOURCE_SETTLED` | Payer side landed; backend driving target side |
| `TARGET_SETTLING` | Target-chain transfer in flight |

---

## Cross-references

* Full prose: [Pure API (no SDK)](../../sdks/pure-api/)
* Endpoint-level schemas: [API Reference / Intents](../../api/intents/)
* Auth header detail: [API Reference / Authentication](../../api/auth/)
* Permit2 signing for BSC / MegaETH: [BSC Signing](../../api/bsc-signing/)
* Error code reference: [Error Handling](../../error-codes/)
