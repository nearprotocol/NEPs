# Fungible Token ([NEP-141](https://github.com/near/NEPs/issues/141))

See also the discussions:
- [Fungible token base](https://github.com/near/NEPs/discussions/146#discussioncomment-298943)
- [Fungible token metadata](https://github.com/near/NEPs/discussions/148)
- [Storage standard](https://github.com/near/NEPs/discussions/145)

Version `1.0.0`

## Summary

A standard interface for fungible tokens that allows for a normal transfer as well as a transfer and method call in a single transaction. The storage standard addresses the needs (and security) of storage staking. The [fungible token metadata standard](FungibleTokenMetadata.md) provides the fields needed for ergonomics across dApps and marketplaces.

## Motivation

NEAR Protocol uses an asynchronous, sharded runtime. This means the following:
 - Storage for different contracts and accounts can be located on the different shards.
 - Two contracts can be executed at the same time in different shards.

While this increases the transaction throughput linearly with the number of shards, it also creates some challenges for cross-contract development.
For example, if one contract wants to query some information from the state of another contract (e.g. current balance), by the time the first contract receive the balance the real balance can change.
It means in the async system, a contract can't rely on the state of another contract and assume it's not going to change.

Instead the contract can rely on temporary partial lock of the state with a callback to act or unlock, but it requires careful engineering to avoid deadlocks.
In this standard we're trying to avoid enforcing locks. A typical approach to this problem is to include an escrow system with allowances. This approach was initially developed for [NEP-21](https://github.com/near/NEPs/pull/21) which is similar to the Ethereum ERC-20 standard. There are a few issues with using an escrow as the only avenue to pay for a service with a fungible token. This frequently requires more than one transaction for common scenarios where fungible tokens are given as payment with the expectation that a method will subsequently be called.
For example, an oracle contract might be paid in fungible tokens. A client contract that wishes to use the oracle must either increase the escrow allowance before each request to the oracle contract, or allocate a large allowance that covers multiple calls. Both have drawbacks and ultimately it would be ideal to be able to send fungible tokens and call a method in a single transaction. This concern is addressed in the `ft_transfer_call` method. The power of this comes from the receiver contract working in concert with the fungible token contract in a secure way. That is, if the receiver contract abides by the standard, a single transaction may transfer and call a method.
Note: there is no reason why an escrow system cannot be included in a fungible token's implementation, but it is simply not necessary in the base standard. Escrow logic should be moved to a separate contract to handle that functionality. One reason for this is because the [Rainbow Bridge](https://near.org/blog/eth-near-rainbow-bridge/) will be transferring fungible tokens from Ethereum to NEAR, where the token locker (a factory) will be using the fungible token base standard.

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

Note, that the precision is not part of the default standard, since it's not required to perform actions. The minimum value is always 1 token.

The standard acknowledges NEAR storage staking model and accounts for the difference in storage that can be introduced by actions on this contract. Since multiple users use the contract, the contract has to account for potential storage increase. Thus every change method of the contract that can change the amount of storage must be payable.
See reference implementation for storage deposits and refunds.

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

<details style="background-color: #000; padding: 3px">
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

Alice needs to issue 1 transaction. The first one to `dai` to set an allowance for `compound` to be able to withdraw tokens from `alice`.
The second transaction is to the `compound` to start the deposit process. Compound will check that the DAI tokens are supported and will try to withdraw the desired amount of DAI from `alice`.
- If transfer succeeded, `compound` can increase local ownership for `alice` to 1000 DAI
- If transfer fails, `compound` doesn't need to do anything in current example, but maybe can notify `alice` of unsuccessful transfer.

**Technical calls**

1. `alice` calls `dai::ft_transfer_call({"receiver_id": "dai", "amount": "1000000000000000000000", "msg": "invest"})`. During the `ft_transfer_call` call, `dai` does the following:
   1. makes async call `compound::ft_on_transfer({"sender_id": "alice", "amount": "1000000000000000000000", "msg": "invest"})`.
   2. attaches a callback `dai::ft_resolve_transfer({"sender_id": "alice", "amount": "1000000000000000000000"})`.
   3. compound finishes investing, using all attached fungible tokens `compound::invest({…})` then returns the value of the tokens that weren't used or needed. In this case, Alice asked for the tokens to be invested, so it will return 0. (In some cases a method may not need to use all the fungible tokens, and would return the remainder.)
   4. the `dai::ft_resolve_transfer` function receives success/failure of the promise. If success, it will contain the unused tokens. Then the `dai` contract uses simple arithmetic (not needed in this case) and updates the balance for Alice.

#### Multi-token swap on DEX

Charlie wants to exchange his wLTC to wBTC on decentralized exchange contract. Alex wants to buy wLTC and has 80 wBTC.

**Assumptions**

- The wLTC token contract is `wltc`.
- The wBTC token contract is `wbtc`.
- The DEX contract is `dex`.
- Charlie's account is `charlie`.
- Alex's account is `alex`.
- The precision ("decimals" in the metadata standard) on both tokens contract is `10^8`.
- The amount of 9001 wLTC tokens is Alex wants is `9001 * 10^8` or as a number is `900100000000`.
- The 80 wBTC tokens is `80 * 10^8` or as a number is `8000000000`.
- Charlie has 1000000 wLTC tokens which is `1000000 * 10^8` or as a number is `100000000000000`
- DEX contract already has an open order to sell 80 wBTC tokens by `alex` towards 9001 wLTC.

<details style="background-color: #000; padding: 3px">
<summary>For this example, you may expand this section to see how a previous fungible token standard using escrows would deal with the scenario.</summary>
<hr/>

**High-level explanation** (NEP-21 standard)

Let's first setup open order by Alex on DEX. It's similar to `Token deposit to a contract` example above.
- Alex sets an allowance on wBTC to DEX
- Alex calls deposit on Dex for wBTC.
- Alex calls DEX to make an new sell order.

Then Charlie comes and decides to fulfill the order by selling his wLTC to Alex on DEX.
Charlie calls the DEX
- Charlie sets the allowance on wLTC to DEX
- Alex calls deposit on Dex for wLTC.
- Then calls DEX to take the order from Alex.

When called, DEX makes 2 async transfers calls to exchange corresponding tokens.
- DEX calls wLTC to transfer tokens DEX to Alex.
- DEX calls wBTC to transfer tokens DEX to Charlie.

**Technical calls** (NEP-21 standard)

1. `alex` calls `wbtc::set_allowance({"escrow_account_id": "dex", "allowance": "8000000000"})`.
2. `alex` calls `dex::deposit({"token": "wbtc", "amount": "8000000000"})`.
   1. `dex` calls `wbtc::transfer_from({"owner_id": "alex", "new_owner_id": "dex", "amount": "8000000000"})`
3. `alex` calls `dex::trade({"have": "wbtc", "have_amount": "8000000000", "want": "wltc", "want_amount": "900100000000"})`.
4. `charlie` calls `wltc::set_allowance({"escrow_account_id": "dex", "allowance": "100000000000000"})`.
5. `charlie` calls `dex::deposit({"token": "wltc", "amount": "100000000000000"})`.
   1. `dex` calls `wltc::transfer_from({"owner_id": "charlie", "new_owner_id": "dex", "amount": "100000000000000"})`
6. `charlie` calls `dex::trade({"have": "wltc", "have_amount": "900100000000", "want": "wbtc", "want_amount": "8000000000"})`.
   - `dex` calls `wbtc::transfer({"new_owner_id": "charlie", "amount": "8000000000"})`
   - `dex` calls `wltc::transfer({"new_owner_id": "alex", "amount": "900100000000"})`

<hr/>
</details>

**High-level explanation**

Let's first set up an open order by Alex on the DEX. It's similar to `Token deposit to a contract` example above.
- Alex sets an allowance on wBTC to DEX
- Alex calls deposit on Dex for wBTC.
- Alex calls DEX to make an new sell order.

Then Charlie comes and decides to fulfill the order by selling his wLTC to Alex on DEX.
Charlie calls the DEX
- Charlie sets the allowance on wLTC to DEX
- Alex calls deposit on Dex for wLTC.
- Then calls DEX to take the order from Alex.

When called, DEX makes 2 async transfers calls to exchange corresponding tokens.
- DEX calls wLTC to transfer tokens DEX to Alex.
- DEX calls wBTC to transfer tokens DEX to Charlie.

**Technical calls**

1. `alex` calls `wbtc::ft_transfer_call({"receiver_id": "dex", "amount": "8000000000", "msg": "deposit"})`.
   1. makes async call `dex::ft_on_transfer({"sender_id": "alex", "amount": "8000000000", "msg": "deposit"})`.
   2. attaches a callback `wbtc::ft_resolve_transfer({"sender_id": "alex", "amount": "8000000000"})`.
   3. the deposit completes on `dex`, all fungible tokens are used, so it returns `0` as the number of unused tokens back to `wbtc`.
   4. the `wbtc::ft_resolve_transfer` function receives success as the execution outcome and `0` as the value. This means 8000000000 will be subtracted from `alex`'s balance.
2. `alex` calls `dex::trade({"have": "wbtc", "have_amount": "8000000000", "want": "wltc", "want_amount": "900100000000"})`.
3. `charlie` calls `wltc::ft_transfer_call({"receiver_id": "dex", "amount": "100000000000000", "msg": "deposit"})`.
   1. makes async call `dex::ft_on_transfer({"sender_id": "charlie", "amount": "100000000000000", "msg": "deposit"})`.
   2. attaches a callback `wltc::ft_resolve_transfer({"sender_id": "charlie", "amount": "100000000000000"})`.
   3. the deposit completes on `dex`, all fungible tokens are used, so it returns `0` as the number of unused tokens back to `wltc`.
   4. the `wltc::ft_resolve_transfer` function receives success as the execution outcome and `0` as the value. This means 100000000000000 will be subtracted from `alex`'s balance.   
4. `charlie` calls `dex::trade({"have": "wltc", "have_amount": "900100000000", "want": "wbtc", "want_amount": "8000000000"})`.
   - `dex` calls `wbtc::transfer({"new_owner_id": "charlie", "amount": "8000000000"})`
   - `dex` calls `wltc::transfer({"new_owner_id": "alex", "amount": "900100000000"})`

## Reference-level explanation

**NOTES**:
- All amounts, balances and allowance are limited by `U128` (max value `2**128 - 1`).
- Token standard uses JSON for serialization of arguments and results.
- Amounts in arguments and results have are serialized as Base-10 strings, e.g. `"100"`. This is done to avoid JSON limitation of max integer value of `2**53`.
- The contract must track the change in storage when adding to and removing from collections. This is not included in this base fungible token standard but instead in the [Storage Standard](../Storage.md).
- To prevent the deployed contract from being modified or deleted, it should not have any access keys on its account.

**Interface**:

```javascript
/************************************/
/* CHANGE METHODS on fungible token */
/************************************/
// Simple transfer to a receiver. Does not call a method.
// Requirements:
// * Caller of the method must attach a deposit of 1 yoctoⓃ for security purposes
// * Caller must have greater than or equal to the `amount` being requested
// `receiver_id` is the valid NEAR account receiving the fungible tokens.
// `amount` is the number of tokens to transfer, wrapped in quotes and treated like a string, although the number will be stored as an unsigned integer with 128 bits.
// The `memo` argument is optional. It's added for use cases that may benefit from indexing or providing information for a transfer.
function ft_transfer(
    receiver_id: string,
    amount: string,
    memo: string|null
): void {}

// Transfer tokens and call a method on a receiver contract. A successful workflow will end in a success execution outcome to the callback on the same contract at the method `ft_resolve_transfer`.
// Requirements:
// * Caller of the method must attach a deposit of 1 yoctoⓃ for security purposes
// * Caller must have greater than or equal to the `amount` being requested
// * The receiving contract must implement `ft_on_transfer` according to the standard.
// * This contract must implement `ft_resolve_transfer` according to the standard.
// `msg` is an argument that may specify any information expected by the receiving contract in order to properly handle the function. It may provide details on which method to call on the receiving contract, extra parameters, etc.
function ft_transfer_call(
   receiver_id: string,
   amount: string,
   msg: string,
   memo: string|null
): Promise {}

// This function isn't called directly and shall implement logic ensuring it's called by "itself" as a callback to the promise sent to the receiver contract.
// Here the `sender_id` will be the sender from the `ft_transfer_call` method call. The `receiver_id` and `amount` will be the same as the values provided to `ft_transfer_call`.
// This returns a string representing a string version of an unsigned 128-bit integer of how many tokens were returned. (This value may be different than `amount`.)
function ft_resolve_transfer(
   sender_id: string,
   receiver_id: string,
   amount: string
): string {}

/****************************************/
/* CHANGE METHODS on receiving contract */
/****************************************/

// This function is implemented on the receving contract.
// As mentioned, the `msg` argument contains information necessary for the receiving contract to know how to process the request. This may include method names and/or arguments. 
// Returns a promise with a value. The value is the number of unused tokens. For instance, if `amount` is 10 but only 9 are needed, it will return 1.
function ft_on_transfer(
    sender_id: string,
    amount: string,
    msg: string
): Promise {}

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

## Drawbacks

- The `msg` argument to `ft_transfer` and `ft_transfer_call` is freeform, which may necessitate conventions.
- The paradigm of an escrow system may be familiar to developers and end users, and education on properly handling this in another contract may be needed.

## Future possibilities

- Support for multiple token types
- Minting and burning