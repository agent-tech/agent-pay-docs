# JavaScript / TypeScript SDK

A universal SDK for Node.js (v18+) and modern browsers. Two clients: **PayClient** (server-side, with secrets) and **PublicPayClient** (client-side, no secrets).

## Install

```bash
npm install @cross402/usdc
```

## PayClient (Server-Side)

Use on the backend when the Agent wallet should sign and execute the transfer.

```ts
import { PayClient } from "@cross402/usdc/server";

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
import { PublicPayClient } from "@cross402/usdc/client";

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
npm install -g @cross402/usdc

# Configure credentials
cross402-usdc auth set --api-key <key> --secret-key <key> --base-url https://api-pay.agent.tech
cross402-usdc auth show          # Display current config (secret key masked)
cross402-usdc auth clear         # Remove stored config

# Server flow: create → execute → get
cross402-usdc intent create --amount 10.00 --payer-chain solana --email merchant@example.com
cross402-usdc intent execute <intent-id>
cross402-usdc intent get <intent-id>

# Client flow (no auth): submit proof only
cross402-usdc intent submit-proof <intent-id> --proof <settle-proof>

# Session & balance management
cross402-usdc intent sessions [--expired]   # List stored intent sessions
cross402-usdc balance read --address <addr> [--rpc-url <url>]  # Read USDC balance on Base
cross402-usdc reset [--yes]                 # Remove all stored config + sessions
```

## Skills

Install the cross402-usdc CLI skill for Cursor (skills.sh) or Clawhub:

```bash
npx skills add cross402/usdc-sdk-js-ts
# or
clawhub skills add cross402/usdc-sdk-js-ts
```

> [View on GitHub](https://github.com/cross402/usdc-sdk-js-ts)
