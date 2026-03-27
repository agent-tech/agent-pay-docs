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
import { PayClient } from '@agenttech/pay';

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

## Best Practices

1. **Polling Strategy**: Use exponential backoff when polling for intent status
2. **Error Handling**: Always check for terminal states (`EXPIRED`, `VERIFICATION_FAILED`) before retrying
3. **Rate Limiting**: Respect the 60 req/min/IP limit and implement client-side throttling if needed
4. **Idempotency**: Intent creation is idempotent — safe to retry on network errors
