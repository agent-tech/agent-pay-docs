---
order: 1
---

# BSC Signing

> Source: https://docs.agent.tech/api/bsc-signing/

# BSC Signing

> BSC is supported. See [Supported Chains](../../../introduction/supported-networks/) for the current set of live chains and per-chain caveats.

BSC (BNB Smart Chain) uses a different signing mechanism from other EVM chains. Instead of EIP-3009 `transferWithAuthorization` (used on Base, Ethereum, etc.), BSC USDC uses **Permit2** — a two-step approval and permit-transfer flow.

This page explains why BSC is different and how to implement it correctly.

> BSC is also a valid `target_chain`. Merchants can receive USDC on BSC by passing `targetChain: "bsc"` on `CreateIntent`. The 18-decimal / Permit2 caveats on this page apply to the payer side. On the target side, Cross402 performs a standard USDC transfer to the merchant from an agent-controlled BSC wallet. See the payer × target matrix in [Supported Chains](../../../introduction/supported-networks/).

---

## Why BSC Is Different

Most chains supported by AgentTech use **EIP-3009** (`transferWithAuthorization`), which lets the payer sign a single off-chain message authorizing a direct transfer. BSC USDC does not support EIP-3009, so the payment flow uses **Permit2** instead.

Permit2 is a canonical contract (`0x000000000022D473030F116dDEE9F6B43aC78BA3`) that introduces a universal approve-once pattern: the payer approves Permit2 once on the USDC contract, then signs off-chain `PermitTransferFrom` messages for each subsequent payment — no additional on-chain approvals needed per transaction.

---

## Two-Step Flow

### Step 1: One-Time Permit2 Approval (on-chain)

Before the first BSC payment, the payer must grant Permit2 an unlimited (or sufficient) allowance on the USDC contract. This is a standard ERC-20 `approve` call and only needs to happen once per wallet.

```
import { createWalletClient, createPublicClient, http, erc20Abi } from "viem";
import { bsc } from "viem/chains";
import { privateKeyToAccount } from "viem/accounts";

const USDC_ADDRESS  = "0x8AC76a51cc950d9822D68b83fE1Ad97B32Cd580d"; // BSC USDC
const PERMIT2_ADDRESS = "0x000000000022D473030F116dDEE9F6B43aC78BA3";
const MIN_ALLOWANCE = BigInt("10000000000000000000"); // 10 USDC (18 decimals)

const account = privateKeyToAccount("0x...");
const publicClient  = createPublicClient({ chain: bsc, transport: http() });
const walletClient  = createWalletClient({ chain: bsc, account, transport: http() });

// Check current allowance
const allowance = await publicClient.readContract({
  address: USDC_ADDRESS,
  abi: erc20Abi,
  functionName: "allowance",
  args: [account.address, PERMIT2_ADDRESS],
});

// Approve Permit2 if needed
if (allowance < MIN_ALLOWANCE) {
  const hash = await walletClient.writeContract({
    address: USDC_ADDRESS,
    abi: erc20Abi,
    functionName: "approve",
    args: [PERMIT2_ADDRESS, BigInt("0xffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffff")],
  });
  await publicClient.waitForTransactionReceipt({ hash });
}
```

> **BSC USDC uses 18 decimals**, not 6. Always read `extra.decimals` from the intent response rather than hardcoding `6`.

---

### Step 2: Sign the Permit2 Message (off-chain)

Each payment requires signing a `PermitWitnessTransferFrom` typed message. This is entirely off-chain — no gas cost.

The signed payload is then submitted to AgentTech as the `settle_proof`.

```
import { parseUnits } from "viem";

// Values come from the CreateIntent response
const { intentId, extra } = await client.createIntent({
  email: "merchant@example.com",
  amount: "10.00",
  payerChain: "bsc",
});

const decimals  = extra.decimals;      // 18 for BSC
const spender   = extra.spender;       // AgentTech contract address
const nonce     = extra.nonce;         // Unique nonce from backend
const deadline  = extra.deadline;      // Unix timestamp

const amountRaw = parseUnits("10.00", decimals);

const signature = await walletClient.signTypedData({
  domain: {
    name: "Permit2",
    chainId: 56, // BSC mainnet
    verifyingContract: PERMIT2_ADDRESS,
  },
  types: {
    PermitTransferFrom: [
      { name: "permitted", type: "TokenPermissions" },
      { name: "spender",   type: "address" },
      { name: "nonce",     type: "uint256" },
      { name: "deadline",  type: "uint256" },
    ],
    TokenPermissions: [
      { name: "token",  type: "address" },
      { name: "amount", type: "uint256" },
    ],
  },
  primaryType: "PermitTransferFrom",
  message: {
    permitted: { token: USDC_ADDRESS, amount: amountRaw },
    spender,
    nonce: BigInt(nonce),
    deadline: BigInt(deadline),
  },
});

await client.submitProof(intentId, { signature, assetTransferMethod: "permit2" });
```

---

## Go Implementation

```
import (
    "math/big"

    "github.com/cross402/usdc-sdk-go"
)

// One-time Permit2 approval (call once per wallet)
if err := client.ApprovePermit2(ctx, walletPrivKey); err != nil {
    return fmt.Errorf("permit2 approval: %w", err)
}

// Create intent
resp, err := client.CreateIntent(ctx, &pay.CreateIntentRequest{
    Email:      "merchant@example.com",
    Amount:     "10.00",
    PayerChain: pay.ChainBSC,
})

// Sign and submit (SDK handles Permit2 typed-data signing)
if err := client.ExecuteIntent(ctx, resp.IntentID); err != nil {
    return fmt.Errorf("execute: %w", err)
}
```

---

## Key Differences vs Other EVM Chains

|  | Base / Arbitrum / Polygon | BSC |
| --- | --- | --- |
| Signing standard | EIP-3009 `transferWithAuthorization` | Permit2 `PermitTransferFrom` |
| On-chain pre-approval | Not required | Required once per wallet |
| USDC decimals | 6 | **18** |
| `assetTransferMethod` | `"eip3009"` | `"permit2"` |

---

## References

* [Permit2 contract source](https://github.com/Uniswap/permit2) — Uniswap's canonical deploy
* [BSC USDC token contract](https://bscscan.com/token/0x8AC76a51cc950d9822D68b83fE1Ad97B32Cd580d)
