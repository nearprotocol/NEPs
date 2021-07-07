- Proposal Name: randomness\_implementation
- Start Date: 2019-11-05
- NEP PR: [nearprotocol/neps#0000](https://github.com/nearprotocol/neps/pull/0022)
- Issue(s): #779

# Summary
[summary]: #summary

This NEP walks through proposed implementation details of the randomness beacon described in [this paper](https://near.ai/randomness)

# Motivation
[motivation]: #motivation

The paper describes a protocol that takes `O(n^3)` memory overhead, and is mostly off-chain.

This NEP proposes a particular implementation that is only `O(n^2)` in the common case (still resorting to `O(n^3)` if `O(n)` block producers are offline or slow), and reuses messages already used in NEAR today, thus significantly reducing the scope of the implementation.

# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

The protocol described in the paper consists of fives steps: Commitment, Consensus, Reveal, Recovery, Output. The last two steps are offline for each block producer. The first three are done in the following way:

## A state machine

We add an extra step for block processing that involves a state machine that has the following states:

 - `NotRevealing` -- no reveal is currently in progress
 - `Revealing since <i>` -- a reveal process is underway, and has started at height `i`
 - `Producing output <v> from <i>` -- a value `v` was emitted, that was unpredictable up until height `i`.

## Commitment
Commitment happens in parallel to the state machine, and is not affected by the current state of it.

A commitment by a block producer is a vector of size `n` (here and below `n` is the number of block producers). Creating a commitment on each block and broadcasting it is extremely wasteful and is what contributes to `O(n^3)` complexity of the commitment phase (reveal is also `O(n^3)`, and is addressed below).

Instead we set some constant `k`, say `k=10`, and for each block producer maintain up to `k` vectors that are not broadcasted, but instead are stored in chunks. Whenever a particular block producer produces a chunk, they include enough vectors in the chunk so that after applying this chunk they have `k` vectors committed.

The header of the chunk contains corresponding commitments (merkle roots of such vectors) for each vector.

The `PartialEncodedChunkMessage`s that are sent to other block producer contain the shares of the vector with the merkle proofs that are specific to that block producer. The logic of retrieving `PartialEncodedChunkMessage`s is changed such that the block producer keeps retrying until all the shares are received.

Since a block is orhpaned unless the block producer has the full chunk or sufficient `PartialEncodedChunkMessage` to reconstruct the shares, for each block that contains the vectors all block producers in the system who accepted the block have their shares.

## Consensus

When a block at height `i` is processed, if the current state of the state machine is `NotRevealing`, we do the following:

 1. Identify the latest block in the current chain that is finalized;
 2. If there are more than 50% of block producers (stake-weighted) that have a committed vector that was committed before or at the last final block, and was not used yet, the state machine moves to state `Revealing since <i>`.

## Reveal
If after processing a block at height `n'` the state machine is in a state `Revealing since <i>` for some `i`, each block producer when creating their `Approval` message for the block include the seed for their earliest unused commitment into their approval.

If all the seeds are revealed, the state machine moves to the state `Producing output <v> from <i>`, where `v` is the `xor` of all the valid seeds (seeds that match the commitment).

If some seeds are missing, and `n' - i >= z` for some system parameter `z` (in practice `z` should be around 2), each block producer `x` creates a message with all the decoded `s_{z_j, x}` or proofs that corresponding elements cannot be decrypted, following the subsection `Reveal` of section 2 in the paper. Such messages are broadcast off-chain via the gossip network, and are locally accumulated by all block producers. If a block producer has enough of such messages to decode all missing seeds, they publish them in the block. Processing such a block naturally moves the state machine into state `Producing output <v> from <i>`.

If at any moment the state machine is in the state `Producing output <v> from <i>`, it immediately moves to the state `NotRevealing`.

# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

## RndCommitment

The commitment is a new struct `struct RndCommitmentVector { data: Vec[Share] }`, where `Share` is a 32-byte long blob.

## ShardChunk

`ShardChunk` gets a new field `rnd_commitment_vectors: Vec<RndCommitmentVector>`. The field is encoded alongside transactions and receipts when the chunk is encoded.

`ShardChunkHeader` gets a new field `rnd_commitments: Vec<MerkleRoot>`.

When `ShardChunk` is processed, it is only considered valid if after applying it the chunk producer has exactly `k` commitments, and if all the `rnd_commitments` match `rnd_commitment_vectors`.

## Approval

`Approval` gets a new field `revealed_seed`, which contains the seed of a block producer that corresponds to a committed vector used for the current reveal, if any. When processed, if the seed doesn't correspond to the commit, it is ignored.

## Reveal message

A new enum that represents the result of decrypting one share of the committed vector is created `enum RndOneElemRevealResult { Ok(Share), Err(RndRevealFailureProof) }`, and a new message is added `RndRevealMessage { ordinal: u64, data: Vec<(AccountId, RndOneElemRevealed)> }` that contains decoded shares for the seeds that were not revealed (where `ordinal` is the ordinal of the revealed elements in the corresponding commitment vectors).

## Block randomness output

If a random number was emitted during processing of a block (which means the state machine went through a state `Producing output <v> from <i>` while the block was processed), the block producer needs to include all the seeds or proofs that they can't be recovered into the block.

A new data structure is introduced `struct RndSeedRevealResult{ Seed(Share), ElemNotDecrypted { ordinal: u64, proof: RndRevealFailureProof }, ErasureDecodeFailure(Vec<Option<Share>>) }`, where the first error corresponds to having an element in the committed vector that cannot be decrypted, and the second error corresponds to having the decrypted commitment vector not being a valid erasure coded vector.

An extra element is added to the `Block` that contains a vector of such structs, that is empty unless an output was emitted, otherwise the set of elements corresponding for each block producer agreed upon during the commitment phase. Each seed failed to recover needs to result in a challenge.

