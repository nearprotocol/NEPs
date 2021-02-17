# Fungible Token ([NEP-21](https://github.com/nearprotocol/NEPs/pull/21))

Version `0.2.0`

## Summary
[summary]: #summary

A standard interface for fungible tokens allowing for ownership, escrow and transfer, specifically targeting third-party marketplace integration.

## Changelog

### `0.2.0`

- Introduce storage deposits. Make every change method payable (able receive attached deposits with the function calls). It requires the caller to attach enough deposit to cover potential storage increase. See [core-contracts/#47](https://github.com/near/core-contracts/issues/47)
- Replace `set_allowance` with `inc_allowance` and `dec_allowance` to address the issue of allowance front-running. See [core-contracts/#49](https://github.com/near/core-contracts/issues/49)
- Validate `owner_id` account ID. See [core-contracts/#54](https://github.com/near/core-contracts/issues/54)
- Enforce that the `new_owner_id` is different from the current `owner_id` for transfer. See [core-contracts/#55](https://github.com/near/core-contracts/issues/55)

## Motivation
[motivation]: #motivation

NEAR Protocol uses an asynchronous sharded Runtime. This means the following:
 - Storage for different contracts and accounts can be located on the different shards.
 - Two contracts can be executed at the same time in different shards.

While this increases the transaction throughput linearly with the number of shards, it also creates some challenges for cross-contract development.
For example, if one contract wants to query some information from the state of another contract (e.g. current balance), by the time the first contract receive the balance the real balance can change.
It means in the async system, a contract can't rely on the state of other contract and assume it's not going to change.

Instead the contract can rely on temporary partial lock of the state with a callback to act or unlock, but it requires careful engineering to avoid dead locks.
In this standard we're trying to avoid enforcing locks, since most actions can still be completed without locks by transferring ownership to an escrow account.

Prior art:
- [ERC-20 standard](https://eips.ethereum.org/EIPS/eip-20)
- NEP#4 NEAR NFT standard: [nearprotocol/neps#4](https://github.com/nearprotocol/neps/pull/4)
- For latest lock proposals see [Safes (#26)](https://github.com/nearprotocol/neps/pull/26)

## Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

We should be able to do the following:
- Initialize contract once. The given total supply will be owned by the given account ID.
- Get the total supply.
- Transfer tokens to a new user.
- Set a given allowance for an escrow account ID.
  - Escrow will be able to transfer up this allowance from your account.
  - Get current balance for a given account ID.
- Transfer tokens from one user to another.
- Get the current allowance for an escrow account on behalf of the balance owner. This should only be used in the UI, since a contract shouldn't rely on this temporary information.

There are a few concepts in the scenarios above:
- **Total supply**. It's the total number of tokens in circulation.
- **Balance owner**. An account ID that owns some amount of tokens.
- **Balance**. Some amount of tokens.
- **Transfer**. Action that moves some amount from one account to another account.
- **Escrow**. A different account from the balance owner who has permission to use some amount of tokens.
- **Allowance**. The amount of tokens an escrow account can use on behalf of the account owner.

Note, that the precision is not part of the default standard, since it's not required to perform actions. The minimum
value is always 1 token.

The standard acknowledges NEAR storage staking model and accounts for the difference in storage that can be introduced
by actions on this contract. Since multiple users use the contract, the contract has to account for potential
storage increase. Thus every change method of the contract that can change the amount of storage must be payable.
See reference implementation for storage deposits and refunds.

### Simple transfer

Alice wants to send 5 wBTC tokens to Bob.

**Assumptions**

- The wBTC token contract is `wbtc`.
- Alice's account is `alice`.
- Bob's account is `bob`.
- The precision on wBTC contract is `10^8`.
- The 5 tokens is `5 * 10^8` or as a number is `500000000`.

**High-level explanation**

Alice needs to issue one transaction to wBTC contract to transfer 5 tokens (multiplied by precision) to Bob.

**Technical calls**

1. `alice` calls `wbtc::transfer({"new_owner_id": "bob", "amount": "500000000"})`.

### Token deposit to a contract

Alice wants to deposit 1000 DAI tokens to a compound interest contract to earn extra tokens.

**Assumptions**

- The DAI token contract is `dai`.
- Alice's account is `alice`.
- The compound interest contract is `compound`.
- The precision on DAI contract is `10^18`.
- The 1000 tokens is `1000 * 10^18` or as a number is `1000000000000000000000`.
- The compound contract can work with multiple token types.

**High-level explanation**

Alice needs to issue 2 transactions. The first one to `dai` to set an allowance for `compound` to be able to withdraw tokens from `alice`.
The second transaction is to the `compound` to start the deposit process. Compound will check that the DAI tokens are supported and will try to withdraw the desired amount of DAI from `alice`.
- If transfer succeeded, `compound` can increase local ownership for `alice` to 1000 DAI
- If transfer fails, `compound` doesn't need to do anything in current example, but maybe can notify `alice` of unsuccessful transfer.

**Technical calls**

1. `alice` calls `dai::set_allowance({"escrow_account_id": "compound", "allowance": "1000000000000000000000"})`.
1. `alice` calls `compound::deposit({"token_contract": "dai", "amount": "1000000000000000000000"})`. During the `deposit` call, `compound` does the following:
   1. makes async call `dai::transfer_from({"owner_id": "alice", "new_owner_id": "compound", "amount": "1000000000000000000000"})`.
   1. attaches a callback `compound::on_transfer({"owner_id": "alice", "token_contract": "dai", "amount": "1000000000000000000000"})`.

### Multi-token swap on DEX

Charlie wants to exchange his wLTC to wBTC on decentralized exchange contract. Alex wants to buy wLTC and has 80 wBTC.

**Assumptions**

- The wLTC token contract is `wltc`.
- The wBTC token contract is `wbtc`.
- The DEX contract is `dex`.
- Charlie's account is `charlie`.
- Alex's account is `alex`.
- The precision on both tokens contract is `10^8`.
- The amount of 9001 wLTC tokens is Alex wants is `9001 * 10^8` or as a number is `900100000000`.
- The 80 wBTC tokens is `80 * 10^8` or as a number is `8000000000`.
- Charlie has 1000000 wLTC tokens which is `1000000 * 10^8` or as a number is `100000000000000`
- Dex contract already has an open order to sell 80 wBTC tokens by `alex` towards 9001 wLTC.
- Without Safes implementation, DEX has to act as an escrow and hold funds of both users before it can do an exchange.

**High-level explanation**

Let's first setup open order by Alex on DEX. It's similar to `Token deposit to a contract` example above.
- Alex sets an allowance on wBTC to DEX
- Alex calls deposit on Dex for wBTC.
- Alex calls DEX to make an new sell order.

Then Charlie comes and decides to fulfill the order by selling his wLTC to Alex on DEX.
Charlie calls the DEX
- Charlie sets the allowance on wLTC to DEX
- Charlie calls deposit on Dex for wLTC.
- Charlie calls DEX to take the order from Alex.

When called, DEX makes 2 async transfers calls to exchange corresponding tokens.
- DEX calls wLTC to transfer tokens DEX to Alex.
- DEX calls wBTC to transfer tokens DEX to Charlie.

**Technical calls**

1. `alex` calls `wbtc::set_allowance({"escrow_account_id": "dex", "allowance": "8000000000"})`.
1. `alex` calls `dex::deposit({"token": "wbtc", "amount": "8000000000"})`.
   1. `dex` calls `wbtc::transfer_from({"owner_id": "alex", "new_owner_id": "dex", "amount": "8000000000"})`
1. `alex` calls `dex::trade({"have": "wbtc", "have_amount": "8000000000", "want": "wltc", "want_amount": "900100000000"})`.
1. `charlie` calls `wltc::set_allowance({"escrow_account_id": "dex", "allowance": "100000000000000"})`.
1. `charlie` calls `dex::deposit({"token": "wltc", "amount": "100000000000000"})`.
   1. `dex` calls `wltc::transfer_from({"owner_id": "charlie", "new_owner_id": "dex", "amount": "100000000000000"})`
1. `charlie` calls `dex::trade({"have": "wltc", "have_amount": "900100000000", "want": "wbtc", "want_amount": "8000000000"})`.
   - `dex` calls `wbtc::transfer({"new_owner_id": "charlie", "amount": "8000000000"})`
   - `dex` calls `wltc::transfer({"new_owner_id": "alex", "amount": "900100000000"})`


## Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

The full implementation in Rust can be found here: [fungible-token](https://github.com/nearprotocol/near-sdk-rs/blob/master/examples/fungible-token/src/lib.rs)

**NOTES:**
- All amounts, balances and allowance are limited by `U128` (max value `2**128 - 1`).
- Token standard uses JSON for serialization of arguments and results.
- Amounts in arguments and results have are serialized as Base-10 strings, e.g. `"100"`. This is done to avoid
JSON limitation of max integer value of `2**53`.
- The contract tracks the change in storage before and after the call. If the storage increases,
the contract requires the caller of the contract to attach enough deposit to the function call
to cover the storage cost.
This is done to prevent a denial of service attack on the contract by taking all available storage.
It's because the gas cost of adding new escrow account is cheap, many escrow allowances can be added until the contract
runs out of storage.
If the storage decreases, the contract will issue a refund for the cost of the released storage.
The unused tokens from the attached deposit are also refunded, so it's safe to attach more deposit than required.
- To prevent the deployed contract from being modified or deleted, it should not have any access
keys on its account.


Interface:

```rust
/******************/
/* CHANGE METHODS */
/******************/

/// Increments the `allowance` for `escrow_account_id` by `amount` on the account of the caller of this contract
/// (`predecessor_id`) who is the balance owner.
/// Requirements:
/// * Caller of the method has to attach deposit enough to cover storage difference at the
///   fixed storage price defined in the contract.
#[payable]
pub fn inc_allowance(&mut self, escrow_account_id: AccountId, amount: U128);

/// Decrements the `allowance` for `escrow_account_id` by `amount` on the account of the caller of this contract
/// (`predecessor_id`) who is the balance owner.
/// Requirements:
/// * Caller of the method has to attach deposit enough to cover storage difference at the
///   fixed storage price defined in the contract.
#[payable]
pub fn dec_allowance(&mut self, escrow_account_id: AccountId, amount: U128);

/// Transfers the `amount` of tokens from `owner_id` to the `new_owner_id`.
/// Requirements:
/// * `amount` should be a positive integer.
/// * `owner_id` should have balance on the account greater or equal than the transfer `amount`.
/// * If this function is called by an escrow account (`owner_id != predecessor_account_id`),
///   then the allowance of the caller of the function (`predecessor_account_id`) on
///   the account of `owner_id` should be greater or equal than the transfer `amount`.
/// * Caller of the method has to attach deposit enough to cover storage difference at the
///   fixed storage price defined in the contract.
#[payable]
pub fn transfer_from(&mut self, owner_id: AccountId, new_owner_id: AccountId, amount: U128);


/// Transfer `amount` of tokens from the caller of the contract (`predecessor_id`) to
/// `new_owner_id`.
/// Act the same was as `transfer_from` with `owner_id` equal to the caller of the contract
/// (`predecessor_id`).
/// Requirements:
/// * Caller of the method has to attach deposit enough to cover storage difference at the
///   fixed storage price defined in the contract.
#[payable]
pub fn transfer(&mut self, new_owner_id: AccountId, amount: U128);

/****************/
/* VIEW METHODS */
/****************/

/// Returns total supply of tokens.
pub fn get_total_supply(&self) -> U128;

/// Returns balance of the `owner_id` account.
pub fn get_balance(&self, owner_id: AccountId) -> U128;

/// Returns current allowance of `escrow_account_id` for the account of `owner_id`.
///
/// NOTE: Other contracts should not rely on this information, because by the moment a contract
/// receives this information, the allowance may already be changed by the owner.
/// So this method should only be used on the front-end to see the current allowance.
pub fn get_allowance(&self, owner_id: AccountId, escrow_account_id: AccountId) -> U128;
```

## Drawbacks
[drawbacks]: #drawbacks

- Current interface doesn't have minting, precision (decimals), naming. But it should be done as extensions, e.g. a Precision extension.
- It's not possible to exchange tokens without transferring them to escrow first.
- It's not possible to transfer tokens to a contract with a single transaction without setting the allowance first.
It should be possible if we introduce `transfer_with` function that transfers tokens and calls escrow contract. It needs to handle result of the execution and contracts have to be aware of this API.


## Future possibilities
[future-possibilities]: #future-possibilities

- Support for multiple token types
- Minting and burning
- Precision, naming and short token name.
