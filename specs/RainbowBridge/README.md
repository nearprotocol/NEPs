## Introduction

The Rainbow Bridge is represented by a family of smart contracts that implement light clients and arbitrary trustless actors who wish to forward necessary data between Near and Ethereum blockchains.

The API allows developers to:

- pass arbitrary data between smart contracts on different chains
- make arbitrary calls between smart contracts on different chains

The Rainbow Bridge vision (not available today) includes several other projects:

- a fully collateralized NEAR stable token available on the Ethereum network
- a fully collateralized ETH stable token available on the NEAR network
- smart contract based Ethereum proxy-accounts for NEAR users
- smart contract based NEAR proxy-accounts for Ethereum users

*The Rainbow Bridge can support any other blockchain for which these components are developed.*

## Architecture

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


## Use Cases

Please refer to the diagram above where (A), (B) and (C) appear to the left and (1), (2) and (3) appear the right.

### Use Case A-B-C-B-A

*NEAR smart contract must verify an event was recorded on the Ethereum blockchain*

- A contract running on the *NEAR blockchain* requires verifiable proof that an event has been recorded on the *Ethereum blockchain*.
- The contract `(A)` invokes a method on the `EthEventVerifier` `(B)`, passing in the appropriate parameters to represent the event and corresponding inclusion proof
- `EthEventVerifier` `(B)` verifies the Ethereum event inclusion into the corresponding block header which exists in the `EthClient` `(C)`
- `EthClient` `(C)` returns a boolean indicating whether the block hash was indeed recorded on the Ethereum blockchain
- `EthEventVerifier` `(B)` passes this result on to the contract `(A)` as proof or rejection of the event as valid

### Use Case 1-2-3-2-1

*Ethereum smart contract must verify a transaction result was recorded on the NEAR blockchain*

- A contract running on the *Ethereum blockchain* requires verifiable proof that a transaction result has been recorded on the *NEAR blockchain*.
- The contract `(1)` invokes a method on the `NearTxResultVerifier` `(2)`, passing in the appropriate parameters to represent the transaction result and inclusion proof
- `NearTxResultVerifier` `(2)` verifies the Near transaction belongs to the related block header which is stored in the `NearClient` `(3)`
- `NearClient` `(3)` returns a boolean indicating whether the block hash was indeed recorded on the NEAR blockchain
- `NearTxResultVerifier` `(2)` passes this result on to the contract `(1)` as proof or rejection of the transaction result as valid

*In both cases, the light clients `EthClient` and `NearClient` are supported by arbitrary trustless actors (`EthRelay` and `NearRelay`) who wish to forward necessary data between Near and Ethereum blockchains, respectively.*


## Related Documents

There are two other documents related to the Rainbow Bridge

(1) `Rainbow Bridge / Developers`

- Introduction
- Implementation
  - Core Components
  - Extensible Components
  - Supporting Scripts
- Usage
  - Installation
  - Examples
- Extensibility
  - Developing Verifiers
  - Developing Clients
- API
  - NearClient
  - EthClient
  - NearTxResultVerifier
  - EthEventVerifier

(2) `Rainbow Bridge / Maintainers.md`

- Introduction
- NEAR Protocol
  - Data Structures
  - Maintenance
  - Futures
- Ethereum
  - Data Structures
  - Maintenance
  - Futures
