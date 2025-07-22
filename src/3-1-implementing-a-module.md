# Anatomy of a Module

As we begin our journey into building a production-ready module, the first step is to understand the two most important architectural concepts in the Sovereign SDK: the **Runtime** and its **Modules**.

### Runtime vs. Modules

The **runtime** is the orchestrator of your rollup. It receives transactions, deserializes them, and routes them to the appropriate modules for execution. Think of it as the central nervous system that connects all your application logic. The `Runtime` struct you define in your rollup code is what specifies which modules are included.

**Modules**, on the other hand, contain the actual, isolated business-logic. Each module manages its own state and defines the specific actions (called "call messages") that users can perform. In our case, `ValueSetter` is a module. When a user wants to interact with it, they send a `CallMessage` targeting the `value_setter` module, and the runtime ensures it gets delivered and executed atomically.

Now that we understand this high-level structure, let's dissect the production-ready version of the `ValueSetter` module piece by piece.

---

## The Module Struct: State and Dependencies

The entry point to any module is a struct that defines its state variables and its dependencies on other modules. This struct should derive `ModuleInfo` and, optionally, `ModuleRestApi`.

```rust
#[derive(Clone, ModuleInfo, ModuleRestApi)]
pub struct ValueSetter<S: Spec> {
    /// The `#[id]` attribute is required and uniquely identifies the module instance.
    #[id]
    pub id: ModuleId,

    /// The `#[state]` attribute marks a field as a state variable.
    /// `StateValue` stores a single, typed value.
    #[state]
    pub value: StateValue<u32>,

    /// We'll also store the address of the admin who is allowed to change the value.
    #[state]
    pub admin: StateValue<S::Address>,
}
```

This struct is defined by several key attributes and the `Spec` generic:

*   **`#[derive(ModuleInfo)]`**: This derive macro is mandatory. It performs essential setup, like laying out your state values in the database. If you forget to derive this trait, the SDK will throw a helpful error.

*   **`#[id]`**: Every module must have exactly one field with this attribute. The SDK uses it to store the module's unique, auto-generated identifier.

*   **`#[state]`**: This attribute marks a field as a state variable that will be stored in the database. We will cover the different state types below.

*   **`#[module]`**: While not used in this example, this attribute declares a dependency on another module. For example, if our `ValueSetter` needed to charge a fee, we could add `#[module] pub bank: sov_bank::Bank<S>`, allowing us to call methods like `self.bank.transfer(...)` from our own logic.

### The `ModuleRestApi` Trait

Deriving the `ModuleRestApi` trait is optional but highly recommended. It automatically generates RESTful API endpoints for the `#[state]` items in your module. Each item's endpoint will have the name `{hostname}/modules/{module-name}/{field-name}/`, with all items automatically converted to `kebab-casing`. For example, for the `value` field in our `ValueSetter` module, the SDK generates an endpoint at the path `/modules/value-setter/value`.

Note that `ModuleRestApi` can't always generate endpoints for you. If it can't figure out how to generate an endpoint for a particular state value, it will simply skip it by default. If you want to override this behavior and throw a compiler error if endpoint generation fails, you can add the `#[rest_api(include)]` attribute.

### The `Spec` Generic

All modules are generic over a `Spec` type. This `Spec` provides all the core types for the rollup, such as `S::Address` and `S::Gas`. By parameterizing your module with `S`, you ensure it remains portable and can be used in any Sovereign SDK rollup, regardless of its specific address format, DA layer, signature type or other configuration.

> **A Quick Tip on Schemas and Generics**
> If you parameterize your `CallMessage` or `Event` over `S` (for example, to include an address of type `S::Address`), you must add the `#[schemars(bound = "S: Spec")]` attribute on top your enum definition. This is a necessary hint for `schemars`, a library that generates a JSON schema for your module's API. It ensures that your generic types can be correctly represented for external tools.

## State Management In-Depth

The SDK provides several "state" types for different use cases, all designed to be stored within your module struct using the `#[state]` attribute.

*   **`StateValue<T>`**: Stores a single item of type `T`. We use this for the `value` and `admin` variables in our example.
*   **`StateMap<K, V>`**: Stores a key-value mapping. Ideal for balances or other user-specific data.
*   **`StateVec<T>`**: Stores an ordered list of items, accessible by index.

The generic types can be any deterministic Rust data structure, anything from a simple `u32` to a complex `BTreeMap`.

**Accessory State**: For each state type, there is a corresponding `AccessoryState*` variant (e.g., `AccessoryStateMap`). Accessory state is special: it can be read via the API, but it is **write-only** during a transaction. This makes it much cheaper to use for data that doesn't affect onchain logic, like indexing purchase histories for an off-chain frontend.

## The `Module` Trait: Defining Your Logic

The `Module` trait is where your business logic lives. Your module struct must implement this trait, which defines its configuration, public methods and events.

Let's look at a complete, production-ready implementation for `ValueSetter` and break down its components.

```rust
// First, we define the data structures our module will use.
use sov_modules_api::macros::serialize;
use sov_modules_api::{Module, Context, CallResponse, GenesisState, TxState, EventEmitter, Spec, UniversalWallet};
use anyhow::Result;
use schemars::JsonSchema;

// The configuration for our module.
#[derive(Clone, Debug, PartialEq, Eq)]
#[serialize(Borsh, Serde)]
#[serde(rename_all = "snake_case")]
pub struct ValueSetterConfig<S: Spec> {
    pub admin: S::Address,
}

// The actions a user can take on our module. Our module only supports one action: setting the value.
#[derive(Clone, Debug, PartialEq, Eq, JsonSchema, UniversalWallet)]
#[serialize(Borsh, Serde)]
#[serde(rename_all = "snake_case")]
pub enum CallMessage {
    SetValue(u32),
}

// The events our module can emit.
#[derive(Clone, Debug, PartialEq, Eq, JsonSchema)]
#[serialize(Borsh, Serde)]
#[serde(rename_all = "snake_case")]
pub enum Event {
    ValueUpdated(u32),
}

// Now, we implement the `Module` trait for our `ValueSetter` struct.
impl<S: Spec> Module for ValueSetter<S> {
    type Spec = S;
    type Config = ValueSetterConfig<S>;
    type CallMessage = CallMessage;
    type Event = Event;

    // genesis() is called when the rollup is deployed to initialize state.
    fn genesis(&mut self, _header: &<S::Da as sov_modules_api::DaSpec>::BlockHeader, config: &Self::Config, state: &mut impl GenesisState<S>) -> Result<()> {
        self.admin.set(&config.admin, state)?;
        Ok(())
    }

    // call() is the entrypoint for user transactions.
    fn call(&mut self, msg: Self::CallMessage, context: &Context<S>, state: &mut impl TxState<S>) -> Result<()> {
        match msg {
            CallMessage::SetValue(new_value) => {
                let admin = self.admin.get(state)??;
                anyhow::ensure!(admin == *context.sender(), "Only the admin can set the value.");

                // Update the `value` in our state
                self.value.set(&new_value, state)?;

                // Emit an event to record this change
                self.emit_event(state, Event::ValueUpdated(new_value));

                Ok()
            }
        }
    }
}
```

Let's break this down.

### Associated Types: `Config`, `CallMessage`, and `Event`

*   **`type Config`**: This struct defines the initial configuration for your module. The SDK reads the relevant section for your module from the `genesis.json` file and automatically deserializes it into this `Config` struct when the rollup is first deployed.

*   **`type CallMessage`**: This is the public API of your module, representing the actions a user can take. An `enum` is usually the best choice. You must derive all the necessary serialization traits (our `#[serialize(Borsh, Serde)]` macro is a convenient helper), as well as `schemars::JsonSchema` and `sov_modules_api::UniversalWallet`. This ensures your `CallMessage` is portable across different languages and frontends.

*   **`type Event`**: This `enum` defines the events your module can emit to notify off-chain systems of state changes.

### Methods: `genesis()` and `call()`

*   **`fn genesis(...)`**: This method is called exactly once, when the rollup is launched. It uses the `Config` struct to set the initial state. In our example, it sets the `admin` address.

*   **`fn call(...)`**: This is the most important method. It's the entry point for all user transactions. The runtime deserializes a user's transaction into the `CallMessage` you defined for this module and passes it to this method. It also provides a `Context`, which contains important metadata like the sender's address (`context.sender()`).

> **A Note on Gas and Security**
> Just like Ethereum smart contracts, modules accept inputs that are pre-validated by the chain. Your `call` method does not need to worry about authenticating the transaction sender. The SDK also automatically meters gas for state accesses. You only need to manually charge gas (using `state.charge_gas(...)`) if your module performs heavy computation outside of state reads/writes.

## Events: Bridging On-Chain and Off-Chain Data

While `call` methods alter your module's state, **events** are how your module broadcasts those changes. They are the primary mechanism for streaming on-chain data to off-chain systems like indexers and front-ends in real-time.

A key guarantee of the Sovereign SDK is that event emission is **atomic** with transaction execution—if a transaction reverts, so do its events. This ensures any off-chain system remains perfectly consistent with the on-chain state. The sequencer provides a robust websocket endpoint that streams sequentially numbered transactions along with their corresponding events. If a client disconnects, it can reliably resume the stream from the last transaction it processed, making it simple to build scalable and fault-tolerant off-chain data pipelines.

To implement events, you define an `Event` enum (as shown in the full example above), then emit it from your `call` method using `self.emit_event(state, <your_event_variant>)`.

## Error Handling

Your module's `call` method must return a `anyhow::Result`. This is the primary mechanism for handling expected failures, like a user providing invalid input or having an insufficient balance.

When your `call` method returns an `Err`, the Sovereign SDK guarantees that all state changes made during that transaction are automatically reverted. This ensures your module's logic is always atomic—it either succeeds completely or has no effect at all.

Our `ValueSetter` example demonstrates the most common way to return an error:

```rust
let admin = self.admin.get(state)??;
anyhow::ensure!(admin == *context.sender(), "Only the admin can set the value.");
```

The `anyhow::ensure!` macro checks a condition and, if it's false, returns an `Err` with the provided message.

For a more comprehensive guide on error handling strategies, including the distinction between recoverable user errors and critical system bugs, see the **[Error Handling](3-5-advanced.md#error-handling)** section in Advanced Topics.

## Next Step: Ensuring Correctness

You now have a deep, conceptual understanding of how a Sovereign SDK module is structured and implemented. With this foundation, you're ready to learn how to prove its correctness.

In the next section, **"Testing Your Module,"** we'll show you how to use the SDK's powerful testing framework to write comprehensive tests.