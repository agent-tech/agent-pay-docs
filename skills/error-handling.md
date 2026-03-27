# Error Handling and Retry Mechanisms

## Overview

- **Function**: Comprehensive error handling and retry strategies for AgentTech SDK operations
- **Use Cases**: Handle API errors, network failures, rate limiting, implement robust retry logic
- **Authentication**: Applies to all operations (with or without authentication)

## Error Types

### 1. RequestError

Returned for HTTP 4xx/5xx responses from the API.

**TypeScript/JavaScript:**
```typescript
import { PayApiError } from '@cross402/usdc';

try {
  const intent = await client.createIntent({...});
} catch (error) {
  if (error instanceof PayApiError) {
    console.log(`HTTP ${error.statusCode}: ${error.body}`);

    // Handle specific status codes
    switch (error.statusCode) {
      case 400:
        console.error('Bad request:', error.body);
        break;
      case 401:
        console.error('Unauthorized - check credentials');
        break;
      case 429:
        console.error('Rate limited');
        break;
    }
  }
}
```

**Go:**
```go
import "github.com/cross402/usdc-sdk-go"

resp, err := client.CreateIntent(ctx, &pay.CreateIntentRequest{...})
var reqErr *pay.RequestError
if errors.As(err, &reqErr) {
    log.Printf("HTTP %d: %s", reqErr.StatusCode, reqErr.Body)
    
    switch reqErr.StatusCode {
    case 400:
        log.Println("Bad request:", reqErr.Body)
    case 401:
        log.Println("Unauthorized - check credentials")
    case 429:
        log.Println("Rate limited")
    }
}
```

### 2. ValidationError

Returned when the SDK rejects a request before it reaches the API (e.g., empty intent ID).

**Go:**
```go
var valErr *pay.ValidationError
if errors.As(err, &valErr) {
    log.Printf("Invalid input: %s", valErr.Message)
    
    // Check for specific validation errors
    if errors.Is(err, pay.ErrEmptyIntentID) {
        log.Println("Intent ID was empty")
    }
}
```

### 3. UnexpectedError

Wraps unexpected internal errors (JSON marshal failure, request creation, etc.).

**Go:**
```go
var unexpErr *pay.UnexpectedError
if errors.As(err, &unexpErr) {
    log.Printf("Unexpected error: %v", unexpErr.Err)
}
```

## HTTP Status Codes

| Status Code | Meaning | Retryable | Action |
|-------------|---------|-----------|--------|
| `400` | Bad request — invalid parameters, amount out of range, or malformed input | No | Fix request parameters |
| `401` | Unauthorized — missing or invalid credentials | No | Check API key and secret key |
| `403` | Forbidden — insufficient permissions | No | Check API key permissions |
| `404` | Not found — intent does not exist | No | Verify intent ID |
| `429` | Rate limited — too many requests (60 req/min/IP typical) | Yes | Implement exponential backoff |
| `503` | Service unavailable — temporary backend issue | Yes | Retry after delay |

## Retry Strategies

### 1. Simple Retry with Fixed Delay

```typescript
async function retryWithFixedDelay<T>(
  fn: () => Promise<T>,
  maxRetries: number = 3,
  delayMs: number = 1000
): Promise<T> {
  for (let i = 0; i < maxRetries; i++) {
    try {
      return await fn();
    } catch (error) {
      if (i === maxRetries - 1) throw error;
      
      // Only retry on retryable errors
      if (error instanceof PayApiError) {
        if (error.statusCode === 429 || error.statusCode === 503) {
          await new Promise(resolve => setTimeout(resolve, delayMs));
          continue;
        }
      }
      throw error;
    }
  }
  throw new Error('Max retries exceeded');
}
```

### 2. Exponential Backoff

```typescript
async function retryWithExponentialBackoff<T>(
  fn: () => Promise<T>,
  maxRetries: number = 5,
  initialDelayMs: number = 1000
): Promise<T> {
  let delay = initialDelayMs;

  for (let i = 0; i < maxRetries; i++) {
    try {
      return await fn();
    } catch (error) {
      if (i === maxRetries - 1) throw error;

      if (error instanceof PayApiError) {
        // Only retry on retryable errors
        if (error.statusCode === 429 || error.statusCode === 503) {
          console.log(`Retry ${i + 1}/${maxRetries} after ${delay}ms`);
          await new Promise(resolve => setTimeout(resolve, delay));
          delay = Math.min(delay * 2, 30000); // Max 30 seconds
          continue;
        }
      }
      throw error;
    }
  }
  throw new Error('Max retries exceeded');
}
```

### 3. Exponential Backoff with Jitter

```typescript
function getDelayWithJitter(baseDelay: number): number {
  // Add random jitter (±25%)
  const jitter = baseDelay * 0.25 * (Math.random() * 2 - 1);
  return Math.max(0, baseDelay + jitter);
}

async function retryWithJitter<T>(
  fn: () => Promise<T>,
  maxRetries: number = 5,
  initialDelayMs: number = 1000
): Promise<T> {
  let delay = initialDelayMs;

  for (let i = 0; i < maxRetries; i++) {
    try {
      return await fn();
    } catch (error) {
      if (i === maxRetries - 1) throw error;

      if (error instanceof PayApiError) {
        if (error.statusCode === 429 || error.statusCode === 503) {
          const jitteredDelay = getDelayWithJitter(delay);
          console.log(`Retry ${i + 1}/${maxRetries} after ${jitteredDelay}ms`);
          await new Promise(resolve => setTimeout(resolve, jitteredDelay));
          delay = Math.min(delay * 2, 30000);
          continue;
        }
      }
      throw error;
    }
  }
  throw new Error('Max retries exceeded');
}
```

## Go Retry Implementation

### Exponential Backoff

```go
package main

import (
    "context"
    "errors"
    "fmt"
    "math"
    "time"
    "github.com/cross402/usdc-sdk-go"
)

func retryWithBackoff(ctx context.Context, maxRetries int, fn func() error) error {
    delay := time.Second
    maxDelay := 30 * time.Second
    
    for i := 0; i < maxRetries; i++ {
        err := fn()
        if err == nil {
            return nil
        }
        
        if i == maxRetries-1 {
            return err
        }
        
        // Check if error is retryable
        var reqErr *pay.RequestError
        if errors.As(err, &reqErr) {
            if reqErr.StatusCode == 429 || reqErr.StatusCode == 503 {
                fmt.Printf("Retry %d/%d after %v\n", i+1, maxRetries, delay)
                time.Sleep(delay)
                delay = time.Duration(math.Min(float64(delay*2), float64(maxDelay)))
                continue
            }
        }
        
        return err
    }
    
    return fmt.Errorf("max retries exceeded")
}
```

## Rate Limiting Handling

### Handle HTTP 429

```typescript
async function handleRateLimit<T>(fn: () => Promise<T>): Promise<T> {
  let delay = 1000;
  const maxDelay = 60000; // 1 minute
  let retries = 0;
  const maxRetries = 10;
  
  while (retries < maxRetries) {
    try {
      return await fn();
    } catch (error) {
      if (error instanceof PayApiError && error.statusCode === 429) {
        retries++;
        console.log(`Rate limited, waiting ${delay}ms (retry ${retries}/${maxRetries})`);
        await new Promise(resolve => setTimeout(resolve, delay));
        delay = Math.min(delay * 2, maxDelay);
        continue;
      }
      throw error;
    }
  }
  
  throw new Error('Rate limit retries exceeded');
}
```

## Complete Error Handling Example

### TypeScript/JavaScript

```typescript
import { PayClient, PayApiError, PayValidationError } from '@cross402/usdc';

async function createIntentWithRetry(
  client: PayClient,
  params: {
    email: string;
    amount: string;
    payerChain: string;
  }
): Promise<any> {
  const maxRetries = 5;
  let delay = 1000;
  const maxDelay = 30000;
  
  for (let attempt = 0; attempt < maxRetries; attempt++) {
    try {
      return await client.createIntent(params);
    } catch (error) {
      // Don't retry on validation errors
      if (error instanceof PayValidationError) {
        throw error;
      }
      
      // Don't retry on client errors (4xx except 429)
      if (error instanceof PayApiError) {
        if (error.statusCode === 400 ||
            error.statusCode === 401 ||
            error.statusCode === 403 ||
            error.statusCode === 404) {
          throw error;
        }
        
        // Retry on rate limit and server errors
        if (error.statusCode === 429 || error.statusCode === 503) {
          if (attempt === maxRetries - 1) {
            throw error;
          }
          
          console.log(`Retry ${attempt + 1}/${maxRetries} after ${delay}ms`);
          await new Promise(resolve => setTimeout(resolve, delay));
          delay = Math.min(delay * 2, maxDelay);
          continue;
        }
      }
      
      // Retry on network errors
      if (attempt === maxRetries - 1) {
        throw error;
      }
      
      console.log(`Retry ${attempt + 1}/${maxRetries} after ${delay}ms`);
      await new Promise(resolve => setTimeout(resolve, delay));
      delay = Math.min(delay * 2, maxDelay);
    }
  }
  
  throw new Error('Max retries exceeded');
}
```

### Go

```go
package main

import (
    "context"
    "errors"
    "log"
    "math"
    "time"
    "github.com/cross402/usdc-sdk-go"
)

func createIntentWithRetry(
    ctx context.Context,
    client *pay.Client,
    req *pay.CreateIntentRequest,
) (*pay.CreateIntentResponse, error) {
    maxRetries := 5
    delay := time.Second
    maxDelay := 30 * time.Second
    
    for attempt := 0; attempt < maxRetries; attempt++ {
        resp, err := client.CreateIntent(ctx, req)
        if err == nil {
            return resp, nil
        }
        
        // Don't retry on validation errors
        var valErr *pay.ValidationError
        if errors.As(err, &valErr) {
            return nil, err
        }
        
        // Don't retry on client errors (4xx except 429)
        var reqErr *pay.RequestError
        if errors.As(err, &reqErr) {
            if reqErr.StatusCode == 400 ||
               reqErr.StatusCode == 401 ||
               reqErr.StatusCode == 403 ||
               reqErr.StatusCode == 404 {
                return nil, err
            }
            
            // Retry on rate limit and server errors
            if reqErr.StatusCode == 429 || reqErr.StatusCode == 503 {
                if attempt == maxRetries-1 {
                    return nil, err
                }
                
                log.Printf("Retry %d/%d after %v", attempt+1, maxRetries, delay)
                time.Sleep(delay)
                delay = time.Duration(math.Min(float64(delay*2), float64(maxDelay)))
                continue
            }
        }
        
        // Retry on other errors
        if attempt == maxRetries-1 {
            return nil, err
        }
        
        log.Printf("Retry %d/%d after %v", attempt+1, maxRetries, delay)
        time.Sleep(delay)
        delay = time.Duration(math.Min(float64(delay*2), float64(maxDelay)))
    }
    
    return nil, errors.New("max retries exceeded")
}
```

## Best Practices

### 1. Retry Only Transient Errors

- **Retry**: 429 (rate limit), 503 (service unavailable), network errors
- **Don't Retry**: 400 (bad request), 401 (unauthorized), 403 (forbidden), 404 (not found)

### 2. Use Exponential Backoff

- Start with short delays (1-2 seconds)
- Double delay on each retry
- Cap maximum delay (e.g., 30 seconds)

### 3. Add Jitter

- Prevents thundering herd problem
- Randomize delay slightly (±25%)

### 4. Set Maximum Retries

- Avoid infinite retry loops
- Typical: 3-5 retries for transient errors
- More retries for critical operations

### 5. Log Retry Attempts

- Track retry attempts for debugging
- Monitor retry rates for system health

### 6. Handle Timeouts

- Set request timeouts
- Set overall operation timeouts
- Fail fast on timeout

## Sentinel Errors

Use `errors.Is` (Go) to check for specific validation failures:

| Sentinel | Meaning |
|----------|---------|
| `ErrEmptyBaseURL` | baseURL was empty in NewClient |
| `ErrEmptyIntentID` | intentID was empty |
| `ErrEmptySettleProof` | settleProof was empty in SubmitProof |
| `ErrMissingAuth` | ExecuteIntent called without auth |
| `ErrNilParams` | params argument was nil |

**Go Example:**
```go
if errors.Is(err, pay.ErrEmptyIntentID) {
    log.Println("Intent ID was empty")
}
```

## Related Links

- [Create Intent](create-intent.md)
- [Execute Intent](execute-intent.md)
- [Submit Proof](submit-proof.md)
- [Query Intent Status](query-intent-status.md)
- [Payment Polling](payment-polling.md)
- [API Documentation: Error Codes](../support/error-codes.md)
