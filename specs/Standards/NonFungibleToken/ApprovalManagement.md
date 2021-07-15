# Non-Fungible Token Approval Management ([NEP-178](https://github.com/near/NEPs/discussions/178))

Version `1.0.0`

## Summary

A system for allowing a set of users or contracts to transfer specific Non-Fungible Tokens on behalf of an owner. Similar to approval management systems in standards like [ERC-721].

  [ERC-721]: https://eips.ethereum.org/EIPS/eip-721

## Motivation

People familiar with [ERC-721] may expect to need an approval management system for basic transfers, where a simple transfer from Alice to Bob requires that Alice first _approve_ Bob to spend one of her tokens, after which Bob can call `transfer_from` to actually transfer the token to himself.

NEAR's [core Non-Fungible Token standard](Core.md) includes good support for safe atomic transfers without such complexity. It even provides "transfer and call" functionality (`nft_transfer_call`) which allows a specific token to be "attached" to a call to a separate contract. For many Non-Fungible Token workflows, these options may circumvent the need for a full-blown Approval Managament system.

However, some Non-Fungible Token developers, marketplaces, dApps, or artists may require greater control. This standard provides a uniform interface allowing token owners to approve other NEAR accounts, whether individuals or contracts, to transfer specific tokens on the owner's behalf.

Prior art:

- Ethereum's [ERC-721]
- [NEP-4](https://github.com/near/NEPs/pull/4), NEAR's old NFT standard that does not include approvals per token ID

## Example Scenarios

Let's consider some examples. Our cast of characters & apps:

* Alice: has account `alice` with no contract deployed to it
* Bob: has account `bob` with no contract deployed to it
* NFT: a contract with account `nft`, implementing only the [Core NFT standard](Core.md) with this Approval Management extension
* Market: a contract with account `market` which sells tokens from `nft` as well as other NFT contracts
* Bazaar: similar to Market, but implemented differently (spoiler alert: has no `nft_on_approve` function!), has account `bazaar`

Alice and Bob are already [registered](../StorageManagement.md) with NFT, Market, and Bazaar, and Alice owns a token on the NFT contract with ID=`"1"`.

Let's examine the technical calls through the following scenarios:

1. [Simple approval](#1-simple-approval): Alice approves Bob to transfer her token.
2. [Approval with cross-contract call (XCC)](#2-approval-with-cross-contract-call): Alice approves Market to transfer one of her tokens and passes `msg` so that NFT will call `nft_on_approve` on Market's contract.
3. [Approval with XCC, edge case](#3-approval-with-cross-contract-call-edge-case): Alice approves Bazaar and passes `msg` again, but what's this? Bazaar doesn't implement `nft_on_approve`, so Alice sees an error in the transaction result. Not to worry, though, she checks `nft_is_approved` and sees that she did successfully approve Bazaar, despite the error.
4. [Approval IDs](#4-approval-ids): Bob buys Alice's token via Market.
5. [Approval IDs, edge case](#5-approval-ids-edge-case): Bob transfers same token back to Alice, Alice re-approves Market & Bazaar. Bazaar has an outdated cache. Bob tries to buy from Bazaar at the old price.
6. [Revoke one](#6-revoke-one): Alice revokes Market's approval for this token.
7. [Revoke all](#7-revoke-all): Alice revokes all approval for this token.

### 1. Simple Approval

Alice approves Bob to transfer her token.

**High-level explanation**

1. Alice approves Bob
2. Alice queries the token to verify
3. Alice verifies a different way

**Technical calls**

1. Alice calls `nft::nft_approve({ "token_id": "1", "account_id": "bob" })`. She attaches 1 yoctoⓃ, (.000000000000000000000001Ⓝ). Using [NEAR CLI](https://docs.near.org/docs/tools/near-cli) to make this call, the command would be:

       near call nft nft_approve \
         '{ "token_id": "1", "account_id": "bob" }' \
         --accountId alice --amount .000000000000000000000001

   The response:

       ''

2. Alice calls view method `nft_token`:

       near view nft nft_token \
         '{ "token_id": "1" }'

   The response:

       {
         "id": "1",
         "owner_id": "alice.near",
         "approvals": {
           "bob": 1,
         }
       }

3. Alice calls view method `nft_is_approved`:

       near view nft nft_is_approved \
         '{ "token_id": "1", "approved_account_id": "bob" }'

   The response:

       true

### 2. Approval with cross-contract call

Alice approves Market to transfer one of her tokens and passes `msg` so that NFT will call `nft_on_approve` on Market's contract. She probably does this via Market's frontend app which would know how to construct `msg` in a useful way.

**High-level explanation**

1. Alice calls `nft_approve` to approve `market` to transfer her token, and passes a `msg`
2. Since `msg` is included, `nft` will schedule a cross-contract call to `market`
3. Market can do whatever it wants with this info, such as listing the token for sale at a given price. The result of this operation is returned as the promise outcome to the original `nft_approve` call.

**Technical calls**

1. Using near-cli:

       near call nft nft_approve '{
         "token_id": "1",
         "account_id": "market",
         "msg": "{\"action\": \"list\", \"price\": \"100\", \"token\": \"nDAI\" }"
       }' --accountId alice --amount .000000000000000000000001

   At this point, near-cli will hang until the cross-contract call chain fully resolves, which would also be true if Alice used a Market frontend using [near-api-js](https://docs.near.org/docs/develop/front-end/near-api-js). Alice's part is done, though. The rest happens behind the scenes.

2. `nft` schedules a call to `nft_on_approve` on `market`. Using near-cli notation for easy cross-reference with the above, this would look like:

       near call market nft_on_approve '{
         "token_id": "1",
         "owner_id": "alice",
         "approval_id": 2,
         "msg": "{\"action\": \"list\", \"price\": \"100\", \"token\": \"nDAI\" }"
       }' --accountId nft

3. `market` now knows that it can sell Alice's token for 100 [nDAI](https://explorer.mainnet.near.org/accounts/6b175474e89094c44da98b954eedeac495271d0f.factory.bridge.near), and that when it transfers it to a buyer using `nft_transfer`, it can pass along the given `approval_id` to ensure that Alice hasn't changed her mind. It can schedule any further cross-contract calls it wants, and if it returns these promises correctly, Alice's initial near-cli call will resolve with the outcome from the final step in the chain. If Alice actually made this call from a Market frontend, the frontend can use this return value for something useful.

### 3. Approval with cross-contract call, edge case

Alice approves Bazaar and passes `msg` again. Maybe she actually does this via near-cli, rather than using Bazaar's frontend, because what's this? Bazaar doesn't implement `nft_on_approve`, so Alice sees an error in the transaction result.

Not to worry, though, she checks `nft_is_approved` and sees that she did successfully approve Bazaar, despite the error. She will have to find a new way to list her token for sale in Bazaar, rather than using the same `msg` shortcut that worked for Market.

**High-level explanation**

1. Alice calls `nft_approve` to approve `bazaar` to transfer her token, and passes a `msg`.
2. Since `msg` is included, `nft` will schedule a cross-contract call to `bazaar`.
3. Bazaar doesn't implement `nft_on_approve`, so this call results in an error. The approval still worked, but Alice sees an error in her near-cli output.
4. Alice checks if `bazaar` is approved, and sees that it is, despite the error.

**Technical calls**

1. Using near-cli:

       near call nft nft_approve '{
         "token_id": "1",
         "account_id": "bazaar",
         "msg": "{\"action\": \"list\", \"price\": \"100\", \"token\": \"nDAI\" }"
       }' --accountId alice --amount .000000000000000000000001

2. `nft` schedules a call to `nft_on_approve` on `market`. Using near-cli notation for easy cross-reference with the above, this would look like:

       near call bazaar nft_on_approve '{
         "token_id": "1",
         "owner_id": "alice",
         "approval_id": 3,
         "msg": "{\"action\": \"list\", \"price\": \"100\", \"token\": \"nDAI\" }"
       }' --accountId nft

3. 💥 `bazaar` doesn't implement this method, so the call results in an error. Alice sees this error in the output from near-cli.

4. Alice checks if the approval itself worked, despite the error on the cross-contract call:

       near view nft nft_is_approved \
         '{ "token_id": "1", "approved_account_id": "bazaar" }'

   The response:

       true

### 4. Approval IDs

Bob buys Alice's token via Market. Bob probably does this via Market's frontend, which will probably initiate the transfer via a call to `ft_transfer_call` on the nDAI contract to transfer 100 nDAI to `market`. Like the NFT standard's "transfer and call" function, [Fungible Token](../FungibleToken/Core.md)'s `ft_transfer_call` takes a `msg` which `market` can use to pass along information it will need to pay Alice and actually transfer the NFT. The actual transfer of the NFT is the only part we care about here.

**High-level explanation**

1. Bob signs some transaction which results in the `market` contract calling `nft_transfer` on the `nft` contract, as described above. To be trustworthy and pass security audits, `market` needs to pass along `approval_id` so that it knows it has up-to-date information.

**Technical calls**

Using near-cli notation for consistency:

    near call nft nft_transfer '{
      "receiver_id": "bob",
      "token_id": "1",
      "approval_id": 2,
    }' --accountId market --amount .000000000000000000000001

### 5. Approval IDs, edge case

Bob transfers same token back to Alice, Alice re-approves Market & Bazaar, listing her token at a higher price than before. Bazaar is somehow unaware of these changes, and still stores `approval_id: 3` internally along with Alice's old price. Bob tries to buy from Bazaar at the old price. Like the previous example, this probably starts with a call to a different contract, which eventually results in a call to `nft_transfer` on `bazaar`. Let's consider a possible scenario from that point.

**High-level explanation**

Bob signs some transaction which results in the `bazaar` contract calling `nft_transfer` on the `nft` contract, as described above. To be trustworthy and pass security audits, `bazaar` needs to pass along `approval_id` so that it knows it has up-to-date information. It does not have up-to-date information, so the call fails. If the initial `nft_transfer` call is part of a call chain originating from a call to `ft_transfer_call` on a fungible token, Bob's payment will be refunded and no assets will change hands.

**Technical calls**

Using near-cli notation for consistency:

    near call nft nft_transfer '{
      "receiver_id": "bob",
      "token_id": "1",
      "approval_id": 3,
    }' --accountId bazaar --amount .000000000000000000000001

### 6. Revoke one

Alice revokes Market's approval for this token.

**Technical calls**

Using near-cli:

    near call nft nft_revoke '{
      "account_id": "market",
      "token_id": "1",
    }' --accountId alice --amount .000000000000000000000001

Note that `market` will not get a cross-contract call in this case. The implementors of the Market app should implement [cron](https://en.wikipedia.org/wiki/Cron)-type functionality to intermittently check that Market still has the access they expect.

### 7. Revoke all

Alice revokes all approval for this token.

**Technical calls**

Using near-cli:

    near call nft nft_revoke_all '{
      "token_id": "1",
    }' --accountId alice --amount .000000000000000000000001

Again, note that no previous approvers will get cross-contract calls in this case.

## Reference-level explanation

The `Token` structure returned by `nft_token` must include an `approvals` field, which is a map of account IDs to approval IDs. Using TypeScript's [Record type](https://www.typescriptlang.org/docs/handbook/utility-types.html#recordkeystype) notation:

```diff
 type Token = {
   id: string,
   owner_id: string,
+  approvals: Record<string, number>,
 };
```

Example token data:

```json
{
  "id": "1",
  "owner_id": "alice.near",
  "approvals": {
    "bob.near": 1,
    "carol.near": 2,
  }
}
```

### What is an "approval ID"?

This is a unique number given to each approval that allows well-intentioned marketplaces or other 3rd-party NFT resellers to avoid a race condition. The race condition occurs when:

1. A token is listed in two marketplaces, which are both saved to the token as approved accounts.
2. One marketplace sells the token, which clears the approved accounts.
3. The new owner sells back to the original owner.
4. The original owner approves the token for the second marketplace again to list at a new price. But for some reason the second marketplace still lists the token at the previous price and is unaware of the transfers happening.
5. The second marketplace, operating from old information, attempts to again sell the token at the old price.

Note that while this describes an honest mistake, the possibility of such a bug can also be taken advantage of by malicious parties via [front-running](https://defi.cx/front-running-ethereum/).

To avoid this possibility, the NFT contract generates a unique approval ID each time it approves an account. Then when calling `nft_transfer` or `nft_transfer_call`, the approved account passes `approval_id` with this value to make sure the underlying state of the token hasn't changed from what the approved account expects.

Keeping with the example above, say the initial approval of the second marketplace generated the following `approvals` data:

```json
{
  "id": "1",
  "owner_id": "alice.near",
  "approvals": {
    "marketplace_1.near": 1,
    "marketplace_2.near": 2,
  }
}
```

But after the transfers and re-approval described above, the token might have `approvals` as:

```json
{
  "id": "1",
  "owner_id": "alice.near",
  "approvals": {
    "marketplace_2.near": 3,
  }
}
```

The marketplace then tries to call `nft_transfer`, passing outdated information:

```bash
# oops!
near call nft-contract.near nft_transfer '{ "approval_id": 2 }'
```


### Interface

The NFT contract must implement the following methods:

```ts
/******************/
/* CHANGE METHODS */
/******************/

// Add an approved account for a specific token.
//
// Requirements
// * Caller of the method must attach a deposit of at least 1 yoctoⓃ for
//   security purposes
// * Contract MAY require caller to attach larger deposit, to cover cost of
//   storing approver data
// * Contract MUST panic if called by someone other than token owner
// * Contract MUST panic if addition would cause `nft_revoke_all` to exceed
//   single-block gas limit. See below for more info.
// * Contract MUST increment approval ID even if re-approving an account
// * If successfully approved or if had already been approved, and if `msg` is
//   present, contract MUST call `nft_on_approve` on `account_id`. See
//   `nft_on_approve` description below for details.
//
// Arguments:
// * `token_id`: the token for which to add an approval
// * `account_id`: the account to add to `approvals`
// * `msg`: optional string to be passed to `nft_on_approve`
//
// Returns void, if no `msg` given. Otherwise, returns promise call to
// `nft_on_approve`, which can resolve with whatever it wants.
function nft_approve(
  token_id: TokenId,
  account_id: string,
  msg: string|null,
): void|Promise<any> {}

// Revoke an approved account for a specific token.
//
// Requirements
// * Caller of the method must attach a deposit of 1 yoctoⓃ for security
//   purposes
// * If contract requires >1yN deposit on `nft_approve`, contract
//   MUST refund associated storage deposit when owner revokes approval
// * Contract MUST panic if called by someone other than token owner
//
// Arguments:
// * `token_id`: the token for which to revoke an approval
// * `account_id`: the account to remove from `approvals`
function nft_revoke(
  token_id: string,
  account_id: string
) {}

// Revoke all approved accounts for a specific token.
//
// Requirements
// * Caller of the method must attach a deposit of 1 yoctoⓃ for security
//   purposes
// * If contract requires >1yN deposit on `nft_approve`, contract
//   MUST refund all associated storage deposit when owner revokes approvals
// * Contract MUST panic if called by someone other than token owner
//
// Arguments:
// * `token_id`: the token with approvals to revoke
function nft_revoke_all(token_id: string) {}

/****************/
/* VIEW METHODS */
/****************/

// Check if a token is approved for transfer by a given account, optionally
// checking an approval_id
//
// Arguments:
// * `token_id`: the token for which to revoke an approval
// * `approved_account_id`: the account to check the existence of in `approvals`
// * `approval_id`: an optional approval ID to check against current approval ID for given account
//
// Returns:
// if `approval_id` given, `true` if `approved_account_id` is approved with given `approval_id`
// otherwise, `true` if `approved_account_id` is in list of approved accounts
function nft_is_approved(
  token_id: string,
  approved_account_id: string,
  approval_id: number|null
): boolean {}
```

### Why must `nft_approve` panic if `nft_revoke_all` would fail later?

In the description of `nft_approve` above, it states:

    Contract MUST panic if addition would cause `nft_revoke_all` to exceed
    single-block gas limit.

What does this mean?

First, it's useful to understand what we mean by "single-block gas limit". This refers to the [hard cap on gas per block at the protocol layer](https://docs.near.org/docs/concepts/gas#thinking-in-gas). This number will increase over time.

Removing data from a contract uses gas, so if an NFT had a large enough number of approvals, `nft_revoke_all` would fail, because calling it would exceed the maximum gas.

Contracts must prevent this by capping the number of approvals for a given token. However, it is up to contract authors to determine a sensible cap for their contract (and the single block gas limit at the time they deploy). Since contract implementations can vary, some implementations will be able to support a larger number of approvals than others, even with the same maximum gas per block.

Contract authors may choose to set a cap of something small and safe like 10 approvals, or they could dynamically calculate whether a new approval would break future calls to `nft_revoke_all`. But every contract MUST ensure that they never break the functionality of `nft_revoke_all`.


### Approved Account Contract Interface

If a contract that gets approved to transfer NFTs wants to, it can implement `nft_on_approve` to update its own state when granted approval for a token:

```ts
// Respond to notification that contract has been granted approval for a token.
//
// Notes
// * Contract knows the token contract ID from `predecessor_account_id`
//
// Arguments:
// * `token_id`: the token to which this contract has been granted approval
// * `owner_id`: the owner of the token
// * `approval_id`: the approval ID stored by NFT contract for this approval.
//   Expected to be a number within the 2^53 limit representable by JSON.
// * `msg`: specifies information needed by the approved contract in order to
//    handle the approval. Can indicate both a function to call and the
//    parameters to pass to that function.
function nft_on_approve(
  token_id: TokenId,
  owner_id: string,
  approval_id: number,
  msg: string,
) {}
```

Note that the NFT contract will fire-and-forget this call, ignoring any return values or errors generated. This means that even if the approved account does not have a contract or does not implement `nft_on_approve`, the approval will still work correctly from the point of view of the NFT contract.

Further note that there is no parallel `nft_on_revoke` when revoking either a single approval or when revoking all. This is partially because scheduling many `nft_on_revoke` calls when revoking all approvals could incur prohibitive [gas fees](https://docs.near.org/docs/concepts/gas). Apps and contracts which cache NFT approvals can therefore not rely on having up-to-date information, and should periodically refresh their caches. Since this will be the necessary reality for dealing with `nft_revoke_all`, there is no reason to complicate `nft_revoke` with an `nft_on_revoke` call.

### No incurred cost for core NFT behavior

NFT contracts should be implemented in a way to avoid extra gas fees for serialization & deserialization of `approvals` for calls to `nft_*` methods other than `nft_token`. See `near-contract-standards` [implementation of `ft_metadata` using `LazyOption`](https://github.com/near/near-sdk-rs/blob/c2771af7fdfe01a4e8414046752ee16fb0d29d39/examples/fungible-token/ft/src/lib.rs#L71) as a reference example.
