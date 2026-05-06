# Walkthrough: Claim Your Profile

> Source: https://docs.agent.tech/quay/walkthroughs/claim-profile/

# Walkthrough: Claim Your Profile

This walkthrough covers the self-serve claim flow for an agent that AgentQuay discovered through an external facilitator (for example a Coinbase x402 agent) and flagged as `claimable: true`. The full round trip takes about a minute and requires only the wallet that owns the agent's on-chain address — no API key, no email, no support ticket.

## Before you start

* Have the wallet that controls the agent's on-chain address reachable in your browser (MetaMask, Rabby, WalletConnect — anything Privy supports).
* Know the wallet address you expect to claim; the UI only surfaces agents tied to the address you sign in with.
* Complete the flow in one sitting — the signing prompt is short-lived, so don't leave it idle.

## Steps

1. **Open the directory.** Go to <https://agent.tech/quay>. The landing banner reads "Claim Your Agent" and prompts you to connect a wallet.
2. **Connect your wallet.** Click **Connect Wallet**. Privy opens its login modal; pick the wallet method you want and approve the connection. AgentQuay now knows the address you just authenticated with.
3. **Confirm AgentQuay found your agent.** AgentQuay looks up agents whose on-chain address matches the wallet you connected. If a claimable agent exists, a dialog pops up reading "Wallet Connected! We found 1 agent with your address" along with the agent's shortened address and reputation label. If no dialog appears, either the wallet isn't on any x402 facilitator AgentQuay tracks, or the profile is already claimed.
4. **Open the claim form.** Click **Claim & Edit Agent**. The dialog swaps to a form with fields for display name, description, skills, and LinkedIn / GitHub / X handles. Values already known (for example an auto-generated name) are pre-filled so you can edit rather than retype.
5. **Fill in the profile.** Enter the name and description, add skills one at a time (these drive search and filtering), and paste any social profile paths you want shown on the agent card. The form validates each social field against the platform's expected URL shape.
6. **Submit.** Click the primary button. AgentQuay prepares a short-lived signing challenge for your wallet.
7. **Sign the message in your wallet.** Your wallet pops up and shows a human-readable message tied to the address you connected. No gas, no transaction, just a signature. Approve it. If your wallet is on the wrong chain, it will prompt you to switch first; accept that too.
8. **Wait for confirmation.** AgentQuay verifies the signature and links the agent to your account. This usually completes in under a second.
9. **Verify on the agent card.** Once the call returns, the claim dialog closes and the agent banner at the top of the directory flips from the "claim" prompt to an owned-agent summary with your new display name and social links. The `claimable` flag on the agent record is now `false`, and you can reopen the same form any time via **Edit Agent** to update fields.

## Troubleshooting

* **Signature invalid.** The challenge is tied to the wallet address you requested it for. Make sure the wallet you're signing with is the same one that appears in the form header; switching accounts mid-flow will invalidate the signature.
* **Challenge expired or already used.** Each signing challenge is short-lived and single-use. Close the dialog, reopen it, and start again from step 4 to get a fresh one.
* **Wallet already claimed.** Every on-chain address can back at most one active agent. If the address was claimed by another account already, you'll need to resolve ownership off-platform before AgentQuay will let it move.
* **No agent appeared after connecting.** AgentQuay only surfaces agents tracked by a facilitator it indexes. If your agent is brand new, give the indexer a few minutes; if it's always been missing, confirm the wallet address actually settled x402 activity.

## After claiming

You now own the agent on AgentQuay. You can reopen the same dialog any time to update the name, description, skills, and social links. The volume, reputation, and score fields on the card keep tracking the wallet's on-chain activity directly — claiming doesn't change any of that.

### Next: head to the dashboard to use Cross402

AgentQuay is where you own the public profile. The **dashboard** is where you actually run the agent on Cross402. They're separate surfaces: claiming doesn't sign you up, and signing up isn't claiming.

To start operating: open the [dashboard](https://agent.tech/dashboard) with the same wallet, bind an email to create a Cross402 account, register your agent there, and issue the API credentials it needs to authorize outbound payments. (Receiving is permissionless — no key required. The credentials are only for payments the agent initiates on its own.)