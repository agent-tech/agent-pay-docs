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
      "status": {
        "type": "string",
        "enum": ["AWAITING_PAYMENT", "PENDING", "SOURCE_SETTLED", "BASE_SETTLING", "BASE_SETTLED", "VERIFICATION_FAILED", "EXPIRED", "PARTIAL_SETTLEMENT"],
        "description": "Current status of the intent"
      },
      "base_payment": {
        "type": "object",
        "description": "Payment receipt information (available when status is BASE_SETTLED)",
        "properties": {
          "tx_hash": {
            "type": "string",
            "description": "Transaction hash on Base chain"
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
            "description": "Block explorer URL"
          }
        }
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
  "status": "BASE_SETTLED",
  "base_payment": {
    "tx_hash": "0x1234...abcd",
    "settle_proof": "...",
    "settled_at": "2024-01-01T12:05:00Z",
    "explorer_url": "https://basescan.org/tx/0x1234...abcd"
  },
  "fee_breakdown": {
    "source_chain": "base",
    "source_chain_fee": "0.001",
    "target_chain": "base",
    "target_chain_fee": "0.0001",
    "platform_fee": "1.00",
    "platform_fee_percentage": "1.0",
    "total_fee": "1.0011"
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
| `SOURCE_SETTLED` | Payment confirmed on the source chain | No |
| `BASE_SETTLING` | Final settlement is being processed on the Base chain | No |
| `BASE_SETTLED` | Success. Funds have arrived at the destination | Yes |
| `VERIFICATION_FAILED` | Source payment verification failed | Yes |
| `EXPIRED` | Intent was not executed within 10 minutes | Yes |
| `PARTIAL_SETTLEMENT` | Partial amount settled on Base | Yes |

## Code Examples

### TypeScript/JavaScript

```typescript
import { PayClient } from '@agenttech/pay';

const client = new PayClient({
  baseUrl: 'https://api-pay.agent.tech',
  auth: { apiKey: 'your-api-key', secretKey: 'your-secret-key' },
});

// Query intent status
const intent = await client.getIntent(intentId);

console.log(`Intent ID: ${intent.intentId}`);
console.log(`Status: ${intent.status}`);

// Handle different statuses
switch (intent.status) {
  case 'BASE_SETTLED':
    console.log('Payment complete!');
    console.log(`Transaction hash: ${intent.basePayment.txHash}`);
    break;
  case 'EXPIRED':
  case 'VERIFICATION_FAILED':
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
    "github.com/agent-tech/agent-sdk-go"
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
    
    // Query intent status
    intent, err := client.GetIntent(ctx, intentID)
    if err != nil {
        log.Fatal(err)
    }

    log.Printf("Intent ID: %s", intent.IntentID)
    log.Printf("Status: %s", intent.Status)

    // Handle different statuses
    switch intent.Status {
    case pay.StatusBaseSettled:
        log.Println("Payment complete!")
        log.Printf("Transaction hash: %s", intent.BasePayment.TxHash)
    case pay.StatusExpired, pay.StatusVerificationFailed:
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
            ├──> PARTIAL_SETTLEMENT (if partial amount settled)
            │
            └──> SOURCE_SETTLED
                    │
                    └──> BASE_SETTLING
                            │
                            └──> BASE_SETTLED (success)
```

## Error Handling

### Common Errors

| HTTP Status | Error Type | Description | Solution |
|-------------|------------|-------------|----------|
| 400 | ValidationError | Empty intent ID | Provide a valid intent ID |
| 404 | RequestError | Intent not found | Verify intent ID exists |
| 429 | RequestError | Rate limited | Implement exponential backoff, retry after delay |
| 503 | RequestError | Service unavailable | Retry after delay |

### Example Error Handling

```typescript
try {
  const intent = await client.getIntent(intentId);
} catch (error) {
  if (error instanceof PayApiError) {
    switch (error.statusCode) {
      case 404:
        console.error("Intent not found. Verify the intent ID.");
        break;
      case 429:
        console.error("Rate limited. Retry after delay.");
        break;
      default:
        console.error(`Error: ${error.statusCode} - ${error.body}`);
    }
  }
}
```

## Polling Strategy

### Basic Polling

```typescript
async function pollIntentStatus(intentId: string, maxAttempts: number = 30) {
  for (let i = 0; i < maxAttempts; i++) {
    const intent = await client.getIntent(intentId);
    
    // Check for terminal states
    if (intent.status === 'BASE_SETTLED') {
      return { success: true, intent };
    }
    
    if (intent.status === 'EXPIRED' || intent.status === 'VERIFICATION_FAILED' || intent.status === 'PARTIAL_SETTLEMENT') {
      return { success: false, intent };
    }

    // Wait before next poll (2-5 seconds recommended)
    await new Promise(resolve => setTimeout(resolve, 3000));
  }
  
  throw new Error('Polling timeout');
}
```

### Exponential Backoff Polling

```typescript
async function pollWithBackoff(intentId: string) {
  let delay = 2000; // Start with 2 seconds
  const maxDelay = 30000; // Max 30 seconds
  
  while (true) {
    const intent = await client.getIntent(intentId);
    
    if (intent.status === 'BASE_SETTLED') {
      return intent;
    }
    
    if (intent.status === 'EXPIRED' || intent.status === 'VERIFICATION_FAILED' || intent.status === 'PARTIAL_SETTLEMENT') {
      throw new Error(`Payment failed: ${intent.status}`);
    }
    
    // Exponential backoff
    await new Promise(resolve => setTimeout(resolve, delay));
    delay = Math.min(delay * 1.5, maxDelay);
  }
}
```

## Important Notes

1. **Terminal States**: Once status reaches `BASE_SETTLED`, `EXPIRED`, `VERIFICATION_FAILED`, or `PARTIAL_SETTLEMENT`, it will not change. Stop polling.

2. **Polling Interval**: Recommended polling interval is 2-5 seconds. Avoid polling too frequently to respect rate limits (60 req/min/IP).

3. **Timeout**: Intents expire 10 minutes after creation. If status is still `AWAITING_PAYMENT` after expiration, it will become `EXPIRED`.

4. **Transaction Receipt**: When status is `BASE_SETTLED`, save the `basePayment.txHash` for record-keeping.

5. **Rate Limiting**: Be mindful of rate limits. Use exponential backoff for retries.

## Best Practices

1. **Polling Strategy**: Use exponential backoff to reduce API calls while maintaining responsiveness.

2. **Terminal State Detection**: Always check for terminal states (`BASE_SETTLED`, `EXPIRED`, `VERIFICATION_FAILED`) to stop polling.

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
