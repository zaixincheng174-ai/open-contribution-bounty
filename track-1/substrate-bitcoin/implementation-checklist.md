# Implementation Checklist: Substrate Bitcoin-Style Course

Use this checklist to build or review a course module for issue #1. The module
should teach a Bitcoin-inspired Substrate chain without claiming full Bitcoin
Core compatibility.

## Repository Scope

- [ ] The course lives under `track-1/substrate-bitcoin/`.
- [ ] The README explains UTXO, proof of work, mining rewards, and address
  conversion boundaries.
- [ ] Any snippets are identified as illustrative unless they are copied from a
  tested implementation.
- [ ] References are public and reachable.
- [ ] No private keys, seed phrases, or RPC secrets are included.

## UTXO Ledger

- [ ] The module defines outpoints as transaction hash plus output index.
- [ ] The module defines transaction inputs and outputs.
- [ ] The module explains how output ownership is represented.
- [ ] The module explains transaction hashing and canonical encoding.
- [ ] Spend validation rejects missing inputs.
- [ ] Spend validation rejects duplicate inputs.
- [ ] Spend validation rejects already spent inputs.
- [ ] Spend validation verifies signatures or unlocking data.
- [ ] Spend validation rejects output sums greater than input sums.
- [ ] State transition removes spent UTXOs and inserts new UTXOs atomically.

## Unsigned Transaction Handling

- [ ] The module explains why UTXO spends can be unsigned Substrate extrinsics.
- [ ] The module requires `ValidateUnsigned` or equivalent validation.
- [ ] Transaction-pool tags prevent two pending spends of the same input.
- [ ] Transaction priority is tied to fee or fee rate.
- [ ] Runtime validation still protects blocks from invalid transactions.

## Proof of Work

- [ ] The module separates node-side PoW from runtime-side UTXO logic.
- [ ] The module names the PoW import queue or consensus crate used.
- [ ] Mining searches for a nonce or seal satisfying a target.
- [ ] Invalid seals are rejected during block import.
- [ ] Difficulty is configurable for local testing.
- [ ] Fork choice or cumulative work is discussed.
- [ ] Manual seal is not presented as proof of work.

## Miner Reward and Fees

- [ ] The module describes one reward output per block.
- [ ] Reward creation is not exposed as a public mint call.
- [ ] Transaction fees are computed from input and output sums.
- [ ] Miner reward can include fees.
- [ ] Optional coinbase maturity is documented if included.
- [ ] Reward tests prove users cannot create arbitrary UTXOs.

## Address Conversion

- [ ] The module distinguishes SS58 accounts from Bitcoin addresses.
- [ ] P2PKH conversion uses a compatible public key hash.
- [ ] P2WPKH conversion uses Bech32 witness encoding.
- [ ] P2TR conversion uses Bech32m and an x-only public key.
- [ ] The module states when conversions are display-only.
- [ ] The module does not imply sr25519 keys can spend Taproot outputs.

## Verification

- [ ] Valid spend succeeds.
- [ ] Missing input fails.
- [ ] Double spend fails.
- [ ] Wrong signature fails.
- [ ] Overspend fails.
- [ ] Mempool conflict fails or is displaced deterministically.
- [ ] Valid PoW block imports.
- [ ] Invalid PoW block is rejected.
- [ ] Miner reward appears exactly once per block.
- [ ] Documentation includes expected observations for each test.

## Safety and Accuracy

- [ ] The course names simplified pieces clearly.
- [ ] The course mentions probabilistic finality and reorgs.
- [ ] The course avoids saying it is Bitcoin-compatible unless script, address,
  networking, difficulty, and wallet behavior are actually compatible.
- [ ] The course explains that UTXO state, not account free balance, is the source
  of spendable value.
