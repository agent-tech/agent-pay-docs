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
const { intentId } = await client.createIntent({
  email: "merchant@example.com",
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

## Skills

Install the Cross402-usdc CLI skill for Cursor (skills.sh) or Clawhub:

```
npx skills add cross402/usdc-sdk-js-ts
# or
clawhub skills add cross402/usdc-sdk-js-ts
```

> [View on GitHub](https://github.com/cross402/usdc-sdk-js-ts)
