# Non-Fungible Token Royalties ([NEP-182](https://github.com/near/NEPs/discussions/182))

Version: `1.0.0`

A standard for Non-Fungible Tokens to implement Royalties. Marketplace apps and other sellers of NFTs must split the proceeds of each sale of an NFT amongst the accounts specified by this standard.

## Motivation

As the NEAR ecosystem grows to encompass many non-fungible token (NFT) marketplaces, a standard system to handle royalty payments for non-fungible tokens allows for an NFT created, purchased, or sold on one marketplace to provide the royalties that the creator is entitled to, regardless of the next marketplace/ecosystem it is sold in.

Having this standard on NEAR as a part of the NFT standard  allows for the ecosystem to avoid the pitfalls of marketplace divergence leading to a lack of interoperability.

Without an agreed standard for implementing royalties, the NFT ecosystem will lack an effective means to collect royalties across all marketplaces. This will hamper the growth and adoption of NFTs and demotivate artists and other NFT creators from minting new and innovative tokens.


## Interface

```ts
// A single 4-digit number, which may be safely multiplied with another
// SafeFraction without resorting to storing a denominator field. Implicit
// denominator is 10,000.
type SafeFraction = {
  numerator: number; // must be four-digits or less
}

type Royalty = {
  // Mapping of addresses to relative percentages of the overall royalty percentage
  split_between: { [key: string]: SafeFraction },
  // The overall royalty percentage taken
  percentage: SafeFraction,
}
```
A new attribute must be added to each `Token` struct:

```diff
 type Token = {
   owner_id: string,
+  royalty: Royalty,
 }
```


## Rust Implementation Guidance

Taking some ideas from Mintbase and the [Staking Pool contract](https://github.com/near/core-contracts/blob/master/staking-pool) regarding the safe representation of a fraction:

```rust
#[derive(BorshSerialize, BorshDeserialize, Serialize, Deserialize, Debug, Clone, PartialEq)]
#[serde(crate = "near_sdk::serde")]
pub struct Royalty {
  /// Mapping of addresses to relative percentages of the overall royalty percentage
  pub split_between: HashMap<AccountId, SafeFraction>,
  /// The overall royalty percentage taken
  pub percentage: SafeFraction,
}

/// Royalty upper limit is 50%.
pub const ROYALTY_UPPER_LIMIT: u32 = 5000;

impl Royalty {
  /// Validates all arguments. Addresses must be valid and percentages must be
  /// within accepted values. Hashmap percentages must add to 10000.
  pub fn new(split_between: HashMap<AccountId, u32>, percentage: u32) -> Self {
    assert!(
      percentage <= ROYALTY_UPPER_LIMIT,
      "percentage: {} must be <= 5000",
      percentage
    );
    assert!(percentage > 0, "percentage cannot be zero");
    assert!(
      split_between.len() <= MAX_HASHMAP_LENGTH,
      "royalty mapping too long"
    );
    assert!(!split_between.is_empty(), "royalty mapping is empty");

    let mut sum: u32 = 0;
    let split_between: HashMap<AccountId, SafeFraction> = split_between
      .into_iter()
      .map(|(addr, numerator)| {
        assert!(env::is_valid_account_id(&addr.as_bytes().to_vec()));
        assert!(numerator > 0, "percentage cannot be zero");

        let sf = SafeFraction::new(numerator);
        sum += sf.numerator;

        (addr, sf)
      })
      .collect();
    assert!(sum == 10_000, "fractions don't add to 10,000");

    Self {
      percentage: SafeFraction::new(percentage),
      split_between,
    }
  }
}
```

Where `SafeFraction` is a single 4-digit number, which may be safely multiplied with another `SafeFraction` without resorting to storing a denominator field:

```rust
#[derive(
  BorshDeserialize, BorshSerialize, Serialize, Deserialize, Debug, PartialEq, Copy, Clone,
)]
#[serde(crate = "near_sdk::serde")]
pub struct SafeFraction {
  pub numerator: u32,
}

impl SafeFraction {
  /// Take a u32 numerator to a 10^4 denominator.
  ///
  /// Upper limit is 10^4 so as to prevent multiplication with overflow.
  pub fn new(numerator: u32) -> Self {
    assert!(
      (0..=10000).contains(&numerator),
      "{} not between 0 and 10,000",
      numerator
    );
    SafeFraction { numerator }
  }

  /// Fractionalize a balance.
  pub fn multiply_balance(self, value: Balance) -> Balance {
    self.numerator as u128 * value / 10_000
  }
}

impl std::ops::Mul for SafeFraction {
  type Output = MultipliedSafeFraction;
  fn mul(self, rhs: Self) -> Self::Output {
    MultipliedSafeFraction {
      numerator: self.numerator * rhs.numerator,
    }
  }
}

/// A SafeFraction that has been multiplied with another SafeFraction. Denominator is 10^8.
pub struct MultipliedSafeFraction {
  pub numerator: u32,
}
```
