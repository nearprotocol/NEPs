# VRF

# Ristretto
VRF scheme requires a prime order group for which it uses `ristretto255` implemented
in `curve25519-dalek`.

Reference: https://ristretto.group/ristretto.html
```
/// A `RistrettoPoint` represents a point in the Ristretto group for
/// Curve25519.  Ristretto, a variant of Decaf, constructs a
/// prime-order group as a quotient group of a subgroup of (the
/// Edwards form of) Curve25519.
///
/// Internally, a `RistrettoPoint` is implemented as a wrapper type
/// around `EdwardsPoint`, with custom equality, compression, and
/// decompression routines to account for the quotient.  This means that
/// operations on `RistrettoPoint`s are exactly as fast as operations on
/// `EdwardsPoint`s.
```

## Key conversion
`vrf::PublicKey` is a `RistrettoPoint`, `vrf::SecretKey` is a secret scalar corresponding to it.

`Ristretto` group is constructed as `[2]E/E[4] (even points on E, mod 4-torsion points)`, and
internally `RistrettoPoint` is an `EdwardsPoint` in the even subgroup with custom
equality to account for the quotient.

All proper `ED25519PublicKey` are in the even subgroup and can be handled by `RistrettoPoint`.

`key_conversion.rs` provides methods to convert from `ED25519PublicKey/ED25519SecretKey`
to `vrf::PublicKey/vrf::SecretKey`.

## VRF scheme
`(sk, pk)` is the keypair, `input` is the VRF input.


```
compute_vrf_with_proof(sk, pk, input) -> (Value(val), Proof(r, c))
    x = sk + hash(pk, input)
    val = G * inv(x)
    c = hash(pk,
             G * inv(x),
             G * hash(x),
             G * inv(x) * hash(x))
    r = hash(x) - c * x
    (val, r, c)
```

```
is_vrf_valid(pk, input, Value(val), Proof(r, c)) -> bool
    c == hash(pk,
              val,
              G * (r + c * hash(pk, input)) + pk * c,
              val * r + G * c)
```
