---
order: 2
---

# USDT Signing

Cross402 supports USDT (and USDT0) as a payer asset on selected chains. The signing flow for USDT differs from USDC because **USDT does not implement EIP-3009 or EIP-2612**, which means the usual gasless authorization paths are unavailable.

> Pass `payerAsset: "usdt"` (or `"usdt0"`) in `CreateIntent` to use USDT. The `payment_requirements` object in the response tells the SDK exactly which signing path to use.

---

## Which chains require gas for USDT?

| Chain | Token | Signing path | Payer needs gas? | Gas token |
| --- | --- | --- | --- | --- |
| BSC (`bsc`) | USDT | Permit2 `approval_sponsorship` | **Yes** | BNB |
| Ethereum (`ethereum`) | USDT | Permit2 `approval_sponsorship` | **Yes** | ETH |
| Polygon (`polygon`) | USDT | Permit2 `approval_sponsorship` | **Yes** | MATIC |
| Base (`base`) | USDC | EIP-3009 | No | — |
| Arbitrum (`arbitrum`) | USDC | EIP-3009 | No | — |
| BSC (`bsc`) | USDC | Permit2 + EIP-2612 (sponsored) | No | — |
| Monad (`monad`) | USDC | Permit2 + EIP-2612 (sponsored) | No | — |
| Solana (`solana`) | USDC | Solana VT v0 (fee payer sponsored) | No | — |

USDT on BSC, Ethereum mainnet, and Polygon always require an on-chain `ERC20.approve(Permit2, amount)` transaction before the payment signature. This transaction costs gas in the chain's native token.

---

## Why USDT needs gas

USDC and some other tokens implement one or both of these standards that enable gasless Permit2 authorization:

* **EIP-3009** `TransferWithAuthorization` — the contract transfers tokens directly on a signed message, no prior approval needed.
* **EIP-2612** `permit()` — an off-chain signature that grants Permit2 an allowance in a single call. Cross402 can use this to sponsor the approval gas on behalf of the payer.

**USDT implements neither.** The only way to authorize Permit2 to move USDT is a standard on-chain `ERC20.approve(Permit2, amount)` call. This requires native gas from the payer's wallet.

---

## The `approval_sponsorship` signal

When the backend sets `payment_requirements.extra.assetTransferMethod = "approval_sponsorship"`, the payer must:

1. **Send an on-chain `approve` tx** — `ERC20.approve(Permit2, amount)` on the token contract. Requires gas.
2. **Sign Permit2 off-chain** — `PermitWitnessTransferFrom` typed-data signature. No gas.
3. **Submit `settle_proof`** — POST to `/api/intents/{intent_id}`.

The SDK handles steps 1–3 automatically. No EIP-2612 extension is included in the proof (USDT doesn't support it).

---

## Ethereum USDT: double-approve quirk

Ethereum mainnet USDT (`0xdAC17F958D2ee523a2206206994597C13D831ec7`) has a non-standard `approve()` that **reverts if you try to change a non-zero allowance directly to another non-zero value**. You must reset it to `0` first.

The SDK handles this automatically:

```
// Pseudocode — SDK does this for you
if currentAllowance > 0 {
    token.approve(Permit2, 0)   // reset
}
token.approve(Permit2, amount) // set
```

If you implement signing yourself, add this reset step for `eip155:1` when the existing Permit2 allowance is non-zero.

---

## Implementation example (JS/TS)

```typescript
import { PublicPayClient, Asset } from '@cross402/usdc';
import { createWalletClient, createPublicClient, http, erc20Abi } from 'viem';
import { mainnet } from 'viem/chains';

const PERMIT2 = '0x000000000022D473030F116dDEE9F6B43aC78BA3';

const intent = await client.createIntent({
  recipient: '0xRecipientAddress',
  amount: '10.00',
  payerChain: 'ethereum',
  payerAsset: Asset.USDT,
});

const { extra } = intent.paymentRequirements;

// Step 1: on-chain approve (only when assetTransferMethod === "approval_sponsorship")
if (extra.assetTransferMethod === 'approval_sponsorship') {
  const current = await publicClient.readContract({
    address: extra.asset,
    abi: erc20Abi,
    functionName: 'allowance',
    args: [walletAddress, PERMIT2],
  });

  // ETH USDT: reset to 0 first if non-zero
  if (current > 0n) {
    const resetHash = await walletClient.writeContract({
      address: extra.asset, abi: erc20Abi,
      functionName: 'approve', args: [PERMIT2, 0n],
    });
    await publicClient.waitForTransactionReceipt({ hash: resetHash });
  }

  const approveHash = await walletClient.writeContract({
    address: extra.asset, abi: erc20Abi,
    functionName: 'approve', args: [PERMIT2, BigInt(extra.amount) * 10n],
  });
  await publicClient.waitForTransactionReceipt({ hash: approveHash });
}

// Step 2: sign Permit2 PermitWitnessTransferFrom (off-chain, no gas)
// Step 3: submit settle_proof
await client.submitProof(intent.intentId, settleProof);
```

---

## Implementation example (Go)

```go
// The sdk_test reference implementation handles approval_sponsorship automatically.
// Set payerAsset in CreateIntentRequest:
resp, err := client.CreateIntent(ctx, &pay.CreateIntentRequest{
    Recipient:   "0xRecipientAddress",
    Amount:      "10.00",
    PayerChain:  "ethereum",
    PayerAsset:  pay.AssetUSDT,
})

// The settle_proof builder reads extra.assetTransferMethod from payment_requirements.
// When "approval_sponsorship" is set, it:
//   1. Calls ensurePermit2Allowance (on-chain approve, gas deducted from payer)
//   2. Signs PermitWitnessTransferFrom (off-chain)
//   3. Returns a Permit2-only proof (no EIP-2612 extension)
```

---

## Key differences vs USDC

| | USDC (Base / Arb / ETH) | USDC (BSC / Monad) | USDT (BSC / ETH / Polygon) |
| --- | --- | --- | --- |
| Standard | EIP-3009 | Permit2 + EIP-2612 | Permit2 only |
| Payer gas required | No | No (Cross402 sponsors) | **Yes** |
| `assetTransferMethod` | absent / `"eip3009"` | `"permit2"` | `"approval_sponsorship"` |
| EIP-2612 extension in proof | No | Yes | No |

---

## References

* [BSC Signing](../bsc-signing/) — Permit2 + EIP-2612 flow for USDC on BSC
* [Supported Chains](../../../introduction/supported-networks/) — full payer chain matrix
* [Permit2 contract](https://github.com/Uniswap/permit2) — `0x000000000022D473030F116dDEE9F6B43aC78BA3`
