---
order: 3
---

# Intents

Intents represent payment requests in Cross402. This section covers all intent-related API methods.

## CreateIntent

Creates a new payment intent. The payer chain and target chain can differ; if the target chain is omitted it defaults to `base`.

### Request Parameters

| Field | JSON | Required | Description |
| --- | --- | --- | --- |
| Email | `email` | One of Email/Recipient | Recipient email address |
| Recipient | `recipient` | One of Email/Recipient | Recipient wallet address, validated against `target_chain` (EVM address for EVM targets, Solana address for `solana`) |
| Amount | `amount` | One of Amount/ToAmount | **ExactOut** target: the merchant receives exactly this dollar amount. String (e.g. `"100.50"`). Denomination is whatever asset the route uses — defaults to USDC; pass `payer_asset` / `target_asset` to opt into USDT0/USDT. |
| ToAmount | `to_amount` | One of Amount/ToAmount | **ExactIn** amount: the payer sends exactly this dollar amount and the merchant receives the remainder after fees. Mutually exclusive with `amount` — set exactly one. |
| PayerChain | `payer_chain` | Yes | Source chain identifier. See [Supported Chains](../../../introduction/supported-networks/). |
| TargetChain | `target_chain` | No | Settlement chain identifier. Defaults to `"base"`. Must be a chain listed by `GET /api/chains`. |
| PayerAsset | `payer_asset` | No | Token the payer signs against on `payer_chain`. One of `"usdc"`, `"usdt"`, `"usdt0"`. Defaults to `"usdc"`. The (chain, asset) pair must be supported — see [Token × Chain matrix](../../../introduction/supported-networks/#token--chain-matrix). |
| TargetAsset | `target_asset` | No | Token the merchant receives on `target_chain`. Same allowed values and default as `payer_asset`. The proxy wallet must hold this asset on `target_chain` for the intent to settle. |
| PayerAddress | `payer_address` | No | Payer wallet address, screened advisorily against sanctions lists at create time. Leave empty to skip; the authoritative payer screen still runs in the async settlement path. |

> Backward compatibility: clients that omit `payer_asset` / `target_asset` continue to behave exactly as before — both fields default to `"usdc"` and the response shape is unchanged. The fields only need to be set when opting into non-USDC routes.

### Amount modes: ExactOut vs ExactIn

An intent carries the amount on exactly one of two fields — set one, never both:

* **`amount` (ExactOut)** — the merchant receives exactly this amount; the payer covers it plus fees. Use this for invoices and checkout where the recipient must land a precise figure.
* **`to_amount` (ExactIn)** — the payer spends exactly this amount; the merchant receives whatever is left after fees. Use this when the spend is fixed (e.g. "send the $5 in this wallet").

The settlement quote (`sending_amount`, `receiving_amount`, `estimated_fee`) on the response shows how the chosen mode resolved.

### Amount Rules

* **Minimum**: 0.02 (denominated in the selected asset)
* **Maximum**: 1,000,000 (denominated in the selected asset)
* **Precision**: Up to 6 decimal places (e.g. `"0.000001"`, `"123.45"`). Chain-native base units (BSC Binance-Peg's 18-dec USDC, MegaETH USDm's 18-dec native) are derived from this dollar string using the asset's deployed `decimals()`; never hardcode `6`.

### Example — base payer, Ethereum settlement

```
const intent = await client.createIntent({
  email: "merchant@example.com",
  amount: "100.50",
  payerChain: "base",
  targetChain: "ethereum",
});
```

```
resp, err := client.CreateIntent(ctx, &pay.CreateIntentRequest{
    Email:       "merchant@example.com",
    Amount:      "100.50",
    PayerChain:  "base",
    TargetChain: "ethereum",
})
```

### Example — ExactIn (fixed spend)

Set `to_amount` instead of `amount` to spend a fixed amount; the merchant receives the remainder after fees.

```typescript
const intent = await client.createIntent({
  email: "merchant@example.com",
  toAmount: "5.00",        // payer sends exactly $5.00
  payerChain: "base",
  targetChain: "ethereum",
});
```

```go
resp, err := client.CreateIntent(ctx, &pay.CreateIntentRequest{
    Email:       "merchant@example.com",
    ToAmount:    "5.00", // payer sends exactly $5.00; omit Amount
    PayerChain:  "base",
    TargetChain: "ethereum",
})
```

### Example — Arbitrum payer paying in USDT0

```typescript
const intent = await client.createIntent({
  email: "merchant@example.com",
  amount: "100.50",
  payerChain: "arbitrum",
  payerAsset: "usdt0",
  targetChain: "base",  // merchant still receives USDC on Base by default
});
```

```go
resp, err := client.CreateIntent(ctx, &pay.CreateIntentRequest{
    Email:       "merchant@example.com",
    Amount:      "100.50",
    PayerChain:  "arbitrum",
    PayerAsset:  "usdt0",
    TargetChain: "base",
})
```

The response's `payment_requirements.extra.name` / `.version` carry the asset's EIP-712 domain (e.g. `"USD₮0"` / `"1"` for Arbitrum USDT0); use these — never hardcode — to construct the EIP-3009 signature. For Polygon USDT0/USDT, `extra.domainType == "salted"` also appears; the payer SDK uses it to switch from the standard chainId-in-domain layout to the salted Polygon-PoS variant. See [Supported Chains](../../../introduction/supported-networks/#polygon-salted-eip-712-domain) for the mechanics.

> When `recipient` is a wallet address, its format is validated against `target_chain`. Passing a Solana address with `target_chain: "ethereum"` is rejected as `invalid recipient`.

## ExecuteIntent

Executes an intent using the Agent wallet. The backend signs and transfers the stablecoin on the agent's source-chain wallet, then transfers to the merchant on the target chain.

**Requires authentication** (Bearer token).

`POST /v2/intents/{intent_id}/execute` enforces the **same ownership policy as `GET /v2/intents`**: looking up an intent owned by another agent — or one created via the unauthenticated `/api` flow — returns `404 payment intent not found`, not 403. This is intentional: collapsing both rejection paths to the same response prevents authenticated callers from probing for valid intent IDs across other agents by observing the 403/404 split. `403` is reserved for future use; today it never appears for ownership rejections.

### Example

```
const result = await client.executeIntent(intentId);
```

```
exec, err := client.ExecuteIntent(ctx, resp.IntentID)
// exec.Status is typically "TARGET_SETTLED"
```

## SubmitProof

Submits a settlement proof after the payer completes X402 payment on the source chain.

**Public endpoint only** — `POST /api/intents/{intent_id}`. There is no `/v2` equivalent. The Go SDK rejects `SubmitProof` with `ErrSubmitProofNotAllowed` when the client is configured with `WithBearerAuth`; the JS `PayClient` simply does not expose the method (use `PublicPayClient` instead).

### Example

```
// PublicPayClient (no auth)
const proof = await client.submitProof(intentId, settleProof);
```

```
// pay.NewClient(baseURL) — no auth option
proof, err := client.SubmitProof(ctx, intentID, settleProof)
```

## GetIntent

Queries the current status of an intent. Use this to poll for status updates.

`GET /v2/intents` enforces ownership: it returns `404 payment intent not found` when the intent's owning `agent_id` does not match the API key's agent (and likewise for intents that were created via the unauthenticated `/api` flow and have no owner). The 404 — instead of 403 — is intentional, so callers cannot use the endpoint to check whether an intent ID exists under another agent. The public `GET /api/intents` flow is unchanged: anyone with the intent ID can read it.

### Example Response

```
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

`agent_id` is populated only on `/v2` responses (intents owned by an authenticated agent); the `/api` flow leaves it unset. The settlement quote fields (`sending_amount`, `receiving_amount`, `estimated_fee`, `fee_breakdown`) are populated by the backend once the intent leaves its initial state — treat them as optional on `GetIntent` responses. The `target_payment` field is populated only once status reaches `TARGET_SETTLED`. For failures, see `error_message` and the [Statuses reference](../../concepts/statuses/).

### Example — polling

```
const intent = await client.getIntent(intentId);
switch (intent.status) {
  case "TARGET_SETTLED":
    // Payment complete — use intent.targetPayment for receipt
    break;
  case "EXPIRED":
  case "VERIFICATION_FAILED":
  case "BLOCKED":
  case "PARTIAL_SETTLEMENT":
    // Terminal failure
    break;
  default:
    // Still processing — poll again
}
```

```
intent, err := client.GetIntent(ctx, intentID)
switch intent.Status {
case pay.StatusTargetSettled:
    // use intent.TargetPayment for receipt
case pay.StatusExpired, pay.StatusVerificationFailed, pay.StatusBlocked, pay.StatusPartialSettlement:
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
| --- | --- | --- |
| `page` | No | 1-indexed page number. Defaults to `1`. Server caps at `1,000,000`. Out-of-range or non-numeric values return `400`. |
| `page_size` | No | Items per page. Defaults to `20`. Values in `[1, 100]` are honored; values above `100` are clamped to `100` (so a client asking for "the largest page" gets the maximum, not the default). Negative or non-numeric values return `400`. |

> The server returns `400` for malformed `page` / `page_size` rather than silently falling back to defaults. SDK callers that previously relied on the silent fallback should pass valid integers (or omit the parameter to use the default).

### Example Response

```
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

```
const page = await client.listIntents({ page: 1, pageSize: 20 });
for (const it of page.intents) {
  console.log(`${it.intentId} ${it.status} ${it.payerChain}→${it.targetChain}`);
}
```

```
list, err := client.ListIntents(ctx, 1, 20) // page=1, page_size=20
for _, it := range list.Intents {
    log.Printf("%s %s %s→%s", it.IntentID, it.Status, it.PayerChain, it.TargetChain)
}
```

## GetMe

Returns the identity of the agent owning the API key in use. Useful for verifying credentials and discovering the agent's funded wallet addresses without a database lookup.

**Endpoint**: `GET /v2/me` (auth required, v2-only). The handler reads from middleware context and does not hit the database.

### Example Response

```
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

```
const me = await client.getMe();
console.log(`agent ${me.agentId} (${me.name}) wallet=${me.walletAddress}`);
```

```
me, err := client.GetMe(ctx)
log.Printf("agent %s (%s) wallet=%s", me.AgentID, me.Name, me.WalletAddress)
```

## Intent Expiration

Intents expire **10 minutes** after creation. If not executed within this window, the intent status becomes `EXPIRED` (terminal state).
