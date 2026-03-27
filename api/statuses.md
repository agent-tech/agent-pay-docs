# 🚦 Intent Statuses

Understand the lifecycle of an AgentTech Intent.

## Lifecycle Flow

```
                          ┌──────────────────┐
                          │ AWAITING_PAYMENT  │
                          └────────┬─────────┘
                                   │
                      ┌────────────┼────────────┐
                      │            │             │
                      ▼            ▼             ▼
               ┌──────────┐ ┌──────────┐ ┌─────────────────────┐
               │ EXPIRED  │ │ PENDING  │ │ VERIFICATION_FAILED │
               └──────────┘ └────┬─────┘ └─────────────────────┘
                                 │
                                 ▼
                        ┌────────────────┐
                        │ SOURCE_SETTLED │
                        └───────┬────────┘
                                │
                                ▼
                        ┌───────────────┐
                        │ BASE_SETTLING │
                        └───────┬───────┘
                                │
                       ┌────────┼────────┐
                       ▼                 ▼
              ┌──────────────┐  ┌──────────────────────┐
              │ BASE_SETTLED │  │ PARTIAL_SETTLEMENT   │
              └──────────────┘  └──────────────────────┘
```

## Status Reference

| Status | Description |
| :--- | :--- |
| `AWAITING_PAYMENT` | Intent created; waiting for the payer to initiate transfer. |
| `PENDING` | Execution initiated, processing. |
| `VERIFICATION_FAILED` | Source payment verification failed (terminal). |
| `SOURCE_SETTLED` | Payment confirmed on the source chain (e.g., Solana). |
| `BASE_SETTLING` | Final settlement is being processed on the Base chain. |
| `BASE_SETTLED` | Success. Funds have arrived at the destination (terminal). |
| `PARTIAL_SETTLEMENT` | Partial amount settled on Base; remainder not fulfilled (terminal). |
| `EXPIRED` | Intent was not executed within 10 minutes (terminal). |

## SDK Constants

Use typed constants instead of raw strings when checking intent status.

### TypeScript / JavaScript

```typescript
import { IntentStatus } from '@cross402/usdc';
```

| Constant | Value |
| :--- | :--- |
| `IntentStatus.AwaitingPayment` | `"AWAITING_PAYMENT"` |
| `IntentStatus.Pending` | `"PENDING"` |
| `IntentStatus.VerificationFailed` | `"VERIFICATION_FAILED"` |
| `IntentStatus.SourceSettled` | `"SOURCE_SETTLED"` |
| `IntentStatus.BaseSettling` | `"BASE_SETTLING"` |
| `IntentStatus.BaseSettled` | `"BASE_SETTLED"` |
| `IntentStatus.PartialSettlement` | `"PARTIAL_SETTLEMENT"` |
| `IntentStatus.Expired` | `"EXPIRED"` |

### Go

| Constant | Value |
| :--- | :--- |
| `pay.StatusAwaitingPayment` | `"AWAITING_PAYMENT"` |
| `pay.StatusPending` | `"PENDING"` |
| `pay.StatusVerificationFailed` | `"VERIFICATION_FAILED"` |
| `pay.StatusSourceSettled` | `"SOURCE_SETTLED"` |
| `pay.StatusBaseSettling` | `"BASE_SETTLING"` |
| `pay.StatusBaseSettled` | `"BASE_SETTLED"` |
| `pay.StatusPartialSettlement` | `"PARTIAL_SETTLEMENT"` |
| `pay.StatusExpired` | `"EXPIRED"` |

## Terminal States

The following states are terminal — the intent will not transition to any other state:
* `BASE_SETTLED` — Success
* `EXPIRED` — Timeout
* `VERIFICATION_FAILED` — Verification error
* `PARTIAL_SETTLEMENT` — Partial settlement
