

# signature.rs

PublicKey/SecretKey types are used to represent keypairs for Ed25519 or Secp256k1.

```rust
pub enum PublicKey {
    ED25519(ED25519PublicKey),
    SECP256K1(Secp256K1PublicKey),
}
```
```rust
pub enum SecretKey {
    ED25519(ED25519SecretKey),
    SECP256K1(secp256k1::key::SecretKey),
}
```
```rust
pub enum Signature {
    ED25519(ed25519_dalek::Signature),
    SECP256K1(Secp256K1Signature),
}
```

The inner types `ED25519PublicKey`, `ED25519SecretKey`, `Secp256K1PublicKey`, `secp256k1::key::SecretKey`
are byte arrays of 32 or 64 bytes which are interpreted as described in sections [Ed25519](Ed25519.md)/[Secp256k1](Secp256k1.md).

## JSON Serialization

In string/JSON format keys/signatures are written as the type (`ed25519`/`secp256k1`) followed by a
semicolon and the underlying data, which is represented as a byte array (as defined by the corresponding
standard) encoded as `base58`.

Example:
```rust
    let sk = SecretKey::from_seed(KeyType::ED25519, "test");
    let pk = sk.public_key();
    let signature = sk.sign(b"123");
    let expected_pk = "\"ed25519:DcA2MzgpJbrUATQLLceocVckhhAqrkingax4oJ9kZ847\"";
    let expected_sk = "\"ed25519:3KyUuch8pYP47krBq4DosFEVBMR5wDTMQ8AThzM8kAEcBQEpsPdYTZ2FPX5ZnSoLrerjwg66hwwJaW1wHzprd5k3\"";
    let expected_sg = "\"ed25519:3s1dvZdQtcAjBksMHFrysqvF63wnyMHPA4owNQmCJZ2EBakZEKdtMsLqrHdKWQjJbSRN6kRknN2WdwSBLWGCokXj\"";
    assert_eq!(serde_json::to_string(&pk).unwrap(), expected_pk);
    assert_eq!(serde_json::to_string(&sk).unwrap(), expected_sk);
    assert_eq!(serde_json::to_string(&signature).unwrap(), expected_sg);
```

```rust
    let sk = SecretKey::from_seed(KeyType::SECP256K1, "test");
    let pk = sk.public_key();
    let data = sha2::Sha256::digest(b"123").to_vec();
    let signature = sk.sign(&data);
    let expected_pk = "\"secp256k1:BtJtBjukUQbcipnS78adSwUKE38sdHnk7pTNZH7miGXfodzUunaAcvY43y37nm7AKbcTQycvdgUzFNWsd7dgPZZ\"";
    let expected_sk = "\"secp256k1:9ZNzLxNff6ohoFFGkbfMBAFpZgD7EPoWeiuTpPAeeMRV\"";
    let expected_sg = "\"secp256k1:7iA75xRmHw17MbUkSpHxBHFVTuJW6jngzbuJPJutwb3EAwVw21wrjpMHU7fFTAqH7D3YEma8utCdvdtsqcAWqnC7r\"";
    assert_eq!(serde_json::to_string(&pk).unwrap(), expected_pk);
    assert_eq!(serde_json::to_string(&sk).unwrap(), expected_sk);
    assert_eq!(serde_json::to_string(&signature).unwrap(), expected_sg);
```

If the key type is not specified, deserialization defaults to `Ed25519`:

```rust
    let expected_pk = "\"ed25519:DcA2MzgpJbrUATQLLceocVckhhAqrkingax4oJ9kZ847\"";
    let expected_noprefix = "\"DcA2MzgpJbrUATQLLceocVckhhAqrkingax4oJ9kZ847\"";
    let pk: PublicKey = serde_json::from_str(expected_noprefix).unwrap();
    assert_eq!(serde_json::to_string(&pk).unwrap(), expected_pk);
```

## Borsh serialization
Borsh format starts with a byte corresponding to key type (`0` for `Ed25519`, `1` for `Secp256k1`) and
is followed by the underlying data serialized as Borsh fixed size array.
