# Non-Fungible Token ([NEP-171](https://github.com/near/NEPs/discussions/171))

Version `1.0.0`

## Summary

A standard interface for non-fungible tokens (NFTs). That is, tokens which each have a unique ID.

## Motivation

In the three years since [ERC-721] was ratified by the Ethereum community, Non-Fungible Tokens have proven themselves as an incredible new opportunity across a wide array of disciplines: collectibles, art, gaming, finance, virtual reality, real estate, and more.

This standard builds off the lessons learned in this early experimentation, and pushes the possibilities further by harnessing unique attributes of the NEAR blockchain:

- an asynchronous, sharded runtime, meaning that two contracts can be executed at the same time in different shards
- a [storage staking] model that separates [gas] fees from the storage demands of the network, enabling greater on-chain storage (see [Metadata] extension) and ultra-low transaction fees

Given these attributes, this NFT standard can accomplish with one user interaction things for which other blockchains need two or three. Most noteworthy is `nft_transfer_call`, by which a user can essentially attach a token to a call to a separate contract. An example scenario:

* An [Exquisite Corpse](https://en.wikipedia.org/wiki/Exquisite_corpse) contract allows three drawings to be submitted, one for each section of a final composition, to be minted as its own NFT and sold on a marketplace, splitting royalties amongst the original artists.
* Alice draws the top third and submits it, Bob the middle third, and Carol follows up with the bottom third. Since they each use `nft_transfer_call` to both transfer their NFT to the Exquisite Corpse contract as well as call a `submit` method on it, the call from Carol can automatically kick off minting a composite NFT from the three submissions, as well as listing this composite NFT in a marketplace.
* When Dan attempts to also call `nft_transfer_call` to submit an unneeded top third of the drawing, the Exquisite Corpse contract can throw an error, and the transfer will be rolled back so that Bob maintains ownership of his NFT.

While this is already flexible and powerful enough to handle all sorts of existing and new use-cases, apps such as marketplaces may still benefit from the [Approval Management] extension.

Prior art:

- [ERC-721]
- [EIP-1155 for multi-tokens](https://eips.ethereum.org/EIPS/eip-1155#non-fungible-tokens)
- [NEAR's Fungible Token Standard](../FungibleToken/Core.md), which first pioneered the "transfer and call" technique

## Reference-level explanation

**NOTES**:
- All amounts, balances and allowance are limited by `U128` (max value `2**128 - 1`).
- Token standard uses JSON for serialization of arguments and results.
- Amounts in arguments and results have are serialized as Base-10 strings, e.g. `"100"`. This is done to avoid JSON limitation of max integer value of `2**53`.
- The contract must track the change in storage when adding to and removing from collections. This is not included in this core fungible token standard but instead in the [Storage Standard](../Storage.md).
- To prevent the deployed contract from being modified or deleted, it should not have any access keys on its account.

### NFT Interface

```ts
// The base structure that will be returned for a token. If contract is using
// extensions such as Approval Management, Metadata, or other
// attributes may be included in this structure.
type Token = {
  id: string,
  owner_id: string,
}

/******************/
/* CHANGE METHODS */
/******************/

// Simple transfer. Transfer a given `token_id` from current owner to
// `receiver_id`.
//
// Requirements
// * Caller of the method MUST attach a deposit of 1 yoctoⓃ for security purposes
// * Contract MUST panic if called by someone other than token owner or,
//   if using Approval Management, one of the approved accounts
// * `approval_id` is for use with Approval Management extension, see
//   that document for full explanation.
// * If using Approval Management, contract MUST nullify approved accounts on
//   successful transfer.
//
// Arguments:
// * `receiver_id`: the valid NEAR account receiving the token.
// * `token_id`: the token to transfer.
// * `approval_id`: expected approval ID. A number smaller than
//    2^53, and therefore representable as JSON. See Approval Management
//    standard for full explanation.
// * `memo` (optional): for use cases that may benefit from indexing or
//    providing information for a transfer.
function nft_transfer(
  receiver_id: string,
  token_id: string,
  approval_id: number|null,
  memo: string|null,
) {}

// Returns `true` if the token was transferred from the sender's account.
//
// Transfer token and call a method on a receiver contract. A successful
// workflow will end in a success execution outcome to the callback on the NFT
// contract at the method `nft_resolve_transfer`.
//
// You can think of this as being similar to attaching native NEAR tokens to a
// function call. It allows you to attach any Non-Fungible Token in a call to a
// receiver contract.
//
// Requirements:
// * Caller of the method MUST attach a deposit of 1 yoctoⓃ for security
//   purposes.
// * Contract MUST panic if called by someone other than token owner or,
//   if using Approval Management, one of the approved accounts.
// * The receiving contract must implement `nft_on_transfer` according to the
//   standard. If it does not, this NFT contract's `nft_resolve_transfer` MUST deal
//   with the resulting failed cross-contract call and roll back the transfer.
// * Contract MUST implement the behavior described in `nft_resolve_transfer`
// * `approval_id` is for use with Approval Management extension, see
//   that document for full explanation.
// * If using Approval Management, contract MUST nullify approved accounts on
//   successful transfer.
//
// Arguments:
// * `receiver_id`: the valid NEAR account receiving the token.
// * `token_id`: the token to send.
// * `approval_id`: expected approval ID. A number smaller than
//    2^53, and therefore representable as JSON. See Approval Management
//    standard for full explanation.
// * `memo` (optional): for use cases that may benefit from indexing or
//    providing information for a transfer.
// * `msg`: specifies information needed by the receiving contract in
//    order to properly handle the transfer. Can indicate both a function to
//    call and the parameters to pass to that function.
function nft_transfer_call(
  receiver_id: string,
  token_id: string,
  approval_id: string|null,
  memo: string|null,
  msg: string,
): Promise {}


/****************/
/* VIEW METHODS */
/****************/

// Returns the token with the given `token_id` or `null` if no such token.
function nft_token(token_id: string): Token|null {}
```

The following behavior is required, but contract authors may name this function something other than the conventional `nft_resolve_transfer` used here.

```ts
// Finalize an `nft_transfer_call` chain of cross-contract calls.
//
// The `nft_transfer_call` process:
//
// 1. Sender calls `nft_transfer_call` on NFT contract.
// 2. NFT contract transfers token from sender to receiver.
// 3. NFT contract calls `nft_on_transfer` on receiver contract.
// 4+. [receiver contract may make other cross-contract calls]
// N. NFT contract resolves promise chain with `nft_resolve_transfer`, and may
//    transfer token back to sender.
//
// Requirements:
// * Contract MUST forbid calls to this function by any account except self.
// * If promise chain failed, contract MUST revert token transfer.
// * If promise chain resolves with `true`, contract MUST return token to
//   `sender_id`.
//
// Arguments:
// * `owner_id`: the owner of `token_id`.
// * `receiver_id`: the `receiver_id` argument given to `nft_transfer_call`.
// * `token_id`: the `token_id` argument given to `nft_transfer_call`.
// * `approvals`: if using Approval Management, contract MUST provide.
//   set of original `approvals` in this argument, and restore these
//   approved accounts in case of revert.
//
// Returns true if token was successfully transferred to `receiver_id`.
function nft_resolve_transfer(
  owner_id: string,
  receiver_id: string,
  token_id: string,
  approvals: null|Record<string, number>,
): boolean {}
```

### Receiver Interface

Contracts which want to make use of `nft_transfer_call` must implement the following:

```ts
// Take some action after receiving a non-fungible token.
//
// Requirements:
// * Contract MUST restrict calls to this function to a set of whitelisted NFT
//   contracts.
//
// Arguments:
// * `sender_id`: the sender of `nft_transfer_call`.
// * `previous_owner_id`: the account that owned the NFT prior to it being
//   transfered to this contract, which can differ from `sender_id` if using
//   Approval Management extension.
// * `token_id`: the `token_id` argument given to `nft_transfer_call`
// * `msg`: information necessary for this contract to know how to process the
//   request. This may include method names and/or arguments.
//
// Returns true if token should be returned to `sender_id`.
function nft_on_transfer(
  sender_id: string,
  previous_owner_id: string,
  token_id: string,
  msg: string,
): Promise<boolean>;
```

  [ERC-721]: https://eips.ethereum.org/EIPS/eip-721
  [storage staking]: https://docs.near.org/docs/concepts/storage-staking
  [gas]: https://docs.near.org/docs/concepts/gas
  [Metadata]: Metadata.md
  [Approval Management]: ApprovalManagement.md

