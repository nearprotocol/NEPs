# Proposal for multi-fungible-token
In Ref Finance, we handle all swap pools in one contract, which means, when we release LP token for each pool, those tokens are exist in single contract, which can NOT be supported by current NEP-141 standard.

Now, based on NEP-141, actually we extended it to support multi-token in one contract, changed some interfaces and leave the whole working process untouched.   
Here is the way:
we change the following interface, add a heading 'm' to the name and, compare to the origin one in NEP-141, `token_id: String` is added as first parameter of each interface:
* mft_resolve_transfer in trait MFTTokenResolver 
* mft_on_transfer in trait MFTTokenReceiver 
* mft_transfer
* mft_transfer_call
* mft_metadata
* mft_register
* mft_total_supply
* mft_balance_of

When the value of this field equals to `predecessor_id` or the token contract id, the interface has the same behaviors with original one in NEP-141. Otherwise, the token_id stands for inner-index in the token contract.


For detailed implementation, please see [here](https://github.com/ref-finance/ref-contracts/blob/main/ref-exchange/src/multi_fungible_token.rs)