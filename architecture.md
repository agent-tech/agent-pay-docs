# 🏗 Architecture

AgentTech acts as a bridge between multiple source chains and the Base settlement layer.

### Payment Flow
1. **Initiation**: The merchant creates a Payment Intent via the SDK.
2. **Payment**: The payer sends USDC on their chosen chain (e.g., Solana).
3. **Verification**: AgentTech monitors the source chain for transaction finality.
4. **Settlement**: Once verified, AgentTech executes the final transfer on the **Base chain** to the merchant's wallet.

### Trust Model
* **Non-Custodial**: Users sign their own transactions for client-side flows.
* **Agent-Managed**: For server-side flows, our secure Agent infrastructure handles the gas and signing on Base.