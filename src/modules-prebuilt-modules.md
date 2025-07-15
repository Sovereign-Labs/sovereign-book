# Prebuilt Modules

The starter rollup includes many prebuilt modules that handle everything 
from token management, sequencer or prover incentives, paymaster logic, 
to transaction deduplication or account management. 

Here's a comprehensive list of all existing modules: 

Most of the modules below come out of the box in your rollup's runtime. You can find their implementations [here](fix-link).

### User Facing Modules

**sov-bank** - Token management module for
creating, transferring, and burning tokens
with unique addresses and names

**sov-chain-state** - Provides access to
blockchain state including block height, hash,
and general chain information

**sov-paymaster** - Enables third-party gas
sponsorship with per-sequencer payer
configuration

**sov-evm** - Full EVM compatibility layer that
processes RLP-encoded Ethereum transactions
and provides standard Ethereum endpoints

### Cross-Chain Communication

**sov-hyperlane-mailbox** - Cross-chain
  messaging via Hyperlane protocol
- **Mailbox**: Sends and receives cross-chain
  messages
- **MerkleTreeHook**: Computes merkle root of
  sent messages
- **InterchainGasPaymaster**: Handles
  cross-chain fee payments to relayers
- **Warp**: Enables interchain token transfers
    - Supports validator announcements and
  multisig ISMs

### Core Modules

**sov-accounts** - Account management system
that automatically creates addresses for
first-time senders and manages
credential-to-address mappings

**sov-uniqueness** - Transaction deduplication
logic using either nonce-based (Ethereum-style) or
generation-based methods (for low-latency applications)

**sov-blob-storage** - Deferred blob storage
system implementing the BlobSelector rollup
capability (which enables soft-confirmations
without losing censorship resistance)


### Incentive & Economic Modules

**sov-attester-incentives** - Complete
attestation/challenge verification workflow
with bonding and rewards for optimistic
rollups

**sov-prover-incentives** - Prover
registration, proof validation, slashing, and
rewards distribution

**sov-sequencer-registry** - Manages sequencer
registration, slashing, and rewards

**sov-revenue-share** - Manages automated revenue sharing

### Development & Testing

**sov-synthetic-load** - Load testing module
exposing heavy transaction types for
performance testing

**sov-value-setter** - Simple testing module
for storing and retrieving a single value

**module-template** - Starter template
demonstrating proper module structure with
state-changing methods and queries
