- Proposal Name: System methods in runtime API
- Start Date: 2019-09-03
- NEP PR: [nearprotocol/neps#0013](https://github.com/nearprotocol/neps/pull/0013)

# Summary
[summary]: #summary

Adds new ability for contracts to perform some system functions:
- create new accounts (with possible code deploy and initialization)
- deploy new code (or redeploying code for upgrades)
- batched function calls
- transfer money
- stake
- add key
- delete key
- delete account

# Motivation
[motivation]: #motivation

Contracts should have the ability to create new accounts, transfer money without calling code and
stake. It will enable full functionality of contract-based accounts.

# Reference
[reference]: #reference

We introduce additional promise APIs to support batched actions.

Firstly, we enable ability to create empty promises without any action. They act similarly to
traditional promises, but don't contain function call action.

Secondly, we add API to append individual actions to promises. For example we can create
a promise with a function_call first using `promise_create` and then attach a transfer action on top
of this promise. So the transfer will only deposit tokens if the function call succeeds. Another example
is how we create accounts now using batched actions. To create a new account, we create a transaction with
the following actions: `create_account`, `transfer`, `add_key`. It creates a new account, deposit some funds on it and the adds a new key.

For more examples see NEP#8: https://github.com/nearprotocol/NEPs/pull/8/files?short_path=15b6752#diff-15b6752ec7d78e7b85b8c7de4a19cbd4

**NOTE: The existing promise API is a special case of the batched promise API.**
- Calling `promise_batch_create` and then `promise_batch_action_function_call` will produce the same promise as calling `promise_create` directly.
- Calling `promise_batch_then` and then `promise_batch_action_function_call` will produce the same promise as calling `promise_then` directly.

## Promises API
[promises-api]: #promises-api

```rust
promise_batch_create(account_id_len: u64, account_id_ptr: u64) -> u64
```
Creates a new promise towards given `account_id` without any actions attached to it.

###### Panics
* If `account_id_len + account_id_ptr` points outside the memory of the guest or host, with `MemoryAccessViolation`.

###### Returns
* Index of the new promise that uniquely identifies it within the current execution of the method.

---

```rust
promise_batch_then(promise_idx: u64, account_id_len: u64, account_id_ptr: u64) -> u64            
```
Attaches a new empty promise that is executed after promise pointed by `promise_idx` is complete.

###### Panics
* If `promise_idx` does not correspond to an existing promise panics with `InvalidPromiseIndex`.
* If `account_id_len + account_id_ptr` points outside the memory of the guest or host, with `MemoryAccessViolation`.

###### Returns
* Index of the new promise that uniquely identifies it within the current execution of the method.

---

```rust
promise_batch_action_create_account(promise_idx: u64)
```
Appends `CreateAccount` action to the batch of actions for the given promise pointed by `promise_idx`.
Details for the action: https://github.com/nearprotocol/NEPs/pull/8/files#diff-15b6752ec7d78e7b85b8c7de4a19cbd4R48

###### Panics
* If `promise_idx` does not correspond to an existing promise panics with `InvalidPromiseIndex`.
* If the promise pointed by the `promise_idx` is an ephemeral promise created by `promise_and`.

---

```rust
promise_batch_action_deploy_contract(promise_idx: u64, code_len: u64, code_ptr: u64)
```
Appends `DeployContract` action to the batch of actions for the given promise pointed by `promise_idx`.
Details for the action: https://github.com/nearprotocol/NEPs/pull/8/files#diff-15b6752ec7d78e7b85b8c7de4a19cbd4R49

###### Panics
* If `promise_idx` does not correspond to an existing promise panics with `InvalidPromiseIndex`.
* If the promise pointed by the `promise_idx` is an ephemeral promise created by `promise_and`.
* If `code_len + code_ptr` points outside the memory of the guest or host, with `MemoryAccessViolation`.

---

```rust
promise_batch_action_function_call(promise_idx: u64,
                                   method_name_len: u64,
                                   method_name_ptr: u64,
                                   arguments_len: u64,
                                   arguments_ptr: u64,
                                   amount_ptr: u64,
                                   gas: u64)
```
Appends `FunctionCall` action to the batch of actions for the given promise pointed by `promise_idx`.
Details for the action: https://github.com/nearprotocol/NEPs/pull/8/files#diff-15b6752ec7d78e7b85b8c7de4a19cbd4R50

*NOTE: Calling `promise_batch_create` and then `promise_batch_action_function_call` will produce the same promise as calling `promise_create` directly.*

###### Panics
* If `promise_idx` does not correspond to an existing promise panics with `InvalidPromiseIndex`.
* If the promise pointed by the `promise_idx` is an ephemeral promise created by `promise_and`.
* If `account_id_len + account_id_ptr` or `method_name_len + method_name_ptr` or `arguments_len + arguments_ptr`
or `amount_ptr + 16` points outside the memory of the guest or host, with `MemoryAccessViolation`.

---

```rust
promise_batch_action_transfer(promise_idx: u64, amount_ptr: u64)
```
Appends `Transfer` action to the batch of actions for the given promise pointed by `promise_idx`.
Details for the action: https://github.com/nearprotocol/NEPs/pull/8/files#diff-15b6752ec7d78e7b85b8c7de4a19cbd4R51

###### Panics
* If `promise_idx` does not correspond to an existing promise panics with `InvalidPromiseIndex`.
* If the promise pointed by the `promise_idx` is an ephemeral promise created by `promise_and`.
* If `amount_ptr + 16` points outside the memory of the guest or host, with `MemoryAccessViolation`.

---

```rust
promise_batch_action_stake(promise_idx: u64,
                           amount_ptr: u64,
                           public_key_len: u64,
                           public_key_ptr: u64)
```
Appends `Stake` action to the batch of actions for the given promise pointed by `promise_idx`.
Details for the action: https://github.com/nearprotocol/NEPs/pull/8/files#diff-15b6752ec7d78e7b85b8c7de4a19cbd4R52

###### Panics
* If `promise_idx` does not correspond to an existing promise panics with `InvalidPromiseIndex`.
* If the promise pointed by the `promise_idx` is an ephemeral promise created by `promise_and`.
* If the given public key is not a valid public key (e.g. wrong length) `InvalidPublicKey`.
* If `amount_ptr + 16` or `public_key_len + public_key_ptr` points outside the memory of the guest or host, with `MemoryAccessViolation`.

---

```rust
promise_batch_action_add_key_with_full_access(promise_idx: u64,
                                              public_key_len: u64,
                                              public_key_ptr: u64,
                                              nonce: u64)
```
Appends `AddKey` action to the batch of actions for the given promise pointed by `promise_idx`.
Details for the action: https://github.com/nearprotocol/NEPs/pull/8/files#diff-15b6752ec7d78e7b85b8c7de4a19cbd4R54
The access key will have `FullAccess` permission, details: https://github.com/nearprotocol/NEPs/blob/master/text/0005-access-keys.md#guide-level-explanation

###### Panics
* If `promise_idx` does not correspond to an existing promise panics with `InvalidPromiseIndex`.
* If the promise pointed by the `promise_idx` is an ephemeral promise created by `promise_and`.
* If the given public key is not a valid public key (e.g. wrong length) `InvalidPublicKey`.
* If `public_key_len + public_key_ptr` points outside the memory of the guest or host, with `MemoryAccessViolation`.

---

```rust
promise_batch_action_add_key_with_function_call(promise_idx: u64,
                                                public_key_len: u64,
                                                public_key_ptr: u64,
                                                nonce: u64,
                                                allowance_ptr: u64,
                                                receiver_id_len: u64,
                                                receiver_id_ptr: u64,
                                                method_names_len: u64,
                                                method_names_ptr: u64)
```
Appends `AddKey` action to the batch of actions for the given promise pointed by `promise_idx`.
Details for the action: https://github.com/nearprotocol/NEPs/pull/8/files#diff-156752ec7d78e7b85b8c7de4a19cbd4R54
The access key will have `FunctionCall` permission, details: https://github.com/nearprotocol/NEPs/blob/master/text/0005-access-keys.md#guide-level-explanation

* If the `allowance` value (not the pointer) is `0`, the allowance is set to `None` (which means unlimited allowance). And positive value represents a `Some(...)` allowance.
* Given `method_names` is a `utf-8` string with `,` used as a separator. The vm will split the given string into a vector of strings.

###### Panics
* If `promise_idx` does not correspond to an existing promise panics with `InvalidPromiseIndex`.
* If the promise pointed by the `promise_idx` is an ephemeral promise created by `promise_and`.
* If the given public key is not a valid public key (e.g. wrong length) `InvalidPublicKey`.
* if `method_names` is not a valid `utf-8` string, fails with `BadUTF8`.
* If `public_key_len + public_key_ptr`, `allowance_ptr + 16`, `receiver_id_len + receiver_id_ptr` or 
`method_names_len + method_names_ptr` points outside the memory of the guest or host, with `MemoryAccessViolation`.

---

```rust
promise_batch_action_delete_key(promise_idx: u64,
                                public_key_len: u64,
                                public_key_ptr: u64)
```
Appends `DeleteKey` action to the batch of actions for the given promise pointed by `promise_idx`.
Details for the action: https://github.com/nearprotocol/NEPs/pull/8/files#diff-15b6752ec7d78e7b85b8c7de4a19cbd4R55

###### Panics
* If `promise_idx` does not correspond to an existing promise panics with `InvalidPromiseIndex`.
* If the promise pointed by the `promise_idx` is an ephemeral promise created by `promise_and`.
* If the given public key is not a valid public key (e.g. wrong length) `InvalidPublicKey`.
* If `public_key_len + public_key_ptr` points outside the memory of the guest or host, with `MemoryAccessViolation`.

---

```rust
promise_batch_action_delete_account(promise_idx: u64,
                                    beneficiary_id_len: u64,
                                    beneficiary_id_ptr: u64)
```
Appends `DeleteAccount` action to the batch of actions for the given promise pointed by `promise_idx`.
Action is used to delete an account. It can be performed on a newly created account, on your own account or an account with
insufficient funds to pay rent. Takes `beneficiary_id` to indicate where to send the remaining funds.

###### Panics
* If `promise_idx` does not correspond to an existing promise panics with `InvalidPromiseIndex`.
* If the promise pointed by the `promise_idx` is an ephemeral promise created by `promise_and`.
* If `beneficiary_id_len + beneficiary_id_ptr` points outside the memory of the guest or host, with `MemoryAccessViolation`.

---

