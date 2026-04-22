# ⛓ Supported Chains

AgentPay supports paying from one chain and settling on another. A payment intent declares two chains:

- `payer_chain` — where the payer sends USDC.
- `target_chain` — where the merchant receives USDC. Optional; defaults to `base`.

The set of target chains available to your integration is served by `GET /api/chains`. If a chain appears in that response, you can use it as a `target_chain`. Any chain supported as a payer can also be used as a target, with two exceptions noted below.

## Payer chains

| Chain | Identifier | Go Constant | TS Constant | USDC Decimals | Notes |
| :--- | :--- | :--- | :--- | :--- | :--- |
| Solana | `solana` | `pay.ChainSolanaMainnet` | `Chain.SolanaMainnet` | 6 | — |
| Base | `base` | `pay.ChainBase` | `Chain.Base` | 6 | — |
| BSC | `bsc` | `pay.ChainBSC` | `Chain.BSC` | **18** | Binance-Peg USDC; Permit2 + EIP-2612 signing. See [BSC Signing](bsc-signing.md). |
| Polygon | `polygon` | `pay.ChainPolygon` | `Chain.Polygon` | 6 | — |
| Arbitrum | `arbitrum` | `pay.ChainArbitrum` | `Chain.Arbitrum` | 6 | — |
| Ethereum | `ethereum` | `pay.ChainEthereum` | `Chain.Ethereum` | 6 | — |
| Monad | `monad` | `pay.ChainMonad` | `Chain.Monad` | 6 | Permit2 + EIP-2612 signing. |
| HyperEVM | `hyperevm` | `pay.ChainHyperEVM` | `Chain.HyperEvm` | 6 | — |
| SKALE Base | `skale-base` | `pay.ChainSkaleBase` | `Chain.SkaleBase` | 6 | EIP-712 domain name is `"Bridged USDC (SKALE Bridge)"` (not `"USD Coin"`). |
| MegaETH | `megaeth` | `pay.ChainMegaETH` | `Chain.MegaEth` | **18** | Native USDm (MegaUSD), EIP-712 domain `name="MegaUSD"`, `version="1"`; Permit2 + EIP-2612 signing. |

## Target chains

Every payer chain is also a valid target, with these caveats:

- `skale-base` and `megaeth` are target-eligible but only when your deployment exposes them. They are always listed in the payer set; they appear in the target set only when `GET /api/chains` reports them. Treat that endpoint as authoritative.
- Target chains are validated at intent creation. If you pass an unsupported value, the API returns `400` with an error that lists the currently accepted target chains.

```
GET /api/chains
```

Response:

```json
{
  "chains": ["arbitrum", "base", "bsc", "ethereum", "hyperevm", "monad", "polygon", "solana"]
}
```

The list is stable per deployment and changes only when operators add or remove chain support.

## Payer × target matrix

Any payer chain can pair with any target chain exposed by your deployment, including same-chain routes (e.g. `base → base`). The most common combinations look like this:

| Payer → Target | CCTP burn/mint | Direct EVM transfer | Solana (SVM) transfer |
| :--- | :---: | :---: | :---: |
| `solana → base` | ✓ | | |
| `solana → ethereum` | ✓ | | |
| `base → ethereum` | | ✓ | |
| `polygon → arbitrum` | | ✓ | |
| `bsc → base` | | ✓ | |
| `base → solana` | | | ✓ |
| `base → base` (same chain) | | ✓ | |

Callers do not pick the settlement mode; AgentPay chooses between CCTP burn/mint, direct EVM transfer, and SVM transfer based on the (payer, target) pair. See [Multi-Chain Settlement](../docs/concepts/multi-chain-settlement.md) for the mechanics.

## Chain-specific caveats

These are the details you need at signing and display time.

### USDC decimals

Always read `extra.decimals` from the `payment_requirements` object on the `CreateIntent` response. Do not hardcode `6`.

| Chain | Decimals | Source |
| :--- | :---: | :--- |
| Solana, Base, Polygon, Arbitrum, Ethereum, Monad, HyperEVM, SKALE Base | 6 | Circle USDC / Bridged USDC |
| BSC | 18 | Binance-Peg USDC |
| MegaETH | 18 | Native USDm (MegaUSD) |

### EIP-712 domain names

When signing EIP-3009 `TransferWithAuthorization` or Permit2, use the `extra.name` and `extra.version` values from the intent response. The defaults are `"USD Coin"` / `"2"`, but two chains are exceptions:

- **SKALE Base**: `name = "Bridged USDC (SKALE Bridge)"`, `version = "2"`.
- **MegaETH**: `name = "MegaUSD"`, `version = "1"`.

Hardcoding the default `"USD Coin"` on either chain makes every signature fail verification.

### Signing flavor

- EIP-3009 `TransferWithAuthorization`: Base, Polygon, Arbitrum, Ethereum, HyperEVM, SKALE Base.
- Permit2 `PermitWitnessTransferFrom` + EIP-2612 `Permit` (gas-sponsored by AgentPay): BSC, Monad, MegaETH.
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
