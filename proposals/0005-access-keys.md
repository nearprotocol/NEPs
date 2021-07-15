- Proposal Code Name: access_keys
- Start Date: 2019-07-08
- NEP PR: [nearprotocol/neps#0000](https://github.com/nearprotocol/neps/pull/0000)
- Issue(s): [nearprotocol/nearcore#687](https://github.com/nearprotocol/nearcore/issues/687)

# Summary
[summary]: #summary

Access keys provide limited access to an account.
Each access key belongs to some account and identified by a unique (within the account) public key.
One account may have large number of access keys.
Access keys will replace original account-level public keys.
Access keys allow to act on behalf of the account by restricting allowed transactions with the access key permissions.

# Motivation
[motivation]: #motivation

Access keys give an ability to use dApps in a secure way without asking the user to sign every transaction in the wallet.
By issuing the access key once for the application, the application can now act on behalf of the user in a restricted environment.
This enables seamless experience for the user.

Access keys also enable a few other use-cases that are discussed in details below.

# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

Here are proposed changes for the AccessKey and Account structs.  

```rust
/// `account_id,public_key` is a key in the state
struct AccessKey {
  /// The nonce for this access key.
  /// It makes sense for nonce to not start from 0, in case the access key is recreated
  /// with the same public key, to avoid replaying of old transactions.
  pub nonce: Nonce,  // u64 
  
  /// Defines permissions for the AccessKey 
  pub permission: AccessKeyPermission,
}

/// Defines permissions for AccessKey 
pub enum AccessKeyPermission {
  /// Restricts AccessKey to only be used for function calls.
  FunctionCall(FunctionCallPermission),

  /// Gives full access to the account.
  /// NOTE: It's used to replace account-level public keys.
  FullAccess,
}

pub struct FunctionCallPermission {
  /// `Some` amount that can be spent for transaction fees by this access key from the account balance.
  /// When used, both account balance and the allowance is decreased.
  /// To change or increase the allowance, the access key can be replaced using SwapKey.
  /// NOTE: If you reuse the public key, make sure to keep the nonce from the old AccessKey.
  /// `None` means unlimited allowance.
  pub allowance: Option<Balance>,  // u128

  /// The AccountID of the receiver of the transaction. The access key will restrict transactions to
  /// only this receiver.
  pub receiver_id: AccountId,  // String
  
  /// If `Some`, the access key would be restricted to calling only the given method name.
  /// `None` means it's restricted to calling the receiver_id contract, but any method name.   
  pub method_name: Option<String>,
}

/// NOTE: This change removes account-level nonce and public keys.
/// Key is `account_id`
struct Account {
  pub balance: Balance(u128),
  pub code_hash: Hash,
  /// Storage usage accounts for all access keys      
  pub storage_usage: StorageUsage(u64),
    /// Last block index at which the storage was paid for.
  pub storage_paid_at: BlockIndex(u64),
}
```

### Examples

#### AccessKey as account-level public key

If an AccessKey has the full access to the account and the allowance set to be the max value for u128, then
it essentially acts as an account-level public key. Which means we can remove account-level
public keys from the account struct and rely only on access keys.

An access key example from user `vasya.near` with full access:
```rust
/// vasya.near,a123bca2
AccessKey {
    nonce: 0,
    
    permission: AccessKeyPermission::FullAccess,
}
```


#### AccessKey for a dApp by a user

This is a simple example where a user wants to use some dApp. The user has to authorize this dApp within their wallet, so the dApp knows who the user is, and also can issue simple function call transactions on behalf of this user.

To create such AccessKey a dApp generates a new key pair and passes the new public key to the user's wallet in a URL.
Then the wallet asks the user to create a new AccessKey with that points to the dApp.
User has to explicitly confirm this in the wallet for AccessKey to be created.

The new access key is restricted to be only used for the app’s contract_id, but is not restricted for any method name.
The user also selects the allowance to some reasonable amount, enough for the application to issue regular transactions.
The application might also hint the user about this desired allowance in some way.

Now the app can issue function call transactions on behalf of the user’s account towards the app’s contract without requiring the user to sign each transaction.

An access key example for chess app from user `vasya.near`:
```rust
/// vasya.near,c5d312f3
AccessKey {
    nonce: 0,
    
    permission: AccessKeyPermission::FunctionCall(FunctionCallPermission {
        // Since the access key is stored on the Chess app front-end, the user has
        // limited the spending amount to some reasonable, but large enough number.
        // NOTE: It's needs to be multiplied to decimals, e.g. 10^-18 
        allowance: Some(1_000_000_000),
        
        // This access key restricts access to `chess.app` contract.
        receiver_id: "chess.app",

        // Any method name on the `chess.app` contract can be called.  
        method_name: None,
    }),
}
```

#### AccessKey issued by a dApp

This is an example where the dApp wants to pay for the user, or it doesn't want to go through the user's sign-in flow.
For whatever reason the dApp decided to issue an access key directly for their account.

For this to work there should be one account with funds (that dApp controls on the backend) which creates access keys for the users.
The difference from the example above is there is only one account (the same for all users) that creates multiple access keys (one per user) towards one other contract (app's contract).
To differentiate users the contract has to use the public key of the access key instead of sender's account ID.

If the access key wants to support user's identity from the account ID. The contract can provide a public method that links user's account ID with a given public key.
Once this is done, a user can request a new access key with the linked public key (sponsored by the app), but it is linked to the user's account ID.

There are some caveats with this approach:
- The dApp is required to have a backend and to have some sybil resistance for users. It's needed to prevent abuse by bots.
- Writing the contract is slightly more complicated, since the contract now needs to handle mapping of the public keys to the account IDs.

An access key example for chess app paid by the chess app from `chess.funds` account:
```rust
/// chess.funds,2bc2b3b
AccessKey {
    nonce: 0,
    
    permission: AccessKeyPermission::FunctionCall(FunctionCallPermission {
        // Since the access key is given to the user, the developer wants to limit the
        // the spending amount to some conservative number, since a user might try to drain it.
        allowance: Some(5_000_000),
        
        // This access key restricts access to `chess.app` contract.
        receiver_id: "chess.app",

        // Any method name on the `chess.app` contract can be called (but some methods might just ignore this key).
        method_name: None,
    }),
}
```

#### AccessKey through a proxy

This examples demonstrates how to have more granular control on top of built-in access key restrictions.

Let's say a user wants to:
- limit the number of calls the access key can make per minute
- support multiple contracts with the same access key
- select which methods name can be called and which can't
- transfer funds from the account up to a certain limit 
- stake from the account, but prevent withdrawing funds


To make it work, we need to have a custom logic at every call.
We can achieve this by running a portion of a smart contract code before any action.
A user can deploy a code on their account and restrict access key to their account and to a method name, e.g. `proxy`.
Now this access key will only be able to issue transactions on behalf of the user that goes to the user's contract code and calls method `proxy`.
The `proxy` method can find out which access key is used by comparing public keys and verify the request before executing it.

E.g. the access key should only be able to call `chess.app` at most 3 times per 20 block and can transfer at most 1M tokens to the `chess.app`.
The `proxy` function internally can validate that this access key is used, fetch its config, validate the passed arguments and proxy the transaction.
A `proxy` method might take the following arguments for a function call:
```json
{
  "action": "call",
  "contractId": "chess.app",
  "methodName": "move",
  "args": "{...serialized args...}",
  "amount": 0
}
```

In this case the `action` is `call`, so the function checks the `amount` to be within the withdrawal limit, check that the contract name is `chess.app` and if there were the last 3 calls were not in the last 20 blocks issue an async call to the `chess.app`.
The same `proxy` function in theory can handle other actions, e.g. staking or vesting.

The benefit of having a proxy function on your own account is that it doesn't require additional receipt, because the account's state and the code are available at the transaction verification time.

An example of an access key limited to `proxy` function:
```rust
/// vasya.near,3bc2b3b
AccessKey {
    nonce: 0,
    
    permission: AccessKeyPermission::FunctionCall(FunctionCallPermission {
        // Allowance can be large enough, since the user is likely trusting the app.
        allowance: Some(1_000_000_000),
        
        // This access key restricts access to user's account `vasya.near` contract.
        // Most likely, the contract code can be deployed and upgraded directly from the wallet.
        receiver_id: "vasya.near",

        // The method is restricted to proxy, which does all the security checks.
        method_name: Some("proxy"),
    }),
}
```

# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

- Access keys are stored with the `account_id,public_key` key. Where `account_id` and `public_key` are actual Account ID and public keys, and `,` is a separator.
They should be stored on the same shard as the account.
- Access keys storage rent should be accounted and paid from the account directly without affecting the allowance.
- Access keys allowance can exceed the account balance.
- To validate a transaction signed with the AccessKey, we need to first validate the signature, then fetch the Account and the AccessKey, validate that we have enough funds and verify permissions. 
- Account creation should now create a full access permission access key, instead of public keys within the account.
- SwapKey transaction should just replace the old access key with the given new access key.

### Technical changes

#### `nonce` on the AccessKey level instead of account level

Since access keys can be used by the different people or parties at the same time, we need to be able to 
have separate nonce for each key instead of a single nonce at the account level.
With a single nonce on the account level, there is a high probability that 2 apps would use the same nonce for 2 different transactions and one of this transactions would be rejected.

Previously we were ordering transactions by nonce and rejecting transactions with a duplicated or lower nonce.
With the access key nonce, we still need to order transactions by nonce, but now we need to group them by `account_id,public_key` key instead of just account_id.
To prevent one access key from having a priority on other access keys, we should order transactions by hash when determining which transactions should be added to the block.

The suggestion from @nearmax:

"
We need to spec out here how transactions from different access keys are going to be ordered with respect to each other. For example:
3 access keys (A,B,C) issue 3 transactions each:
A1, A2, A3; B1,B2,B3; C1, C2, C3;
All these transactions operate on the same state so they need to have an order. First transaction to execute is one of {A1,B1,C1} that has lowest hash, let's say it is B1. Second transaction to execute is one of {A1,B2,C1} with lowest hash, etc.
"

We should also restrict the nonce of the next transaction to be exactly the previous nonce incremented by 1.
It will help us with ordering transactions.

The transaction ordering should be a separate topic which should also include security for transactions expiration and fork selection.

#### `allowance` field

Allowance is the amount of tokens the AccessKey can spend from the account balance.
When some amount is spent, it's subtracted from both the allowance of the access key and from the account balance.
If in some case the user wants to have unlimited allowance for this key, then we have a `None` allowance option.

NOTE: In the previous iteration of access keys, we used balance instead of the allowance.
But it required to sum up all access keys balances to get the the total account balance.
It also prevented sharing of the account balance between access keys.

#### Permissions

Almost all desired use-cases of access keys can be achieved by using the old permissions model.
It restricts access keys to only issue function call with no attached tokens.
The function calls are restricted to the selected `receiver_id` and potentially restricted to a single `method_name`.
Anything non-trivial can be done by the contract that receives this call, e.g. through `proxy` function.

To remove public keys from the account, we added a new permission that full access to the account and not limited by the allowance.

#### How is `storage_usage` computed?

If we use protobuf size to compute the `storage_usage` value, then protobuf might compress `u128` value and it would affect storage usage every time the `allowance` is modified.

The best option would be is to change `storage_usage` only when the access key is created or removed.
So that changes to the `allowance` value shouldn't change the `storage_usage` value.
For this to work, we might need to update the storage computation formula for the access key, e.g. the one that ignores the compressed size of the `allowance` and instead just relies on the 16 bytes of `u128` size.
Especially, because we currently don't use the proto size for the storage_usage for the account itself.

# Drawbacks
[drawbacks]: #drawbacks

Currently the permission model is quite limited to either a function call with one or any method names, or a full access key.
But we may add more permissions in the future in order to handle this issue. 

# Rationale and alternatives
[rationale-and-alternatives]: #rationale-and-alternatives

## Alternatives

#### More permissions directly on the access key

For example we can have multiple method names, multiple contract_id/method_name pairs or different transactions types (e.g. only allow staking transactions).

This can be achieved with the contract and a dedicated function that does this control. So to keep the runtime simple and secure we should avoid doing more checks, since they are not accounted for fees.

It's also can be achieved if we refactor SignedTransaction to only use method_names instead of oneof body types.

#### Balance instead of allowance

Allowance enables sharing of a single account balance with multiple access keys. E.g. if you use 5 apps, you can give full allowance to each app instead of splitting balance into 5 parts.

It's also easier to work with, than access keys balances.

Previously we have AccessKey's balance owner, so the dApp could sponsor users. But it can be achieved by dApps creating access keys from their account, effectively paying for all transactions.
 
#### Not exposing `nonce` on each AccessKey

If you use 2 applications at the same time, e.g. a mobile app and a desktop wallet, you might run into a `nonce` collision at the account level, which would cancel one of the transaction. It would happen more frequently with more apps being used.

As for the runtime multi nonce handling per account, we need to think and verify security a little more.

#### `receiver_id` being an `Option<AccountId>`

In the previous design, the `receiver_id` was called `contract_id` and was an option field. But it didn't remove the requirement for the receiver when it was `None`. Instead it was assuming the access key is pointed to the owner's account. 
We can potentially use `None` to mean unlimited key, and require user to explicitly specify their own account_id if they want to use proxy function.

# Unresolved questions
[unresolved-questions]: #unresolved-questions

#### Transactions ordering and nonce restrictions

That question is still unresolved. Whether we should restrict TX nonce to be +1 or not restricting.
It's not a blocking change, but it would make sense to do this change with other SignedTransaction security features such as minimum hash of a block header and block expiration.

#### Permissions

Not clear whether a single pair of `receiver_id`/`method_name` is enough to cover all use-cases at the moment.
E.g. if I want to use my account that already has some code on it, e.g. vesting locked account. I can't deploy a new code on it, so I can't use a `proxy` method.

# Future possibilities
[future-possibilities]: #future-possibilities

For all use-cases to work we need to add all missing runtime methods that are currently only possible with `SignedTransaction` at the moment, e.g. staking, account creation, public/access key management and code deployment.

Next we might consider refactoring stake out of `Account` and also refactor `SignedTransaction` to support text based method names instead of enums.

We should also think about storing the same code (by hash) only once instead of storing for each account. Especially, if we adopt `proxy` model. 
