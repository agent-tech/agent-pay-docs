# Go SDK

High-performance, zero-dependency SDK designed for backend services and microservice architectures.

### Smart Routing
The Go SDK features a unified `Client` that automatically routes requests based on the authentication options provided during initialization (`WithBearerAuth`).

* **Authenticated Mode**: Targets `/v2` endpoints for Agent-led execution.
* **Public Mode**: Targets `/api` endpoints for public intent creation.

### Multi-Chain Support
A payment intent declares a payer chain and a target chain. `PayerChain` is required; `TargetChain` is optional and defaults to `"base"`. The merchant receives USDC on the target chain.

Use chain constants from the SDK instead of hardcoded strings. The **Status** column indicates which chains can be called today; coming-soon constants exist in the SDK so you can compile against them, but `CreateIntent` will reject them with `400` until they go live.

| Chain | Constant | USDC Decimals | Status | Notes |
| :--- | :--- | :--- | :--- | :--- |
| Solana | `pay.ChainSolanaMainnet` (`"solana-mainnet-beta"`) | 6 | Live | — |
| Base | `pay.ChainBase` (`"base"`) | 6 | Live | — |
| Ethereum | `pay.ChainEthereum` (`"ethereum"`) | 6 | Live | — |
| Polygon | `pay.ChainPolygon` (`"polygon"`) | 6 | Live | — |
| HyperEVM | `pay.ChainHyperEVM` (`"hyperevm"`) | 6 | Live | — |
| Arbitrum | `pay.ChainArbitrum` (`"arbitrum"`) | 6 | 🚧 Coming soon | — |
| BSC | `pay.ChainBSC` (`"bsc"`) | **18** | 🚧 Coming soon | Binance-Peg USDC |
| Monad | `pay.ChainMonad` (`"monad"`) | 6 | 🚧 Coming soon | — |
| SKALE Base | `pay.ChainSkaleBase` (`"skale-base"`) | 6 | 🚧 Coming soon | EIP-712 domain name `"Bridged USDC (SKALE Bridge)"` |
| MegaETH | `pay.ChainMegaETH` (`"megaeth"`) | **18** | 🚧 Coming soon | Native USDm (MegaUSD) |

```go
resp, err := client.CreateIntent(ctx, &pay.CreateIntentRequest{
    Email:       "merchant@example.com",
    Amount:      "100.50",
    PayerChain:  pay.ChainBase,
    TargetChain: pay.ChainEthereum, // optional; defaults to "base"
})
```

> **BSC / MegaETH decimals**: BSC USDC and MegaETH's native USDm use 18 decimals. Always read `resp.Extra.Decimals` from the intent response rather than hardcoding `6`. See [Chain-Specific Notes](../api/chains.md#chain-specific-caveats) for details.

### Intent Status Constants

Use status constants instead of raw strings when checking intent status:

| Constant | Value | Terminal |
| :--- | :--- | :--- |
| `pay.StatusAwaitingPayment` | `"AWAITING_PAYMENT"` | No |
| `pay.StatusPending` | `"PENDING"` | No |
| `pay.StatusSourceSettled` | `"SOURCE_SETTLED"` | No |
| `pay.StatusTargetSettling` | `"TARGET_SETTLING"` | No |
| `pay.StatusTargetSettled` | `"TARGET_SETTLED"` | Yes |
| `pay.StatusVerificationFailed` | `"VERIFICATION_FAILED"` | Yes |
| `pay.StatusPartialSettlement` | `"PARTIAL_SETTLEMENT"` | Yes |
| `pay.StatusExpired` | `"EXPIRED"` | Yes |

```go
intent, err := client.GetIntent(ctx, intentID)
switch intent.Status {
case pay.StatusTargetSettled:
    // Payment complete — use intent.TargetPayment for receipt
    log.Printf("paid: tx=%s url=%s", intent.TargetPayment.TxHash, intent.TargetPayment.ExplorerURL)
case pay.StatusExpired, pay.StatusVerificationFailed, pay.StatusPartialSettlement:
    // Terminal failure
}
```

> [View on GitHub](https://github.com/cross402/usdc-sdk-go)
