# Anatomy of a Module

As we begin our journey into building a production-ready rollup, the first step is to understand the two most important architectural concepts in the Sovereign SDK: the **Runtime** and its **Modules**.

## Runtime vs. Modules

The **runtime** is the orchestrator of your rollup. It receives transactions, deserializes them, and routes them to the appropriate modules for execution. Think of it as the central nervous system that connects all your application logic. The [`Runtime`](https://github.com/Sovereign-Labs/sov-rollup-starter-wip/blob/main/crates/stf/stf-declaration/src/lib.rs#L51) struct you define in your rollup code specifies which modules are included.

**Modules** contain the actual business-logic. Each module manages its own state and defines the specific actions (called "call messages") that users can perform. Modules are usually small and self-contained, but they can contain dependencies on other modules when it 
makes sense to.

Now that we understand this high-level structure, let's dissect the `ValueSetter` module you built and enhance it with production-grade features.

## Dissecting the `ValueSetter` Module

#### The Module Struct: State and Dependencies

First, let's look at the `ValueSetter` struct, which defined its state variables and its dependencies on other modules.

```rust
#[derive(Clone, ModuleInfo, ModuleRestApi)]
pub struct ValueSetter<S: Spec> {
    #[id]
    pub id: ModuleId,

    #[state]
    pub value: StateValue<u32>,

    #[state]
    pub admin: StateValue<S::Address>,
}
```
This struct is defined by several key attributes and the `Spec` generic:

*   `#[derive(ModuleInfo)]`: This derive macro is mandatory. It performs essential setup, like laying out your state values in the database.
*   `#[id]`: Every module must have exactly one field with this attribute. The SDK uses it to store the module's unique, auto-generated identifier.
*   `#[state]`: This attribute marks a field as a state variable that will be stored in the database. More on [state management](#state-management-in-depth) later.
*   **The** `Spec` **Generic**: All modules are generic over a `Spec`. This provides core types like `S::Address` and makes your module portable across things like DA layers, zkVMs, and address formats.
*   `#[module]`: While not used in this example, this attribute declares a dependency on another module. For example, if our `ValueSetter` needed to charge a fee, we could add `#[module] pub bank: sov_bank::Bank<S>`, allowing us to call methods like `self.bank.transfer(...)` from our own logic.

#### The `ModuleRestApi` Trait

Deriving the `ModuleRestApi` trait is optional but highly recommended. It automatically generates RESTful API endpoints for the `#[state]` items in your module. Each item's endpoint will have the name `{hostname}/modules/{module-name}/{field-name}/`, with all items automatically converted to `kebab-casing`. For example, for the `value` field in our `ValueSetter` module, the SDK generates an endpoint at the path `/modules/value-setter/value`.

Note that `ModuleRestApi` can't always generate endpoints for you. If it can't figure out how to generate an endpoint for a particular state value, it will simply skip it by default. If you want to override this behavior and throw a compiler error if endpoint generation fails, you can add the `#[rest_api(include)]` attribute.

### State Management In-Depth

The SDK provides several "state" types for different use cases. All three types of state can be added to your module struct using the `#[state]` attribute.

*   `StateValue<T>`: Stores a single item of type `T`. We used this for the `value` and `admin` variables in our example.
*   `StateMap<K, V>`: Stores a key-value mapping. This is ideal for balances or other user-specific data.
*   `StateVec<T>`: Stores an ordered list of items, accessible by index.

The generic types can be any (deterministically) serializable Rust data structure.

**Accessory State**: For each state type, there is a corresponding `AccessoryState*` variant (e.g., `AccessoryStateMap`). Accessory state is special: it can be read via the API, but it is **write-only** during transaction execution. This makes it a simple and cheap storage to use for data that doesn't affect onchain logic, like purchase histories for an off-chain frontend.

### The `Module` Trait

The `Module` trait is where your business logic lives. Let's review the pieces you implemented for `ValueSetter` in the quickstart.

*   **`type Config` and `fn genesis()`**: You created a `ValueSetterConfig` and used it in the `genesis` method to initialize the `admin` state. This is a standard pattern: `Config` defines the initial data, read from `genesis.json`, and `genesis()` applies it to the module's state when the rollup is first deployed.

*   **`type CallMessage` and `fn call()`**: You defined a `CallMessage` enum for the public `SetValue` action. This enum is the public API of your module, representing the actions a user can take. The `call()` method is the entry point for these actions. The runtime passes in the `CallMessage` and a `Context` containing metadata like the sender's address, which you used for the admin check.

*   **Error Handling**: In your `call` method, you used `anyhow::ensure!` to handle a **user error** (an invalid sender). When a `call` method returns an `Err`, the SDK guarantees that all state changes are automatically reverted, ensuring atomicity. This `Result`-based approach is for predictable user errors, while unrecoverable system bugs should cause a `panic!`. A more detailed guide is available in the [`Advanced Topics`](./4-4-advanced-topics.md) section.

> **A Quick Tip on Parametrizing Your Types Over S**
>
> If you parameterize your `CallMessage` or `Event` over `S` (for example, to include an address of type `S::Address`), you must add the `#[schemars(bound = "S: Spec", rename = "MyEnum")]` attribute on top your enum definition. This is a necessary hint for `schemars`, a library that generates a JSON schema for your module's API. It ensures that your generic types can be correctly represented for external tools.

> **Quick Tip: Handling `Vector` and `String` in CallMessage** 
> Use the fixed‑size wrappers `SafeVector` and `SafeString` for any fields that are deserialized directly into a `CallMessage`; they limit payload size and prevent DoS attacks. After deserialization, feel free to convert them to regular `Vector` and `String` values and use them as usual.

## Adding Events

Your `ValueSetter` module works, but it's a "black box." Off-chain applications have no way of knowing when the value changes without constantly polling the API. To solve this, we introduce **Events**.

Events are the primary mechanism for streaming on-chain data to off-chain systems like indexers and front-ends in real-time. Let's add one to our module.

First, define an `Event` enum.

```rust
// In examples/value-setter/src/lib.rs

#[derive(Clone, Debug, PartialEq, Eq, JsonSchema)]
#[serialize(Borsh, Serde)]
#[serde(rename_all = "snake_case")]
pub enum Event {
    ValueUpdated(u32),
}
```

Next, update your `Module` implementation to use this new `Event` type and emit it from the `call` method.

```rust
// In examples/value-setter/src/lib.rs

impl<S: Spec> Module for ValueSetter<S> {
    type Spec = S;
    type Config = ValueSetterConfig<S>;
    type CallMessage = CallMessage;
    type Event = Event; // Change this from ()

    // The `genesis` method is unchanged.
    fn genesis(&mut self, _header: &<S::Da as sov_modules_api::DaSpec>::BlockHeader, config: &Self::Config, state: &mut impl GenesisState<S>) -> Result<()> {
        // ...
    }

    fn call(&mut self, msg: Self::CallMessage, context: &Context<S>, state: &mut impl TxState<S>) -> Result<()> {
        match msg {
            CallMessage::SetValue(new_value) => {
                let admin = self.admin.get(state)??;
                anyhow::ensure!(admin == *context.sender(), "Only the admin can set the value.");

                self.value.set(&new_value, state)?;

                // NEW: Emit an event to record this change.
                self.emit_event(state, Event::ValueUpdated(new_value));

                Ok(())
            }
        }
    }
}
```

Now, whenever the admin successfully calls `set_value`, the module will emit a `ValueUpdated` event.

A key guarantee of the Sovereign SDK is that event emission is **atomic** with transaction execution—if a transaction reverts, so do its events. This ensures any off-chain system remains consistent with the on-chain state. 

To make it simple to build scalable and faul-tolertant off-chain data pipelines, the sequencer provides a websocket endpoint that streams sequentially numbered transactions along with their corresponding events. If a client disconnects, it can reliably resume the stream from the last transaction it processed.

## Next Step: Ensuring Correctness

You now have a strong conceptual understanding of how a Sovereign SDK module is structured.

In the next chapter, **"Testing Your Module,"** we'll show you how to test your modules.
