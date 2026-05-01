# Execute Payment Intent

## Overview

- **Function**: Execute a payment intent using an agent-controlled wallet. The backend signs and moves USDC on the payer chain, then transfers to the merchant on the target chain.
- **Use Cases**: Automated payouts, batch payments, server-side payment processing
- **Authentication**: Required (Bearer token with API key and secret key)

## JSON Schema Definition

```json
{
  "name": "execute_payment_intent",
  "description": "Executes a payment intent using the Agent wallet. The backend signs on the payer chain and settles on the target chain automatically. Requires authentication.",
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
        "enum": ["PENDING", "SOURCE_SETTLED", "TARGET_SETTLING", "TARGET_SETTLED", "VERIFICATION_FAILED", "EXPIRED", "PARTIAL_SETTLEMENT"],
        "description": "Current status of the intent"
      },
      "target_payment": {
        "type": "object",
        "description": "Payment receipt information on the target chain (present on GetIntent when status is TARGET_SETTLED)",
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
      }
    },
    "required": ["intent_id", "status"]
  }
}
```

> The immediate response from `execute_payment_intent` contains the status and fee summary. The `target_payment` block itself is returned from `GetIntent` once the intent reaches `TARGET_SETTLED`.

## Parameters

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `intent_id` | string | Yes | The intent ID returned from createIntent operation |

## Return Value

### Success Response (after polling `GetIntent`)

```json
{
  "intent_id": "int_abc123xyz",
  "status": "TARGET_SETTLED",
  "target_payment": {
    "tx_hash": "0x1234...abcd",
    "settle_proof": "...",
    "settled_at": "2024-01-01T12:05:00Z",
    "explorer_url": "https://etherscan.io/tx/0x1234...abcd"
  }
}
```

### Status Values

- `PENDING`: Execution initiated, processing
- `SOURCE_SETTLED`: Payment confirmed on the payer chain
- `TARGET_SETTLING`: Settlement is being processed on the target chain
- `TARGET_SETTLED`: Transfer complete (terminal state)
- `VERIFICATION_FAILED`: Source payment verification failed (terminal state)
- `EXPIRED`: Intent was not executed within 10 minutes (terminal state)
- `PARTIAL_SETTLEMENT`: Source settled but target transfer did not complete (terminal state; contact support)

## Code Examples

### TypeScript/JavaScript

```typescript
import { PayClient, IntentStatus } from '@cross402/usdc';

const client = new PayClient({
  baseUrl: 'https://api-pay.agent.tech',
  auth: { apiKey: 'your-api-key', secretKey: 'your-secret-key' },
});

// Execute intent
const result = await client.executeIntent(intentId);

console.log(`Intent ID: ${result.intentId}`);
console.log(`Status: ${result.status}`);

// Poll for terminal status, then read the receipt:
const final = await client.getIntent(intentId);
if (final.status === IntentStatus.TargetSettled) {
  console.log(`Transaction hash: ${final.targetPayment.txHash}`);
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

    exec, err := client.ExecuteIntent(ctx, intentID)
    if err != nil {
        log.Fatal(err)
    }

    log.Printf("Intent ID: %s", exec.IntentID)
    log.Printf("Status: %s", exec.Status)

    // Poll GetIntent for the target_payment receipt
    final, err := client.GetIntent(ctx, intentID)
    if err != nil {
        log.Fatal(err)
    }
    if final.Status == pay.StatusTargetSettled {
        log.Printf("Transaction hash: %s", final.TargetPayment.TxHash)
    }
}
```

## Error Handling

### Common Errors

| HTTP Status | Error Type | Description | Solution |
|-------------|------------|-------------|----------|
| 400 | ValidationError | Empty intent ID | Provide a valid intent ID |
| 400 | RequestError | Intent expired or invalid status | Create a new intent |
| 401 | RequestError | Unauthorized - missing or invalid credentials | Provide valid API key and secret key |
| 402 | RequestError | Insufficient agent balance on payer chain | Top up the agent wallet for that chain |
| 404 | RequestError | Intent not found, **or** intent owned by another agent. The v2 endpoint collapses both rejections to the same `404 payment intent not found` body so callers cannot probe foreign intent IDs by observing a 403/404 split | Verify the intent ID exists *and* was created under the same API key |
| 429 | RequestError | Rate limited | Implement exponential backoff, retry after delay |
| 503 | RequestError | Insufficient proxy balance / service unavailable | Retry after delay |

> For comprehensive error handling patterns and retry strategies, see [Error Handling](error-handling.md).

## Important Notes

1. **Authentication Required**: This operation requires Bearer token authentication with valid API key and secret key.

2. **Automatic Processing**: The agent wallet automatically handles:
   - Signing the X402 authorization on the payer chain
   - Paying gas fees on both chains
   - Dispatching the USDC transfer on the target chain

3. **Status Progression**: `AWAITING_PAYMENT` → `PENDING` → `SOURCE_SETTLED` → `TARGET_SETTLING` → `TARGET_SETTLED`. If the target transfer can't be dispatched, the intent may briefly roll back to `SOURCE_SETTLED` and retry; see [Statuses](../api/statuses.md).

4. **Terminal States**: Once the status reaches `TARGET_SETTLED`, `EXPIRED`, `VERIFICATION_FAILED`, or `PARTIAL_SETTLEMENT`, it will not change.

5. **Polling**: For long-running operations, use [Query Intent Status](query-intent-status.md) to poll for status updates.

6. **Intent Expiration**: Intents expire 10 minutes after creation. Execute before expiration.

## Best Practices

1. **Error Recovery**: Implement retry logic with exponential backoff for transient errors (429, 503).

2. **Status Verification**: After execution, poll the intent status to confirm completion and read `target_payment`.

3. **Transaction Receipt**: When status is `TARGET_SETTLED`, save `target_payment.tx_hash` for record-keeping.

4. **Idempotency**: The operation is idempotent — safe to retry on network errors.

## Related Links

- [Create Intent](create-intent.md)
- [Query Intent Status](query-intent-status.md)
- [Payment Polling](payment-polling.md)
- [Error Handling](error-handling.md)
- [API Documentation: Intents](../api/intents.md)
