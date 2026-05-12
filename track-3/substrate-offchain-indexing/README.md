# Substrate's Off-Chain Indexing Solution

Substrate chains often need data that is useful to nodes, workers, or local
services but too expensive or unnecessary to keep in consensus storage.
Off-chain indexing is one tool for that problem. It lets runtime execution
write selected key-value data into a node's local Offchain DB while the block is
processed.

This article explains what off-chain indexing is, how it differs from on-chain
storage and off-chain workers, how to design indexing keys, and what risks
builders should review before using it in a production chain.

## The problem: not all useful data belongs on-chain

On-chain storage is expensive because every full node must execute, verify, and
retain it according to the chain's state rules. It is the right place for data
that must be part of consensus: balances, ownership, governance state, staking
state, pallet configuration, and any value that block validity depends on.

Many useful values do not need that guarantee. A chain may want to make a large
payload available to local off-chain logic, keep derived data for later
processing, or pass contextual data from an extrinsic to a worker without
storing the full value in consensus state.

Writing all of that to on-chain storage can create avoidable cost and state
bloat. Off-chain indexing gives the runtime a way to emit local indexed data
without increasing the consensus state root.

## What off-chain indexing does

Off-chain indexing writes key-value pairs into a node-local off-chain database.
In Polkadot SDK, the low-level host interface is exposed through
`sp_io::offchain_index`. The runtime can call `set` to write a key-value pair
and `clear` to remove one.

The key distinction is that this data is not normal pallet storage. It is not
part of the on-chain state root, and smart contracts or runtime logic should
not depend on it for block validity. It is local indexed data produced while a
node processes blocks.

This makes off-chain indexing useful for:

- Passing large or detailed payloads to off-chain workers.
- Storing local copies of data whose hash or commitment is kept on-chain.
- Helping node-local services find data without scanning every event manually.
- Building chain-specific local caches.
- Reducing pressure to store bulky helper data in consensus storage.

It is not a replacement for a full external indexing platform such as a
database-backed indexer. It is lower level and node-local. Use it when local
runtime-driven writes are the right primitive.

## Off-chain indexing vs off-chain workers

Off-chain workers and off-chain indexing are related, but they solve different
problems.

An off-chain worker is code that runs outside deterministic block execution
after block import. It can make HTTP requests, use local storage, perform
computations, and submit transactions back to the chain.

Off-chain indexing is a runtime-to-node write path for local indexed data. It
can be called during runtime execution so that data associated with a block is
recorded in the node's Offchain DB.

The common pattern is:

1. An extrinsic carries data or a commitment.
2. Runtime logic verifies what must be verified for consensus.
3. The runtime writes selected local data through off-chain indexing.
4. An off-chain worker or local service later reads that local data and acts on
   it.

The on-chain part should still store whatever is needed for security. If the
raw payload matters for later verification, keep a hash, commitment, or bounded
summary on-chain so that local data can be checked.

## Designing indexing keys

Key design is the most important implementation detail. Off-chain indexing is
local storage, so careless key selection can cause overwrites or confusing
reads.

Good keys are explicit, namespaced, and unique enough for the data model. A
typical key should include:

- A pallet or feature prefix.
- A version marker if the encoding may change.
- A block number or block hash if the value is block-specific.
- A subject identifier such as account, asset, order id, or request id.
- A content hash when the value represents a large payload.

Avoid generic keys such as `latest` unless overwriting is the intended
behavior. Forks and replays can cause the same runtime logic to execute more
than once for competing blocks. If two executions write to the same key, local
data may be overwritten in ways that differ across nodes.

When in doubt, use keys that preserve enough context to distinguish block,
call, and payload identity.

## Runtime write sketch

The runtime write path is small, but the surrounding checks matter. The
following sketch shows the shape of an extrinsic that stores only a commitment
on-chain while writing the larger payload into the node-local Offchain DB.

This is an illustrative pallet fragment, not a complete drop-in module:

```rust
use codec::Encode;

fn document_key<T: frame_system::Config>(
    owner: &T::AccountId,
    block_number: T::BlockNumber,
    document_hash: T::Hash,
) -> Vec<u8> {
    (
        b"my_pallet",
        b"v1",
        b"document",
        owner,
        block_number,
        document_hash,
    )
        .encode()
}

pub fn submit_document(
    origin: OriginFor<T>,
    document: BoundedVec<u8, T::MaxDocumentBytes>,
    document_hash: T::Hash,
) -> DispatchResult {
    let owner = ensure_signed(origin)?;
    ensure!(T::Hashing::hash(&document) == document_hash, Error::<T>::BadHash);

    DocumentCommitments::<T>::insert(
        &owner,
        document_hash,
        frame_system::Pallet::<T>::block_number(),
    );

    let key = document_key::<T>(
        &owner,
        frame_system::Pallet::<T>::block_number(),
        document_hash,
    );
    sp_io::offchain_index::set(&key, &document.encode());

    Ok(())
}
```

The on-chain `DocumentCommitments` entry is the part that other consensus
logic can rely on. The indexed bytes are only helper data for a node, worker,
or local service that has off-chain indexing enabled and has processed the
block that wrote the value.

When testing this pattern, run at least one local node with off-chain indexing
enabled and verify that the reader:

- Derives the same key as the runtime.
- Decodes the expected version of the payload.
- Hashes the local bytes and compares them with the on-chain commitment.
- Handles a missing local value as a recoverable availability problem.
- Waits for finality if the downstream action must only use canonical data.

## Data encoding and versioning

Off-chain indexed values are bytes. The chain team must choose and document the
encoding. SCALE-encoded runtime types are common because they match the rest of
the Substrate ecosystem, but JSON or another format may be reasonable for a
local service boundary.

Version the format deliberately. A runtime upgrade can change types,
semantics, or key layouts. If existing local data may outlive the runtime that
wrote it, include a version byte or versioned key prefix so readers can decode
old and new values safely.

Also consider size. Off-chain indexing reduces on-chain state bloat, but it
does not make storage free. Large values still consume disk on every node that
has indexing enabled and processes the relevant blocks.

## Consistency and availability

Off-chain indexed data is local. It may only exist on nodes that have the
feature enabled and have processed the relevant blocks. A service that depends
on this data should be explicit about that operational requirement.

Do not assume every RPC node has the same Offchain DB. Do not assume a light
client can read it. Do not assume a newly synced node has historical local
indexed data unless it actually replayed the blocks with indexing enabled and
persisted the writes.

For critical data, store a commitment on-chain and treat off-chain indexed
payloads as locally available helper data. If a node lacks the helper data, it
should be able to resync, refetch from a trusted source, or degrade gracefully.

## Forks, reorgs, and replay

Runtime execution can happen for blocks that later do not become canonical.
That matters for local indexing because local writes may happen while a node
evaluates or imports competing branches.

Designs should avoid assuming that every indexed value corresponds to finalized
canonical history. Safer patterns include:

- Put the block hash or block number in the key.
- Include the extrinsic index or event subject when relevant.
- Store finality status separately if a downstream consumer needs final data.
- Let readers verify against chain state before taking irreversible action.
- Use on-chain commitments to validate local payloads.

If a local consumer needs finalized-only data, it should wait for finality or
check finality before acting.

## Security boundaries

Off-chain indexed data should be treated as local untrusted input when a
separate service reads it later. The runtime may have written it, but the local
database is not the chain's consensus state. A compromised node, corrupted disk,
manual operator change, or stale database can produce bad local reads.

Security-sensitive consumers should verify:

- The key namespace and version.
- The expected encoding.
- The content hash or commitment against on-chain state.
- The block hash or finality condition, if relevant.
- Size limits before decoding.

The runtime must not depend on a later local read to decide block validity.
Only consensus storage, block data, and deterministic runtime checks can do
that.

## Example architecture

Imagine a pallet that accepts a large document hash and a compact summary. The
full document is too large to store on-chain, but an off-chain worker needs it
for later processing.

A reasonable design could be:

1. The extrinsic includes the document bytes, document hash, and owner
   signature.
2. The runtime verifies the signature and stores only the hash and compact
   metadata on-chain.
3. The runtime writes the full document bytes to the Offchain DB under a key
   like `my_pallet:v1:document:<block_hash>:<document_hash>`.
4. The off-chain worker reads the local bytes, checks that the hash matches
   on-chain state, and performs the expensive processing.
5. If the worker needs to publish a result, it submits a normal transaction
   that the runtime validates.

This design keeps the consensus state small while preserving a verification
anchor for the local payload.

## Testing checklist

Off-chain indexing tests should cover more than the happy path.

Review:

- Key derivation for uniqueness and namespace separation.
- Encoding and decoding round trips.
- Runtime behavior when the same extrinsic executes more than once.
- Behavior across runtime version changes.
- Size limits for indexed payloads.
- Reader behavior when the local value is missing.
- Reader behavior when the local value does not match the on-chain commitment.
- Finality handling for consumers that require finalized data.

Testing should also include a local node run with off-chain indexing enabled,
because otherwise a developer may only test the runtime logic and miss the
node-level persistence behavior.

## When not to use it

Do not use off-chain indexing when the data must be available to all chain
participants through normal state queries. Use on-chain storage for consensus
data.

Do not use it as a replacement for public analytics infrastructure. If users
need rich historical queries, dashboards, joins, and aggregations, an external
indexer is often the right tool.

Do not use it to hide a trust assumption. If later logic needs to trust a value,
put the verification rule or commitment on-chain.

## Summary

Off-chain indexing is a useful Substrate primitive for writing local indexed
data from runtime execution into a node's Offchain DB. It can reduce state
bloat, support off-chain workers, and make local chain-specific services easier
to build.

The tradeoff is that the data is not consensus storage. Builders must design
good keys, version encodings, handle forks and finality, verify local data
against on-chain commitments, and test behavior with indexing actually enabled.

Used well, off-chain indexing is a practical bridge between deterministic
runtime execution and local off-chain processing. Used carelessly, it can create
stale caches, overwritten keys, hidden availability assumptions, or security
logic that depends on data the chain itself does not guarantee.

## References

- [sp_io::offchain_index | Polkadot SDK Docs](https://paritytech.github.io/polkadot-sdk/master/sp_io/offchain_index/index.html)
- [StorageChanges | Polkadot SDK Docs](https://paritytech.github.io/polkadot-sdk/master/sp_api/type.StorageChanges.html)
- [sp_state_machine | Polkadot SDK Docs](https://paritytech.github.io/polkadot-sdk/master/sp_state_machine/index.html)
- [FRAME Offchain Workers | Polkadot SDK Docs](https://paritytech.github.io/polkadot-sdk/master/polkadot_sdk_docs/reference_docs/frame_offchain_workers/index.html)
- [Indexers | Polkadot Developer Docs](https://docs.polkadot.com/develop/toolkit/integrations/indexers/)
