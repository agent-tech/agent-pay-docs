# Go SDK

High-performance, zero-dependency SDK designed for backend services and microservice architectures.

### Smart Routing
The Go SDK features a unified `Client` that automatically routes requests based on the authentication options provided during initialization (`WithBearerAuth`).

* **Authenticated Mode**: Targets `/v2` endpoints for Agent-led execution.
* **Public Mode**: Targets `/api` endpoints for public intent creation.

### Multi-Chain Support
All payments settle on **Base**, while the payer can pay from multiple source chains via `payer_chain`.

Use chain constants from the SDK instead of hardcoded strings:

| Chain | Constant | Status |
| :--- | :--- | :--- |
| Solana | `pay.ChainSolanaMainnet` (`"solana-mainnet-beta"`) | Available |
| Base | `pay.ChainBase` (`"base"`) | Available |
| Polygon | `pay.ChainPolygon` (`"polygon"`) | Under maintenance |
| Arbitrum | `pay.ChainArbitrum` (`"arbitrum"`) | Under maintenance |
| Ethereum | `pay.ChainEthereum` (`"ethereum"`) | Under maintenance |
| Monad | `pay.ChainMonad` (`"monad"`) | Under maintenance |
| HyperEVM | `pay.ChainHyperEVM` (`"hyperevm"`) | Under maintenance |

```go
resp, err := client.CreateIntent(ctx, &pay.CreateIntentRequest{
    Email:      "merchant@example.com",
    Amount:     "100.50",
    PayerChain: pay.ChainBase, // use constants instead of bare strings
})
```

### Intent Status Constants

Use status constants instead of raw strings when checking intent status:

| Constant | Value |
| :--- | :--- |
| `pay.StatusAwaitingPayment` | `"AWAITING_PAYMENT"` |
| `pay.StatusPending` | `"PENDING"` |
| `pay.StatusVerificationFailed` | `"VERIFICATION_FAILED"` |
| `pay.StatusSourceSettled` | `"SOURCE_SETTLED"` |
| `pay.StatusBaseSettling` | `"BASE_SETTLING"` |
| `pay.StatusBaseSettled` | `"BASE_SETTLED"` |
| `pay.StatusPartialSettlement` | `"PARTIAL_SETTLEMENT"` |
| `pay.StatusExpired` | `"EXPIRED"` |

```go
intent, err := client.GetIntent(ctx, intentID)
switch intent.Status {
case pay.StatusBaseSettled:
    // Payment complete
case pay.StatusExpired, pay.StatusVerificationFailed:
    // Terminal failure
}
```

> [View on GitHub](https://github.com/cross402/usdc-sdk-go)
