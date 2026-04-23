# 🏗 Architecture

AgentPay is a programmable USDC payment layer that routes value from one chain to another. A single intent carries two chains — a **payer chain** where funds originate and a **target chain** where the merchant is paid — and AgentPay bridges the two transparently.

## Payment Flow

```
 payer_chain                                      target_chain
──────────────                                   ──────────────
    payer                                            merchant
      │                                                 ▲
      │  1. payer-side x402 payload                     │
      │     (EIP-3009 / Permit2+2612 /                  │
      │      Solana partial sign)                       │
      ▼                                                 │
  ┌──────────────────┐    2. facilitator executes       │
  │    AgentPay      │────── on-chain, funds land ──────┤
  │ (x402 facilitator│    in AgentPay proxy wallet      │
  │   + orchestrator)│                                  │
  └──────────────────┘                                  │
           │                                            │
           │  3. target-side x402 payload               │
           │     signed by proxy wallet                 │
           │     (flavor per target chain)              │
           ▼                                            │
     facilitator executes ───────────────────── payout ─┘
```

### Step by step

1. **Create intent** — The caller supplies `payer_chain`, `target_chain`, and amount. AgentPay builds the x402 `payment_requirements` and returns an `intent_id`.
2. **Payer pays on source chain** — The payer signs an x402 payload (EIP-3009 on most EVM chains, Permit2 + EIP-2612 on BSC / Monad / MegaETH, a partially signed VersionedTransaction v0 on Solana). AgentPay forwards the signed payload to the x402 facilitator, which submits the transaction on the payer chain. Status moves through `PENDING → SOURCE_SETTLED`.
3. **Pay out on target chain** — AgentPay's proxy wallet signs a second x402 payload that moves USDC to the merchant address on `target_chain`. The signing flavor is chosen from the target chain's USDC contract capabilities (EIP-3009, Permit2+2612, or Solana VersionedTransaction). The same x402 facilitator submits it on-chain.
4. **Finalize** — Status moves `SOURCE_SETTLED → TARGET_SETTLING → TARGET_SETTLED`. The `target_payment` block on `GetIntent` carries the final tx hash and explorer URL.

Both legs use the same x402 protocol and the same facilitator. The caller does not pick a signing flavor — the SDK produces the right payload from `payment_requirements`.

## Failure and Rollback

If the target-chain transfer cannot be dispatched (for example the agent wallet for the target chain is not provisioned, or the chain's payment service is temporarily unavailable), AgentPay atomically rolls the intent back from `TARGET_SETTLING` to `SOURCE_SETTLED`. The settlement pipeline will retry. This rollback is transparent to callers that poll `GetIntent`; the status may appear to oscillate `SOURCE_SETTLED ↔ TARGET_SETTLING` until the target-side issue clears.

Two failure modes are terminal and not retried:

- **`VERIFICATION_FAILED`** — the source-chain authorization could not be verified or settled. No funds moved on the payer side.
- **`PARTIAL_SETTLEMENT`** — source settled but the target transfer never completed. Funds moved on the payer side; contact support for reconciliation.

See [Multi-Chain Settlement](docs/concepts/multi-chain-settlement.md) for the cross-chain mechanics and [Statuses](api/statuses.md) for the full transition table.

## Trust Model

* **Non-Custodial payer side** — The payer signs their own X402 authorization. AgentPay only acts on authorizations the payer has already signed.
* **Agent-managed target side** — Once source-chain funds are confirmed, an agent-managed wallet on the target chain moves USDC to the merchant. The specific wallet depends on integration mode: the v2 `/v2/intents/:id/execute` flow uses a per-agent sub-wallet; the public `/api/intents/:id` flow uses the global proxy wallet.
