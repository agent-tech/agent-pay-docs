# Execute Payment Intent

## Overview

- **Function**: Execute a payment intent using the Agent wallet. The backend automatically signs and transfers USDC on Base.
- **Use Cases**: Automated payouts, batch payments, server-side payment processing
- **Authentication**: Required (Bearer token with API key and secret key)

## JSON Schema Definition

```json
{
  "name": "execute_payment_intent",
  "description": "Executes a payment intent using the Agent wallet. The backend signs and transfers USDC on Base automatically. Requires authentication.",
  "input_schema": {
    "type": "object",
    "properties": {
      "intent_id": {
        "type": "string",
        "description": "The intent ID returned from createIntent operation"
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
        "enum": ["PENDING", "SOURCE_SETTLED", "BASE_SETTLING", "BASE_SETTLED", "VERIFICATION_FAILED", "EXPIRED", "PARTIAL_SETTLEMENT"],
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
      }
    },
    "required": ["intent_id", "status"]
  }
}
```

## Parameters

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `intent_id` | string | Yes | The intent ID returned from createIntent operation |

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
  }
}
```

### Status Values

- `PENDING`: Execution initiated, processing
- `SOURCE_SETTLED`: Payment confirmed on the source chain
- `BASE_SETTLING`: Final settlement is being processed on the Base chain
- `BASE_SETTLED`: Transfer complete (terminal state)
- `VERIFICATION_FAILED`: Source payment verification failed (terminal state)
- `EXPIRED`: Intent was not executed within 10 minutes (terminal state)
- `PARTIAL_SETTLEMENT`: Partial amount settled on Base (terminal state)

## Code Examples

### TypeScript/JavaScript

```typescript
import { PayClient } from '@agenttech/pay';

const client = new PayClient({
  baseUrl: 'https://api-pay.agent.tech',
  auth: { apiKey: 'your-api-key', secretKey: 'your-secret-key' },
});

// Execute intent
const result = await client.executeIntent(intentId);

console.log(`Intent ID: ${result.intentId}`);
console.log(`Status: ${result.status}`);

if (result.status === 'BASE_SETTLED') {
  console.log(`Transaction hash: ${result.basePayment.txHash}`);
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
    
    // Execute intent
    exec, err := client.ExecuteIntent(ctx, intentID)
    if err != nil {
        log.Fatal(err)
    }

    log.Printf("Intent ID: %s", exec.IntentID)
    log.Printf("Status: %s", exec.Status)
    
    if exec.Status == pay.StatusBaseSettled {
        log.Printf("Transaction hash: %s", exec.BasePayment.TxHash)
    }
}
```

## Error Handling

### Common Errors

| HTTP Status | Error Type | Description | Solution |
|-------------|------------|-------------|----------|
| 400 | ValidationError | Empty intent ID | Provide a valid intent ID |
| 401 | RequestError | Unauthorized - missing or invalid credentials | Provide valid API key and secret key |
| 403 | RequestError | Forbidden - insufficient permissions | Check API key permissions |
| 404 | RequestError | Intent not found | Verify intent ID exists |
| 429 | RequestError | Rate limited | Implement exponential backoff, retry after delay |
| 503 | RequestError | Service unavailable | Retry after delay |

### Example Error Handling

```typescript
try {
  const result = await client.executeIntent(intentId);
} catch (error) {
  if (error instanceof PayApiError) {
    switch (error.statusCode) {
      case 401:
        console.error("Authentication failed. Check your API credentials.");
        break;
      case 404:
        console.error("Intent not found. Verify the intent ID.");
        break;
      case 429:
        console.error("Rate limited. Retry after delay.");
        // Implement exponential backoff
        break;
      default:
        console.error(`Error: ${error.statusCode} - ${error.body}`);
    }
  }
}
```

## Important Notes

1. **Authentication Required**: This operation requires Bearer token authentication with valid API key and secret key.

2. **Automatic Processing**: The Agent wallet automatically handles:
   - Signing the transaction
   - Paying gas fees
   - Transferring USDC on Base chain

3. **Status Progression**: The intent status typically progresses:
   - `AWAITING_PAYMENT` → `PENDING` → `SOURCE_SETTLED` → `BASE_SETTLING` → `BASE_SETTLED`

4. **Terminal States**: Once the status reaches `BASE_SETTLED`, `EXPIRED`, `VERIFICATION_FAILED`, or `PARTIAL_SETTLEMENT`, it will not change.

5. **Polling**: For long-running operations, use [Query Intent Status](query-intent-status.md) to poll for status updates.

6. **Intent Expiration**: Intents expire 10 minutes after creation. Execute before expiration.

## Best Practices

1. **Error Recovery**: Implement retry logic with exponential backoff for transient errors (429, 503).

2. **Status Verification**: After execution, poll the intent status to confirm completion.

3. **Transaction Receipt**: When status is `BASE_SETTLED`, save the transaction hash for record-keeping.

4. **Idempotency**: The operation is idempotent - safe to retry on network errors.

## Related Links

- [Create Intent](create-intent.md)
- [Query Intent Status](query-intent-status.md)
- [Payment Polling](payment-polling.md)
- [Error Handling](error-handling.md)
- [API Documentation: Intents](../api/intents.md)
