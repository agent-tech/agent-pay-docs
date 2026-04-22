# Multi-Chain Settlement

AgentPay decouples where a payer sends USDC from where the merchant receives it. This page explains that model from the caller's perspective: what the two chain fields mean, how AgentPay chooses between settlement paths, and what the caller observes as status transitions.

## Two chains per intent

Every `CreateIntent` request carries:

- `payer_chain` — required. The chain on which the payer holds USDC and signs an X402 authorization.
- `target_chain` — optional, defaults to `"base"`. The chain on which the merchant receives USDC.

The merchant address is validated against `target_chain`. If `target_chain` is an EVM chain, `recipient` must be a 20-byte hex address; if it is `solana`, `recipient` must be a Solana public key. For email recipients, AgentPay resolves the email to a wallet on the target chain via Privy — so the same email returns a Solana address when `target_chain` is `solana`, and an EVM address when `target_chain` is an EVM chain.

Any chain exposed by `GET /api/chains` may be used as either `payer_chain` or `target_chain`, including same-chain combinations (`base → base`, `polygon → polygon`, etc.).

## Settlement modes

AgentPay selects one of three settlement modes per intent. The caller never chooses — the mode follows from the `(payer_chain, target_chain)` pair and the chain registry.

### 1. CCTP burn/mint

Used when the source-to-target route is a Circle Cross-Chain Transfer Protocol pair. Typical examples:

- `solana → base`
- `solana → ethereum`
- EVM-to-EVM routes where CCTP is the cheapest path

AgentPay burns USDC on the source chain and mints it on the target chain through Circle's attestation infrastructure.

### 2. Direct EVM transfer

Used for same-chain routes and for EVM-to-EVM routes that don't need a bridge. An agent-controlled wallet on the target chain sends USDC directly to the merchant.

- `base → ethereum`
- `polygon → arbitrum`
- `base → base` (same-chain is a legitimate combination)

On BSC, Monad, and MegaETH — which require Permit2 + EIP-2612 signing at the payer side — the target-side transfer is still a plain USDC transfer.

### 3. Solana (SVM) direct

Used whenever `target_chain` is `solana`. An agent-controlled Solana wallet transfers USDC (SPL token) to the merchant's associated token account.

## What the caller observes

The status progression is the same regardless of settlement mode:

```
AWAITING_PAYMENT
      │  (payer submits X402 authorization)
      ▼
   PENDING
      │  (verification + source-chain settlement)
      ▼
SOURCE_SETTLED
      │  (AgentPay dispatches target-chain transfer)
      ▼
TARGET_SETTLING
      │  (target-chain tx confirms)
      ▼
 TARGET_SETTLED   ← terminal
```

On a `GetIntent` response for `TARGET_SETTLED`, the `target_payment` block holds the tx hash and explorer URL for the transfer on `target_chain`. The `source_payment` block holds the corresponding source-chain tx.

### Rollback

If AgentPay moves the intent to `TARGET_SETTLING` but the target transfer cannot be dispatched — for example because the agent's target-chain wallet hasn't been provisioned for that chain — the status is rolled back atomically to `SOURCE_SETTLED`. Callers that poll will see this as a transient dip; the pipeline retries automatically.

Two failures are terminal:

- **`VERIFICATION_FAILED`** — source-side authorization could not be verified. No funds moved on the payer side.
- **`PARTIAL_SETTLEMENT`** — source settled but target transfer could not be completed at all. Payer funds have moved; contact support for reconciliation.

See [Statuses](../../api/statuses.md) for the full transition table.

## Fees

Fees are computed per `(payer_chain, target_chain)` pair and returned in `fee_breakdown` on every intent response:

- `source_chain_fee` — gas/network cost on the payer side.
- `target_chain_fee` — gas/network cost on the target side (for example Ethereum gas is meaningfully higher than Base or Polygon).
- `platform_fee` and `platform_fee_percentage` — AgentPay's service fee.
- `total_fee` — sum.

See [Fee Breakdown](../../api/fees.md) for the exact shape.

## Related

- [Supported Chains](../../api/chains.md) — payer × target matrix and per-chain caveats.
- [Statuses](../../api/statuses.md) — full state machine.
- [Architecture](../../architecture.md) — end-to-end flow.
