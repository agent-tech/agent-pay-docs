---
order: 3
---

# JavaScript / TypeScript SDK

A universal SDK for Node.js (v18+) and modern browsers. Two clients: **PayClient** (server-side, with secrets) and **PublicPayClient** (client-side, no secrets).

## Install

```
npm install @cross402/usdc
```

## PayClient (Server-Side)

Use on the backend when the Agent wallet should sign and execute the transfer.

```
import { PayClient, IntentStatus } from "@cross402/usdc/server";

const client = new PayClient({
  baseUrl: "https://api-pay.agent.tech",
  auth: { apiKey: "your-api-key", secretKey: "your-secret-key" },
});

// Create intent → execute → poll until terminal status.
// Payer pays on Base; merchant receives on Ethereum.
// Use `email` to resolve to a wallet address, or pass `recipient` directly.
const { intentId } = await client.createIntent({
  email: "merchant@example.com",  // or: recipient: "0xMerchantWallet"
  amount: "100.50",
  payerChain: "base",
  targetChain: "ethereum",
});
await client.executeIntent(intentId);

const intent = await client.getIntent(intentId);
if (intent.status === IntentStatus.TargetSettled) {
  console.log("paid on", intent.targetPayment?.txHash);
  console.log("owned by agent", intent.agentId);
}
```

`targetChain` is optional; omit it to settle on Base. See the full chain list and per-chain caveats in [Supported Chains](../../../introduction/supported-networks/).

`agentId` is exposed on every v2 intent response (`createIntent`, `executeIntent`, `getIntent`) and identifies the owning agent. It is intentionally absent on `PublicPayClient` responses (the `/api` flow has no owner). On `getIntent`, the v2 endpoint also enforces ownership: a `PayClient` configured with another agent's API key receives `404 payment intent not found`, not the intent.

The settlement quote (`sendingAmount`, `receivingAmount`, `estimatedFee`, `feeBreakdown`) is always present on `createIntent` / `executeIntent` responses but **optional** on `GetIntentResponse` — the backend only fills these in once the intent leaves its initial state. Always null-check them when polling.

### Amount: ExactOut vs ExactIn

`createIntent` takes the amount on exactly one of two fields — set one, never both:

* `amount` — **ExactOut**: the merchant receives exactly this dollar amount (the payer covers it plus fees).
* `toAmount` — **ExactIn**: the payer sends exactly this dollar amount and the merchant receives the remainder after fees.

```typescript
// ExactIn — payer spends exactly $5.00; omit `amount`.
const intent = await client.createIntent({
  email: "merchant@example.com",
  toAmount: "5.00",
  payerChain: "base",
  targetChain: "ethereum",
});
```

Pass the optional `payerAddress` to have the payer wallet screened advisorily against sanctions lists at create time (the authoritative screen still runs during settlement).

### Discovering enabled chains

Both clients expose `listSupportedChains()` (backed by `GET /api/chains`):

```
const { chains, targetChains } = await client.listSupportedChains();
// chains       — valid as payerChain
// targetChains — valid as targetChain
```

### Agent identity (v2)

`PayClient` exposes the v2 agent surface backed by `GET /v2/me` and `GET /v2/intents/list`:

```
// Identity of the agent owning the API key in use.
const me = await client.getMe();
console.log(`agent ${me.agentId} (${me.name}) wallet=${me.walletAddress}`);

// Paginated list of intents owned by the calling agent (most recent first).
// page is 1-indexed; pageSize ∈ [1,100]. Both default server-side (1, 20)
// when omitted; out-of-range values throw PayValidationError.
const page = await client.listIntents({ page: 1, pageSize: 20 });
for (const it of page.intents) {
  console.log(`${it.intentId} ${it.status} ${it.payerChain}→${it.targetChain}`);
}
```

## PublicPayClient (Client-Side)

Use in the browser or payer-side when the user holds their own wallet and will sign an X402 payment and submit the settle proof.

```
import { PublicPayClient } from "@cross402/usdc/client";

const client = new PublicPayClient({
  baseUrl: "https://api-pay.agent.tech",
});

const { intentId } = await client.createIntent({
  recipient: "0x...",
  amount: "10.00",
  payerChain: "base",
  targetChain: "ethereum",
});
// ... payer signs X402 payment off-chain ...
await client.submitProof(intentId, settleProof);
const intent = await client.getIntent(intentId);
```

## CLI

```
npm install -g @cross402/usdc

# Configure credentials
cross402-usdc auth set --api-key <key> --secret-key <key> --base-url https://api-pay.agent.tech
cross402-usdc auth show          # Display current config (secret key masked)
cross402-usdc auth clear         # Remove stored config

# Server flow: create → execute → get
cross402-usdc intent create --amount 10.00 --payer-chain base --target-chain ethereum --email merchant@example.com
cross402-usdc intent execute <intent-id>
cross402-usdc intent get <intent-id>

# Client flow (no auth): submit proof only
cross402-usdc intent submit-proof <intent-id> --proof <settle-proof>

# Session & balance management
cross402-usdc intent sessions [--expired]   # List stored intent sessions
cross402-usdc balance read --address <addr> [--rpc-url <url>]  # Read USDC balance on Base
cross402-usdc reset [--yes]                 # Remove all stored config + sessions
```

## Swap

Both `PayClient` and `PublicPayClient` expose the full swap surface. No authentication is required for any swap method.

### executeSwap (Agent Wallet)

When the agent has a Privy-hosted wallet, `executeSwap` performs the entire swap flow — quoting, ERC-20 approval, and broadcasting — with no private key needed. Available on `PayClient` only (requires auth).

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
  // slippageBps: 100, // optional, default 50
  // toChain: "polygon", // optional, cross-chain
});

console.log("tx:", result.txHash);
console.log("estimated output:", result.estimatedOutput);
```

`ExecuteSwapRequest` fields:

| Field | Type | Required | Description |
| --- | --- | --- | --- |
| `chain` | `string` | Yes | Source chain identifier |
| `fromToken` | `string` | Yes | Input token contract address |
| `toToken` | `string` | Yes | Output token contract address |
| `fromAmount` | `number` | Yes | Input amount in smallest unit (wei) |
| `slippageBps` | `number` | No | Slippage tolerance in basis points (default 50, max 500) |
| `toChain` | `string` | No | Destination chain for cross-chain swaps |

> **Native ETH**: To swap native ETH (not WETH), pass `fromToken: "0x0000000000000000000000000000000000000000"` (zero address). The backend passes this directly to LiFi, which treats the zero address as the native chain token.

> **Slippage**: For native-token swaps (ETH → USDC), the default 50 bps may be insufficient. Set `slippageBps: 300` or higher for more reliable execution.

`ExecuteSwapResponse` fields: `txHash`, `chain`, `fromToken`, `toToken`, `fromAmount` (string), `estimatedOutput` (string).

### getSwapQuote

Fetch a price quote. Set `fromAmount` for ExactIn (you spend a fixed input) or `toAmount` for ExactOut (you receive a fixed output). Pass `userAddress` to include a ready-to-sign `swapTransaction` in the response.

```typescript
import { PublicPayClient } from "@cross402/usdc";

const client = new PublicPayClient({ baseUrl: "https://api-pay.agent.tech" });

// ExactIn: 1 WETH → USDC on Base (quote only)
const { quote } = await client.getSwapQuote({
  chain: "base",
  inputToken: "0x4200000000000000000000000000000000000006",
  outputToken: "0x833589fCD6eDb6E08f4c7C32D4f71b54bdA02913",
  fromAmount: 1_000_000_000_000_000_000,
});
console.log("min out:", quote.minOutputAmount);

// ExactIn with swap transaction (ready to sign)
const result = await client.getSwapQuote({
  chain: "base",
  inputToken: "0x4200000000000000000000000000000000000006",
  outputToken: "0x833589fCD6eDb6E08f4c7C32D4f71b54bdA02913",
  fromAmount: 1_000_000_000_000_000_000,
  userAddress: "0xYourWallet",
});
// result.swapTransaction: { transaction (hex), to, value, gasLimit, expiresAt }

// Cross-chain: Base → Solana
const cross = await client.getSwapQuote({
  chain: "base",
  inputToken: "0x833589fCD6eDb6E08f4c7C32D4f71b54bdA02913",
  outputToken: "EPjFWdd5AufqSSqeM2qN1xzybapC8G4wEGGkZwyTDt1v",
  fromAmount: 10_000_000,
  toChain: "solana",
  userAddress: "0xYourEVMWallet",
  toUserAddress: "YourSolanaPublicKey",
});
```

> **Address resolution by email**: instead of `userAddress` / `toUserAddress` you can pass `email` / `toUserEmail`, and the backend resolves each to the corresponding wallet address. An explicit address wins when both are supplied.

### getSwapApproval

When you broadcast the swap yourself (instead of using `executeSwap`), check the ERC-20 allowance first. `getSwapApproval` returns the approval transaction(s) to sign only when `needsApproval` is `true` — Solana and native-token swaps need none. Some tokens (e.g. USDT) also return a `cancel` transaction that resets the allowance to zero before the new approval can be granted. Pass `userAddress` or `email`.

```typescript
const appr = await client.getSwapApproval({
  chain: "base",
  token: "0x4200000000000000000000000000000000000006", // spend token
  amount: 1_000_000_000_000_000_000,                   // smallest unit
  tokenOut: "0x833589fCD6eDb6E08f4c7C32D4f71b54bdA02913", // output token
  userAddress: "0xYourWallet", // or: email: "you@example.com"
});
if (appr.needsApproval) {
  // sign & broadcast appr.cancel (if present) then appr.approval before swapping
}
```

### registerSwapIntent

After the swap transaction is broadcast on-chain, register its hash to have Cross402 track settlement. Returns an `intentId` you can poll with `getIntent`.

```typescript
const { intentId, status } = await client.registerSwapIntent({
  sourceTxHash: "0xabc...",
  fromChain: "base",
  toChain: "base",
  fromToken: "0x4200000000000000000000000000000000000006",
  toToken: "0x833589fCD6eDb6E08f4c7C32D4f71b54bdA02913",
  payerAddress: "0xYourWallet",
  recipientAddress: "0xRecipient",
  sendingTokenAmount: "1000000000000000000",
});
// status: "PENDING" — poll with client.getIntent(intentId)
```

### getSwapStatus

For cross-chain swaps, poll settlement by source-chain transaction hash. The call returns HTTP 404 (a `PayRequestError`) until the source chain settles — typically 30 s+ — so retry. `fromChain` / `toChain` / `bridge` are optional hints that speed up the lookup.

```typescript
const status = await client.getSwapStatus({
  txHash: "0xabc...",
  fromChain: "base",
  toChain: "solana",
});
console.log(status.status, status.destTxHash); // e.g. "DONE" "0xdef..."
```

### Discovery

Enumerate tokens, chains, and routes supported by LI.FI (raw JSON passthrough).

```typescript
// Tokens on Base and Polygon
const tokens = await client.getSwapTokens({ chains: "8453,137" });

// All EVM chains
const chains = await client.getSwapChains({ chainTypes: "EVM" });

// Routes from Base USDC → Polygon USDT
const connections = await client.getSwapConnections({
  fromChain: "8453",
  toChain: "137",
  fromToken: "0x833589fCD6eDb6E08f4c7C32D4f71b54bdA02913",
});
```

### SwapJobStatus Constants

```typescript
import { SwapJobStatus } from "@cross402/usdc";
```

| Constant | Value | Terminal |
| --- | --- | --- |
| `SwapJobStatus.Pending` | `"PENDING"` | No |
| `SwapJobStatus.Done` | `"DONE"` | Yes |
| `SwapJobStatus.Failed` | `"FAILED"` | Yes |
| `SwapJobStatus.Canceled` | `"CANCELED"` | Yes |

### Asset Constants

Use `Asset` when setting `payerAsset` / `targetAsset` on `createIntent`, or when constructing swap token pairs.

```typescript
import { Asset } from "@cross402/usdc";

await client.createIntent({
  email: "merchant@example.com",
  amount: "10.00",
  payerChain: "arbitrum",
  payerAsset: Asset.USDT0,  // "usdt0"
  targetChain: "base",
});
```

| Constant | Value |
| --- | --- |
| `Asset.USDC` | `"usdc"` |
| `Asset.USDT` | `"usdt"` |
| `Asset.USDT0` | `"usdt0"` |

---

## Intent Status Constants

```
import { IntentStatus } from "@cross402/usdc";
```

| Constant | Value | Terminal |
| --- | --- | --- |
| `IntentStatus.AwaitingPayment` | `"AWAITING_PAYMENT"` | No |
| `IntentStatus.Pending` | `"PENDING"` | No |
| `IntentStatus.SourceSettled` | `"SOURCE_SETTLED"` | No |
| `IntentStatus.TargetSettling` | `"TARGET_SETTLING"` | No |
| `IntentStatus.TargetSettled` | `"TARGET_SETTLED"` | Yes |
| `IntentStatus.VerificationFailed` | `"VERIFICATION_FAILED"` | Yes |
| `IntentStatus.PartialSettlement` | `"PARTIAL_SETTLEMENT"` | Yes |
| `IntentStatus.Expired` | `"EXPIRED"` | Yes |

> The backend can also return `"BLOCKED"` — a terminal compliance reject (sanctions screen hit). The SDK does not yet ship a constant for it, so compare `intent.status` against the literal `"BLOCKED"`. See [Intent Statuses](../../concepts/statuses/).

## Skills

Install the Cross402-usdc CLI skill for Cursor (skills.sh) or Clawhub:

```
npx skills add cross402/usdc-sdk-js-ts
# or
clawhub skills add cross402/usdc-sdk-js-ts
```

> [View on GitHub](https://github.com/cross402/usdc-sdk-js-ts)
