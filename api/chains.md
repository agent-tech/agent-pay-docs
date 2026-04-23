# ⛓ Supported Chains

AgentPay supports paying from one chain and settling on another. A payment intent declares two chains:

- `payer_chain` — where the payer sends USDC.
- `target_chain` — where the merchant receives USDC. Optional; defaults to `base`.

The set of target chains available to your integration is served by `GET /api/chains`. If a chain appears in that response, you can use it as a `target_chain`. Any chain supported as a payer can also be used as a target, subject to the status of each chain (some are live today; others are documented for roadmap visibility but not yet callable — see the **Status** column below).

## Status legend

- **Live** — callable today. You can pass the identifier as `payer_chain` or `target_chain` right now and the call will succeed (assuming the usual validation).
- **🚧 Coming soon** — the identifier, SDK constant, decimals, and signing flavor are documented so you can prepare integration code, but the chain is not yet enabled. `CreateIntent` calls with a coming-soon chain return `400 invalid payer_chain` / `400 invalid target_chain` today. Treat `GET /api/chains` as authoritative for what's callable right now.

## Payer chains

| Chain | Identifier | Go Constant | TS Constant | USDC Decimals | Status | Notes |
| :--- | :--- | :--- | :--- | :--- | :--- | :--- |
| Solana | `solana` | `pay.ChainSolanaMainnet` | `Chain.SolanaMainnet` | 6 | Live | — |
| Base | `base` | `pay.ChainBase` | `Chain.Base` | 6 | Live | — |
| Ethereum | `ethereum` | `pay.ChainEthereum` | `Chain.Ethereum` | 6 | Live | — |
| Polygon | `polygon` | `pay.ChainPolygon` | `Chain.Polygon` | 6 | Live | — |
| HyperEVM | `hyperevm` | `pay.ChainHyperEVM` | `Chain.HyperEvm` | 6 | Live | — |
| Arbitrum | `arbitrum` | `pay.ChainArbitrum` | `Chain.Arbitrum` | 6 | 🚧 Coming soon | — |
| BSC | `bsc` | `pay.ChainBSC` | `Chain.BSC` | **18** | 🚧 Coming soon | Binance-Peg USDC; Permit2 + EIP-2612 signing. See [BSC Signing](bsc-signing.md). |
| Monad | `monad` | `pay.ChainMonad` | `Chain.Monad` | 6 | 🚧 Coming soon | Permit2 + EIP-2612 signing. |
| SKALE Base | `skale-base` | `pay.ChainSkaleBase` | `Chain.SkaleBase` | 6 | 🚧 Coming soon | EIP-712 domain name is `"Bridged USDC (SKALE Bridge)"` (not `"USD Coin"`). |
| MegaETH | `megaeth` | `pay.ChainMegaETH` | `Chain.MegaEth` | **18** | 🚧 Coming soon | Native USDm (MegaUSD), EIP-712 domain `name="MegaUSD"`, `version="1"`; Permit2 + EIP-2612 signing. |

## Target chains

Every **Live** payer chain is also a valid target. Coming-soon chains will become valid targets when they go live. Two caveats apply:

- `GET /api/chains` is authoritative. It lists the chains currently accepted as a `target_chain`; coming-soon chains are omitted until enabled.
- Target chains are validated at intent creation. If you pass an unsupported value (including a coming-soon chain), the API returns `400` with an error that lists the currently accepted target chains.

```
GET /api/chains
```

Response (today):

```json
{
  "chains": ["base", "ethereum", "hyperevm", "polygon", "solana"]
}
```

The list is stable per deployment and expands only as chains move from coming-soon to live.

## Payer × target matrix

Any payer chain can pair with any target chain exposed by your deployment, including same-chain routes (e.g. `base → base`). The table below covers common combinations; rows flagged `🚧 Coming soon` are documented for roadmap visibility and will return `400` until the chain goes live.

| Payer → Target | CCTP burn/mint | Direct EVM transfer | Solana (SVM) transfer | Status |
| :--- | :---: | :---: | :---: | :--- |
| `solana → base` | ✓ | | | Live |
| `solana → ethereum` | ✓ | | | Live |
| `base → ethereum` | | ✓ | | Live |
| `base → polygon` | | ✓ | | Live |
| `polygon → base` | | ✓ | | Live |
| `base → solana` | | | ✓ | Live |
| `base → base` (same chain) | | ✓ | | Live |
| `polygon → arbitrum` | | ✓ | | 🚧 Coming soon |
| `bsc → base` | | ✓ | | 🚧 Coming soon |

Callers do not pick the settlement mode; AgentPay chooses between CCTP burn/mint, direct EVM transfer, and SVM transfer based on the (payer, target) pair. See [Multi-Chain Settlement](../docs/concepts/multi-chain-settlement.md) for the mechanics.

## Chain-specific caveats

These are the details you need at signing and display time.

### USDC decimals

Always read `extra.decimals` from the `payment_requirements` object on the `CreateIntent` response. Do not hardcode `6`.

| Chain | Decimals | Source | Status |
| :--- | :---: | :--- | :--- |
| Solana, Base, Ethereum, Polygon, HyperEVM | 6 | Circle USDC / Bridged USDC | Live |
| Arbitrum, Monad, SKALE Base | 6 | Circle USDC / Bridged USDC | 🚧 Coming soon |
| BSC | 18 | Binance-Peg USDC | 🚧 Coming soon |
| MegaETH | 18 | Native USDm (MegaUSD) | 🚧 Coming soon |

### EIP-712 domain names

When signing EIP-3009 `TransferWithAuthorization` or Permit2, use the `extra.name` and `extra.version` values from the intent response. The defaults are `"USD Coin"` / `"2"`, but two chains are exceptions:

- **SKALE Base**: `name = "Bridged USDC (SKALE Bridge)"`, `version = "2"`.
- **MegaETH**: `name = "MegaUSD"`, `version = "1"`.

Hardcoding the default `"USD Coin"` on either chain makes every signature fail verification.

### Signing flavor

- EIP-3009 `TransferWithAuthorization`: Base, Ethereum, Polygon, HyperEVM (Live); Arbitrum, SKALE Base (🚧 Coming soon).
- Permit2 `PermitWitnessTransferFrom` + EIP-2612 `Permit` (gas-sponsored by AgentPay): BSC, Monad, MegaETH (all 🚧 Coming soon).
- Solana partial-signed VersionedTransaction v0: Solana (Live).

The `payment_requirements.extra.assetTransferMethod` field is set to `"permit2"` when Permit2 is required; otherwise it is absent (EIP-3009 path) or uses the Solana payload shape.

## Usage

Specify chains with SDK constants, not hardcoded strings. `targetChain` is optional.

```typescript
import { Chain } from '@cross402/usdc';

const intent = await client.createIntent({
  email: "merchant@example.com",
  amount: "100.50",
  payerChain: Chain.Base,
  targetChain: Chain.Ethereum,
});
```

```go
resp, err := client.CreateIntent(ctx, &pay.CreateIntentRequest{
    Email:       "merchant@example.com",
    Amount:      "100.50",
    PayerChain:  pay.ChainBase,
    TargetChain: pay.ChainEthereum,
})
```
