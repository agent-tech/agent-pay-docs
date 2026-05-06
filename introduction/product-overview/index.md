---
order: 3
---

# Product Overview

agent.tech ships two products. Cross402 is the rail that moves stablecoins across chains in response to agent or user intent. AgentQuay is the public directory of agents transacting on x402 rails. A single agent can appear in both, and the two are usable in isolation: you can integrate Cross402 without ever touching AgentQuay, and you can browse AgentQuay without integrating Cross402.

## Cross402

Cross402 is a developer-first stack for moving stablecoins across chains in response to agent or user intent. A single API call routes value from a payer chain to a target chain; the SDK handles signing, settlement, and status tracking. Cross402 is a production implementation of the x402 HTTP payment protocol, extended to multi-chain stablecoin environments.

Three surfaces are exposed to integrators:

- **SDKs** in JS/TS and Go, embedded directly in agent code.
- **Dashboard** at <https://agent.tech/dashboard> for human-side configuration: agent registration, credential issuance, and payment history.
- **Skill bundles**, prompt-ready payment workflows that an LLM-driven agent ingests as a single unit and executes end-to-end.

See [Cross402 Quick Start](../../cross402/quick-start/) to integrate, or [Architecture](../../cross402/concepts/architecture/) for the end-to-end payment flow.

## AgentQuay

AgentQuay indexes agents transacting across x402 rails, including Cross402, Coinbase, Virtuals, and other facilitators, and surfaces their on-chain activity, behavioural patterns, and reputation signals. Anyone can browse without signing in. Agent operators can claim externally-sourced profiles to add metadata.

AgentQuay is cross-protocol by design. Agents from any compatible facilitator appear in the directory. ERC-8004 registration is surfaced when present but is not a listing requirement.

See [What is AgentQuay](../../agentquay/) for the visitor's tour, or [For Agent Operators](../../agentquay/for-agent-operators/) for what is shown about your agent.

## Relationship between the two

A single agent can show up in both products. Cross402 is the rail it pays through; AgentQuay is the public record of what it has done. Neither depends on the other. Together they form a minimal loop: an agent settles a payment through Cross402 and the same activity, indexed via the public x402 settlement layer, becomes part of its AgentQuay profile.
