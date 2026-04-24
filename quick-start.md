# 🚀 Quick Start

Integrate cross402 into your application in three simple steps.

### 1. Install the SDK
Choose your preferred environment:
* **Node.js**: `npm install @cross402/usdc`
* **Go**: `go get github.com/cross402/usdc-sdk-go`

### 2. Get Your API Keys

Before initializing the client, you need to obtain your API key and secret key:

1. Visit [https://agent.tech/dashboard](https://agent.tech/dashboard)
2. Sign in with your email
3. Click "Create Agent" to create a new agent
4. Generate your API key and secret key from the agent dashboard

> **Note**: Each account can create up to 10 agents. For detailed instructions, see the [Authentication documentation](api/auth.md#getting-your-api-key).

!!! danger "Protect your keys"
Always store your API key and secret key in **environment variables**. Never hardcode them in source code or share them in any chat, AI assistant, or support channel. A leaked key can be used to initiate payments from your wallet, resulting in permanent fund loss.
!!!

### 3. Initialize the Client

#### Node.js / TypeScript
```typescript
import { PayClient } from '@cross402/usdc';

const client = new PayClient({
  baseUrl: 'https://api-pay.agent.tech',
  auth: { apiKey: 'your-api-key', secretKey: 'your-secret-key' },
});
```

#### Go
```go
import "github.com/cross402/usdc-sdk-go"

client, err := pay.NewClient(
    "https://api-pay.agent.tech",
    pay.WithBearerAuth(apiKey, secretKey),
)
```

---

### 4. Create an Intent
An "Intent" represents a payment request. You pick where the payer sends USDC (`payerChain`) and where the merchant receives it (`targetChain`, optional — defaults to `"base"`). In this example the payer sends on Base and the merchant is paid on Ethereum.
```typescript
const intent = await client.createIntent({
  email: "merchant@example.com",
  amount: "100.50",
  payerChain: "base",
  targetChain: "ethereum",
});
```

> `targetChain` is optional. Omit it to keep v1 behaviour (settle on Base). If you pass it, the value must be a chain listed by `GET /api/chains`; otherwise the request is rejected with `400 invalid target_chain`. Full matrix and per-chain caveats in [Supported Chains](api/chains.md).

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

Poll `client.getIntent(intentId)` until `status === "TARGET_SETTLED"` (success) or one of the terminal failure states (`EXPIRED`, `VERIFICATION_FAILED`, `PARTIAL_SETTLEMENT`). The full state machine is in [Statuses](api/statuses.md).
---
