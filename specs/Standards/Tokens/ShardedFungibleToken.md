# Sharded Fungible Token

## Summary
[summary]: #summary

A standard interface for sharded fungible tokens allowing for ownership, escrow and transfer, specifically targeting third-party marketplace integration.

## Motivation
[motivation]: #motivation

NEAR Protocol uses an asynchronous sharded Runtime. This means the following:
 - Storage for different contracts and accounts can be located on the different shards.
 - Two contracts can be executed at the same time in different shards.

While this increases the transaction throughput linearly with the number of shards, it is obvious that single smart contract transaction throughput is strictly limited.

Prior art:
- [ERC-20 standard](https://eips.ethereum.org/EIPS/eip-20)
- [Near Fungible Token Standard](https://github.com/nearprotocol/NEPs/blob/master/specs/Standards/Tokens/FungibleToken.md)

## Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

Smart contract logic should be sharded, to have separate smart contract for each user and allow smart contract to interact in peer-to-peer manner.

Supported functions:

- Simple token transfers
- Querying tokens from sender


## Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

The full implementation in Solidity can be found there: https://github.com/k06a/ShardedToken

Interface:

```Solidity
interface IShardedTokenExt {
  	function extensionOf(address user) external view returns(address);
}

interface IShardedTokenExt {
  	function balance() external view returns(uint256);
	  function allowance(address to) external view returns(uint256);
	  function extensionOf(address user) external view returns(address);

  	function transfer(address to, uint256 amount) external returns(bool);
  	function approve(address to, uint256 amount) external returns(bool);
  	function transferFrom(address from, uint256 amount) external returns(bool);

	  function received(address from, uint256 amount) external returns(bool);
}
```

## Drawbacks
[drawbacks]: #drawbacks

- Current interface doesn't have minting, precision (decimals), naming. But it should be done as extensions.


## Future possibilities
[future-possibilities]: #future-possibilities

- Support for multiple token types
- Minting and burning
- Precision, naming and short token name.