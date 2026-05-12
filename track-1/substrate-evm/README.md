# EVM-Compatible Substrate Blockchain

This module explains how to design a Substrate-based chain that can execute
Ethereum Virtual Machine contracts and, when needed, expose Ethereum-compatible
JSON-RPC behavior for tools such as MetaMask, Remix, Hardhat, Foundry, ethers,
and viem.

The goal is not to claim that every Substrate chain becomes Ethereum mainnet.
The goal is to teach the runtime and node boundaries needed for an educational
EVM-compatible chain:

- `pallet-evm` or an equivalent EVM execution pallet runs EVM bytecode.
- `pallet-ethereum` emulates Ethereum-style blocks, transactions, logs, and
  receipts when full Ethereum compatibility is required.
- Frontier RPC components expose Ethereum JSON-RPC endpoints from the node.
- Gas, weight, fees, precompiles, and account mapping are configured explicitly
  so compatibility does not hide unsafe economic assumptions.

Current Polkadot documentation also describes `pallet-revive` as a modern smart
contract path that can support both PVM and EVM bytecode. This course focuses on
the Frontier-style EVM-compatible Substrate chain requested by the bounty, while
calling out where learners should compare it with `pallet-revive`.

## Learning Outcomes

By the end of this module, a learner should be able to:

- Explain the difference between EVM execution support and full Ethereum
  ecosystem compatibility.
- Identify the responsibilities of `pallet-evm`, `pallet-ethereum`, and
  Frontier RPC components.
- Configure EVM accounts, chain ID, gas-to-weight mapping, fees, and precompile
  sets at a high level.
- Explain how Ethereum `H160` addresses map to Substrate accounts and balances.
- Run a local EVM-compatible node and connect Ethereum developer tools to it.
- Deploy and call a simple Solidity contract through Ethereum JSON-RPC.
- Verify event logs, receipts, nonces, chain ID replay protection, gas limits,
  and native balance accounting.
- Name the major security risks: gas/weight mispricing, unsafe precompiles,
  weak RPC compatibility, chain ID mistakes, and misleading Ethereum-equivalence
  claims.

## Architecture Overview

An EVM-compatible Substrate chain has two separate compatibility layers.

### Runtime Layer

The runtime decides whether EVM transactions are valid and how they change
state. It owns:

- EVM account storage.
- Contract bytecode and contract storage.
- Nonces and balances for EVM accounts.
- Gas accounting and gas-to-weight conversion.
- Transaction fee charging.
- Precompile dispatch.
- Optional Ethereum-formatted transaction validation through
  `pallet-ethereum`.

### Node and RPC Layer

The node makes the runtime usable from Ethereum tools. It owns:

- Ethereum JSON-RPC methods such as `eth_chainId`, `eth_blockNumber`,
  `eth_getBalance`, `eth_sendRawTransaction`, `eth_getTransactionReceipt`, and
  log filters.
- Frontier database mappings between Substrate blocks and Ethereum-style blocks.
- Pending transaction handling for Ethereum-formatted transactions.
- WebSocket and HTTP RPC endpoints, commonly exposed on port `8545` for local
  Ethereum tooling.

Keep these layers separate. A runtime that can execute EVM bytecode is not
automatically compatible with MetaMask or Hardhat. Ethereum tools also need
Ethereum-style transaction encoding, receipts, logs, and JSON-RPC behavior.

## Compatibility Modes

### Mode 1: EVM Execution Only

Use this mode when the chain only needs to execute EVM bytecode inside the
runtime. Learners can still interact through Substrate-native APIs or custom
extrinsics.

Core pieces:

- EVM execution pallet.
- Address mapping between Substrate account IDs and `H160` addresses.
- Runtime balances used to fund EVM accounts.
- Gas-to-weight mapping and fee charging.
- Optional precompiles.

This mode is simpler, but it should not promise unmodified Ethereum tooling.
Without Ethereum RPC and Ethereum transaction emulation, users cannot assume
MetaMask, Hardhat, or existing dapps will work unchanged.

### Mode 2: Full Ethereum Compatibility

Use this mode when the chain should feel like an Ethereum-compatible network to
existing Ethereum tools.

Core pieces:

- EVM execution pallet.
- Ethereum transaction/block/log emulation.
- Ethereum JSON-RPC endpoints.
- Frontier block and transaction mapping database.
- Chain ID and replay protection.
- Receipt and event-log support.

This mode is more useful for dapp migration but harder to validate. The course
should include compatibility tests, not only runtime unit tests.

## Core Components

### EVM Execution Pallet

The EVM execution pallet is responsible for contract execution. A course module
should explain how it handles:

- Contract creation and calls.
- EVM account nonce and balance.
- Contract bytecode storage.
- Contract storage key-value state.
- Gas limit and gas used.
- Revert, out-of-gas, and successful execution outcomes.
- Logs emitted by contracts.

Learners should understand that EVM execution changes state through Substrate's
runtime, so block authors and validators still validate the final Substrate
state transition.

### Account Mapping

Ethereum uses 20-byte `H160` addresses. Substrate runtimes often use 32-byte
account IDs encoded as SS58 addresses.

A course must state which mapping it uses:

- One-way mapping from `H160` into a Substrate account ID.
- Mapping from a Substrate account into an EVM address.
- Separate EVM account storage funded from native balances.
- A unified account model where EVM and native accounts share the same balance
  source.

The mapping is not cosmetic. It decides who can sign, where balances live, what
wallets can spend funds, and whether native and Ethereum-compatible APIs show
the same account state.

### Gas, Weight, and Fees

Ethereum meters execution with gas. Substrate meters block resources with
weight. An EVM-compatible chain must map gas to weight and decide how fees are
charged.

The course should cover:

- Maximum gas per transaction.
- Maximum gas per block.
- Gas-to-weight conversion.
- Base fee, gas price, or fee multiplier policy.
- Native token used to pay for gas.
- Refund behavior after execution.
- Failure behavior for out-of-gas and reverted transactions.

Incorrect gas-to-weight mapping can make the chain vulnerable to denial of
service or make contracts uneconomical to use.

### Ethereum Transaction Emulation

For full Ethereum compatibility, the chain needs to accept and process
Ethereum-formatted transactions.

The module should cover:

- RLP-encoded signed transactions.
- `chainId` replay protection.
- Account nonce checks.
- Gas limit and gas price validation.
- Contract creation transactions.
- Contract call transactions.
- Receipt generation.
- Log and bloom generation if supported by the RPC layer.

If a chain only supports Substrate extrinsics that call EVM functions, it should
be described as EVM execution support, not full Ethereum compatibility.

### Ethereum JSON-RPC

Ethereum tools expect standard JSON-RPC behavior. At minimum, a local workshop
should demonstrate:

- `eth_chainId`
- `eth_blockNumber`
- `eth_getBalance`
- `eth_getTransactionCount`
- `eth_sendRawTransaction`
- `eth_getTransactionReceipt`
- `eth_call`
- `eth_estimateGas`
- `eth_getLogs`

The course should also explain which RPCs are intentionally unsupported. Some
Ethereum testing helpers assume RPC methods that may not exist on a Polkadot SDK
chain.

### Precompiles

Precompiles expose native functionality at special EVM addresses. They are
powerful because they let Solidity contracts call runtime functionality without
rewriting the contract environment.

A safe precompile section should cover:

- Address assignment and collision avoidance.
- Weight charging for the native operation.
- Input decoding and output encoding.
- Origin and permission checks.
- Failure and revert behavior.
- Tests for malformed input and underpriced execution.

Do not present precompiles as a shortcut around runtime permissions. A
precompile is still a runtime boundary and needs the same review discipline as a
normal pallet dispatch path.

## Suggested Course Walkthrough

### Milestone 1: Choose the Compatibility Target

1. Decide whether the course builds EVM execution only or full Ethereum
   compatibility.
2. Document the expected developer workflow.
3. Choose the local RPC ports for Substrate and Ethereum APIs.
4. Choose the native token that pays EVM gas.

Learner checkpoint:

- The learner can explain why `pallet-evm` alone is not enough for MetaMask
  compatibility.
- The learner can name the extra components required for Ethereum RPC support.

### Milestone 2: Configure Runtime EVM Execution

1. Add the EVM execution pallet or equivalent runtime module.
2. Configure address mapping.
3. Configure gas-to-weight mapping.
4. Configure chain ID.
5. Configure block gas and transaction gas limits.
6. Add the fee currency and withdrawal path.
7. Add initial funded EVM accounts in genesis for local testing.

Learner checkpoint:

- A funded EVM address can deploy a contract.
- A wrong chain ID transaction is rejected.
- A transaction with insufficient balance cannot pay gas.

### Milestone 3: Add Ethereum Transaction and Block Compatibility

1. Add Ethereum-formatted transaction handling if full compatibility is in
   scope.
2. Configure receipt and log storage.
3. Configure block-to-Ethereum mapping.
4. Ensure Ethereum nonces and Substrate transaction pool behavior agree.
5. Confirm finalized and best block queries return expected data.

Learner checkpoint:

- `eth_blockNumber` changes as blocks are produced.
- `eth_getTransactionReceipt` returns a receipt after deployment.
- Event logs from a contract call can be queried.

### Milestone 4: Expose Ethereum JSON-RPC

1. Enable HTTP and WebSocket RPC endpoints.
2. Enable the Ethereum RPC modules needed by the workshop.
3. Document local endpoint URLs.
4. Connect MetaMask or another wallet to the local chain.
5. Connect Hardhat, Foundry, Remix, ethers, or viem.

Learner checkpoint:

- A wallet displays the correct chain ID.
- A Hardhat deployment script deploys a contract through the local RPC.
- A read-only `eth_call` returns expected contract state.

### Milestone 5: Add Precompiles

1. Start with standard Ethereum-native precompiles if available.
2. Add one custom educational precompile only if it teaches a clear runtime
   boundary.
3. Charge weight for every precompile path.
4. Test malformed input, insufficient gas, and permission failures.

Learner checkpoint:

- A Solidity contract calls the precompile successfully.
- Invalid input reverts or fails deterministically.
- The precompile cannot bypass runtime permissions.

### Milestone 6: Verify End-to-End Compatibility

1. Run a local node.
2. Fund a development EVM account.
3. Deploy a Solidity storage contract.
4. Call a write method and a read method.
5. Check the receipt, gas used, event logs, and account nonce.
6. Restart the node and confirm state persists as expected.
7. Compare Substrate block state with Ethereum RPC responses.

Learner checkpoint:

- Runtime state, Ethereum RPC state, and developer-tool output agree.
- The course names any RPC or Ethereum behavior that remains unsupported.

## Test Plan

The course should include tests or manual verification notes for:

- Contract deployment succeeds.
- Contract deployment fails when gas is too low.
- Contract call changes storage.
- Reverted call returns a failed execution result.
- Event logs are queryable by contract address and block range.
- Account nonce increments after accepted transactions.
- Wrong chain ID transaction fails.
- Insufficient balance transaction fails.
- Native balance and EVM balance behavior matches the documented account model.
- Precompile calls charge weight and reject malformed input.
- Hardhat or equivalent tooling can deploy through the Ethereum RPC endpoint.

## Common Pitfalls

- Saying "EVM-compatible" without stating which compatibility layer is present.
- Exposing EVM execution but omitting Ethereum JSON-RPC.
- Forgetting chain ID replay protection.
- Underpricing EVM execution when mapping gas to weight.
- Allowing precompiles to bypass runtime authorization.
- Treating SS58 and `H160` addresses as the same address format.
- Assuming Ethereum testing helpers work when the RPC method is unsupported.
- Promising Ethereum mainnet equivalence when block structure, finality,
  randomness, receipts, or RPC behavior differ.

## References

- [OpenGuild bounty issue #6](https://github.com/openguild-labs/open-contribution-bounty/issues/6)
- [Polkadot Developer Docs: Add Smart Contract Functionality](https://docs.polkadot.com/parachains/customize-runtime/add-smart-contract-functionality/)
- [Frontier Overview](https://polkadot-evm.github.io/frontier/overview.html)
- [Frontier GitHub repository](https://github.com/polkadot-evm/frontier)
- [Polkadot Developer Docs: Hardhat](https://docs.polkadot.com/smart-contracts/dev-environments/hardhat/)
- [Polkadot Utilities](https://docs.polkadot.com/tools/)
