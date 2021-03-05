# Multi-Fungible-Tok Contract ([NEP-138](https://github.com/nearprotocol/NEPs/pull/138))

Version `0.1.0`

## Summary
[summary]: #summary

A standard interface for a contract handling multiple fungible tokens.

This standard does not include functions to mint and burn tokens. It only declares the interface to create accounts, trasnfer tokens, and query balances.

## Changelog

### `0.0.1`

- Initial Proposal

## Motivation
[motivation]: #motivation

1. The combination of scaling by sharding and the cross-contract calls mechanics in NEAR (async, no 2PC) create certain condiions where in some cases is recommendable to manage multiple tokens in the same contract instead of the standard Ethereum-way of a contract per token.

2. The [meta-pool contract](https://narwallets.github.io/meta-pool/) is an example of this kind of situation. The contract manages three "tokens": NEAR, stNEAR and $META.

Prior art:
- [#141 Fungible token standard](https://github.com/near/NEPs/issues/141),  [#136 NEP Interactive Fungible Token](https://github.com/robert-zaremba/nep-136-fungible-token), [#122 Allowance-free vault-based token](https://github.com/near/NEPs/issues/122), [#21 Fungible Token](https://github.com/near/NEPs/blob/master/specs/Standards/Tokens/FungibleToken.md)

## Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

A contract implementing this standard would allow users to:
- Use a pre-existent deployment of this contract to create a new Fungible Token
- Ease up the creation of fungible tokens, reusing audited stable code, needing less configurationin wallets.
- Ease up the creation of circunscribed Swaps between the tokens the contract serves

We should be able to do the following:
- Initialize contract once.
- Create and delete tokens
- Create and delete user accounts
- Transfer tokens between accounts
- Transfer tokens to a contract with a callback
- Get the current balance for any account
- Get full information about any of the tokens

There are a few concepts in the scenarios above:
- **Total supply**. It's the total number of tokens in circulation for each token.
- **Contract owner**. An account ID that owns the contract.
- **Token owner**. An account ID that owns operations on a specific token.
- **Balance**. Some amount of tokens.
- **Account**. A single account can have a balance on every token the contract manages.
- **Account owner**. An near account ID that owns the account and the token balances.
- **Transfer**. Action that moves some amount of a specific token from one account to another account.

Note, that the precision is always fixed at 24 decimal places (as the native token NEAR). The minimun amount is 1 Yocto equivalent to 1e-24 token.

The standard acknowledges NEAR storage backing model. Since multiple users use the contract, the contract has to account for potential
storage increase. Thus the create_account method than increases storage requirements is payable and requires some attached NEAR (less than 1 NEAR)

### Simple transfer

Alice wants to send 5 wBTC tokens to Bob.

**Assumptions**

- Alice's account is `alice`.
- Bob's account is `bob`.
- The 5 tokens is `5 * 10^24`, a 5 followed by 24 zeroes.

**High-level explanation**

Alice needs to issue one transaction to transfer 5 tokens to Bob.

**Technical calls**

1. `alice` calls `transfer_to_user({"receiver_id": "bob", "symbol":"wBTC", "amount": "50000000000000000000000"})`.

### Token deposit to a contract

Alice wants to deposit 1000 nDAI tokens to a compound interest contract to earn extra tokens.

**Assumptions**

- The nDAI token symbol is `nDAI`.
- Alice's account is `alice`.
- The compound interest contract is `compound.near`.
- The 1000 tokens is `1000 * 10^24` or 10 followed by 24 zeroes.
- The compound contract can work with multiple token types.
- The compound contract already has an account in this multi-fungible-token contract.

**High-level explanation**

Alice needs to issue one transaction calling `transfer_to_contract`. by calling `transfer_to_contract` the reciving contract, comppund.near will receive a call after the transfer in the form: `call compound.near.on_multifuntok_transfer(sender_id: "alice', symbol:"nDAI", amount: "100000000000000000000000", memo:'xxxx')`.

The receiving contract must process alice's deposit and complete the call. If the call fails, the transaction is reversed. The receiving contract can choose to make a partial return transfer if the sent amount exceedes what's required.
- The multi-fun-tok contract transfers from `alice` to `compound.near`
- The multi-fun-tok transfers calls `compound.nearcompound.near.on_multifuntok_transfer(sender_id: "alice', symbol:"nDAI", amount: "100000000000000000000000", memo:'xxxx')`
- If the call fails, the multi-fun-tok contract undoes the transfer, transferring back from `compound.near` to `alice` 
- If the call completes, the multi-fun-tok contract doesn't need to do anything.

**Technical calls**

1. `alice` calls `transfer_to_contract({"receiver_id": "compound.near", "symbol":"nDAI", "amount": "100000000000000000000000"})`.
1. The multi-fun-tok contract transfers from `alice` to `compound.near`
   1. The multi-fun-tok contract calls `compound.nearcompound.near.on_multifuntok_transfer(sender_id: "alice', symbol:"nDAI", amount: "100000000000000000000000", memo:'xxxx')`
   1. If the call fails, the multi-fun-tok contract undoes the transfer, transferring back from `compound.near` to `alice` 

### dual-token internal swap (future expansion)

Charlie wants to exchange 80 wLTC for at least 530 wBTC, both tokens are managed in this contract.

**Assumptions**

- The wLTC token symbol is `wLTC`.
- The wBTC token symbol is `wBTC`.
- There's an `wLTC/wBTC` Liquidity Pool.

**High-level explanation**

- Charlie calls `swap('wLTC',80*1e24, 'wBTC',530*1e24) -> u128String`
- If the price allows the output token amount to be at least the amount requested, a swap is made using the liquidity pool, the fn returns the actual `wBTC` amount trasnferred to Charlie (>= of the amount requested, that is >= 530*1e24).
- If the price does not allows the output token requested amount, the transaction is rejected asking Charlie to lower his required amount

**Technical calls**

1. `swap({"input_symbol": "wLTC", "input_amount":80*1e24, "output_symbol":"wBTC", "output_amount":530*1e24}) -> U128String //recevied output amount`.

## Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

* All amounts, balances and allowance are limited by U128 (max value 2**128 - 1).
* All tokens use 24 decimal places as precision, to mimic native NEAR preceision, simplify mamangement, log interpretations, comparisions, and wallets showing token balances.
* u128 Amounts in arguments and results are serialized as Base-10 strings (U128String), e.g. 100_000_000_000:u128 => "100000000000":U128String. This is done to avoid JSON limitation of max precision of 15 digits (f64), versus 38 digits for u128.
* The contract migth require users to create accounts before operating. The create_account call migth require the user to attach enough NEAR to cover the storage cost of creating an account. This is done to prevent a denial of service attack on the contract by taking all available storage. The unused tokens from the attached deposit are also refunded, so it's safe to attach more deposit than required.
* The contract can choose to not enforce the previous requirement up-to a certain point or for some kind of accounts. For example, the diversifying-pool can waive this requirement for lockup-accounts or for all accounts to ease the oboarding of users.
* To prevent the deployed contract from being modified or deleted, it should not have any access keys on its account.


### Reference Implementations:
#### Rust:
```rust

type U128String = U128;

/// one for Each served token
struct SymbolInfo {
    symbol: String,     // token symbol
    name: String,       // token name
    total_supply: U128String, //total circulating supply
    owner_account_id: String, // owner of this particular token
    reference: String,  // URL to additional resources about the token.
}

/// NEP-138 Multiple Fungible Tokens Contract
pub trait MultiFunToken {
    
    //---------TOKENS---------------

    /// Creates a new Fungible Token 
    /// Requirements:
    /// * Caller can only by the main owner
    pub fn create_token(&mut self, symbol_info: SymbolInfo);

    /// Deletes a Fungible Token 
    /// Requirements:
    /// * Caller can be the main owner or the token owner
    /// * Symbol.total_supply == 0
    pub fn delete_token(&mut self, symbol: String);

    //---------ACCOUNTS---------------

    /// Creates an internal `Account` record. Every account has a balance for each one of the served tokens
    /// Requirements:
    /// * Caller must attach enough NEAR to cover storage cost at the fixed storage price defined in the contract.
    [#payable]
    pub fn create_account(&mut self, receiver_id: AccountId);

    // deletes an account and transfer all balances to beneficiary_id. beneficiary_id must pre-exists
    // Notes: account_to_delete_id is superflous on purpose
    // assert!(`account_to_delete_id`==`predecessor_id`)
    pub fn delete_account(&mut self, account_to_delete_id: AccountId, beneficiary_id: AccountId);

    /// Transfer `amount` of tok tokens from the caller of the contract (`predecessor_id`) to `receiver_id`.
    /// Requirements:
    /// * receiver_id must pre-exist
    pub fn transfer_to_user(&mut self, receiver_id: AccountId, symbol:String, amount: U128String);

    /// Transfer `amount` of symbol tokens from the caller of the contract (`predecessor_id`) to `receiver_id`.
    /// Requirements:
    /// * receiver_id must pre-exist
    /// * receiver_id must be a contract and must respond to `on_multifuntok_transfer(sender_id: AccountId, symbol:String, amount: U128, memo:String)`
    /// * if receiver_id is not a contract or `on_multifuntok_transfer` fails, the transfer is rolled-back
    pub fn transfer_to_contract(&mut self, receiver_id: AccountId, symbol:String, amount: U128String, memo:String);

    //---------VIEW METHODS-------------

    /// return the list of all tokens this contract serves
    pub fn get_symbols(&self) -> Vec<SymbolInfo>;

    /// Returns info & total supply of tokens of a symbol
    pub fn get_symbol(&self, symbol:String) -> SymbolInfo;

    /// Checks if account already exists
    pub fn account_exists(&self, account_id:Account) -> bool;

    /// return all symbols & balances for an account
    pub fn get_balances(&self, account_id:Account) -> Vec<SymbolBalance>;

    /// Returns balance of the `owner_id` account & token.
    pub fn get_balance(&self, account_id: AccountId, symbol:String) -> U128String;

}

///item returned from get_balances -> Vec<SymbolBalance>;
struct SymbolBalance {
    symbol: String,      //token symbol
    balance: U128String, //account balance
}

    //---------INTERACTING CONTRACTS-------------

/// Interface for recipient contract on multi-fungible-token transfers.
pub trait MultiFunTokenTrasnferRecipient {
    fn on_multifuntok_transfer(sender_id: AccountId, symbol:String, amount: U128, memo:String);
}
```

####  ts/pseudocode:
```typescript
```

## Drawbacks
[drawbacks]: #drawbacks

TBD 

## Unresolved questions
[unresolved-questions]: #unresolved-questions

TBD

## Future possibilities
[future-possibilities]: #future-possibilities

* Automated swap mechanisms. Since several tokens are served from the same contract, a standard uniswap mechanism can be coded to serve any combination of the serverd tokens.

