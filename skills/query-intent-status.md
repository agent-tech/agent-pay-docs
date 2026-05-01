# Query Intent Status

## Overview

- **Function**: Query the current status of a payment intent
- **Use Cases**: Status polling, payment confirmation, monitoring payment progress
- **Authentication**: Optional (works with or without authentication)

## JSON Schema Definition

```json
{
  "name": "query_intent_status",
  "description": "Queries the current status of a payment intent. Use this to poll for status updates.",
  "input_schema": {
    "type": "object",
    "properties": {
      "intent_id": {
        "type": "string",
        "description": "The intent ID to query"
      }
    },
    "required": ["intent_id"]
  },
  "output_schema": {
    "type": "object",
    "properties": {
      "intent_id": {
        "type": "string",
        "description": "The intent ID"
      },
      "agent_id": {
        "type": "string",
        "description": "UUID of the owning agent. Present only on /v2 responses (intents created with API-key auth); absent on the public /api flow."
      },
      "status": {
        "type": "string",
        "enum": ["AWAITING_PAYMENT", "PENDING", "SOURCE_SETTLED", "TARGET_SETTLING", "TARGET_SETTLED", "VERIFICATION_FAILED", "EXPIRED", "PARTIAL_SETTLEMENT"],
        "description": "Current status of the intent"
      },
      "target_payment": {
        "type": "object",
        "description": "Payment receipt on the target chain (present when status is TARGET_SETTLED)",
        "properties": {
          "tx_hash": {
            "type": "string",
            "description": "Transaction hash on the target chain"
          },
          "settle_proof": {
            "type": "string",
            "description": "Settlement proof"
          },
          "settled_at": {
            "type": "string",
            "description": "Settlement timestamp"
          },
          "explorer_url": {
            "type": "string",
            "description": "Block explorer URL on the target chain"
          }
        }
      },
      "source_payment": {
        "type": "object",
        "description": "Payment details on the payer (source) chain (present once the source chain has settled)"
      },
      "fee_breakdown": {
        "type": "object",
        "description": "Fee breakdown information"
      },
      "created_at": {
        "type": "string",
        "format": "date-time",
        "description": "ISO 8601 timestamp when the intent was created"
      },
      "expires_at": {
        "type": "string",
        "format": "date-time",
        "description": "ISO 8601 timestamp when the intent expires"
      }
    },
    "required": ["intent_id", "status"]
  }
}
```

## Parameters

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `intent_id` | string | Yes | The intent ID to query |

## Return Value

### Success Response

```json
{
  "intent_id": "int_abc123xyz",
  "agent_id": "8b2e9c4a-3f7a-4d1b-9e2c-5a6b7c8d9e0f",
  "status": "TARGET_SETTLED",
  "payer_chain": "base",
  "target_chain": "ethereum",
  "merchant_recipient": "0x742d35Cc...",
  "target_payment": {
    "tx_hash": "0x1234...abcd",
    "settle_proof": "...",
    "settled_at": "2024-01-01T12:05:00Z",
    "explorer_url": "https://etherscan.io/tx/0x1234...abcd"
  },
  "source_payment": {
    "chain": "base",
    "tx_hash": "0x9876...fedc",
    "settle_proof": "...",
    "settled_at": "2024-01-01T12:04:30Z",
    "explorer_url": "https://basescan.org/tx/0x9876...fedc"
  },
  "fee_breakdown": {
    "source_chain": "base",
    "source_chain_fee": "0.001",
    "target_chain": "ethereum",
    "target_chain_fee": "0.85",
    "platform_fee": "1.00",
    "platform_fee_percentage": "1.0",
    "total_fee": "1.851"
  },
  "created_at": "2024-01-01T12:00:00Z",
  "expires_at": "2024-01-01T12:10:00Z"
}
```

### Status Values

| Status | Description | Terminal |
|--------|-------------|----------|
| `AWAITING_PAYMENT` | Intent created; waiting for the payer to initiate transfer | No |
| `PENDING` | Execution initiated, processing | No |
| `SOURCE_SETTLED` | Payment confirmed on the payer chain | No |
| `TARGET_SETTLING` | Settlement is being processed on the target chain | No |
| `TARGET_SETTLED` | Success. Funds have arrived at the merchant on the target chain | Yes |
| `VERIFICATION_FAILED` | Source payment verification failed | Yes |
| `EXPIRED` | Intent was not executed within 10 minutes | Yes |
| `PARTIAL_SETTLEMENT` | Source settled but target transfer could not be completed | Yes |

## Code Examples

### TypeScript/JavaScript

```typescript
import { PayClient, IntentStatus } from '@cross402/usdc';

const client = new PayClient({
  baseUrl: 'https://api-pay.agent.tech',
  auth: { apiKey: 'your-api-key', secretKey: 'your-secret-key' },
});

// Query intent status
const intent = await client.getIntent(intentId);

console.log(`Intent ID: ${intent.intentId}`);
console.log(`Status: ${intent.status}`);

switch (intent.status) {
  case IntentStatus.TargetSettled:
    console.log('Payment complete!');
    console.log(`Transaction hash: ${intent.targetPayment.txHash}`);
    break;
  case IntentStatus.Expired:
  case IntentStatus.VerificationFailed:
  case IntentStatus.PartialSettlement:
    console.log('Payment failed:', intent.status);
    break;
  default:
    console.log('Payment still processing...');
    // Poll again
}
```

### Go

```go
package main

import (
    "context"
    "log"
    "github.com/cross402/usdc-sdk-go"
)

func main() {
    client, err := pay.NewClient(
        "https://api-pay.agent.tech",
        pay.WithBearerAuth(apiKey, secretKey),
    )
    if err != nil {
        log.Fatal(err)
    }

    ctx := context.Background()

    intent, err := client.GetIntent(ctx, intentID)
    if err != nil {
        log.Fatal(err)
    }

    log.Printf("Intent ID: %s", intent.IntentID)
    log.Printf("Status: %s", intent.Status)

    switch intent.Status {
    case pay.StatusTargetSettled:
        log.Println("Payment complete!")
        log.Printf("Transaction hash: %s", intent.TargetPayment.TxHash)
    case pay.StatusExpired, pay.StatusVerificationFailed, pay.StatusPartialSettlement:
        log.Printf("Payment failed: %s", intent.Status)
    default:
        log.Println("Payment still processing...")
        // Poll again
    }
}
```

## Status Lifecycle

```
AWAITING_PAYMENT
    │
    ├──> EXPIRED (if not executed within 10 minutes)
    │
    └──> PENDING (after execution)
            │
            ├──> VERIFICATION_FAILED (if verification fails)
            │
            └──> SOURCE_SETTLED
                    │
                    └──> TARGET_SETTLING
                            │
                            ├──> TARGET_SETTLED (success)
                            │
                            └──> PARTIAL_SETTLEMENT (source settled, target not completed)
```

`TARGET_SETTLING` may roll back to `SOURCE_SETTLED` and retry; this is transparent to polling callers and only observable as a transient state dip.

## Error Handling

### Common Errors

| HTTP Status | Error Type | Description | Solution |
|-------------|------------|-------------|----------|
| 400 | ValidationError | Empty intent ID | Provide a valid intent ID |
| 404 | RequestError | Intent not found, **or** intent owned by another agent (v2 ownership check returns 404, not 403, so the endpoint cannot be used to enumerate other agents' intent IDs) | Verify the intent ID and that it was created under the same API key |
| 429 | RequestError | Rate limited | Implement exponential backoff, retry after delay |
| 503 | RequestError | Service unavailable | Retry after delay |

> For comprehensive error handling patterns and retry strategies, see [Error Handling](error-handling.md).

> For polling strategies and best practices, see [Payment Polling](payment-polling.md).

## Important Notes

1. **Terminal States**: Once status reaches `TARGET_SETTLED`, `EXPIRED`, `VERIFICATION_FAILED`, or `PARTIAL_SETTLEMENT`, it will not change. Stop polling.

2. **Polling Interval**: Recommended polling interval is 2-5 seconds. Avoid polling too frequently to respect rate limits (60 req/min/IP).

3. **Timeout**: Intents expire 10 minutes after creation. If status is still `AWAITING_PAYMENT` after expiration, it will become `EXPIRED`.

4. **Transaction Receipt**: When status is `TARGET_SETTLED`, save `target_payment.tx_hash` for record-keeping.

5. **Rate Limiting**: Be mindful of rate limits. Use exponential backoff for retries.

6. **v2 ownership**: `GET /v2/intents` returns the intent only if its owning `agent_id` matches the API key's agent. Mismatches and intents created via the `/api` flow both return `404 payment intent not found`. Use the same API key that created the intent, or fall back to the public `GET /api/intents` endpoint.

7. **Optional quote fields on `getIntent`**: `sending_amount`, `receiving_amount`, `estimated_fee`, and `fee_breakdown` are populated by the backend after the intent leaves its initial state. Treat them as optional when polling — they are guaranteed only on `createIntent` / `executeIntent` responses.

## Best Practices

1. **Polling Strategy**: Use exponential backoff to reduce API calls while maintaining responsiveness.

2. **Terminal State Detection**: Always check for terminal states (`TARGET_SETTLED`, `EXPIRED`, `VERIFICATION_FAILED`, `PARTIAL_SETTLEMENT`) to stop polling.

3. **Error Handling**: Implement proper error handling for network errors and API errors.

4. **Timeout Handling**: Set a maximum polling duration to avoid infinite loops.

5. **User Feedback**: Update UI/UX based on status changes during polling.

## Related Links

- [Create Intent](create-intent.md)
- [Execute Intent](execute-intent.md)
- [Submit Proof](submit-proof.md)
- [Payment Polling](payment-polling.md)
- [Error Handling](error-handling.md)
- [API Documentation: Statuses](../api/statuses.md)
