# 📝 Intents

Intents represent payment requests in AgentPay. This section covers all intent-related API methods.

## CreateIntent

Creates a new payment intent. Returns an intent ID that you'll use for subsequent operations.

### Request Parameters

| Field | JSON | Required | Description |
| :--- | :--- | :--- | :--- |
| Email | `email` | One of Email/Recipient | Recipient email address |
| Recipient | `recipient` | One of Email/Recipient | Recipient wallet address |
| Amount | `amount` | Yes | USDC amount as string (e.g. "100.50") |
| PayerChain | `payer_chain` | Yes | Source chain: `"base"` or `"solana"` |

### Amount Rules

* **Minimum**: 0.01 USDC
* **Maximum**: 1,000,000 USDC
* **Precision**: Up to 6 decimal places (e.g. `"0.000001"`, `"123.45"`)

### Example

```typescript
const intent = await client.createIntent({
  email: "merchant@example.com",
  amount: "100.50",
  payerChain: "base"
});
```

```go
resp, err := client.CreateIntent(ctx, &pay.CreateIntentRequest{
    Email:      "merchant@example.com",
    Amount:     "100.50",
    PayerChain: "base",
})
```

## ExecuteIntent

Executes an intent using the Agent wallet. The backend signs and transfers USDC on Base automatically.

**Requires authentication** (Bearer token).

### Example

```typescript
const result = await client.executeIntent(intentId);
```

```go
exec, err := client.ExecuteIntent(ctx, resp.IntentID)
// exec.Status is typically "BASE_SETTLED"
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

### Example

```typescript
const intent = await client.getIntent(intentId);
switch (intent.status) {
  case "BASE_SETTLED":
    // Payment complete
    break;
  case "EXPIRED":
  case "VERIFICATION_FAILED":
    // Terminal failure
    break;
  default:
    // Still processing - poll again
}
```

```go
intent, err := client.GetIntent(ctx, intentID)
switch intent.Status {
case pay.StatusBaseSettled:
    // use intent.BasePayment for receipt
case pay.StatusExpired, pay.StatusVerificationFailed:
    // terminal failure
default:
    // still processing — poll again
}
```

## Intent Expiration

Intents expire **10 minutes** after creation. If not executed within this window, the intent status becomes `EXPIRED` (terminal state).
