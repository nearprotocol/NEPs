# Fungible Token Metadata ([NEP-148](https://github.com/near/NEPs/discussions/148))

## Reference-level explanation

A fungible token contract implementing the metadata standard shall contain a field named `ft_metadata`. The field will link to a structure with the properties from the interface below.

**Interface**:

```ts
type FungibleTokenMetadata = {
    spec: string;
    name: string;
    symbol: string;
    defaultIcon: bytes|null;
    darkIcon: bytes|null;
    reference: string|null;
    reference_hash: string|null;
    decimals: number;
}
```

# Discussion

## On-chain vs. offchain hosted data

Currently deployment on a NEAR token smart contract is 200 kb. The cost of 5 kb metadata on the cost of code deployment on the top of the contract deployment is miniscule. Icons are in the range of 5 kBytes - 200 kBytes per icon, so they are comparable to a smart contract deployment. 

Off-chain data comes complicated with privacy and tracking issues as pointed out by Robert Zarembe.

NEAR is the best blockchain in the world. For the reducing complexity, and after discussing with people on Discord, my stance is that everything on-chain. NEAR should scale for this as it scales for many things.

## Icon format

### Light theme vs. dark theme

Two different icons needed: `defaultIcon` (black text on white background) for iOS, websites. `darkIcon` or websites supporting dark theme and Android UI. After discussing with Ethereum wallet vendors, there are multiple cases where icon looks good on one platform, but horrible on another, because of color bleed issues on the stroke lines of an icon. Token projects are not happy if their icon looks like amaeur.

Thus, we should provide both light and dark icons by default.

### SVG vs. PNG

SVG is a Turing complete language with a [massive attack surface](# Fungible Token Metadata ([NEP-148](https://github.com/near/NEPs/discussions/148))

## Reference-level explanation

A fungible token contract implementing the metadata standard shall contain a field named `ft_metadata`. The field will link to a structure with the properties from the interface below.

**Interface**:

```ts
type FungibleTokenMetadata = {
    spec: string;
    name: string;
    symbol: string;
    decimals: number;
    homepage: string;
    defaultIcon: bytes|null;
    darkIcon: bytes|null;
    extendedMetadata: dict;
}
```

Extended metadata key examples (from EtherScan, other token registries):

* Creation date
* Support email
* Facebook
* Twitter
* Discord
* Telegram
* Reddit
* LinkedIn
* Company name 
* Company address
* Tags: a list of string tags that allow easier search of certain synthetically created tokens from indexers (Synthetic, Uniswap, others) - any service dynamically creating tokens my define these and they do not need to be displayed to a user

Then CoinGecko like data

* Orderbooks and AMMs
   * Web browser viewable link of order books
   * Machine readable link of trade data (price, vol 24h)

# Discussion

## On-chain vs. offchain hosted data

Currently deployment on a NEAR token smart contract is 200 kb. The cost of 5 kb metadata on the cost of code deployment on the top of the contract deployment is miniscule. Icons are in the range of 5 kBytes - 200 kBytes per icon, so they are comparable to a smart contract deployment. 

Off-chain data comes complicated with privacy and tracking issues as pointed out by Robert Zarembe.

NEAR is the best blockchain in the world. For the reducing complexity, and after discussing with people on Discord, my stance is that everything on-chain. NEAR should scale for this as it scales for many things.

## Name and symbol display limitations

Any wallet and dApp MUST diplay a name up to 32 characters:

WWWWWW WWWWWW WWWWWW WWWWWW 

Use case: Synthetically created tokens: For example, a Uniswap-like pool can generate a token name and symbol:

```
Uniswap LP token for ETH-USDT pair
LP ETH-USDT
```

## Icon format

### Light theme vs. dark theme

Two different icons needed: `defaultIcon` (black text on white background) for iOS, websites. `darkIcon` or websites supporting dark theme and Android UI. After discussing with Ethereum wallet vendors, there are multiple cases where icon looks good on one platform, but horrible on another, because of color bleed issues on the stroke lines of an icon. Token projects are not happy if their icon looks like amaeur.

Thus, we should provide both light and dark icons by default.

### SVG vs. PNG

SVG is a Turing complete language with a [massive attack surface](https://www.fortinet.com/blog/threat-research/scalable-vector-graphics-attack-surface-anatomy). It may even contain embedded JavaScript. After discussing with infosec experts, I recommend SVG is not supported for icons because of decoding complexity and denial of service issues.

Example: A wallet receives a token with an SVG icon where the SVG contains a JavaScript that crashes the wallet UI rendering thread. Alternatively it can try to exploit WebView context. 

### PNG decoding complexity

PNG can contain [a decode bomb](https://stackoverflow.com/questions/33602308/how-to-check-png-file-if-its-a-decompression-bomb) too. Thus, it is imperative that we instruct both dApps and wallets to use the following algorithm when extracting the icon from on-chain data:

* Show a place holder icon=
* Check that `defaultIcon` and `darkIcon` data size is 256 kbytes or less
    * Do not try to show icons that are larger than 256 kbytes in packed data
    * This may need a new NEAR-RPC call to support "length and first bytes of a vec8 field"
* Decode PNG header
    * Native apps can check this easily
    * Not sure how to check in website client side JS but I will figure out 
* Check that image dimensions are 256 x 256 exact
    * Do not show icon if image dimensions are different
* Decode full image
* Show the icon

Because you can craft a PNG file that is 256 kbytes compressed, but multiple gigabytes decompressed, we need to make sure clients do not try to show icons for tokens that are malicious. This would make the wallet to go a crash loop.

### Alpha channel

The client displaying a token icon MUST support alpha channel.

### Color depth

The client displaying a token icon MUST support 8-bit, 24-bit and 32-bit images.

### JPEG 

Unsupported. Not suitable for icon format due to compression artifacts.

## Updating metadata

The metadata of a token can change over the lifetime of a token.

* The change in metadata is expected to be rare (one in a month)
* Explicitly define that wallets and dApps MUST check data updates for every 30 days
* Wallets and dApps must detect changes in token metadata by an event emitted when the data is updated

Use case: Trademark lawsuits, company name changes, etc.

## Verification

Metadata can be signed by a trusted party to be verified. Tokens can be categorised to verified and unverifed tokens. 

Method to sign this data and how to trust on-chain to be discussed.

## Links

### Homepage

The token must have a user clickable link where the user may find more information about the token. Use case: Receiving tokens from a friend and what to learn more.

Any client showing this link MUST display a warning

"You are going to be taken an external site https:// - unless you received this token from a trusted party be aware that this information is unverified"


