# ⛓ Supported Chains

AgentPay supports multiple source chains, with all payments ultimately settling on **Base**.

## Chain Roles

| Chain | Role | Notes |
| :--- | :--- | :--- |
| Base | Payer chain (source) | Supported in authenticated and public modes |
| Solana | Payer chain (source) | Supported as source chain |
| BSC | Payer chain (source) | Supported as source chain |
| Polygon | Payer chain (source) | Supported as source chain |
| Arbitrum | Payer chain (source) | Supported as source chain |
| Ethereum | Payer chain (source) | Supported as source chain |
| Monad | Payer chain (source) | Supported as source chain |

## Settlement Chain

**All payments settle on Base** regardless of the source chain. The `payer_chain` field in `CreateIntentRequest` specifies where the payer initiates the payment, but the final USDC transfer always occurs on Base.

## Chain Identifiers

Use the chain constants from each SDK instead of hardcoded strings.

| Chain | Testnet Identifier | Mainnet Identifier |
| :--- | :--- | :--- |
| Solana | `"solana-devnet"` | `"solana-mainnet-beta"` |
| Base | `"base-sepolia"` | `"base"` |
| BSC | `"bsc-testnet"` | `"bsc"` |
| Polygon | `"polygon-amoy"` | `"polygon"` |
| Arbitrum | `"arbitrum-sepolia"` | `"arbitrum"` |
| Ethereum | `"ethereum-sepolia"` | `"ethereum"` |
| Monad | `"monad-testnet"` | `"monad"` |

## Usage

When creating an intent, specify the source chain:

```typescript
const intent = await client.createIntent({
  email: "merchant@example.com",
  amount: "100.50",
  payerChain: "base"  // e.g. "base", "polygon", "solana-mainnet-beta"
});
```

```go
resp, err := client.CreateIntent(ctx, &pay.CreateIntentRequest{
    Email:      "merchant@example.com",
    Amount:     "100.50",
    PayerChain: pay.ChainBase, // or pay.ChainPolygon / pay.ChainSolanaMainnet
})
```
