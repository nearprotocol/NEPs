# Multiple Fungible Token ()
Version 1.0.0

## Summary

A standard interface for multi fungible tokens that allows for a normal transfer as well as a transfer and method call in a single transaction. The storage standard and the fungible token metadata standard are the same with NEP-141. 

## Motivation

There are many needs that request to handle more than one token in a single NEAR contract. For example, Ref Finance handles their all swap pools in one contract, and release LP tokens for each pool. Those tokens are all different fungible tokens which exist in a single contract. it can NOT be supported by NEP-141 standard directly.  

Someone may argue why not we split those tokens into different contracts? There are several reasons for this choice:
* There are some inner logic relations between those tokens and cross-contract call will introduce async mechanism which unsuitable for some dapps, and cause meanless complexity to some dapps inner logic.
* Putting homogeneous tokens into different contracts cause meaningless storage waste and widespread access keys.
* There are some management and maintainance consideration that request for using single contract to handle multiple tokens, such as upgradation and etc.  

This proposal will address that problem. With our MFT protocol, users and developers can organize multiple tokens in one contract, and has them having all the abilities that NEP-141 can do.

Prior art:
* [NEP-141](https://nomicon.io/Standards/FungibleToken/Core.html) 

## Guide-level explanation

MFT is an abbreviation for Multple Fungilbe Token. On NEAR network, it means using one single contract to manage mulitple fungible tokens, and those tokens are very likely to have similar logic, purpose or in same category.

To understand this standard, you should first read [NEP-141 guide-level explanation](https://nomicon.io/Standards/FungibleToken/Core.html#guide-level-explanation).  

The difference here is we add a special field `token_id` into those interfaces to enable distinguish tokens in the multi-fungible-token contract. Generally speaking, when the value of this field equals to predecessor_id or the token contract id, the interface has the same behaviors with original one in NEP-141. Otherwise, the token_id stands for an internal index in that MFT contract.  

The [example scenarios](https://nomicon.io/Standards/FungibleToken/Core.html#example-scenarios) is just the same with NEP-141, only need remember to add that special field as the first argument for following interface:
* `mft_resolve_transfer` in trait MFTTokenResolver 
* `mft_on_transfer` in trait MFTTokenReceiver 
* `mft_transfer`
* `mft_transfer_call`
* `mft_metadata`, return the same FungibleTokenMetadata structure in NEP-141
* `mft_register`
* `mft_total_supply`
* `mft_balance_of`


## Reference-level explanation
### NOTES:

* All amounts, balances and allowance are limited by U128 (max value 2**128 - 1).
* Token standard uses JSON for serialization of arguments and results.
* Amounts in arguments and results have are serialized as Base-10 strings, e.g. "100". This is done to avoid JSON limitation of max integer value of 2**53.
* The contract must track the change in storage when adding to and removing from collections. This is not included in this core fungible token standard but instead in the Storage Standard.
* To prevent the deployed contract from being modified or deleted, it should not have any access keys on its account.
### Interface:
we only list those are different with NEP-141
```javascript
/*********************************************/
/* CHANGE METHODS on multiple fungible token */
/*********************************************/
// Simple transfer to a receiver.
//
// Requirements:
// * Caller of the method must attach a deposit of 1 yoctoⓃ for security purposes
// * Caller must have greater than or equal to the `amount` being requested
//
// Arguments:
// * `token_id`: the internal token index for mft, empty or same as predecessor_id 
//   would make it be taken as normal ft.
// * `receiver_id`: the valid NEAR account receiving the fungible tokens.
// * `amount`: the number of tokens to transfer, wrapped in quotes and treated
//   like a string, although the number will be stored as an unsigned integer
//   with 128 bits.
// * `memo` (optional): for use cases that may benefit from indexing or
//    providing information for a transfer.
function mft_transfer(
    token_id: String,
    receiver_id: string,
    amount: string,
    memo: string|null
): void {}

// Transfer tokens and call a method on a receiver contract. A successful
// workflow will end in a success execution outcome to the callback on the same
// contract at the method `ft_resolve_transfer`.
//
// You can think of this as being similar to attaching native NEAR tokens to a
// function call. It allows you to attach any Fungible Token in a call to a
// receiver contract.
//
// Requirements:
// * Caller of the method must attach a deposit of 1 yoctoⓃ for security
//   purposes
// * Caller must have greater than or equal to the `amount` being requested
// * The receiving contract must implement `ft_on_transfer` according to the
//   standard. If it does not, FT contract's `ft_resolve_transfer` MUST deal
//   with the resulting failed cross-contract call and roll back the transfer.
// * Contract MUST implement the behavior described in `ft_resolve_transfer`
//
// Arguments:
// * `token_id`: the internal token index for mft, empty or same as predecessor_id 
//   would make it be taken as normal ft.
// * `receiver_id`: the valid NEAR account receiving the fungible tokens.
// * `amount`: the number of tokens to transfer, wrapped in quotes and treated
//   like a string, although the number will be stored as an unsigned integer
//   with 128 bits.
// * `memo` (optional): for use cases that may benefit from indexing or
//    providing information for a transfer.
// * `msg`: specifies information needed by the receiving contract in
//    order to properly handle the transfer. Can indicate both a function to
//    call and the parameters to pass to that function.
function mft_transfer_call(
   token_id: string,
   receiver_id: string,
   amount: string,
   memo: string|null,
   msg: string
): Promise {}

/****************************************/
/* CHANGE METHODS on receiving contract */
/****************************************/

// This function is implemented on the receving contract.
// As mentioned, the `msg` argument contains information necessary for the receiving contract to know how to process the request. This may include method names and/or arguments. 
// Returns a value, or a promise which resolves with a value. The value is the
// number of unused tokens in string form. For instance, if `amount` is 10 but only 9 are
// needed, it will return "1".
function mft_on_transfer(
    token_id: string,
    sender_id: string,
    amount: string,
    msg: string
): string {}

/****************/
/* VIEW METHODS */
/****************/

// * `token_id`: the internal token index for mft, empty would make it be taken as normal ft.
// Returns the total supply of fungible tokens as a string representing the value as an unsigned 128-bit integer.
function mft_total_supply(token_id: string): string {}

// * `token_id`: the internal token index for mft, empty would make it be taken as normal ft.
// Returns the balance of an account in string form representing a value as an unsigned 128-bit integer. If the account doesn't exist must returns `"0"`.
function mft_balance_of(
    token_id: string,
    account_id: string
): string {}

```

The following behavior is required, but contract authors may name this function something other than the conventional mft_resolve_transfer used here.
```javascript
// Finalize an `mft_transfer_call` chain of cross-contract calls.
//
// The `mft_transfer_call` process:
//
// 1. Sender calls `mft_transfer_call` on MFT contract
// 2. MFT contract transfers `amount` tokens from sender to receiver
// 3. MFT contract calls `mft_on_transfer` on receiver contract
// 4+. [receiver contract may make other cross-contract calls]
// N. MFT contract resolves promise chain with `mft_resolve_transfer`, 
// and may refund sender some or all of original `amount`
//
// Requirements:
// * Contract MUST forbid calls to this function by any account except self
// * If promise chain failed, contract MUST revert token transfer
// * If promise chain resolves with a non-zero amount given as a string,
//   contract MUST return this amount of tokens to `sender_id`
//
// Arguments:
// * `token_id`: the `token_id` argument given to `mft_transfer_call`
// * `sender_id`: the sender of `mft_transfer_call`
// * `receiver_id`: the `receiver_id` argument given to `mft_transfer_call`
// * `amount`: the `amount` argument given to `mft_transfer_call`
//
// Returns a string representing a string version of an unsigned 128-bit
// integer of how many total tokens were spent by sender_id. Example: if sender
// calls `mft_transfer_call({ "token_id": "0", "amount": "100" })`, but 
// `receiver_id` only uses 80, `mft_on_transfer` will resolve with `"20"`, 
// and `mft_resolve_transfer` will return `"80"`.
function mft_resolve_transfer(
   token_id: string,
   sender_id: string,
   receiver_id: string,
   amount: string
): string {}

```

## Drawbacks

* the `token_id` argument is freeform, which may necessitate conventions.

## Future possibilities

* Minting and burning