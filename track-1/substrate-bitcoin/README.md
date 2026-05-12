# Substrate Bitcoin-Style Blockchain

This module shows how to design a Substrate-based chain with Bitcoin-like
behavior: a UTXO ledger, proof-of-work block production, miner rewards,
transaction validation, and optional address conversion from Substrate accounts
to Bitcoin-style address formats.

The goal is not to rebuild Bitcoin Core inside Substrate. The goal is to teach
the runtime and node boundaries needed to build a small educational chain whose
state transition model feels like Bitcoin while still using the Polkadot SDK
framework.

## Learning Outcomes

By the end of this module, a learner should be able to:

- Explain the difference between Substrate's default account model and a UTXO
  model.
- Define transaction inputs, outputs, outpoints, ownership scripts, and spend
  validation.
- Build a runtime pallet that stores and spends UTXOs safely.
- Explain why UTXO transactions are usually submitted as unsigned extrinsics
  whose signatures live inside the transaction payload.
- Wire proof-of-work block production in the node service.
- Add miner rewards without letting users mint arbitrary UTXOs.
- Test double-spend rejection, signature validation, and overspend rejection.
- Explain the limits of converting Substrate account IDs into Bitcoin address
  formats.

## Architecture Overview

A Bitcoin-style Substrate chain has two main layers:

- Runtime layer:
  Owns UTXO storage, transaction validity, spend rules, fee rules, and reward
  creation.
- Node layer:
  Owns proof-of-work mining, block import, difficulty calculation, and the
  transaction pool.

Keep these layers separate. A common failure mode is putting mining logic inside
the UTXO pallet or letting the node accept transactions that the runtime cannot
validate.

## Core Components

### UTXO Pallet

The pallet is the ledger. It should store a map from outpoint to output:

```text
OutPoint = transaction_hash + output_index
Output = value + locking condition + optional metadata
```

The pallet should provide one main spend call:

```text
spend(transaction)
```

The transaction should contain:

- Inputs that reference existing outpoints.
- Unlocking data or signatures for each input.
- Outputs with new values and locking conditions.
- Fee calculation as `sum(inputs) - sum(outputs)`.
- A canonical encoding used for transaction hashing and signing.

The pallet should reject:

- Missing inputs.
- Duplicate inputs within the same transaction.
- Inputs that were already spent.
- Invalid signatures or unlocking data.
- Output sums greater than input sums.
- Zero-output or malformed transactions unless explicitly allowed.
- Outputs below the configured dust threshold, if the course includes one.

An educational implementation can start with this shape. It is intentionally
illustrative; concrete derives, bounds, and encoding choices depend on the
runtime template used by the course.

```rust
pub struct OutPoint {
    pub tx_hash: H256,
    pub output_index: u32,
}

pub struct TransactionInput {
    pub previous_output: OutPoint,
    pub signature: Vec<u8>,
    pub public_key: Vec<u8>,
}

pub struct TransactionOutput<Balance> {
    pub value: Balance,
    pub locking_key: Vec<u8>,
}

pub struct Transaction<Balance> {
    pub inputs: Vec<TransactionInput>,
    pub outputs: Vec<TransactionOutput<Balance>>,
}
```

The course should then make learners implement the state transition as one
atomic operation: load all referenced inputs, verify the spend, remove consumed
UTXOs, and insert new outputs. Avoid a partial update path where one input is
removed before a later signature or value check fails.

### Unsigned Extrinsic Validation

Bitcoin transactions are signed internally. A Substrate UTXO pallet can therefore
submit them through unsigned extrinsics, but unsigned calls must be protected by
strict `ValidateUnsigned` logic.

The validation path should:

- Recompute the transaction hash.
- Verify all referenced UTXOs exist.
- Verify the transaction signatures.
- Reject duplicate input references.
- Reject transactions with insufficient fees.
- Provide a stable transaction pool tag so two transactions spending the same
  input cannot both be accepted.
- Assign priority from fees or fee rate.

Unsigned does not mean unauthenticated. It means the Substrate extrinsic wrapper
does not use an account signature because the UTXO transaction already carries
its own authorization.

A useful validation sketch is:

```rust
impl<T: Config> ValidateUnsigned for Pallet<T> {
    type Call = Call<T>;

    fn validate_unsigned(
        _source: TransactionSource,
        call: &Self::Call,
    ) -> TransactionValidity {
        let Call::spend { transaction } = call else {
            return InvalidTransaction::Call.into();
        };

        let checked = Self::check_transaction(transaction)
            .map_err(|_| InvalidTransaction::Custom(1))?;

        ValidTransaction::with_tag_prefix("utxo-spend")
            .and_provides(checked.input_tags)
            .priority(checked.fee_rate)
            .longevity(64_u64)
            .propagate(true)
            .build()
    }
}
```

The exact API may differ by Polkadot SDK version, but the lesson should keep the
same security boundary: transaction-pool validation rejects bad signatures,
missing inputs, duplicate inputs, stale formats, and input conflicts before a
miner includes the transaction.

### Proof-of-Work Node Service

Proof of work is node-side consensus. The runtime can expose difficulty and
reward rules, but the node service must:

- Build candidate blocks.
- Hash block headers or seal payloads.
- Search for a nonce satisfying the target.
- Import valid blocks.
- Reject blocks with invalid seals.
- Track cumulative work or an equivalent fork-choice signal.

For an educational chain, a simple CPU miner is enough. The course should still
make the production boundary clear: the runtime validates state transitions, and
the node validates PoW seals and imports blocks.

### Miner Reward

A Bitcoin-like chain needs a controlled way to create new UTXOs. Do not expose a
public `mint` call.

Use one of these patterns:

- A block reward inherent that creates a coinbase output for the block author.
- A runtime hook that creates the reward once per block.
- A special coinbase transaction accepted only when produced by block authoring.

The reward path should enforce:

- One reward per block.
- Configurable subsidy amount.
- Fee collection from transactions in the block.
- Optional coinbase maturity before the reward can be spent.

### Address Conversion

The bounty allows an optional converter from Substrate addresses to Bitcoin
Legacy, SegWit, and Taproot addresses. This is useful as a learning exercise,
but it needs careful caveats.

Substrate commonly uses SS58-encoded account IDs. Bitcoin address formats encode
different payloads:

- Legacy P2PKH addresses encode a hash160 of a secp256k1 public key.
- Native SegWit P2WPKH addresses encode a witness program.
- Taproot P2TR addresses encode an x-only Schnorr public key.

Do not imply that every Substrate account can safely become every Bitcoin
address type. If the source key is sr25519 or ed25519, it is not a Bitcoin
spending key. A converter can demonstrate encoding formats, but production
spending compatibility requires the correct key type and signing rules.

## Suggested Course Walkthrough

### Milestone 1: Build the UTXO Ledger

1. Define `OutPoint`, `TransactionInput`, `TransactionOutput`, and
   `Transaction`.
2. Store unspent outputs in pallet storage.
3. Add a genesis helper that creates initial UTXOs for local accounts.
4. Implement transaction hashing.
5. Implement spend validation.
6. Implement atomic spend: remove consumed UTXOs and insert new UTXOs.

Learner checkpoint:

- A valid transaction spends one existing output and creates two new outputs.
- A second transaction that reuses the same input is rejected.
- A transaction with outputs greater than inputs is rejected.

### Milestone 2: Add Signatures

1. Pick a key type for the educational chain.
2. Define the signing payload.
3. Verify each input against its referenced output's locking condition.
4. Add negative tests for wrong signer, wrong payload, and tampered output value.

Learner checkpoint:

- Authorization belongs to the UTXO input, not to a Substrate account wrapper.
- The transaction hash is stable across encode/decode.

### Milestone 3: Protect the Transaction Pool

1. Implement unsigned transaction validation.
2. Tag every input as a provided or required resource.
3. Reject duplicate input spends before block inclusion.
4. Use fee or fee rate as priority.

Learner checkpoint:

- Two mempool transactions cannot spend the same outpoint.
- Invalid transactions are rejected before block production.
- Runtime validation still rejects invalid transactions even if they reach a
  block.

### Milestone 4: Add Proof of Work

1. Configure a PoW import queue in the node service.
2. Implement or reuse a simple PoW algorithm.
3. Add a local mining command.
4. Mine blocks containing UTXO transactions.
5. Verify invalid seals are rejected.

Learner checkpoint:

- Blocks are produced by mining, not by Aura or manual seal.
- Difficulty is visible and adjustable for local testing.
- Fork choice prefers the chain with the correct work metric.

### Milestone 5: Add Miner Rewards

1. Add a block reward UTXO.
2. Add transaction fees to the reward.
3. Prevent user-created reward outputs.
4. Optionally enforce coinbase maturity.

Learner checkpoint:

- The miner reward appears once per block.
- Fees are conserved.
- A user cannot call a public mint to create arbitrary funds.

### Milestone 6: Add Address Conversion

1. Convert a compatible public key to P2PKH.
2. Convert a compatible public key hash to Bech32 P2WPKH.
3. Convert a compatible x-only public key to Bech32m P2TR.
4. Document which conversions are display-only and which can spend.

Learner checkpoint:

- The course distinguishes address encoding from spend authorization.
- The course does not claim sr25519 SS58 accounts are Taproot spending keys.

## Runtime Verification Matrix

| Test | Expected result |
| --- | --- |
| Spend an existing UTXO with a valid signature | Transaction succeeds |
| Spend a missing UTXO | Transaction rejected |
| Reuse an input in the same transaction | Transaction rejected |
| Spend an already consumed UTXO | Transaction rejected |
| Provide wrong signer data | Transaction rejected |
| Create outputs greater than inputs | Transaction rejected |
| Submit two mempool transactions for one input | One is rejected or displaced |
| Mine a block with a valid PoW seal | Block imports |
| Import a block with an invalid seal | Block rejected |
| Claim two miner rewards in one block | Block rejected |

## Review Notes

A strong submission should be honest about what is educational and what is
production-ready. A small Substrate UTXO chain can teach Bitcoin-like concepts,
but production Bitcoin compatibility involves much more:

- Peer-to-peer networking behavior.
- Script interpreter compatibility.
- Difficulty retargeting over long windows.
- Chain reorganization handling.
- Wallet standards.
- Address and key-type compatibility.
- Mature fee market behavior.

The course should not over-claim compatibility. It should explain which pieces
are Bitcoin-inspired and which pieces are simplified for a Substrate learning
environment.

## Common Mistakes

- Treating balances as accounts:
  A UTXO chain should derive spendable value from unspent outputs, not from a
  free balance map.
- Trusting unsigned extrinsics:
  Every unsigned UTXO spend needs strict validation and transaction-pool tags.
- Forgetting atomicity:
  Spending inputs and inserting outputs must happen as one state transition.
- Letting users mint:
  Reward creation must be restricted to the block reward path.
- Ignoring key formats:
  SS58 account IDs and Bitcoin addresses are not interchangeable spending
  formats.
- Calling manual seal PoW:
  Manual seal is useful for tests, but it is not proof of work.
- Skipping reorg discussion:
  Bitcoin-like chains have probabilistic finality, so course material should
  mention reorganizations even if the local demo is simple.

## Primary References

- [Substrate UTXO Workshop][utxo-workshop]
- [Polkadot SDK `sc-consensus-pow` crate][pow-crate]
- [Polkadot SDK `sp-consensus-pow` crate][sp-pow-crate]
- [Bitcoin Developer Guide: Transactions][bitcoin-transactions]
- [Polkadot Wiki: Accounts and SS58][ss58-accounts]
- [BIP 173: Bech32 SegWit address format][bip-173]
- [BIP 350: Bech32m address format][bip-350]

[utxo-workshop]: https://github.com/substrate-developer-hub/utxo-workshop
[pow-crate]: https://paritytech.github.io/polkadot-sdk/master/sc_consensus_pow/index.html
[sp-pow-crate]: https://paritytech.github.io/polkadot-sdk/master/sp_consensus_pow/index.html
[bitcoin-transactions]: https://developer.bitcoin.org/devguide/transactions.html
[ss58-accounts]: https://wiki.polkadot.com/learn/learn-account-advanced/
[bip-173]: https://github.com/bitcoin/bips/blob/master/bip-0173.mediawiki
[bip-350]: https://github.com/bitcoin/bips/blob/master/bip-0350.mediawiki
