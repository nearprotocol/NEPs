- Proposal Name: `wasm_bindings`
- Start Date: 2019-07-22
- NEP PR: [nearprotocol/neps#0000](https://github.com/nearprotocol/neps/pull/0000)

# Summary
[summary]: #summary

Wasm bindings, a.k.a imports, are functions that the runtime (a.k.a host) exposes to the Wasm code (a.k.a guest) running on the virtual machine.
These functions are arguably the most difficult thing to change in our entire ecosystem, after we have contracts running on our blockchain,
since once the bindings change the old smart contracts will not be able to run on the new nodes.
Additionally, we need a highly detailed specification of the bindings to be able to write unit tests for our contracts,
since currently we only allow integration tests. Currently, writing unit tests is not possible since we cannot have
a precise mock of the host in the smart contract unit tests, e.g. we don't know how to mock the range iterator (what does it do
when given an empty or inverted range?).

In this proposal we give a detailed specification of the functions that we will be relying on for many months to come.

# Motivation
[motivation]: #motivation

The current imports have the following issues:
* **Trie API.** The behavior of trie API is currently unspecified. Many things are unclear: what happens when we try
iterating over an empty range, what happens if we try accessing a non-existent key, etc. Having a trie API specification
is important for being able to create a testing framework for Rust and AssemblyScript smart contracts, since in unit
tests the contracts will be running on a mocked implementation of the host;
* **Promise API.** Recently we have discussed the changes to our promise mechanics. The schema does not need to change,
but the specification now needs to be clarified;
* `data_read` currently has mixed functionality -- it can be used both for reading data from trie and to read data from
the context. In former it expects pointers to be passed as arguments, in later it expects enum to be passed. It achieves
juxtaposition by casting pointer type in enum when needed;
* **Economics API.** The functions that provide access to balance and such might need to be added or removed since we
now consider splitting attached balance into two.

# Specification

## Registers
Registers allow the host function to return the data into a buffer located inside the host oppose to the buffer
located on the client. A special operation can be used to copy the content of the buffer into the host. Memory pointers
can then be used to point either to the memory on the guest or the memory on the host, see below. Benefits:
* We can have functions that return values that are not necessarily used, e.g. inserting key-value into a trie can
also return the preempted old value, which might not be necessarily used. Previously, if we returned something we
would have to pass the blob from host into the guest, even if it is not used;
* We can pass blobs of data between host functions without going through the guest, e.g. we can remove the value
from the storage and insert it into under a different key;
* It makes API cleaner, because we don't need to pass `buffer_len` and `buffer_ptr` as arguments to other functions;
* It allows merging certain functions together, see `storage_iter_next`;
* This is consistent with other APIs that were created for high performance, e.g. allegedly Ewasm have implemented
SNARK-like computations in Wasm by exposing a bignum library through stack-like interface to the guest. The guest
can manipulate then with the stack of 256-bit numbers that is located on the host.

#### Host → host blob passing
The registers can be used to pass the blobs between host functions. For any function that
takes a pair of arguments `*_len: u64, *_ptr: u64` this pair is pointing to a region of memory either on the guest or
the host:
* If `*_len != u64::MAX` it points to the memory on the guest;
* If `*_len == u64::MAX` it points to the memory under the register `*_ptr` on the host.

For example:
`storage_write(u64::MAX, 0, u64::MAX, 1, 2)` -- insert key-value into storage, where key is read from register 0,
value is read from register 1, and result is saved to register 2.

Note, if some function takes `register_id` then it means this function can copy some data into this register. If
`register_id == u64::MAX` then the copying does not happen. This allows some micro-optimizations in the future.

Note, we allow multiple registers on the host, identified with `u64` number. The guest does not have to use them in
order and can for instance save some blob in register `5000` and another value in register `1`.

#### Specification
```rust
read_register(register_id: u64, ptr: u64)
```
Writes the entire content from the register `register_id` into the memory of the guest starting with `ptr`.
###### Panics
* If the content extends outside the memory allocated to the guest. In Wasmer, it returns `MemoryAccessViolation` error message;
* If `register_id` is pointing to unused register returns `InvalidRegisterId` error message.

###### Undefined Behavior
* If the content of register extends outside the preallocated memory on the host side, or the pointer points to a
wrong location this function will overwrite memory that it is not supposed to overwrite causing an undefined behavior.

---
```rust
register_len(register_id: u64) -> u64
```
Returns the size of the blob stored in the given register.
###### Normal operation
* If register is used, then returns the size, which can potentially be zero;
* If register is not used, returns `u64::MAX`

## Trie API

Here we provide a specification of trie API. After this NEP is merged, the cases where our current implementation does
not follow the specification are considered to be bugs that need to be fixed.

---
```rust
storage_write(key_len: u64, key_ptr: u64, value_len: u64, value_ptr: u64, register_id: u64) -> u64
```
Writes key-value into storage.
###### Normal operation
* If key is not in use it inserts the key-value pair and does not modify the register;
* If key is in use it inserts the key-value and copies the old value into the `register_id`.

###### Returns
* If key was not used returns `0`;
* If key was used returns `1`.

###### Panics
* If `key_len + key_ptr` or `value_len + value_ptr` exceeds the memory container or points to an unused register it panics
with `MemoryAccessViolation`. (When we say that something panics with the given error we mean that we use Wasmer API to
create this error and terminate the execution of VM. For mocks of the host that would only cause a non-name panic.)
* If returning the preempted value into the registers exceed the memory container it panics with `MemoryAccessViolation`;

###### Current bugs
*  `External::storage_set` trait can return an error which is then converted to a generic non-descriptive
   `StorageUpdateError`, [here](https://github.com/nearprotocol/nearcore/blob/942bd7bdbba5fb3403e5c2f1ee3c08963947d0c6/runtime/wasm/src/runtime.rs#L210)
   however the actual implementation does not return error at all, [see](https://github.com/nearprotocol/nearcore/blob/4773873b3cd680936bf206cebd56bdc3701ddca9/runtime/runtime/src/ext.rs#L95);
* Does not return into the registers.

---
```rust
storage_read(key_len: u64, key_ptr: u64, register_id: u64) -> u64
```
Reads the value stored under the given key.
###### Normal operation
* If key is used copies the content of the value into the `register_id`, even if the content is zero bytes;
* If key is not present then does not modify the register.

###### Returns
* If key was not present returns `0`;
* If key was present returns `1`.

###### Panics
* If `key_len + key_ptr` exceeds the memory container or points to an unused register it panics with `MemoryAccessViolation`;
* If returning the preempted value into the registers exceed the memory container it panics with `MemoryAccessViolation`;

###### Current bugs
* This function currently does not exist.

---
```rust
storage_remove(key_len: u64, key_ptr: u64, register_id: u64) -> u64
```
Removes the value stored under the given key.
###### Normal operation
Very similar to `storage_read`:
* If key is used, removes the key-value from the trie and copies the content of the value into the `register_id`, even if the content is zero bytes.
* If key is not present then does not modify the register.

###### Returns
* If key was not present returns `0`;
* If key was present returns `1`.

###### Panics
* If `key_len + key_ptr` exceeds the memory container or points to an unused register it panics with `MemoryAccessViolation`;
* If the registers exceed the memory limit panics with `MemoryAccessViolation`;
* If returning the preempted value into the registers exceed the memory container it panics with `MemoryAccessViolation`;


###### Current bugs
* Does not return into the registers.

---
```rust
storage_has_key(key_len: u64, key_ptr: u64) -> u64
```
Checks if there is a key-value pair.
###### Normal operation
* If key is used returns `1`, even if the value is zero bytes;
* Otherwise returns `0`.

###### Panics
* If `key_len + key_ptr` exceeds the memory container it panics with `MemoryAccessViolation`;

---
```rust
storage_iter_prefix(prefix_len: u64, prefix_ptr: u64) -> u64
```
Creates an iterator object inside the host.
Returns the identifier that uniquely differentiates the given iterator from other iterators that can be simultaneously
created.
###### Normal operation
* It iterates over the keys that have the provided prefix. The order of iteration is defined by the lexicographic
order of the bytes in the keys. If there are no keys, it creates an empty iterator, see below on empty iterators;
###### Panics
* If `prefix_len + prefix_ptr` exceeds the memory container it panics with `MemoryAccessViolation`;

---
```rust
storage_iter_range(start_len: u64, start_ptr: u64, end_len: u64, end_ptr: u64) -> u64
```
Similarly to `storage_iter_prefix`
creates an iterator object inside the host.
###### Normal operation
Unless lexicographically `start < end`, it creates an empty iterator.
Iterates over all key-values such that keys are between `start` and `end`, where `start` is inclusive and `end` is exclusive.

Note, this definition allows for `start` or `end` keys to not actually exist on the given trie.

###### Panics:
* If `start_len + start_ptr` or `end_len + end_ptr` exceeds the memory container or points to an unused register it panics with `MemoryAccessViolation`;

---
```rust
storage_iter_next(iterator_id: u64, key_register_id: u64, value_register_id: u64) -> u64
```
Advances iterator and saves the next key and value in the register.
###### Normal operation
* If iterator is not empty (after calling next it points to a key-value), copies the key into `key_register_id` and value into `value_register_id` and returns `1`;
* If iterator is empty returns `0`.

This allows us to iterate over the keys that have zero bytes stored in values.

###### Panics
* If `key_register_id == value_register_id` panics with `MemoryAccessViolation`;
* If the registers exceed the memory limit panics with `MemoryAccessViolation`;
* If `iterator_id` does not correspond to an existing iterator panics with  `InvalidIteratorId`
* If between the creation of the iterator and calling `storage_iter_next` any modification to storage was done through
  `storage_write` or `storage_remove` the iterator is invalidated and the error message is `IteratorWasInvalidated`.

###### Current bugs
* Not implemented, currently we have `storage_iter_next` and `data_read` + `DATA_TYPE_STORAGE_ITER` that together fulfill
the purpose, but have unspecified behavior.

## Context API
Context API mostly provides read-only functions that access current information about the blockchain, the accounts
(that originally initiated the chain of cross-contract calls, the immediate contract that called the current one, the account of the current contract),
other important information like storage usage.

Many of the below functions are currently implemented through `data_read` which allows to read generic context data.
However, there is no reason to have `data_read` instead of the specific functions:
* `data_read` does not solve forward compatibility. If later we want to add another context function, e.g. `executed_operations`
we can just declare it as a new function, instead of encoding it as `DATA_TYPE_EXECUTED_OPERATIONS = 42` which is passed
as the first argument to `data_read`;
* `data_read` does not help with renaming. If later we decide to rename `signer_account_id` to `originator_id` then one could
argue that contracts that rely on `data_read` would not break, while contracts relying on `signer_account_id()` would. However
the name change often means the change of the semantics, which means the contracts using this function are no longer safe to
execute anyway.

However there is one reason to not have `data_read` -- it makes `API` more human-like which is a general direction Wasm APIs, like WASI are moving towards to.

---
```rust
current_account_id(register_id: u64)
```
Saves the account id of the current contract that we execute into the register.

###### Panics
* If the registers exceed the memory limit panics with `MemoryAccessViolation`;

---
```rust
signer_account_id(register_id: u64)
```
All contract calls are a result of some transaction that was signed by some account using
some access key and submitted into a memory pool (either through the wallet using RPC or by a node itself). This function returns the id of that account.

###### Normal operation
* Saves the bytes of the signer account id into the register.

###### Panics
* If the registers exceed the memory limit panics with `MemoryAccessViolation`;

###### Current bugs
* Currently we conflate `originator_id` and `sender_id` in our code base.

---
```rust
signer_account_pk(register_id: u64)
```
Saves the public key fo the access key that was used by the signer into the register.
In rare situations smart contract might want to know the exact access key that was used to send the original transaction,
e.g. to increase the allowance or manipulate with the public key.

###### Panics
* If the registers exceed the memory limit panics with `MemoryAccessViolation`;


###### Current bugs
* Not implemented.

---
```rust
predecessor_account_id(register_id: u64)
```
All contract calls are a result of a receipt, this receipt might be created by a transaction
that does function invocation on the contract or another contract as a result of cross-contract call.

###### Normal operation
* Saves the bytes of the predecessor account id into the register.

###### Panics
* If the registers exceed the memory limit panics with `MemoryAccessViolation`;

###### Current bugs
* Not implemented.

---
```rust
input(register_id: u64)
```
Reads input to the contract call into the register. Input is expected to be in JSON-format.

###### Normal operation
* If input is provided saves the bytes (potentially zero) of input into register.
* If input is not provided does not modify the register.

###### Returns
* If input was not provided returns `0`;
* If input was provided returns `1`; If input is zero bytes returns `1`, too.

###### Panics
* If the registers exceed the memory limit panics with `MemoryAccessViolation`;

###### Current bugs
* Implemented as part of `data_read`. However there is no reason to have one unified function, like `data_read` that can
be used to read all

---
```rust
block_index() -> u64
```
Returns the current block index.

---
```rust
storage_usage() -> u64
```
Returns the number of bytes used by the contract if it was saved to the trie as of the
invocation. This includes:
* The data written with `storage_*` functions during current and previous execution;
* The bytes needed to store the account protobuf and the access keys of the given account.

## Economics API
Accounts own certain balance; and each transaction and each receipt have certain amount of balance and prepaid gas
attached to them.
During the contract execution, the contract has access to the following `u128` values:
* `account_balance` -- the balance attached to the given account. This includes the `attached_deposit` that was attached
  to the transaction;
* `attached_deposit` -- the balance that was attached to the call that will be immediately deposited before
  the contract execution starts;
* `prepaid_gas` -- the tokens attached to the call that can be used to pay for the gas;
* `used_gas` -- the gas that was already burnt during the contract execution and attached to promises (cannot exceed `prepaid_gas`);

If contract execution fails `prepaid_gas - used_gas` is refunded back to `signer_account_id` and `attached_balance`
is refunded back to `predecessor_account_id`.

The following spec is the same for all functions:
```rust
account_balance(balance_ptr: u64)
attached_deposit(balance_ptr: u64)

```
 -- writes the value into the `u128` variable pointed by `balance_ptr`.

###### Panics
* If `balance_ptr + 16` points outside the memory of the guest with `MemoryAccessViolation`;

###### Current bugs
* Use a different name;

---
```rust
prepaid_gas() -> u64
used_gas() -> u64
```

## Math

```rust
random_seed(register_id: u64)
```
Returns random seed that can be used for pseudo-random number generation in deterministic way.

###### Panics
* If the size of the registers exceed the set limit `MemoryAccessViolation`;

---
```rust
sha256(value_len: u64, value_ptr: u64, register_id: u64)
```
Hashes the random sequence of bytes using sha256 and returns it into `register_id`.
###### Panics
* If `value_len + value_ptr` points outside the memory or the registers use more memory than the limit with `MemoryAccessViolation`.

###### Current bugs
* Current name `hash` is not specific to what hash is being used.
* We have `hash32` that largely duplicates the mechanics of `hash` because it returns the first 4 bytes only.

---
```rust
check_ethash(block_number_ptr: u64,
             header_hash_ptr: u64,
             nonce: u64,
             mix_hash_ptr: u64,
             difficulty_ptr: u64) -> u64
```
-- verifies hash of the header that we created using [Ethash](https://en.wikipedia.org/wiki/Ethash). Parameters are:
* `block_number` -- `u256`/`[u64; 4]`, number of the block on Ethereum blockchain. We use the pointer to the slice of 32 bytes on guest memory;
* `header_hash` -- `h256`/`[u8; 32]`, hash of the header on Ethereum blockchain. We use the pointer to the slice of 32 bytes on guest memory;
* `nonce` -- `u64`/`h64`/`[u8; 8]`, nonce that was used to find the correct hash, passed as `u64` without pointers;
* `mix_hash` -- `h256`/`[u8; 32]`, special hash that avoid griefing attack. We use the pointer to the slice of 32 bytes on guest memory;
* `difficulty` -- `u256`/`[u64; 4]`, the difficulty of mining the block. We use the pointer to the slice of 32 bytes on guest memory;

###### Returns
* `1` if the Ethash is valid;
* `0` otherwise.

###### Panics
* If `block_number_ptr + 32` or `header_hash_ptr + 32` or `mix_hash_ptr + 32` or `difficulty_ptr + 32` point outside the memory or registers use more memory than the limit with `MemoryAccessViolation`.

###### Current bugs
* `block_number` and `difficulty` are currently exposed as `u64` which are casted to `u256` which breaks Ethereum compatibility;
* Currently, we also pass the length together with `header_hash_ptr` and `mix_hash_ptr` which is not necessary since
we know their length.

## Promises API

```rust
promise_create(account_id_len: u64,
               account_id_ptr: u64,
               method_name_len: u64,
               method_name_ptr: u64,
               arguments_len: u64,
               arguments_ptr: u64,
               amount_ptr: u64,
               gas: u64) -> u64
```
Creates a promise that will execute a method on account with given arguments and attaches the given amount.
`amount_ptr` point to slices of bytes representing `u128`.

###### Panics
* If `account_id_len + account_id_ptr` or `method_name_len + method_name_ptr` or `arguments_len + arguments_ptr`
or `amount_ptr + 16` points outside the memory of the guest or host, with `MemoryAccessViolation`.

###### Returns
* Index of the new promise that uniquely identifies it within the current execution of the method.

---

```rust
promise_then(promise_idx: u64,
             account_id_len: u64,
             account_id_ptr: u64,
             method_name_len: u64,
             method_name_ptr: u64,
             arguments_len: u64,
             arguments_ptr: u64,
             amount_ptr: u64,
             gas: u64) -> u64
```
Attaches the callback that is executed after promise pointed by `promise_idx` is complete.

###### Panics
* If `promise_idx` does not correspond to an existing promise panics with `InvalidPromiseIndex`.
* If `account_id_len + account_id_ptr` or `method_name_len + method_name_ptr` or `arguments_len + arguments_ptr`
or `amount_ptr + 16` points outside the memory of the guest or host, with `MemoryAccessViolation`.

###### Returns
* Index of the new promise that uniquely identifies it within the current execution of the method.

---
```rust
promise_and(promise_idx_ptr: u64, promise_idx_count: u64) -> u64
```
Creates a new promise which completes when time all promises passed as arguments complete. Cannot be used with registers.
`promise_idx_ptr` points to an array of `u64` elements, with `promise_idx_count` denoting the number of elements.
The array contains indices of promises that need to be waited on jointly.

###### Panics
* If `promise_ids_ptr + 8 * promise_idx_count` extend outside the guest memory with `MemoryAccessViolation`;
* If any of the promises in the array do not correspond to existing promises panics with `InvalidPromiseIndex`.

###### Returns
* Index of the new promise that uniquely identifies it within the current execution of the method.

---
```rust
promise_results_count() -> u64
```
If the current function is invoked by a callback we can access the execution results of the promises that
caused the callback. This function returns the number of complete and incomplete callbacks.

Note, we are only going to have incomplete callbacks once we have `promise_or` combinator.
###### Normal execution
* If there is only one callback `promise_results_count()` returns `1`;
* If there are multiple callbacks (e.g. created through `promise_and`) `promise_results_count()` returns their number.
* If the function was called not through the callback `promise_results_count()` returns `0`.


---
```rust
promise_result(result_idx: u64, register_id: u64) -> u64
```
If the current function is invoked by a callback we can access the execution results of the promises that
caused the callback. This function returns the result in blob format and places it into the register.

###### Normal execution
* If promise result is complete and successful copies its blob into the register;
* If promise result is complete and failed or incomplete keeps register unused;

###### Returns
* If promise result is not complete returns `0`;
* If promise result is complete and successful returns `1`;
* If promise result is complete and failed returns `2`.

###### Panics
* If `result_idx` does not correspond to an existing result panics with `InvalidResultIndex`.
* If copying the blob exhausts the memory limit it panics with `MemoryAccessViolation`.

###### Current bugs
* We currently have two separate functions to check for result completion and copy it.

---
```rust
promise_return(promise_idx: u64)
```
When promise `promise_idx` finishes executing its result is considered to be the result of the current function.

###### Panics
* If `promise_idx` does not correspond to an existing promise panics with `InvalidPromiseIndex`.

###### Current bugs
* The current name `return_promise` is inconsistent with the naming convention of Promise API.

## Miscellaneous API
```rust
value_return(value_len: u64, value_ptr: u64)
```
Sets the blob of data as the return value of the contract.

##### Panics
* If `value_len + value_ptr` exceeds the memory container or points to an unused register it panics with `MemoryAccessViolation`;

---
```rust
panic()
```
Terminates the execution of the program with panic `GuestPanic`.

---
```rust
log_utf8(len: u64, ptr: u64)
```
Logs the UTF-8 encoded string. See https://stackoverflow.com/a/5923961 that explains
that null termination is not defined through encoding.

###### Normal behavior
If `len == u64::MAX` then treats the string as null-terminated with character `'\0'`;

###### Panics
* If string extends outside the memory of the guest with `MemoryAccessViolation`;

---
```rust
log_utf16(len: u64, ptr: u64)
```
Logs the UTF-16 encoded string. `len` is the number of bytes in the string.

###### Normal behavior
If `len == u64::MAX` then treats the string as null-terminated with two-byte sequence of `0x00 0x00`.

###### Panics
* If string extends outside the memory of the guest with `MemoryAccessViolation`;

---
```rust
abort(msg_ptr: u32, filename_ptr: u32, line: u32, col: u32)
```
Special import kept for compatibility with AssemblyScript contracts. Not called by smart contracts directly, but instead
called by the code generated by AssemblyScript.


# Future Improvements

In the future we can have some of the registers to be on the guest.
For instance a guest can tell the host that it has some pre-allocated memory that it wants to be used for the register,
e.g.

```rust
set_guest_register(register_id: u64, register_ptr: u64, max_register_size: u64)
```
will assign `register_id` to a span of memory on the guest. Host then would also know the size of that buffer on guest
and can throw a panic if there is an attempted copying that exceeds the guest register size.
