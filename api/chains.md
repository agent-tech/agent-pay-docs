# ⛓ Supported Chains

AgentTech supports multiple source chains, with all payments ultimately settling on **Base**.

## Supported Chains

| Chain | Identifier | Go Constant | TS Constant |
| :--- | :--- | :--- | :--- |
| Solana | `"solana-mainnet-beta"` | `pay.ChainSolanaMainnet` | `Chain.SolanaMainnet` |
| Base | `"base"` | `pay.ChainBase` | `Chain.Base` |
| Polygon | `"polygon"` | `pay.ChainPolygon` | `Chain.Polygon` |
| Arbitrum | `"arbitrum"` | `pay.ChainArbitrum` | `Chain.Arbitrum` |
| Ethereum | `"ethereum"` | `pay.ChainEthereum` | `Chain.Ethereum` |
| Monad | `"monad"` | `pay.ChainMonad` | `Chain.Monad` |
| HyperEVM | `"hyperevm"` | `pay.ChainHyperEVM` | `Chain.HyperEvm` |

## Settlement Chain

**All payments settle on Base** regardless of the source chain. The `payerChain` field in `CreateIntentRequest` specifies where the payer initiates the payment, but the final USDC transfer always occurs on Base.

## Chain Identifiers

Use the chain constants from each SDK instead of hardcoded strings.

| Chain | Identifier |
| :--- | :--- |
| Solana | `"solana-mainnet-beta"` |
| Base | `"base"` |
| Polygon | `"polygon"` |
| Arbitrum | `"arbitrum"` |
| Ethereum | `"ethereum"` |
| Monad | `"monad"` |
| HyperEVM | `"hyperevm"` |

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
