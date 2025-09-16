---
sidebar_position: 8
---

# Instruction Execution (CPI)

Welcome back, Goki explorers! In our previous chapter, [Transaction State Account](transaction-state-account), we learned that the Smart Wallet Program stores all the details of a proposed transaction, including the actual instructions to be performed, in a special `Transaction State Account`. But how does the Smart Wallet actually *perform* those instructions? It's not like the Smart Wallet itself is a general-purpose computer that can just "do" anything.

This is where **Instruction Execution (CPI)** comes in. At its core, your Goki Smart Wallet doesn't directly perform actions like sending SOL or modifying accounts. Instead, it acts as a powerful signer that can authorize and execute instructions from *other* Solana programs on its behalf.

## The CEO and Their Departments

Imagine a busy CEO (your Goki Smart Wallet) who needs many different tasks done: sending money, creating new projects, or hiring new staff. The CEO doesn't do all these tasks themselves. Instead, they have specialized departments (other Solana programs) for each job:
*   The "Finance Department" (Solana System Program) for sending SOL.
*   The "Token Department" (Solana SPL Token Program) for managing tokens.
*   And so on.

When the CEO wants something done, they write a clear request (an "instruction") and then digitally *sign* it with their authority, handing it over to the correct department. The department then performs the task, knowing it has the CEO's official approval.

On Solana, this delegation of tasks is achieved through **Cross-Program Invocation (CPI)**. Your Goki Smart Wallet Program can "call" another program, digitally signing the instruction with its own authority (or the authority of one of its derived addresses). It's like a central command center delegating tasks to specialized departments while maintaining overall control.

**What problem does it solve?** It allows your Goki Smart Wallet to interact with any other program on Solana. The Smart Wallet itself only needs to know *how to delegate*, not *how to perform every single blockchain operation*. This makes it incredibly flexible and powerful, as it can manage anything that Solana programs can do.

## Key Concepts: Delegating with Authority

Let's break down how CPI works in the context of your Goki Smart Wallet:

| Concept                      | Description                                                                                                                                                                                                                            |
| :--------------------------- | :------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Cross-Program Invocation (CPI)** | The act of one Solana program calling or "invoking" another Solana program. It's how programs interact and build on top of each other.                                                                                                |
| **Delegation**               | The Smart Wallet doesn't execute the final action itself. It delegates the task to the specific program designed to perform that action (e.g., the System Program for transfers).                                                           |
| **Signing Authority**        | Crucially, when the Smart Wallet delegates, it also provides its own digital "signature" (or the signature of one of its [Program Derived Addresses (PDAs)](program-derived-addresses-pdas)). This signature gives the delegated instruction the authority of the Smart Wallet, allowing it to spend SOL from the Smart Wallet, or change settings for accounts owned by the Smart Wallet. |
| **Instructions (from other programs)** | The actions stored in the [Transaction State Account](transaction-state-account) are not Goki's own instructions. They are standard Solana `TransactionInstruction` objects meant for *other* programs. For example, `SystemProgram.transfer` is an instruction for the Solana System Program. |

## How the Smart Wallet Uses CPI for Execution

Recall the `executeTransaction` function from your [SmartWalletWrapper](smart-wallet-wrapper). This is where CPI happens.

When you call `executeTransaction`, here's how the Smart Wallet uses CPI:

### 1. Proposing a Transaction with an External Instruction

First, you define an instruction for another program. For example, to send SOL, you use `SystemProgram.transfer`, which is an instruction for the Solana System Program:

```typescript
import { SystemProgram } from "@solana/web3.js";
// Assume smartWalletWrapper is already set up from Chapter 1

const recipient = Keypair.generate().publicKey;
const amountToSend = 0.1 * LAMPORTS_PER_SOL;

// This instruction is for the Solana System Program!
const transferInstruction = SystemProgram.transfer({
  fromPubkey: smartWalletWrapper.key, // Smart Wallet is the sender
  toPubkey: recipient,
  lamports: amountToSend,
});

// Propose the transaction, storing the SystemProgram.transfer instruction
const { transactionKey, tx: proposeTx } =
  await smartWalletWrapper.newTransaction({
    proposer: ownerA.publicKey,
    instructions: [transferInstruction], // This is what gets stored
  });
proposeTx.signers.push(ownerA);
await proposeTx.confirm();
console.log("Transfer transaction proposed. ID:", transactionKey.toBase58());
```
*Explanation*: The `transferInstruction` is a standard Solana instruction telling the `SystemProgram` to move SOL. When `newTransaction` is called, this instruction (including its `programId`, `keys`, and `data`) is stored inside the [Transaction State Account](transaction-state-account) on the blockchain.

### 2. Executing the Transaction (CPI in Action)

Once enough [Owners & Threshold](owners-threshold) are met and any [Timelock](timelock) has passed, you call `executeTransaction`. This is when the Smart Wallet performs CPI.

```typescript
// ... (previous code for transactionKey)

// Execute the transaction
await smartWalletWrapper
  .executeTransaction({
    transactionKey,
    owner: ownerA.publicKey,
  })
  .addSigners(ownerA)
  .confirm();
console.log("Transaction executed successfully!");
// At this point, the Smart Wallet has delegated to the SystemProgram,
// and 0.1 SOL has been sent to the recipient!
```
*Explanation*: When `executeTransaction` is confirmed, the [Smart Wallet Program](smart-wallet-program) on the blockchain retrieves the `transferInstruction` from the [Transaction State Account](transaction-state-account). It then performs a CPI call, telling the Solana runtime: "Hey, System Program! Please execute this `transferInstruction`, and let me, the Smart Wallet, sign for it!" The System Program then performs the transfer, decrementing the SOL balance of the Smart Wallet.

## Under the Hood: CPI in the Smart Wallet Program

The actual CPI magic happens within the [Smart Wallet Program](smart-wallet-program)'s `execute_transaction` function (or `execute_transaction_derived`, for [PDAs](program-derived-addresses-pdas)).

Here's a simplified view of the flow:

```mermaid
sequenceDiagram
    participant User
    participant SmartWalletWrapper
    participant SolanaNetwork
    participant SmartWalletProgram
    participant TransactionStateAccount
    participant OtherProgram as System Program (or other)

    User->>SmartWalletWrapper: 1. Call executeTransaction()
    SmartWalletWrapper->>SolanaNetwork: 2. Send 'execute_transaction' instruction
    SolanaNetwork->>SmartWalletProgram: 3. Invoke Smart Wallet Program
    SmartWalletProgram->>TransactionStateAccount: 4. Read instructions, approvals, ETA
    Note over SmartWalletProgram: 5. Verify threshold & timelock conditions
    SmartWalletProgram->>OtherProgram: 6. CPI: Invoke Program<br>(with Smart Wallet's signature)
    OtherProgram-->>SmartWalletProgram: 7. Execute instruction & return result
    SmartWalletProgram->>TransactionStateAccount: 8. Mark transaction as executed
    TransactionStateAccount-->>SmartWalletProgram: 9. Status updated
    SmartWalletProgram-->>SolanaNetwork: 10. Acknowledge completion
    SolanaNetwork-->>SmartWalletWrapper: 11. Confirmation
    SmartWalletWrapper-->>User: 12. Return result
```
*Explanation*:
1.  **User Initiates**: You call `executeTransaction` on your `smartWalletWrapper`.
2.  **Instruction to Network**: The `SmartWalletWrapper` prepares an `execute_transaction` instruction for the [Smart Wallet Program](smart-wallet-program) and sends it to the Solana network.
3.  **Program Invoked**: The Solana network routes this instruction to the [Smart Wallet Program](smart-wallet-program).
4.  **Read Transaction**: The [Smart Wallet Program](smart-wallet-program) fetches the details of the proposed transaction from its [Transaction State Account](transaction-state-account).
5.  **Checks & Balances**: It verifies that enough owners have approved (meeting the [Threshold](owners-threshold)) and that the [Timelock](timelock) (ETA) has passed. If these conditions are not met, the transaction fails here.
6.  **CPI Call (`invoke_signed`)**: If all checks pass, the [Smart Wallet Program](smart-wallet-program) takes the `instructions` stored in the [Transaction State Account](transaction-state-account). For *each* of these instructions, it makes a special **CPI call** using a Solana function called `invoke_signed`. This function tells the Solana runtime to execute the instruction *as if the Smart Wallet itself (or one of its [Program Derived Addresses (PDAs)](program-derived-addresses-pdas)) had directly signed it*.
7.  **Other Program Acts**: The target program (e.g., System Program) receives this CPI-invoked instruction, recognizes the Smart Wallet's signature, and performs the requested action (e.g., transfers SOL).
8.  **Program Finalizes**: After all delegated instructions are successfully executed, the [Smart Wallet Program](smart-wallet-program) marks the [Transaction State Account](transaction-state-account) as executed.
9.  **Confirmation**: The Solana network confirms the overall transaction, and the result is returned to your `SmartWalletWrapper` and then to your application.

## Conclusion

**Instruction Execution (CPI)** is how your Goki Smart Wallet, acting as a powerful central authority, delegates tasks to other specialized programs on the Solana blockchain. By digitally signing these delegated instructions with its own authority (often through a PDA), the Smart Wallet can perform a wide range of actions, from sending SOL to managing tokens, all under the strict multi-signature and timelock rules you define. This mechanism makes Goki incredibly flexible and a true "smart" wallet.

In the next chapter, we will take a closer look at [Program Derived Addresses (PDAs)](program-derived-addresses-pdas), the special types of addresses frequently used by programs like Goki to sign for CPI calls and manage assets on the blockchain without needing a private key.
