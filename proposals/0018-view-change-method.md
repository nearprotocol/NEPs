- Proposal Name: Improve view/change methods in contracts
- Start Date: 2019-09-26
- NEP PR: [nearprotocol/neps#0000](https://github.com/nearprotocol/neps/pull/18)

# Summary
[summary]: #summary

Currently the separation between view methods and change methods on the contract level is not very well defined and causes
quite a bit of confusion among developers. We propose in the NEP to elucidate the difference between view methods
and change methods and how they should be used. In short, we would like to restrict view methods from accessing certain
context variables and do not distinguish between view and change methods on the contract level. Developers have the option
to differentiate between the two in frontend or through near-shell.

# Motivation
[motivation]: #motivation

From the feedback we received it seems that developers are confused by the results they get from view calls, which are
mainly caused by the fact that some binding methods such as `signer_account_id`, `current_account_id`, `attached_deposit`
do not make sense in a view call. 
To avoid such confusion and create better developer experience, it is better if those context variables
are prohibited in view calls.

# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

Among binding methods that we expose from nearcore, some do make sense in a view call, such as `block_index`,
while the majority does not. 
Here we explicitly list the methods are not allowed in a view call and, in case they are invoked, the contract will panic with
`<method_name> is not allowed in view calls`.

The following methods are prohibited:
  * `signer_account_id`
  * `signer_account_pk`
  * `predecessor_account_id`
  * `attached_deposit`
  * `prepaid_gas`
  * `used_gas`
  * `promise_create`
  * `promise_then`
  * `promise_and`
  * `promise_batch_create`
  * `promise_batch_then`
  * `promise_batch_action_create_account`
  * `promise_batch_action_deploy_account`
  * `promise_batch_action_function_call`
  * `promise_batch_action_transfer`
  * `promise_batch_action_stake`
  * `promise_batch_action_add_key_with_full_access`
  * `promise_batch_action_add_key_with_function_call`
  * `promise_batch_action_delete_key`
  * `promise_batch_action_delete_account`
  * `promise_results_count`
  * `promise_result`
  * `promise_return`

From the developer perspective, if they want to call view functions from command line on some contract, they would just
call `near view <contractName> <methodName> [args]`. If they are building an app and want to call a view function from the
frontend, they should follow the same pattern as we have right now, specifying `viewMethods` and `changeMethods` in
`loadContract`.

# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

To implement this NEP, we need to change how binding methods are handled in runtime. More specifically, we can rename
`free_of_charge` to `is_view` and use that to indicate whether we are processing a view call. In addition we can add
 a variant `ProhibitedInView(String)` to `HostError` so that if `is_view` is true,
then all the access to the prohibited
methods will error with `HostError::ProhibitedInView(<method_name>)`.

# Drawbacks
[drawbacks]: #drawbacks

In terms of not allowing context variables, I don't see any drawback as those variables do not have a proper meaning
in view functions. For alternatives, see the section below.

# Rationale and alternatives
[rationale-and-alternatives]: #rationale-and-alternatives

This design is very simple and requires very little change to the existing infrastructure. An alternative solution is
to distinguish between view methods and change methods on the contract level. One way to do it is through decorators, as
described [here](https://github.com/nearprotocol/NEPs/pull/3). However, enforcing such distinction on the contract level
requires much more work and is not currently feasible for Rust contracts. 

# Unresolved questions
[unresolved-questions]: #unresolved-questions

# Future possibilities
[future-possibilities]: #future-possibilities


