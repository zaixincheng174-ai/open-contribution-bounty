# Review Checklist: Substrate Bitcoin-Style Course

Use this file when reviewing submissions for the Substrate Bitcoin blockchain
course bounty.

## Fast Acceptance Scan

- [ ] The submission targets issue #1.
- [ ] The module covers UTXO state, proof of work, rewards, transaction
  validation, and address conversion caveats.
- [ ] The module is usable as a course without requiring hidden Discord context.
- [ ] The module does not claim full Bitcoin compatibility.
- [ ] The module includes implementation and verification guidance.
- [ ] Public references are included.

## UTXO Correctness

- [ ] Outputs are stored and spent by outpoint.
- [ ] Input signatures or unlocking data authorize each spend.
- [ ] Output value cannot exceed input value.
- [ ] Duplicate inputs are rejected.
- [ ] Double spends are rejected.
- [ ] State updates are atomic.
- [ ] The course explains how fees are derived.

## Consensus Correctness

- [ ] PoW is handled by node consensus, not by a normal runtime call.
- [ ] The block import path validates PoW seals.
- [ ] Difficulty or target selection is discussed.
- [ ] Fork choice or cumulative work is mentioned.
- [ ] Manual seal is described only as a test helper.
- [ ] Probabilistic finality and reorg risk are mentioned.

## Reward Correctness

- [ ] Reward creation is restricted to block authoring.
- [ ] Fees can be added to miner rewards.
- [ ] The course rejects arbitrary user minting.
- [ ] Optional coinbase maturity is clearly identified.

## Address Correctness

- [ ] SS58 is not treated as a Bitcoin address format.
- [ ] P2PKH, P2WPKH, and P2TR are described with different payloads.
- [ ] The course distinguishes display conversion from spend compatibility.
- [ ] Key-type constraints are stated.

## Red Flags

- [ ] The submission uses account balances instead of UTXOs.
- [ ] The submission accepts unsigned extrinsics without validation.
- [ ] The submission lets any user mint coinbase outputs.
- [ ] The submission says Aura, BABE, or manual seal is proof of work.
- [ ] The submission ignores transaction-pool double-spend conflicts.
- [ ] The submission treats sr25519 Substrate accounts as Taproot keys.
- [ ] The submission omits verification tests or expected observations.

If any red flag is present, request changes before considering the bounty
complete.
