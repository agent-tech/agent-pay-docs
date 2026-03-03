# 🚀 Quick Start

Integrate AgentPay into your application in three simple steps.

### 1. Install the SDK
Choose your preferred environment:
* **Node.js**: `npm install @agent-tech/pay`
* **Go**: `go get github.com/agent-tech/agent-sdk-go`

### 2. Get Your API Keys

Before initializing the client, you need to obtain your API key and secret key:

1. Visit [https://agent.tech/dashboard](https://agent.tech/dashboard)
2. Sign in with your email
3. Click "Create Agent" to create a new agent
4. Generate your API key and secret key from the agent dashboard

> **Note**: Each account can create up to 10 agents. For detailed instructions, see the [Authentication documentation](api/auth.md#getting-your-api-key).

### 3. Initialize the Client

#### Node.js / TypeScript
```typescript
import { PayClient } from '@agent-tech/pay';

const client = new PayClient({
  baseURL: 'https://api-pay.agent.tech',
  apiKey: 'your-api-key',
  secretKey: 'your-secret-key',
});
```

#### Go
```go
import "github.com/agent-tech/agent-sdk-go"

client, err := pay.NewClient(
    "https://api-pay.agent.tech",
    pay.WithBearerAuth(apiKey, secretKey),
)
```

---

### 4. Create an Intent
An "Intent" represents a payment request.
```typescript
const intent = await client.createIntent({
  email: "merchant@example.com",
  amount: "100.50",
  payerChain: "base"
});
```
---

### 5. Finalize Payment

Depending on your integration type, choose one of the following methods to complete the transaction:

#### Method A: Server-side (Automated)
Ideal for automated payouts or background tasks where the Agent wallet signs on behalf of the system.
* **Action**: Call the `execute` method.
* **Code**: `await client.executeIntent(intentId);`

#### Method B: Client-side (User-initiated)
Ideal for e-commerce checkouts where the actual user (payer) must sign with their own wallet (e.g., MetaMask).
1. **Sign**: The payer signs the **X402 payment** off-chain via their wallet.
2. **Submit**: Use the SDK to submit the settlement proof to our API.
* **Code**: `await client.submitProof(intentId, settleProof);`
---