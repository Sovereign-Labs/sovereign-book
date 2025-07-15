# Prebuilt Modules

Here's a comprehensive list of all existing modules: 

### User Facing Modules

[**sov-bank**](fix-link) - Token management module for
creating, transferring, and burning tokens
with unique addresses and names

[**sov-chain-state**](fix-link) - Provides access to
blockchain state including block height, hash,
and general chain information

[**sov-paymaster**](fix-link) - Enables third-party gas
sponsorship with per-sequencer payer
configuration, so that users don't need any gas 
tokens to start transacting on your rollup

[**sov-evm**](fix-link) - EVM compatibility layer that
processes RLP-encoded Ethereum transactions
and provides standard Ethereum endpoints

[**sov-svm**](fix-link) - SVM compatibility layer that
processes Solana transactions
and provides standard Solana endpoints (maintained by the [Termina](https://www.termina.technology/) team)

### Cross-Chain Communication

[**sov-hyperlane-mailbox**](fix-link) - All five of these modules are 
part of the Hyperlane (bridging) integration. They enable 
any Sovereign SDK rollup to bridge messages and tokens from
any EVM, SVM or Cosmos SDK chain.
- [**Mailbox**](fix-link): Sends and receives cross-chain
  messages
- [**MerkleTreeHook**](fix-link): Computes merkle root of
  sent messages
- [**InterchainGasPaymaster**](fix-link): Handles
  cross-chain fee payments to relayers
- [**Warp**](fix-link): Enables interchain token transfers
    - Supports validator announcements and
  multisig ISMs

### Core Modules

[**sov-accounts**](fix-link) - Account management system
that automatically creates addresses for
first-time senders and manages
credential-to-address mappings

[**sov-uniqueness**](fix-link) - Transaction deduplication
logic using either nonce-based (Ethereum-style) or
generation-based methods (for low-latency applications)

[**sov-blob-storage**](fix-link) - Deferred blob storage
system implementing the BlobSelector rollup
capability (which enables soft-confirmations
without losing censorship resistance)


### Incentive & Economic Modules

[**sov-attester-incentives**](fix-link) - Complete
attestation/challenge verification workflow
with bonding and rewards for optimistic
rollups

[**sov-prover-incentives**](fix-link) - Prover
registration, proof validation, slashing, and
rewards distribution

[**sov-sequencer-registry**](fix-link) - Manages sequencer
registration, slashing, and rewards

[**sov-revenue-share**](fix-link) - Manages automated revenue sharing

### Development & Testing

[**sov-synthetic-load**](fix-link) - Load testing module
exposing heavy transaction types for
performance testing

[**sov-value-setter**](fix-link) - Simple testing module
for storing and retrieving a single value

[**module-template**](fix-link) - Starter template
demonstrating proper module structure with
state-changing methods and queries
