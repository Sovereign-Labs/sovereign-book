# Writing Your Application

At its core, a rollup is a specialized blockchain that processes transactions from a data availability (DA) layer. The logic that determines how your rollup behaves is defined by two key components: the **runtime** and its **modules**.

The runtime is the orchestrator of your rollup. It receives serialized transactions from the DA layer, deserializes them, and routes them to the appropriate modules for execution. Think of it as the central nervous system that connects all your application logic together. The runtime defines which modules your rollup supports, how they interact with each other, and how the rollup's state is initialized at genesis.

Modules, on the other hand, contain the actual business logic of your application. Each module manages its own state and defines the operations (called "call messages") that users can perform. For example, you might have a token module for handling transfers, a governance module for voting, or a custom trading module for your specific use case. When a user wants to interact with your rollup, they send a call message targeting a specific module, and the runtime ensures it gets delivered and executed atomically.

The starter package already includes several production-ready modules like Bank (for token management), Sequencer Registry, Accounts, and Hyperlane (for cross-chain messaging). It also provides an Example Module that serves as a template you can modify. 

### Let's Begin

With this context in mind, we're ready to start building. This chapter will guide you through the complete journey of application development on the Sovereign SDK, from creating your first module to enabling user interactions and exploring advanced features.

Here is the path we'll take:

1.  [**Implementing a Module:**](3-1-implementing-a-module.md) First, we'll define your module's state and business logic—the heart of your application.
2.  [**Testing Your Module:**](3-2-testing-your-module.md) We'll then write robust tests to ensure your logic is correct and secure.
3.  [**Integrating Your Module:**](3-3-integrating-your-module.md) Next, you'll learn how to add your finished module into a live rollup runtime.
4.  [**Wallets and Accounts:**](3-4-signing-and-submitting-txs.md) With your module integrated, we'll explore how users can create accounts and sign transactions to interact with it.
5.  [**Advanced Topics:**](3-5-advanced.md) From there, we'll dive into powerful features like hooks and custom APIs to extend your module's capabilities.
6.  [**Performance:**](3-6-performance.md) You'll learn how to optimize your module for maximum throughput and efficiency.
7.  [**Prebuilt Modules:**](3-7-prebuilt-modules.md) Finally, we'll review the rich ecosystem of existing modules you can leverage to accelerate your development.

Let's dive in!