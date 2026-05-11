---
order: 4
---

# Pure API (no SDK)

Call AgentTech's payment API directly over HTTP. Use this chapter when you cannot install one of the official SDKs — different language runtime (Rust, Ruby, Elixir, shell scripts), browser environments where you control the fetch layer, or when you need fine-grained control over headers, timeouts, and retries.

The wire protocol is identical to what the JS/TS and Go SDKs already speak. Anything you can do through `PayClient` you can do with `curl`.

## When to use Pure API

* **No SDK for your language** — anything outside Node/TS and Go.
* **Embedded / constrained runtimes** — shell scripts, CI, k6 load tests, internal admin tools.
* **Custom transport** — you already have an HTTP client with auth, tracing, and retry plumbing and you do not want a second one from the SDK.
* **Auditability** — you want to log the exact request/response bytes that crossed the wire.

If none of the above apply, prefer the [JS/TS SDK](../js-ts/) or the [Go SDK](../go/). The SDK does the same calls plus client-side validation, typed errors, status constants, and Permit2 / X402 typed-data signing.

## Base URL

```
https://api-pay.agent.tech
```

All examples below use that host. The API prefix is selected by the auth mode:

* **Authenticated (API key)** → `/v2` — backend signs with the agent wallet on `ExecuteIntent`.
* **Public (no auth, payer signs)** → `/api` — payer signs an X402 payment off-chain and submits a settle proof.

---

## Authentication

There are exactly **two** flows: API key (for the v2 agent-led path) and private-key signing (for the public payer-signs path). Pick one — they are not stacked.

### Flow 1 — API Key (`/v2`)

Used when you want the **AgentTech backend to sign and settle on behalf of your agent wallet**. Suitable for automated payouts, agent-driven purchases, and any server-side caller that already holds an API key.

#### Provision a key

1. Visit <https://agent.tech/dashboard>, sign in, and create an agent (max 10 per account).
2. From the agent's page, generate an API key. You receive a `client_id` and a `client_secret`. The secret is shown **once** — store it in an environment variable, never in source.

```
export AGENTTECH_CLIENT_ID="ag_live_..."
export AGENTTECH_CLIENT_SECRET="sk_live_..."
```

#### Send the key

The middleware accepts either header style. Pick one per request.

**Style A — Bearer (recommended, matches the SDKs):**

```
Authorization: Bearer <base64(client_id:client_secret)>
```

**Style B — split headers:**

```
X-Client-Id: <client_id>
X-Api-Key: <client_secret>
```

Both are equivalent. The Bearer form is what `PayClient` / `pay.WithBearerAuth` produce.

#### Verify the key (`GET /v2/me`)

The cheapest call you can make. Returns the agent's identity straight from the middleware context — no DB hit — so it is safe to run on every cold start as a configuration check.

```
TOKEN=$(printf '%s:%s' "$AGENTTECH_CLIENT_ID" "$AGENTTECH_CLIENT_SECRET" | base64)

curl -sS https://api-pay.agent.tech/v2/me \
  -H "Authorization: Bearer $TOKEN"
```

Response:

```json
{
  "agent_id": "8b2e9c4a-3f7a-4d1b-9e2c-5a6b7c8d9e0f",
  "agent_number": "A-000123",
  "name": "checkout-agent",
  "status": "active",
  "wallet_address": "0x742d35Cc...",
  "solana_wallet_address": "Es9vMFrz..."
}
```

`wallet_address` and `solana_wallet_address` are omitted when the agent has no wallet for that chain family. A `401` here is a **hard configuration error** — do not retry, do not back off; fix the key.

##### danger "Never paste your secret"

Do not paste `client_secret` into AI assistants, support channels, screenshots, or commit messages. A leaked secret lets an attacker drain the agent wallet via `POST /v2/intents/{id}/execute`. If it leaks, rotate immediately from the dashboard.

### Flow 2 — Private-key signing (`/api`)

Used when the **payer holds their own wallet and signs the payment off-chain**. No API key required. The flow is identical to what `PublicPayClient` (JS/TS) and the unauthenticated `pay.NewClient` (Go) do internally.

The payer signs a typed message that authorizes the stablecoin transfer; the signature is the `settle_proof` you POST back to AgentTech, which then submits it to the facilitator on-chain.

#### What you sign

Two signing standards are used depending on the payer chain. The `payment_requirements` object on the `CreateIntent` response tells you which one to use.

| Payer chain | Standard | Signed payload |
| --- | --- | --- |
| Base, Ethereum, Polygon, Arbitrum, HyperEVM, Monad, SKALE Base | **EIP-3009** `transferWithAuthorization` (EIP-712 typed data) | `from`, `to`, `value`, `validAfter`, `validBefore`, `nonce` against the USDC contract's EIP-712 domain |
| BSC, MegaETH | **Permit2** `PermitTransferFrom` (EIP-712 typed data) | `permitted{token, amount}`, `spender`, `nonce`, `deadline` against the canonical Permit2 contract `0x000000000022D473030F116dDEE9F6B43aC78BA3` |
| Solana | Solana transaction signature | Standard Solana transfer signed by the payer keypair |

The exact `domain`, `types`, and `message` shape is returned to you in `payment_requirements.extra` on `CreateIntent` — you do **not** hardcode the EIP-712 domain. See [BSC Signing](../../api/bsc-signing/) for the Permit2 path step-by-step (the one-time on-chain approval is BSC-specific).

#### Three-step flow

1. **Create** the intent → backend returns `payment_requirements` (typed-data spec the payer must sign).
2. **Sign** off-chain with the payer's private key (no gas).
3. **Submit** the signature as `settle_proof` to `POST /api/intents/{intent_id}`.

A worked end-to-end example follows in [Quickstart](#quickstart-flow) below.

##### danger "Never expose your private key"

Sign in the **payer's environment** — browser wallet, hardware wallet, or backend KMS. Never send the raw key over the network, never log it, and never paste it into AI assistants. AgentTech only ever sees the resulting signature.

---

## Quickstart flow

Goal: pay 100.50 USDC from a Base wallet, settle to a merchant on Ethereum.

### Path A — API key (`/v2`, backend signs)

#### Step 1. Create the intent

```
TOKEN=$(printf '%s:%s' "$AGENTTECH_CLIENT_ID" "$AGENTTECH_CLIENT_SECRET" | base64)

curl -sS -X POST https://api-pay.agent.tech/v2/intents \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "email": "merchant@example.com",
    "amount": "100.50",
    "payer_chain": "base",
    "target_chain": "ethereum"
  }'
```

Expected response (`201 Created`):

```json
{
  "intent_id": "int_abc123xyz",
  "agent_id": "8b2e9c4a-3f7a-4d1b-9e2c-5a6b7c8d9e0f",
  "merchant_recipient": "0x742d35Cc...",
  "sending_amount": "100.50",
  "receiving_amount": "98.65",
  "estimated_fee": "1.85",
  "fee_breakdown": {
    "source_chain": "base",
    "source_chain_fee": "0.05",
    "target_chain": "ethereum",
    "target_chain_fee": "1.30",
    "platform_fee": "0.50",
    "platform_fee_percentage": "0.5",
    "total_fee": "1.85"
  },
  "payer_chain": "base",
  "target_chain": "ethereum",
  "status": "AWAITING_PAYMENT",
  "created_at": "2024-01-01T12:00:00Z",
  "expires_at": "2024-01-01T12:10:00Z",
  "payment_requirements": { "...": "x402 payment requirements" }
}
```

Intents expire **10 minutes** after creation. If you do not call execute / submit-proof inside that window, the intent moves to `EXPIRED` (terminal).

#### Step 2. Preflight (optional but recommended)

There is no dedicated `preflight` endpoint. The cheap preflight check is `GET /v2/me` (covered in [Authentication](#flow-1--api-key-v2)) plus inspecting the `estimated_fee` and `fee_breakdown` returned by `CreateIntent` before you call execute. If the fee exceeds your budget you can simply discard the intent and let it expire — `CreateIntent` itself does not move funds.

To check supported chains before constructing the request:

```
curl -sS https://api-pay.agent.tech/api/chains
```

Response:

```json
{
  "chains":        ["base", "ethereum", "solana", "bsc", "polygon", "arbitrum", "hyperevm", "monad", "skale-base", "megaeth"],
  "target_chains": ["base", "ethereum", "solana", "bsc", "polygon", "arbitrum", "hyperevm", "monad"]
}
```

`chains` is valid as `payer_chain`; `target_chains` is valid as `target_chain`. Use this list rather than hardcoding — payer-only chains (SKALE Base, MegaETH) are absent from `target_chains` on purpose.

#### Step 3. Execute (settle)

```
curl -sS -X POST https://api-pay.agent.tech/v2/intents/int_abc123xyz/execute \
  -H "Authorization: Bearer $TOKEN"
```

Response (`200 OK`):

```json
{
  "intent_id": "int_abc123xyz",
  "agent_id": "8b2e9c4a-3f7a-4d1b-9e2c-5a6b7c8d9e0f",
  "merchant_recipient": "0x742d35Cc...",
  "sending_amount": "100.50",
  "receiving_amount": "98.65",
  "estimated_fee": "1.85",
  "fee_breakdown": { "...": "same shape as create" },
  "status": "TARGET_SETTLED",
  "created_at": "2024-01-01T12:00:00Z",
  "expires_at": "2024-01-01T12:10:00Z"
}
```

`POST /v2/intents/{intent_id}/execute` enforces ownership: an intent owned by another agent — or one created via the unauthenticated `/api` flow — returns `404 payment intent not found`, not `403`. This is intentional: collapsing both rejection paths prevents agents from probing intent IDs across the system.

#### Step 4. Poll for receipt

```
curl -sS "https://api-pay.agent.tech/v2/intents?intent_id=int_abc123xyz" \
  -H "Authorization: Bearer $TOKEN"
```

Poll until `status` reaches a terminal state: `TARGET_SETTLED` (success) or `EXPIRED` / `VERIFICATION_FAILED` / `PARTIAL_SETTLEMENT` (terminal failure). The full state machine is in [Statuses](../../concepts/statuses/).

```json
{
  "intent_id": "int_abc123xyz",
  "agent_id": "8b2e9c4a-3f7a-4d1b-9e2c-5a6b7c8d9e0f",
  "status": "TARGET_SETTLED",
  "payer_chain": "base",
  "target_chain": "ethereum",
  "merchant_recipient": "0x742d35Cc...",
  "target_payment": {
    "tx_hash": "0x1234...abcd",
    "settle_proof": "...",
    "settled_at": "2024-01-01T12:05:00Z",
    "explorer_url": "https://etherscan.io/tx/0x1234...abcd"
  },
  "source_payment": {
    "chain": "base",
    "tx_hash": "0x9876...fedc",
    "settle_proof": "...",
    "settled_at": "2024-01-01T12:04:30Z",
    "explorer_url": "https://basescan.org/tx/0x9876...fedc"
  }
}
```

`agent_id`, `sending_amount`, `receiving_amount`, `estimated_fee`, and `fee_breakdown` are only populated once the intent leaves its initial state — treat them as optional when polling. `target_payment` appears only at `TARGET_SETTLED`.

### Path B — Private-key signing (`/api`, payer signs)

#### Step 1. Create the intent

```
curl -sS -X POST https://api-pay.agent.tech/api/intents \
  -H "Content-Type: application/json" \
  -d '{
    "recipient": "0x742d35Cc...",
    "amount": "10.00",
    "payer_chain": "base",
    "target_chain": "ethereum"
  }'
```

No `Authorization` header. The response is the same shape as Path A, with one difference: `agent_id` is **absent** (intent is not owned by any agent), and `payment_requirements.extra` contains the typed-data spec the payer must sign.

```json
{
  "intent_id": "int_pub789",
  "merchant_recipient": "0x742d35Cc...",
  "sending_amount": "10.00",
  "receiving_amount": "9.45",
  "estimated_fee": "0.55",
  "payer_chain": "base",
  "target_chain": "ethereum",
  "status": "AWAITING_PAYMENT",
  "created_at": "2024-01-01T12:00:00Z",
  "expires_at": "2024-01-01T12:10:00Z",
  "payment_requirements": {
    "scheme": "exact",
    "network": "base",
    "amount": "10000000",
    "payTo": "0xAgent...",
    "maxTimeoutSeconds": 600,
    "asset": "0x833589fCD6eDb6E08f4c7C32D4f71b54bdA02913",
    "extra": {
      "name": "USD Coin",
      "version": "2",
      "decimals": 6
    }
  }
}
```

The `extra` block is the EIP-712 domain you sign against (for BSC, `extra` exposes `nonce`, `deadline`, `spender` for Permit2 instead — see [BSC Signing](../../api/bsc-signing/) for the exact shape).

#### Step 2. Sign off-chain with the payer's private key

Sign the X402 `PaymentRequirements` typed data using the payer wallet. Most callers use a library (`viem`, `ethers`, `@solana/web3.js`); the resulting `settle_proof` is **`base64(JSON-encoded PaymentPayload)`** — a single string. The detailed Permit2 signing example is in [BSC Signing](../../api/bsc-signing/); EIP-3009 signing for Base/Ethereum/etc. follows the standard `transferWithAuthorization` flow using the `extra` domain returned above.

#### Step 3. Submit the settle proof

```
curl -sS -X POST https://api-pay.agent.tech/api/intents/int_pub789 \
  -H "Content-Type: application/json" \
  -d '{
    "settle_proof": "<base64-encoded-payment-payload>"
  }'
```

Response (`200 OK`):

```json
{
  "intent_id": "int_pub789",
  "merchant_recipient": "0x742d35Cc...",
  "sending_amount": "10.00",
  "receiving_amount": "9.45",
  "estimated_fee": "0.55",
  "status": "SOURCE_SETTLED",
  "created_at": "2024-01-01T12:00:00Z",
  "expires_at": "2024-01-01T12:10:00Z"
}
```

`SOURCE_SETTLED` means the payer's signature verified and the source-chain transfer landed; the backend now drives the target-chain leg. Poll `GET /api/intents?intent_id=...` (no auth) until `TARGET_SETTLED` or a terminal failure.

> `POST /api/intents/{id}` (submit proof) has **no `/v2` equivalent**. If you call it with an `Authorization: Bearer` header expecting it to be agent-scoped, the route is simply the public one and ownership is **not** checked — that is by design. v2 callers should use `POST /v2/intents/{id}/execute` instead.

---

## Reference

All endpoints accept and return `application/json`. Field names are `snake_case`.

| Method | Path | Auth | Purpose |
| --- | --- | --- | --- |
| `GET`  | `/api/chains` | None | List supported `payer_chain` and `target_chain` values |
| `POST` | `/api/intents` | None | Create a public intent (payer will sign) |
| `POST` | `/api/intents/{intent_id}` | None | Submit `settle_proof` for a public intent |
| `GET`  | `/api/intents?intent_id=...` | None | Read a public intent's status / receipt |
| `POST` | `/v2/intents` | API key | Create an agent-owned intent |
| `POST` | `/v2/intents/{intent_id}/execute` | API key | Settle an agent-owned intent (backend signs) |
| `GET`  | `/v2/intents?intent_id=...` | API key | Read an agent-owned intent (404 if not owned) |
| `GET`  | `/v2/intents/list?page=&page_size=` | API key | Paginated list of the calling agent's intents |
| `GET`  | `/v2/me` | API key | Identity + wallet addresses of the calling agent |

Field-level schemas for request and response bodies are documented in the [API Reference / Intents](../../api/intents/) page — they are identical to the shapes used by the SDKs (the SDKs are thin typed wrappers).

### Pagination on `/v2/intents/list`

| Parameter | Default | Behaviour |
| --- | --- | --- |
| `page` | `1` | 1-indexed. Out-of-range / non-numeric → `400`. |
| `page_size` | `20` | `[1, 100]` honored; values above `100` are clamped to `100`; negative / non-numeric → `400`. |

---

## Errors

All errors return a JSON body with `error`, `message`, and `status_code`:

```json
{
  "error": "Unauthorized",
  "message": "invalid api key",
  "status_code": 401
}
```

### HTTP status codes

| Status | Meaning |
| --- | --- |
| `400` | Bad request — invalid params, amount out of range, malformed input, or out-of-range pagination |
| `401` | Unauthorized — missing / invalid / revoked API key. Hard config error — do not retry |
| `402` | Payment required — **caller's** agent wallet has insufficient stablecoin on the payer chain (`/v2/execute` only) |
| `403` | Forbidden — reserved for future use; v2 ownership rejections return `404`, not `403` |
| `404` | Not found — intent does not exist **or** is owned by a different agent (v2 collapses both to `404` to prevent enumeration) |
| `429` | Rate limited — 60 req/IP/min typical. Back off exponentially |
| `503` | Service unavailable — backend wallet temporarily out of funds, or transient backend issue. Safe to retry with backoff |

### Domain error messages

These appear in `message` on `400` / `402` / `503` responses and are stable strings. Match on these from your code rather than parsing free-form text.

| Message | When | Action |
| --- | --- | --- |
| `invalid payer_chain: must be solana, base, or bsc` | `payer_chain` not in supported set | Fix request; call `GET /api/chains` to discover live set |
| `invalid target_chain: must be one of solana, base, bsc, polygon, arbitrum, ethereum, monad, hyperevm` | `target_chain` not in supported set | Fix request |
| `invalid email format` | Email regex failed | Fix request |
| `invalid recipient address format` | Wallet address fails chain-specific validation (EVM vs Solana) | Fix request |
| `invalid amount: must be a positive number with up to 6 decimal places` | Amount out of `[0.02, 1_000_000]` or > 6 decimal precision | Fix request |
| `either email or recipient is required` | Both fields empty on create | Fix request |
| `cannot provide both email and recipient` | Both fields set | Pick one |
| `payment intent has expired` | Intent older than 10 minutes | Create a new intent |
| `invalid intent status for this operation` | E.g. execute on already-settled intent | Re-read status before retrying |
| `proof validation failed` | `settle_proof` did not verify | Re-sign; check chain ID, domain, nonce |
| `concurrent update detected: intent status has changed` | Two callers raced | Re-read status; do not blindly retry |
| `intent does not belong to this agent` | API key mismatches intent's owner | Use the correct key; do not retry |
| `agent wallet has insufficient balance on payer chain` | `402` on `/v2/execute` | Top up the agent wallet (read its address from `/v2/me`) |
| `service temporarily unavailable: payment wallet has insufficient balance` | `503` on `/api` flow — global proxy wallet low | Retry with backoff |
| `payment intent not found` | `404` — intent does not exist **or** is owned by another agent | Do not retry; verify intent ID |

### Status strings (terminal vs in-flight)

The `status` field on intent responses takes one of these values. Terminal states do not transition further.

| Value | Terminal | Meaning |
| --- | --- | --- |
| `AWAITING_PAYMENT` | No | Created; waiting for execute or submit-proof |
| `PENDING` | No | Backend is processing |
| `SOURCE_SETTLED` | No | Payer-chain transfer landed; driving target chain |
| `TARGET_SETTLING` | No | Target-chain transfer in flight |
| `TARGET_SETTLED` | Yes | Success — `target_payment` is populated |
| `VERIFICATION_FAILED` | Yes | Proof did not verify on-chain |
| `PARTIAL_SETTLEMENT` | Yes | Source landed but target failed; contact support |
| `EXPIRED` | Yes | Not executed within 10 minutes |

When `status` is `SOURCE_SETTLED`, the `error_message` field (if present) is surfaced verbatim from the backend — it has operational value for diagnosing why settlement stalled (e.g. `"insufficient USDC balance: have 0.10, need 0.19"`). For other failure states, `error_message` is mapped to a generic, secret-free string.

---

## Comparison vs SDK

Same task, side by side. Both code paths hit the exact same endpoints.

| Task | SDK (JS/TS) | Pure API (curl) |
| --- | --- | --- |
| Verify key | `await client.getMe()` | `curl -H "Authorization: Bearer $TOKEN" /v2/me` |
| Create intent | `await client.createIntent({...})` | `curl -X POST /v2/intents -d '{...}'` |
| Execute (backend signs) | `await client.executeIntent(id)` | `curl -X POST /v2/intents/{id}/execute` |
| Submit proof (payer signs) | `await publicClient.submitProof(id, proof)` | `curl -X POST /api/intents/{id} -d '{"settle_proof":...}'` |
| Poll status | `await client.getIntent(id)` | `curl /v2/intents?intent_id={id}` |
| List intents | `await client.listIntents({page, pageSize})` | `curl /v2/intents/list?page=1&page_size=20` |
| Discover chains | `await client.listSupportedChains()` | `curl /api/chains` |

### What the SDK gives you that curl does not

* **Client-side validation** — empty intent ID, missing auth on `execute`, malformed pagination are caught before the HTTP call (saves a round-trip and a `400`).
* **Typed errors / sentinels** — `pay.ErrEmptyIntentID`, `PayApiError`, `PayValidationError` so you can `errors.Is` / `instanceof` instead of parsing strings.
* **Status constants** — `IntentStatus.TargetSettled` / `pay.StatusTargetSettled` instead of magic strings.
* **Permit2 typed-data signing** — `client.ApprovePermit2()` and the Permit2 EIP-712 builder are bundled (you do not have to depend on `viem` / `ethers`).
* **Smart routing** — passing or omitting `WithBearerAuth` automatically picks `/v2` vs `/api`; pure-API callers must choose the path themselves.

If you only need a single payment in a one-off script, the curl route is fine. For anything production-shaped (retries, observability, multi-chain payer signing), use an SDK.
