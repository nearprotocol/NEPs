# Non-Fungible Token Metadata ([NEP-177](https://github.com/near/NEPs/discussions/177))

Version `1.0.0`

## Summary

An interface for a non-fungible token's metadata. The goal is to keep the metadata future-proof as well as lightweight. This will be important to dApps needing additional information about an NFT's properties, and broadly compatible with other token standards such that the [NEAR Rainbow Bridge](https://near.org/blog/eth-near-rainbow-bridge/) can move tokens between chains.

## Motivation

The primary value of non-fungible tokens comes from their metadata. While the [core standard](Core.md) provides the minimum interface that can be considered a non-fungible token, most artists, developers, and dApps will want to associate more data with each NFT, and will want a predictable way to interact with any NFT's metadata.

NEAR's unique [storage staking](https://docs.near.org/docs/concepts/storage-staking) approach makes it feasible to store more data on-chain than other blockchains. This standard leverages this strength for common metadata attributes, and provides a standard way to link to additional offchain data to support rapid community experimentation.

This standard also provides a `spec` version. This makes it easy for consumers of NFTs, such as marketplaces, to know if they support all the features of a given token.

Prior art:

- NEAR's [Fungible Token Metadata Standard](../FungibleToken/Metadata.md)
- Discussion about NEAR's complete NFT standard: #171

## Interface

Metadata applies at both the contract level (`NFTContractMetadata`) and the token level (`TokenMetadata`). The relevant metadata for each:

```ts
type NFTContractMetadata = {
  spec: string, // required, essentially a version like "nft-1.0.0"
  name: string, // required, ex. "Mochi Rising — Digital Edition" or "Metaverse 3"
  symbol: string, // required, ex. "MOCHI"
  icon: string|null, // Data URL
  base_uri: string|null, // Centralized gateway known to have reliable access to decentralized storage assets referenced by `reference` or `media` URLs
  reference: string|null, // URL to a JSON file with more info
  reference_hash: string|null, // Base64-encoded sha256 hash of JSON from reference field. Required if `reference` is included.
}

type TokenMetadata = {
  title: string|null, // ex. "Arch Nemesis: Mail Carrier" or "Parcel #5055"
  description: string|null, // free-form description
  media: string|null, // URL to associated media, preferably to decentralized, content-addressed storage
  media_hash: string|null, // Base64-encoded sha256 hash of content referenced by the `media` field. Required if `media` is included.
  copies: number|null, // number of copies of this set of metadata in existence when token was minted.
  issued_at: string|null, // When token was issued or minted, Unix epoch in milliseconds
  expires_at: string|null, // When token expires, Unix epoch in milliseconds
  starts_at: string|null, // When token starts being valid, Unix epoch in milliseconds
  updated_at: string|null, // When token was last updated, Unix epoch in milliseconds
  extra: string|null, // anything extra the NFT wants to store on-chain. Can be stringified JSON.
  reference: string|null, // URL to an off-chain JSON file with more info.
  reference_hash: string|null // Base64-encoded sha256 hash of JSON from reference field. Required if `reference` is included.
}
```

Three new functions MUST be supported on the NFT contract:

```ts
// Get the metadata for the NFT contract.
function nft_metadata(): NFTContractMetadata {}
// Get the on-chain metadata for the token with with `token_id`.
function nft_token_metadata(token_id: string): TokenMetadata {}
// Get the URI (preferably of a decentralized storage platform) where the token metadata
// is stored.
function nft_token_uri(token_id: string): String {}
```

A new attribute MUST be added to each `Token` struct:

```diff
 type Token = {
   id: string,
   owner_id: string,
+  metadata: TokenMetadata,
 }
```

### An implementing contract MUST include the following fields on-chain

- `spec`: a string that MUST be formatted `nft-1.0.0` to indicate that a Non-Fungible Token contract adheres to the current versions of this Metadata spec. This will allow consumers of the Non-Fungible Token to know if they support the features of a given contract.
- `name`: the human-readable name of the contract.
- `symbol`: the abbreviated symbol of the contract, like MOCHI or MV3
- `base_uri`: Centralized gateway known to have reliable access to decentralized storage assets referenced by `reference` or `media` URLs. Can be used by other frontends for initial retrieval of assets, even if these frontends then replicate the data to their own decentralized nodes, which they are encouraged to do.

### An implementing contract MAY include the following fields on-chain

For `NFTContractMetadata`:

- `icon`: a small image associated with this contract. Encouraged to be a [data URL](https://developer.mozilla.org/en-US/docs/Web/HTTP/Basics_of_HTTP/Data_URIs), to help consumers display it quickly while protecting user data. Recommendation: use [optimized SVG](https://codepen.io/tigt/post/optimizing-svgs-in-data-uris), which can result in high-resolution images with only 100s of bytes of [storage cost](https://docs.near.org/docs/concepts/storage-staking). (Note that these storage costs are incurred to the contract deployer, but that querying these icons is a very cheap & cacheable read operation for all consumers of the contract and the RPC nodes that serve the data.) Recommendation: create icons that will work well with both light-mode and dark-mode websites by either using middle-tone color schemes, or by [embedding `media` queries in the SVG](https://timkadlec.com/2013/04/media-queries-within-svg/).
- `reference`: a link to a valid JSON file containing various keys offering supplementary details on the token. Example: "/ipfs/QmdmQXB2mzChmMeKY47C43LxUdg1NDJ5MWcKMKxDu7RgQm", "https://example.com/token.json", etc. If the information given in this document conflicts with the on-chain attributes, the values in `reference` shall be considered the source of truth.
- `reference_hash`: the base64-encoded sha256 hash of the JSON file contained in the `reference` field. This is to guard against off-chain tampering.

For `TokenMetadata`:

- `name`:  The name of this specific token.
- `description`: A longer description of the token.
- `media`: URL to associated media. Preferably to decentralized, content-addressed storage.
- `media_hash`: the base64-encoded sha256 hash of content referenced by the `media` field. This is to guard against off-chain tampering.
- `copies`: The number of tokens with this set of metadata or `media` known to exist at time of minting.
- `issued_at`: [ISO 8601](https://www.iso.org/iso-8601-date-and-time-format.html) datetime when token was issued or minted
- `expires_at`: [ISO 8601](https://www.iso.org/iso-8601-date-and-time-format.html) datetime when token expires
- `starts_at`: [ISO 8601](https://www.iso.org/iso-8601-date-and-time-format.html) datetime when token starts being valid
- `updated_at`: [ISO 8601](https://www.iso.org/iso-8601-date-and-time-format.html) datetime when token was last updated
- `extra`: anything extra the NFT wants to store on-chain. Can be stringified JSON.
- `reference`: URL to an off-chain JSON file with more info.
- `reference_hash`: Base64-encoded sha256 hash of JSON from reference field. Required if `reference` is included.

### No incurred cost for core NFT behavior

Contracts should be implemented in a way to avoid extra gas fees for serialization & deserialization of metadata for calls to `nft_*` methods other than `nft_metadata` or `nft_token`. See `near-contract-standards` [implementation using `LazyOption`](https://github.com/near/near-sdk-rs/blob/c2771af7fdfe01a4e8414046752ee16fb0d29d39/examples/fungible-token/ft/src/lib.rs#L71) as a reference example.

## Drawbacks

* When this NFT contract is created and initialized, the storage use per-token will be higher than an NFT Core version. Frontends can account for this by adding extra deposit when minting. This could be done by padding with a reasonable amount, or by the frontend using the [RPC call detailed here](https://docs.near.org/docs/develop/front-end/rpc#genesis-config) that gets genesis configuration and actually determine precisely how much deposit is needed.
* Convention of `icon` being a data URL rather than a link to an HTTP endpoint that could contain privacy-violating code cannot be done on deploy or update of contract metadata, and must be done on the consumer/app side when displaying token data.
* If on-chain icon uses a data URL or is not set but the document given by `reference` contains a privacy-violating `icon` URL, consumers & apps of this data should not naïvely display the `reference` version, but should prefer the safe version. This is technically a violation of the "`reference` setting wins" policy described above.

## Future possibilities

- Detailed conventions that may be enforced for versions.
- A fleshed out schema for what the `reference` object should contain.
