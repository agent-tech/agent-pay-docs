# ⛓ Supported Chains

AgentTech supports multiple source chains, with all payments ultimately settling on **Base**.

## Supported Chains

| Chain | Identifier | Go Constant | TS Constant | USDC Decimals | Status |
| :--- | :--- | :--- | :--- | :--- | :--- |
| Solana | `"solana-mainnet-beta"` | `pay.ChainSolanaMainnet` | `Chain.SolanaMainnet` | 6 | Available |
| Base | `"base"` | `pay.ChainBase` | `Chain.Base` | 6 | Available |
| BSC | `"bsc"` | `pay.ChainBSC` | `Chain.BSC` | **18** | Available |
| Polygon | `"polygon"` | `pay.ChainPolygon` | `Chain.Polygon` | 6 | Available |
| Arbitrum | `"arbitrum"` | `pay.ChainArbitrum` | `Chain.Arbitrum` | 6 | Available |
| Ethereum | `"ethereum"` | `pay.ChainEthereum` | `Chain.Ethereum` | 6 | Available |
| Monad | `"monad"` | `pay.ChainMonad` | `Chain.Monad` | 6 | Available |
| HyperEVM | `"hyperevm"` | `pay.ChainHyperEVM` | `Chain.HyperEvm` | 6 | Available |

## Settlement Chain

**All payments settle on Base** regardless of the source chain. The `payerChain` field in `CreateIntentRequest` specifies where the payer initiates the payment, but the final USDC transfer always occurs on Base.

## Chain Identifiers

Use the chain constants from each SDK instead of hardcoded strings.

| Chain | Identifier | USDC Decimals | Status |
| :--- | :--- | :--- | :--- |
| Solana | `"solana-mainnet-beta"` | 6 | Available |
| Base | `"base"` | 6 | Available |
| BSC | `"bsc"` | **18** | Available |
| Polygon | `"polygon"` | 6 | Available |
| Arbitrum | `"arbitrum"` | 6 | Available |
| Ethereum | `"ethereum"` | 6 | Available |
| Monad | `"monad"` | 6 | Available |
| HyperEVM | `"hyperevm"` | 6 | Available |

## Chain-Specific Notes

### BSC (BNB Smart Chain)

> **Important**: USDC on BSC uses **18 decimals**, unlike the standard 6 decimals used on most other chains.

Two things to be aware of when paying from BSC:

1. **Amount scaling**: Do not hardcode 6 decimals for USDC amounts. Always use `extra.decimals` from the backend response to determine the correct decimal precision for the source chain.
2. **Permit2 pre-approval**: BSC requires a one-time Permit2 contract approval before the first payment. This is specific to BSC — other EVM chains do not require this step.

```go
// Always use extra.decimals from the intent response, not hardcoded values
decimals := resp.Extra.Decimals // e.g. 18 for BSC, 6 for Base/Polygon
amount := new(big.Int).Mul(
    amountFloat,
    new(big.Int).Exp(big.NewInt(10), big.NewInt(int64(decimals)), nil),
)
```

## Usage

When creating an intent, specify the source chain using SDK constants:

```typescript
import { Chain } from '@cross402/usdc';

const intent = await client.createIntent({
  email: "merchant@example.com",
  amount: "100.50",
  payerChain: Chain.Base,
});
```

```go
resp, err := client.CreateIntent(ctx, &pay.CreateIntentRequest{
    Email:      "merchant@example.com",
    Amount:     "100.50",
    PayerChain: pay.ChainBase,
})
```
