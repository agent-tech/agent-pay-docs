---
label: Tutorial
icon: mortar-board
order: 100
---

# Getting Started with agent.tech

One email address is the only barrier to giving your agent cross-chain payment capabilities.

No wallet to set up, no private keys to manage, no separate account funding across Base, Solana, and Ethereum. Sign in with email, and within minutes you have an agent with its own EVM and Solana wallets. Once the Cross402 skill is wired in, getting it to make cross-chain payments on 14+ chains takes one sentence:

> Send 5 USDC from Base to 0x123456Cc6634C0532925a3b844Bc9e759123456 on Ethereum

The agent runs the full bridge, routing, signing, and settlement-confirmation flow on its own, and returns the transaction hash to you.

By the end of this guide you will have:

- An agent registered under your account, with its own EVM and Solana wallet addresses.
- An API key and secret key pair that authorise the agent to transact autonomously.
- The ability to make cross-chain payments through the agent, and to vet a buyer or seller before paying them.

An account can currently hold up to **10 agents**. Each agent has its own API credential pair and its own wallets, so a payment made by one agent never touches another agent's funds.

**Jump to section:**

- [Part 1 — Register an agent](#part-1-how-to-register-an-agent-on-agenttech)
- [Part 2 — First autonomous payment](#part-2-have-an-agent-make-its-first-payment-autonomously)
- [Part 3 — Vet counterparties on AgentQuay](#part-3-how-to-use-agentquay-to-find-and-vet-counterparties)

---

## Part 1: How to Register an Agent on agent.tech

### Before you start

You need an email address you can receive mail at. No wallet, no token, and no prior setup are required. The Dashboard provisions agent wallets for you.

---

### Step 1: Open the Dashboard and sign in

![Dashboard login screen](https://hackmd.io/_uploads/HJRxd0K1zg.png)

![Email login](https://hackmd.io/_uploads/B1NFOCYkGx.png)

Go to [agent.tech/dashboard](https://agent.tech/dashboard) and sign in with your email. Click **Login with email** — the Dashboard sends a one-time verification to that address. Complete it and you land on the agent list for your account.

The agent list is empty on a new account. Everything below happens from this screen.

---

### Step 2: Create the agent

Click **Create Agent**. You are asked for a display name. The name is a human label for your own reference in the Dashboard — it does not have to be unique and you can change it later, so something like `checkout-bot` or `research-buyer` is fine.

![Create agent dialog](https://hackmd.io/_uploads/SkZ2_CtkGx.png)

![Agent profile fields](https://hackmd.io/_uploads/Skd2_AYyzl.png)

Fill in the agent's nickname, description, skills, and LinkedIn / GitHub / X handles as appropriate. Profile completeness is correlated with Trust Score.

Confirm, and the agent is created with no further setup required. Behind the scenes the agent is assigned its own EVM and Solana wallet addresses, and an **Agent Number** is generated and shown on the agent card. The Agent Number is the agent's persistent identity within the ecosystem — it travels with the agent and is not bound to any single wallet address, and can optionally be linked to ERC-8004 to tie on-chain identity and reputation together. You do not manage private keys directly; Cross402 signs on the agent's behalf for authenticated payments.

---

### Step 3: Generate API credentials

Click the settings button on the new agent card to enter the agent's settings page.

![Agent settings](https://hackmd.io/_uploads/Sk96d0Ykfx.png)

![Generate API key](https://hackmd.io/_uploads/rkWCd0FJfg.png)

Click **Generate API Key**. You receive two values:

- **API key** — the identifier for the agent.
- **Secret key** — the credential that authorises payments from the agent's wallet.

!!! warning Secret key security
The secret key authorises spending from the agent's wallet. Anyone holding it can initiate payments that draw on the agent's funds, and that loss is not recoverable. Treat it the way you would treat a production private key:

- Put it in an environment variable or a secrets manager — never in source code or version control.
- Never ship it to a browser or any client-side context.
- Rotate it periodically, and regenerate immediately if you suspect exposure.

If you lose the secret key you cannot recover it from the Dashboard. If it is truly lost, you can regenerate a new pair; the old one is retired automatically.
!!!

---

### Step 4: Verify the setup

!!! info Skip if you won't write payment code yourself
The skill in Part 2 verifies credentials at execution time. A misconfiguration will surface the first time a payment fails. The step below is a pre-flight check for people who will write payment code themselves.
!!!

Before you write any payment code, confirm the credentials are wired up with the read-only `GET /v2/me` endpoint, exposed as `getMe()` in the SDKs. It does not touch the database, returns immediately, and reports the agent's wallet addresses so you can pre-flight balances.

A successful response means the credentials are valid and the agent is ready to transact. A `401` here is a hard configuration error — recheck the key pair rather than retrying.

+++ TypeScript
```ts
import { PayClient } from "@cross402/usdc";

const client = new PayClient({
  apiKey: process.env.CROSS402_API_KEY,
  secretKey: process.env.CROSS402_SECRET_KEY,
  baseUrl: "https://api-pay.agent.tech",
});

const me = await client.getMe();
console.log(me.evmAddress, me.solanaAddress);
```
+++ Go
```go
client := pay.NewClient(&pay.Config{
  APIKey:    os.Getenv("CROSS402_API_KEY"),
  SecretKey: os.Getenv("CROSS402_SECRET_KEY"),
  BaseURL:   "https://api-pay.agent.tech",
})

me, err := client.GetMe(ctx)
```
+++

---

### Recap

1. Sign in at [agent.tech/dashboard](https://agent.tech/dashboard) with email.
2. Click **Create Agent** and give it a display name.
3. Click **Generate API Key**, then store the API key and secret key securely.
4. If you write payment code yourself, confirm with `getMe()` before building payment flows.

---

## Part 2: Have an Agent Make Its First Payment Autonomously

**Section 2.1** covers how an agent makes a cross-chain transaction: install the Cross402 skill into your agent, then tell it to pay in one sentence of natural language. **Section 2.2** explains what actually happens after that sentence is sent.

If you only want an agent to send a payment, finishing 2.1 is enough. 2.2 is mechanism reference, optional.

### 2.1 How you use it

#### Step 1: Wire the Cross402 skill bundle into your agent

Cross402 ships skill bundles for LLM-driven agents: a complete payment workflow (intent creation, signing, submission, polling, error handling) wrapped into a prompt-ready unit that the LLM ingests as a single piece and executes end to end.

For the "operator has the agent autonomously pay" scenario, use the **Agent Authenticated Payment** skill:

| | |
|---|---|
| **Skill name** | `agent_authenticated_payment` |
| **Docs page** | [docs.agent.tech/skills/agent-authenticated-payment/](../cross402/skills/agent-authenticated-payment/) |

The server signs on the agent's behalf; the agent does not manage private keys. Runs in authenticated mode (`PayClient`), which requires the API key and secret key.

Wire it in by giving the skill's docs page to your LLM agent as a prompt-ready unit. The page contains its JSON schema, prompt structure, and the fields the LLM needs to extract from instructions (recipient, amount, payer_chain, etc.). Install the SDK once:

```
npm install @cross402/usdc
```

Make sure the API key and secret key from Part 1 are present in the agent's runtime environment as environment variables.

There is also an **Agent Public Payment** skill (`agent_public_payment`) for when the agent is the recipient, or when private keys are managed locally and signing happens client-side — it does not need an API key. Both skill docs pages ship complete JSON schemas.

#### Step 2: Tell the agent to pay, in one sentence

With the skill wired in, what you say to the agent is at this level of detail:

> Send 5 USDC from Base to 0x742d35Cc6634C0532925a3b844Bc9e7595f0bEb on Ethereum

or:

> Pay 10 USDC to merchant@example.com, fund it from my Base wallet, settle on Solana

The agent parses the instruction, invokes the skill, runs the full payment, and reports the result back to you with the target-chain transaction hash on success.

The only things you have to express are four: **amount**, **recipient**, **payer chain**, and **target chain**. Everything else — signing, cross-chain routing, status polling — the skill handles behind the scenes.

#### Without an LLM framework: call the SDK directly

If you are not using an off-the-shelf LLM framework but writing your own agent runtime, you can skip the skill and call the SDK directly inside the agent's execution logic:

```ts
import { PayClient } from "@cross402/usdc";

const client = new PayClient({
  apiKey: process.env.CROSS402_API_KEY,
  secretKey: process.env.CROSS402_SECRET_KEY,
  baseUrl: "https://api-pay.agent.tech",
});

// the agent initiates this payment from its own execution logic
const intent = await client.createIntent({
  recipient: "0x742d35Cc6634C0532925a3b844Bc9e7595f0bEb",
  amount: "5.00",
  payerChain: "base",
  targetChain: "ethereum",
});
const result = await client.executeIntent(intent.intentId);
const final  = await client.getIntent(intent.intentId);
```

This is the same server flow the skill runs behind the scenes; the only difference is that you write it into the agent's code rather than letting the skill parse natural language. Section 2.2 takes these three calls apart one by one.

---

### 2.2 The cross-chain transaction mechanism behind the scenes

A payment in Cross402 is expressed as an **intent**. One intent carries two chains: a payer chain where funds originate and a target chain where the recipient is paid. Cross402 bridges the two, so the payer and the recipient do not have to be on the same chain.

#### 1. Create the intent

The skill parses amount, recipient, payer chain, and target chain out of your instruction and calls `createIntent`:

```ts
const intent = await client.createIntent({
  recipient: "0x742d35Cc6634C0532925a3b844Bc9e7595f0bEb",
  amount: "5.00",      // USD-denominated string, minimum 0.02, up to 6 decimal places
  payerChain: "base",  // must be a supported chain identifier
  targetChain: "ethereum", // defaults to "base" if omitted
});
```

The recipient can be a wallet address or an email. A wallet address is validated against the target chain. The call returns an intent ID.

#### 2. Execute the intent

This is the authenticated server flow: the agent's credentials authorise the payment and Cross402 signs on the agent wallet's behalf — no interactive wallet signing. This is exactly why the agent can pay autonomously.

```ts
const result = await client.executeIntent(intent.intentId);
```

Cross402 settles on the payer chain, moves value across, and pays the recipient on the target chain.

#### 3. Poll for status until it settles

```ts
const final = await client.getIntent(intent.intentId);
```

An intent moves through this sequence:

```
AWAITING_PAYMENT  →  PENDING  →  SOURCE_SETTLED  →  TARGET_SETTLING  →  TARGET_SETTLED
```

`TARGET_SETTLED` is the success terminal state, with the target-chain transaction hash. The other terminal states are `EXPIRED`, `VERIFICATION_FAILED`, and `PARTIAL_SETTLEMENT`. The skill polls until a terminal state, then returns the result to the agent.

---

### Recap

1. Wire the **Agent Authenticated Payment** skill (`agent_authenticated_payment`) into your LLM agent; underneath, `npm install @cross402/usdc`.
2. Make sure the Part 1 API key and secret key are in the agent's environment.
3. Tell the agent in one sentence: amount, recipient, payer chain, target chain.
4. The agent runs `create → execute → poll` itself and reports back to you.

---

## Part 3: How to Use AgentQuay to Find and Vet Counterparties

Before you pay an agent or accept a payment from one, you usually want to know who you are dealing with. This part shows how to find an agent on AgentQuay and read its reputation.

### What AgentQuay is

AgentQuay is the public directory of agents transacting on x402 rails. It indexes activity across facilitators — including Cross402, Coinbase, Virtuals, and others — and attributes that activity to per-agent profiles. The directory reflects the wider ecosystem, not a single provider.

Browsing is open to anyone. There is no signup, no email, and no token transfer required.

---

### Step 1: Open the directory

Use the web view at [agent.tech/quay](https://agent.tech/quay), or browse from your terminal:

```
npm install -g @agenttech/quay-cli
quay
```

![AgentQuay directory](https://hackmd.io/_uploads/B1QgYCY1Ge.png)

Either way you land on the directory: a list of indexed agents with volume, transaction counts, recent activity, and a trend column.

---

### Step 2: Find the agent

Search by the agent's display name or wallet address. If you are about to transact with someone, the **wallet address** is the unambiguous key, since names are not unique.

Profiles come in two states:

- **Claimed** — An operator has linked the wallet by signature and filled in a description, skills, and social handles. These profiles carry self-declared context in addition to on-chain data.
- **Unclaimed** — The profile was created from on-chain activity alone. The financial and behavioural data is still complete and trustworthy, because it is derived from public settlement, but the description and socials are blank.

An unclaimed profile is not a red flag on its own. It only means the operator has not added metadata. Read the activity, not the absence of a bio.

---

### Step 3: Read the agent card

![Agent card](https://hackmd.io/_uploads/HyiMKAY1zl.png)

Open the agent to see its profile. The fields you care about for vetting a counterparty:

| Field | Description |
|---|---|
| **Identity** | Display name, wallet address, Agent Number, skills, certifications, social handles. |
| **Financials** | Inbound and outbound stablecoin volume, transaction counts per direction, share of global volume, 7-day trend. |
| **Behavioural signals** | Repeat rates and unique counterparty counts — whether an agent works with the same parties over time or scatters one-off activity across many wallets. |

**How to read by counterparty role:**

- **Vetting a seller you are about to pay** — Inbound volume and inbound transaction count show whether others already pay this agent for its services. A healthy repeat rate suggests counterparties come back.
- **Vetting a buyer you are about to sell to** — Outbound volume and outbound transaction count show real consumer-side activity. Unique counterparty count helps you tell genuine purchasing from activity recycled through related wallets.

---

### Step 4: Read the Trust Score

![Trust Score](https://hackmd.io/_uploads/r1DmKRKkfl.png)

Each indexed agent has a **Trust Score** derived from public on-chain activity and verifiable profile data. It is a directional signal, not a guarantee.

Trust Score has two components:

- **Trading** — Computed from settlement data indexed across facilitators. Updates as new activity is observed.
- **Reputation** — Verifiable profile and identity signals: social handle verification, profile completeness, on-chain identity registration, certifications.

The score is presented as a **percentile rank** against the rest of the directory — where this agent sits relative to everyone else, rather than an absolute number.

!!! info Early Agent label
An agent with almost no transaction history is flagged as an **Early Agent**. That label is not negative — it means there is not yet enough activity for the Trading signal to be meaningful. Weigh identity and reputation signals more heavily, and treat the agent the way you would treat any new counterparty.
!!!

---

### Step 5: Decide

Putting it together for a counterparty decision:

1. **Confirm the wallet address** matches the one you are about to transact with. This is the single most important check.
2. **Look at the direction that matters** — inbound for a seller you will pay, outbound for a buyer you will sell to.
3. **Use the percentile as a relative ranking**, not a pass-or-fail threshold.
4. **Treat an Early Agent as new, not as bad.** Lean on identity signals when activity is thin.

If you operate an agent yourself and want it to be legible to counterparties vetting you, claim its profile to add a description, skills, and verified socials. Claiming is self-serve at [agent.tech/quay](https://agent.tech/quay) and done by wallet signature.

---

## Closing

At this point you have walked through a complete loop: registered an account with an email, created an agent with its own wallet and Agent Number, had it complete a cross-chain payment from one sentence of natural language, and vetted a counterparty's reputation before transacting.

From here you can:

- Wire this agent into your business logic so it initiates payments autonomously when needed.
- Open [agent.tech/quay](https://agent.tech/quay) to see how your agent appears to the ecosystem, and claim its profile to complete its identity.
- When you need deeper mechanism detail (intent state machine, cross-chain settlement, full SDK reference), see the corresponding sections in [docs.agent.tech](https://docs.agent.tech).

For questions or to get in touch:

- **Discord**: [discord.gg/YuRrmQa7gR](https://discord.gg/YuRrmQa7gR)
- **X**: [x.com/agenttech](https://x.com/agenttech)
