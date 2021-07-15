- Proposal Name: Batched Transactions
- Start Date: 2019-07-22
- NEP PR: [nearprotocol/neps#0008](https://github.com/nearprotocol/neps/pull/8)

# Summary
[summary]: #summary

Refactor signed transactions and receipts to support batched atomic transactions and data dependency.

# Motivation
[motivation]: #motivation

It simplifies account creation, by supporting batching of multiple transactions together instead of
creating more complicated transaction types.

For example, we want to create a new account with some account balance and one or many access keys, deploy a contract code on it and run an initialization method to restrict access keys permissions for a `proxy` function.

To be able to do this now, we need to have a `CreateAccount` transaction with all the parameters of a new account.
Then we need to handle it in one operation in a runtime code, which might have duplicated code for executing some WASM code with the rollback conditions.

Alternative to this is to execute multiple simple transactions in a batch within the same block.
It has to be done in a row without any commits to the state until the entire batch is completed.
We propose to support this type of transaction batching to simplify the runtime.

Currently callbacks are handled differently from async calls, this NEP simplifies data dependencies and callbacks by unifying them.

# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

### New transaction and receipts

Previously, in the runtime to produce a block we first executed new signed transactions and then executed received receipts. It resulted in duplicated code that might be shared across similar actions, e.g. function calls for async calls, callbacks and self-calls.
It also increased the complexity of the runtime implementation.

This NEP proposes changing it by first converting all signed transactions into receipts and then either execute them immediately before received receipts, or put them into the list of the new receipts to be routed.
To achieve this, NEP introduces a new message `Action` that represents one of atomic actions, e.g. a function call.
`TransactionBody` is now called just `Transaction`. It contains the list of actions that needs to be performed in a single batch and the information shared across these actions.

`Transaction` contains the following fields
- `signer_id` is an account ID of the transaction signer.
- `public_key` is a public key used to identify the access key and to sign the transaction.
- `nonce` is used to deduplicate and order transactions (per access key).
- `receiver_id` is the account ID of the destination of this transaction. It's where the generated receipt will be routed for execution.
- `action` is the list of actions to perform.

An `Action` can be of the following:
- `CreateAccount` creates a new account with the `receiver_id` account ID. The action fails if the account already exists. `CreateAccount` also grants permission for all subsequent batched action for the newly created account. For example, permission gto deploy code on the new account. Permission details are described in the reference section below. 
- `DeployContract` deploys given binary wasm code on the account. Either the `receiver_id` equals to the `signer_id`, or the batch of actions has started with `CreateAccount`, which granted that permission.
- `FunctionCall` executes a function call on the last deployed contract. The action fails if the account or the code doesn't exist. E.g. if the previous action was `DeployContract`, then the code to execute will be the new deployed contract. `FunctionCall` has `method_name` and `args` to identify method with arguments to call. It also has `gas` and the `deposit`. `gas` is a prepaid amount of gas for this call (the price of gas is determined when a signed transaction is converted to a receipt. `deposit` is the attached deposit balance of NEAR tokens that the contract can spend, e.g. 10 tokens to pay for a crypto-corgi.
- `Transfer` transfers the given `deposit` balance of tokens from the predecessor to the receiver.
- `Stake` stakes the new total `stake` balance with the given `public_key`. The difference in stake is taken from the account's balance (if the new stake is greater than the current one) at the moment when this action is executed, so it's not prepaid. There is no particular reason to stake on behalf of a newly created account, so we may disallow it.
- `DeleteKey` deletes an old `AccessKey` identified by the given `public_key` from the account. Fails if the access key with the given public key doesn't exist. All next batched actions will continue to execute, even if the public key that authorized that transaction was removed.
- `AddKey` adds a new given `AccessKey` identified by a new given `public_key` to the account. Fails if an access key with the given public key already exists. We removed `SwapKeyTransaction`, because it can be replaced with 2 batched actions - delete an old key and add a new key.
- `DeleteAccount` deletes `receiver_id` account if the account doesn't have enough balance to pay the rent, or the `receiver_id` is the `predecessor_id`. Sends the remaining balance to the `beneficiary_id` account.

The new `Receipt` contains the shared information and either one of the receipt actions or a list of actions:
- `predecessor_id` the account ID of the immediate previous sender (predecessor) of this receipt. It can be different from the `signer_id` in some cases, e.g. for promises.
- `receiver_id` the account ID of the current account, on which we need to perform action(s).
- `receipt_id` is a unique ID of this receipt (previously was called `nonce`). It's generated from either the signed transaction or the parent receipt.
- `receipt` can be one of 2 types:
  - `ActionReceipt` is used to perform some actions on the receiver.
  - `DataReceipt` is used when some data needs to be passed from the predecessor to the receiver, e.g. an execution result.

To support promises and callbacks we introduce a concept of cross-shard data sharing with dependencies. Each `ActionReceipt` may have a list of input `data_id`. The execution will not start until all required inputs are received. Once the execution completes and if there is `output_data_id`, it produces a `DataReceipt` that will be routed to the `output_receiver_id`.
 
`ActionReceipt` contains the following fields:
- `signer_id` the account ID of the signer, who signed the transaction.
- `signer_public_key` the public key that the signer used to sign the original signed transaction.
- `output_data_id` is the data ID to create DataReceipt. If it's absent, then the `DataReceipt` is not created.
- `output_receiver_id` is the account ID of the data receiver. It's needed to route `DataReceipt`. It's absent if the DataReceipt is not needed.
- `input_data_id` is the list of data IDs that are required for the execution of the `ActionReceipt`. If some of data IDs is not available when the receipt is received, then the `ActionReceipt` is postponed until all data is available. Once the last `DataReceipt` for the required input data arrives, the action receipt execution is triggered.
- `action` is the list of actions to execute. The execution doesn't need to validate permissions of the actions, but need to fail in some cases. E.g. when the receiver's account doesn't exist and the action acts on the account, or when the action is a function call and the code is not present.

`DataReceipt` contains the following fields:
- `data_id` is the data ID to be used as an input.
- `success` is true if the `ActionReceipt` that generated this `DataReceipt` finished the execution without any failures.
- `data` is the binary data that is returned from the last action of the `ActionReceipt`. Right now, it's empty for all actions except for function calls. For function calls the data is the result of the code execution. But in the future we might introduce non-contract state reads.

Data should be stored at the same shard as the receiver's account, even if the receiver's account doesn't exist.

### Refunds

In case an `ActionReceipt` execution fails the runtime can generate a refund.
We've removed `refund_account_id` from receipts, because the account IDs for refunds can be determined from the `signer_id` and `predecessor_id` in the `ActionReceipt`.
All unused gas and action fees (also measured in gas) are always refunded back to the `signer_id`, because fees are always prepaid by the signer. The gas is converted into tokens using the `gas_price`.
The deposit balances from `FunctionCall` and `Transfer` are refunded back to the `predecessor_id`, because they were deducted from predecessor's account balance.
It's also important to note that the account ID of predecessor for refund receipts is `system`.
It's done to prevent refund loops, e.g. when the account to receive the refund was deleted before the refund arrives. In this case the refund is burned.

If the function call action with the attached `deposit` fails in the middle of the execution, then 2 refund receipts can be generated, one for the unused gas and one for the deposits.
The runtime should combine them into one receipt if `signer_id` and `predecessor_id` is the same.

Example of a receipt for a refund of `42000` atto-tokens to `vasya.near`:
```json
{
    "predecessor_id": "system",
    "receiver_id": "vasya.near",
    "receipt_id": ...,

    "action": {
        "signer_id": "vasya.near",
        "signer_public_key": ...,
        
        "gas_price": "3",
        
        "output_data_id": null,
        "output_receiver_id": null,

        "input_data_id": [],
        
        "action": [
            {
                "transfer": {
                    "deposit": "42000"
                }
            }
        ]
    }
}
```
### Examples

#### Account Creation

To create a new account we can create a new `Transaction`:

```json
{
    "signer_id": "vasya.near",
    "public_key": ...,
    "nonce": 42,
    "receiver_id": "vitalik.vasya.near",

    "action": [
        {
            "create_account": {
            }
        },
        {
            "transfer": {
                "deposit": "19231293123"
            }
        },
        {
            "deploy_contract": {
                "code": ...
            }
        },
        {
            "add_key": {
                "public_key": ...,
                "access_key": ...
            }
        },
        {
            "function_call": {
                "method_name": "init",
                "args": ...,
                "gas": 20000,
                "deposit": "0"
            }
        }
    ]
}
```

This transaction is sent from `vasya.near` signed with a `public_key`.
The receiver is `vitalik.vasya.near`, which is a new account id.
The transaction contains a batch of actions.
First we create the account, then we transaction a few tokens on the new account, then we deploy code on the new account, add a new access key with some given public key, and as a final action initializing the deployed code by calling a method `init` with some arguments.

For this transaction to work `vasya.near` needs to have enough balance on the account cover gas and deposits for all actions at once.
Every action has some associated action gas fee with it. While `transfer` and `function_call` actions need additional balance for deposits and gas (for executions and promises).

Once we validated and subtracted the total amount from `vasya.near` account, this transaction is transformed into a `Receipt`:

```json
{
    "predecessor_id": "vasya.near",
    "receiver_id": "vitalik.vasya.near",
    "receipt_id": ...,

    "action": {
        "signer_id": "vasya.near",
        "signer_public_key": ...,
        
        "gas_price": "3",
        
        "output_data_id": null,
        "output_receiver_id": null,

        "input_data_id": [],
        
        "action": [...]
    }
}
```
In this example the gas price at the moment when the transaction was processed was 3 per gas.
This receipt will be sent to `vitalik.vasya.near`'s shard to be executed.
In case the `vitalik.vasya.near` account already exists, the execution will fail and some amount of prepaid_fees will be refunded back to `vasya.near`.
If the account creation receipt succeeds, it wouldn't create a `DataReceipt`, because `output_data_id` is `null`.
But it will generate a refund receipt for the unused portion of prepaid function call `gas`.

#### Deploy code example

Deploying code with initialization is pretty similar to creating account, except you can't deploy code on someone else account. So the transaction's `receiver_id` has to be the same as the `signer_id`.

#### Simple promise with callback

Let's say the transaction contained a single action which is a function call to `a.contract.near`.
It created a new promise `b.contract.near` and added a callback to itself.
Once the execution completes it will result in the following new receipts:

The receipt for the new promise towards `b.contract.near`
```json
{
    "predecessor_id": "a.contract.near",
    "receiver_id": "b.contract.near",
    "receipt_id": ...,

    "action": {
        "signer_id": "vasya.near",
        "signer_public_key": ...,
        
        "gas_price": "3",

        "output_data_id": "data_123_1",
        "output_receiver_id": "a.contract.near",

        "input_data_id": [],
        
        "action": [
            {
                "function_call": {
                    "method_name": "sum",
                    "args": ...,
                    "gas": 10000,
                    "deposit": "0"
                }
            }
        ]
    }
}
```
Interesting details:
- `signer_id` is still `vasya.near`, because it's the account that initialized the transaction, but not the creator of the promise.
- `output_data_id` contains some unique data ID. In this example we used `data_123_1`.
- `output_receiver_id` indicates where to route the result of the execution.


The other receipt is for the callback which will stay in the same shard.
```json
{
    "predecessor_id": "a.contract.near",
    "receiver_id": "a.contract.near",
    "receipt_id": ...,

    "action": {
        "signer_id": "vasya.near",
        "signer_public_key": ...,
        
        "gas_price": "3",

        "output_data_id": null,
        "output_receiver_id": null,

        "input_data_id": ["data_123_1"],
        
        "action": [
            {
                "function_call": {
                    "method_name": "process_sum",
                    "args": ...,
                    "gas": 10000,
                    "deposit": "0"
                }
            }
        ]
    }
}
```
It looks very similar to the new promise, but instead of `output_data_id` it has an `input_data_id`.
This action receipt will be postponed until the other receipt is routed, executed and generated a data receipt.

Once the new promise receipt is successfully executed, it will generate the following receipt:
```json
{
    "predecessor_id": "b.contract.near",
    "receiver_id": "a.contract.near",
    "receipt_id": ...,

    "data": {
        "data_id": "data_123_1",
        "success": true,
        "data": ...
    }
}
```
It contains the data ID `data_123_1` and routed to the `a.contract.near`.

Let's say the callback receipt was processed and postponed, then this data receipt will trigger execution of the callback receipt, because the all input data is now available.

#### Remote callback with 2 joined promises, with a callback on itself

Let's say `a.contract.near` wants to call `b.contract.near` and `c.contract.near`, and send the result to `d.contract.near` for joining before processing the result on itself.
It will generate 2 receipts for new promises, 1 receipt for the remote callback and 1 receipt for the callback on itself.

Part of the receipt (#1) for the promise towards `b.contract.near`:
```
...
"output_data_id": "data_123_b",
"output_receiver_id": "d.contract.near",

"input_data_id": [],
...
```

Part of the receipt (#2) for the promise towards `c.contract.near`:
```
...
"output_data_id": "data_321_c",
"output_receiver_id": "d.contract.near",

"input_data_id": [],
...
```

The receipt (#3) for the remote callback that has to be executed on `d.contract.near` with data from `b.contract.near` and `c.contract.near`:
```json
{
    "predecessor_id": "a.contract.near",
    "receiver_id": "d.contract.near",
    "receipt_id": ...,

    "action": {
        "signer_id": "vasya.near",
        "signer_public_key": ...,
        
        "gas_price": "3",

        "output_data_id": "bla_543",
        "output_receiver_id": "a.contract.near",

        "input_data_id": ["data_123_b", "data_321_c"],
        
        "action": [
            {
                "function_call": {
                    "method_name": "join_data",
                    "args": ...,
                    "gas": 10000,
                    "deposit": "0"
                }
            }
        ]
    }
}
```
It also has the `output_data_id` and `output_receiver_id` that is specified back towards `a.contract.near`.

And finally the part of the receipt (#4) for the local callback on `a.contract.near`:
```
...
"output_data_id": null,
"output_receiver_id": null,

"input_data_id": ["bla_543"],
...
```

For all of this to execute the first 3 receipts needs to go to the corresponding shards and be processed.
If for some reason the data arrived before the corresponding action receipt, then this data will be hold there until the action receipt arrives.
An example for this is if the receipt #3 is delayed for some reason, while the receipt #2 was processed and generated a data receipt towards `d.contract.near` which arrived before #3. 

Also if any of the function calls fail, the receipt still going to generate a new `DataReceipt` because it has `output_data_id` and `output_receiver_id`. Here is an example of a DataReceipt for a failed execution:
```json
{
    "predecessor_id": "b.contract.near",
    "receiver_id": "d.contract.near",
    "receipt_id": ...,

    "data": {
        "data_id": "data_123_b",
        "success": false,
        "data": null
    }
}
```

#### Swap Key example

Since there are no swap key action, we can just batch 2 actions together. One for adding a new key and one for deleting the old key. The actual order is not important if the public keys are different, but if the public key is the same then you need to first delete the old key and only after this add a new key.


# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation


### Updated protobufs

**public_key.proto**
```proto
syntax = "proto3";

message PublicKey {
    enum KeyType {
        ED25519 = 0;
    }
    KeyType key_type = 1;
    bytes data = 2;
}
```

**signed_transaction.proto**
```proto
syntax = "proto3";

import "access_key.proto";
import "public_key.proto";
import "uint128.proto";

message Action {
    message CreateAccount {
        // empty
    }

    message DeployContract {
        // Binary wasm code
        bytes code = 1;
    }

    message FunctionCall {
        string method_name = 1;
        bytes args = 2;
        uint64 gas = 3;
        Uint128 deposit = 4;
    }

    message Transfer {
        Uint128 deposit = 1;
    }

    message Stake {
        // New total stake
        Uint128 stake = 1;
        PublicKey public_key = 2;
    }

    message AddKey {
        PublicKey public_key = 1;
        AccessKey access_key = 2;
    }

    message DeleteKey {
        PublicKey public_key = 1;
    }

    message DeleteAccount {
        // The account ID which would receive the remaining funds.
        string beneficiary_id = 1;
    }

    oneof action {
        CreateAccount create_account = 1;
        DeployContract deploy_contract = 2;
        FunctionCall function_call = 3;
        Transfer transfer = 4;
        Stake stake = 5;
        AddKey add_key = 6;
        DeleteKey delete_key = 7;
        DeleteAccount delete_account = 8;
    }
}

message Transaction {
    string signer_id = 1;
    PublicKey public_key = 2;
    uint64 nonce = 3;
    string receiver_id = 4;

    repeated Action actions = 5;
}

message SignedTransaction {
    bytes signature = 1;

    Transaction transaction = 2;
}

```

**receipt.proto**
```proto
syntax = "proto3";

import "public_key.proto";
import "signed_transaction.proto";
import "uint128.proto";
import "wrappers.proto";

message DataReceipt {
    bytes data_id = 1;
    google.protobuf.BytesValue data = 2;
}

message ActionReceipt {
    message DataReceiver {
        bytes data_id = 1;
        string receiver_id = 2;
    }

    string signer_id = 1;
    PublicKey signer_public_key = 2;

    // The price of gas is determined when the original SignedTransaction is
    // converted into the Receipt. It's used for refunds.
    Uint128 gas_price = 3;

    // List of data receivers where to route the output data
    // (e.g. result of execution)
    repeated DataReceiver output_data_receivers = 4;

    // Ordered list of data ID to provide as input results.
    repeated bytes input_data_ids = 5;

    repeated Action actions = 6;
}

message Receipt {
    string predecessor_id = 1;
    string receiver_id = 2;
    bytes receipt_id = 3;

    oneof receipt {
        ActionReceipt action = 4;
        DataReceipt data = 5;
    }
}

```

### Validation and Permissions

To validate `SignedTransaction` we need to do the following:
- verify transaction hash against signature and the given public key
- verify `signed_id` is a valid account ID
- verify `receiver_id` is a valid account ID
- fetch account for the given `signed_id`
- fetch access key for the given `signed_id` and `public_key`
- verify access key `nonce`
- get the current price of gas
- compute total required balance for the transaction, including action fees (in gas), deposits and prepaid gas.
- verify account balance is larger than required balance.
- verify actions are allowed by the access key permissions, e.g. if the access key only allows function call, then need to verify receiver, method name and allowance.

Before we convert a `Transaction` to a new `ActionReceipt`, we don't need to validate permissions of the actions or their order. It's checked during `ActionReceipt` execution.

`ActionReceipt` doesn't need to be validated before we start executing it.
The actions in the `ActionReceipt` are executed in given order.
Each action has to check for the validity before execution.

Since `CreateAccount` gives permissions to perform actions on the new account, like it's your account, we introduce temporary variable `actor_id`.
At the beginning of the execution `actor_id` is set to the value of `predecessor_id`. 

Validation rules for actions:
- `CreateAccount`
  - check the account `receiver_id` doesn't exist
- `DeployContract`, `Stake`, `AddKey`, `DeleteKey`
  - check the account `receiver_id` exists
  - check `actor_id` equals to `receiver_id`
- `FunctionCall`, `Transfer`
  - check the account `receiver_id` exists

When `CreateAccount` completes, the `actor_id` changes to `receiver_id`.
NOTE: When we implement `DeleteAccount` action, its completion will change `actor_id` back to `predecessor_id`.

Once validated, each action might still do some additional checks, e.g. `FunctionCall` might check that the code exists and `method_name` is valid.

### `DataReceipt` generation rules

If `ActionReceipt` doesn't have `output_data_id` and `output_receiver_id`, then `DataReceipt` is not generated.
Otherwise, `DataReceipt` depends on the last action of `ActionReceipt`. There are 4 different outcomes:

1. Last action is invalid, failed or the execution stopped on some previous action.
    - `DataReceipt` is generated
    - `data_id` is set to the value of `output_data_id` from the `ActionReceipt`
    - `success` is set to `false`
    - `data` is set to `null`
2. Last action is valid and finished successfully, but it's not a `FunctionCall`. Or a `FunctionCall`, that returned no value.
    - `DataReceipt` is generated
    - `data_id` is set to the value of `output_data_id` from the `ActionReceipt`
    - `success` is set to `true`
    - `data` is set to `null`
3. Last action is `FunctionCall`, and the result of the execution is some value.
    - `DataReceipt` is generated
    - `data_id` is set to the value of `output_data_id` from the `ActionReceipt`
    - `success` is set to `true`
    - `data` is set to the bytes of the returned value
4. Last action is `FunctionCall`, and the result of the execution is a promise ID
    - `DataReceipt` is NOT generated, because we don't have the value for the execution.
    - Instead we should modify the `ActionReceipt` generated for the returned promise ID.
    - In this receipt the `output_data_id` should be set to the `output_data_id` of the action receipt that we just finished executed.
    - `output_receiver_id` is set the same way as `output_data_id` described above.

#### Example for the case #4

A user called contract `a.app`, which called `b.app` and expect a callback to `a.app`. So `a.app` generated 2 receipts:
Towards `b.app`:
```
...
"receiver_id": "b.app",
...
"output_data_id": "data_a",
"output_receiver_id": "a.app",

"input_data_id": [],
...
```
Towards itself:
```
...
"receiver_id": "a.app",
...
"output_data_id": "null",
"output_receiver_id": "null",

"input_data_id": ["data_a"],
...
```

Now let's say `b.app` doesn't actually do the work, but it's just a middleman that charges some fees before redirecting the work to the actual contract `c.app`.
In this case `b.app` creates a new promise by calling `c.app` and returns it instead of data.
This triggers the case #4, so it doesn't generate the data receipt yet, instead it creates an action receipt which would look like that:
```
...
"receiver_id": "c.app",
...
"output_data_id": "data_a",
"output_receiver_id": "a.app",

"input_data_id": [],
...
```
Once it completes, it would send a data receipt to `a.app` (unless `c.app` is a middleman as well). 

But let's say `b.app` doesn't want to reveal it's a middleman.
In this case it would call `c.app`, but instead of returning data directly to `a.app`, `b.app` wants to wrap the result into some nice wrapper.
Then instead of returning the promise to `c.app`, `b.app` would attach a callback to itself and return the promise ID of that callback. Here is how it would look: 
Towards `c.app`:
```
...
"receiver_id": "c.app",
...
"output_data_id": "data_b",
"output_receiver_id": "b.app",

"input_data_id": [],
...
```

So when the callback receipt first generated, it looks like this:
```
...
"receiver_id": "b.app",
...
"output_data_id": "null",
"output_receiver_id": "null",

"input_data_id": ["data_b"],
...
```
But once, its promise ID is returned with `promise_return`, it is updated to return data towards `a.app`:
```
...
"receiver_id": "b.app",
...
"output_data_id": "data_a",
"output_receiver_id": "a.app",

"input_data_id": ["data_b"],
...
```

### Data storage

We should maintain the following persistent maps per account (`receiver_id`)
- Received data: `data_id -> (success, data)`
- Postponed receipts: `receipt_id -> Receipt` 
- Pending input data: `data_id -> receipt_id`

When `ActionReceipt` is received, the runtime iterates through the list of `input_data_id`.
If `input_data_id` is not present in the received data map, then a pair `(input_data_id, receipt_id)` is added to pending input data map and the receipt marked as postponed.
At the end of the iteration if the receipt is marked as postponed, then it's added to map of postponed receipts keyed by `receipt_id`.
If all `input_data_id`s are available in the received data, then `ActionReceipt` is executed.

When `DataReceipt` is received, a pair `(data_id, (success, data))` is added to the received data map.
Then the runtime checks if `data_id` is present in the pending input data.
If it's present, then `data_id` is removed from the pending input data and the corresponding `ActionReceipt` is checked again (see above).

NOTE: we can optimize by not storing `data_id` in the received data map when the pending input data is present and it was the final input data item in the receipt.

When `ActionReceipt` is executed, the runtime deletes all `input_data_id` from the received data map.
The `receipt_id` is deleted from the postponed receipts map (if present).  

### TODO Receipt execution

- input data is available to all function calls in the batched actions
- TODODO

# Future possibilities
[future-possibilities]: #future-possibilities

- We can add `or` based data selector, so data storage can be affected. 
