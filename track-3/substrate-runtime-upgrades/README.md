# Runtime Upgrades in Substrate: A Technical Walkthrough

Runtime upgrades are one of the most important features of a Substrate or
Polkadot SDK-based chain. They let a network change its state transition
function without coordinating a hard fork across every node operator. Instead
of asking all participants to stop, download a new binary, and restart at the
same time, the chain stores new runtime code on-chain and begins executing that
runtime after the upgrade is enacted.

This article walks through the full upgrade path from a builder's point of
view: what the runtime is, what changes during an upgrade, how to prepare the
Wasm artifact, how storage migrations fit into the process, and what operators
should validate before and after an upgrade.

## Node code and runtime code

A Polkadot SDK node has two related but different pieces of software.

The node, sometimes called the client, is the native executable that handles
networking, block import, consensus participation, database access, RPC, and
host functions. The runtime is the chain-specific state transition logic. It
defines pallets, dispatchable calls, storage items, runtime APIs, fees, weights,
events, errors, and the rules for applying extrinsics to state.

The runtime is compiled to WebAssembly. That Wasm blob is stored on-chain, so
nodes can fetch and execute the runtime that the chain itself has accepted.
This separation is what makes forkless upgrades possible. A node binary may
remain the same while the runtime changes from version N to version N + 1.

The separation is not unlimited. Runtime code still depends on the execution
environment provided by the node. If a new runtime requires host functions or
behavior that older node binaries do not provide, operators may need a client
upgrade before or alongside the runtime upgrade. Treat runtime compatibility as
an explicit release item, not an assumption.

## What a runtime upgrade changes

A runtime upgrade replaces the on-chain Wasm runtime code. In practical terms,
that can change:

- Pallet configuration and composition.
- Dispatchable calls available to users.
- Events, errors, and metadata exposed to applications.
- Runtime APIs used by nodes, indexers, wallets, and tooling.
- Fee, weight, and benchmark results.
- Storage layout or the interpretation of existing storage values.
- Governance, staking, balances, XCM, or parachain-specific logic.

Some changes are simple. Adding a new call to a custom pallet may only require
code changes, tests, a runtime version bump, and a new Wasm build. Other changes
are high risk. Anything that changes storage encoding or removes storage needs
a migration plan because the new runtime must be able to read state written by
the old runtime.

## The preparation workflow

A safe upgrade starts before the Wasm file exists.

First, define the upgrade scope. A small runtime upgrade should have a clear
reason: bug fix, feature addition, parameter change, pallet upgrade, migration,
or dependency update. Reviewers should be able to explain what state, calls,
APIs, and users are affected.

Second, audit storage impact. If the change modifies a storage type, enum
ordering, key prefix, pallet name, removed item, or encoded interpretation, plan
a storage migration. If the change only adds a new storage item, migration may
not be necessary because there is no old value to transform. The important
point is to decide this deliberately.

Third, bump the runtime version. In Polkadot SDK runtimes, `spec_version` is
the key version used to identify a new runtime specification. Forgetting this
bump is a common failure mode: the chain may not recognize the submitted code
as a new runtime.

Fourth, build the runtime Wasm artifact. Production upgrade submissions use
the compact compressed Wasm produced by the runtime build pipeline, not the
native node binary.

Fifth, run local validation. This should include unit tests, integration tests
where available, runtime API checks, metadata checks, and migration tests. If
the upgrade changes weights, rerun benchmarks and update the committed weight
files instead of relying on stale estimates.

## Submitting the upgrade

The exact submission path depends on the chain.

In a tutorial or local development chain, the Sudo pallet may be used to call a
root-level upgrade path such as `system.setCode`. That is useful for learning,
but it is not the governance model for a mature production network.

On a production chain, runtime upgrades are normally authorized through the
chain's governance or another configured root-origin process. The important
security rule is that replacing runtime code is a privileged action. It should
be reviewed, approved, and executed through the chain's intended authority
model.

For parachains, the upgrade flow may also involve parachain-specific validation
function handling. Builders should follow the current documentation and chain
runtime configuration rather than assuming every network exposes the same
extrinsic sequence.

## Storage migrations

Storage migrations are the part of runtime upgrades that most often causes
permanent damage when handled casually.

A migration is required when the new runtime cannot correctly decode or
interpret existing on-chain values without transforming them. Common examples
include changing a stored struct, reordering enum variants, changing a storage
key, renaming storage in a way that affects keys, removing old storage, or
changing the semantic meaning of already stored bytes.

Polkadot SDK migrations are typically implemented with the `OnRuntimeUpgrade`
trait or with `UncheckedOnRuntimeUpgrade` wrapped in `VersionedMigration`.
The migration runs during the runtime upgrade before normal block logic
continues. It must return a correct weight and it must finish within the
available limits for the chain.

Good migration design has several properties:

- It reads old storage using the old format.
- It writes new storage using the new format.
- It only runs for the intended storage version.
- It updates storage versioning after success.
- It has bounded reads and writes, or uses a multi-block migration design when
  a single block cannot safely do all the work.
- It includes `pre_upgrade` and `post_upgrade` hooks behind `try-runtime`
  features so the migration can be tested against representative state.

Single-block migrations are simple but risky. If a migration needs to touch a
large or unbounded number of keys, a multi-block migration or staged design may
be necessary. An overweight migration can halt a chain because the runtime
upgrade code is mandatory once the new runtime is enacted.

## Migration review template

For any upgrade that changes storage, reviewers should require a short
migration note before the proposal is submitted. A useful note answers these
questions in one place:

- Old storage: pallet, storage item, old type, old storage version, and
  expected key count.
- New storage: new type, new storage version, and whether old data is retained,
  transformed, or deleted.
- Trigger: why the migration is needed and which runtime change would fail
  without it.
- Bounds: maximum reads, writes, decoded value sizes, and whether the work fits
  in one block.
- Weight: how the migration weight was estimated and where that weight is
  returned.
- Guards: storage-version checks that prevent the migration from running twice.
- Try-runtime state: snapshot, live state source, or fixture used for
  `pre_upgrade` and `post_upgrade`.
- Invariants: conditions that must hold after the upgrade, such as item counts,
  balances, ownership, or indexes.
- Cleanup: old prefixes, obsolete values, or compatibility paths removed after
  success.
- Operations: who watches the upgrade block, which events or logs confirm
  success, and what follow-up checks run.

This template is intentionally plain. Its purpose is to force the team to make
implicit migration assumptions reviewable before those assumptions become
on-chain code.

## Testing with try-runtime

`try-runtime` exists to catch migration mistakes before they reach production.
The key workflow is to execute an `on-runtime-upgrade` test using the new
runtime Wasm against representative chain state. This calls the migration
testing hooks, runs the upgrade logic, and checks post-upgrade invariants.

A practical test plan should include:

- A current or recent chain snapshot where possible.
- `try-runtime on-runtime-upgrade` for migrations.
- Unit tests for pure migration transformation logic.
- Runtime API compatibility checks for clients and indexers.
- Metadata comparison for wallets and application tooling.
- Weight checks for any changed dispatchables or migrations.
- Manual smoke tests for the feature or fix that motivated the upgrade.

If `try-runtime` fails, do not treat it as a tooling nuisance. It often means
the migration cannot decode old state, the post-upgrade invariant is wrong, the
runtime was built with the wrong features, or the migration weight is not
accounted for correctly.

## Operational rollout checklist

Before submitting an upgrade, the team should be able to answer these
questions:

1. What user-visible behavior changes after the upgrade?
2. Which pallets, runtime APIs, metadata, events, and errors changed?
3. Was `spec_version` bumped?
4. Was the Wasm artifact built from the reviewed commit?
5. Were benchmarks rerun for changed weights?
6. Does the upgrade require node operators to upgrade their client binary?
7. Does any storage migration run, and has it been tested on representative
   state?
8. Is the submission path governance, sudo, or another root-origin process?
9. Who monitors the upgrade block and the blocks immediately after it?
10. What is the incident plan if the new runtime behaves incorrectly?

The incident plan matters because a runtime upgrade is not "rolled back" like
a normal deployment. If the network accepts bad runtime code, the usual fix is
another runtime upgrade that repairs the problem. That means the team needs a
prepared communication path, monitoring, and a fast but reviewed remediation
process.

## Common failure modes

Forgetting `spec_version` is the simple one. The code may build, the proposal
may be submitted, and the team may still not get the expected runtime change.

The more dangerous failures involve storage. A runtime that decodes old state
with the wrong type can panic or corrupt meaning. A migration that assumes a
small number of keys may exceed block limits on production state. A migration
that does not gate on storage version may run twice. A migration that does not
clean removed storage can leave state bloat behind.

Compatibility failures are also common. A new runtime may compile while
breaking downstream applications that depend on metadata, runtime APIs, event
names, or dispatchable call indexes. Wallets, indexers, dashboards, and bots
are part of the upgrade surface even when they do not participate in consensus.

Finally, teams sometimes under-plan operations. A runtime upgrade should have
a named monitoring window, known block numbers or enactment timing, expected
events, clear success criteria, and a path for users to report problems.

## Summary

A Substrate runtime upgrade is a controlled replacement of the on-chain Wasm
state transition function. The happy path is straightforward: change the
runtime, bump the version, build Wasm, submit through the chain's authority
model, and verify the new runtime is active.

The engineering discipline is in the edge cases. Storage migrations must be
versioned, weighted, bounded, and tested. Runtime APIs and metadata must be
reviewed as public interfaces. Operators must know whether client binaries
also need to change. Governance or root-origin submission must be treated as a
security-sensitive release process.

When handled well, runtime upgrades let a live chain evolve without hard forks.
When handled casually, they can break the chain at the exact layer where every
node must agree. That is why a good runtime upgrade is not only a Wasm file; it
is a reviewed release, migration, testing, and operations plan.

## References

- [Runtime Upgrades | Polkadot Developer Docs](https://docs.polkadot.com/develop/parachains/maintenance/runtime-upgrades/)
- [Storage Migrations | Polkadot Developer Docs](https://docs.polkadot.com/parachains/runtime-maintenance/storage-migrations/)
- [Node and Runtime | Polkadot Developer Docs](https://docs.polkadot.com/polkadot-protocol/parachain-basics/node-and-runtime/)
- [FRAME Runtime Upgrades and Migrations | Polkadot SDK Docs](https://paritytech.github.io/polkadot-sdk/master/polkadot_sdk_docs/reference_docs/frame_runtime_upgrades_and_migrations/index.html)
- [try-runtime CLI documentation](https://paritytech.github.io/try-runtime-cli/try_runtime/)
