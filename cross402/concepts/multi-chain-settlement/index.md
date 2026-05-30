---
order: 2
---

# Multi-Chain Settlement

Cross402 decouples where a payer sends the stablecoin from where the merchant receives it. This page explains that model from the caller's perspective: what the two chain fields mean, how the x402 protocol handles both legs, and what the caller observes as status transitions.

## Two chains per intent (and optionally two assets)

Every `CreateIntent` request carries:

- `payer_chain` — required. The chain on which the payer holds stablecoins and signs an X402 authorization.
- `target_chain` — optional, defaults to `"base"`. The chain on which the merchant receives stablecoins.
- `payer_asset` — optional, defaults to `"usdc"`. One of `usdc` | `usdt` | `usdt0`. Selects the stablecoin the payer signs against on `payer_chain`.
- `target_asset` — optional, defaults to `"usdc"`. Same set as `payer_asset`. Selects the stablecoin the merchant receives on `target_chain`.

Asset and chain are independent: a payer can sign in USDT0 on Arbitrum while the merchant still receives USDC on Base. Clients that omit the asset fields keep the historical USDC-everywhere behaviour with no wire-format change. See the [Token × Chain matrix](../../../introduction/supported-networks/#token--chain-matrix) for the supported (chain, asset) combinations.

The merchant address is validated against `target_chain`. If `target_chain` is an EVM chain, `recipient` must be a 20-byte hex address; if it is `solana`, `recipient` must be a Solana public key. For email recipients, Cross402 resolves the email to a wallet on the target chain via Privy — so the same email returns a Solana address when `target_chain` is `solana`, and an EVM address when `target_chain` is an EVM chain.

Any chain exposed by `GET /api/chains` may be used as either `payer_chain` or `target_chain`, including same-chain combinations (`base → base`, `polygon → polygon`, etc.). See [Supported Chains](../../../introduction/supported-networks/) for the full set and per-chain caveats.

## How settlement works

Both sides of an intent — payer-side collection and target-side merchant payout — run through the **x402 protocol**. Each side produces a signed x402 payload that Cross402 forwards to the shared x402 facilitator, which executes the payment on-chain.

* **Payer side** — the payer's client signs an x402 payload authorizing the stablecoin to move from the payer's wallet to Cross402.
* **Target side** — once the source leg settles, Cross402's proxy wallet signs its own x402 payload authorizing the stablecoin to move from that proxy wallet to the merchant address on `target_chain`.

The facilitator is the same component on both sides; the only thing that changes per chain is the **signing flavor**, which depends on what the chain's USDC contract supports:

### Signing flavors

The signing flavor is per-(chain, asset), not per-chain. The same chain can use different flavors for different assets (e.g. on MegaETH, native USDm uses Permit2 while USDT0 uses EIP-3009).

- **EIP-3009 `TransferWithAuthorization`** — Circle USDC on Base / Ethereum / Polygon / Arbitrum; USDT0 on Arbitrum / Monad / HyperEVM / MegaETH / Polygon; Polygon native USDT (alias of USDT0). A single signed authorization is sufficient; the facilitator relays it on-chain. Polygon USDT0/USDT use a **salted** variant of the EIP-712 domain — `payment_requirements.extra.domainType == "salted"` signals the change.
- **Permit2 + optional EIP-2612** — chains with non-standard USDC (BSC, Monad, MegaETH). The payload combines a Permit2 `PermitWitnessTransferFrom` signature with, when needed, a gasless EIP-2612 `Permit` that grants Permit2 the USDC allowance. cross402 sponsors the on-chain submission.
- **Approval sponsorship** — legacy native USDT on Ethereum / BSC / Base (no EIP-3009, no permit). The facilitator pays gas to submit `approve` + `transferFrom` on the payer's behalf; first payment from a (chain, wallet) pair carries an `approve` (5–60s extra latency), subsequent payments are signature-only. Currently gated behind a deployment flag.
- **Solana VersionedTransaction (v0)** — Solana. The payer or proxy wallet partially signs a VersionedTransaction carrying an SPL `TransferChecked` instruction; the facilitator co-signs as fee payer before submission.

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

If Cross402 moves the intent to `TARGET_SETTLING` but the target transfer cannot be dispatched — for example because the agent's target-chain wallet hasn't been provisioned for that chain — the status is rolled back atomically to `SOURCE_SETTLED`. Callers that poll will see this as a transient dip; the pipeline retries automatically.

Two failures are terminal:

* **`VERIFICATION_FAILED`** — source-side authorization could not be verified. No funds moved on the payer side.
* **`PARTIAL_SETTLEMENT`** — source settled but target transfer could not be completed at all. Payer funds have moved; contact support for reconciliation.

See [Statuses](../statuses/) for the full transition table.

## Fees

Fees are computed per `(payer_chain, target_chain)` pair and returned in `fee_breakdown` on every intent response:

* `source_chain_fee` — gas/network cost on the payer side.
* `target_chain_fee` — gas/network cost on the target side (for example Ethereum gas is meaningfully higher than Base or Polygon).
* `platform_fee` and `platform_fee_percentage` — Cross402's service fee.
* `total_fee` — sum.

See [Fee Breakdown](../../api/fees/) for the exact shape.

## Related

* [Supported Chains](../../../introduction/supported-networks/) — payer × target matrix and per-chain caveats.
* [Statuses](../statuses/) — full state machine.
* [Architecture](../architecture/) — end-to-end flow.
