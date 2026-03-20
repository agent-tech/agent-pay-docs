# JavaScript / TypeScript SDK

A universal SDK for Node.js (v18+) and modern browsers. Two clients: **PayClient** (server-side, with secrets) and **PublicPayClient** (client-side, no secrets).

## Install

```bash
npm install @agenttech/pay
```

## PayClient (Server-Side)

Use on the backend when the Agent wallet should sign and execute the transfer.

```ts
import { PayClient } from "@agenttech/pay/server";

const client = new PayClient({
  baseUrl: "https://api-pay.agent.tech",
  auth: { apiKey: "your-api-key", secretKey: "your-secret-key" },
});

// Create intent → execute on Base → check status
const { intentId } = await client.createIntent({
  email: "merchant@example.com",
  amount: "100.50",
  payerChain: "solana",
});
await client.executeIntent(intentId);
const intent = await client.getIntent(intentId);
console.log(intent.status);
```

## PublicPayClient (Client-Side)

Use in the browser or payer-side when the user holds their own wallet and will sign an X402 payment and submit the settle proof.

```ts
import { PublicPayClient } from "@agenttech/pay/client";

const client = new PublicPayClient({
  baseUrl: "https://api-pay.agent.tech",
});

const { intentId } = await client.createIntent({
  recipient: "0x...",
  amount: "10.00",
  payerChain: "base",
});
// ... payer signs X402 payment off-chain ...
await client.submitProof(intentId, settleProof);
const intent = await client.getIntent(intentId);
```

## CLI

```bash
npm install -g @agenttech/pay
agent-pay auth set --api-key <key> --secret-key <key> --base-url https://api-pay.agent.tech

# Server flow: create → execute → get
agent-pay intent create --amount 10.00 --payer-chain solana --email merchant@example.com
agent-pay intent execute <intent-id>
agent-pay intent get <intent-id>

# Client flow (no auth): submit proof only
agent-pay intent submit-proof <intent-id> --proof <settle-proof>
```

## Skills

Install the agent-pay CLI skill for Cursor (skills.sh) or Clawhub:

```bash
npx skills add agent-tech/AgentPay-SDK-JS-TS
# or
clawhub skills add agent-tech/AgentPay-SDK-JS-TS
```

> 🔗 [View on GitHub](https://github.com/agent-tech/AgentPay-SDK-JS-TS)
