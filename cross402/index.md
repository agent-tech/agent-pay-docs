---
order: 20
label: Cross402
---

# What is Cross402

Cross402 is a programmable stablecoin payment layer for AI agents. A single API call routes value from any supported payer chain to any supported target chain. SDKs in JS/TS and Go ship alongside skill bundles for LLM-driven agents, so an agent can create a payment intent, sign, settle, and confirm without human intervention.

## Why it exists

Agents need to pay and get paid across chains, and the existing on-chain rails were not built with autonomous callers in mind: signing flows differ per chain, USDC contracts behave inconsistently, and no shared standard exists for "request payment, then settle." Cross402 is a production implementation of the [x402](https://www.x402.org/) HTTP payment protocol, extended to multi-chain stablecoin environments. The caller declares intent (payer chain, target chain, amount, recipient); Cross402 picks the right signing flavor, runs both legs through the x402 facilitator, and exposes a single status stream.

Cross402 is a sibling to [AgentQuay](../agentquay/) under the agent.tech umbrella. Cross402 is the rail that moves the stablecoin; AgentQuay is the public record of who used it and how.

## At a glance

* **One API, many chains.** Pay from Solana, Base, Ethereum, Polygon, HyperEVM, Arbitrum, BSC, Monad, SKALE Base, or MegaETH; settle to any of the live target chains. See [Supported Chains](../introduction/supported-networks/).
* **Right signing flavor, automatically.** EIP-3009, Permit2 + EIP-2612, or Solana VersionedTransaction — picked from `payment_requirements`, never from your code.
* **SDKs + skill bundles.** JS/TS and Go SDKs for direct integration; prompt-ready skill bundles for LLM-driven agents (create-intent, submit-proof, execute-intent, polling, error handling).
* **Transparent fees and status.** Per-pair fee breakdown on every intent; a deterministic status machine from `AWAITING_PAYMENT` through `TARGET_SETTLED`.

## Quick install

```
npm install @cross402/usdc
```

## Start here

* **Building an integration?** Run through the [Quick Start](quick-start/) — a working payment in under five minutes.
* **Want the mechanics?** Read [Architecture](concepts/architecture/) and [Multi-Chain Settlement](concepts/multi-chain-settlement/).
* **Wiring it into an LLM agent?** Browse the [skill bundles](skills/agent-public-payment/).
* **Looking up an endpoint?** Open the [API Reference](api/auth/).
