# ⛓ Supported Chains

AgentPay supports multiple source chains, with all payments ultimately settling on **Base**.

## Supported Chains

### Testnet

| Chain | Identifier | Go Constant | TS Constant |
| :--- | :--- | :--- | :--- |
| Solana Devnet | `"solana-devnet"` | `pay.ChainSolanaDevnet` | `Chain.SolanaDevnet` |
| Base Sepolia | `"base-sepolia"` | `pay.ChainBaseSepolia` | `Chain.BaseSepolia` |
| BSC Testnet | `"bsc-testnet"` | `pay.ChainBSCTestnet` | `Chain.BscTestnet` |
| Polygon Amoy | `"polygon-amoy"` | `pay.ChainPolygonAmoy` | `Chain.PolygonAmoy` |
| Arbitrum Sepolia | `"arbitrum-sepolia"` | `pay.ChainArbitrumSepolia` | `Chain.ArbitrumSepolia` |
| Ethereum Sepolia | `"ethereum-sepolia"` | `pay.ChainEthereumSepolia` | `Chain.EthereumSepolia` |
| Monad Testnet | `"monad-testnet"` | `pay.ChainMonadTestnet` | `Chain.MonadTestnet` |
| HyperEVM Testnet | `"hyperevm-testnet"` | `pay.ChainHyperEVMTestnet` | `Chain.HyperEvmTestnet` |

### Mainnet

| Chain | Identifier | Go Constant | TS Constant |
| :--- | :--- | :--- | :--- |
| Solana | `"solana-mainnet-beta"` | `pay.ChainSolanaMainnet` | `Chain.SolanaMainnet` |
| Base | `"base"` | `pay.ChainBase` | `Chain.Base` |
| BSC | `"bsc"` | `pay.ChainBSC` | `Chain.Bsc` |
| Polygon | `"polygon"` | `pay.ChainPolygon` | `Chain.Polygon` |
| Arbitrum | `"arbitrum"` | `pay.ChainArbitrum` | `Chain.Arbitrum` |
| Ethereum | `"ethereum"` | `pay.ChainEthereum` | `Chain.Ethereum` |
| Monad | `"monad"` | `pay.ChainMonad` | `Chain.Monad` |
| HyperEVM | `"hyperevm"` | `pay.ChainHyperEVM` | `Chain.HyperEvm` |

## Settlement Chain

**All payments settle on Base** regardless of the source chain. The `payerChain` field in `CreateIntentRequest` specifies where the payer initiates the payment, but the final USDC transfer always occurs on Base.

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
| HyperEVM | `"hyperevm-testnet"` | `"hyperevm"` |

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
