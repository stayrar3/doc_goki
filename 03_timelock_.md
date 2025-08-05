# Chapter 3: Timelock

Welcome back to Goki! In the previous chapter, [Owners & Threshold](02_owners___threshold_.md), we learned how your Goki Smart Wallet sets up shared control: who can make decisions (owners) and how many need to agree (threshold). These are powerful for security, but what if a decision is approved that turns out to be a mistake or even malicious?

This is where **Timelock** comes in. It's an extra layer of security that acts like a mandatory "cooling-off" period.

## The Bank Account Safety Net

Let's go back to our shared bank account analogy. Imagine you and your friends approve a large payment. But just after everyone agrees, you realize someone made a typo in the amount, or even worse, that the payment address is wrong!

Without a safety net, that money would be sent instantly. But what if the bank had a rule: "Any payment over $1,000 must wait 24 hours before it's processed"? That 24-hour window gives you crucial time to detect the error and cancel the payment.

On the Solana blockchain, `Timelock` provides this exact safety net for your Goki Smart Wallet. It's a delay between when a transaction is approved by the required number of [Owners & Threshold](02_owners___threshold_.md) and when it can actually be executed on the blockchain.

## Key Concepts: Timelock

`Timelock` introduces two important values:

1.  **Minimum Delay**: This is a setting for the *entire* Smart Wallet. It's the shortest amount of time (in seconds) that *any* transaction must wait after being approved, before it can be executed. Think of it as a base security policy for your wallet.

2.  **Estimated Time of Arrival (ETA)**: This is a specific time (a timestamp in seconds) set for *each individual transaction*. When you propose a transaction, you can specify its `ETA`. The `ETA` must be at least the current time plus the Smart Wallet's `minimumDelay`. A transaction cannot be executed until its `ETA` has passed.

Why both? The `minimumDelay` ensures that even if you forget to set an `ETA` for a specific transaction, there's always a baseline delay for the entire wallet. The `ETA` gives you flexibility to set longer delays for particularly sensitive operations.

## Using Timelock in Goki

Let's see how we can add this crucial safety feature when creating our Smart Wallet and when proposing transactions.

### 1. Setting the Smart Wallet's Minimum Delay

You define the `minimumDelay` when you first create your Goki Smart Wallet. This sets a default waiting period for all future transactions.

```typescript
import { Keypair, PublicKey, LAMPORTS_PER_SOL } from "@solana/web3.js";
import { makeSDK } from "./workspace";
import BN from "bn.js";

const sdk = makeSDK();
const ownerA = Keypair.generate();
const owners = [ownerA.publicKey];
const threshold = new BN(1); // 1-of-1 multisig for simplicity

const fiveSeconds = new BN(5); // Our minimum delay: 5 seconds

let smartWalletWrapper;

// Create a new Smart Wallet with a minimum delay
const { smartWalletWrapper: wrapperInner, tx } = await sdk.newSmartWallet({
  numOwners: owners.length,
  owners,
  threshold,
  delay: fiveSeconds, // <-- Here's the new 'delay' parameter!
});
await tx.confirm();
smartWalletWrapper = wrapperInner;
console.log("Smart Wallet created with 5s delay:", smartWalletWrapper.key.toBase58());
```
*Explanation*: We've added a `delay` parameter to `sdk.newSmartWallet`. This tells the Goki program that any transaction for this `smartWalletWrapper` will have a minimum waiting period of 5 seconds after approval.

### 2. Proposing a Transaction with an ETA

When you propose a new transaction, you can specify an `eta` (Estimated Time of Arrival). This `eta` must be at least the current time plus the `minimumDelay` you set for the wallet.

```typescript
import { SystemProgram } from "@solana/web3.js";
// ... (previous code for smartWalletWrapper)
import { sleep } from "@saberhq/token-utils"; // Need this for delay

const recipient = Keypair.generate().publicKey;
const amountToSend = 0.01 * LAMPORTS_PER_SOL;

// Calculate ETA: Current time + Smart Wallet's minimum delay + some buffer
const smartWalletData = await smartWalletWrapper.reloadData(); // Get current data
const minDelay = smartWalletData.minimumDelay.toNumber();
const eta = new BN(Date.now() / 1000 + minDelay + 1); // Add 1 second buffer

const transferInstruction = SystemProgram.transfer({
  fromPubkey: smartWalletWrapper.key,
  toPubkey: recipient,
  lamports: amountToSend,
});

const { transactionKey, tx: proposeTx } =
  await smartWalletWrapper.newTransaction({
    proposer: ownerA.publicKey,
    instructions: [transferInstruction],
    eta, // <-- Here we set the ETA!
  });
proposeTx.signers.push(ownerA);
await proposeTx.confirm();
console.log("Transaction proposed with ETA:", transactionKey.toBase58());
```
*Explanation*: We calculate `eta` by taking the current time, adding the Smart Wallet's `minimumDelay` (which is 5 seconds in our example), and a small buffer. This ensures the `eta` is valid. We then pass this `eta` when proposing the transaction. Since our threshold is 1-of-1, `ownerA` proposing also means it's approved.

### 3. Attempting to Execute Before Timelock Expires

Now, let's try to execute this transaction immediately. Since we have a `minimumDelay` and an `eta`, it should fail if the `eta` hasn't passed yet.

```typescript
// ... (previous code for transactionKey)

// Try to execute immediately
try {
  await smartWalletWrapper
    .executeTransaction({
      transactionKey,
      owner: ownerA.publicKey,
    })
    .addSigners(ownerA)
    .confirm();
  console.log("This should not be printed!");
} catch (e) {
  // We expect this to fail because the timelock hasn't passed
  console.log("Failed to execute (as expected):", (e as Error).message);
}
```
*Explanation*: When `executeTransaction` is called, the Goki Smart Wallet program checks if the `eta` for this specific transaction has passed. Since we're trying to execute it immediately, it will likely fail with a "Transaction Not Ready" error.

### 4. Executing After Timelock Expires

To successfully execute, we need to wait until the `eta` has passed.

```typescript
// ... (previous code for transactionKey, eta)

// Wait until the ETA has passed
const timeToWait = eta.toNumber() - (Date.now() / 1000) + 1; // Add 1 sec buffer
console.log(`Waiting for ${timeToWait.toFixed(0)} seconds for Timelock...`);
await sleep(timeToWait * 1000); // Sleep function from @saberhq/token-utils

// Now, execute the transaction
await smartWalletWrapper
  .executeTransaction({
    transactionKey,
    owner: ownerA.publicKey,
  })
  .addSigners(ownerA)
  .confirm();
console.log("Transaction executed successfully after Timelock!");
// The 0.01 SOL has now been sent!
```
*Explanation*: We use a `sleep` function to pause our program until the `eta` is definitely in the past. After waiting, when `executeTransaction` is called again, the Goki program will see that the `eta` condition is met, and the transaction will proceed successfully.

## Under the Hood: How Timelock Works

Timelock is enforced directly by the [Smart Wallet Program](05_smart_wallet_program_.md) on the blockchain. Here's a simplified look at what happens:

```mermaid
sequenceDiagram
    participant User
    participant SmartWalletWrapper
    participant SolanaNetwork
    participant SmartWalletProgram
    participant SmartWalletAccount
    participant TransactionStateAccount

    User->>SmartWalletWrapper: 1. Call newSmartWallet(delay)
    SmartWalletWrapper->>SolanaNetwork: 2. Send 'Create SW' instruction
    SolanaNetwork->>SmartWalletProgram: 3. Invoke Smart Wallet Program
    SmartWalletProgram->>SmartWalletAccount: 4. Store 'minimum_delay'
    SmartWalletAccount-->>SmartWalletProgram: 5. Data saved

    User->>SmartWalletWrapper: 6. Call newTransaction(eta)
    SmartWalletWrapper->>SolanaNetwork: 7. Send 'Create TX with Timelock' instruction
    SolanaNetwork->>SmartWalletProgram: 8. Invoke Smart Wallet Program
    Note over SmartWalletProgram: 9. Check if eta >= current_time + minimum_delay
    SmartWalletProgram->>TransactionStateAccount: 10. Store 'eta'
    TransactionStateAccount-->>SmartWalletProgram: 11. Data saved

    User->>SmartWalletWrapper: 12. Call executeTransaction()
    SmartWalletWrapper->>SolanaNetwork: 13. Send 'Execute TX' instruction
    SolanaNetwork->>SmartWalletProgram: 14. Invoke Smart Wallet Program
    Note over SmartWalletProgram: 15. Check if current_time >= eta
    SmartWalletProgram--xSolanaNetwork: 16. If not, fail (TransactionNotReady error)
    SolanaNetwork--xSmartWalletWrapper: 17. Error returned
```
*Explanation*:
1.  **Wallet Creation**: When you create a Smart Wallet with a `delay`, the `SmartWalletWrapper` tells the [Smart Wallet Program](05_smart_wallet_program_.md) to store this `minimum_delay` value directly in your `SmartWallet`'s data on the blockchain ([SmartWalletAccount](05_smart_wallet_program_.md)).
2.  **Transaction Proposal**: When you propose a transaction with an `eta`, the `SmartWalletWrapper` sends a `create_transaction_with_timelock` instruction to the [Smart Wallet Program](05_smart_wallet_program_.md). The program first verifies that the `eta` you provided is valid (meaning it's in the future and meets the `minimum_delay` requirement). If valid, it stores this `eta` in the new [Transaction State Account](06_transaction_state_account_.md).
3.  **Transaction Execution**: When anyone tries to `executeTransaction`, the [Smart Wallet Program](05_smart_wallet_program_.md) checks the `eta` stored in the [Transaction State Account](06_transaction_state_account_.md). If the current time on the blockchain (provided by the `Clock` sysvar) is *before* the `eta`, the program stops the execution and returns an error (`TransactionNotReady`). Only once the `eta` has passed will the program allow the transaction to proceed.

### Diving into the Code (Simplified)

Let's look at how the `minimum_delay` and `eta` are handled within the core [Smart Wallet Program](05_smart_wallet_program_.md) written in Rust.

First, storing the `minimum_delay` in the `SmartWallet` account (`programs/smart-wallet/src/state.rs`):

```rust
// Simplified from programs/smart-wallet/src/state.rs

/// A [SmartWallet] is a multisig wallet with Timelock capabilities.
#[account]
#[derive(Default, Debug, PartialEq)]
pub struct SmartWallet {
    // ... other fields ...

    /// Minimum delay between approval and execution, in seconds.
    pub minimum_delay: i64,

    // ... other fields ...
}
```
*Explanation*: The `SmartWallet` struct, which defines the data for your Smart Wallet on the blockchain, includes a `minimum_delay` field to store this setting.

And storing the `eta` in the `Transaction` account (`programs/smart-wallet/src/state.rs`):

```rust
// Simplified from programs/smart-wallet/src/state.rs

/// A [Transaction] is a series of instructions that may be executed
/// by a [SmartWallet].
#[account]
#[derive(Debug, Default, PartialEq)]
pub struct Transaction {
    // ... other fields ...

    /// Estimated time the [Transaction] will be executed.
    pub eta: i64,

    // ... other fields ...
}
```
*Explanation*: Each `Transaction` (our proposed action) gets its own `eta` field, storing the estimated time when it can be executed.

Now, let's see how these are set and checked (`programs/smart-wallet/src/lib.rs`):

```rust
// Simplified from programs/smart-wallet/src/lib.rs

#[program]
pub mod smart_wallet {
    use super::*;

    pub fn create_smart_wallet(
        ctx: Context<CreateSmartWallet>,
        _bump: u8,
        max_owners: u8,
        owners: Vec<Pubkey>,
        threshold: u64,
        minimum_delay: i64, // <-- input delay
    ) -> Result<()> {
        let smart_wallet = &mut ctx.accounts.smart_wallet;
        smart_wallet.minimum_delay = minimum_delay; // <-- stored here!
        // ... other initialization ...
        Ok(())
    }

    pub fn create_transaction_with_timelock(
        ctx: Context<CreateTransaction>,
        _bump: u8,
        instructions: Vec<TXInstruction>,
        eta: i64, // <-- input ETA for the transaction
    ) -> Result<()> {
        let smart_wallet = &ctx.accounts.smart_wallet;
        let clock = Clock::get()?;
        let current_ts = clock.unix_timestamp;

        // Check against Smart Wallet's minimum delay
        if smart_wallet.minimum_delay != 0 {
            invariant!(
                eta >= unwrap_int!(current_ts.checked_add(smart_wallet.minimum_delay as i64)),
                InvalidETA // Error if ETA is too soon
            );
        }
        // ... more ETA validation ...

        let tx = &mut ctx.accounts.transaction;
        tx.eta = eta; // <-- stored in the Transaction account!
        // ... other transaction initialization ...
        Ok(())
    }

    pub fn execute_transaction(ctx: Context<ExecuteTransaction>) -> Result<()> {
        let tx = &ctx.accounts.transaction;
        let clock = Clock::get()?;
        let current_ts = clock.unix_timestamp;

        // Check if timelock has passed
        if tx.eta != NO_ETA && current_ts < tx.eta {
            return err!(ErrorCode::TransactionNotReady); // <-- Error if not ready!
        }
        // ... other execution logic ...
        Ok(())
    }
}
```
*Explanation*:
1.  **`create_smart_wallet`**: The `minimum_delay` provided during wallet creation is directly saved into the `smart_wallet` account.
2.  **`create_transaction_with_timelock`**: This function receives the `eta` for the new transaction. It performs crucial checks:
    *   It ensures the `eta` is at least the current time plus the `smart_wallet.minimum_delay`. If not, it returns `InvalidETA`.
    *   Once validated, the `eta` is saved into the `transaction` account.
3.  **`execute_transaction`**: Before executing the instructions, this function fetches the `eta` from the `transaction` account. It compares it with the `current_ts` (current time on the blockchain). If `current_ts` is less than `tx.eta` (and `eta` is not `NO_ETA`, which means no timelock), it returns `TransactionNotReady`, preventing early execution.

This on-chain enforcement makes Timelock a robust security feature, ensuring that even if your application or users make a mistake, the blockchain itself will prevent the transaction from being processed prematurely.

## Conclusion

**Timelock** adds a powerful time-based security layer to your Goki Smart Wallet. By introducing a minimum delay between approval and execution, it provides a crucial window to detect and prevent errors or malicious activities, significantly enhancing the safety of your digital assets. This feature is especially valuable for high-value operations where even a small mistake could have large consequences.

In the next chapter, we'll shift our focus to the [Goki SDK](04_goki_sdk_.md) itself, understanding how it provides the tools and functions to easily interact with all these Smart Wallet features from your applications.

---
 <sub><sup>**References**: [[1]](https://github.com/GokiProtocol/goki/blob/87aff0569301acd16f3bdcbfec09cae6ba3e62cc/programs/smart-wallet/src/lib.rs), [[2]](https://github.com/GokiProtocol/goki/blob/87aff0569301acd16f3bdcbfec09cae6ba3e62cc/programs/smart-wallet/src/state.rs), [[3]](https://github.com/GokiProtocol/goki/blob/87aff0569301acd16f3bdcbfec09cae6ba3e62cc/src/wrappers/smartWallet/types.ts), [[4]](https://github.com/GokiProtocol/goki/blob/87aff0569301acd16f3bdcbfec09cae6ba3e62cc/tests/smartWallet.spec.ts)</sup></sub>