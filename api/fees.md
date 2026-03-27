# 💰 Fee Breakdown

All intent responses include a `FeeBreakdown` structure that details the cost components of a payment.

## Fee Structure

| Field | JSON | Description |
| :--- | :--- | :--- |
| SourceChain | `source_chain` | Source chain identifier |
| SourceChainFee | `source_chain_fee` | Gas/network fee on the source chain |
| TargetChain | `target_chain` | Target chain (always "base") |
| TargetChainFee | `target_chain_fee` | Gas/network fee on Base |
| PlatformFee | `platform_fee` | Platform service fee |
| PlatformFeePercentage | `platform_fee_percentage` | Platform fee as a percentage |
| TotalFee | `total_fee` | Sum of all fees |

## Understanding Fees

* **Source Chain Fee**: The gas/transaction fee required on the payer's source chain (e.g., Solana or Base).
* **Target Chain Fee**: The gas fee for the final USDC transfer on Base.
* **Platform Fee**: AgentTech's service fee for processing the cross-chain settlement.
* **Total Fee**: The sum of all fees, which is deducted from the payment amount.

## Example Response

```json
{
  "fee_breakdown": {
    "source_chain": "base",
    "source_chain_fee": "0.001",
    "target_chain": "base",
    "target_chain_fee": "0.0001",
    "platform_fee": "1.00",
    "platform_fee_percentage": "1.0",
    "total_fee": "1.0011"
  }
}
```
