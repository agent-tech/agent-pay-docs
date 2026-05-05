# 💰 Fee Breakdown

All intent responses include a `FeeBreakdown` structure that details the cost components of a payment. Fees are computed per `(payer_chain, target_chain)` pair — the same payer chain can produce different `target_chain_fee` values depending on the target.

## Fee Structure

| Field | JSON | Description |
| :--- | :--- | :--- |
| SourceChain | `source_chain` | Payer chain identifier |
| SourceChainFee | `source_chain_fee` | Gas/network fee on the payer chain |
| TargetChain | `target_chain` | Target (settlement) chain identifier |
| TargetChainFee | `target_chain_fee` | Gas/network fee on the target chain |
| PlatformFee | `platform_fee` | Platform service fee |
| PlatformFeePercentage | `platform_fee_percentage` | Platform fee as a percentage |
| TotalFee | `total_fee` | Sum of all fees |

## Understanding Fees

* **Source Chain Fee**: The gas/transaction cost required on the payer's chain (for example a Solana priority fee or Base gas).
* **Target Chain Fee**: The gas cost for the USDC transfer on the target chain. This varies meaningfully between targets — Ethereum is typically the most expensive, while Base and Polygon are much cheaper. Budget accordingly if you let users pick their target chain.
* **Platform Fee**: cross402's service fee for processing the cross-chain settlement.
* **Total Fee**: The sum of all fees, deducted from the payer's sending amount before the merchant receives the net amount.

## Example Response — base → Ethereum

```json
{
  "fee_breakdown": {
    "source_chain": "base",
    "source_chain_fee": "0.001",
    "target_chain": "ethereum",
    "target_chain_fee": "0.85",
    "platform_fee": "1.00",
    "platform_fee_percentage": "1.0",
    "total_fee": "1.851"
  }
}
```

`target_chain` and `target_chain_fee` reflect whichever target you selected on `CreateIntent`. Requesting `target_chain: "polygon"` on the same payer would return a much lower `target_chain_fee`.
