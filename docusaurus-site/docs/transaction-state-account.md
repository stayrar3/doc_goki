---
sidebar_position: 7
---

# Transaction State Account

Welcome back, Goki explorers! In our previous chapter, [Smart Wallet Program](smart-wallet-program), we learned that the Smart Wallet Program is the "brain" of your Goki Smart Wallet, enforcing all the rules for multi-signature actions and timelocks. But where does this "brain" keep track of all the proposed actions, who approved them, and when they are ready?

This is where the **Transaction State Account** comes in. It's a special place on the Solana blockchain specifically dedicated to holding all the temporary details of a *proposed* action that your Goki Smart Wallet needs to perform.

## The Digital Proposal Form

Imagine your Goki Smart Wallet is like a club's shared bank account, but on the blockchain. When someone wants to spend money or change a club rule, they don't just do it instantly. Instead, they fill out a "proposal form" and submit it. This form isn't the final action; it's a request that needs to be reviewed and approved by other members.

On the Solana blockchain, the **Transaction State Account** serves as this digital proposal form. When an owner uses their [SmartWalletWrapper](smart-wallet-wrapper) to `newTransaction`, the [Smart Wallet Program](smart-wallet-program) creates one of these accounts. It then records every detail about that specific proposed action, allowing other owners to review and approve it.

**What problem does it solve?** It provides a clear, transparent, and auditable record of every proposed action, tracking its progress through the multi-signature approval process until it's ready to be executed. It ensures that all necessary conditions ([Owners & Threshold](owners-threshold), [Timelock](timelock)) are met before the actual actions take place.

Our goal in this chapter is to understand how the **Transaction State Account** is the temporary "holding pen" for transactions, storing all the necessary information until they are ready to be finalized.

## Key Concepts: What a Transaction State Account Stores

The Transaction State Account is essentially a data storage unit on Solana. Here's a look at the crucial details it holds:

| Detail                  | Description                                                                                                                                                                                                                                |
| :---------------------- | :----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Proposer**            | The public key (address) of the owner who initially submitted this transaction proposal.                                                                                                                                                   |
| **Instructions**        | The exact list of Solana blockchain actions (like "send SOL to this address" or "change a setting") that the Smart Wallet needs to perform once the proposal is approved and executed.                                                    |
| **Signers (Approvals)** | A list of "Yes" or "No" for each owner of the Smart Wallet, indicating which owners have already given their approval for this specific transaction. This is how the [Smart Wallet Program](smart-wallet-program) tracks the [Threshold](owners-threshold). |
| **ETA (Timelock)**      | The "Estimated Time of Arrival" – a specific timestamp (in seconds since epoch) when this transaction is allowed to be executed. This enforces the [Timelock](timelock) delay for this particular transaction.                      |
| **Status**              | It also keeps track of the transaction's lifecycle: whether it's still pending approvals, if it's already been executed, or if it expired.                                                                                                   |

Think of the `transactionKey` you've seen in previous examples (like `await smartWalletWrapper.newTransaction(...)`) as the unique address (Public Key) of this specific Transaction State Account on the blockchain.

## Using the Transaction State Account

As a user or developer, you don't directly manipulate the `Transaction State Account`. Instead, you interact with it through the convenient functions provided by the [SmartWalletWrapper](smart-wallet-wrapper), which then tells the [Smart Wallet Program](smart-wallet-program) what to do.

Let's revisit a familiar example, focusing on how the Transaction State Account is involved:

### 1. Proposing a Transaction (`newTransaction`)

When you propose a new action, the [Smart Wallet Program](smart-wallet-program) creates a new Transaction State Account for it.

```typescript
import { SystemProgram } from "@solana/web3.js";
// Assume smartWalletWrapper is already set up from Chapter 1

const recipient = Keypair.generate().publicKey; // Our friend's address
const amountToSend = 0.1 * LAMPORTS_PER_SOL;

const transferInstruction = SystemProgram.transfer({
  fromPubkey: smartWalletWrapper.key, // From our Smart Wallet
  toPubkey: recipient,
  lamports: amountToSend,
});

// This action creates the Transaction State Account on chain
const { transactionKey, tx: proposeTx } =
  await smartWalletWrapper.newTransaction({
    proposer: ownerA.publicKey, // Proposer is recorded
    instructions: [transferInstruction], // Instructions are recorded
    // eta: new BN(Date.now() / 1000 + 60), // ETA (Timelock) can also be recorded
  });
proposeTx.signers.push(ownerA); // Sign the transaction to send the proposal
await proposeTx.confirm();
console.log("Transaction proposed. Its unique ID (Transaction State Account Address):", transactionKey.toBase58());
```
*Explanation*: When `smartWalletWrapper.newTransaction` is called, it sends a request to the [Smart Wallet Program](smart-wallet-program) to create a new `Transaction State Account`. The program then populates this new account with the `proposer`, the `instructions`, and any `eta` (timelock) you might have set. The `transactionKey` returned is the unique address of this new Transaction State Account on the blockchain.

### 2. Checking the State (`fetchTransaction`)

You can easily read the current state of a proposed transaction (and its corresponding Transaction State Account) using `fetchTransaction`.

```typescript
// ... (previous code for transactionKey)

const txState = await smartWalletWrapper.fetchTransaction(transactionKey);
console.log("Details from Transaction State Account:");
console.log("  Proposer:", txState.proposer.toBase58());
console.log("  Instructions count:", txState.instructions.length);
console.log("  Approvals received:", txState.numApprovals.toNumber()); // This count comes from 'signers' field
console.log("  Execution time (ETA):", txState.eta.toNumber() === -1 ? "None" : new Date(txState.eta.toNumber() * 1000).toLocaleString());
```
*Explanation*: `smartWalletWrapper.fetchTransaction` simply reads the data stored in the `Transaction State Account` identified by `transactionKey`. This lets you see who proposed it, what it's trying to do, how many approvals it has received so far (based on its `signers` field), and its `eta` ([Timelock](timelock)).

### 3. Approving a Transaction (`approveTransaction`)

When an owner approves a transaction, the [Smart Wallet Program](smart-wallet-program) updates the relevant Transaction State Account.

```typescript
// ... (previous code for transactionKey)

// Owner B approves the transaction
await smartWalletWrapper
  .approveTransaction(transactionKey, ownerB.publicKey)
  .addSigners(ownerB)
  .confirm();
console.log("Transaction approved by Owner B.");

const updatedTxState = await smartWalletWrapper.fetchTransaction(transactionKey);
console.log("Approvals after Owner B:", updatedTxState.numApprovals.toNumber());
// The 'signers' list within the Transaction State Account has been updated!
```
*Explanation*: `smartWalletWrapper.approveTransaction` sends an instruction to the [Smart Wallet Program](smart-wallet-program). The program then finds the `Transaction State Account` for `transactionKey` and marks `ownerB` as having approved it in its `signers` field.

### 4. Executing a Transaction (`executeTransaction`)

Once enough approvals are gathered and any [Timelock](timelock) has passed, the `Transaction State Account` is marked as executed, and its stored instructions are run.

```typescript
// ... (previous code for transactionKey)

// Any owner (e.g., Owner C) can execute once conditions are met
await smartWalletWrapper
  .executeTransaction({
    transactionKey,
    owner: ownerC.publicKey,
  })
  .addSigners(ownerC)
  .confirm();
console.log("Transaction executed successfully!");

const finalTxState = await smartWalletWrapper.fetchTransaction(transactionKey);
console.log("Transaction executed at:", new Date(finalTxState.executed_at.toNumber() * 1000).toLocaleString());
// The 'executed_at' field in the Transaction State Account is now updated!
```
*Explanation*: When `smartWalletWrapper.executeTransaction` is called, the [Smart Wallet Program](smart-wallet-program) reads the `Transaction State Account` to verify the `threshold` and `eta` conditions. If met, it then executes the `instructions` stored within this account. Finally, it marks the `Transaction State Account` itself as `executed` (by setting `executed_at`), preventing it from being run again.

## Under the Hood: The Transaction State Account's Lifecycle

The Transaction State Account is created and managed directly by the [Smart Wallet Program](smart-wallet-program). It's a temporary account that tracks the status of a specific proposed action.

Here's a simplified flow of how the `Transaction State Account` is used:

```mermaid
sequenceDiagram
    participant YourApp
    participant SmartWalletWrapper
    participant SolanaNetwork
    participant SmartWalletProgram
    participant TransactionStateAccount

    YourApp->>SmartWalletWrapper: 1. Propose (newTransaction)
    SmartWalletWrapper->>SolanaNetwork: 2. Send 'create_transaction' instruction
    SolanaNetwork->>SmartWalletProgram: 3. Invoke Smart Wallet Program
    SmartWalletProgram->>TransactionStateAccount: 4. Create new account &<br>store proposer, instructions, ETA
    Note over TransactionStateAccount: This is the digital proposal form!
    TransactionStateAccount-->>SmartWalletProgram: 5. Account ready
    SmartWalletProgram-->>SolanaNetwork: 6. Confirmation
    SolanaNetwork-->>SmartWalletWrapper: 7. Confirmation (with transactionKey)
    SmartWalletWrapper-->>YourApp: 8. Return transactionKey

    YourApp->>SmartWalletWrapper: 9. Approve (approveTransaction)
    SmartWalletWrapper->>SolanaNetwork: 10. Send 'approve' instruction
    SolanaNetwork->>SmartWalletProgram: 11. Invoke Smart Wallet Program
    SmartWalletProgram->>TransactionStateAccount: 12. Update 'signers' (approvals)
    TransactionStateAccount-->>SmartWalletProgram: 13. Approvals recorded
    SmartWalletProgram-->>SolanaNetwork: 14. Confirmation
    SolanaNetwork-->>SmartWalletWrapper: 15. Confirmation
    SmartWalletWrapper-->>YourApp: 16. Confirmation

    YourApp->>SmartWalletWrapper: 17. Execute (executeTransaction)
    SmartWalletWrapper->>SolanaNetwork: 18. Send 'execute_transaction' instruction
    SolanaNetwork->>SmartWalletProgram: 19. Invoke Smart Wallet Program
    Note over SmartWalletProgram: 20. Read 'signers' & 'ETA' from TransactionStateAccount<br>21. Check threshold & timelock conditions
    SmartWalletProgram->>TransactionStateAccount: 22. Mark as executed<br>(set 'executed_at')
    TransactionStateAccount-->>SmartWalletProgram: 23. Status updated
    SmartWalletProgram-->>SolanaNetwork: 24. Execute stored instructions<br> (if conditions met)
    SolanaNetwork-->>SmartWalletWrapper: 25. Confirmation
    SmartWalletWrapper-->>YourApp: 26. Confirmation
```
*Explanation*:
1.  **Creation**: When `newTransaction` is called, the [Smart Wallet Program](smart-wallet-program) creates a new `Transaction State Account` on the blockchain. This account is filled with the transaction's details: who proposed it, what instructions it contains, and its optional [Timelock](timelock) `eta`.
2.  **Updates**: When `approveTransaction` is called, the [Smart Wallet Program](smart-wallet-program) updates the `signers` field within that specific `Transaction State Account` to record the new approval.
3.  **Validation & Execution**: When `executeTransaction` is called, the [Smart Wallet Program](smart-wallet-program) reads the `Transaction State Account`. It checks the `signers` list against the `threshold` (from the main `SmartWallet` account) and the current time against the `eta` ([Timelock](timelock)). If all conditions are met, the program then executes the `instructions` stored in the `Transaction State Account` on behalf of the Smart Wallet. Finally, it updates the `Transaction State Account` itself to mark it as `executed` (by setting `executed_at`), preventing it from being run again.

## Conclusion

The **Transaction State Account** is a fundamental component of the Goki Protocol, acting as the dedicated on-chain record for every proposed action of a Smart Wallet. It meticulously stores all the details, from the initial proposal and its instructions to the ongoing approvals and any time-based restrictions. This transparent and persistent record is crucial for ensuring that multi-signature rules are correctly enforced and that actions are only executed when all conditions are met.

Now that we understand how proposed transactions are recorded and tracked, let's move on to the next exciting step: how the [Smart Wallet Program](smart-wallet-program) actually *performs* the instructions stored in the Transaction State Account using a powerful Solana feature called [Instruction Execution (CPI)](instruction-execution-cpi).
