## Introduction



## NEAR Protocol


### Data Structures


**Primitives**

*Please refer to [nearcore primitive types](https://github.com/nearprotocol/nearcore/blob/master/core/primitives/src/types.rs) for details*

```rust
Gas u64
Nonce u64
Balance u128
AccountId String
```

**Crypto**

*Please refer to [nearcore crypto signature](https://github.com/nearprotocol/nearcore/blob/master/core/crypto/src/signature.rs) for details*

```rust
struct CryptoHash([u8; 32])                                     // Sha256 digest
struct MerkleHash(CryptoHash)
struct EpochId(CryptoHash)

enum PublicKey {
    ED25519(sodiumoxide::crypto::sign::ed25519::PublicKey)      // [u8; 32]
    SECP256K1(Secp256K1PublicKey)                               // [u8; 64]
}

enum Signature {
    ED25519(sodiumoxide::crypto::sign::ed25519::Signature)      // [u8; 64]
    SECP256K1(Secp2561KSignature)                               // [u8; 65]
}
```

**Block header**

*Please refer to [nearcore block primitives](https://github.com/nearprotocol/nearcore/blob/master/core/primitives/src/block.rs) for details*

```rust
struct ValidatorStake {
    account_id AccountId                        // Account that stakes money.
    public_key PublicKey                        // Public key of the proposed validator.
    amount Balance                              // Stake / weight of the validator.
}

struct BlockHeaderInner {
    height u64                                  // Height of this block since the genesis block (height 0).
    epoch_id EpochId                            // Epoch start hash of this block's epoch. Used for retrieving validator information
    prev_hash CryptoHash                        // Hash of the block previous to this in the chain.
    prev_state_root MerkleHash                  // Root hash of the state at the previous block.
    tx_root MerkleHash                          // Root hash of the transactions in the given block.
    timestamp u64                               // Timestamp at which the block was built.
    approval_mask Vec<bool>                     // Approval mask, given current block producers.
    approval_sigs Vec<Signature>                // Approval signatures for previous block.
    total_weight Weight                         // Total weight.
    validator_proposals Vec<ValidatorStake>     // Validator proposals.
    chunk_mask Vec<bool>                        // Mask for new chunks included in the block
    gas_used Gas                                // Sum of gas used across all chunks.
    gas_limit Gas                               // Gas limit. Same for all chunks.
    gas_price Balance                           // Gas price. Same for all chunks
    total_supply Balance                        //
}

struct BlockHeader {
    inner BlockHeaderInner                      // Inner part of the block header that gets hashed.
    signature Signature                         // Signature of the block producer.
}
```




**Transaction**

*Please refer to [nearcore transaction primitives](https://github.com/nearprotocol/nearcore/blob/master/core/primitives/src/transaction.rs) for details*

Access

```rust
struct FunctionCallPermission {
    allowance Option<Balance>
    receiver_id AccountId
    method_names Vec<String>
}

enum AccessKeyPermission {
    FunctionCall(FunctionCallPermission)
    FullAccess
}


struct AccessKey {
    nonce Nonce
    permission AccessKeyPermission
}
```

Actions

```rust
struct CreateAccountAction

struct DeployContractAction {
    code Vec<u8>
}

struct FunctionCallAction {
    method_name String
    args Vec<u8>
    gas Gas
    deposit Balance
}

struct TransferAction {
    deposit Balance
}

struct StakeAction {
    stake Balance
    public_key PublicKey
}

struct AddKeyAction {
    public_key PublicKey
    access_key AccessKey
}

struct DeleteKeyAction {
    public_key PublicKey
}

struct DeleteAccountAction {
    beneficiary_id AccountId
}
```

Transaction

```rust
enum Action {
    CreateAccount(CreateAccountAction)
    DeployContract(DeployContractAction)
    FunctionCall(FunctionCallAction)
    Transfer(TransferAction)
    Stake(StakeAction)
    AddKey(AddKeyAction)
    DeleteKey(DeleteKeyAction)
    DeleteAccount(DeleteAccountAction)
}

struct Transaction {
    signer_id AccountId
    public_key PublicKey
    nonce Nonce
    receiver_id AccountId
    block_hash CryptoHash                       // The hash of the block in the blockchain on top of which the given transaction is valid.
    actions Vec<Action>
}

struct SignedTransaction {
    transaction Transaction
    signature Signature
}
```

**Transaction execution results**

*Please refer to [nearcore transaction primitives](https://github.com/nearprotocol/nearcore/blob/master/core/primitives/src/transaction.rs) for details*

```rust
enum TransactionStatus {
    Unknown
    Completed
    Failed
}

struct TransactionResult {
    status TransactionStatus                    // Transaction status.
    logs Vec<LogEntry>                          // Logs from this transaction.
    receipts Vec<CryptoHash>                    // Receipt ids generated by this transaction.
    result Option<Vec<u8>>                      // Execution Result
}

struct TransactionLog {
    hash CryptoHash                             // Hash of a transaction or a receipt that generated this result.
    result TransactionResult
}
```

**Final transaction result views**

```rust
enum FinalTransactionStatus {
    Unknown
    Started
    Failed
    Completed
}

struct TransactionResultView {
    status TransactionStatus
    logs Vec<LogEntry>
    receipts Vec<CryptoHashView>
    result Option<String>
}

struct TransactionLogView {
    hash CryptoHashView
    result TransactionResultView
}

struct FinalTransactionResult {
    status FinalTransactionStatus               // Status of the whole transaction and it's receipts.
    transactions Vec<TransactionLogView>        // Transaction results.
}
```

### Maintenance

### Futures



## Ethereum

### Data Structures

### Maintenance

### Futures