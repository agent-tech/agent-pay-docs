# 📝 Intents

Intents represent payment requests in cross402. This section covers all intent-related API methods.

## CreateIntent

Creates a new payment intent. The payer chain and target chain can differ; if the target chain is omitted it defaults to `base`.

### Request Parameters

| Field | JSON | Required | Description |
| :--- | :--- | :--- | :--- |
| Email | `email` | One of Email/Recipient | Recipient email address |
| Recipient | `recipient` | One of Email/Recipient | Recipient wallet address, validated against `target_chain` (EVM address for EVM targets, Solana address for `solana`) |
| Amount | `amount` | Yes | USDC amount as string (e.g. "100.50") |
| PayerChain | `payer_chain` | Yes | Source chain identifier. See [Supported Chains](chains.md). |
| TargetChain | `target_chain` | No | Settlement chain identifier. Defaults to `"base"`. Must be a chain listed by `GET /api/chains`. |

### Amount Rules

* **Minimum**: 0.02 USDC
* **Maximum**: 1,000,000 USDC
* **Precision**: Up to 6 decimal places (e.g. `"0.000001"`, `"123.45"`)

### Example — base payer, Ethereum settlement

```typescript
const intent = await client.createIntent({
  email: "merchant@example.com",
  amount: "100.50",
  payerChain: "base",
  targetChain: "ethereum",
});
```

```go
resp, err := client.CreateIntent(ctx, &pay.CreateIntentRequest{
    Email:       "merchant@example.com",
    Amount:      "100.50",
    PayerChain:  "base",
    TargetChain: "ethereum",
})
```

> When `recipient` is a wallet address, its format is validated against `target_chain`. Passing a Solana address with `target_chain: "ethereum"` is rejected as `invalid recipient`.

## ExecuteIntent

Executes an intent using the Agent wallet. The backend signs and transfers USDC on the agent's source-chain wallet, then transfers to the merchant on the target chain.

**Requires authentication** (Bearer token).

### Example

```typescript
const result = await client.executeIntent(intentId);
```

```go
exec, err := client.ExecuteIntent(ctx, resp.IntentID)
// exec.Status is typically "TARGET_SETTLED"
```

## SubmitProof

Submits a settlement proof after the payer completes X402 payment on the source chain.

**Public endpoint only** — `POST /api/intents/{intent_id}`. There is no `/v2` equivalent. The Go SDK rejects `SubmitProof` with `ErrSubmitProofNotAllowed` when the client is configured with `WithBearerAuth`; the JS `PayClient` simply does not expose the method (use `PublicPayClient` instead).

### Example

```typescript
// PublicPayClient (no auth)
const proof = await client.submitProof(intentId, settleProof);
```

```go
// pay.NewClient(baseURL) — no auth option
proof, err := client.SubmitProof(ctx, intentID, settleProof)
```

## GetIntent

Queries the current status of an intent. Use this to poll for status updates.

`GET /v2/intents` enforces ownership: it returns `404 payment intent not found` when the intent's owning `agent_id` does not match the API key's agent (and likewise for intents that were created via the unauthenticated `/api` flow and have no owner). The 404 — instead of 403 — is intentional, so callers cannot use the endpoint to check whether an intent ID exists under another agent. The public `GET /api/intents` flow is unchanged: anyone with the intent ID can read it.

### Example Response

```json
{
  "intent_id": "int_abc123xyz",
  "agent_id": "8b2e9c4a-3f7a-4d1b-9e2c-5a6b7c8d9e0f",
  "status": "TARGET_SETTLED",
  "payer_chain": "base",
  "target_chain": "ethereum",
  "merchant_recipient": "0x742d35Cc...",
  "target_payment": {
    "tx_hash": "0x1234...abcd",
    "settle_proof": "...",
    "settled_at": "2024-01-01T12:05:00Z",
    "explorer_url": "https://etherscan.io/tx/0x1234...abcd"
  },
  "source_payment": {
    "chain": "base",
    "tx_hash": "0x9876...fedc",
    "settle_proof": "...",
    "settled_at": "2024-01-01T12:04:30Z",
    "explorer_url": "https://basescan.org/tx/0x9876...fedc"
  }
}
```

`agent_id` is populated only on `/v2` responses (intents owned by an authenticated agent); the `/api` flow leaves it unset. The settlement quote fields (`sending_amount`, `receiving_amount`, `estimated_fee`, `fee_breakdown`) are populated by the backend once the intent leaves its initial state — treat them as optional on `GetIntent` responses. The `target_payment` field is populated only once status reaches `TARGET_SETTLED`. For failures, see `error_message` and the [Statuses reference](statuses.md).

### Example — polling

```typescript
const intent = await client.getIntent(intentId);
switch (intent.status) {
  case "TARGET_SETTLED":
    // Payment complete — use intent.targetPayment for receipt
    break;
  case "EXPIRED":
  case "VERIFICATION_FAILED":
  case "PARTIAL_SETTLEMENT":
    // Terminal failure
    break;
  default:
    // Still processing — poll again
}
```

```go
intent, err := client.GetIntent(ctx, intentID)
switch intent.Status {
case pay.StatusTargetSettled:
    // use intent.TargetPayment for receipt
case pay.StatusExpired, pay.StatusVerificationFailed, pay.StatusPartialSettlement:
    // terminal failure
default:
    // still processing — poll again
}
```

## ListIntents

Lists intents owned by the calling agent, most recent first.

**Endpoint**: `GET /v2/intents/list?page=&page_size=` (auth required, v2-only).

### Query Parameters

| Parameter | Required | Description |
| :--- | :--- | :--- |
| `page` | No | 1-indexed page number. Defaults to `1`. |
| `page_size` | No | Items per page. Defaults to `20`; clamped to `[1, 100]`. |

### Example Response

```json
{
  "intents": [
    {
      "intent_id": "int_abc123xyz",
      "agent_id": "8b2e9c4a-3f7a-4d1b-9e2c-5a6b7c8d9e0f",
      "merchant_recipient": "0x742d35Cc...",
      "sending_amount": "100.50",
      "receiving_amount": "98.65",
      "estimated_fee": "1.85",
      "payer_chain": "base",
      "target_chain": "ethereum",
      "status": "TARGET_SETTLED",
      "created_at": "2024-01-01T12:00:00Z",
      "expires_at": "2024-01-01T12:10:00Z"
    }
  ],
  "total": 1,
  "page": 1,
  "page_size": 20
}
```

```typescript
const page = await client.listIntents({ page: 1, pageSize: 20 });
for (const it of page.intents) {
  console.log(`${it.intentId} ${it.status} ${it.payerChain}→${it.targetChain}`);
}
```

```go
list, err := client.ListIntents(ctx, 1, 20) // page=1, page_size=20
for _, it := range list.Intents {
    log.Printf("%s %s %s→%s", it.IntentID, it.Status, it.PayerChain, it.TargetChain)
}
```

## GetMe

Returns the identity of the agent owning the API key in use. Useful for verifying credentials and discovering the agent's funded wallet addresses without a database lookup.

**Endpoint**: `GET /v2/me` (auth required, v2-only). The handler reads from middleware context and does not hit the database.

### Example Response

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

`wallet_address` and `solana_wallet_address` are omitted when the agent has no wallet for that chain family.

```typescript
const me = await client.getMe();
console.log(`agent ${me.agentId} (${me.name}) wallet=${me.walletAddress}`);
```

```go
me, err := client.GetMe(ctx)
log.Printf("agent %s (%s) wallet=%s", me.AgentID, me.Name, me.WalletAddress)
```

## Intent Expiration

Intents expire **10 minutes** after creation. If not executed within this window, the intent status becomes `EXPIRED` (terminal state).
