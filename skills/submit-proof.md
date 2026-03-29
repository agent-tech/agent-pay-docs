# Submit Payment Proof

## Overview

- **Function**: Submit a settlement proof after the payer completes X402 payment on the source chain
- **Use Cases**: Client-side payment flows, user-initiated payments with wallet signatures
- **Authentication**: Not required (public endpoint)

## JSON Schema Definition

```json
{
  "name": "submit_payment_proof",
  "description": "Submits a settlement proof after the payer completes X402 payment on the source chain. No authentication required.",
  "input_schema": {
    "type": "object",
    "properties": {
      "intent_id": {
        "type": "string",
        "description": "The intent ID returned from createIntent operation"
      },
      "settle_proof": {
        "type": "string",
        "description": "The settlement proof from X402 payment. This is the signed proof from the user's wallet after completing the payment on the source chain."
      }
    },
    "required": ["intent_id", "settle_proof"]
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
        "enum": ["PENDING", "SOURCE_SETTLED", "BASE_SETTLING", "BASE_SETTLED", "VERIFICATION_FAILED", "PARTIAL_SETTLEMENT"],
        "description": "Current status of the intent after proof submission"
      },
      "message": {
        "type": "string",
        "description": "Status message"
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
| `settle_proof` | string | Yes | The settlement proof from X402 payment (signed by user's wallet) |

## Return Value

### Success Response

```json
{
  "intent_id": "int_abc123xyz",
  "status": "PENDING",
  "message": "Proof submitted successfully"
}
```

### Status Values

- `PENDING`: Proof submitted, verification in progress
- `SOURCE_SETTLED`: Payment confirmed on the source chain
- `BASE_SETTLING`: Final settlement is being processed on the Base chain
- `BASE_SETTLED`: Transfer complete (terminal state)
- `VERIFICATION_FAILED`: Proof verification failed (terminal state)
- `PARTIAL_SETTLEMENT`: Partial amount settled on Base (terminal state)

## Code Examples

### TypeScript/JavaScript

```typescript
import { PublicPayClient } from '@cross402/usdc';

// Use PublicPayClient for client-side operations (no secrets required)
const client = new PublicPayClient({
  baseUrl: 'https://api-pay.agent.tech',
});

// After user signs X402 payment with their wallet
const settleProof = await getUserSignedProof(); // Get proof from wallet

// Submit the proof
const result = await client.submitProof(intentId, settleProof);

console.log(`Intent ID: ${result.intentId}`);
console.log(`Status: ${result.status}`);
console.log(`Message: ${result.message}`);
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
    // Public mode - no authentication required
    client, err := pay.NewClient("https://api-pay.agent.tech")
    if err != nil {
        log.Fatal(err)
    }

    ctx := context.Background()
    
    // Get settle proof from user's wallet (X402 payment)
    settleProof := getUserSignedProof()
    
    // Submit the proof
    proof, err := client.SubmitProof(ctx, intentID, settleProof)
    if err != nil {
        log.Fatal(err)
    }

    log.Printf("Intent ID: %s", proof.IntentID)
    log.Printf("Status: %s", proof.Status)
    log.Printf("Message: %s", proof.Message)
}
```

## Payment Flow

### X402 Payment Process

1. **Create Intent**: User initiates payment, create intent via SDK
2. **User Signs**: User signs X402 payment off-chain via their wallet (MetaMask, etc.)
3. **Get Proof**: Wallet returns signed settlement proof
4. **Submit Proof**: Submit proof to AgentTech API using this skill
5. **Verification**: AgentTech verifies the proof and processes settlement
6. **Settlement**: Final USDC transfer on Base chain

### Example Flow

```typescript
// Step 1: Create intent
const intent = await client.createIntent({
  email: "merchant@example.com",
  amount: "100.50",
  payerChain: "base"
});

// Step 2: User signs X402 payment (wallet interaction)
const settleProof = await signX402Payment(intent.intentId);

// Step 3: Submit proof
const result = await client.submitProof(intent.intentId, settleProof);

// Step 4: Poll for status (see query-intent-status.md)
// ...
```

## Error Handling

### Common Errors

| HTTP Status | Error Type | Description | Solution |
|-------------|------------|-------------|----------|
| 400 | ValidationError | Empty intent ID or settle proof | Provide valid intent ID and proof |
| 400 | ValidationError | Invalid settle proof format | Verify proof is correctly signed |
| 404 | RequestError | Intent not found | Verify intent ID exists |
| 400 | RequestError | Proof verification failed | Verify proof matches the intent |
| 429 | RequestError | Rate limited | Implement exponential backoff, retry after delay |
| 503 | RequestError | Service unavailable | Retry after delay |

> For comprehensive error handling patterns and retry strategies, see [Error Handling](error-handling.md).

## Important Notes

1. **No Authentication Required**: This is a public endpoint, safe for client-side use.

2. **X402 Payment**: The settle proof must come from a valid X402 payment signed by the user's wallet.

3. **Proof Format**: The settle proof is a string containing the signed payment data from the wallet.

4. **Verification**: AgentTech verifies the proof before processing. Invalid proofs will result in `VERIFICATION_FAILED` status.

5. **Status Progression**: After successful proof submission:
   - `PENDING` → `SOURCE_SETTLED` → `BASE_SETTLING` → `BASE_SETTLED`

6. **Polling**: Use [Query Intent Status](query-intent-status.md) to poll for status updates after submission.

## Best Practices

1. **Client-Side Use**: Use `PublicPayClient` (JavaScript/TypeScript) or public mode (Go) for client-side operations.

2. **Error Handling**: Implement proper error handling for proof submission failures.

3. **Status Verification**: After submission, poll the intent status to confirm processing.

4. **User Experience**: Show loading states while proof is being verified and processed.

5. **Retry Logic**: For transient errors (429, 503), implement retry with exponential backoff.

## Related Links

- [Create Intent](create-intent.md)
- [Query Intent Status](query-intent-status.md)
- [Payment Polling](payment-polling.md)
- [Error Handling](error-handling.md)
- [API Documentation: Intents](../api/intents.md)
