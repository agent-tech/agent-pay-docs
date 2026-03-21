# Go SDK

High-performance, zero-dependency SDK designed for backend services and microservice architectures.

### Smart Routing
The Go SDK features a unified `Client` that automatically routes requests based on the authentication options provided during initialization (`WithBearerAuth`).

* **Authenticated Mode**: Targets `/v2` endpoints for Agent-led execution.
* **Public Mode**: Targets `/api` endpoints for public intent creation.

### Multi-Chain Support
All payments settle on **Base**, while the payer can pay from multiple source chains via `payer_chain`.

Use chain constants from the SDK instead of hardcoded strings:

| Chain | Testnet Constant | Mainnet Constant |
| :--- | :--- | :--- |
| Solana | `pay.ChainSolanaDevnet` (`"solana-devnet"`) | `pay.ChainSolanaMainnet` (`"solana-mainnet-beta"`) |
| Base | `pay.ChainBaseSepolia` (`"base-sepolia"`) | `pay.ChainBase` (`"base"`) |
| BSC | `pay.ChainBSCTestnet` (`"bsc-testnet"`) | `pay.ChainBSC` (`"bsc"`) |
| Polygon | `pay.ChainPolygonAmoy` (`"polygon-amoy"`) | `pay.ChainPolygon` (`"polygon"`) |
| Arbitrum | `pay.ChainArbitrumSepolia` (`"arbitrum-sepolia"`) | `pay.ChainArbitrum` (`"arbitrum"`) |
| Ethereum | `pay.ChainEthereumSepolia` (`"ethereum-sepolia"`) | `pay.ChainEthereum` (`"ethereum"`) |
| Monad | `pay.ChainMonadTestnet` (`"monad-testnet"`) | `pay.ChainMonad` (`"monad"`) |

```go
resp, err := client.CreateIntent(ctx, &pay.CreateIntentRequest{
    Email:      "merchant@example.com",
    Amount:     "100.50",
    PayerChain: pay.ChainBase, // use constants instead of bare strings
})
```

> 🔗 [View on GitHub](https://github.com/agent-tech/AgentPay-SDK-Go)