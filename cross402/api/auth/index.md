---
order: 4
---

# Authentication

> Source: https://docs.agent.tech/api/auth/

# Authentication

AgentTech uses API keys and Bearer tokens to secure requests.

## Getting Your API Key

To obtain your API key and secret key, follow these steps:

1. **Visit the Dashboard**: Go to <https://agent.tech/dashboard>
2. **Sign In**: Log in with your email address
3. **Create an Agent**: Click on "Create Agent" below the login area
4. **Generate API Key**: After creating an agent, you can generate your API key and secret key

**Important Notes:**

* Each account can create up to **10 agents** maximum
* Each agent will have its own unique API key and secret key
* Keep your secret key secure and never expose it in client-side code

## Authentication Modes

### Authenticated Mode (v2 API)

Requires Bearer token authentication. Enables Agent-led execution where the backend signs transactions automatically.

**Use Case**: Server-side operations, automated payouts, backend services.

#### Go SDK

```
client, err := pay.NewClient(baseURL,
    pay.WithBearerAuth(apiKey, secretKey),
)
```

#### JavaScript/TypeScript SDK

```
const client = new PayClient({
  baseUrl: 'https://api-pay.agent.tech',
  auth: { apiKey: 'your-api-key', secretKey: 'your-secret-key' },
});
```

**Bearer Token Format**: Base64-encode your `apiKey:secretKey` string.

### Public Mode (api API)

No authentication required. Safe for frontend use.

**Use Case**: Client-side intent creation, user-initiated payments.

#### Go SDK

```
client, err := pay.NewClient(baseURL)
// No auth required - automatically uses /api endpoints
```

#### JavaScript/TypeScript SDK

```
const client = new PublicPayClient({
  baseUrl: 'https://api-pay.agent.tech',
});
```

## Endpoint Routing

The SDK automatically selects the API prefix based on authentication:

* **With auth** (`WithBearerAuth`) → `/v2` prefix — create intent → execute (backend signs)
* **Without auth** → `/api` prefix — create intent → payer signs → submit settle\_proof

## Verifying Credentials (`GET /v2/me`)

Once you have an API key, call `GET /v2/me` (auth required) to confirm it works and to discover the agent's funded wallet addresses without hitting the dashboard:

```
const me = await client.getMe(); // JS/TS SDK
// me.agentId, me.agentNumber, me.name, me.status,
// me.walletAddress (EVM), me.solanaWalletAddress
```

```
me, err := client.GetMe(ctx) // Go SDK
// me.AgentID, me.AgentNumber, me.Name, me.Status,
// me.WalletAddress (EVM), me.SolanaWalletAddress
```

The handler reads from middleware context (no DB hit), so it is cheap to call as a startup health-check. Treat any 401 as a hard configuration error — never retry.

## Verifying Credentials (`GET /v2/me`)

Once you have an API key, call `GET /v2/me` (auth required) to confirm it works and to discover the agent's funded wallet addresses without hitting the dashboard:

```typescript
const me = await client.getMe(); // JS/TS SDK
// me.agentId, me.agentNumber, me.name, me.status,
// me.walletAddress (EVM), me.solanaWalletAddress
```

```go
me, err := client.GetMe(ctx) // Go SDK
// me.AgentID, me.AgentNumber, me.Name, me.Status,
// me.WalletAddress (EVM), me.SolanaWalletAddress
```

The handler reads from middleware context (no DB hit), so it is cheap to call as a startup health-check. Treat any 401 as a hard configuration error — never retry.

## Security Best Practices

##### danger "Never expose your API key or secret key"

Store credentials in environment variables, not in source code or config files.

```
export AGENTTECH_API_KEY="your-api-key"
export AGENTTECH_SECRET_KEY="your-secret-key"
```

```
apiKey := os.Getenv("AGENTTECH_API_KEY")
secretKey := os.Getenv("AGENTTECH_SECRET_KEY")
```

**Do not paste your API key or secret key into any AI assistant, chat, or support channel.** If your key is compromised, an attacker can initiate payments from your agent wallet, resulting in permanent fund loss.

1. **Never expose secrets in frontend code** — use `PublicPayClient` for browser environments
2. **Store API keys in environment variables** — never hardcode credentials in source code or commit them to version control
3. **Rotate keys regularly** — update credentials periodically for better security
4. **Use HTTPS only** — ensure all API calls are made over encrypted connections
