# 🔐 Authentication

AgentPay uses API keys and Bearer tokens to secure requests.

## Authentication Modes

### Authenticated Mode (v2 API)

Requires Bearer token authentication. Enables Agent-led execution where the backend signs transactions automatically.

**Use Case**: Server-side operations, automated payouts, backend services.

#### Go SDK

```go
client, err := pay.NewClient(baseURL,
    pay.WithBearerAuth(apiKey, secretKey),
)
```

#### JavaScript/TypeScript SDK

```typescript
const client = new PayClient({
  baseURL: 'https://api-pay.agent.tech',
  apiKey: 'your-api-key',
  secretKey: 'your-secret-key',
});
```

**Bearer Token Format**: Base64-encode your `apiKey:secretKey` string.

### Public Mode (api API)

No authentication required. Safe for frontend use.

**Use Case**: Client-side intent creation, user-initiated payments.

#### Go SDK

```go
client, err := pay.NewClient(baseURL)
// No auth required - automatically uses /api endpoints
```

#### JavaScript/TypeScript SDK

```typescript
const client = new PublicPayClient({
  baseURL: 'https://api-pay.agent.tech',
});
```

## Endpoint Routing

The SDK automatically selects the API prefix based on authentication:

* **With auth** (`WithBearerAuth`) → `/v2` prefix — create intent → execute (backend signs)
* **Without auth** → `/api` prefix — create intent → payer signs → submit settle_proof

## Security Best Practices

1. **Never expose secrets in frontend code** — use `PublicPayClient` for browser environments
2. **Store API keys securely** — use environment variables or secret management services
3. **Rotate keys regularly** — update credentials periodically for better security
4. **Use HTTPS only** — ensure all API calls are made over encrypted connections