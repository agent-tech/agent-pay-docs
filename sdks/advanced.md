# 🔧 Advanced Configuration

Advanced configuration options for customizing SDK behavior.

## Custom HTTP Client

### Go SDK

Use a custom HTTP client with retry logic:

```go
client, err := pay.NewClient(baseURL,
    pay.WithBearerAuth(apiKey, secretKey),
    pay.WithHTTPClient(&http.Client{
        Timeout:   60 * time.Second,
        Transport: &retryTransport{base: http.DefaultTransport},
    }),
)
```

### JavaScript/TypeScript SDK

```typescript
import { PayClient } from '@cross402/usdc';

const client = new PayClient({
  baseUrl: 'https://api-pay.agent.tech',
  auth: { apiKey: 'your-api-key', secretKey: 'your-secret-key' },
  fetcher: customFetcher, // Your custom fetch implementation
});
```

## Retry Logic

Implement exponential backoff for rate-limited requests (HTTP 429):

```go
var reqErr *pay.RequestError
if errors.As(err, &reqErr) && reqErr.StatusCode == 429 {
    backoff := time.Duration(attempt) * time.Second
    time.Sleep(backoff)
    // retry request
}
```

## Timeout Configuration

Set appropriate timeouts based on your use case:

* **Intent Creation**: Typically fast (< 1s)
* **Intent Execution**: May take 10-30s depending on chain confirmation
* **Status Polling**: Use shorter intervals (2-5s) for better UX

## Chain-Native Amount Handling

USDC uses different decimal precision on different chains. Do **not** hardcode `6` decimals. The backend response includes an `extra.decimals` field that reflects the actual decimal precision for the source chain.

| Chain | USDC Decimals |
| :--- | :--- |
| Base, Solana, Polygon, Arbitrum, Ethereum | 6 |
| **BSC** | **18** |

```go
// CORRECT: use extra.decimals from the response
decimals := int64(resp.Extra.Decimals)
multiplier := new(big.Int).Exp(big.NewInt(10), big.NewInt(decimals), nil)
chainAmount := new(big.Int).Mul(parsedAmount, multiplier)

// WRONG: hardcoding 6 decimals breaks BSC payments
chainAmount := new(big.Int).Mul(parsedAmount, big.NewInt(1_000_000))
```

### BSC: Permit2 Pre-Approval

BSC requires a one-time Permit2 contract approval before the first USDC transfer. This step is **BSC-only** — other chains (Polygon, Arbitrum, Ethereum) do not use Permit2 and do not require this step.

Call the pre-approval once per wallet before initiating any BSC payment:

```go
// BSC only — skip for all other chains
if req.PayerChain == pay.ChainBSC {
    if err := client.ApprovePermit2(ctx, walletKey); err != nil {
        log.Fatal("Permit2 pre-approval failed:", err)
    }
}
```

## Best Practices

1. **Polling Strategy**: Use exponential backoff when polling for intent status
2. **Error Handling**: Always check for terminal states (`EXPIRED`, `VERIFICATION_FAILED`) before retrying
3. **Rate Limiting**: Respect the 60 req/min/IP limit and implement client-side throttling if needed
4. **Idempotency**: Intent creation is idempotent — safe to retry on network errors
5. **BSC decimals**: Never hardcode USDC decimal precision — read `extra.decimals` from the backend response
