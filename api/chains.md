# ⛓ Supported Chains

cross402 supports paying from one chain and settling on another. A payment intent declares two chains:

- `payer_chain` — where the payer sends USDC.
- `target_chain` — where the merchant receives USDC. Optional; defaults to `base`.

The set of target chains available to your integration is served by `GET /api/chains`. If a chain appears in that response, you can use it as a `target_chain`. Any chain supported as a payer can also be used as a target.

## Status legend

- **Live** — callable. You can pass the identifier as `payer_chain` or `target_chain` and the call will succeed (assuming the usual validation). Treat `GET /api/chains` as authoritative for the exact set exposed by your deployment.

## Payer chains

| Chain | Identifier | Go Constant | TS Constant | USDC Decimals | Status | Notes |
| :--- | :--- | :--- | :--- | :--- | :--- | :--- |
| Solana | `solana` | `pay.ChainSolanaMainnet` | `Chain.SolanaMainnet` | 6 | Live | — |
| Base | `base` | `pay.ChainBase` | `Chain.Base` | 6 | Live | — |
| Ethereum | `ethereum` | `pay.ChainEthereum` | `Chain.Ethereum` | 6 | Live | — |
| Polygon | `polygon` | `pay.ChainPolygon` | `Chain.Polygon` | 6 | Live | — |
| HyperEVM | `hyperevm` | `pay.ChainHyperEVM` | `Chain.HyperEvm` | 6 | Live | — |
| Arbitrum | `arbitrum` | `pay.ChainArbitrum` | `Chain.Arbitrum` | 6 | Live | — |
| BSC | `bsc` | `pay.ChainBSC` | `Chain.BSC` | **18** | Live | Binance-Peg USDC; Permit2 + EIP-2612 signing. See [BSC Signing](bsc-signing.md). |
| Monad | `monad` | `pay.ChainMonad` | `Chain.Monad` | 6 | Live | Permit2 + EIP-2612 signing. |
| SKALE Base | `skale-base` | `pay.ChainSKALEBase` | `Chain.SkaleBase` | 6 | Live | Payer-only. EIP-712 domain name is `"Bridged USDC (SKALE Bridge)"` (not `"USD Coin"`). Testnet: `skale-base-sepolia` (JS: `Chain.SkaleBaseSepolia`; Go has no testnet constant yet). |
| MegaETH | `megaeth` | `pay.ChainMegaETH` | `Chain.MegaEth` | **18** | Live | Payer-only. Native USDm (MegaUSD), EIP-712 domain `name="MegaUSD"`, `version="1"`; Permit2 + EIP-2612 signing. |

## Target chains

Every payer chain is also a valid target. One caveat applies:

- `GET /api/chains` is authoritative. It lists the chains accepted as a `target_chain` for your deployment. Target chains are validated at intent creation; if you pass an unsupported value, the API returns `400` with an error that lists the currently accepted target chains.

```
GET /api/chains
```

Response:

```json
{
  "chains": ["base", "ethereum", "hyperevm", "polygon", "solana", "skale-base", "megaeth"],
  "target_chains": ["base", "ethereum", "hyperevm", "polygon", "solana"]
}
```

The two lists are independent — `chains` enumerates valid `payer_chain` values and `target_chains` enumerates valid `target_chain` values. Payer-only chains (e.g. `skale-base`, `megaeth`) appear in `chains` but not in `target_chains`. The lists are stable per deployment.

## Payer × target matrix

Any payer chain can pair with any target chain exposed by your deployment, including same-chain routes (e.g. `base → base`). The table below covers common combinations. The **Payer sig** and **Target sig** columns show the x402 signing flavor used on each leg — the SDK picks these automatically from `payment_requirements`.

| Payer → Target | Payer sig | Target sig | Status |
| :--- | :--- | :--- | :--- |
| `solana → base` | Solana VT v0 | EIP-3009 | Live |
| `solana → ethereum` | Solana VT v0 | EIP-3009 | Live |
| `base → ethereum` | EIP-3009 | EIP-3009 | Live |
| `base → polygon` | EIP-3009 | EIP-3009 | Live |
| `polygon → base` | EIP-3009 | EIP-3009 | Live |
| `base → solana` | EIP-3009 | Solana VT v0 | Live |
| `base → base` (same chain) | EIP-3009 | EIP-3009 | Live |
| `polygon → arbitrum` | EIP-3009 | EIP-3009 | Live |
| `bsc → base` | Permit2 + EIP-2612 | EIP-3009 | Live |

Callers do not choose the signing flavor; the SDK derives it from `payment_requirements` and the `(payer, target)` pair. Both legs run through the x402 protocol. See [Multi-Chain Settlement](../docs/concepts/multi-chain-settlement.md) for the mechanics.

## Chain-specific caveats

These are the details you need at signing and display time.

### USDC decimals

Always read `extra.decimals` from the `payment_requirements` object on the `CreateIntent` response. Do not hardcode `6`.

| Chain | Decimals | Source | Status |
| :--- | :---: | :--- | :--- |
| Solana, Base, Ethereum, Polygon, HyperEVM | 6 | Circle USDC / Bridged USDC | Live |
| Arbitrum, Monad, SKALE Base | 6 | Circle USDC / Bridged USDC | Live |
| BSC | 18 | Binance-Peg USDC | Live |
| MegaETH | 18 | Native USDm (MegaUSD) | Live |

### EIP-712 domain names

When signing EIP-3009 `TransferWithAuthorization` or Permit2, use the `extra.name` and `extra.version` values from the intent response. The defaults are `"USD Coin"` / `"2"`, but two chains are exceptions:

- **SKALE Base**: `name = "Bridged USDC (SKALE Bridge)"`, `version = "2"`.
- **MegaETH**: `name = "MegaUSD"`, `version = "1"`.

Hardcoding the default `"USD Coin"` on either chain makes every signature fail verification.

### Signing flavor

- EIP-3009 `TransferWithAuthorization`: Base, Ethereum, Polygon, HyperEVM, Arbitrum, SKALE Base.
- Permit2 `PermitWitnessTransferFrom` + EIP-2612 `Permit` (gas-sponsored by cross402): BSC, Monad, MegaETH.
- Solana partial-signed VersionedTransaction v0: Solana.

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
