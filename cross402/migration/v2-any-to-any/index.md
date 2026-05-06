# v2 Migration: Any-Chain → Any-Chain

> Source: https://docs.agent.tech/docs/migration/v2-any-to-any/

# v2 Migration: Any-Chain → Any-Chain

Cross402 v2 makes the target chain a first-class intent field. Payments are no longer assumed to settle on Base; callers pick both the payer chain and the target chain. Three externally visible things changed.

## 1. `CreateIntent` request

A new optional `target_chain` field selects the settlement chain. If you omit it, Cross402 defaults to `base`, so existing callers that only passed `payer_chain` continue to work unchanged.

**Before (v1)**

```
await client.createIntent({
  email: "merchant@example.com",
  amount: "100.50",
  payerChain: "solana",
});
// Implicitly settled on Base.
```

**After (v2)**

```
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
| --- | --- |
| `BASE_SETTLING` | `TARGET_SETTLING` |
| `BASE_SETTLED` | `TARGET_SETTLED` |

Other statuses are unchanged: `AWAITING_PAYMENT`, `PENDING`, `SOURCE_SETTLED`, `VERIFICATION_FAILED`, `PARTIAL_SETTLEMENT`, `EXPIRED`.

A new internal transition — `TARGET_SETTLING → SOURCE_SETTLED` — can occur when target-chain settlement cannot be dispatched; Cross402 rolls back and retries. See [Statuses](../../concepts/statuses/).

## 3. Response field rename

The `GetIntent` response now exposes the receipt as `target_payment`. The v1 field `base_payment` is gone.

**Before (v1)**

```
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

```
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
| --- | --- |
| `IntentStatus.BaseSettling` | `IntentStatus.TargetSettling` |
| `IntentStatus.BaseSettled` | `IntentStatus.TargetSettled` |
| `intent.basePayment` | `intent.targetPayment` |

### Go

| Before | After |
| --- | --- |
| `pay.StatusBaseSettling` | `pay.StatusTargetSettling` |
| `pay.StatusBaseSettled` | `pay.StatusTargetSettled` |
| `intent.BasePayment` | `intent.TargetPayment` |

## 4. v2 agent-scoped endpoints

v2 also adds three SDK-facing endpoints, all under `/v2` and authenticated:

| Endpoint | Description |
| --- | --- |
| `GET /v2/me` | Returns the calling agent's identity (id, number, name, status, EVM/Solana wallet addresses). Reads from middleware context, no DB hit. Useful as a credential health-check. |
| `GET /v2/intents/list` | Paginated list of intents owned by the calling agent, most recent first. `?page=` is 1-indexed; `?page_size=` is in `[1, 100]`. Out-of-range values now return `400` instead of silently clamping; values above the max collapse to the max. |
| `GET /v2/intents` | Existing endpoint, now enforcing **per-agent ownership**. Looking up an intent owned by another agent — or one created via the unauthenticated `/api` flow — returns `404 payment intent not found`. The same policy applies to `POST /v2/intents/{id}/execute`. |

The `404` response on cross-agent lookups is **deliberate**: collapsing 403 and 404 to the same body prevents authenticated callers from probing for valid intent IDs across other agents by observing the rejection split. SDK callers that previously distinguished 403 from 404 should now treat both as "not visible to me." The `403` status is reserved for future use.

`agent_id` is now surfaced on `CreateIntentResponse`, `IntentResponse`, `SubmitProofResponse`, and on every row of `ListIntentsResponse` (as `*string` + `omitempty`). For intents created via the unauthenticated `/api` flow, the field is omitted.

## Minimal migration checklist

* Search the codebase for `BASE_SETTLED`, `BASE_SETTLING`, `base_payment`, `basePayment`, `BasePayment`, `StatusBaseSettled`, `StatusBaseSettling`, `IntentStatus.BaseSettled`, `IntentStatus.BaseSettling` — rename them to their `Target*` equivalents.
* Decide whether your merchant should receive on something other than Base; if so, add `target_chain` to your `CreateIntent` calls.
* If you validate merchant addresses client-side, make sure the format check matches the chosen `target_chain`.
* Confirm `GET /api/chains` in your environment lists the target chains you plan to use; unlisted chains will be rejected at intent creation.
* If your error handling switches on `403`, fold it into the `404` branch — v2 ownership rejections no longer return `403`.
* If you call `GET /v2/intents/list` with custom `page` / `page_size`, ensure the values are in range; the server now returns `400` rather than silently falling back to defaults.

## Related

* [Supported Chains](../../../introduction/supported-networks/)
* [Intents](../../api/intents/)
* [Statuses](../../concepts/statuses/)
* [Multi-Chain Settlement](../../concepts/multi-chain-settlement/)