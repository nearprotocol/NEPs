- Proposal Name: auto-unlock-with-safes
- Start Date: 2019-12-10
- NEP PR: [nearprotocol/neps#26](https://github.com/nearprotocol/neps/pull/26)
- Issue(s):  [nearprotocol/neps#23](https://github.com/nearprotocol/neps/pull/23), [nearprotocol/neps#24](https://github.com/nearprotocol/neps/pull/24)

# Summary
[summary]: #summary

Introducing a new concept of safes that allows to securely lock some data from a contract with
automatic unlock mechanism.

# Motivation
[motivation]: #motivation

There are a few NEPs that proposed solutions to address the cross-shard communication problem.
For example when an decentralized exchange tries to swap 2 tokens without owning them.

One solution is to introduce locks with automatic unlock capabilities, but without explicitly exposing the locked structure.
While it solves the issue for a simple exchange use-case. It has some limitations and complexity for non trivial use cases.

It might lead to an unexpected behavior when a token is behind the proxy contract.

This proposal is to introduce explicit locked data storage which we call a `safe` that can't be copied and always resolved at the end.
It's a familiar concept for an asynchronous development similar to guards.
When a guard is released, the destructor (or Drop in Rust) is called and the lock can be resolved.


# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

We introduce a new concept which we call a `safe`.

- Only the owner contract can create a safe.
- The safe has one owner.
- A safe always has a promise attached on it that calls the owner to release the safe (unlock). 
- Safe can't be cloned or copied, so there are always only 1 instance of a particular safe.
- Non-owner contracts can pass safes to other contracts and read the content, but can't modify the content of the safe.
- Owner contract can modify the content of the safe, but not the automatic release promise.

### Example:

When a decentralized exchange tries to move some tokens, it first has to acquire and lock the funds.
- Dex calls token contract to lock 1000 tokens for `alice`.
- Token contract creates a safe with the following data:
```
{
  owner_id: "alice",
  amount: 1000,
  locked_by: "dex",
}
```
- Token contract creates a promise to release the safe, e.g. by calling `unlock` on itself.
- Token returns OK and this safe to the `dex`.

Now Dex has this safe from the token contract.
Dex can read the content of the safe and assert the content is correct.

- Dex calls `transfer` on the token contract and pass this safe with this promise.
- Token contract reads the content of the safe that it received and transfers the required amount `400` to the new owner.
- Token contract modifies the content of the safe and decreased the safe amount by `400` by the transferred amount. E.g.
```
{
  owner_id: "alice",
  amount: 600,
  locked_by: "dex",
}
```
- Token contract can now return OK and the safe back to `dex`.

Transfer has completed successfully, but `dex` may want to do more transfers. It's safe to drop the safe now.

- Dex returns OK and drops the safe.
- When the safe is dropped (not passed to any contract and not returned), it calls the associated promise on the token contract.

NOTE, that the promise is always called even if the content of the safe was fully used.
It's because the promise is fully prepaid during the creation of the safe.

- Token contract checks the content of the safe and if there are still funds, it can return them back to `alice`.

# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

## Runtime API

Introducing new Runtime API to handle safes:

### API to create safes and read/write content 
```rust
/// Creates a new safe with the provided `method_name`, `arguments` and `gas` for the unlock callback.
/// Returns the Safe index.
/// 
/// NOTE: Internally it crease a new promise, that is associated with the given safe.
/// You can't reuse this promise for anything else.
/// The promise will be called only when the safe is dropped.
/// The content of the safe will be given as the first promise result.
fn safe_create(method_name_len: u64,
               method_name_ptr: u64,
               arguments_len: u64,
               arguments_ptr: u64,
               gas: u64) -> u64;

/// Writes the content to the safe identified by `safe_idx`.
/// Can only be called by the safe owner. 
fn safe_content_write(safe_idx: u64, value_len: u64, value_ptr: u64);

/// Reads the content from the safe identified by `safe_idx` into a `register_id`. 
fn safe_content_read(safe_idx: u64, register_id: u64);

/// Reads the owner account ID of the safe identified by `safe_idx` into a `register_id`. 
fn safe_owner_id(safe_idx: u64, register_id: u64);
```

### Passing safes

If you don't return or pass a safe, then this safe will be dropped at the end of the contract execution.  

```rust
/// Returns the safe identified by `safe_idx` from the contract with the outgoing data.
/// You can return multiple safes from the contract by calling this method on multiple safes.
/// This method consumes the associated safe, so it can't be used afterwards.
/// 
/// The call will panic if the contract has multiple outgoing dependencies.
/// 
/// NOTE: If a contract returns a promise using `promise_return` or there are no outgoing dependencies,
/// the safe will be dropped at the end of the contract execution.
fn safe_return(safe_idx: u64);

/// Attaches the safe identified by `safe_idx` to the promise identified by the `promise_idx`.
/// This method consumes the associated safe, so it can't be used afterwards.
/// The promise can't be a joined promise created using `promise_and`.
fn promise_attach_safe(promise_idx: u64, safe_idx: u64);
```

### Receiving safes 

Safes can be received in two ways:
- Receiving safes attached to the input of this call.
- Receiving safes with a promise result.

```rust
/// Returns the number of safes received with the input for this function.
fn input_safes_count() -> u64;

/// Returns `safe_idx` at the position `input_safe_idx` in the input safes.
fn input_safe(input_safe_idx: u64) -> u64;

/// Returns the number of safes returned with the given result at the position `result_idx`.
fn promise_result_safes_count(result_idx: u64) -> u64;

/// Returns `safe_idx` at the position `result_safe_idx` for a result at the position `result_idx`.
/// For example, there are 3 results:
/// - result #0 has 2 safes with safe idx: [0, 1]
/// - result #1 has no safes: []
/// - result #2 has 1 safes with safe idx: [2]
/// Call results will be:
/// ```
/// assert_eq!(promise_result_safes_count(0), 2);
/// assert_eq!(promise_result_safe(0, 0), 0);
/// assert_eq!(promise_result_safe(0, 1), 1);
/// assert_eq!(promise_result_safe(2, 0), 2);
/// ```
fn promise_result_safe(result_idx: u64, result_safe_idx: u64) -> u64;
```

## Low-level contract example

This is still pseudo-code. But it should highlight how safes work.
E.g. this code don't use registers and assumes core functions return vectors. 

```rust
// Dex contract
impl Dex {
    /// Initiates trades between `owner1` who owns `amount1` tokens of `token1` with
    /// `owner2` who owns `amount2` tokens of `token2`. 
    /// Simplified logic, cause we don't check open orders, permissions, etc.
    pub fn trade(
        owner1: AccountId,
        token1: AccountId,
        amount1: Balance,
        owner2: AccountId,
        token2: AccountId,
        amount2: Balance,
    ) {
        // Locking `amount1` of tokens `token1` from `owner1`
        let promiseLock1 = TokenContract::new(token1).lock(LockArgs {
            owner: owner1,
            amount: amount1,
        }.try_to_vec().unwrap());
        // Locking `amount2` of tokens `token2` from `owner2`
        let promiseLock2 = TokenContract::new(token2).lock(LockArgs {
            owner: owner2,
            amount: amount2,
        }.try_to_vec().unwrap());
        // Join promises, so we can wait on both of them.
        let promisesJoinedLocks = promise_and([promiseLock1, promiseLock2]);
        // Attaching a callback back to us, that will receive results of locks.
        let callback = promise_then(promisesJoinedLocks, current_account_id(), "on_locks", OnLocksArgs {
            owner1,
            token1,
            amount1,
            owner2,
            token2,
            amount2,
        }.try_to_vec().unwrap());
        // Return our callback, so the execution doesn't return result yet.
        promise_return(callback);
    }
    
    /// Callback to process locks received from the token contracts.
    pub fn on_locks(
        owner1: AccountId,
        token1: AccountId,
        amount1: Balance,
        owner2: AccountId,
        token2: AccountId,
        amount2: Balance,
    ) {
        // Check it's a callback by this contract.
        assert_eq!(predecessor_account_id(), current_account_id());
        // Get indices for safes. It would fail if one of the locks failed,
        // So the callback will fail as well. 
        let safe_idx1 = promise_result_safe(0, 0);
        let safe_idx2 = promise_result_safe(1, 0);
        // We can verify safe owners.
        // It's unnecessary, because we trust token contracts.
        assert_eq!(safe_owner(safe_idx1), token1);
        assert_eq!(safe_owner(safe_idx2), token2);
        // We can also check content of the safes. E.g. check the amounts.
        // But we probably shouldn't, because the implementation might be different.
        assert_eq!(TokenSafeContent.try_from_slice(safe_content_read(safe_idx1)).unwrap().amount, amount1);
        assert_eq!(TokenSafeContent.try_from_slice(safe_content_read(safe_idx2)).unwrap().amount, amount2);
        // Now the actual code for transfers.
        
        // Create transfer of `amount1` tokens `token1` to the new owner `owner2`
        let promiseTransfer1 = TokenContract::new(tokenFrom).transfer_with_safe(TransferArgs {
            new_owner: owner2,
            amount: amount1,
        }.try_to_vec().unwrap());
        // Attaching a safe to the new promise. Now we can't use the safe `safe_idx1` anymore.
        promise_attach_safe(promiseTransfer1, safe_idx1);

        // Create transfer of `amount2` tokens `token2` to the new owner `owner1`
        let promiseTransfer2 = TokenContract::new(tokenTo).transfer_with_safe(TransferArgs {
            new_owner: owner1,
            amount: amount2,
        }.try_to_vec().unwrap());
        // Attaching a safe to the new promise. Now we can't use the safe `safe_idx2` anymore.
        promise_attach_safe(promiseTransfer2, safe_idx2);
        
        // We are done. Don't need to return anything or depend on results.
    }
}

// Token contract implementation
impl Token {
    /// Locks tokens from owner
    pub fn lock(
        &mut self,
        owner: AccountId,
        amount: Balance,
    ) {
        // Check the predecessor has enough allowance and token balance.
        self.assertPermission(onwer, amount);
        let safe_idx = safe_create("unlock", UnlockArgs {
            // unlock parameters can go there.
            // We don't need them right now, because content of the safe is enough.
        }.try_to_vec().unwrap(), ENOUGH_GAS);
        // Decrease owner's balance and the allowances
        self.token[owner].balance -= amount;
        self.token[owner].allowances[predecessor_account_id()] -= amount;
        // Create safe content
        safe_content_write(safe_idx, TokenSafeContent {
            owner,
            amount,
            caller: predecessor_account_id(),
        }.try_to_vec().unwrap());
        
        // Return safe from the contract.
        safe_return(safe_idx);
    }
    
    /// Transferring `amount` of tokens to `new_owner` from the given `safe`.
    pub fn transfer_with_safe(
        &mut self,
        new_owner: AccountId,
        amount: Balance,
    ) {
        // Get the safe index first. It will fail, if the safe is not passed.
        let safe_idx = input_safe(0);
        // Check that the safe is from this token contract.
        assert_eq!(safe_owner(safe_idx), current_account_id());
        // Get content of the safe.
        let mut safe = TokenSafeContent.try_from_slice(safe_content_read(safe_idx)).unwrap();
        // Check that the safe has enough amount;
        assert!(safe.amount >= amount);
        // Transfer tokens from safe to the new owner balance.
        safe.amount -= amount;
        self.token[new_owner].balance += amount;
        // Update safe content
        safe_content_write(safe_idx, safe.try_to_vec().unwrap());
        
        // We are done. The safe will be dropped and the remaining balance returned to the owner.
    }
    
    /// Unlocks the safe. Returns the remaining balance back to the owner and the remaining
    /// allowance back to the caller.
    /// NOTE: The content of the safe is given as the first promise result.
    pub fn unlock(&mut self) {
        // Check that it's a callback by this contract.
        assert_eq!(predecessor_account_id(), current_account_id());
        // The content of the safe is given as the first promise result.
        let safe = TokenSafeContent.try_from_slice(promise_result(0)).unwrap();
        if safe.amount > 0 {
            self.token[safe.owner].balance += safe.amount;
            self.token[safe.owner].allowances[safe.caller] += safe.amount;
        }
    }
}
```

## On access to safes from multiple actions.

Since safes are passed to a promise and not to a particular function call.
Let's say a promise contains 2 function calls.
All safe(s) will be given to the first function call. If the function call doesn't consume a safe, it will be passed towards the next function call.
Once the safe reaches the last action and the safe is not consumed by the last action, the safe will be dropped.

If the content of the safe is modified by the contract during one of the function call, but then the next action fails. The content of the safe
is reverted to the original content, the content before the first action has started.
   
## Runtime internal implementation

To handle safes properly we need the following:
 - To track safes across actions. This includes:
   - To track safe owner.
   - To track current safe content and the original safe content.
   - To track whether the safe was created during this action was passed in.
   - To track which promise to call when the safe is dropped.
 - Providing safes for actions and updating them.
 - To pass safes with action receipts (for `promise_attach_safe`)
 - To pass safes with data receipts (for `safe_return`).
 - To resolve all remaining safes that were not consumed.

### Tracking safes

The easiest option to handle safes is to accumulate all safes at the beginning of action receipt processing.
Same way we accumulate input_data, we can accumulate safes with content. Let's introduce `Safe` data structure:
```rust
pub struct Safe {
    /// Account ID of the safe owner. It's the account ID of the contract that created the safe.
    /// It's where the safe content will be sent when the safe is dropped. 
    pub owner_id: AccountId,
    
    /// Unique ID of the data. It's similar `data_id` in `DataReceipt`s and in `DataReceiver`s.
    /// When this safe is dropped, the content of the safe will be sent using (`owner_id`, `data_id`)
    /// within a `DataReceipt`. It'll trigger the `ActionReceipt` on the `owner_id` account that 
    /// will handle unlocking/resolving logic of the safe.
    pub data_id: CryptoHash,
    
    /// Content of the safe. Safe contains an empty vec by default.
    pub content: Vec<u8>,
}
```

Then we add a new vector of `all_safes` into `apply_action_receipt` within a Runtime.
This vector allows us to drop all safes with the original content in case any action fails during
processing of this `ActionReceipt`.
```rust
    /// Contains safes attached to the action receipt and all safes from all promise results.
    let original_safes: Vec<Safe>;
```

```rust
    /// Mutable clone of original safes.
    /// NOTE: It's easier to clone original safes than trying to wrap them
    /// and then clone on demand (e.g. when attaching to receipts). 
    let mut safes: Vec<Safe>;

    /// Safe indices that are attached to this `ActionReceipt`. 
    let mut input_safes_idxs: Vec<SafeIndex>;
    /// Safe indices that were received with the promise results. 
    let mut promise_results_safes_idxs: Vec<Vec<SafeIndex>>;
```

### Processing safes with actions

Each action will receive a mutable reference to `input_safes` and `promise_results_safes`.

If any action fails, then we don't care about `input_safes` and `promise_results_safes` anymore, because
we'll just drop safes from `original_safes`.

A function call on an account that owns a safe may update the content of the safe.
A function call can also consume some safes from either vector, or create new safes and add them to `input_safes`.
Newly created safes that are not consumed will be passed to the next action as an input, so it can act the safe if needed.

The content of the safes will be handled through `RuntimeExt` crate.

Inside a VMLogic, we'll track safes the following way.
```rust
    // Immutable:
    /// Safe indices that are attached to this `ActionReceipt`. 
    input_safes_idxs: Vec<SafeIndex>,
    /// Safe indices that were received with the promise results. 
    promise_results_safes_idxs: Vec<Vec<SafeIndex>>,
    
    // Mutable:
    /// Safe indices that were created during the execution.
    new_safes_idxs: Vec<SafeIndex>,
    /// Safe indices that were consumed during the execution.
    /// Once a safe is consumed it can't be used later.  
    consumed_safes_idxs: HashSet<SafeIndex>,
    /// Returned safe indices.
    /// NOTE: These safes are recorded as consumed as well.
    returned_safes_idxs: Vec<SafeIndex>,
```

Handling of returned safes:
- If there are 2 or more outgoing dependencies, the `safe_return` call will panic immediately.
NOTE: Even though the safe can be attached to the promise, with multiple outgoing dependencies, you can do this by
attaching it directly instead of relying on `safe_return`.
- Otherwise the safe index is added to both `consumed_safes_idxs` and
  `returned_safes_idxs`. NOTE: if the outgoing dependencies are empty, the safe will not be returned
  anywhere, so it effectively will be dropped after this action.

We also need to update `Promise` enum to indicate safe resolving promise:
```rust
/// Promises API allows to create a DAG-structure that defines dependencies between smart contract
/// calls. A single promise can be created with zero or several dependencies on other promises.
/// * If a promise was created from a receipt (using `promise_create` or `promise_then`) it's a
///   `Receipt`;
/// * If a promise was created by merging several promises (using `promise_and`) then
///   it's a `NotReceipt`, but has receipts of all promises it depends on.
/// * If a promise was created by creating a safe, then this promise can't be used for any other
///   promise operations.
#[derive(Debug)]
enum Promise {
    Receipt(ReceiptIndex),
    NotReceipt(Vec<ReceiptIndex>),
    /// The promise was created with a safe, so it shouldn't be use or exposed.
    Safe(SafeIndex),
}
```

`RuntimeExt` needs the following methods:
```rust
    /// Creates a new safe. The given action receipt will be called when the safe is dropped.
    /// VMLogic guarantees that the receipt doesn't have any input or output dependencies yet.
    /// The receiver ID of the corresponding receipt will be the owner of the new safe.
    fn safe_create(&mut self, receipt_index: u64) -> SafeIndex;
    
    /// Updates the content of the given safe.
    /// VMLogic should ensure that the safe is owned by the current account ID.
    fn safe_set_content(&mut self, safe_idx: SafeIndex, content: Vec<u8>);

    /// Returns content of the safe.
    fn safe_get_content(&self, safe_idx: SafeIndex) -> &[u8];
    
    /// Returns account ID of the safe owner.
    fn safe_get_owner_id(&self, safe_idx: SafeIndex) -> AccountId;
    
    /// Consumes given safe and attaches it to the given action receipt.
    fn safe_attach(&mut self, receipt_index: u64, safe_idx: SafeIndex);
``` 

Need to add the following fields to the `ActionResult`:
```rust
    /// Safes that needs to passed towards the single dependency in `outgoing_dependencies`.
    /// Or dropped if there are no `outgoing_dependencies`.
    pub returned_safes: Vec<SafeWrapper>,
    /// Safes that were dropped by returning safes from earlier actions.
    /// Need to track them, since new safes might have been created in between.
    pub dropped_safes: Vec<SafeWrapper>,
``` 

Collecting safes after successful execution of a Function Call action:
- `VMOutcome` should contain the following fields from `VMLogic`:
    - `new_safes_idxs`
    - `consumed_safes_idxs`
    - `returned_safes_idxs`
- `RuntimeExt` should return `mut safes` back to `Runtime`.
- `Runtime` should do the following:
    - Filter `safes` by removing all `consumed_safes_idxs`:
        - Remember the remapping for the old indices towards a new indices.
        - Returned safes should be retained in a `returned_safes` in `ActionResult`.
    - Add `new_safes_idxs` to `input_safes_idxs`.
    - Update `input_safes_idxs` and `promise_results_safes_idxs` by retaining only safe
    indices that were not consumed and remapping the old indices to the new indices.

Merging `ActionResult`:
- If we merging a new `ActionResult`, all old `returned_safes` should be moved to old `dropped_safes`.
The reason for this is there shouldn't be any `outgoing_dependencies` in the old `ActionResult`.
Because the old action was not the last action and only the last action can have `outgoing_dependencies`.
- If the new `ActionResult` result is `Err`, all new `returned_safes` should be moved to new `dropped_safes`.
- new `dropped_safes` are added after old `dropped_safes`.

### Update `ActionReceipt` and `DataReceipt`

Need to add one field to `ActionReceipt`:
```rust
    /// Attached safes.
    pub safes: Vec<Safe>,
```

Also need to add one field to `DataReceipt`:
```rust
    /// Safes returned with this data.
    /// It will be empty for failed executions.
    pub safes: Vec<Safe>,
```

### Resolving safes at the end.

If the `ActionResult` result is `Err`:
- drop all `original_safes`

If the `ActionResult` result is `Ok`, we have safes in the following fields:
- `returned_safes` and `dropped_safes` in the `ActionResult`
- some safes that are already attached to `Receipt` in the `ActionResult`
- all the remaining `safes` that were not consumed and should be dropped.
- if the `return_data` is `PromiseIndex` all `returned_safes` should be moved to `dropped_safes`.
- if there are no `outgoing_dependencies` all `returned_safes` should be moved to `dropped_safes`.

Handling `returned_safes` for `Ok` with exactly 1 outgoing dependency:
- Append all safes from `returned_safes` to `safes` from the outgoing `DataReceipt`. 

Dropping safes:
- To drop a safe, we need to create a `Receipt` with a `DataReceipt`. 
    - `receiver_id` is `safe.owner_id`.
    - `data_id` is `safe.data_id`.
    - `data` is the `Some(safe.content)`.
