- Proposal Name: Dapp nodes
- Start Date: 2020-01-17
- NEP PR: [nearprotocol/neps#0028](https://github.com/nearprotocol/neps/pull/28

# Summary
[summary]: #summary

Currently we see no efficient way to allow view-calls of smart contract to have subcalls of view-methods of other smart contracts. Or even simulate non-view transactions as view-call, as Etheruem nodes allows for example.

We propose this NEP to define architecture of software allowing clients to execute view-calls of any complexity. This should be light-client of multiple shard tracking storage of dapp smart contracts and related to subcalls. Dapp node would allow view-call to only few pre-configured smart contracts, which could utilize other smart contracts and all touched smart contracts should be tracked by node.

# Motivation
[motivation]: #motivation

We need apps to access smart contracts data as some dapps are completely backendless and smart contract is their backed.

# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation



# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation



# Drawbacks
[drawbacks]: #drawbacks



# Rationale and alternatives
[rationale-and-alternatives]: #rationale-and-alternatives



# Unresolved questions
[unresolved-questions]: #unresolved-questions



# Future possibilities

[future-possibilities]: #future-possibilities


