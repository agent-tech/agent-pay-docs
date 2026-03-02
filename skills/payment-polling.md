# Payment Status Polling

## Overview

- **Function**: Best practices for polling payment intent status until completion
- **Use Cases**: Wait for payment completion, monitor payment progress, handle async payment flows
- **Authentication**: Optional (works with or without authentication)

## Polling Strategies

### 1. Basic Fixed Interval Polling

Simple polling with fixed intervals. Suitable for short-duration operations.

```typescript
async function pollStatus(intentId: string, intervalMs: number = 3000, maxAttempts: number = 60) {
  for (let attempt = 0; attempt < maxAttempts; attempt++) {
    const intent = await client.getIntent(intentId);
    
    // Terminal states - stop polling
    if (intent.status === 'BASE_SETTLED') {
      return { success: true, intent };
    }
    
    if (intent.status === 'EXPIRED' || intent.status === 'VERIFICATION_FAILED') {
      return { success: false, intent, reason: intent.status };
    }
    
    // Wait before next poll
    await new Promise(resolve => setTimeout(resolve, intervalMs));
  }
  
  throw new Error('Polling timeout - maximum attempts reached');
}
```

### 2. Exponential Backoff Polling

Reduces API calls over time. Recommended for longer operations.

```typescript
async function pollWithExponentialBackoff(intentId: string) {
  let delay = 2000; // Start with 2 seconds
  const maxDelay = 30000; // Maximum 30 seconds
  const maxDuration = 600000; // Maximum 10 minutes (intent expiration)
  const startTime = Date.now();
  
  while (true) {
    // Check timeout
    if (Date.now() - startTime > maxDuration) {
      throw new Error('Polling timeout - exceeded maximum duration');
    }
    
    const intent = await client.getIntent(intentId);
    
    // Terminal states - stop polling
    if (intent.status === 'BASE_SETTLED') {
      return { success: true, intent };
    }
    
    if (intent.status === 'EXPIRED' || intent.status === 'VERIFICATION_FAILED') {
      return { success: false, intent, reason: intent.status };
    }
    
    // Exponential backoff: increase delay by 1.5x each time
    await new Promise(resolve => setTimeout(resolve, delay));
    delay = Math.min(delay * 1.5, maxDelay);
  }
}
```

### 3. Adaptive Polling

Adjusts polling interval based on status progression.

```typescript
async function adaptivePoll(intentId: string) {
  let delay = 2000; // Initial delay: 2 seconds
  
  while (true) {
    const intent = await client.getIntent(intentId);
    
    // Terminal states
    if (intent.status === 'BASE_SETTLED') {
      return { success: true, intent };
    }
    
    if (intent.status === 'EXPIRED' || intent.status === 'VERIFICATION_FAILED') {
      return { success: false, intent, reason: intent.status };
    }
    
    // Adjust delay based on status
    switch (intent.status) {
      case 'AWAITING_PAYMENT':
        delay = 5000; // Poll every 5 seconds while waiting
        break;
      case 'PENDING':
      case 'SOURCE_SETTLED':
        delay = 3000; // Poll every 3 seconds during processing
        break;
      case 'BASE_SETTLING':
        delay = 2000; // Poll every 2 seconds during final settlement
        break;
    }
    
    await new Promise(resolve => setTimeout(resolve, delay));
  }
}
```

## Go Implementation

### Basic Polling

```go
package main

import (
    "context"
    "fmt"
    "time"
    "github.com/agent-tech/agent-sdk-go"
)

func pollStatus(ctx context.Context, client *pay.Client, intentID string, maxAttempts int) (*pay.Intent, error) {
    for i := 0; i < maxAttempts; i++ {
        intent, err := client.GetIntent(ctx, intentID)
        if err != nil {
            return nil, err
        }

        // Terminal states
        switch intent.Status {
        case pay.StatusBaseSettled:
            return intent, nil
        case pay.StatusExpired, pay.StatusVerificationFailed:
            return intent, fmt.Errorf("payment failed: %s", intent.Status)
        }

        // Wait before next poll
        time.Sleep(3 * time.Second)
    }

    return nil, fmt.Errorf("polling timeout - maximum attempts reached")
}
```

### Exponential Backoff

```go
func pollWithBackoff(ctx context.Context, client *pay.Client, intentID string) (*pay.Intent, error) {
    delay := 2 * time.Second
    maxDelay := 30 * time.Second
    maxDuration := 10 * time.Minute
    startTime := time.Now()

    for {
        // Check timeout
        if time.Since(startTime) > maxDuration {
            return nil, fmt.Errorf("polling timeout - exceeded maximum duration")
        }

        intent, err := client.GetIntent(ctx, intentID)
        if err != nil {
            return nil, err
        }

        // Terminal states
        switch intent.Status {
        case pay.StatusBaseSettled:
            return intent, nil
        case pay.StatusExpired, pay.StatusVerificationFailed:
            return intent, fmt.Errorf("payment failed: %s", intent.Status)
        }

        // Exponential backoff
        time.Sleep(delay)
        delay = time.Duration(float64(delay) * 1.5)
        if delay > maxDelay {
            delay = maxDelay
        }
    }
}
```

## Error Handling

### Handle Rate Limiting

```typescript
async function pollWithRateLimitHandling(intentId: string) {
  let delay = 2000;
  const maxDelay = 30000;
  
  while (true) {
    try {
      const intent = await client.getIntent(intentId);
      
      if (intent.status === 'BASE_SETTLED') {
        return { success: true, intent };
      }
      
      if (intent.status === 'EXPIRED' || intent.status === 'VERIFICATION_FAILED') {
        return { success: false, intent };
      }
      
      // Reset delay on success
      delay = 2000;
      await new Promise(resolve => setTimeout(resolve, delay));
      
    } catch (error) {
      if (error instanceof RequestError && error.statusCode === 429) {
        // Rate limited - increase delay
        delay = Math.min(delay * 2, maxDelay);
        console.log(`Rate limited, waiting ${delay}ms`);
        await new Promise(resolve => setTimeout(resolve, delay));
      } else {
        throw error;
      }
    }
  }
}
```

### Retry on Network Errors

```typescript
async function pollWithRetry(intentId: string, maxRetries: number = 3) {
  let delay = 2000;
  
  while (true) {
    let retries = 0;
    let intent;
    
    // Retry logic for network errors
    while (retries < maxRetries) {
      try {
        intent = await client.getIntent(intentId);
        break; // Success
      } catch (error) {
        retries++;
        if (retries >= maxRetries) {
          throw error;
        }
        // Wait before retry
        await new Promise(resolve => setTimeout(resolve, 1000 * retries));
      }
    }
    
    // Check terminal states
    if (intent.status === 'BASE_SETTLED') {
      return { success: true, intent };
    }
    
    if (intent.status === 'EXPIRED' || intent.status === 'VERIFICATION_FAILED') {
      return { success: false, intent };
    }
    
    await new Promise(resolve => setTimeout(resolve, delay));
    delay = Math.min(delay * 1.5, 30000);
  }
}
```

## Best Practices

### 1. Polling Intervals

- **Initial**: 2-3 seconds for quick feedback
- **Processing**: 3-5 seconds during normal processing
- **Final Settlement**: 2-3 seconds when close to completion
- **Maximum**: 30 seconds to avoid excessive delays

### 2. Timeout Handling

- Set maximum polling duration (e.g., 10 minutes = intent expiration time)
- Stop polling immediately when terminal state is reached
- Provide user feedback on timeout

### 3. Rate Limiting

- Respect API rate limits (60 req/min/IP)
- Use exponential backoff to reduce API calls
- Handle HTTP 429 responses gracefully

### 4. User Experience

- Show loading indicators during polling
- Update UI with current status
- Provide clear error messages
- Allow user to cancel long-running polls

### 5. Error Recovery

- Retry on transient errors (network, 503)
- Handle rate limiting (429) with backoff
- Fail fast on terminal errors (404, 400)

## Complete Example

```typescript
import { PayClient, RequestError } from '@agent-tech/pay';

async function waitForPaymentCompletion(intentId: string): Promise<{
  success: boolean;
  intent: any;
  transactionHash?: string;
}> {
  const client = new PayClient({
    baseURL: 'https://api-pay.agent.tech',
    apiKey: 'your-api-key',
    secretKey: 'your-secret-key',
  });

  let delay = 2000;
  const maxDelay = 30000;
  const maxDuration = 600000; // 10 minutes
  const startTime = Date.now();

  while (true) {
    // Check timeout
    if (Date.now() - startTime > maxDuration) {
      throw new Error('Payment polling timeout');
    }

    try {
      const intent = await client.getIntent(intentId);

      // Terminal states
      if (intent.status === 'BASE_SETTLED') {
        return {
          success: true,
          intent,
          transactionHash: intent.base_payment?.transaction_hash,
        };
      }

      if (intent.status === 'EXPIRED' || intent.status === 'VERIFICATION_FAILED') {
        return {
          success: false,
          intent,
        };
      }

      // Reset delay on success
      delay = 2000;
      
    } catch (error) {
      if (error instanceof RequestError) {
        if (error.statusCode === 429) {
          // Rate limited - increase delay
          delay = Math.min(delay * 2, maxDelay);
          console.log(`Rate limited, waiting ${delay}ms`);
        } else if (error.statusCode === 404) {
          throw new Error('Intent not found');
        } else {
          throw error;
        }
      } else {
        throw error;
      }
    }

    // Wait before next poll
    await new Promise(resolve => setTimeout(resolve, delay));
  }
}
```

## Related Links

- [Query Intent Status](query-intent-status.md)
- [Create Intent](create-intent.md)
- [Execute Intent](execute-intent.md)
- [Error Handling](error-handling.md)
- [API Documentation: Statuses](../api/statuses.md)
