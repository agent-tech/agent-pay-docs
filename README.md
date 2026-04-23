# 👋 Welcome to AgentPay

AgentPay is a developer-first programmable payment layer for USDC. A single intent routes value from **any supported payer chain** to **any supported target chain** — the caller picks both ends, AgentPay handles the settlement.

### Why AgentPay?

**Any-chain to any-chain settlement.** Payers can send USDC from **Solana, Base, Ethereum, Polygon, and HyperEVM** today. Merchants can receive on any of those same chains — `GET /api/chains` is authoritative for what's callable right now. Every intent settles through the x402 protocol on both the payer side and the target side; the SDK picks the right signing flavor (EIP-3009, Permit2+EIP-2612, or Solana VersionedTransaction) for each chain automatically.

> **Roadmap:** Arbitrum, BSC, Monad, SKALE Base, and MegaETH are documented in [Supported Chains](api/chains.md) for integration preparation; they are 🚧 *coming soon* and will return `400` until enabled.

**Agent-led automation.** Managed agent wallets execute target-chain payouts so your backend doesn't hold hot keys per chain.

**Developer first.** Native SDKs for JS/TS and Go, covering everything from web checkouts to backend payout jobs.

[Get Started with the Quick Start Guide →](quick-start.md)

---

## v2 Breaking Notes

AgentPay v2 removed the assumption that payments settle on Base. Three things changed from v1:

- **New optional request field `target_chain`** on `CreateIntent`. Defaults to `"base"`, so v1 callers that only pass `payer_chain` continue to work.
- **Status enum renamed**: `BASE_SETTLING → TARGET_SETTLING`, `BASE_SETTLED → TARGET_SETTLED`. The old names are gone; SDK constants renamed accordingly (`IntentStatus.BaseSettled → TargetSettled`, `pay.StatusBaseSettled → pay.StatusTargetSettled`).
- **Response field renamed**: `base_payment` is now `target_payment` on the `GetIntent` response. SDK accessors renamed (`intent.basePayment → intent.targetPayment`, `intent.BasePayment → intent.TargetPayment`).

For the full before/after and a search-and-replace checklist, see [v2 Migration: Any-Chain → Any-Chain](docs/migration/v2-any-to-any.md).
