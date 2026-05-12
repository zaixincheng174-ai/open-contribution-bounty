# Implementation Checklist: EVM-Compatible Substrate Course

Use this checklist to build or review a course module for issue #6. The module
should teach EVM compatibility on a Substrate-based chain without claiming full
Ethereum equivalence unless the Ethereum compatibility stack is actually
present.

## Repository Scope

- [ ] The course lives under `track-1/substrate-evm/`.
- [ ] The README explains the difference between EVM execution support and full
  Ethereum ecosystem compatibility.
- [ ] The README identifies the runtime layer and node/RPC layer separately.
- [ ] Runtime configuration and RPC examples are concrete enough to guide a
  learner while remaining clearly labeled as sketches.
- [ ] References are public and reachable.
- [ ] No private keys, seed phrases, RPC secrets, or funded wallet material are
  included.

## Compatibility Target

- [ ] The course states whether it targets EVM execution only or full Ethereum
  JSON-RPC compatibility.
- [ ] If full compatibility is claimed, the course includes Ethereum transaction,
  receipt, log, and RPC behavior.
- [ ] If only EVM execution is included, the course does not promise MetaMask or
  Hardhat compatibility.
- [ ] The course compares Frontier-style EVM compatibility with the newer
  `pallet-revive` path at a high level.

## Runtime Configuration

- [ ] The module identifies the EVM execution pallet or equivalent runtime
  module.
- [ ] Address mapping between `H160` and Substrate account IDs is documented.
- [ ] The native token used for gas and fees is documented.
- [ ] Chain ID is configured and explained.
- [ ] Gas-to-weight mapping is documented.
- [ ] Block gas and transaction gas limits are documented.
- [ ] EVM account nonce handling is described.
- [ ] Genesis funding for local EVM accounts is documented if used.

## Ethereum Transaction and RPC Support

- [ ] Ethereum-formatted transactions are described if full compatibility is in
  scope.
- [ ] Replay protection through `chainId` is tested or manually verified.
- [ ] Receipts and event logs are described.
- [ ] The Frontier or Ethereum RPC components are identified.
- [ ] The local HTTP and WebSocket RPC endpoints are documented.
- [ ] Unsupported Ethereum RPC methods are called out instead of hidden.
- [ ] RPC smoke-test commands verify chain ID, block number, and account
  balance against the local node.

## Developer Tooling

- [ ] The course includes a local node startup path.
- [ ] The course includes a wallet connection path.
- [ ] The course includes a Hardhat, Foundry, Remix, ethers, or viem deployment
  path.
- [ ] Deployment examples use environment variables for local development keys
  and do not commit key material.
- [ ] The course verifies `eth_chainId`.
- [ ] The course verifies contract deployment.
- [ ] The course verifies a read-only contract call.
- [ ] The course verifies a write transaction and receipt.
- [ ] The course verifies event-log queries.

## Precompiles

- [ ] The course explains what precompiles are.
- [ ] Precompile addresses are documented.
- [ ] Precompile gas and weight charging is documented.
- [ ] Permission checks are documented for any runtime-dispatching precompile.
- [ ] Malformed input tests are included.
- [ ] The course does not present precompiles as a way to bypass runtime
  authorization.

## Verification

- [ ] Contract deployment succeeds.
- [ ] Deployment with insufficient gas fails.
- [ ] Contract storage changes after a write call.
- [ ] Reverted contract calls report failure.
- [ ] Event logs are queryable by address and block range.
- [ ] Account nonce increments after accepted transactions.
- [ ] Wrong chain ID transactions fail.
- [ ] Insufficient balance transactions fail.
- [ ] Native and EVM balance behavior matches the documented account model.
- [ ] Hardhat or equivalent tooling can deploy through the Ethereum RPC endpoint.

## Safety and Accuracy

- [ ] The course does not claim Ethereum mainnet equivalence unless block,
  transaction, receipt, log, finality, and RPC behavior have been verified.
- [ ] Gas-to-weight economics are treated as a security boundary.
- [ ] RPC compatibility is treated as part of the deliverable, not only a nice
  developer convenience.
- [ ] SS58 addresses and Ethereum `H160` addresses are not treated as the same
  format.
- [ ] Any simplified local development behavior is clearly labeled as local-only.
