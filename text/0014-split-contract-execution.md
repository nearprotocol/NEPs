- Proposal Code Name: `split_contract_execution`
- Start Date: 2019-09-13
- NEP PR: [nearprotocol/neps#00014](https://github.com/nearprotocol/NEPs/pull/14)

# Summary
[summary]: #summary

Split contract execution is a convenience tool, a framework built on the top of near-bindgen that allows performing
very long and heavy computations that otherwise would not be possible to perform with a single function call because
either they take too much time/gas or require storing too much state.

For example, we want to do BLS signature aggregation of a very long list of signatures. Specifically, if we
attempt such execution the contract will exceed the maximum gas allowed to be burnt per contract function call.
Also, even if we introduce a host-side function (or precompile) that does aggregation we cannot directly feed it
BLS signatures because the computation will slowdown the block production significantly.

If such computation can be split into logical steps where at the end of step X we save the state and then start from it
at step X+1 then we can split such execution across multiple contract calls. In the above example we can add one BLS
signature to the aggregate at each step. When smart contract finishes aggregating signature X, it saves the current
aggregate into its state and calls itself using promise API to aggregate the next signature X+1. When it reaches the
final signature it stops calling itself.

Similarly, when contract execution requires too much space we can partition it across multiple contracts (in expectation
that they will end up on different shards). For instance ethash computation requires storing a very large graph (~1GiB)
which we can split across multiple contract instances that perform their part of verification in parallel before calling
the "aggregator" to call the result, i.e. MapReduce.

## Non-distributed execution

For non-distributed execution we can provide nice convenience functions to the contract developer using near-bindgen.

Here is an example of BLS aggregator:

```rust
pub fn aggregate_bls(&mut self, signatures: Vec<BLS>) -> Aggregate {
    for signature in signatures.split_execution() {
        self.aggregate.aggregate(signature);
    }
    self.aggregate
}
```
1. When this method is called the first time the loop performs only one iteration, saves the state and invokes a promise
that calls itself;
2. When receipt of the promise calls this method second time it performs one more iteration, this time using the second
element and issues the promise to itself again. Step (2) is then performed until the collection is exhausted;
3. When collection is exhausted the execution stops and the result is propagated using `promise_then` such that from
RPC standpoint it looks like the contract call returned that aggregated result.


## Distributed execution

For general use-cases we can implement a framework that allows doing MapReduce. We can decide implementation details
once we have a specific use case.

For specialized cases, like ethash computation we can implement distributed computation without the framework.

## Alternatives

I was not able to find the alternative approaches by other blockchains. In most cases they solve expensive computation
by either making a fast host-function or a precompile, which does not help if computation is by itself very slow
and heavy.

### Preserving the entire Wasm state

It is possible to stop and resume Wasm https://kripken.github.io/blog/wasm/2019/07/16/asyncify.html which could be
used to perform split execution. We can even first introduce blocking contract calls as an alternative to asynchronous
contract calls. With blocking calls a contract stop execution while it waits for another contract to finish execution.
Blocking contract calls then would naturally enable split contract execution.

