# 🚦 Intent Statuses

Understand the lifecycle of an AgentPay Intent.

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
                                ▼
                        ┌──────────────┐
                        │ BASE_SETTLED │
                        └──────────────┘
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
| `EXPIRED` | Intent was not executed within 10 minutes (terminal). |

## Terminal States

The following states are terminal — the intent will not transition to any other state:
* `BASE_SETTLED` — Success
* `EXPIRED` — Timeout
* `VERIFICATION_FAILED` — Verification error