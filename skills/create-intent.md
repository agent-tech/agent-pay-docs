# Create Payment Intent

## Overview

- **Function**: Create a new payment intent for USDC settlement on Base chain
- **Use Cases**: E-commerce checkout, initiate payment request, automated payouts
- **Authentication**: Optional (public mode) or Required (authenticated mode with Bearer token)

## JSON Schema Definition

```json
{
  "name": "create_payment_intent",
  "description": "Creates a new payment intent for USDC settlement on Base chain. Returns an intent ID for subsequent operations.",
  "input_schema": {
    "type": "object",
    "properties": {
      "email": {
        "type": "string",
        "description": "Recipient email address. Required if recipient is not provided."
      },
      "recipient": {
        "type": "string",
        "description": "Recipient wallet address. Required if email is not provided."
      },
      "amount": {
        "type": "string",
        "description": "USDC amount as string (e.g. '100.50'). Minimum: 0.01, Maximum: 1,000,000. Up to 6 decimal places.",
        "pattern": "^[0-9]+(\\.[0-9]{1,6})?$"
      },
      "payer_chain": {
        "type": "string",
        "description": "Source chain identifier. See Supported Chains documentation for the full list of supported chains."
      }
    },
    "required": ["amount", "payer_chain"],
    "oneOf": [
      { "required": ["email"] },
      { "required": ["recipient"] }
    ]
  },
  "output_schema": {
    "type": "object",
    "properties": {
      "intent_id": {
        "type": "string",
        "description": "Unique identifier for the created intent"
      },
      "status": {
        "type": "string",
        "enum": ["AWAITING_PAYMENT"],
        "description": "Initial status of the intent"
      },
      "expires_at": {
        "type": "string",
        "format": "date-time",
        "description": "ISO 8601 timestamp when the intent expires (10 minutes after creation)"
      }
    },
    "required": ["intent_id", "status"]
  }
}
```

## Parameters

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `email` | string | One of email/recipient | Recipient email address |
| `recipient` | string | One of email/recipient | Recipient wallet address |
| `amount` | string | Yes | USDC amount as string (e.g. "100.50") |
| `payer_chain` | string | Yes | Source chain identifier. See [Supported Chains](../api/chains.md) for the full list. |

### Amount Rules

- **Minimum**: 0.01 USDC
- **Note**: The JS/TS SDK enforces a 0.2 USDC minimum client-side.
- **Maximum**: 1,000,000 USDC
- **Precision**: Up to 6 decimal places (e.g. `"0.000001"`, `"123.45"`)

## Return Value

### Success Response

```json
{
  "intent_id": "int_abc123xyz",
  "status": "AWAITING_PAYMENT",
  "expires_at": "2024-01-01T12:10:00Z",
  "fee_breakdown": {
    "source_chain": "base",
    "source_chain_fee": "0.001",
    "target_chain": "base",
    "target_chain_fee": "0.0001",
    "platform_fee": "1.00",
    "platform_fee_percentage": "1.0",
    "total_fee": "1.0011"
  }
}
```

### Error Response

See [Error Handling](error-handling.md) for detailed error information.

## Code Examples

### TypeScript/JavaScript

```typescript
import { PayClient } from '@agenttech/pay';

const client = new PayClient({
  baseUrl: 'https://api-pay.agent.tech',
  auth: { apiKey: 'your-api-key', secretKey: 'your-secret-key' },
});

// Create intent with email
const intent = await client.createIntent({
  email: "merchant@example.com",
  amount: "100.50",
  payerChain: "base"
});

console.log(`Intent created: ${intent.intentId}`);
console.log(`Status: ${intent.status}`);
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
    
    // Create intent with email
    resp, err := client.CreateIntent(ctx, &pay.CreateIntentRequest{
        Email:      "merchant@example.com",
        Amount:     "100.50",
        PayerChain: "base",
    })
    if err != nil {
        log.Fatal(err)
    }

    log.Printf("Intent created: %s", resp.IntentID)
    log.Printf("Status: %s", resp.Status)
}
```

## Error Handling

### Common Errors

| HTTP Status | Error Type | Description | Solution |
|-------------|------------|-------------|----------|
| 400 | ValidationError | Invalid amount (out of range) | Ensure amount is between 0.01 and 1,000,000 USDC |
| 400 | ValidationError | Missing email or recipient | Provide either email or recipient |
| 400 | ValidationError | Invalid payer_chain | Use a valid chain identifier (see Supported Chains) |
| 401 | RequestError | Unauthorized (if auth required) | Provide valid API key and secret key |
| 429 | RequestError | Rate limited | Implement exponential backoff, retry after delay |

### Example Error Handling

```typescript
try {
  const intent = await client.createIntent({
    email: "merchant@example.com",
    amount: "100.50",
    payerChain: "base"
  });
} catch (error) {
  if (error instanceof PayApiError) {
    if (error.statusCode === 400) {
      console.error("Invalid request:", error.body);
    } else if (error.statusCode === 429) {
      console.error("Rate limited, retry after delay");
    }
  }
}
```

## Important Notes

1. **Intent Expiration**: Intents expire 10 minutes after creation. If not executed within this window, the status becomes `EXPIRED` (terminal state).

2. **Authentication Modes**:
   - **Public Mode**: No auth required, uses `/api` endpoints
   - **Authenticated Mode**: Requires Bearer token, uses `/v2` endpoints

3. **Chain Support**: See [Supported Chains](../api/chains.md) for the full list of 14 supported chains (7 testnet + 7 mainnet).

4. **Settlement**: All payments ultimately settle on Base chain regardless of the source chain.

## Related Links

- [API Documentation: Intents](../api/intents.md)
- [Execute Intent](execute-intent.md)
- [Query Intent Status](query-intent-status.md)
- [Error Handling](error-handling.md)
