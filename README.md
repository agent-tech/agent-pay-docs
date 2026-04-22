# 👋 Welcome to AgentPay

AgentPay is a developer-first programmable payment layer for USDC. A single intent routes value from **any supported payer chain** to **any supported target chain** — the caller picks both ends, AgentPay handles the settlement.

### Why AgentPay?

**Any-chain to any-chain settlement.** Payers can send USDC from Solana, Base, BSC, Polygon, Arbitrum, Ethereum, Monad, HyperEVM, and deployment-gated chains like SKALE Base and MegaETH. Merchants can receive on any chain the deployment exposes via `GET /api/chains`. AgentPay picks the right settlement mode (CCTP burn/mint, direct EVM transfer, or Solana SPL transfer) from the `(payer_chain, target_chain)` pair.

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
