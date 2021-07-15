- Proposal Name: `voting_contract`
- Start Date: 05-02-2020
- NEP PR: [nearprotocol/neps#0000](https://github.com/nearprotocol/neps/pull/0000)
- Issue(s): https://github.com/nearprotocol/nearcore/issues/2474, https://github.com/nearprotocol/nearcore/issues/2475.

# Summary
[summary]: #summary

This NEP proposes a way to implement voting contract on chain. This voting contract can be used for various types of
governance activities, including but not limited to deciding when to unlock token transfer, when to upgrade the network, etc.
More specifically, this contract allows anyone to make proposals (such as the height of the next network reset), but limits
voting to validators. Once a proposal receives a predefined fraction of the total votes, it is considered final.

# Motivation
[motivation]: #motivation

For a decentralized protocol, governance never fails to be one of the most important piece of the entire system. Without
proper governance the network will likely fall apart. On-chain voting is a crucial constituent of governance and this NEP
aims to make a first step in that direction by laying out the framework of a general-purpose voting contract that allows
validators as a community to make decisions on chain. 

# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

The structure of this contract looks like the following:

```rust
pub type ProposalId = U64;

#[near_bindgen]
pub struct Poll {
    /// Human readable description of the poll.
    description: String,
    /// All proposals for this poll.
    proposals: Map<ProposalId, Proposal>,
    /// Accounts that have participated in this poll and the corresponding stake voted.
    accounts: Map<AccountId, Balance>,
    /// Next proposal id.
    next_proposal_id: ProposalId,
    /// Threshold for closing the poll, i.e, if the ratio of stake on a certain proposal over total stake reaches
    /// threshold, the poll is closed.
    threshold: Fraction,
    /// Fee required to submit a proposal to avoid spamming the state.
    proposal_init_fee: Balance,
    /// Voting result. `None` means the poll is still open.
    result: Option<ProposalId>,
}
```

On the top level we maintain a poll where there are multiple proposals that can be voted on. The poll keeps track of the
proposals and accounts that have participated in the poll so far and their total voted stake. When the stake voted on a given
proposal reaches the threshold for this poll, the proposal becomes final and the poll is closed.

As an important constituent of the poll, `Proposal` contains all the voting information of this particular proposal,
in addition to the actual content of the proposal:
```rust
pub struct Proposal {
    /// Human readable description of the proposal.
    description: String,
    /// Serialized metadata of the proposal.
    metadata: String,
    /// When this proposal expires.
    expiration_height: BlockHeight,
    /// Current votes on this proposal.
    votes: Map<AccountId, Balance>,    
}
```
Here `metadata` is the json serialized content of the proposal. For example, if the poll is about when the network upgrade
should happen, `metadata` will be something like `{"height": 10000}`. Validators can vote on multiple different proposals
at the same time, provided that the sum of stake voted does not exceed their current stake.

The voting contract has the following methods to allow creation of polls and proposals, as well as voting for proposals.

```rust
impl Poll {
    /// Initialize the poll on some topic.
    pub fn new(topic: String, threshold: Fraction, proposal_init_fee: U128) -> Self;
    /// Create a proposal for a given poll.
    #[payable]
    pub fn create_proposal(&mut self, description: String, metadata: String) -> ProposalId;
    /// Vote on a given proposal with certain amont of stake.
    pub fn vote(&mut self, proposal_id: ProposalId, stake: U128);
    /// View proposal
    pub fn get_proposal(&mut self, proposal_id: ProposalId) -> Proposal;
    /// Get result of the poll. `None` if the poll is still open.
    pub fn get_result(&mut self) -> Option<PollResult>;
}
```

Anyone can create a proposal by calling `create_proposal`, provided that they also pay a fee for submitting this proposal
specified by this poll.

Validators then use `vote` function to vote on proposals. 
Notice that the `vote` function can also be used to withdraw a vote by putting 0 stake on the vote.

When a majority agreed on some proposal and the poll ends, we record the result in `PollResult`, which includes not only
the winning proposal but also some context such as block height and block timestamp:

```rust
pub struct PollResult {
    /// Id of the winning prooposal.
    pub proposal_id: ProposalId,
    /// Block height at which the poll ends.
    pub block_height: BlockHeight,
    /// Timestamp of the block at which the poll ends.
    pub block_timestamp: u64
}
```

# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

A major difficulty in managing the votes comes from the fact that validator stake can change from epoch to epoch, which
means that we need to carefully update validator votes. For example, if a validator has 100 stake in the previous epoch and
they voted 80 on some proposal, but in the current epoch their stake has decreased to 80, and now if they try to vote 20
on some other proposal, it should fail. To address this, we introduce a helper function `resolve_proposal`, which is called
before `vote` to ensure consistency of voted stake within a proposal:
```rust
fn resolve_proposal(&mut self, proposal_id: ProposalId) {
    // if the epoch height has changed, then
    // for each vote in the proposal, its stake is changed to orginal_stake * current_total_account_stake / previous_total_account_stake.
}
```

Since the contract requires knowing the stake of current validators, we need to augment our runtime to expose that information.
More specifically, we need the following two functions in `near-vm-logic`:

```rust
/// Returns the stake of an account, if the account is currently a validator. Otherwise returns 0.
///
/// # Cost
///
/// For not nul-terminated account id:
/// `base + read_memory_base + read_memory_byte * num_bytes + utf8_decoding_base + utf8_decoding_byte * num_bytes + memory_write_base + memory_write_size * 16 + validator_stake_base`
///
/// For nul-terminated account id:
/// `base + (read_memory_base + read_memory_byte) * num_bytes + utf8_decoding_base + utf8_decoding_byte * num_bytes + memory_write_base + memory_write_size * 16 + validator_stake_base`
pub fn validator_stake(
    &mut self,
    account_id_len: u64,
    account_id_ptr: u64,
    stake_ptr: u64,
) -> Result<()>;

/// Returns the total validator stake of the current epoch.
///
/// # Cost
///
/// `base + memory_write_base + memory_write_size * 16 + validator_total_stake_base`
pub fn validator_total_stake(&mut self, stake_ptr: u64) -> Result<()>;
```

This allows us to know on the smart contract side whether an account id is allowed to vote, how much stake they currently have,
and the total stake in the current epoch, which together are sufficient for calculating the result of voting.

# Drawbacks
[drawbacks]: #drawbacks

- In the current proposal, we do not allow the contract to hold a number of polls. This reduces the complexity of the contract
  and makes it easier to implement. However, it can be argued that having multiple polls in one place allows people to easily view and vote on different
  polls at the same time, which is better for governance purposes.

# Rationale and alternatives
[rationale-and-alternatives]: #rationale-and-alternatives

-  In the current design, the weight on each vote is exactly the amount of stake allocated for this vote, which is very
straightforward. We can consider some alternatives like quadratic voting which might be better in terms of
expressing preferences.

# Unresolved questions
[unresolved-questions]: #unresolved-questions

- After the poll is finished, how long should we keep them?
- When the state of the contract is large (a lot of proposals), it is possible that `resolve` will not be able to finish
in one function call due to gas limit.
