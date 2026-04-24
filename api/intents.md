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

**No authentication required** (public endpoint).

### Example

```typescript
const proof = await client.submitProof(intentId, settleProof);
```

```go
proof, err := client.SubmitProof(ctx, intentID, settleProof)
```

## GetIntent

Queries the current status of an intent. Use this to poll for status updates.

### Example Response

```json
{
  "intent_id": "int_abc123xyz",
  "status": "TARGET_SETTLED",
  "payer_chain": "base",
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

The `target_payment` field is populated only once status reaches `TARGET_SETTLED`. For failures, see `error_message` and the [Statuses reference](statuses.md).

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

## Intent Expiration

Intents expire **10 minutes** after creation. If not executed within this window, the intent status becomes `EXPIRED` (terminal state).
