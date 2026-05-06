---
order: 1
---

# Intent Statuses

> Source: https://docs.agent.tech/api/statuses/

# Intent Statuses

Understand the lifecycle of a Cross402 Intent.

## Lifecycle Flow

```
                          ┌──────────────────┐
                          │ AWAITING_PAYMENT │
                          └────────┬─────────┘
                                   │
                      ┌────────────┼────────────┐
                      │            │            │
                      ▼            ▼            ▼
               ┌──────────┐ ┌──────────┐ ┌─────────────────────┐
               │ EXPIRED  │ │ PENDING  │ │ VERIFICATION_FAILED │
               └──────────┘ └────┬─────┘ └─────────────────────┘
                                 │
                                 ▼
                        ┌────────────────┐
                        │ SOURCE_SETTLED │◄────────────┐
                        └───────┬────────┘             │
                                │                      │ rollback on
                                ▼                      │ target failure
                        ┌─────────────────┐            │
                        │ TARGET_SETTLING │────────────┘
                        └────────┬────────┘
                                 │
                       ┌─────────┼─────────┐
                       ▼                   ▼
              ┌────────────────┐  ┌────────────────────┐
              │ TARGET_SETTLED │  │ PARTIAL_SETTLEMENT │
              └────────────────┘  └────────────────────┘
```

## Status Reference

| Status | Description |
| --- | --- |
| `AWAITING_PAYMENT` | Intent created; waiting for the payer to initiate transfer. |
| `PENDING` | Execution initiated, processing. |
| `VERIFICATION_FAILED` | Source payment verification failed (terminal). |
| `SOURCE_SETTLED` | Payment confirmed on the source (payer) chain. |
| `TARGET_SETTLING` | Settlement is being processed on the target chain. |
| `TARGET_SETTLED` | Success. Funds have arrived at the merchant on the target chain (terminal). |
| `PARTIAL_SETTLEMENT` | Source settled but target transfer could not be completed; remainder requires reconciliation (terminal). |
| `EXPIRED` | Intent was not executed within 10 minutes (terminal). |

### Rollback: `TARGET_SETTLING → SOURCE_SETTLED`

If the target-chain transfer cannot be dispatched — for example the agent's wallet for the target chain is missing or the payment service is temporarily unavailable — the intent is atomically rolled back from `TARGET_SETTLING` to `SOURCE_SETTLED` so the settlement pipeline can retry. This is an internal transition; to a polling caller the status briefly becomes `TARGET_SETTLING` then returns to `SOURCE_SETTLED`, then proceeds to `TARGET_SETTLED` on the next attempt.

`PARTIAL_SETTLEMENT` is distinct from this rollback: it is terminal and signals that the source chain settled but the target-chain settlement could not be completed at all. Funds on the payer side have moved; contact support for reconciliation.

## SDK Constants

Use typed constants instead of raw strings when checking intent status.

### TypeScript / JavaScript

```
import { IntentStatus } from '@cross402/usdc';
```

| Constant | Value |
| --- | --- |
| `IntentStatus.AwaitingPayment` | `"AWAITING_PAYMENT"` |
| `IntentStatus.Pending` | `"PENDING"` |
| `IntentStatus.VerificationFailed` | `"VERIFICATION_FAILED"` |
| `IntentStatus.SourceSettled` | `"SOURCE_SETTLED"` |
| `IntentStatus.TargetSettling` | `"TARGET_SETTLING"` |
| `IntentStatus.TargetSettled` | `"TARGET_SETTLED"` |
| `IntentStatus.PartialSettlement` | `"PARTIAL_SETTLEMENT"` |
| `IntentStatus.Expired` | `"EXPIRED"` |

### Go

| Constant | Value |
| --- | --- |
| `pay.StatusAwaitingPayment` | `"AWAITING_PAYMENT"` |
| `pay.StatusPending` | `"PENDING"` |
| `pay.StatusVerificationFailed` | `"VERIFICATION_FAILED"` |
| `pay.StatusSourceSettled` | `"SOURCE_SETTLED"` |
| `pay.StatusTargetSettling` | `"TARGET_SETTLING"` |
| `pay.StatusTargetSettled` | `"TARGET_SETTLED"` |
| `pay.StatusPartialSettlement` | `"PARTIAL_SETTLEMENT"` |
| `pay.StatusExpired` | `"EXPIRED"` |

## Terminal States

The following states are terminal — the intent will not transition to any other state:

* `TARGET_SETTLED` — Success
* `EXPIRED` — Timeout
* `VERIFICATION_FAILED` — Verification error
* `PARTIAL_SETTLEMENT` — Source settled, target did not complete
