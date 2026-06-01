---
order: 9
---

# Swap Payment — Token Swap + Settlement Workflow

Full end-to-end flow for swapping any supported token into a stablecoin and tracking settlement through Cross402. The SDK handles quoting and intent registration; signing the swap transaction is the agent's (or user's) responsibility.

**Use Case:** An agent holds a non-USDC token and needs to pay a merchant in USDC. Or an agent performs a cross-chain swap as part of a payment flow. The swap is routed through Jupiter (Solana) or LI.FI (EVM / cross-chain); once the source transaction is broadcast, Cross402 tracks delivery.

**Authentication:** Not required. `POST /api/swap/intents` is public.

---

## JSON Schema Definition

```json
{
  "name": "swap_payment",
  "description": "Swap a token into USDC (or another stablecoin) and register the result as a Cross402 payment intent for tracked settlement. Steps: quote → sign → broadcast → register → poll.",
  "input_schema": {
    "type": "object",
    "properties": {
      "chain": {
        "type": "string",
        "description": "Source chain identifier (e.g. 'solana', 'base', 'bsc')"
      },
      "input_token": {
        "type": "string",
        "description": "Token to swap from — mint address (Solana) or contract address (EVM)"
      },
      "output_token": {
        "type": "string",
        "description": "Token to swap to — typically USDC on the target chain"
      },
      "from_amount": {
        "type": "integer",
        "description": "ExactIn amount in smallest unit (lamports or wei). Mutually exclusive with to_amount."
      },
      "to_amount": {
        "type": "integer",
        "description": "ExactOut amount in smallest unit. Mutually exclusive with from_amount."
      },
      "user_address": {
        "type": "string",
        "description": "Signer's wallet address. Required to receive a swap transaction."
      },
      "to_chain": {
        "type": "string",
        "description": "Destination chain for cross-chain swaps. Omit for same-chain."
      },
      "to_user_address": {
        "type": "string",
        "description": "Recipient on the destination chain. Required for cross-family routes (e.g. EVM → Solana) when user_address is set."
      },
      "recipient_address": {
        "type": "string",
        "description": "Final token recipient for intent registration (may differ from user_address in cross-chain flows)"
      },
      "slippage_bps": {
        "type": "integer",
        "description": "Slippage tolerance in basis points. Default 50 (0.5%). Max 500 (5%)."
      }
    },
    "required": ["chain", "input_token", "output_token", "user_address"]
  },
  "output_schema": {
    "type": "object",
    "properties": {
      "intent_id": {
        "type": "string",
        "description": "Cross402 intent ID for tracking settlement"
      },
      "status": {
        "type": "string",
        "enum": ["PENDING", "DONE", "FAILED", "CANCELED"],
        "description": "Swap intent settlement status"
      }
    },
    "required": ["intent_id", "status"]
  }
}
```

---

## Flow Overview

```
1. (EVM only) Check ERC-20 approval
       ↓ sign & broadcast approval tx if needed
2. Get swap quote + swap transaction
       ↓ user signs and broadcasts the swap tx
3. Register swap intent  →  intent_id
       ↓
4. Poll GET /api/intents/{intent_id}  until  DONE / FAILED
```

---

## Step 1 — ERC-20 Approval (EVM only)

EVM swaps often require the router contract to spend the input token on the user's behalf. Skip this step for Solana.

```bash
GET /api/swap/approval?chain=base&token=0x4200...&token_out=0x8335...&amount=1000000000000000000&user_address=0xYourWallet
```

If the response has `needs_approval: true`, sign and broadcast the `approval` transaction before proceeding.

```typescript
// No SDK wrapper — call the API directly
const res = await fetch(
  `https://api-pay.agent.tech/api/swap/approval?chain=base&token=${inputToken}&token_out=${outputToken}&amount=${amount}&user_address=${walletAddress}`
);
const { needsApproval, approval } = await res.json();

if (needsApproval) {
  // sign approval.data, send to approval.to using your wallet library
  await wallet.sendTransaction({ to: approval.to, data: approval.data, value: approval.value });
}
```

```go
// No SDK wrapper — call the API directly
approvalURL := fmt.Sprintf(
  "%s/api/swap/approval?chain=base&token=%s&token_out=%s&amount=%d&user_address=%s",
  baseURL, inputToken, outputToken, amount, walletAddress,
)
// ... http.Get, parse JSON, sign and broadcast if needsApproval
```

---

## Step 2 — Get Quote and Swap Transaction

```typescript
import { PublicPayClient } from "@cross402/usdc";

const client = new PublicPayClient({ baseUrl: "https://api-pay.agent.tech" });

const result = await client.getSwapQuote({
  chain: "base",
  inputToken: "0x4200000000000000000000000000000000000006", // WETH
  outputToken: "0x833589fCD6eDb6E08f4c7C32D4f71b54bdA02913", // USDC
  fromAmount: 1_000_000_000_000_000_000, // 1 WETH in wei
  userAddress: "0xYourWallet",
});

console.log("expected USDC out:", result.quote.outputAmount);
console.log("min USDC out:", result.quote.minOutputAmount);
// result.swapTransaction contains { transaction, to, value, gasLimit, expiresAt }
```

```go
import pay "github.com/cross402/usdc-sdk-go"

client := pay.New(pay.WithBaseURL("https://api-pay.agent.tech"))

resp, err := client.GetSwapQuote(ctx, &pay.SwapQuoteParams{
    Chain:       "base",
    InputToken:  "0x4200000000000000000000000000000000000006",
    OutputToken: "0x833589fCD6eDb6E08f4c7C32D4f71b54bdA02913",
    FromAmount:  1_000_000_000_000_000_000,
    UserAddress: "0xYourWallet",
})
// resp.SwapTransaction.Transaction is hex calldata to sign
// resp.SwapTransaction.To is the contract address
// resp.SwapTransaction.ExpiresAt is Unix timestamp — don't broadcast after this
```

**Cross-chain example (Base → Solana):**

```typescript
const result = await client.getSwapQuote({
  chain: "base",
  inputToken: "0x833589fCD6eDb6E08f4c7C32D4f71b54bdA02913",
  outputToken: "EPjFWdd5AufqSSqeM2qN1xzybapC8G4wEGGkZwyTDt1v",
  fromAmount: 10_000_000, // 10 USDC
  toChain: "solana",
  userAddress: "0xYourEVMWallet",
  toUserAddress: "YourSolanaWalletPublicKey",
});
```

### Signing the Swap Transaction

The `swap_transaction` field contains a chain-specific payload. Sign and broadcast it using your wallet library.

| Chain | `transaction` encoding | How to sign |
| --- | --- | --- |
| Solana | base64 VersionedTransaction | Deserialize → sign with keypair → `sendRawTransaction` |
| EVM | hex calldata (`data`) | Send `{ to, data, value, gasLimit }` via `eth_sendRawTransaction` |

> Check `swap_transaction.expires_at` before broadcasting. If the quote has expired, fetch a new one.

---

## Step 3 — Register Swap Intent

After the swap transaction is confirmed on-chain, register it so Cross402 tracks settlement.

```typescript
const { intentId, status } = await client.registerSwapIntent({
  sourceTxHash: "0xabc...",     // broadcast tx hash
  fromChain: "base",
  toChain: "base",              // same as fromChain for same-chain swaps
  fromToken: "0x4200000000000000000000000000000000000006",
  toToken: "0x833589fCD6eDb6E08f4c7C32D4f71b54bdA02913",
  payerAddress: "0xYourWallet",
  recipientAddress: "0xYourWallet", // or a different recipient
  sendingTokenAmount: "1000000000000000000", // amount sent, decimal string
});

console.log(intentId, status); // status is "PENDING"
```

```go
resp, err := client.RegisterSwapIntent(ctx, &pay.RegisterSwapIntentRequest{
    SourceTxHash:       "0xabc...",
    FromChain:          "base",
    ToChain:            "base",
    FromToken:          "0x4200000000000000000000000000000000000006",
    ToToken:            "0x833589fCD6eDb6E08f4c7C32D4f71b54bdA02913",
    PayerAddress:       "0xYourWallet",
    RecipientAddress:   "0xYourWallet",
    SendingTokenAmount: "1000000000000000000",
})
fmt.Println(resp.IntentID, resp.Status) // "PENDING"
```

---

## Step 4 — Poll for Settlement

Use the standard intent polling loop with the `intent_id` returned above.

```typescript
import { SwapJobStatus } from "@cross402/usdc";

const POLL_INTERVAL_MS = 5_000;
const TERMINAL = new Set([SwapJobStatus.Done, SwapJobStatus.Failed, SwapJobStatus.Canceled]);

let swapStatus = status;
while (!TERMINAL.has(swapStatus)) {
  await new Promise(r => setTimeout(r, POLL_INTERVAL_MS));
  const intent = await client.getIntent(intentId);
  swapStatus = intent.status as any;
}

if (swapStatus === SwapJobStatus.Done) {
  console.log("Swap settled ✓");
} else {
  console.error("Swap did not complete:", swapStatus);
}
```

```go
for {
    time.Sleep(5 * time.Second)
    job, err := client.GetIntent(ctx, resp.IntentID)
    if err != nil {
        log.Println("poll error:", err)
        continue
    }
    if job.Status == "DONE" || job.Status == "FAILED" || job.Status == "CANCELED" {
        log.Println("final status:", job.Status)
        break
    }
}
```

### Swap Intent Status Values

| Status | Meaning |
| --- | --- |
| `PENDING` | Swap transaction received; monitoring source chain |
| `DONE` | Settlement complete (terminal) |
| `FAILED` | Settlement failed — source tx not confirmed or route error (terminal) |
| `CANCELED` | Job dropped (terminal) |

---

## Error Reference

| HTTP | Message | Action |
| --- | --- | --- |
| 400 | `from_amount and to_amount are mutually exclusive` | Set exactly one amount field |
| 400 | `from_amount or to_amount is required` | Set at least one amount field |
| 400 | `to_user_address is required for cross-family routes` | Add `toUserAddress` for EVM↔Solana routes |
| 400 | `no swap route found` | Check token addresses and chain support; try a higher amount |
| 400 | `insufficient liquidity for swap` | Reduce amount or increase slippage |
| 400 | `source_tx_hash is required` | Provide the broadcast tx hash in RegisterSwapIntent |
| 404 | `transfer not found` (GetSwapStatus) | LI.FI hasn't indexed the tx yet; retry after 30 s |
| 502 | `swap service temporarily unavailable` | Upstream DEX / LI.FI error; retry with backoff |

---

## Related Links

- [Swap API Reference](../../api/swap/)
- [Query Intent Status](../query-intent-status/)
- [Payment Polling](../payment-polling/)
- [Error Handling](../error-handling/)
- [Supported Chains](../../../introduction/supported-networks/)
