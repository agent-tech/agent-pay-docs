# Multi-Chain Settlement

cross402 decouples where a payer sends USDC from where the merchant receives it. This page explains that model from the caller's perspective: what the two chain fields mean, how the x402 protocol handles both legs, and what the caller observes as status transitions.

## Two chains per intent

Every `CreateIntent` request carries:

- `payer_chain` — required. The chain on which the payer holds USDC and signs an X402 authorization.
- `target_chain` — optional, defaults to `"base"`. The chain on which the merchant receives USDC.

The merchant address is validated against `target_chain`. If `target_chain` is an EVM chain, `recipient` must be a 20-byte hex address; if it is `solana`, `recipient` must be a Solana public key. For email recipients, cross402 resolves the email to a wallet on the target chain via Privy — so the same email returns a Solana address when `target_chain` is `solana`, and an EVM address when `target_chain` is an EVM chain.

Any chain exposed by `GET /api/chains` may be used as either `payer_chain` or `target_chain`, including same-chain combinations (`base → base`, `polygon → polygon`, etc.). See [Supported Chains](../../api/chains.md) for the full set and per-chain caveats.

## How settlement works

Both sides of an intent — payer-side collection and target-side merchant payout — run through the **x402 protocol**. Each side produces a signed x402 payload that cross402 forwards to the shared x402 facilitator, which executes the payment on-chain.

- **Payer side** — the payer's client signs an x402 payload authorizing USDC to move from the payer's wallet to cross402.
- **Target side** — once the source leg settles, cross402's proxy wallet signs its own x402 payload authorizing USDC to move from that proxy wallet to the merchant address on `target_chain`.

The facilitator is the same component on both sides; the only thing that changes per chain is the **signing flavor**, which depends on what the chain's USDC contract supports:

### Signing flavors

- **EIP-3009 `TransferWithAuthorization`** — used on Circle-native USDC chains (Base, Ethereum, Polygon, Arbitrum). A single signed authorization is sufficient; the facilitator relays it on-chain.
- **Permit2 + optional EIP-2612** — used on chains with non-standard USDC (BSC, Monad, MegaETH). The payload combines a Permit2 `PermitWitnessTransferFrom` signature with, when needed, a gasless EIP-2612 `Permit` that grants Permit2 the USDC allowance. cross402 sponsors the on-chain submission.
- **Solana VersionedTransaction (v0)** — used when the chain is Solana. The payer or proxy wallet partially signs a VersionedTransaction carrying an SPL `TransferChecked` instruction; the facilitator co-signs as fee payer before submission.

Callers do not select the flavor. The SDK (or a compliant x402 client) produces the correct payload from the `payment_requirements` block returned on `CreateIntent`. From the caller's viewpoint the contract is unchanged: `POST createIntent`, submit the signed payload, poll status.

## What the caller observes

The status progression is the same regardless of which chains are paired:

```
AWAITING_PAYMENT
      │  (payer submits X402 authorization)
      ▼
   PENDING
      │  (verification + source-chain settlement)
      ▼
SOURCE_SETTLED
      │  (cross402 dispatches target-chain transfer)
      ▼
TARGET_SETTLING
      │  (target-chain tx confirms)
      ▼
 TARGET_SETTLED   ← terminal
```

On a `GetIntent` response for `TARGET_SETTLED`, the `target_payment` block holds the tx hash and explorer URL for the transfer on `target_chain`. The `source_payment` block holds the corresponding source-chain tx.

### Rollback

If cross402 moves the intent to `TARGET_SETTLING` but the target transfer cannot be dispatched — for example because the agent's target-chain wallet hasn't been provisioned for that chain — the status is rolled back atomically to `SOURCE_SETTLED`. Callers that poll will see this as a transient dip; the pipeline retries automatically.

Two failures are terminal:

- **`VERIFICATION_FAILED`** — source-side authorization could not be verified. No funds moved on the payer side.
- **`PARTIAL_SETTLEMENT`** — source settled but target transfer could not be completed at all. Payer funds have moved; contact support for reconciliation.

See [Statuses](../../api/statuses.md) for the full transition table.

## Fees

Fees are computed per `(payer_chain, target_chain)` pair and returned in `fee_breakdown` on every intent response:

- `source_chain_fee` — gas/network cost on the payer side.
- `target_chain_fee` — gas/network cost on the target side (for example Ethereum gas is meaningfully higher than Base or Polygon).
- `platform_fee` and `platform_fee_percentage` — cross402's service fee.
- `total_fee` — sum.

See [Fee Breakdown](../../api/fees.md) for the exact shape.

## Related

- [Supported Chains](../../api/chains.md) — payer × target matrix and per-chain caveats.
- [Statuses](../../api/statuses.md) — full state machine.
- [Architecture](../../architecture.md) — end-to-end flow.
