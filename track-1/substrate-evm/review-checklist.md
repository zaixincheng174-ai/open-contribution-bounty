# Review Checklist: EVM-Compatible Substrate Course

Use this file when reviewing submissions for the EVM-compatible Substrate
blockchain course bounty.

## Fast Acceptance Scan

- [ ] The submission targets issue #6.
- [ ] The module covers EVM execution, Ethereum compatibility layers, account
  mapping, gas/weight, RPC, tooling, precompiles, and verification.
- [ ] The module is usable as a course without hidden Discord context.
- [ ] The module distinguishes `pallet-evm` execution from full Ethereum
  JSON-RPC compatibility.
- [ ] The module includes implementation guidance and expected observations.
- [ ] Public references are included.

## Runtime Correctness

- [ ] The runtime owns EVM state transitions.
- [ ] EVM account storage, nonce, balance, bytecode, and contract storage are
  discussed.
- [ ] Address mapping between `H160` and Substrate accounts is explicit.
- [ ] Chain ID and replay protection are explicit.
- [ ] Gas-to-weight mapping is treated as required, not optional.
- [ ] Fee currency and balance source are documented.

## Ethereum Compatibility Correctness

- [ ] The course explains when `pallet-ethereum` or equivalent transaction and
  block emulation is needed.
- [ ] Receipts and logs are included when full compatibility is claimed.
- [ ] Ethereum JSON-RPC modules are named.
- [ ] The local RPC endpoint is documented.
- [ ] Unsupported RPC methods are identified.
- [ ] Ethereum tooling compatibility is verified by a deployment or call path.

## Tooling Correctness

- [ ] Wallet connection verifies the expected chain ID.
- [ ] Hardhat, Foundry, Remix, ethers, or viem is used in a realistic workflow.
- [ ] Contract deployment produces a transaction receipt.
- [ ] Contract writes change state.
- [ ] Contract reads return expected state.
- [ ] Event logs can be queried.

## Precompile Correctness

- [ ] Precompile addresses are stable and documented.
- [ ] Precompile weight charging is documented.
- [ ] Runtime permissions are checked.
- [ ] Malformed input behavior is tested.
- [ ] Precompile failures are deterministic.

## Red Flags

- [ ] The submission says `pallet-evm` alone guarantees MetaMask compatibility.
- [ ] The submission omits chain ID or replay protection.
- [ ] The submission ignores gas-to-weight mapping.
- [ ] The submission uses precompiles without weight or permission checks.
- [ ] The submission treats SS58 and Ethereum addresses as interchangeable.
- [ ] The submission claims Ethereum equivalence while omitting receipts, logs,
  or JSON-RPC compatibility.
- [ ] The submission includes private keys, seed phrases, or funded wallet
  material.
- [ ] The submission has no verification path for an actual Solidity deployment.

If any red flag is present, request changes before considering the bounty
complete.
