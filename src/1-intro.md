# The Sovereign SDK Book

Welcome to the Sovereign SDK Book, your comprehensive guide to the industry's most flexible toolkit for building high-performance, real-time rollups.

We built this SDK to give developers, from solo builders to large teams, the power to create onchain applications that were previously impossible. 

With transaction confirmations under 2-5 milliseconds, the Sovereign SDK is fast enough to bring complex financial systems, like Central-Limit Orderbooks (CLOBs), fully on-chain.

Let's build the next Hyperliquid.

<img src="https://github.com/Sovereign-Labs/sovereign-sdk/blob/nightly/assets/banner.jpg?raw=true" style="border-radius: 10px">

## Why Build a Dedicated Rollup For Your Application?

For almost a decade, developers have been forced to build applications on shared, general-purpose blockchains. This model forces apps with vastly different needs to compete for the same limited blockspace. Building your application as a dedicated rollup gives you three strategic advantages:

1.  **Dedicated Throughput:** Your users will never have to compete with a viral NFT drop. A rollup gives your application its own dedicated lane, ensuring a consistently fast and affordable user experience.
2.  **Capturing More Value:** On shared blockchains, user fees primarily benefit the chain operators (i.e. L1 validators or general-purpose L2 sequencer). With a rollup, your application and its users can capture the vast majority of that value, creating a sustainable economic engine for your project.
3.  **Full Control & Flexibility:** Go beyond the limitations of a shared virtual machine. A rollup gives you full control over the execution environment, allowing you to define your own rules for how transactions are processed. **With a rollup, you're in the driver's seat.**

## Why Choose the Sovereign SDK?

The Sovereign SDK is designed around four key principles to provide an unmatched developer and user experience:

-   **Total Customization:** While rollups promise flexibility, existing frameworks are overly restrictive. Sovereign SDK delivers on that promise with its modular Rust runtime, empowering you to customize as much or as little as needed. Easily add custom fee logic, integrate tailored authenticators, prioritize specific transaction types, or even swap out the authenticated state store—all without wrestling with legacy code.
-   **Best-in-Class Performance:** With 2-5ms soft confirmations and throughput exceeding 10,000 TPS, the Sovereign SDK is orders of magnitude faster than competing frameworks like Orbit, the OP Stack, or the Cosmos SDK.
-   **A Developer-Friendly Experience:** Write your logic in standard Rust, run `cargo build`, and get a complete full-node implementation with REST & WebSocket APIs, an indexer, auto-generated OpenAPI specs, and a sequencer  with automatic failover out of the box. No boilerplate or deep blockchain expertise required.
-   **Future-Proof Architecture:** Never get locked into yesterday's tech stack. With the Sovereign SDK, you can switch data availability layers or zkVMs with just a few lines of code, ensuring your project remains agile for years to come.


## How It Works

As a developer, you write your rollup's business logic in Rust, and the SDK handles the complexity of creating a complete, production-ready node implementation.

The magic happens in two stages: **real-time execution** and **on-chain settlement**.

1.  **Real-Time Execution (Soft Confirmations):** Users send transactions to a **sequencer**. The sequencer executes these transactions instantly (typically in under 2-5ms) and returns a "soft confirmation" back to the user. This provides a real-time user experience that feels like a traditional web application.

2.  **On-Chain Settlement & Verification:** Periodically, the sequencer batches thousands of these transactions and posts them to an underlying **Data Availability (DA) layer** like Celestia. From this point, the rest of the network—the full nodes—can read the ordered data and execute the transactions to independently verify the new state of the rollup.

Finally, specialized actors called **provers** (in zk-rollup mode) or **attesters** (in optimistic-rollup mode) generate cryptographic proofs  or attestations that the state was computed correctly. These are posted back to the DA layer, allowing light clients and bridges to securely verify the rollup's state without having to re-execute every transaction.

This two-stage process gives you the best of both worlds: the instant, centralized execution needed for high-performance applications, combined with the censorship-resistance and trust-minimized verification of a traditional blockchain.

## Ready to Build?

Now that you understand the power and flexibility of the Sovereign SDK, you're ready to get your hands dirty. In the next chapter, **"Getting Started,"** we'll walk you through cloning a starter repository and running your first rollup in minutes.