# v2 Migration: Any-Chain → Any-Chain

AgentPay v2 makes the target chain a first-class intent field. Payments are no longer assumed to settle on Base; callers pick both the payer chain and the target chain. Three externally visible things changed.

## 1. `CreateIntent` request

A new optional `target_chain` field selects the settlement chain. If you omit it, AgentPay defaults to `base`, so existing callers that only passed `payer_chain` continue to work unchanged.

**Before (v1)**

```typescript
await client.createIntent({
  email: "merchant@example.com",
  amount: "100.50",
  payerChain: "solana",
});
// Implicitly settled on Base.
```

**After (v2)**

```typescript
await client.createIntent({
  email: "merchant@example.com",
  amount: "100.50",
  payerChain: "base",
  targetChain: "ethereum",
});
```

The `recipient` address format is validated against `target_chain`: EVM address for EVM targets, Solana address for `solana`. For email recipients, Privy resolves the email to a wallet on the target chain.

## 2. Status enum rename

The `BASE_*` statuses were renamed to `TARGET_*`. The old names are gone — this is a one-shot break, not a deprecation.

| Before | After |
| :--- | :--- |
| `BASE_SETTLING` | `TARGET_SETTLING` |
| `BASE_SETTLED` | `TARGET_SETTLED` |

Other statuses are unchanged: `AWAITING_PAYMENT`, `PENDING`, `SOURCE_SETTLED`, `VERIFICATION_FAILED`, `PARTIAL_SETTLEMENT`, `EXPIRED`.

A new internal transition — `TARGET_SETTLING → SOURCE_SETTLED` — can occur when target-chain settlement cannot be dispatched; AgentPay rolls back and retries. See [Statuses](../../api/statuses.md).

## 3. Response field rename

The `GetIntent` response now exposes the receipt as `target_payment`. The v1 field `base_payment` is gone.

**Before (v1)**

```json
{
  "status": "BASE_SETTLED",
  "base_payment": {
    "tx_hash": "0x...",
    "settle_proof": "...",
    "settled_at": "2024-01-01T12:05:00Z",
    "explorer_url": "https://basescan.org/tx/0x..."
  }
}
```

**After (v2)**

```json
{
  "status": "TARGET_SETTLED",
  "target_payment": {
    "tx_hash": "0x...",
    "settle_proof": "...",
    "settled_at": "2024-01-01T12:05:00Z",
    "explorer_url": "https://etherscan.io/tx/0x..."
  }
}
```

`target_payment` is omitted until the intent reaches `TARGET_SETTLED`. The separate `source_payment` field (present in both v1 and v2) carries the source-chain receipt.

## SDK constant rename

### TypeScript / JavaScript

| Before | After |
| :--- | :--- |
| `IntentStatus.BaseSettling` | `IntentStatus.TargetSettling` |
| `IntentStatus.BaseSettled` | `IntentStatus.TargetSettled` |
| `intent.basePayment` | `intent.targetPayment` |

### Go

| Before | After |
| :--- | :--- |
| `pay.StatusBaseSettling` | `pay.StatusTargetSettling` |
| `pay.StatusBaseSettled` | `pay.StatusTargetSettled` |
| `intent.BasePayment` | `intent.TargetPayment` |

## Minimal migration checklist

- [ ] Search the codebase for `BASE_SETTLED`, `BASE_SETTLING`, `base_payment`, `basePayment`, `BasePayment`, `StatusBaseSettled`, `StatusBaseSettling`, `IntentStatus.BaseSettled`, `IntentStatus.BaseSettling` — rename them to their `Target*` equivalents.
- [ ] Decide whether your merchant should receive on something other than Base; if so, add `target_chain` to your `CreateIntent` calls.
- [ ] If you validate merchant addresses client-side, make sure the format check matches the chosen `target_chain`.
- [ ] Confirm `GET /api/chains` in your environment lists the target chains you plan to use; unlisted chains will be rejected at intent creation.

## Related

- [Supported Chains](../../api/chains.md)
- [Intents](../../api/intents.md)
- [Statuses](../../api/statuses.md)
- [Multi-Chain Settlement](../concepts/multi-chain-settlement.md)
