# 🏗 Architecture

AgentPay is a programmable USDC payment layer that routes value from one chain to another. A single intent carries two chains — a **payer chain** where funds originate and a **target chain** where the merchant is paid — and AgentPay bridges the two transparently.

## Payment Flow

```
 payer_chain                             target_chain
──────────────                          ──────────────
    payer                                   merchant
      │                                         ▲
      │  1. X402 authorization                  │
      │     (EIP-3009 / Permit2 /               │
      │      Solana partial sign)               │
      ▼                                         │
  ┌─────────────────┐     2. verify + settle    │
  │ AgentPay source │───────────────────────────┤
  │    receiver     │                           │
  └─────────────────┘                           │
                                    3. route    │
                                  TargetPayment │
                                    Router      │
                                        │       │
                            ┌───────────┼───────┴──────────┐
                            ▼           ▼                  ▼
                      ┌──────────┐ ┌──────────┐     ┌──────────────┐
                      │  CCTP    │ │ Direct   │     │ Solana (SVM) │
                      │ burn/mint│ │ EVM xfer │     │ direct       │
                      └──────────┘ └──────────┘     └──────────────┘
```

### Step by step

1. **Create intent** — The caller supplies `payer_chain`, `target_chain`, and amount. AgentPay builds the X402 payment requirements and returns an `intent_id`.
2. **Payer pays on source chain** — The payer signs an X402 authorization (EIP-3009 on most EVM chains, Permit2 + EIP-2612 on BSC / Monad / MegaETH, a partially signed VersionedTransaction v0 on Solana) and either the payer or an agent wallet submits it. Status moves through `PENDING → SOURCE_SETTLED`.
3. **Route to target chain** — AgentPay selects a settlement mode based on the `(payer_chain, target_chain)` pair:
    - **CCTP burn/mint** when the source-to-target route is a Circle CCTP pair (e.g. `solana → base`, `solana → ethereum`).
    - **Direct EVM transfer** when both payer and target are EVM chains that don't need a bridge (e.g. `base → ethereum`, `polygon → base`, or same-chain routes like `base → base`). An agent-controlled wallet on the target chain sends USDC to the merchant.
    - **Solana (SVM) direct** when the target is `solana`. An agent-controlled Solana wallet transfers USDC to the merchant's SPL token account.
4. **Finalize** — Status moves `SOURCE_SETTLED → TARGET_SETTLING → TARGET_SETTLED`. The `target_payment` block on `GetIntent` carries the final tx hash and explorer URL.

The caller never chooses the settlement mode. That selection happens server-side from the `(payer_chain, target_chain)` pair and the chain registry.

## Failure and Rollback

If the target-chain transfer cannot be dispatched (for example the agent wallet for the target chain is not provisioned, or the chain's payment service is temporarily unavailable), AgentPay atomically rolls the intent back from `TARGET_SETTLING` to `SOURCE_SETTLED`. The settlement pipeline will retry. This rollback is transparent to callers that poll `GetIntent`; the status may appear to oscillate `SOURCE_SETTLED ↔ TARGET_SETTLING` until the target-side issue clears.

Two failure modes are terminal and not retried:

- **`VERIFICATION_FAILED`** — the source-chain authorization could not be verified or settled. No funds moved on the payer side.
- **`PARTIAL_SETTLEMENT`** — source settled but the target transfer never completed. Funds moved on the payer side; contact support for reconciliation.

See [Multi-Chain Settlement](docs/concepts/multi-chain-settlement.md) for the cross-chain mechanics and [Statuses](api/statuses.md) for the full transition table.

## Trust Model

* **Non-Custodial payer side** — The payer signs their own X402 authorization. AgentPay only acts on authorizations the payer has already signed.
* **Agent-managed target side** — Once source-chain funds are confirmed, an agent-managed wallet on the target chain moves USDC to the merchant. The specific wallet depends on integration mode: the v2 `/v2/intents/:id/execute` flow uses a per-agent sub-wallet; the public `/api/intents/:id` flow uses the global proxy wallet.
