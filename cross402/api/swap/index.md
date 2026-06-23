---
order: 5
---

# Swap

The Swap API lets agents convert any supported token into USDC (or another stablecoin) as part of a payment flow. Same-chain swaps use Jupiter (Solana) or LI.FI (EVM); cross-chain swaps route through LI.FI's bridging layer. After the swap transaction is signed and broadcast, register the source transaction hash with `POST /api/swap/intents` to have Cross402 track and settle the receiving leg.

## ExecuteSwap

Execute a token swap on behalf of the authenticated agent's Privy-hosted wallet. No private key or transaction signing is required — the backend handles quoting, ERC-20 approval, and broadcasting via Privy.

`POST /v2/swap/execute`

**Authentication required.** Use your API key in the `Authorization: Bearer <base64(apiKey:secretKey)>` header.

This endpoint is EVM-only. The agent must have an active Privy-hosted wallet (`wallet_address` and a Privy wallet ID set on the account).

### Request Body

| Field | Required | Type | Default | Description |
| --- | --- | --- | --- | --- |
| `chain` | Yes | string | — | Source chain identifier. See [Supported Chains](../../../introduction/supported-networks/). |
| `from_token` | Yes | string | — | Token contract address to swap from. |
| `to_token` | Yes | string | — | Token contract address to swap to. |
| `from_amount` | Yes | uint64 | — | Input amount in smallest unit (wei). |
| `slippage_bps` | No | uint16 | `50` | Slippage tolerance in basis points (max 500). |
| `to_chain` | No | string | same as `chain` | Destination chain for cross-chain swaps. |

### Response (200)

```json
{
  "tx_hash": "0xabc123...",
  "chain": "base",
  "from_token": "0x4200000000000000000000000000000000000006",
  "to_token": "0x833589fCD6eDb6E08f4c7C32D4f71b54bdA02913",
  "from_amount": "1000000000000000000",
  "estimated_output": "2498123456"
}
```

| Field | Type | Description |
| --- | --- | --- |
| `tx_hash` | string | On-chain transaction hash of the broadcasted swap |
| `chain` | string | Source chain |
| `from_token` | string | Input token address |
| `to_token` | string | Output token address |
| `from_amount` | string | Input amount in smallest unit (decimal string) |
| `estimated_output` | string | Expected output amount in smallest unit (decimal string) |

### SDK Examples

**JavaScript / TypeScript**

```typescript
import { PayClient } from "@cross402/usdc/server";

const client = new PayClient({
  baseUrl: "https://api-pay.agent.tech",
  auth: { apiKey: "your-api-key", secretKey: "your-secret-key" },
});

const result = await client.executeSwap({
  chain: "base",
  fromToken: "0x4200000000000000000000000000000000000006", // WETH
  toToken: "0x833589fCD6eDb6E08f4c7C32D4f71b54bdA02913",   // USDC
  fromAmount: 1_000_000_000_000_000_000, // 1 WETH in wei
});

console.log("tx:", result.txHash);
console.log("estimated output:", result.estimatedOutput);
```

**Go**

```go
import pay "github.com/cross402/usdc-sdk-go"

client, _ := pay.NewClient(
    "https://api-pay.agent.tech",
    pay.WithBearerAuth("your-api-key", "your-secret-key"),
)

resp, err := client.ExecuteSwap(ctx, &pay.ExecuteSwapRequest{
    Chain:      "base",
    FromToken:  "0x4200000000000000000000000000000000000006",
    ToToken:    "0x833589fCD6eDb6E08f4c7C32D4f71b54bdA02913",
    FromAmount: 1_000_000_000_000_000_000,
})
if err != nil {
    log.Fatal(err)
}
fmt.Println("tx:", resp.TxHash)
fmt.Println("estimated output:", resp.EstimatedOutput)
```

### Error Reference

| Status | Error | Cause |
| --- | --- | --- |
| 401 | `api key required` | Missing or invalid API key |
| 403 | `agent wallet not ready` | Agent has no Privy-hosted wallet configured |
| 400 | `chain is required` | Missing `chain` field |
| 400 | `from_token is required` | Missing `from_token` field |
| 400 | `to_token is required` | Missing `to_token` field |
| 400 | `from_amount must be greater than 0` | Zero or missing `from_amount` |
| 400 | `slippage_bps exceeds maximum allowed value` | Slippage > 500 bps |
| 400 | `<chain> has no CAIP-2 identifier` | Chain not supported for agent execution |
| 502 | `failed to send transaction via Privy` | Privy wallet signing error |
| 502 | `swap service temporarily unavailable` | Upstream LI.FI error |

---

## GetSwapQuote

Fetches a price quote for a token swap.

`GET /api/swap/quote`

Exactly one of `from_amount` or `to_amount` must be set. `from_amount` requests an **ExactIn** quote (you spend a fixed input); `to_amount` requests an **ExactOut** quote (you receive a fixed output).

When `user_address` is supplied the response includes a ready-to-sign `swap_transaction`. Omit it to get a quote only.

### Query Parameters

| Parameter | Required | Type | Default | Description |
| --- | --- | --- | --- | --- |
| `chain` | Yes | string | — | Source chain identifier. See [Supported Chains](../../../introduction/supported-networks/). |
| `input_token` | Yes | string | — | Token to swap from. Solana: mint address. EVM: contract address. |
| `output_token` | Yes | string | — | Token to swap to. Usually the USDC address on the target chain. |
| `from_amount` | One of | uint64 | — | ExactIn: input amount in smallest unit (lamports / wei). Mutually exclusive with `to_amount`. |
| `to_amount` | One of | uint64 | — | ExactOut: desired output amount in smallest unit. Mutually exclusive with `from_amount`. |
| `slippage_bps` | No | uint16 | 50 | Slippage tolerance in basis points (max 500 = 5%). |
| `to_chain` | No | string | same as `chain` | Destination chain for cross-chain swaps. |
| `user_address` | No | string | — | Signer's wallet address. When set (or resolved from `email`), `swap_transaction` is returned. |
| `to_user_address` | No | string | — | Recipient address on the destination chain. Required for cross-family routes (e.g. Solana → EVM) when `user_address` is provided. |
| `email` | No | string | — | Email to resolve into `user_address`. Ignored when `user_address` is set. |
| `to_user_email` | No | string | — | Email to resolve into `to_user_address`. Ignored when `to_user_address` is set. |

> **Legacy alias:** `amount` is accepted as a deprecated alias for `from_amount`. Do not mix `amount` with `from_amount` or `to_amount` in the same request.

### Response

```json
{
  "quote": {
    "input_token": "So11111111111111111111111111111111111111112",
    "output_token": "EPjFWdd5AufqSSqeM2qN1xzybapC8G4wEGGkZwyTDt1v",
    "input_amount": "1000000000",
    "output_amount": "25000000",
    "min_output_amount": "24875000",
    "price_impact_pct": "0.05"
  },
  "swap_transaction": {
    "transaction": "<base64 or hex>",
    "expires_at": 1748649600,
    "last_valid_block_height": 12345678
  }
}
```

#### `quote` fields

| Field | Type | Description |
| --- | --- | --- |
| `input_token` | string | Input token address |
| `output_token` | string | Output token address |
| `input_amount` | string | Actual input amount in smallest unit |
| `output_amount` | string | Expected output in smallest unit |
| `min_output_amount` | string | Minimum output after slippage — use this for display |
| `price_impact_pct` | string | Price impact as a percentage string (e.g. `"0.05"` = 0.05%) |

#### `swap_transaction` fields (only when `user_address` is provided)

| Field | Type | Description |
| --- | --- | --- |
| `transaction` | string | Serialized transaction: base64 VersionedTransaction (Solana) or hex calldata (EVM) |
| `to` | string | Contract to call (EVM only) |
| `value` | string | Native token value to send (EVM only; for native-token input swaps) |
| `gas_limit` | string | Suggested gas limit as hex string (EVM only) |
| `expires_at` | int64 | Unix timestamp when this transaction expires |
| `last_valid_block_height` | uint64 | Last valid block height (Solana only) |

### SDK Examples

**JavaScript / TypeScript**

```typescript
import { PublicPayClient } from "@cross402/usdc";

const client = new PublicPayClient({ baseUrl: "https://api-pay.agent.tech" });

// ExactIn: swap 1 SOL → USDC on Solana (quote only)
const { quote } = await client.getSwapQuote({
  chain: "solana",
  inputToken: "So11111111111111111111111111111111111111112",
  outputToken: "EPjFWdd5AufqSSqeM2qN1xzybapC8G4wEGGkZwyTDt1v",
  fromAmount: 1_000_000_000,
});
console.log("expected out:", quote.outputAmount, "min:", quote.minOutputAmount);

// ExactIn with transaction (ready to sign)
const result = await client.getSwapQuote({
  chain: "base",
  inputToken: "0x4200000000000000000000000000000000000006", // WETH
  outputToken: "0x833589fCD6eDb6E08f4c7C32D4f71b54bdA02913", // USDC
  fromAmount: 1_000_000_000_000_000_000n,
  userAddress: "0xYourEVMWallet",
});
// result.swapTransaction contains { transaction, to, value, gasLimit, expiresAt }

// Cross-chain: Base → Solana
const crossQuote = await client.getSwapQuote({
  chain: "base",
  inputToken: "0x833589fCD6eDb6E08f4c7C32D4f71b54bdA02913",
  outputToken: "EPjFWdd5AufqSSqeM2qN1xzybapC8G4wEGGkZwyTDt1v",
  fromAmount: 10_000_000, // 10 USDC
  toChain: "solana",
  userAddress: "0xYourEVMWallet",
  toUserAddress: "YourSolanaWalletPublicKey",
});
```

**Go**

```go
import pay "github.com/cross402/usdc-sdk-go"

client := pay.New(pay.WithBaseURL("https://api-pay.agent.tech"))

// ExactIn: swap 1 SOL → USDC on Solana (quote only)
resp, err := client.GetSwapQuote(ctx, &pay.SwapQuoteParams{
    Chain:       "solana",
    InputToken:  "So11111111111111111111111111111111111111112",
    OutputToken: "EPjFWdd5AufqSSqeM2qN1xzybapC8G4wEGGkZwyTDt1v",
    FromAmount:  1_000_000_000,
})
fmt.Println("min out:", resp.Quote.MinOutputAmount)

// ExactIn with transaction
resp, err := client.GetSwapQuote(ctx, &pay.SwapQuoteParams{
    Chain:       "base",
    InputToken:  "0x4200000000000000000000000000000000000006",
    OutputToken: "0x833589fCD6eDb6E08f4c7C32D4f71b54bdA02913",
    FromAmount:  1_000_000_000_000_000_000,
    UserAddress: "0xYourEVMWallet",
})
// resp.SwapTransaction != nil — sign and broadcast it
```

---

## GetSwapApproval

Checks whether the user's wallet has sufficient ERC-20 allowance for the swap and, if not, returns the approval transaction to sign first.

`GET /api/swap/approval`

**EVM only.** Solana does not require token approvals.

### Query Parameters

| Parameter | Required | Type | Description |
| --- | --- | --- | --- |
| `chain` | Yes | string | Source chain identifier |
| `token` | Yes | string | Token contract address to approve |
| `token_out` | Yes | string | Output token address (used to resolve the spender) |
| `amount` | Yes | uint64 | Amount in smallest unit that will be spent |
| `user_address` | One of user_address/email | string | Signer's wallet address |
| `email` | One of user_address/email | string | Email to resolve into `user_address`. Ignored when `user_address` is set. |
| `to_chain` | No | string | Destination chain for cross-chain swaps |
| `to_user_address` | No | string | Destination recipient |
| `to_user_email` | No | string | Email to resolve into `to_user_address`. Ignored when `to_user_address` is set. |
| `include_gas_info` | No | bool | When `true`, includes `gas_fee` and `cancel_gas_fee` estimates |

### Response

```json
{
  "request_id": "abc123",
  "needs_approval": true,
  "gas_fee": "0.001",
  "cancel_gas_fee": "0.0005",
  "approval": {
    "to": "0xTokenContract",
    "from": "0xYourWallet",
    "data": "0x...",
    "value": "0x0",
    "chain_id": 8453,
    "gas_limit": "0x186A0"
  },
  "cancel": { ... }
}
```

When `needs_approval` is `false`, the `approval` field is absent — proceed directly to `GetSwapQuote` with `user_address`.

---

## GetSwapStatus

Polls the status of a submitted cross-chain swap. Backed by LI.FI's `/v1/status` endpoint.

`GET /api/swap/status`

Returns `404` when LI.FI has no record yet. Retry after source-chain settlement (typically 30 s+).

### Query Parameters

| Parameter | Required | Type | Description |
| --- | --- | --- | --- |
| `tx_hash` | Yes | string | Source-chain transaction hash |
| `from_chain` | No | string | Source chain identifier |
| `to_chain` | No | string | Destination chain identifier |
| `bridge` | No | string | Bridge identifier (LI.FI `tool`) |

### Response

```json
{
  "status": "DONE",
  "substatus": "COMPLETED",
  "message": "Transfer completed",
  "tool": "lifi",
  "source_tx_hash": "0xabc...",
  "dest_tx_hash": "0xdef...",
  "received_amount": "9980000",
  "explorer_link": "https://..."
}
```

Possible `status` values: `PENDING`, `DONE`, `FAILED`, `INVALID`, `NOT_FOUND`.

---

## RegisterSwapIntent

After the user signs and broadcasts the swap transaction, call this endpoint to register it as a Cross402 payment intent. The backend monitors the source chain, tracks cross-chain settlement, and makes the intent queryable via the standard intent endpoints.

`POST /api/swap/intents`

### Request Body

All fields are required.

| Field | JSON | Type | Description |
| --- | --- | --- | --- |
| SourceTxHash | `source_tx_hash` | string | Broadcast swap transaction hash |
| FromChain | `from_chain` | string | Source chain identifier |
| ToChain | `to_chain` | string | Destination chain identifier |
| FromToken | `from_token` | string | Input token address |
| ToToken | `to_token` | string | Output token address |
| PayerAddress | `payer_address` | string | Signer's wallet address |
| RecipientAddress | `recipient_address` | string | Token recipient on the destination chain |
| SendingTokenAmount | `sending_token_amount` | string | Amount sent, in smallest unit as a decimal string |

### Response (201)

```json
{
  "intent_id": "int_abc123",
  "status": "PENDING"
}
```

Use `intent_id` with `GET /v2/intents/{intent_id}` (authenticated) or `GET /api/intents/{intent_id}` (public) to poll settlement. Terminal statuses are `DONE` and `FAILED`; `CANCELED` means the job was dropped.

### SDK Examples

**JavaScript / TypeScript**

```typescript
const { intentId, status } = await client.registerSwapIntent({
  sourceTxHash: "0xabc...",
  fromChain: "base",
  toChain: "solana",
  fromToken: "0x833589fCD6eDb6E08f4c7C32D4f71b54bdA02913",
  toToken: "EPjFWdd5AufqSSqeM2qN1xzybapC8G4wEGGkZwyTDt1v",
  payerAddress: "0xYourEVMWallet",
  recipientAddress: "YourSolanaWalletPublicKey",
  sendingTokenAmount: "10000000",
});
console.log(intentId, status); // "PENDING"
```

**Go**

```go
resp, err := client.RegisterSwapIntent(ctx, &pay.RegisterSwapIntentRequest{
    SourceTxHash:       "0xabc...",
    FromChain:          "base",
    ToChain:            "solana",
    FromToken:          "0x833589fCD6eDb6E08f4c7C32D4f71b54bdA02913",
    ToToken:            "EPjFWdd5AufqSSqeM2qN1xzybapC8G4wEGGkZwyTDt1v",
    PayerAddress:       "0xYourEVMWallet",
    RecipientAddress:   "YourSolanaWalletPublicKey",
    SendingTokenAmount: "10000000",
})
fmt.Println(resp.IntentID, resp.Status) // "PENDING"
```

---

## Discovery Endpoints

These three endpoints are LI.FI passthroughs that return raw JSON. Use them to enumerate tokens, chains, and routes before building a swap UI.

### GetSwapTokens

`GET /api/swap/tokens`

Returns LI.FI's `/v1/tokens` response. Useful for building a token picker.

| Parameter | Required | Description |
| --- | --- | --- |
| `chains` | No | Comma-separated chain IDs to filter (e.g. `1,137,8453`) |
| `chainTypes` | No | Chain type filter (e.g. `EVM`, `SVM`) |

**JS/TS:**
```typescript
const tokens = await client.getSwapTokens({ chains: "8453,137" });
```

**Go:**
```go
raw, err := client.GetSwapTokens(ctx, "8453,137", "")
// raw is []byte of the LiFi JSON response
```

### GetSwapChains

`GET /api/swap/chains`

Returns LI.FI's `/v1/chains` response. Useful for building a chain picker.

| Parameter | Required | Description |
| --- | --- | --- |
| `chainTypes` | No | Filter to `EVM` or `SVM` |

**JS/TS:**
```typescript
const chains = await client.getSwapChains();
```

**Go:**
```go
raw, err := client.GetSwapChains(ctx, "")
```

### GetSwapConnections

`GET /api/swap/connections`

Returns LI.FI's `/v1/connections` response. Shows which token→token routes are available between two chains.

| Parameter | Required | Description |
| --- | --- | --- |
| `fromChain` | No | Source chain ID |
| `toChain` | No | Destination chain ID |
| `fromToken` | No | Input token address |
| `toToken` | No | Output token address |

**JS/TS:**
```typescript
const connections = await client.getSwapConnections({
  fromChain: "8453",
  toChain: "137",
});
```

**Go:**
```go
raw, err := client.GetSwapConnections(ctx, "8453", "137", "", "")
```

---

## Error Reference

| Status | Error string | Cause |
| --- | --- | --- |
| 400 | `chain parameter is required` | Missing `chain` |
| 400 | `input_token parameter is required` | Missing `input_token` |
| 400 | `output_token parameter is required` | Missing `output_token` |
| 400 | `from_amount or to_amount is required` | Neither amount field set |
| 400 | `from_amount and to_amount are mutually exclusive` | Both amount fields set |
| 400 | `invalid amount: must be a positive integer` | Non-integer or zero amount |
| 400 | `slippage_bps exceeds maximum allowed value` | Slippage > 500 bps |
| 400 | `unsupported chain for swaps: <chain>` | Chain not supported |
| 400 | `no swap route found` | No liquidity path |
| 400 | `insufficient liquidity for swap` | Pool too thin |
| 400 | `to_user_address is required for cross-family routes` | Cross-family swap without destination address |
| 400 | `source_tx_hash is required` | Missing field on POST /swap/intents |
| 404 | `transfer not found` | LI.FI has no record yet — retry after ~30 s |
| 502 | `swap service temporarily unavailable` | Upstream DEX / LI.FI error |

## Common Token Addresses

| Chain | Token | Address |
| --- | --- | --- |
| Solana | SOL (Wrapped) | `So11111111111111111111111111111111111111112` |
| Solana | USDC | `EPjFWdd5AufqSSqeM2qN1xzybapC8G4wEGGkZwyTDt1v` |
| Base | WETH | `0x4200000000000000000000000000000000000006` |
| Base | USDC | `0x833589fCD6eDb6E08f4c7C32D4f71b54bdA02913` |
| Ethereum | WETH | `0xC02aaA39b223FE8D0A0e5C4F27eAD9083C756Cc2` |
| Ethereum | USDC | `0xA0b86991c6218b36c1d19D4a2e9Eb0cE3606eB48` |
| BSC | WBNB | `0xbb4CdB9CBd36B01bD1cBaEBF2De08d9173bc095c` |
| BSC | USDC | `0x8AC76a51cc950d9822D68b83fE1Ad97B32Cd580d` |
| Polygon | WMATIC | `0x0d500B1d8E8eF31E21C99d1Db9A6444d3ADf1270` |
| Polygon | USDC | `0x3c499c542cEF5E3811e1192ce70d8cC03d5c3359` |
| Arbitrum | WETH | `0x82aF49447D8a07e3bd95BD0d56f35241523fBab1` |
| Arbitrum | USDC | `0xaf88d065e77c8cC2239327C5EDb3A432268e5831` |

## Amount Units

| Chain | Token | Decimals | Example: 1 token |
| --- | --- | --- | --- |
| Solana | SOL | 9 | `1000000000` |
| Solana | USDC | 6 | `1000000` |
| EVM | WETH / WBNB / WMATIC | 18 | `1000000000000000000` |
| EVM | USDC | 6 | `1000000` |
