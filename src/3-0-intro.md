# Building for Production

Having completed the quickstart, you now understand the fundamental developer loop: write, integrate, and interact. You've seen how easy it is to get custom logic running on a Sovereign rollup.

Now, let's establish the key concepts that make it all work.

## Runtime vs. Modules

At its core, a rollup is a specialized blockchain that processes transactions from a data availability (DA) layer. The logic that determines how 
your rollup behaves is defined by two key components: the **runtime** and its **modules**. The **runtime** is the orchestrator of your rollup. It receives serialized transactions from the DA layer, deserializes them, and routes them to 
the appropriate modules for execution. Think of it as the central nervous system that connects all your application logic together. The runtime 
defines which modules your rollup supports, how they interact with each other, and how the rollup's state is initialized at genesis.

**Modules**, on the other hand, contain the actual business logic of your application. Each module manages its own state and defines the operations (called "call messages") that users can perform. For example, you might have a token module for handling transfers, a governance module for voting, or a custom trading module for your specific use case. When a user wants to interact with your rollup, they send a call message targeting a specific module, and the runtime ensures it gets delivered and executed atomically.

### From Basics to Production-Ready

This chapter takes you beyond the basics. We'll transform the simple `ValueSetter` module into a production-ready component by diving deep into the features and best practices that ensure your application is robust, secure, and feature-rich.

Here's what we'll cover:

1.  [**Anatomy of a Module:**](3-1-implementing-a-module.md) A deep dive into the `Module` trait, state management, events, and error handling patterns.
2.  [**Testing Your Module:**](3-2-testing-your-module.md) How to write comprehensive tests using the SDK's powerful testing framework.
3.  [**Wallets and Accounts:**](3-4-signing-and-submitting-txs.md) A closer look at how users create accounts and sign transactions.
4.  [**Advanced Topics:**](3-5-advanced.md) Exploring powerful features like hooks, custom APIs, and MEV mitigation.
5.  [**Performance:**](3-6-performance.md) You'll learn how to optimize your module for maximum throughput and efficiency.
6.  [**Prebuilt Modules:**](3-7-prebuilt-modules.md) Finally, we'll review the rich ecosystem of existing modules you can leverage to accelerate your development.

Let's begin.