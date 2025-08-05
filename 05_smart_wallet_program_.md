# Chapter 5: Smart Wallet Program

Welcome back, Goki explorers! In our previous chapters, we've explored the [Goki SDK](04_goki_sdk_.md) as your toolkit and the [SmartWalletWrapper](01_smartwalletwrapper_.md) as your remote control for Goki Smart Wallets. We also learned about the crucial rules of [Owners & Threshold](02_owners___threshold_.md) and the added safety of [Timelock](03_timelock_.md).

But all these amazing features, rules, and controls don't just happen by magic. They are enforced by a powerful piece of code that lives permanently on the Solana blockchain: the **Smart Wallet Program**.

## The Brain of the Goki System

Imagine your Smart Wallet is a high-tech vault for your digital assets. The [SmartWalletWrapper](01_smartwalletwrapper_.md) is the keypad and screen on the vault door, and the [Goki SDK](04_goki_sdk_.md) is the engineering manual for the keypad. But the **Smart Wallet Program** is the actual *computer brain* inside the vault that processes every command, checks all the rules, and ensures everything is secure.

This program is the **core smart contract** residing directly on the Solana blockchain. It defines the complete logic and rules for how a Goki Smart Wallet operates. When you use the [Goki SDK](04_goki_sdk_.md) or [SmartWalletWrapper](01_smartwalletwrapper_.md) to propose, approve, or execute a transaction, you're ultimately sending instructions to this Smart Wallet Program.

**What problem does it solve?** It provides a decentralized, transparent, and unchangeable set of rules for managing shared digital assets. Because it lives on the blockchain, everyone can see its code, and once deployed, its rules cannot be changed without another multi-signature process, providing maximum security and trust.

## Key Concepts: What is a Program on Solana?

Before diving deeper, let's quickly clarify what a "program" means in the context of Solana:

*   **Like an App on a Smartphone**: Think of a program on Solana as an application or software that runs directly on the blockchain. Just as an app has specific functions (like "send message" or "take photo"), a Solana program has specific instructions it can execute (like "create a Smart Wallet" or "execute a transaction").
*   **Written in Rust**: Most Solana programs, including Goki's Smart Wallet Program, are written in a programming language called Rust.
*   **Immutable and Public**: Once a program is deployed to the Solana blockchain, its code is generally unchangeable and publicly visible. This transparency is a key security feature, as anyone can verify its logic.
*   **Manages Accounts**: Programs interact with "accounts" on the blockchain. Accounts are like files or storage locations where data is kept. The Smart Wallet Program manages two main types of accounts:
    *   The `SmartWallet` account: Stores the wallet's main configuration (like [Owners & Threshold](02_owners___threshold_.md) and [Timelock](03_timelock_.md) settings).
    *   The [Transaction State Account](06_transaction_state_account_.md): Stores the details of a specific proposed transaction, including its approvals.

### The Program's Role in Goki

The Smart Wallet Program is responsible for:

| Feature                   | Program's Role                                                                                                                                                                                          |
| :------------------------ | :------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| **Wallet Creation**       | It creates the `SmartWallet` account, storing the initial [Owners & Threshold](02_owners___threshold_.md) list, and the [Timelock](03_timelock_.md) `minimum_delay`.                                      |
| **Transaction Proposal**  | It creates a [Transaction State Account](06_transaction_state_account_.md) for each proposed action, recording the proposed instructions and the `eta` ([Timelock](03_timelock_.md)).                  |
| **Approval Enforcement**  | When an `owner` approves, it checks if the owner is valid and updates the `signers` list within the [Transaction State Account](06_transaction_state_account_.md).                                       |
| **Threshold Check**       | Before execution, it verifies that the `num_signers` on a transaction meets or exceeds the `threshold` set in the `SmartWallet` account.                                                                 |
| **Timelock Enforcement**  | Before execution, it checks if the current time has passed the `eta` set for the transaction. If not, it prevents execution.                                                                            |
| **Instruction Execution** | Once all conditions (threshold, timelock) are met, it directly executes the instructions stored within the [Transaction State Account](06_transaction_state_account_.md) on behalf of the Smart Wallet. |

## How the Smart Wallet Program Works with Your Commands

As a developer using Goki, you don't directly write Rust code to interact with the Smart Wallet Program. Instead, you use the convenient functions provided by the [Goki SDK](04_goki_sdk_.md) and the [SmartWalletWrapper](01_smartwalletwrapper_.md). These tools then construct the proper messages (called "instructions") and send them to the Smart Wallet Program on the Solana blockchain.

Let's revisit our example of creating a Smart Wallet and executing a transaction, but this time, focusing on the Smart Wallet Program's perspective:

### 1. Creating a Smart Wallet

When you call `sdk.newSmartWallet`, the [Goki SDK](04_goki_sdk_.md) sends an instruction to the Smart Wallet Program.

```typescript
// From Chapter 4: Goki SDK
// ...
const { smartWalletWrapper, tx } = await sdk.newSmartWallet({
  numOwners: owners.length,
  owners,
  threshold,
  delay: delaySeconds, // The program will store this
});
await tx.confirm();
console.log("New Smart Wallet created by SDK:", smartWalletWrapper.key.toBase58());
```
*Program's Action*: The Smart Wallet Program receives this instruction. It then creates a new `SmartWallet` account on the blockchain, storing the `owners` list, the `threshold`, and the `delay` (minimum timelock) inside this account. This defines the fundamental security rules for your new Smart Wallet.

### 2. Proposing a Transaction

When you call `smartWalletWrapper.newTransaction`, the [SmartWalletWrapper](01_smartwalletwrapper_.md) sends another instruction to the Smart Wallet Program.

```typescript
// From Chapter 1: SmartWalletWrapper
// ...
const { transactionKey, tx: proposeTx } =
  await smartWalletWrapper.newTransaction({
    proposer: ownerA.publicKey,
    instructions: [transferInstruction],
    eta, // The program will store this
  });
proposeTx.signers.push(ownerA);
await proposeTx.confirm();
console.log("Transaction proposed:", transactionKey.toBase58());
```
*Program's Action*: The Smart Wallet Program receives the proposal. It creates a new [Transaction State Account](06_transaction_state_account_.md) to hold the details of this pending transaction. It records the proposed `instructions`, the `proposer`, and the `eta` ([Timelock](03_timelock_.md)) for this specific transaction. It also marks `ownerA` as having approved it.

### 3. Approving a Transaction

When `smartWalletWrapper.approveTransaction` is called by another owner:

```typescript
// From Chapter 1: SmartWalletWrapper
// ...
await smartWalletWrapper
  .approveTransaction(transactionKey, ownerB.publicKey)
  .addSigners(ownerB)
  .confirm();
console.log("Transaction approved by Owner B.");
```
*Program's Action*: The Smart Wallet Program checks if `ownerB` is indeed one of the authorized [Owners](02_owners___threshold_.md) of this Smart Wallet. If yes, it updates the `signers` list within the [Transaction State Account](06_transaction_state_account_.md) to reflect `ownerB`'s approval. It now checks if the number of `signers` meets the `threshold` for the Smart Wallet.

### 4. Executing a Transaction

When `smartWalletWrapper.executeTransaction` is called:

```typescript
// From Chapter 1: SmartWalletWrapper
// ...
await smartWalletWrapper
  .executeTransaction({
    transactionKey,
    owner: ownerC.publicKey,
  })
  .addSigners(ownerC)
  .confirm();
console.log("Transaction executed successfully!");
```
*Program's Action*: This is where the Smart Wallet Program performs its most critical checks:
1.  **Threshold Check**: It first checks the [Transaction State Account](06_transaction_state_account_.md) to see if `num_signers` (total approvals) is greater than or equal to the `threshold` stored in the `SmartWallet` account. If not, it stops.
2.  **Timelock Check**: It checks if the current time on the blockchain has passed the `eta` stored in the [Transaction State Account](06_transaction_state_account_.md). If the `eta` is in the future, it stops.
3.  **Execution**: If both checks pass, the Smart Wallet Program then takes the `instructions` from the [Transaction State Account](06_transaction_state_account_.md) and executes them directly on the Solana blockchain. This is when the SOL is actually sent, or other actions occur. It then marks the transaction as "executed" to prevent it from being run again.

## Under the Hood: The Program in Action

The Smart Wallet Program (`programs/smart-wallet/src/lib.rs`) is the central piece of logic that makes Goki work. All the high-level commands from the [Goki SDK](04_goki_sdk_.md) translate into specific *instructions* that the program processes.

Here's a simplified view of how your commands flow to the Smart Wallet Program:

```mermaid
sequenceDiagram
    participant YourApp
    participant GokiSDK
    participant SolanaNetwork
    participant SmartWalletProgram
    participant SmartWalletAccount
    participant TransactionStateAccount

    YourApp->>GokiSDK: 1. Call high-level function<br>(e.g., newSmartWallet)
    GokiSDK->>SolanaNetwork: 2. Send transaction with instruction<br>(e.g., create_smart_wallet)
    SolanaNetwork->>SmartWalletProgram: 3. Invoke instruction handler
    SmartWalletProgram->>SmartWalletAccount: 4. Create/Update wallet data<br>(e.g., owners, threshold)
    SmartWalletAccount-->>SmartWalletProgram: 5. Data stored
    SmartWalletProgram-->>SolanaNetwork: 6. Acknowledge completion
    SolanaNetwork-->>GokiSDK: 7. Confirm transaction
    GokiSDK-->>YourApp: 8. Return result (e.g., SmartWalletWrapper)

    YourApp->>GokiSDK: 9. Call another function<br>(e.g., approveTransaction)
    GokiSDK->>SolanaNetwork: 10. Send transaction with instruction<br>(e.g., approve)
    SolanaNetwork->>SmartWalletProgram: 11. Invoke instruction handler
    SmartWalletProgram->>TransactionStateAccount: 12. Update approvals
    TransactionStateAccount-->>SmartWalletProgram: 13. Data updated
    SmartWalletProgram-->>SolanaNetwork: 14. Acknowledge completion
    SolanaNetwork-->>GokiSDK: 15. Confirm transaction
    GokiSDK-->>YourApp: 16. Confirmation
```
*Explanation*: Each action you take through the [Goki SDK](04_goki_sdk_.md) eventually gets turned into a specific instruction for the `SmartWalletProgram`. The program then reads or writes data to the relevant accounts (`SmartWalletAccount` or [Transaction State Account](06_transaction_state_account_.md)) on the blockchain, enforcing all the rules we've discussed.

### Diving into the Code (Simplified Rust Snippets)

Let's look at some highly simplified code snippets from the core Smart Wallet Program to see where the rules are defined and enforced. These files are located under `programs/smart-wallet/src/`.

**1. Program's On-Chain Identity (`lib.rs`)**
Every Solana program has a unique identifier on the blockchain.

```rust
// programs/smart-wallet/src/lib.rs
// ...
declare_id!("GokivDYuQXPZCWRkwMhdH2h91KpDQXBEmpgBgs55bnpH"); // The unique address of this program
// ...
```
*Explanation*: This line declares the unique public key (address) where the Smart Wallet Program lives on the Solana blockchain. When the [Goki SDK](04_goki_sdk_.md) wants to talk to the Smart Wallet Program, it uses this address.

**2. Storing Smart Wallet Data (`state.rs`)**
The `SmartWallet` struct defines the data that gets stored for each Goki Smart Wallet you create.

```rust
// programs/smart-wallet/src/state.rs
pub struct SmartWallet {
    pub base: Pubkey,
    pub bump: u8,
    pub threshold: u64,       // Min approvals needed
    pub minimum_delay: i64,   // Min timelock delay
    pub owner_set_seqno: u32,
    pub num_transactions: u64,
    pub owners: Vec<Pubkey>,  // List of authorized owners
    pub reserved: [u64; 16],
}
// ...
```
*Explanation*: This `SmartWallet` struct is the blueprint for how your Smart Wallet's core configuration is stored on the blockchain. Notice it contains `threshold`, `minimum_delay`, and `owners` – the very concepts we discussed in previous chapters!

**3. Storing Transaction Data (`state.rs`)**
The `Transaction` struct defines the data for each proposed transaction.

```rust
// programs/smart-wallet/src/state.rs
pub struct Transaction {
    pub smart_wallet: Pubkey,
    pub index: u64,
    pub bump: u8,
    pub proposer: Pubkey,
    pub instructions: Vec<TXInstruction>, // The actual actions to perform
    pub signers: Vec<bool>,              // Which owners have approved
    pub owner_set_seqno: u32,
    pub eta: i64,                        // Estimated execution time (Timelock)
    pub executor: Pubkey,
    pub executed_at: i64,
}
// ...
```
*Explanation*: This `Transaction` struct holds all the details about a proposed operation, including the actual `instructions` to be executed, which `owners` have signed (`signers`), and the `eta` for [Timelock](03_timelock_.md).

**4. Creating a Smart Wallet (`lib.rs`)**
This is how the program handles the initial setup.

```rust
// programs/smart-wallet/src/lib.rs
pub fn create_smart_wallet(
    ctx: Context<CreateSmartWallet>,
    _bump: u8,
    max_owners: u8,
    owners: Vec<Pubkey>,      // Input owners
    threshold: u64,           // Input threshold
    minimum_delay: i64,       // Input min delay
) -> Result<()> {
    let smart_wallet = &mut ctx.accounts.smart_wallet;
    smart_wallet.threshold = threshold;       // Program stores the threshold
    smart_wallet.minimum_delay = minimum_delay; // Program stores the min delay
    smart_wallet.owners = owners.clone();     // Program stores the owners
    // ... other initialization ...
    Ok(())
}
// ...
```
*Explanation*: When you create a new Smart Wallet, this function in the program takes your `owners`, `threshold`, and `minimum_delay` and writes them directly into the `smart_wallet` account on the blockchain, making them permanent.

**5. Approving a Transaction (`instructions/approve.rs`)**
This function updates the approvals for a proposed transaction.

```rust
// programs/smart-wallet/src/instructions/approve.rs
pub fn handler(ctx: Context<Approve>) -> Result<()> {
    let owner_index = ctx
        .accounts
        .smart_wallet
        .try_owner_index(ctx.accounts.owner.key())?; // Checks if signer is an owner
    ctx.accounts.transaction.signers[owner_index] = true; // Mark as approved
    // ... emit event ...
    Ok(())
}
// ...
```
*Explanation*: When an owner approves, the program first verifies that the signer is indeed a valid owner. Then, it simply sets the corresponding `true` in the `signers` list for that owner in the [Transaction State Account](06_transaction_state_account_.md).

**6. Executing a Transaction (`lib.rs`)**
This is the core logic that checks all conditions before executing instructions.

```rust
// programs/smart-wallet/src/lib.rs
pub fn execute_transaction(ctx: Context<ExecuteTransaction>) -> Result<()> {
    let smart_wallet = &ctx.accounts.smart_wallet;
    let tx = &mut ctx.accounts.transaction;
    let clock = Clock::get()?; // Get current blockchain time
    let current_ts = clock.unix_timestamp;

    // Check if timelock has passed
    invariant!(current_ts >= tx.eta, TransactionNotReady);

    // Do we have enough signers to execute the TX?
    let sig_count = tx.num_signers();
    invariant!((sig_count as u64) >= smart_wallet.threshold, NotEnoughSigners);

    // Execute the actual instructions
    for ix in tx.instructions.iter() {
        solana_program::program::invoke_signed(&(ix).into(), ctx.remaining_accounts, seeds)?;
    }
    tx.executed_at = current_ts; // Mark as executed
    // ... emit event ...
    Ok(())
}
// ...
```
*Explanation*: This snippet shows the `execute_transaction` function. It performs the crucial `invariant!` checks to ensure that the [Timelock](03_timelock_.md) has passed (`current_ts >= tx.eta`) and that enough owners have approved (`sig_count >= smart_wallet.threshold`). Only if both conditions are met does it proceed to `invoke_signed`, which actually runs the proposed instructions! It also marks the transaction as executed.

## Conclusion

The **Smart Wallet Program** is the secure, rule-enforcing engine that powers every Goki Smart Wallet. It's the central smart contract on the Solana blockchain that guarantees the multi-signature and timelock rules are always followed. While you interact with it through the user-friendly [Goki SDK](04_goki_sdk_.md) and [SmartWalletWrapper](01_smartwalletwrapper_.md), understanding its role as the on-chain "brain" is key to appreciating the robust security Goki offers.

In the next chapter, we'll delve deeper into the [Transaction State Account](06_transaction_state_account_.md), the specific place on the blockchain where the Smart Wallet Program stores all the information about your proposed and ongoing multi-signature transactions.

---
<sub><sup>**References**: [[1]](https://github.com/GokiProtocol/goki/blob/87aff0569301acd16f3bdcbfec09cae6ba3e62cc/programs/smart-wallet/Cargo.toml), [[2]](https://github.com/GokiProtocol/goki/blob/87aff0569301acd16f3bdcbfec09cae6ba3e62cc/programs/smart-wallet/README.md), [[3]](https://github.com/GokiProtocol/goki/blob/87aff0569301acd16f3bdcbfec09cae6ba3e62cc/programs/smart-wallet/src/events.rs), [[4]](https://github.com/GokiProtocol/goki/blob/87aff0569301acd16f3bdcbfec09cae6ba3e62cc/programs/smart-wallet/src/instructions/approve.rs), [[5]](https://github.com/GokiProtocol/goki/blob/87aff0569301acd16f3bdcbfec09cae6ba3e62cc/programs/smart-wallet/src/instructions/unapprove.rs), [[6]](https://github.com/GokiProtocol/goki/blob/87aff0569301acd16f3bdcbfec09cae6ba3e62cc/programs/smart-wallet/src/lib.rs), [[7]](https://github.com/GokiProtocol/goki/blob/87aff0569301acd16f3bdcbfec09cae6ba3e62cc/programs/smart-wallet/src/state.rs), [[8]](https://github.com/GokiProtocol/goki/blob/87aff0569301acd16f3bdcbfec09cae6ba3e62cc/programs/smart-wallet/src/validators.rs)</sup></sub>