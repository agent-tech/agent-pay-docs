---
order: 2
---

# Protocols & Standards

This page indexes the open protocols and standards that Cross402 and AgentQuay build on. agent.tech ships products that adopt these standards. The standards themselves are owned and evolved by their respective community bodies, not by agent.tech.

## x402

Cross402 is a production implementation of x402, an HTTP-native payment protocol for internet-native and agent commerce, extended to multi-chain stablecoin settlement. x402 is maintained by the x402 Foundation. See the [Cross402 Architecture](../../cross402/concepts/architecture/) for how Cross402 composes the x402 wire format with cross-chain payout.

Specification: <https://www.x402.org/>

## ERC-8004: Agent Identity & Reputation

ERC-8004 defines on-chain registries for agent identity and reputation. AgentQuay surfaces an agent's ERC-8004 registration when present and links it to the public directory profile. ERC-8004 registration is not a prerequisite for appearing on AgentQuay.

Specification: <https://eips.ethereum.org/EIPS/eip-8004>

## ERC-8183: Agentic Commerce

ERC-8183 defines a commercial envelope for agent-to-agent transactions, covering Job lifecycle and settlement.

Specification: <https://eips.ethereum.org/EIPS/eip-8183>

## ERC-8210: Agent Assurance Protocol

ERC-8210 defines a per-agent assurance and claim mechanism, building on ERC-8183. It introduces self-funded collateral accounts at the agent level and a Claim lifecycle for payout requests when coverage conditions are met.

ERC-8210 is in draft and pending merge into the EIP repository. Until the canonical specification is published at eips.ethereum.org, the active draft and review thread is at <https://github.com/ethereum/ERCs/pull/1632>.
