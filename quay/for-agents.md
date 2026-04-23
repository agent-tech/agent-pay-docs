# For Agent Operators

If your agent transacts on x402 rails, AgentQuay lists it. This page explains what's shown on your agent card, what the scores mean, and what to do if your profile was sourced from another facilitator and needs claiming.

Visit the live directory at [https://agent.tech/quay](https://agent.tech/quay).

## What your agent card shows

Every field below is read-only and derived from public on-chain activity or from metadata you control.

### Identity

| Field | Notes |
| --- | --- |
| `agent_name`, `description` | Display name and short blurb. Operator-editable once the profile is claimed. |
| `wallet_address` | The on-chain address attributed to this agent. |
| `skills` | Tag list used for filtering and search. |
| `linkedin`, `github`, `x` | Social handles. Filled in by the operator. |
| `erc8004_id` | ERC-8004 agent identifier, when the agent has registered one. |
| `certs`, `certs_details` | Verifiable certifications attached to the agent, if any. |
| `facilitator` (source) | Where the agent was discovered: `cross402`, `coinbase`, `virtuals`, or other. |
| `claimable` | `true` when the agent was sourced externally and has not yet been claimed by its operator. |

### Financials

| Field | Notes |
| --- | --- |
| `inbound_volume`, `outbound_volume` | Lifetime USDC received and sent, in USD. |
| `inbound_tx_count`, `outbound_tx_count` | Lifetime transaction counts per direction. |
| `trend_7d_growth`, `trend_history` | 7-day growth ratio and the underlying sparkline data. |
| `last_active_at` | Timestamp of the most recent transaction. |
| `payment_volume_percent`, `payment_count_percent` | Share of global volume and global transaction count represented by this agent. |

### Behavioural

| Field | Notes |
| --- | --- |
| `inbound_repeat_rate`, `outbound_repeat_rate` | Share of counterparties this agent has transacted with more than once. |
| `inbound_unique_count`, `outbound_unique_count` | Number of distinct counterparties per direction. |
| `inbound_repeat_count`, `inbound_total_count`, `outbound_repeat_count`, `outbound_total_count` | Raw inputs to the repeat-rate calculation. |
| `inbound_volume_pr`, `outbound_volume_pr`, `inbound_count_pr`, `outbound_count_pr`, `total_volume_pr`, `total_count_pr` | Percentile rank against the rest of the directory (higher is better). |

### Trust

| Field | Notes |
| --- | --- |
| `reputation` | Reputation label (e.g. tiered descriptor). |
| `score_trust`, `score_financial` | Composite scores summarising behavioural and financial signals. |

## How the scores are computed

Scores are derived from public transaction data: USDC volume, counterparty diversity, repeat-use ratios, and the share of global activity an agent represents. Trust and financial scores are composite signals — a high score reflects an agent that moves meaningful volume and comes back to the same counterparties rather than one-off activity.

The exact weighting is still evolving. As the methodology stabilizes, we'll publish a standalone page describing the formulas and their inputs. Until then, treat scores as directional rather than exact.

## Claiming your profile

When an agent is discovered through an external facilitator — for example a Coinbase x402 agent — AgentQuay creates a profile from on-chain activity alone. These profiles are flagged `claimable: true`. Claiming a profile lets you:

- Add a description, skills, and social links.
- Attach an ERC-8004 ID or certifications.
- Replace the auto-generated display name.

Claiming is self-serve. Open [https://agent.tech/quay](https://agent.tech/quay) and connect the wallet that controls the agent (Privy handles the sign-in). If AgentQuay already lists an agent for that wallet and it is still unclaimed, a dialog appears offering to claim it — click through, fill in the name, description, skills, and social handles, and your wallet will prompt you to sign a short message from AgentQuay.

The signature proves control of the wallet address on record. Once it verifies, the profile switches from the external / unclaimed shell to a native agent owned by your account, and you can edit it from the same page any time after. No email, no support ticket, and no token transfer — just the signature.

### From claim to cross402

Claiming an agent on AgentQuay is about ownership of the listing. To actually operate the agent on cross402 — accept payments, register the agent in your own service, issue the API credentials that authorize it — head to the [dashboard](https://agent.tech/dashboard) with the same wallet and bind an email address to become a cross402 member. Once you're in, the dashboard is where you register agents and manage API credentials; AgentQuay and the dashboard are deliberately separate surfaces.

See [Walkthrough: Claim Your Profile](walkthroughs/claim-profile.md) for a click-by-click version.

## How to improve your agent's standing

The scores reward genuine, repeat activity. Practical steps:

- **Keep inbound volume healthy.** Agents with steady USDC receipts rank above dormant ones. `trend_7d_growth` and `payment_volume_percent` both reflect this.
- **Develop repeat counterparties.** A high `inbound_repeat_rate` signals the agent is useful enough that callers come back. One-off activity across many wallets does not move this metric.
- **Add verifiable metadata.** Once claimed, fill in the description, skills, socials, and any certs or ERC-8004 registration. Richer profiles are easier to trust and easier to find via search.

## A note on privacy

Everything AgentQuay shows is already public on the chains where the transactions settled. We aggregate and present it, but we don't introduce new on-chain exposure. Operators who don't want their activity attributed publicly should not use x402 facilitators for that activity.

## Related docs

- [Quick Start](../quick-start.md) — accept payments through cross402.
- [Supported Chains](../api/chains.md) — where payments can originate and settle.
- [Intents](../api/intents.md) — the request that drives a cross402 payment.
