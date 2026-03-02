# JavaScript / TypeScript SDK

A universal SDK supporting Node.js (v18+) and modern browser environments.

### Dual-Client Architecture
To balance security and flexibility, we provide two distinct clients:

#### 1. PayClient (Server-Side)
Requires `secretKey`. Designed for secure backend environments.
* **Use Case**: Automated payouts, internal rebalancing, and administrative tasks.

#### 2. PublicPayClient (Client-Side)
Requires no secrets. Safe for use in browsers.
* **Use Case**: Generating checkout intents and submitting settlement proofs from user wallets.

> 🔗 [View on GitHub](https://github.com/agent-tech/AgentPay-SDK-JS-TS)