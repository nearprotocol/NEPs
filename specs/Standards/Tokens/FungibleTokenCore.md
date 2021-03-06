# Fungible Token ([NEP-141](https://github.com/near/NEPs/blob/master/specs/Standards/Tokens/FungibleTokenCore.md))

Version `1.0.0`

## Summary

A standard interface for fungible tokens that allows for a normal transfer as well as a transfer and method call in a single transaction. The [storage standard](../StorageManagement.md) addresses the needs (and security) of storage staking. The [fungible token metadata standard](FungibleTokenMetadata.md) provides the fields needed for ergonomics across dApps and marketplaces.

## Motivation

NEAR Protocol uses an asynchronous, sharded runtime. This means the following:
 - Storage for different contracts and accounts can be located on the different shards.
 - Two contracts can be executed at the same time in different shards.

While this increases the transaction throughput linearly with the number of shards, it also creates some challenges for cross-contract development. For example, if one contract wants to query some information from the state of another contract (e.g. current balance), by the time the first contract receives the balance the real balance can change. In such an async system, a contract can't rely on the state of another contract and assume it's not going to change.

Instead the contract can rely on temporary partial lock of the state with a callback to act or unlock, but it requires careful engineering to avoid deadlocks. In this standard we're trying to avoid enforcing locks. A typical approach to this problem is to include an escrow system with allowances. This approach was initially developed for [NEP-21](https://github.com/near/NEPs/pull/21) which is similar to the Ethereum ERC-20 standard. There are a few issues with using an escrow as the only avenue to pay for a service with a fungible token. This frequently requires more than one transaction for common scenarios where fungible tokens are given as payment with the expectation that a method will subsequently be called.

For example, an oracle contract might be paid in fungible tokens. A client contract that wishes to use the oracle must either increase the escrow allowance before each request to the oracle contract, or allocate a large allowance that covers multiple calls. Both have drawbacks and ultimately it would be ideal to be able to send fungible tokens and call a method in a single transaction. This concern is addressed in the `ft_transfer_call` method. The power of this comes from the receiver contract working in concert with the fungible token contract in a secure way. That is, if the receiver contract abides by the standard, a single transaction may transfer and call a method.

Note: there is no reason why an escrow system cannot be included in a fungible token's implementation, but it is simply not necessary in the core standard. Escrow logic should be moved to a separate contract to handle that functionality. One reason for this is because the [Rainbow Bridge](https://near.org/blog/eth-near-rainbow-bridge/) will be transferring fungible tokens from Ethereum to NEAR, where the token locker (a factory) will be using the fungible token core standard.

Prior art:

- [ERC-20 standard](https://eips.ethereum.org/EIPS/eip-20)
- NEP#4 NEAR NFT standard: [near/neps#4](https://github.com/near/neps/pull/4)

Learn about NEP-141:

- [Figment Learning Pathway](https://learn.figment.io/network-documentation/near/tutorials/1-project_overview/2-fungible-token)

## Guide-level explanation

We should be able to do the following:
- Initialize contract once. The given total supply will be owned by the given account ID.
- Get the total supply.
- Transfer tokens to a new user.
- Transfer tokens from one user to another.
- Transfer tokens to a contract, have the receiver contract call a method and "return" any fungible tokens not used.
- Remove state for the key/value pair corresponding with a user's account, withdrawing a nominal balance of Ⓝ that was used for storage.

There are a few concepts in the scenarios above:
- **Total supply**: the total number of tokens in circulation.
- **Balance owner**: an account ID that owns some amount of tokens.
- **Balance**: an amount of tokens.
- **Transfer**: an action that moves some amount from one account to another account, either an externally owned account or a contract account.
- **Transfer and call**: an action that moves some amount from one account to a contract account where the receiver calls a method.
- **Storage amount**: the amount of storage used for an account to be "registered" in the fungible token. This amount is denominated in Ⓝ, not bytes, and represents the [storage staked](https://docs.near.org/docs/concepts/storage-staking).

Note that precision (the number of decimal places supported by a given token) is not part of this core standard, since it's not required to perform actions. The minimum value is always 1 token. See the [Fungible Token Metadata Standard](FungibleTokenMetadata.md) to learn how to support precision/decimals in a standardized way.

Given that multiple users will use a Fungible Token contract and the their activity will result in an increased [storage staking](https://docs.near.org/docs/concepts/storage-staking) burden for the contract's account, this standard is designed to interoperate nicely with [the Account Storage standard](../Storage.md) for storage deposits and refunds.

### Example scenarios

#### Simple transfer

Alice wants to send 5 wBTC tokens to Bob.

**Assumptions**

- The wBTC token contract is `wbtc`.
- Alice's account is `alice`.
- Bob's account is `bob`.
- The precision ("decimals" in the metadata standard) on wBTC contract is `10^8`.
- The 5 tokens is `5 * 10^8` or as a number is `500000000`.

**High-level explanation**

Alice needs to issue one transaction to wBTC contract to transfer 5 tokens (multiplied by precision) to Bob.

**Technical calls**

1. `alice` calls `wbtc::ft_transfer({"receiver_id": "bob", "amount": "500000000"})`.

#### Token deposit to a contract

Alice wants to deposit 1000 DAI tokens to a compound interest contract to earn extra tokens.

**Assumptions**

- The DAI token contract is `dai`.
- Alice's account is `alice`.
- The compound interest contract is `compound`.
- The precision ("decimals" in the metadata standard) on DAI contract is `10^18`.
- The 1000 tokens is `1000 * 10^18` or as a number is `1000000000000000000000`.
- The compound contract can work with multiple token types.

<details style="background-color: #000; padding: 3px; color: #fff">
<summary>For this example, you may expand this section to see how a previous fungible token standard using escrows would deal with the scenario.</summary>
<hr/>

**High-level explanation** (NEP-21 standard)

Alice needs to issue 2 transactions. The first one to `dai` to set an allowance for `compound` to be able to withdraw tokens from `alice`.
The second transaction is to the `compound` to start the deposit process. Compound will check that the DAI tokens are supported and will try to withdraw the desired amount of DAI from `alice`.
- If transfer succeeded, `compound` can increase local ownership for `alice` to 1000 DAI
- If transfer fails, `compound` doesn't need to do anything in current example, but maybe can notify `alice` of unsuccessful transfer.

**Technical calls** (NEP-21 standard)

1. `alice` calls `dai::set_allowance({"escrow_account_id": "compound", "allowance": "1000000000000000000000"})`.
2. `alice` calls `compound::deposit({"token_contract": "dai", "amount": "1000000000000000000000"})`. During the `deposit` call, `compound` does the following:
   1. makes async call `dai::transfer_from({"owner_id": "alice", "new_owner_id": "compound", "amount": "1000000000000000000000"})`.
   2. attaches a callback `compound::on_transfer({"owner_id": "alice", "token_contract": "dai", "amount": "1000000000000000000000"})`.
<hr/>
</details>

**High-level explanation**

Alice needs to issue 1 transaction, as opposed to 2 with a typical escrow workflow.

**Technical calls**

1. `alice` calls `dai::ft_transfer_call({"receiver_id": "compound", "amount": "1000000000000000000000", "msg": "invest"})`. During the `ft_transfer_call` call, `dai` does the following:
   1. makes async call `compound::ft_on_transfer({"sender_id": "alice", "amount": "1000000000000000000000", "msg": "invest"})`.
   2. attaches a callback `dai::ft_resolve_transfer({"sender_id": "alice", "receiver_id": "compound", "amount": "1000000000000000000000"})`.
   3. compound finishes investing, using all attached fungible tokens `compound::invest({…})` then returns the value of the tokens that weren't used or needed. In this case, Alice asked for the tokens to be invested, so it will return 0. (In some cases a method may not need to use all the fungible tokens, and would return the remainder.)
   4. the `dai::ft_resolve_transfer` function receives success/failure of the promise. If success, it will contain the unused tokens. Then the `dai` contract uses simple arithmetic (not needed in this case) and updates the balance for Alice.

#### Swapping one token for another via an Automated Market Maker (AMM) like Uniswap

Alice wants to swap 5 wrapped NEAR (wNEAR) for BNNA tokens at current market rate, with less than 2% slippage.

**Assumptions**

- The wNEAR token contract is `wnear`.
- Alice's account is `alice`.
- The AMM's contract is `amm`.
- BNNA's contract is `bnna`.
- The precision ("decimals" in the metadata standard) on wBTC contract is `10^24`.
- The 5 tokens is `5 * 10^24` or as a number is `5000000000000000000000000`.

**High-level explanation**

Alice needs to issue one transaction to wNEAR contract to transfer 5 tokens (multiplied by precision) to `amm`, specifying her desired action (swap), her destination token (BNNA) & minimum slippage (<2%) in `msg`.

Alice will probably make this call via a UI that knows how to construct `msg` in a way the `amm` contract will understand. However, it's possible that the `amm` contract itself may provide view functions which take desired action, destination token, & slippage as input and return data ready to pass to `msg` for `ft_transfer_call`. For the sake of this example, let's say `amm` implements a view function called `ft_data_to_msg`.

Alice needs to attach one yoctoNEAR. This will result in her seeing a confirmation page in her preferred NEAR wallet. NEAR wallet implementations will (eventually) attempt to provide useful information in this confirmation page, so receiver contracts should follow a strong convention in how they format `msg`. We will update this documentation with a recommendation, as community consensus emerges.

Altogether then, Alice may take two steps, though the first may be a background detail of the app she uses.

**Technical calls**

1. View `amm::ft_data_to_msg({ action: "swap", destination_token: "bnna", min_slip: 2 })`. Using [NEAR CLI](https://docs.near.org/docs/tools/near-cli):

      near view amm ft_data_to_msg '{
        "action": "swap",
        "destination_token": "bnna",
        "min_slip": 2
      }'

   Then Alice (or the app she uses) will hold onto the result and use it in the next step. Let's say this result is `"swap:bnna,2"`.

2. Call `wnear::ft_on_transfer`. Using NEAR CLI:
       near call wnear ft_transfer_call '{
         "receiver_id": "amm",
         "amount": "5000000000000000000000000",
         "msg": "swap:bnna,2"
       }' --accountId alice --amount .000000000000000000000001

   During the `ft_transfer_call` call, `wnear` does the following:

   1. Decrease the balance of `alice` and increase the balance of `amm` by 5000000000000000000000000.
   2. Makes async call `amm::ft_on_transfer({"sender_id": "alice", "amount": "5000000000000000000000000", "msg": "swap:bnna,2"})`.
   3. Attaches a callback `wnear::ft_resolve_transfer({"sender_id": "alice", "receiver_id": "compound", "amount": "5000000000000000000000000"})`.
   4. `amm` finishes the swap, either successfully swapping all 5 wNEAR within the desired slippage, or failing.
   5. The `wnear::ft_resolve_transfer` function receives success/failure of the promise. Assuming `amm` implements all-or-nothing transfers (as in, it will not transfer less-than-the-specified amount in order to fulfill the slippage requirements), `wnear` will do nothing at this point if the swap succeeded, or it will decrease the balance of `amm` and increase the balance of `alice` by 5000000000000000000000000.


## Reference-level explanation

**NOTES**:
- All amounts, balances and allowance are limited by `U128` (max value `2**128 - 1`).
- Token standard uses JSON for serialization of arguments and results.
- Amounts in arguments and results have are serialized as Base-10 strings, e.g. `"100"`. This is done to avoid JSON limitation of max integer value of `2**53`.
- The contract must track the change in storage when adding to and removing from collections. This is not included in this core fungible token standard but instead in the [Storage Standard](../Storage.md).
- To prevent the deployed contract from being modified or deleted, it should not have any access keys on its account.

**Interface**:

```javascript
/************************************/
/* CHANGE METHODS on fungible token */
/************************************/
// Simple transfer to a receiver.
//
// Requirements:
// * Caller of the method must attach a deposit of 1 yoctoⓃ for security purposes
// * Caller must have greater than or equal to the `amount` being requested
//
// Arguments:
// * `receiver_id`: the valid NEAR account receiving the fungible tokens.
// * `amount`: the number of tokens to transfer, wrapped in quotes and treated
//   like a string, although the number will be stored as an unsigned integer
//   with 128 bits.
// * `memo` (optional): for use cases that may benefit from indexing or
//    providing information for a transfer.
function ft_transfer(
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
// * `receiver_id`: the valid NEAR account receiving the fungible tokens.
// * `amount`: the number of tokens to transfer, wrapped in quotes and treated
//   like a string, although the number will be stored as an unsigned integer
//   with 128 bits.
// * `memo` (optional): for use cases that may benefit from indexing or
//    providing information for a transfer.
// * `msg`: specifies information needed by the receiving contract in
//    order to properly handle the transfer. Can indicate both a function to
//    call and the parameters to pass to that function.
function ft_transfer_call(
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
function ft_on_transfer(
    sender_id: string,
    amount: string,
    msg: string
): string {}

/****************/
/* VIEW METHODS */
/****************/

// Returns the total supply of fungible tokens as a string representing the value as an unsigned 128-bit integer.
function ft_total_supply(): string {}

// Returns the balance of an account in string form representing a value as an unsigned 128-bit integer. If the account doesn't exist must returns `"0"`.
function ft_balance_of(
    account_id: string
): string {}
```

The following behavior is required, but contract authors may name this function something other than the conventional `ft_resolve_transfer` used here.

```ts
// Finalize an `ft_transfer_call` chain of cross-contract calls.
//
// The `ft_transfer_call` process:
//
// 1. Sender calls `ft_transfer_call` on FT contract
// 2. FT contract transfers `amount` tokens from sender to receiver
// 3. FT contract calls `ft_on_transfer` on receiver contract
// 4+. [receiver contract may make other cross-contract calls]
// N. FT contract resolves promise chain with `ft_resolve_transfer`, and may
//    refund sender some or all of original `amount`
//
// Requirements:
// * Contract MUST forbid calls to this function by any account except self
// * If promise chain failed, contract MUST revert token transfer
// * If promise chain resolves with a non-zero amount given as a string,
//   contract MUST return this amount of tokens to `sender_id`
//
// Arguments:
// * `sender_id`: the sender of `ft_transfer_call`
// * `receiver_id`: the `receiver_id` argument given to `ft_transfer_call`
// * `amount`: the `amount` argument given to `ft_transfer_call`
//
// Returns a string representing a string version of an unsigned 128-bit
// integer of how many total tokens were spent by sender_id. Example: if sender
// calls `ft_transfer_call({ "amount": "100" })`, but `receiver_id` only uses
// 80, `ft_on_transfer` will resolve with `"20"`, and `ft_resolve_transfer`
// will return `"80"`.
function ft_resolve_transfer(
   sender_id: string,
   receiver_id: string,
   amount: string
): string {}
```

## Drawbacks

- The `msg` argument to `ft_transfer` and `ft_transfer_call` is freeform, which may necessitate conventions.
- The paradigm of an escrow system may be familiar to developers and end users, and education on properly handling this in another contract may be needed.

## Future possibilities

- Support for multiple token types
- Minting and burning

## History

See also the discussions:
- [Fungible token core](https://github.com/near/NEPs/discussions/146#discussioncomment-298943)
- [Fungible token metadata](https://github.com/near/NEPs/discussions/148)
- [Storage standard](https://github.com/near/NEPs/discussions/145)
