# Writing Your Application

At its core, a rollup is a specialized blockchain that processes transactions from a data availability (DA) layer. The logic that determines how your rollup behaves is defined by two key components: the **runtime** and its **modules**.

The runtime is the orchestrator of your rollup. It receives serialized transactions from the DA layer, deserializes them, and routes them to the appropriate modules for execution. Think of it as the central nervous system that connects all your application logic together. The runtime defines which modules your rollup supports, how they interact with each other, and how the rollup's state is initialized at genesis.

Modules, on the other hand, contain the actual business logic of your application. Each module manages its own state and defines the operations (called "call messages") that users can perform. For example, you might have a token module for handling transfers, a governance module for voting, or a custom trading module for your specific use case. When a user wants to interact with your rollup, they send a call message targeting a specific module, and the runtime ensures it gets delivered and executed atomically.

The starter package already includes several production-ready modules like Bank (for token management), Sequencer Registry, Accounts, and Hyperlane (for cross-chain messaging). It also provides an Example Module that serves as a template you can modify. In this chapter, we'll walk through implementing a module from scratch, testing it thoroughly, and integrating it into your runtime.