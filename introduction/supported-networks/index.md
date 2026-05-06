---
order: 1
---

# Supported Chains

> Source: https://docs.agent.tech/api/chains/

# Supported Chains

Cross402 supports paying from one chain and settling on another. A payment intent declares two chains:

* `payer_chain` — where the payer sends USDC.
* `target_chain` — where the merchant receives USDC. Optional; defaults to `base`.

The set of target chains available to your integration is served by `GET /api/chains`. If a chain appears in that response, you can use it as a `target_chain`. Any chain supported as a payer can also be used as a target.

## Status legend

* **Live** — callable. You can pass the identifier as `payer_chain` or `target_chain` and the call will succeed (assuming the usual validation). Treat `GET /api/chains` as authoritative for the exact set exposed by your deployment.

## Payer chains

| Chain | Identifier | Go Constant | TS Constant | USDC Decimals | Status |
| --- | --- | --- | --- | --- | --- |
| Solana | `solana` | `pay.ChainSolanaMainnet` | `Chain.SolanaMainnet` | 6 | Live |
| Base | `base` | `pay.ChainBase` | `Chain.Base` | 6 | Live |
| Ethereum | `ethereum` | `pay.ChainEthereum` | `Chain.Ethereum` | 6 | Live |
| Polygon | `polygon` | `pay.ChainPolygon` | `Chain.Polygon` | 6 | Live |
| HyperEVM | `hyperevm` | `pay.ChainHyperEVM` | `Chain.HyperEvm` | 6 | Live |
| Arbitrum | `arbitrum` | `pay.ChainArbitrum` | `Chain.Arbitrum` | 6 | Live |
| BSC | `bsc` | `pay.ChainBSC` | `Chain.BSC` | **18** | Live |
| Monad | `monad` | `pay.ChainMonad` | `Chain.Monad` | 6 | Live |
| SKALE Base | `skale-base` | `pay.ChainSKALEBase` | `Chain.SkaleBase` | 6 | Live (payer-only) |
| MegaETH | `megaeth` | `pay.ChainMegaETH` | `Chain.MegaEth` | **18** | Live (payer-only) |

Per-chain notes for BSC, Monad, SKALE Base, and MegaETH appear at the end of this page under [Per-chain notes](#per-chain-notes).

## Target chains

Every payer chain is also a valid target. One caveat applies:

* `GET /api/chains` is authoritative. It lists the chains accepted as a `target_chain` for your deployment. Target chains are validated at intent creation; if you pass an unsupported value, the API returns `400` with an error that lists the currently accepted target chains.

```
GET /api/chains
```

Response:

```
{
  "chains": ["base", "ethereum", "hyperevm", "polygon", "solana", "skale-base", "megaeth"],
  "target_chains": ["base", "ethereum", "hyperevm", "polygon", "solana"]
}
```

The two lists are independent — `chains` enumerates valid `payer_chain` values and `target_chains` enumerates valid `target_chain` values. Payer-only chains (e.g. `skale-base`, `megaeth`) appear in `chains` but not in `target_chains`. The lists are stable per deployment.

## Payer × target matrix

Any payer chain can pair with any target chain exposed by your deployment, including same-chain routes (e.g. `base → base`). The table below covers common combinations. The **Payer sig** and **Target sig** columns show the x402 signing flavor used on each leg — the SDK picks these automatically from `payment_requirements`.

| Payer → Target | Payer sig | Target sig | Status |
| --- | --- | --- | --- |
| `solana → base` | Solana VT v0 | EIP-3009 | Live |
| `solana → ethereum` | Solana VT v0 | EIP-3009 | Live |
| `base → ethereum` | EIP-3009 | EIP-3009 | Live |
| `base → polygon` | EIP-3009 | EIP-3009 | Live |
| `polygon → base` | EIP-3009 | EIP-3009 | Live |
| `base → solana` | EIP-3009 | Solana VT v0 | Live |
| `base → base` (same chain) | EIP-3009 | EIP-3009 | Live |
| `polygon → arbitrum` | EIP-3009 | EIP-3009 | Live |
| `bsc → base` | Permit2 + EIP-2612 | EIP-3009 | Live |

Callers do not choose the signing flavor; the SDK derives it from `payment_requirements` and the `(payer, target)` pair. Both legs run through the x402 protocol. See [Multi-Chain Settlement](../../cross402/concepts/multi-chain-settlement/) for the mechanics.

## Chain-specific caveats

These are the details you need at signing and display time.

### USDC decimals + token contracts

Always read `extra.decimals` from the `payment_requirements` object on the `CreateIntent` response. Do not hardcode `6`. Mainnet token addresses are pinned by the backend per chain — copy from the table below or `GET /api/chains` (which echoes the same values).

| Chain | Decimals | Token | Mainnet Address | Status |
| --- | --- | --- | --- | --- |
| Solana | 6 | USDC | `EPjFWdd5AufqSSqeM2qN1xzybapC8G4wEGGkZwyTDt1v` | Live |
| Base | 6 | USDC | `0x833589fcd6edb6e08f4c7c32d4f71b54bda02913` | Live |
| Ethereum | 6 | USDC | `0xA0b86991c6218b36c1d19D4a2e9Eb0cE3606eB48` | Live |
| Polygon | 6 | USDC | `0x3c499c542cEF5E3811e1192ce70d8cC03d5c3359` | Live |
| HyperEVM | 6 | USDC | `0xb88339CB7199b77E23DB6E890353E22632Ba630f` | Live |
| Arbitrum | 6 | USDC | `0xaf88d065e77c8cC2239327C5EDb3A432268e5831` | Live |
| Monad | 6 | USDC | `0x754704Bc059F8C67012fEd69BC8A327a5aafb603` | Live |
| **SKALE Base** | 6 | **USDC.e** (Bridged USDC) | `0x85889c8c714505E0c94b30fcfcF64fE3Ac8FCb20` | Live (payer-only) |
| BSC | 18 | Binance-Peg USDC | `0x8AC76a51cc950d9822D68b83fE1Ad97B32Cd580d` | Live |
| **MegaETH** | 18 | **USDm** (MegaUSD, native) | `0xFAfDdbb3FC7688494971a79cc65DCa3EF82079E7` | Live (payer-only) |

### EIP-712 domain names

When signing EIP-3009 `TransferWithAuthorization` or Permit2, use the `extra.name` and `extra.version` values from the intent response. The defaults are `"USD Coin"` / `"2"`, but two chains are exceptions:

* **SKALE Base**: `name = "Bridged USDC (SKALE Bridge)"`, `version = "2"`.
* **MegaETH**: `name = "MegaUSD"`, `version = "1"`.

Hardcoding the default `"USD Coin"` on either chain makes every signature fail verification.

### Signing flavor

* EIP-3009 `TransferWithAuthorization`: Base, Ethereum, Polygon, HyperEVM, Arbitrum, SKALE Base.
* Permit2 `PermitWitnessTransferFrom` + EIP-2612 `Permit` (gas-sponsored by Cross402): BSC, Monad, MegaETH.
* Solana partial-signed VersionedTransaction v0: Solana.

The `payment_requirements.extra.assetTransferMethod` field is set to `"permit2"` when Permit2 is required; otherwise it is absent (EIP-3009 path) or uses the Solana payload shape.

## Usage

Specify chains with SDK constants, not hardcoded strings. `targetChain` is optional.

```
import { Chain } from '@cross402/usdc';

const intent = await client.createIntent({
  email: "merchant@example.com",
  amount: "100.50",
  payerChain: Chain.Base,
  targetChain: Chain.Ethereum,
});
```

```
resp, err := client.CreateIntent(ctx, &pay.CreateIntentRequest{
    Email:       "merchant@example.com",
    Amount:      "100.50",
    PayerChain:  pay.ChainBase,
    TargetChain: pay.ChainEthereum,
})
```

## Per-chain notes

Caveats for chains whose token, signing flavor, or role differ from the default (Circle-native USDC, EIP-3009, payer + target).

### BSC

* **Token**: Binance-Peg USDC at `0x8AC76a51cc950d9822D68b83fE1Ad97B32Cd580d`, **18 decimals**.
* **Signing**: Permit2 `PermitWitnessTransferFrom` + EIP-2612 `Permit`. Gas is sponsored by Cross402.
* **Role**: payer and target.
* See [BSC Signing](../../cross402/api/bsc-signing/) for the full flow.

### Monad

* **Token**: USDC at `0x754704Bc059F8C67012fEd69BC8A327a5aafb603`, 6 decimals.
* **Signing**: Permit2 `PermitWitnessTransferFrom` + EIP-2612 `Permit`. Gas is sponsored by Cross402.
* **Role**: payer and target.

### SKALE Base

* **Token**: **USDC.e** (Bridged USDC) at `0x85889c8c714505E0c94b30fcfcF64fE3Ac8FCb20`, 6 decimals. Not Circle-native USDC.
* **Signing**: EIP-3009 `TransferWithAuthorization`. EIP-712 domain `name = "Bridged USDC (SKALE Bridge)"`, `version = "2"`. Hardcoding `"USD Coin"` will fail signature verification.
* **Role**: **payer-only**. Cannot be used as `target_chain`.

### MegaETH

* **Token**: **USDm** (MegaUSD, native) at `0xFAfDdbb3FC7688494971a79cc65DCa3EF82079E7`, **18 decimals**. Not USDC.
* **Signing**: Permit2 `PermitWitnessTransferFrom` + EIP-2612 `Permit`. EIP-712 domain `name = "MegaUSD"`, `version = "1"`. Gas is sponsored by Cross402.
* **Role**: **payer-only**. Cannot be used as `target_chain`.
