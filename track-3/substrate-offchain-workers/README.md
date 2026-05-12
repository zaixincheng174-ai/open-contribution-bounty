# Implementing Off-Chain Workers in Substrate

Substrate off-chain workers let a runtime trigger node-local work after a block
is imported. They are useful when a chain needs data fetching, heavy
computation, indexing, or background coordination that should not happen inside
normal block execution.

The important mental model is simple: consensus logic must stay deterministic,
but not every useful task is deterministic or cheap enough to run on-chain.
Off-chain workers provide a bridge between those worlds. The runtime can define
when off-chain logic runs, while each node performs the actual work outside the
state transition function.

This article explains what off-chain workers are, where they fit in a
Substrate chain, how they submit results back on-chain, and what builders should
watch for when designing production-ready workers.

## Why off-chain workers exist

Every validator and full node must agree on block validity. That means normal
runtime execution cannot depend on unpredictable external systems such as HTTP
APIs, wall-clock network latency, local files, or random node-specific state.
If block execution directly fetched a web API, one node might receive a
different response than another node, and consensus would break.

At the same time, blockchains often need information or computation from
outside deterministic execution:

- Price feeds and oracle data.
- Batch processing for expensive calculations.
- Chain-specific indexing and derived data.
- Periodic monitoring or automation.
- Signed transaction generation based on local observations.
- External service calls that should not block block production.

Off-chain workers solve this by moving that work outside block validation. They
run after block import on nodes that execute the relevant runtime code. If the
worker needs to affect chain state, it must submit a transaction that the
runtime later validates like any other extrinsic.

## Where off-chain workers run

An off-chain worker is defined in a pallet with the `offchain_worker` hook.
When a node imports a block, the runtime can call that hook for the pallet.
The hook receives the block number and can start off-chain logic.

That logic runs in the node's off-chain execution environment, not inside the
deterministic state transition. It can use off-chain APIs exposed by the host,
including HTTP requests, local off-chain storage, timestamp access, and
transaction submission helpers.

This distinction matters:

- The worker can read on-chain state through runtime APIs.
- The worker can perform node-local or network-dependent work.
- The worker cannot directly mutate on-chain storage.
- The worker can submit extrinsics that may later mutate on-chain storage if
  accepted by the runtime.

Treat off-chain workers as producers of candidate information, not as direct
state writers.

## The basic implementation flow

A typical off-chain worker design has four parts.

First, the pallet defines the `offchain_worker` hook. This is where the worker
decides whether to run on the current block. Many workers should avoid running
on every block. A modulo schedule, a storage-based deadline, or a lock can
prevent duplicated work.

Second, the worker fetches or computes data. This may involve an HTTP call,
local storage lookup, cryptographic check, or off-chain indexing read. The
worker should keep timeouts short and handle failure as normal operation.
External APIs fail, nodes restart, and network access may be unavailable.

Third, the worker prepares a transaction if the result must become part of
chain state. Substrate supports signed transactions, unsigned transactions, and
unsigned transactions with signed payloads. The right choice depends on the
trust model.

Fourth, the runtime validates the submitted transaction. This is the real
security boundary. The runtime must check signatures, freshness, replay
protection, bounds, origin, and data validity before writing anything on-chain.

## Signed transactions

Signed transactions are the safest default when a worker acts on behalf of a
known account. The off-chain worker uses a local key and submits an extrinsic
with a normal account signature. The runtime can rely on the usual transaction
authentication path, fees, nonces, and origin checks.

Use signed transactions when:

- The worker's action should be attributable to a specific account.
- The chain wants normal fee and nonce handling.
- The operation should not be accepted from arbitrary nodes.
- The account is expected to manage keys and balances.

The tradeoff is operational. Nodes need access to the signing key. In a
validator or production node environment, key management and permissions must
be treated carefully.

## Unsigned transactions

Unsigned transactions can be useful when the chain wants any node to submit
data without requiring a funded account. They are also dangerous if validation
is weak. An unsigned transaction has no normal account signature, so the
runtime must provide strong validation logic to prevent spam, replay, and false
data.

Use unsigned transactions only when the pallet can validate them independently.
For example, the transaction might include a signed payload from an authorized
key, a block-number freshness bound, a unique key in storage, or a proof that
the submitted value is acceptable.

The `ValidateUnsigned` implementation is critical. It should define:

- Which calls may be submitted unsigned.
- How stale submissions are rejected.
- How duplicate submissions are handled.
- What priority and longevity the transaction has.
- Whether propagation to other peers is allowed.

Weak unsigned validation can turn an off-chain worker feature into a network
spam vector.

## Local storage and off-chain indexing

Off-chain workers can use local off-chain storage for node-local state. This is
useful for caching external responses, recording when a worker last ran, or
coordinating repeated attempts.

Local storage is not consensus state. Different nodes may have different local
values, and that is expected. Do not use local off-chain storage as if it were
shared chain storage.

Substrate also has off-chain indexing support. Runtime execution can write
selected data into an off-chain database so that off-chain components can query
it later. This is useful for derived data or local indexing patterns, but the
same rule applies: indexed off-chain data is not a substitute for validated
on-chain state.

## HTTP requests and external data

HTTP is one of the common reasons to use off-chain workers. A worker may query
an API, parse a response, and submit a transaction with the result.

Production designs should assume the external service is unreliable or
adversarial:

- Set timeouts.
- Validate response format.
- Reject unexpected status codes.
- Bound response size.
- Avoid trusting a single centralized API for critical state.
- Include freshness checks in the submitted transaction.
- Consider requiring multiple independent submissions or aggregation.

For oracle-like data, the runtime should not blindly accept the first value
submitted by an off-chain worker. The worker is a transport mechanism; the
trust model still belongs in the pallet design.

## Concurrency and duplicate work

Many nodes may run the same off-chain worker at the same block. If every node
submits the same transaction, the network can receive duplicates. That is not a
bug in off-chain workers; it is a design problem the pallet should handle.

Common mitigation patterns include:

- A local lock so one node process does not run duplicate work concurrently.
- A block interval so the worker only runs periodically.
- A storage item that records the latest accepted result.
- Transaction validation that rejects stale or duplicate submissions.
- A deterministic selection rule for which account or authority should submit.

The runtime must remain correct even if duplicate or late submissions arrive.

## Testing strategy

Off-chain worker tests should cover both local worker behavior and runtime
transaction validation.

At a minimum, test:

- The worker scheduling rule.
- HTTP success and failure paths, using mocks.
- Signed transaction construction.
- Unsigned transaction validation, including bad signatures or stale payloads.
- Replay and duplicate rejection.
- Storage updates after the extrinsic is accepted.
- Behavior when local storage is empty or corrupted.

Do not stop at testing that the worker can make a request. The important
question is whether invalid or repeated data can change chain state.

## Operational checklist

Before shipping an off-chain worker, review these questions:

1. Does the worker need to run on every node, every authority, or only selected
   nodes?
2. How often does it run, and what prevents duplicate local execution?
3. What happens if the external API is down or slow?
4. Is any submitted transaction signed, unsigned, or unsigned with a signed
   payload?
5. How does the runtime reject stale, duplicate, or malicious submissions?
6. Are keys managed safely on production nodes?
7. Are response sizes, parsing, and timeouts bounded?
8. Is the worker observable through logs or metrics?
9. Does the chain still work if no off-chain worker submits data for a while?
10. Is there a manual recovery path if bad data is submitted?

## Common mistakes

The first mistake is treating off-chain work as trusted. It is not. Any value
that enters chain state through an extrinsic must be validated by deterministic
runtime logic.

The second mistake is weak unsigned transaction validation. If an attacker can
submit unlimited unsigned transactions, the chain has a spam problem.

The third mistake is depending on one external API without fallback or
freshness checks. Off-chain workers make external integration possible, but
they do not make external data reliable.

The fourth mistake is ignoring duplicate submissions. In a decentralized
network, many nodes can observe the same event and act at the same time.

The fifth mistake is putting too much work into the worker without timeouts,
locks, or observability. A worker that blocks forever or floods logs is still
an operational incident even though it does not directly block consensus.

## Summary

Off-chain workers are a practical Substrate tool for moving nondeterministic or
expensive tasks out of block execution. They can fetch data, compute results,
cache local state, and submit transactions back to the chain.

Their power comes with a strict boundary: off-chain workers do not directly
write consensus state. The runtime must validate anything they submit. Good
design means short timeouts, bounded data, clear signing rules, replay
protection, duplicate handling, and tests that prove bad submissions are
rejected.

Used well, off-chain workers let a chain integrate with the outside world
without sacrificing deterministic block validation. Used casually, they can
become a source of spam, stale data, operational fragility, or hidden trust
assumptions.

## References

- [FRAME Offchain Workers | Polkadot SDK Docs](https://paritytech.github.io/polkadot-sdk/master/polkadot_sdk_docs/reference_docs/frame_offchain_workers/index.html)
- [Offchain Worker Example Pallet | Polkadot SDK Docs](https://paritytech.github.io/polkadot-sdk/master/pallet_example_offchain_worker/index.html)
- [ValidateUnsigned | Polkadot SDK Docs](https://paritytech.github.io/polkadot-sdk/master/polkadot_service/runtime_traits/trait.ValidateUnsigned.html)
- [Offchain Host Functions | Polkadot SDK Docs](https://paritytech.github.io/polkadot-sdk/master/sp_io/offchain/index.html)
- [Offchain Index Host Functions | Polkadot SDK Docs](https://paritytech.github.io/polkadot-sdk/master/sp_io/offchain_index/index.html)
