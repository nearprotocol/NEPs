# Fungible Token Metadata ([NEP-148](https://github.com/near/NEPs/discussions/148))

Version `1.0.0`

## Summary
[summary]: #summary

An interface for a fungible token's metadata. The goal is to keep the metadata future-proof as well as lightweight. This will be important to dApps needing additional information about an FT's properties, and broadly compatible with other tokens standards such that the [NEAR Rainbow Bridge](https://near.org/blog/eth-near-rainbow-bridge/) can move tokens between chains.

## Motivation

Custom fungible tokens play a major role in decentralized applications today. FTs can contain custom properties to differentiate themselves from other tokens or contracts in the ecosystem. In NEAR, many common properties can be stored right on-chain. Other properties are best stored off-chain or in a decentralized storage platform, in order to save on storage costs and allow rapid community experimentation.

As blockchain technology advances, it becomes increasingly important to provide backwards compatibility and a concept of a spec. This standard encompasses all of these concerns.

Prior art:

- [EIP-1046](https://eips.ethereum.org/EIPS/eip-1046)
- [OpenZeppelin's ERC-721 Metadata standard](https://docs.openzeppelin.com/contracts/2.x/api/token/erc721#ERC721Metadata) also helped, although it's for non-fungible tokens.

## Guide-level explanation

A fungible token smart contract allows for discoverable properties. Some properties can be determined by other contracts on-chain, or return in view method calls. Others can only be determined by an oracle system to be used on-chain, or by a frontend with the ability to access a linked reference file.

### Examples scenario

#### Token provides metadata upon deploy and initialization

Alice deploys a wBTC fungible token contract.

**Assumptions**

- The wBTC token contract is `wbtc`.
- Alice's account is `alice`.
- The precision ("decimals" in this metadata standard) on wBTC contract is `10^8`.

**High-level explanation**

Alice issues a transaction to deploy and initialize the fungible token contract, providing arguments to the initialization function that set metadata fields.

**Technical calls**

1. `alice` deploys a contract and calls `wbtc::new` with all metadata. If this deploy and initialization were done using [NEAR CLI](https://docs.near.org/docs/tools/near-cli) the command would be:

    near deploy wbtc --wasmFile res/ft.wasm --initFunction new --initArgs '{
      "owner_id": "wbtc",
      "total_supply": "100000000000000",
      "metadata": {
         "spec": "ft-1.0.0",
         "name": "Wrapped Bitcoin",
         "symbol": "WBTC",
         "icon": "data:image/svg+xml,%3C…",
         "reference": "https://example.com/wbtc.json",
         "reference_hash": "AK3YRHqKhCJNmKfV6SrutnlWW/icN5J8NUPtKsNXR1M=",
         "decimals": 8
      }
    }' --accountId alice

## Reference-level explanation

A fungible token contract implementing the metadata standard shall contain a field named `ft_metadata`. The field will link to a structure with the properties from the interface below.

**Interface**:

```ts
type FungibleTokenMetadata = {
    spec: string;
    name: string;
    symbol: string;
    icon: string|null;
    reference: string|null;
    reference_hash: string|null;
    decimals: number;
}

/**********************************/
/* VIEW METHODS on fungible token */
/**********************************/

// Returns metadata for a token
ft_metadata(): FungibleTokenMetadata
```

**An implementing contract MUST include the following fields on-chain**

- `spec`: a string. Should be `ft-1.0.0` to indicate that a Fungible Token contract adheres to the current versions of this Metadata and the [Fungible Token Core](./FungibleTokenCore.md) specs. This will allow consumers of the Fungible Token to know if they support the features of a given contract.
- `name`: the human-readable name of the token.
- `symbol`: the abbreviation, like wETH or AMPL.
- `decimals`: used in frontends to show the proper significant digits of a token. This concept is explained well in this [OpenZeppelin post](https://docs.openzeppelin.com/contracts/3.x/erc20#a-note-on-decimals).

**An implementing contract MAY include the following fields on-chain**

- `icon`: a small image associated with this token. Must be a [data URL](https://developer.mozilla.org/en-US/docs/Web/HTTP/Basics_of_HTTP/Data_URIs), to help consumers display it quickly while protecting user data. Recommendation: use [optimized SVG](https://codepen.io/tigt/post/optimizing-svgs-in-data-uris), which can result in high-resolution images with only 100s of bytes of [storage cost](https://docs.near.org/docs/concepts/storage-staking). (Note that these storage costs are incurred to the token owner/deployer, but that querying these icons is a very cheap & cacheable read operation for all consumers of the contract and the RPC nodes that serve the data.) Recommendation: create icons that will work well with both light-mode and dark-mode websites by either using middle-tone color schemes, or by [embedding `media` queries in the SVG](https://timkadlec.com/2013/04/media-queries-within-svg/).
- `reference`: a link to a valid JSON file containing various keys offering supplementary details on the token. Example: "/ipfs/QmdmQXB2mzChmMeKY47C43LxUdg1NDJ5MWcKMKxDu7RgQm", "https://example.com/token.json", etc. If the information given in this document conflicts with the on-chain attributes, the values in `reference` shall be considered the source of truth.
- `reference_hash`: the base64-encoded sha256 hash of the JSON file contained in the `reference` field. This is to guard against off-chain tampering.


**No incurred cost for core FT behavior**

Contracts should be implemented in a way to avoid extra gas fees for serialization & deserialization of metadata for calls to `ft_*` methods other than `ft_metadata`. See `near-contract-standards` [implementation using `LazyOption`](https://github.com/near/near-sdk-rs/blob/c2771af7fdfe01a4e8414046752ee16fb0d29d39/examples/fungible-token/ft/src/lib.rs#L71) as a reference example.


## Drawbacks

- It could be argued that `symbol` and even `name` could belong as key/values in the `reference` JSON object.
- Enforcement of `icon` to be a data URL rather than a link to an HTTP endpoint that could contain privacy-violating code cannot be done on deploy or update of contract metadata, and must be done on the consumer/app side when displaying token data.
- If on-chain icon uses a data URL or is not set but the document given by `reference` contains a privacy-violating `icon` URL, consumers & apps of this data should not naïvely display the `reference` version, but should prefer the safe version. This is technically a violation of the "`reference` setting wins" policy described above.

## Future possibilities

- Detailed conventions that may be enforced for versions.
- A fleshed out schema for what the `reference` object should contain.
