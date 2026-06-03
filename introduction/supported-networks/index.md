---
order: 1
---

# Supported Chains

Cross402 supports paying from one chain and settling on another, optionally in a different stablecoin. A payment intent declares two chains and (optionally) two assets:

* `payer_chain` — where the payer sends stablecoins.
* `target_chain` — where the merchant receives stablecoins. Optional; defaults to `base`.
* `payer_asset` — the token the payer signs against. Optional; defaults to `"usdc"`. One of `"usdc"` | `"usdt"` | `"usdt0"`.
* `target_asset` — the token the merchant receives. Optional; defaults to `"usdc"`. Same allowed values.

Clients that omit `payer_asset` / `target_asset` continue to behave exactly as before — both fields default to `"usdc"` with no change in wire format. See [Token × Chain matrix](#token--chain-matrix) for what's supported per chain.

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

## Token × Chain matrix

Every chain ships at minimum with USDC. USDT0 (LayerZero's omni-Tether) is wired on the chains where it has a verified deployment; legacy native USDT is wired on Ethereum / BSC / Base / Polygon (gated behind a deployment flag). The matrix below is authoritative.

| Chain | USDC | USDT0 | USDT (native) |
| :--- | :--- | :--- | :--- |
| Solana | ✅ Live | — | — |
| Base | ✅ Live | — | 🚩 gated · `SchemeApprovalSponsorship` |
| Ethereum | ✅ Live | — | 🚩 gated · `SchemeApprovalSponsorship` |
| Polygon | ✅ Live (standard EIP-3009) | ✅ Live · **salted** EIP-712 domain | ✅ Live · alias of USDT0 (same contract) |
| Arbitrum | ✅ Live | ✅ Live · domain `name="USD₮0"` (U+20AE TUGRIK), `version="1"` | ✅ alias of USDT0 (same contract, EIP-3009) |
| Monad | ✅ Live | ✅ Live · domain `name="USDT0"`, `version="1"` | — |
| HyperEVM | ✅ Live | ✅ Live · domain `name="USD₮0"`, `version="1"` | — |
| MegaETH | Live (payer-only) (native USDm) | ✅ Live · domain `name="USDT0"`, `version="1"` | — |
| BSC | Live (Binance-Peg) | — | 🚩 gated · `SchemeApprovalSponsorship` |
| SKALE Base | Live (payer-only) | — | — |

Legend: **✅ Live** = the (chain, asset) pair is callable today. **🚩 gated** = wired but gated behind the `USDT_ENABLED` deployment flag; calls return `400 invalid payer_asset` while the gate is off. **—** = not wired; `400 payer_asset not configured on chain` if requested.

## Chain-specific caveats

These are the details you need at signing and display time.

### Token contracts and decimals

Always read `extra.decimals` from the `payment_requirements` object on the `CreateIntent` response. Do not hardcode `6`. Mainnet token addresses are pinned by the backend per chain.

#### USDC (default, `payerAsset` omitted or `"usdc"`)

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

#### USDT (`payerAsset: "usdt"`)

> **USDT payers on BSC, Ethereum, and Base must hold native gas** (BNB / ETH) to send a one-time on-chain `approve(Permit2, amount)` before payment. Polygon USDT uses EIP-2612 with a salted domain — no gas required. See [USDT Signing](../../cross402/api/usdt-signing/) for details.

| Chain | Decimals | Token | Mainnet Address | Payer gas required |
| --- | --- | --- | --- | --- |
| BSC | 18 | USDT (BEP-20) | `0x55d398326f99059fF775485246999027B3197955` | **Yes (BNB)** |
| Ethereum | 6 | USDT (ERC-20) | `0xdAC17F958D2ee523a2206206994597C13D831ec7` | **Yes (ETH)** |
| Base | 6 | USDT (ERC-20) | `0xfde4C96c8593536E31F229EA8f37b2ADa2699bb2` | **Yes (ETH)** |
| Polygon | 6 | USDT (PoS) | `0xc2132D05D31c914a87C6611C10748AEb04B58e8F` | **No** (EIP-2612 salted, gas sponsored) |

### EIP-712 domain values per asset

Always read `extra.name`, `extra.version`, and (where present) `extra.domainType` from `payment_requirements`. Never hardcode these — they vary per (chain, asset) deployment:

* **USDC** — defaults `name = "USD Coin"`, `version = "2"`. Exceptions: SKALE Base uses `"Bridged USDC (SKALE Bridge)"`, MegaETH native USDm uses `"MegaUSD"` + `version = "1"`.
* **USDT0** — `name` is `"USD₮0"` (Tugrik U+20AE) on Arbitrum / HyperEVM; plain ASCII `"USDT0"` on Monad / MegaETH / Polygon. `version = "1"` everywhere.
* **USDT (native)** — only signature-bearing on Polygon (alias of USDT0, EIP-3009). Ethereum / BSC / Base USDT use `SchemeApprovalSponsorship` and do not sign EIP-712 at all.

Hardcoding `"USD Coin"` v2 on SKALE Base or MegaETH produces signatures the on-chain contract rejects.

### Per-asset transfer scheme

`TransferScheme` is per-(chain, asset), not per-chain. The same chain can use different schemes for different assets:

* **EIP-3009 `TransferWithAuthorization`** — Circle USDC on Base / Ethereum / Polygon / HyperEVM / Arbitrum / SKALE Base; USDT0 on Arbitrum / Monad / HyperEVM / MegaETH.
* **Permit2 + EIP-2612** — USDC on BSC / Monad / MegaETH; **Polygon USDT / USDT0** (same contract `0xc2132D...`, uses EIP-2612 against a salted EIP-712 domain — no EIP-3009). Gas is sponsored by Cross402.
* **`SchemeApprovalSponsorship`** — legacy native USDT on Ethereum / BSC / Base. The facilitator pays gas to submit `approve` + `transferFrom` on the payer's behalf; first payment from a (chain, wallet) pair carries an `approve` (5–60s extra latency), subsequent payments are signature-only. Currently gated behind the `USDT_ENABLED` deployment flag.
* **Solana SPL** — USDC on Solana; partial-signed VersionedTransaction v0.

The `payment_requirements.extra.assetTransferMethod` field signals the scheme:

| Value | Meaning |
| --- | --- |
| absent | EIP-3009 (default for USDC on most EVM chains) |
| `"permit2"` | Permit2 + EIP-2612, gas sponsored by Cross402 |
| `"approval_sponsorship"` | Permit2 only; payer must `approve(Permit2)` on-chain first (USDT) |

See [USDT Signing](../../cross402/api/usdt-signing/) for the full USDT flow.

### Polygon: salted EIP-712 domain

Polygon's native Tether contract (`0xc2132D05D31c914a87C6611C10748AEb04B58e8F`, used as both USDT0 and native USDT) signs against the non-standard Polygon-PoS layout:

```solidity
EIP712Domain(string name, string version, address verifyingContract, bytes32 salt)
```

where `salt = bytes32(chainID)` — chainId moves out of the domain type and into a salt field. The intent response's `payment_requirements.extra.domainType == "salted"` is the explicit signal to switch the signer's DOMAIN_SEPARATOR construction. Standard chainId-in-domain EIP-712 signatures **fail verification** against these contracts.

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
