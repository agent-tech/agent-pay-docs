---
order: 2
---

# Go SDK

High-performance, zero-dependency SDK designed for backend services and microservice architectures.

### Smart Routing

The Go SDK features a unified `Client` that automatically routes requests based on the authentication options provided during initialization (`WithBearerAuth`).

* **Authenticated Mode**: Targets `/v2` endpoints for Agent-led execution.
* **Public Mode**: Targets `/api` endpoints for public intent creation.

### Multi-Chain Support

A payment intent declares a payer chain and a target chain. `PayerChain` is required; `TargetChain` is optional and defaults to `"base"`. The merchant receives the stablecoin on the target chain.

Use chain constants from the SDK instead of hardcoded strings. Treat `GET /api/chains` as authoritative for the exact set exposed by your deployment.

| Chain | Constant | USDC Decimals | Status | Notes |
| --- | --- | --- | --- | --- |
| Solana | `pay.ChainSolanaMainnet` (`"solana-mainnet-beta"`) | 6 | Live | — |
| Base | `pay.ChainBase` (`"base"`) | 6 | Live | — |
| Ethereum | `pay.ChainEthereum` (`"ethereum"`) | 6 | Live | — |
| Polygon | `pay.ChainPolygon` (`"polygon"`) | 6 | Live | — |
| HyperEVM | `pay.ChainHyperEVM` (`"hyperevm"`) | 6 | Live | — |
| Arbitrum | `pay.ChainArbitrum` (`"arbitrum"`) | 6 | Live | — |
| BSC | `pay.ChainBSC` (`"bsc"`) | **18** | Live | Binance-Peg USDC |
| Monad | `pay.ChainMonad` (`"monad"`) | 6 | Live | — |
| SKALE Base | `pay.ChainSKALEBase` (`"skale-base"`, payer-only) | 6 | Live | Token is **USDC.e** (Bridged USDC). EIP-712 domain name `"Bridged USDC (SKALE Bridge)"`. |
| MegaETH | `pay.ChainMegaETH` (`"megaeth"`, payer-only) | **18** | Live | Native **USDm** (MegaUSD) |

```
resp, err := client.CreateIntent(ctx, &pay.CreateIntentRequest{
    Email:       "merchant@example.com",
    Amount:      "100.50",
    PayerChain:  pay.ChainBase,
    TargetChain: pay.ChainEthereum, // optional; defaults to "base"
})
```

> **BSC / MegaETH decimals**: BSC USDC and MegaETH's native USDm use 18 decimals. Always read `resp.Extra.Decimals` from the intent response rather than hardcoding `6`. See [Chain-Specific Notes](../../../introduction/supported-networks/#chain-specific-caveats) for details.

### Amount: ExactOut vs ExactIn

`CreateIntentRequest` carries the amount on exactly one of two fields — set one, never both:

* `Amount` — **ExactOut**: the merchant receives exactly this dollar amount (the payer covers it plus fees).
* `ToAmount` — **ExactIn**: the payer sends exactly this dollar amount and the merchant receives the remainder after fees.

```go
// ExactIn — payer spends exactly $5.00; omit Amount.
resp, err := client.CreateIntent(ctx, &pay.CreateIntentRequest{
    Email:      "merchant@example.com",
    ToAmount:   "5.00",
    PayerChain: pay.ChainBase,
})
```

`PayerAddress` is optional: pass the payer's wallet address to have it screened advisorily against sanctions lists at create time (the authoritative screen still runs during settlement). Leave it empty to skip.

### Assets (USDT0 / USDT)

Each intent carries an asset on both legs — `PayerAsset` (what the payer sends on `PayerChain`) and `TargetAsset` (what the merchant receives on `TargetChain`). The two are independent of each other and of the chains, so you can mix them freely. Omit either to default it to `pay.AssetUSDC`. Use the `Asset` constants instead of bare strings:

| Constant | Value | Notes |
| --- | --- | --- |
| `pay.AssetUSDC` | `"usdc"` | Circle USDC. The default when an asset is omitted. |
| `pay.AssetUSDT0` | `"usdt0"` | Tether's LayerZero USDT0 omnichain token. |
| `pay.AssetUSDT` | `"usdt"` | Native Tether USDT. Some deployments are gated by the backend operator. |

```go
// Pay USDT0 on Arbitrum, settle USDC on Base.
resp, err := client.CreateIntent(ctx, &pay.CreateIntentRequest{
    Email:       "merchant@example.com",
    Amount:      "100.50",
    PayerChain:  pay.ChainArbitrum,
    PayerAsset:  pay.AssetUSDT0,
    TargetChain: pay.ChainBase,
    TargetAsset: pay.AssetUSDC, // omit to default to USDC
})
```

There is no runtime discovery endpoint for per-chain assets, so an unsupported `(chain, asset)` pair is rejected by the backend with **HTTP 400** (a `*pay.RequestError`). `PayerAsset` / `TargetAsset` are echoed back on both `CreateIntentResponse` and `GetIntentResponse`. See the [Token × Chain matrix](../../../introduction/supported-networks/#token--chain-matrix) for current coverage.

### Intent Status Constants

Use status constants instead of raw strings when checking intent status:

| Constant | Value | Terminal |
| --- | --- | --- |
| `pay.StatusAwaitingPayment` | `"AWAITING_PAYMENT"` | No |
| `pay.StatusPending` | `"PENDING"` | No |
| `pay.StatusSourceSettled` | `"SOURCE_SETTLED"` | No |
| `pay.StatusTargetSettling` | `"TARGET_SETTLING"` | No |
| `pay.StatusTargetSettled` | `"TARGET_SETTLED"` | Yes |
| `pay.StatusVerificationFailed` | `"VERIFICATION_FAILED"` | Yes |
| `pay.StatusBlocked` | `"BLOCKED"` | Yes |
| `pay.StatusPartialSettlement` | `"PARTIAL_SETTLEMENT"` | Yes |
| `pay.StatusExpired` | `"EXPIRED"` | Yes |

> `pay.StatusBlocked` is a terminal compliance reject (sanctions / OFAC SDN hit), distinct from `pay.StatusVerificationFailed` and never retried. See [Intent Statuses](../../concepts/statuses/).

```
intent, err := client.GetIntent(ctx, intentID)
switch intent.Status {
case pay.StatusTargetSettled:
    // Payment complete — use intent.TargetPayment for receipt
    log.Printf("paid: tx=%s url=%s", intent.TargetPayment.TxHash, intent.TargetPayment.ExplorerURL)
case pay.StatusExpired, pay.StatusVerificationFailed, pay.StatusPartialSettlement:
    // Terminal failure
}
```

### Agent Identity (v2)

Two helpers expose the v2 agent surface backed by `GET /v2/me` and `GET /v2/intents/list`. Both require `WithBearerAuth`.

```
// Identity of the agent owning the API key in use.
me, err := client.GetMe(ctx)
log.Printf("agent %s (%s) base=%s solana=%s",
    me.AgentID, me.Name, me.WalletAddress, me.SolanaWalletAddress)

// Paginated list of intents owned by the calling agent.
// page is 1-indexed; pageSize must be in [1,100]. Pass 0/0 for server defaults (1, 20).
list, err := client.ListIntents(ctx, 1, 20)
for _, it := range list.Intents {
    log.Printf("%s %s→%s status=%s", it.IntentID, it.PayerChain, it.TargetChain, it.Status)
}
```

`IntentBase.AgentID` is populated on every v2 intent response (`CreateIntent`, `ExecuteIntent`, `GetIntent`, `ListIntents`) and is empty for intents created via the public `/api` flow. `GetIntent` against `/v2` enforces ownership and returns `404 payment intent not found` when the intent belongs to a different agent or has no owner.

### Swap

`GetSwapQuote`, `GetSwapApproval`, `GetSwapStatus`, `RegisterSwapIntent`, and the three discovery methods are available on every `Client` regardless of authentication mode. No API key is required.

#### ExecuteSwap (Agent Wallet)

When the agent has a Privy-hosted wallet, use `ExecuteSwap` to swap tokens without managing private keys. The SDK calls `POST /v2/swap/execute`; the backend handles quoting, ERC-20 approval, and broadcasting.

Requires `WithBearerAuth`.

```go
client, _ := pay.NewClient(
    "https://api-pay.agent.tech",
    pay.WithBearerAuth("your-api-key", "your-secret-key"),
)

resp, err := client.ExecuteSwap(ctx, &pay.ExecuteSwapRequest{
    Chain:      "base",
    FromToken:  "0x4200000000000000000000000000000000000006", // WETH
    ToToken:    "0x833589fCD6eDb6E08f4c7C32D4f71b54bdA02913", // USDC
    FromAmount: 1_000_000_000_000_000_000, // 1 WETH in wei
    // SlippageBps: 100, // optional, default 50
    // ToChain: "polygon", // optional, cross-chain
})
fmt.Println("tx:", resp.TxHash, "estimated output:", resp.EstimatedOutput)
```

`ExecuteSwapRequest` fields:

| Field | Go type | Required | Description |
| --- | --- | --- | --- |
| `Chain` | `string` | Yes | Source chain |
| `FromToken` | `string` | Yes | Input token contract address |
| `ToToken` | `string` | Yes | Output token contract address |
| `FromAmount` | `uint64` | Yes | Input amount in smallest unit (wei) |
| `SlippageBps` | `uint16` | No | Slippage in basis points (default 50) |
| `ToChain` | `string` | No | Destination chain for cross-chain swaps |

> **Native ETH**: To swap native ETH (not WETH), pass `FromToken: "0x0000000000000000000000000000000000000000"` (zero address). The backend passes this directly to LiFi, which treats the zero address as the native chain token.

> **Slippage**: For native-token swaps (ETH → USDC), the default 50 bps may be insufficient. Set `SlippageBps: 300` or higher for more reliable execution.

`ExecuteSwapResponse` fields: `TxHash`, `Chain`, `FromToken`, `ToToken`, `FromAmount` (string), `EstimatedOutput` (string).

```go
import pay "github.com/cross402/usdc-sdk-go"

client := pay.New(pay.WithBaseURL("https://api-pay.agent.tech"))

// ExactIn quote (no user address → quote only)
resp, err := client.GetSwapQuote(ctx, &pay.SwapQuoteParams{
    Chain:       "solana",
    InputToken:  "So11111111111111111111111111111111111111112",
    OutputToken: "EPjFWdd5AufqSSqeM2qN1xzybapC8G4wEGGkZwyTDt1v",
    FromAmount:  1_000_000_000,
})
fmt.Println("min out:", resp.Quote.MinOutputAmount)

// ExactIn with swap transaction (EVM)
resp, err := client.GetSwapQuote(ctx, &pay.SwapQuoteParams{
    Chain:       "base",
    InputToken:  "0x4200000000000000000000000000000000000006",
    OutputToken: "0x833589fCD6eDb6E08f4c7C32D4f71b54bdA02913",
    FromAmount:  1_000_000_000_000_000_000,
    UserAddress: "0xYourWallet",
})
// resp.SwapTransaction != nil — { Transaction (hex), To, Value, GasLimit, ExpiresAt }

// Cross-chain: Base → Solana
resp, err := client.GetSwapQuote(ctx, &pay.SwapQuoteParams{
    Chain:         "base",
    InputToken:    "0x833589fCD6eDb6E08f4c7C32D4f71b54bdA02913",
    OutputToken:   "EPjFWdd5AufqSSqeM2qN1xzybapC8G4wEGGkZwyTDt1v",
    FromAmount:    10_000_000,
    ToChain:       "solana",
    UserAddress:   "0xYourEVMWallet",
    ToUserAddress: "YourSolanaPublicKey",
})
```

When you broadcast the swap yourself (rather than via `ExecuteSwap`), check the ERC-20 allowance first with `GetSwapApproval`. It returns the approval transaction(s) to sign only when needed — Solana and native-token swaps require no approval. Some tokens (e.g. USDT) also return a `Cancel` transaction that resets the allowance to zero before the new approval can be granted.

```go
appr, err := client.GetSwapApproval(ctx, &pay.SwapApprovalParams{
    Chain:       "base",
    Token:       "0x4200000000000000000000000000000000000006", // spend token
    Amount:      1_000_000_000_000_000_000,                    // smallest unit
    UserAddress: "0xYourWallet",
    TokenOut:    "0x833589fCD6eDb6E08f4c7C32D4f71b54bdA02913", // output token
})
if appr.NeedsApproval {
    // sign & broadcast appr.Cancel (if non-nil) then appr.Approval before swapping
}
```

After signing and broadcasting the swap transaction, register it:

```go
reg, err := client.RegisterSwapIntent(ctx, &pay.RegisterSwapIntentRequest{
    SourceTxHash:       "0xabc...",
    FromChain:          "base",
    ToChain:            "base",
    FromToken:          "0x4200000000000000000000000000000000000006",
    ToToken:            "0x833589fCD6eDb6E08f4c7C32D4f71b54bdA02913",
    PayerAddress:       "0xYourWallet",
    RecipientAddress:   "0xRecipient",
    SendingTokenAmount: "1000000000000000000",
})
fmt.Println(reg.IntentID, reg.Status) // "PENDING"
```

For cross-chain swaps, poll settlement by source-chain tx hash with `GetSwapStatus`. It returns HTTP 404 (a `*pay.RequestError`) until the source chain settles — typically 30 s+ — so retry.

```go
st, err := client.GetSwapStatus(ctx, &pay.SwapStatusParams{
    TxHash:    "0xabc...",
    FromChain: "base",    // optional hints that speed up the lookup
    ToChain:   "solana",
})
fmt.Println(st.Status, st.DestTxHash) // e.g. "DONE" "0xdef..."
```

Discovery (LI.FI passthrough — returns `json.RawMessage`):

```go
tokens, err := client.GetSwapTokens(ctx, "8453,137", "")  // chains, chainTypes
chains, err := client.GetSwapChains(ctx, "EVM")
conns, err := client.GetSwapConnections(ctx, "8453", "137", "", "")
```

#### SwapJobStatus Values

| Constant | Value | Terminal |
| --- | --- | --- |
| `pay.SwapJobStatusPending` | `"PENDING"` | No |
| `pay.SwapJobStatusDone` | `"DONE"` | Yes |
| `pay.SwapJobStatusFailed` | `"FAILED"` | Yes |
| `pay.SwapJobStatusCanceled` | `"CANCELED"` | Yes |

---

### SubmitProof is `/api`-only

`SubmitProof` belongs to the unauthenticated `/api` flow. When called on a client configured with `WithBearerAuth`, the SDK rejects the call locally with `pay.ErrSubmitProofNotAllowed` instead of letting it 404 against `/v2/intents/{intent_id}`. v2 callers should rely on `ExecuteIntent` to drive settlement.

> [View on GitHub](https://github.com/cross402/usdc-sdk-go)
