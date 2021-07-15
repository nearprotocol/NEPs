- Proposal Name: add-secp-validator-key
- Start Date: 2021-02-20
- NEP PR: [nearprotocol/neps#0000](https://github.com/nearprotocol/neps/pull/0000)
- Issue(s): link to relevant issues in relevant repos (not required).

# Summary
[summary]: #summary

We would like to introduce a sepc256k1 validator key in addition to the existing ed25519 validator key.
This change will allow us to verify near light client blocks fully on Ethereum, thereby eliminating the need
to have challenges in the NEAR -> Eth bridge.

# Motivation
[motivation]: #motivation

Currently validator keys are ed25519 keys and it is very expensive to verify ed25519 signatures on Ethereum.
As a result, our bridge has to be optimistic on NEAR -> Eth side and rely on challenges to make sure that the block headers submitted are valid.
This means that it suffers from the problems with optimistic approaches – the challenge period has to be sufficiently long to have practical security, which hinders usability significantly.
The introduction of secp validator keys will allow the solidity smart contract on Ethereum side to verify all signatures,
which will enable us to transfer from NEAR to Ethereum almost instantaneously.

# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

There are mainly three aspects of this proposal: changes to the protocol, changes to the NEAR -> Eth bridge, and changes to
staking pool contracts.

## Changes to the protocol

There are mainly three changes on the protocol level:
- Every approval will contain a message signed by the secp validator key that can be used for the bridge and such signatures
will be stored in the block body.
- We need to introduce a new staking action that takes both an ed25519 public key and a secp256k1 public key.
- Existing staking pool contracts need to be upgraded on the protocol level since they are permissionless and do not have
the ability to self-upgrade.

## Changes to the bridge

Instead of optimistically trust the validator signatures, the smart contract on Ethereuem side, which runs a
light client of NEAR, will fully verify the light client blocks submitted by the bridge relayers.

## Changes to the staking pool contract

A new staking pool contract needs to be written to account for the new staking action and also gives the owner
the ability to upgrade to the new pool.

# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

In this section, we will explain further the technical details of the aforementioned changes and the upgrade path.

## Changes to the protocol

`Approval` will be changed to
```rust
pub struct Approval {
    pub inner: ApprovalInner,
    pub target_height: BlockHeight,
    pub signature: Signature,
    pub bridge_signature: Signature,
    pub account_id: AccountId,
}
```

and `Block` will be changed to

```rust
pub struct BlockV3 {
    pub header: BlockHeader,
    pub chunks: Vec<ShardChunkHeader>,
    pub challenges: Challenges,
    pub bridge_signatures: Vec<Option<Signature>>,

    pub vrf_value: near_crypto::vrf::Value,
    pub vrf_proof: near_crypto::vrf::Proof,
}
```

A new staking action `StakeAction2` will be introduced:
```rust
pub struct StakeAction2 {
    pub stake: Balance,
    pub secp_public_key: PublicKey,
    pub ed25519_public_key: PublicKey,
}
```

The bridge signature here is a signature the following message:
```rust
(last_final_block_height, last_final_blocks_merkle_root, next_secp_merkle_root, next_total_stake)
```

For staking pools, we will change the contract code when it is touched. More specifically, `action_stake` will
be modified in the following way:
```rust
pub(crate) fn action_stake(
    account: &mut Account,
    result: &mut ActionResult,
    account_id: &AccountId,
    stake: &StakeAction,
    last_block_hash: &CryptoHash,
    epoch_info_provider: &dyn EpochInfoProvider,
) -> Result<(), RuntimeError> {
    if account.code_hash == STAKING_POOL_V1_HASH {
        action_deploy_contract(new_staking_pool_contract);
    }
    // Then proceed to do what the function does now.
}
```

## Changes to the bridge

Let's say the block that is signed is `B`, and the last final block in its ancestry is `F`.
`last_final_block_height` is the height of `F`, `last_final_blocks_merkle_root` is block_merkle_root of `F`,
and `next_secp_merkle_root` is the merkle root of the secp public keys and corresponding stakes of the validators in the next epoch.
Any final block can be relayed. The logic of the block relaying on Eth side is then (in pseudocode):
```
last_known_height
last_known_secp_merkle_root
last_known_total_stake
next_known_secp_merkle_root
next_known_total_stake
last_known_blocks_merkle_root
fn relay_block(block_height, blocks_merkle_root, cur_secp_merkle_root, cur_total_stake, next_secp_merkle_root, next_total_stake, signatures: Vec<public key, stake, signature, merkle proof>) {
   assert ((cur_secp_merkle_root == last_known_secp_merkle_root && cur_total_stake == last_known_total_stake) || (cur_secp_merkle_root == next_known_secp_merkle_root && cur_total_stake == next_known_total_stake));
   assert (last_known_height < block_height)

   signed_stake = 0
   for public_key, stake, signature, merkle_proof in signatures {
      assert merkle_root_matches((public_key, stake), merkle_path, cur_secp_merkle_root)
      assert signature validates(public_key, signature, (block_height, blocks_merkle_root, next_secp_merkle_root, next_total_stake))
      signed_stake += stake
   }
   assert signed_stake > total_stake * 2 / 3

   last_known_height = block_height
   last_known_secp_merkle_root = cur_secp_merkle_root
   next_known_secp_merkle_root = next_secp_merkle_root
   // same for stakes
   last_known_blocks_merkle_root = blocks_merkle_root
}
```

With this approach, we can calculate the new gas consumption on Eth side assuming that we have 100 validators on NEAR and therefore merkle proof of size 7 * 32 bytes:

For each signature, there is call data of size (64 + 7 * 32 + 32 + 16) * 16 GAS = 5376 and the cost of signature verification is 3700.
In addition, merkle verification costs 772 * 7 = 5404, which gives us a total of 14480.

Some possible savings here: we don’t need to send the public key along, because ecrecover can recover it, it will save 32*16=512 gas per signature.

For 34 signatures: 492320 gas (474912 if we save those 512 gas / signature)

For cost of storage, since we overwrite 6 words, it is 6 * 5000 = 30000. 

Base call cost is 21000 and this means that the total cost is 543320 gas + some overhead for loops and arithmetic.

We can also save on not storing total stake, and instead committing to it as part of the secp_merkle_root, it will save another 10K gas, but will add a SHA256 and a bit more overhead.

So the total cost is somewhere between 500K and 600K

When we switch to the new bridge design, the change will be to commit to the state merkle root, not to the blocks_merkle_root, but other than that it’s identical.

If a block contains most of the signatures (in particular, the signatures from the top validators that together have 34% of stake), the cost will be cheaper. E.g. today 10 signatures is enough to collect 34%, and thus the cost will be

30000 + 21000 + 14480 * 10 = 195800

## Changes to the staking pool contract

There are several contracts involved in this change. First, we need to modify [the existing staking pool contract](https://github.com/near/core-contracts/blob/master/staking-pool/src/lib.rs)
to include a new method called `upgrade`:
```rust
fn upgrade(&mut self, new_code_hash: CryptoHash) {
    // First check whether it is called by the owner of the staking pool contract.
    // ask the upgrader contract to give it the new contract binary with the given `code_hash`
    // after the receipt comes back, checks that code hash matches and deploy the new contract
}
```
and the upgrader contract is a factory contract which has a hardcoded account name and holds the new
staking pool contract. It will be deployed by NEAR Foundation.

The new staking pool contract will have another field `secp_public_key` and the staking actions used will also be changed
to `StakeAction2`. The field `secp_public_key` is initially empty and only set when the owner calls `update_staking_key`, which
will also be modified to include a secp public key.

Note that the state needs to be reset when the new staking pool contract is deployed through the call to `upgrade` to account for
the new field `secp_public_key`.

## The upgrade path

Due to the multifaceted nature of this proposal, it is important that we figure out the right way to upgrade.
The plan is as follows:
- Step 0: we upgrade `Approval` and `Block` as described above and we introduce `StakeAction2`. After this upgrade,
the signatures are not checked.
- Step 1: we deploy the upgrader contract and on the protocol level, implement the change that will upgrade the staking pool
contract.
- Step 2: After all current validators have upgraded their staking pool contracts and set their secp public key,
we enable the check for signatures and complete the entire upgrade process.

# Drawbacks
[drawbacks]: #drawbacks

From a design perspective, a major downside is that this proposal tightly couples the Eth <> NEAR bridge with
the protocol itself, since the message that validators sign with their secp keys is designed specifically for the bridge.
This means that we essentially build the bridge support into the protocol and need to maintain it going forward as well.

# Rationale and alternatives
[rationale-and-alternatives]: #rationale-and-alternatives

Assuming that we want to add secp validator keys, there are a few alternatives as to how exactly we do it.
For example, one alternative proposed by @SkidanovAlex is to, instead of upgrading staking pool contracts on the protocol level,
introduce a new action
```rust
pub struct SetValidatorExtraKeys { 
    account_id: AccountId,
    validator_key: ED25519PublicKey,
    secp_key: SECP256K1PublicKey,
    bls_key: BLS12381PublicKey,
    signature: ED25519Signature 
}
```
which allows an existing validator to bind a secp key to their ed25519 validator key. However, the downsides
of this approach are:
- We still need to implement a new staking pool contract with the new staking action that includes secp key for future validators.
- Adding an action as an temporary workaround isn't great as changes to transaction format needs to be supported by all the devtools
and they cannot really be later removed due to the existence of archival nodes.

Therefore we decide not to go with this approach and choose to upgrade the contracts through a protocol change instead.

# Unresolved questions
[unresolved-questions]: #unresolved-questions

The exact message to be signed by the secp validator key is still not fully decided. What is proposed currently is one option
and may not be the best one.

# Future possibilities
[future-possibilities]: #future-possibilities

The introduction of secp validator keys also enables us to potentially build a bridge to bitcoin.
