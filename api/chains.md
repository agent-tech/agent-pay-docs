# ⛓ Supported Chains

AgentPay supports multiple source chains, with all payments ultimately settling on **Base**.

## Chain Roles

| Chain | Identifier | Role | Notes |
| :--- | :--- | :--- | :--- |
| **base** | `"base"` | Payer chain (source) | Supported in both authenticated and public modes |
| **solana** | `"solana"` | Payer chain (source) | Public mode only |

## Settlement Chain

**All payments settle on Base** regardless of the source chain. The `payer_chain` field in `CreateIntentRequest` specifies where the payer initiates the payment, but the final USDC transfer always occurs on Base.

## Usage

When creating an intent, specify the source chain:

```typescript
const intent = await client.createIntent({
  email: "merchant@example.com",
  amount: "100.50",
  payerChain: "base"  // or "solana" for public mode
});
```

```go
resp, err := client.CreateIntent(ctx, &pay.CreateIntentRequest{
    Email:      "merchant@example.com",
    Amount:     "100.50",
    PayerChain: "base",  // or "solana" for public mode
})
```
