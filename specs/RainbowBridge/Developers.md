## Introduction


## Implementation


```text
+-------------------------------+                 +-------------------------------+
|  NEAR Blockchain              |                 | Ethereum Blockchain           |
|                               |                 |                               |
|  .-------------.              |                 |              .-------------.  |
|  | .-------------.            |                 |            .-------------. |  |
|  '-| .-------------.          |                 |          .-------------. |-'  |
|    '-|  contracts  |      (A) |                 | (1)      |  contracts  |-'    |
|      '-----+-+-----'          |                 |          '-----+-+-----'      |
|            | |                |                 |                | |            |
|            \ /                |                 |                \ /            |
|  +----------------------+     |                 |     +----------------------+  |
|  |                      |     |                 |     |                      |  |
|  |   EthEventVerifier   | (B) |                 | (2) | NearTxResultVerifier |  |
|  |                      |     |                 |     |                      |  |
|  +-------------+-+------+     |                 |     +-----+-+--------------+  |
|                | |            |                 |           | |                 |
|                \ /            |                 |           \ /                 |
|          +--------------+     \                 |     +--------------+          |
|          |              |      o - NearRelay --------->              |          |
|          |              |     /                 |     |              |          |
|          |  EthClient   | (C) |                 | (3) |  NearClient  |          |
|          |              |     |                 /     |              |          |
|          |              <--------- EthRelay  - o      |              |          |
|          +--------------+     |                 \     +--------------+          |
|                               |                 |                               |
+-------------------------------+                 +-------------------------------+
```


### Core Components

Two smart contracts, `(C)` and `(3)` establish the baseline source of truth for blockchain activity on the "other" network.

- **(C)** `NearClient`- *smart contract hosted on **Ethereum** network*  \
   This contract is a NEAR light client that receives NEAR block headers.  It verifies the NEAR chain of blocks following Near Light Client Specification and stores block hashes only.

- **(3)** `EthClient` – *smart contract hosted on the **NEAR** network*  \
   This contract is an Ethereum light client that receives Ethereum block headers.  It verifies ethash and longest chain rule and stores block hashes only.

### Extensible Components

Two smart contracts demonstrate what is possible using the Rainbow Bridge.  These contract are intended to be used as **reference material** for developing other contracts about various blockchain platforms.

*A note about NEAR and Ethereum blockchains*

- At its core, NEAR is a sharded transaction processing engine that records *transaction results*.  These *transaction results* are a verifiable source of truth about the NEAR blockchain.
- At its core, Ethereum is single-threaded computer that, at various points in time, records *events*.  These *events* are a verifiable source of truth about the Ethereum blockchain.

- **(B)** `NearTxResultVerifier` - *smart contract on **Ethereum** network*  \
  This contract performs verification of **NEAR transaction results** that have been included into NEAR blocks. It uses Merkle trees and hash preimages for verification.

- **(2)** `EthEventVerifier` - *smart contract on **NEAR** network*  \
  This contract performs verification of **Ethereum events** that have been included into Ethereum blocks. It uses Merkle trees and hash preimages for verification.

### Supporting Scripts

- `NearRelay` - an arbitrary, trustless actor that forwards NEAR block headers to the `NearClient` smart contract which is hosted on the Ethereum network.
- `EthRelay` - an arbitrary, trustless actor that forwards Ethereum block headers to the `EthClient` smart contract which is hosted on the NEAR network.


## Usage

### Installation

1. clone the repo `git clone --recurse-submodules git@github.com:nearprotocol/near-bridge.git`
2. run `NearRelay` node script to do two things:
   - deploy `NearClient` contract (written in Solidity) to Ethereum network (note: just use ABI file to connect if the `NearClient` contract is already deployed)
   - start sending NEAR block headers to `NearClient` every 10 seconds
3. build and deploy `EthClient` contract (written in Rust and compiled to Wasm) to NEAR network
4. run `EthRelay` script to start sending Ethereum block headers to `EthClient` as often as they're available, at least once every 10 seconds

Now both networks have knowledge of one another

- Ethereum Network is queried by `EthRelay` at ~10s intervals for latest block headers.  `EthRelay` then sends this to `EthClient`, a contract deployed on the NEAR network
- NEAR network is queried by `NearRelay` at ~10s intervals for latest block headers.  `NearRelay` then sends this to `NearClient`, a contract deployed on the Ethereum network

As part of our development of the Rainbow Bridge framework, we have chosen something about each network (Ethereum and NEAR) which is as close to "ground truth" as possible. By this we mean some fact or detail about the network which establishes a baseline from which we can build up.

For Ethereum we've selected **cryptographic proofs of events** on the Ethereum network.  This is called `EthEventVerifier`, a contract deployed on the NEAR network which can verifiably prove that an event occurred on the Ethereum network.  You could say this contract is available for use by any other contracts running on the **NEAR network** as an *Ethereum Event Oracle*.

For NEAR we've selected **cryptographic proofs of transaction execution results** on the NEAR network.  This is called `NearTxResultVerifier`, a contract deployed on the Ethereum network which can verifiably prove that a transaction was executed on the NEAR network.  You could say this contract is available for use by any other contracts running on the **Ethereum network** as a *Near Transaction Result Oracle*.

Once deployed, `EthEventVerifier` and `NearTxResultVerifier` are available for use by smart contracts operating in their respective environments.

### Examples

## Extensibility

### Developing Verifiers

### Developing Clients



## API

### NearClient

```java
contract NearBridge is Ownable {
    mapping(uint256 => bytes32) public blockHashes;
    function addBlockHeaders(bytes[] memory blockHeaders) public onlyOwner {}
}
```

### EthClient

```rust
pub struct DoubleNodeWithMerkleProof {}
impl DoubleNodeWithMerkleProof {}
pub struct HeaderInfo {}

#[near_bindgen]
pub struct EthBridge {}
impl Default for EthBridge {}

#[near_bindgen]
impl EthBridge {
    // EthRelay-specific API methods:
    pub fn last_block_number(&self) -> u64;
    pub fn add_block_header(&mut self, block_header: Vec<u8>, dag_nodes: Vec<DoubleNodeWithMerkleProof>);

    // Public smart contracts API methods:
    pub fn block_hash(&self, index: u64) -> Option<H256>;
    pub fn block_hash_safe(&self, index: u64) -> Option<H256>;
}
```

### NearTxResultVerifier

### EthEventVerifier