# Smart contract upgradability

In NEAR smart contracts are upgradable in-place by default.
This means that given a specific `account-id` - the contract code may change depending on various rules.

## Non-upgradable

There are many contracts that should not be upgradable.

There are two ways that contract code can be upgraded in-place: using an access key and contract re-deploying it's own code.

To determine if contract is not upgradable, need to check that contract doesn't have any full access keys and that contract's code doesn't contain any `DeployContract` calls to itself.

Additionally, contract code can contain ability to add FullAccessKeys to itself, and in this way return back control to the facilitator (for example multisig and lockup contracts have this ability under certain circumstances).

## Accessible contract

Accessible contracts are contracts that have access keys with ability to re-deploy new code on them.

Usually this will be used for development or in some highly permissioned settings.

For example, contracts like multisig is really owned by the person or people who control it and hence they can re-deploy the code on it to upgrade or remove multisig.

## Owner-pattern

Owner pattern means that contract's upgradability is owned by a different account.

This allows to introduce a more sophisticated management, anything from multisig to DAO. This also allows to easily evolve the management over time. For example, starting from simple account, one can change owner later to multisig and later upgrade to a DAO.

Owner upgradable pattern introduces next set of methods on a contract that follows this standard:

```
    /// Returns current owner account.
    get_owner() -> AccountId

    /// Sets new owner. If owner is set to `system` this contract is considered to be non-upgradable without hard forks.
    /// Can only be called by owner.
    set_owner(owner: AccountId)

    /// Returns how long between staging code and deploy can be called in nanoseconds.
    /// Usually staging duration is set at deploy. Good defaults are 1 and 7 days.
    get_staging_duration() -> WrappedDuration

    /// Stages new contract on the given account. 
    /// Can only be called by owner.
    /// This means that contract code is ready to be activated after specific staging time passes.
    /// Caller is responsible for attaching extra balance to cover storage.
    stage_code(code: bytes)

    /// Deploys staged code. 
    /// Can only be called by owner.
    /// If there is no staged code, will fail.
    deploy_code()
```

TBD: do we want to have a way to change staging duration?

## State migrations

This section is TBD on state migration patters
