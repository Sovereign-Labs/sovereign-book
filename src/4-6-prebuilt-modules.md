# Prebuilt Modules

Here are the suite of prebuilt, production-tested modules that handle common blockchain primitives. Leveraging these modules allows you to focus on your application's unique logic instead of reinventing the wheel.

This page serves as a reference guide to the standard modules included with the SDK.

| Module                  | Crate Link                                      | Description                                                                                                                                     |
| ----------------------- | ----------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------- |
| **User-Facing**         |                                                 | _Modules that directly provide user-facing application logic._                                                                                  |
| Bank                    | [`sov-bank`](fix-link)                          | Creates and manages fungible tokens. Handles minting, transfers, and burning.                                                                   |
| Paymaster               | [`sov-paymaster`](fix-link)                     | Enables gas sponsorship (meta-transactions), allowing users to transact without needing to hold gas tokens.                                     |
| Chain State             | [`sov-chain-state`](fix-link)                   | Provides on-chain access to block metadata like the current block height and hash.                                                              |
| EVM                     | [`sov-evm`](fix-link)                           | An EVM compatibility layer that executes standard, RLP-encoded Ethereum transactions.                                                           |
| SVM                     | [`sov-svm`](fix-link)                           | A Solana VM compatibility layer that executes standard Solana transactions (maintained by the [Termina](https://www.termina.technology/) team). |
| **Core Infrastructure** |                                                 | _Modules that provide fundamental, system-level capabilities._                                                                                  |
| Accounts                | [`sov-accounts`](fix-link)                      | Manages user accounts, public keys, and nonces.                                                                                                 |
| Uniqueness              | [`sov-uniqueness`](fix-link)                    | Provides transaction deduplication logic using either nonces (Ethereum-style) or generation numbers (for low-latency applications).               |
| Blob Storage            | [`sov-blob-storage`](fix-link)                  | A deferred blob storage system that enables soft-confirmations without losing censorship resistance.                                            |
| **Bridging**            |                                                 | _Modules for interoperability with other blockchains._                                                                                          |
| Hyperlane Bridge        | [`sov-hyperlane-mailbox`](fix-link)             | An integration with the Hyperlane interoperability protocol, enabling messaging and token bridging to other blockchains (EVM, SVM, Cosmos).      |
| **Rollup Economics**    |                                                 | _Modules for managing incentives and fee distribution._                                                                                         |
| Sequencer Registry      | [`sov-sequencer-registry`](fix-link)            | Manages sequencer registration, bonding, and rewards distribution (for decentralized sequencing).                                               |
| Prover Incentives       | [`sov-prover-incentives`](fix-link)             | Manages prover registration, proof validation, and rewards distribution.                                                                        |
| Attester Incentives     | [`sov-attester-incentives`](fix-link)           | Manages the attestation/challenge process for optimistic rollups, including bonding and rewards.                                                |
| Revenue Share           | [`sov-revenue-share`](fix-link)                 | Automates on-chain fee sharing for the use of premium SDK components, such as the low-latency sequencer.                                                                      |
| **Development & Testing** |                                                 | _Helper modules for the development and testing lifecycle._                                                                                     |
| Value Setter            | [`sov-value-setter`](fix-link)                  | A minimal example module used throughout the documentation for teaching purposes.                                                               |
| Synthetic Load          | [`sov-synthetic-load`](fix-link)                | A utility module for generating heavy transactions to assist with performance testing and benchmarking.                                         |
| Module Template         | [`module-template`](fix-link)                   | A starter template demonstrating best practices for module structure, including state, calls, and events.                                       |

## Next Steps

While this section of the book covered the core development workflow, the SDK includes many more production-grade features. 

The next section, **"Additional Capabilities,"** provides a high-level overview of these features and how we can help you integrate them.