# Create Payment Intent

## Overview

- **Function**: Create a new payment intent. Payer pays on `payer_chain`; merchant receives on `target_chain` (defaults to `base`).
- **Use Cases**: E-commerce checkout, initiate payment request, automated payouts
- **Authentication**: Optional (public mode) or Required (authenticated mode with Bearer token)

## JSON Schema Definition

```json
{
  "name": "create_payment_intent",
  "description": "Creates a new payment intent. The payer chain and target (settlement) chain can differ. Returns an intent ID for subsequent operations.",
  "input_schema": {
    "type": "object",
    "properties": {
      "email": {
        "type": "string",
        "description": "Recipient email address. Required if recipient is not provided."
      },
      "recipient": {
        "type": "string",
        "description": "Recipient wallet address. Required if email is not provided. Format is validated against target_chain (EVM address for EVM targets, Solana public key for target_chain='solana')."
      },
      "amount": {
        "type": "string",
        "description": "USDC amount as string (e.g. '100.50'). Minimum: 0.02, Maximum: 1,000,000. Up to 6 decimal places.",
        "pattern": "^[0-9]+(\\.[0-9]{1,6})?$"
      },
      "payer_chain": {
        "type": "string",
        "description": "Source chain identifier. See Supported Chains for the full list."
      },
      "target_chain": {
        "type": "string",
        "description": "Settlement chain identifier. Optional; defaults to 'base'. Must be a chain listed by GET /api/chains."
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
| `recipient` | string | One of email/recipient | Recipient wallet address; format validated against `target_chain` |
| `amount` | string | Yes | USDC amount as string (e.g. "100.50") |
| `payer_chain` | string | Yes | Source chain identifier. See [Supported Chains](../api/chains.md). |
| `target_chain` | string | No | Settlement chain identifier. Defaults to `"base"`. Must be listed by `GET /api/chains`. |

### Amount Rules

- **Minimum**: 0.02 USDC
- **Maximum**: 1,000,000 USDC
- **Precision**: Up to 6 decimal places (e.g. `"0.000001"`, `"123.45"`)

## Return Value

### Success Response

```json
{
  "intent_id": "int_abc123xyz",
  "agent_id": "8b2e9c4a-3f7a-4d1b-9e2c-5a6b7c8d9e0f",
  "status": "AWAITING_PAYMENT",
  "payer_chain": "base",
  "target_chain": "ethereum",
  "expires_at": "2024-01-01T12:10:00Z",
  "fee_breakdown": {
    "source_chain": "base",
    "source_chain_fee": "0.001",
    "target_chain": "ethereum",
    "target_chain_fee": "0.85",
    "platform_fee": "1.00",
    "platform_fee_percentage": "1.0",
    "total_fee": "1.851"
  }
}
```

`agent_id` is returned only on the authenticated `/v2` flow (it identifies the agent that owns the intent). The unauthenticated `/api` flow leaves it unset.

### Error Response

See [Error Handling](error-handling.md) for detailed error information.

## Code Examples

### TypeScript/JavaScript

```typescript
import { PayClient } from '@cross402/usdc';

const client = new PayClient({
  baseUrl: 'https://api-pay.agent.tech',
  auth: { apiKey: 'your-api-key', secretKey: 'your-secret-key' },
});

// Payer pays on Base; merchant receives on Ethereum.
const intent = await client.createIntent({
  email: "merchant@example.com",
  amount: "100.50",
  payerChain: "base",
  targetChain: "ethereum",
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

    resp, err := client.CreateIntent(ctx, &pay.CreateIntentRequest{
        Email:       "merchant@example.com",
        Amount:      "100.50",
        PayerChain:  "base",
        TargetChain: "ethereum",
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
| 400 | ValidationError | Invalid amount (out of range) | Ensure amount is between 0.02 and 1,000,000 USDC |
| 400 | ValidationError | Missing email or recipient | Provide either email or recipient |
| 400 | ValidationError | Invalid `payer_chain` | Use a valid chain identifier (see Supported Chains) |
| 400 | ValidationError | Invalid `target_chain` | Use a chain listed by `GET /api/chains` |
| 400 | ValidationError | Invalid recipient format | Recipient must match `target_chain` (EVM hex for EVM, Solana public key for `solana`) |
| 401 | RequestError | Unauthorized (if auth required) | Provide valid API key and secret key |
| 429 | RequestError | Rate limited | Implement exponential backoff, retry after delay |

> For comprehensive error handling patterns and retry strategies, see [Error Handling](error-handling.md).

## Important Notes

1. **Intent Expiration**: Intents expire 10 minutes after creation. If not executed within this window, the status becomes `EXPIRED` (terminal state).

2. **Authentication Modes**:
   - **Public Mode**: No auth required, uses `/api` endpoints
   - **Authenticated Mode**: Requires Bearer token, uses `/v2` endpoints

3. **Chain Support**: See [Supported Chains](../api/chains.md) for the full payer × target matrix.

4. **Settlement**: The merchant receives USDC on `target_chain`. Defaults to `base` if omitted. For the mechanics, see [Multi-Chain Settlement](../docs/concepts/multi-chain-settlement.md).

## Related Links

- [API Documentation: Intents](../api/intents.md)
- [Multi-Chain Settlement](../docs/concepts/multi-chain-settlement.md)
- [Execute Intent](execute-intent.md)
- [Query Intent Status](query-intent-status.md)
- [Error Handling](error-handling.md)
