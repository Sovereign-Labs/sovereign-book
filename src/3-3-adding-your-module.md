# Adding Your Module to Your Runtime

Once you've built and tested your module, the final step is to integrate it into your rollup's runtime. The `sov-rollup-starter` that we've been working in already has the `ValueSetter` module integrated. This section will walk you through how it's integrated, so you can follow the same pattern for your own future modules.

## Step 1: Add Your Module as a Dependency

First, your module needs to be added as a dependency of the runtime crate. In the starter template, the `value-setter` is an example inside the same workspace, so we add it via a path dependency.

In the root `Cargo.toml`, you'll find it under [workspace.dependencies]:

```toml
# In sov-rollup-starter/Cargo.toml

[workspace.dependencies]
# ... existing dependencies ...

value-setter = { path = "examples/value-setter" }
```

Then, in the runtime crate's `Cargo.toml` (located at `crates/stf/Cargo.toml`), it's included as a dependency:

```toml
# In sov-rollup-starter/crates/stf/Cargo.toml

[dependencies]
# ... existing dependencies ...

value-setter = { workspace = true }

[features]
default = []
native = [
    # ... existing native features ...
    "value-setter/native",
]
```

## Step 2: Add Your Module to the Runtime Struct

The central piece of your rollup's logic is the [`Runtime` struct](fix-link), usually found in `crates/stf/src/runtime.rs`. This struct lists all the modules that compose your rollup. 

To integrate the `ValueSetter`, we simply add it as a new field to this struct. The derive macros (`#[derive(Genesis, DispatchCall, ...)]`) on the Runtime automatically generate the necessary boilerplate code for initialization, transaction dispatching, and message encoding.

```rust
// In sov-rollup-starter/crates/stf/src/runtime.rs

use your_module::YourModule;

#[derive(Genesis, Hooks, DispatchCall, Event, MessageCodec, RuntimeRestApi)]
pub struct Runtime<S: Spec> {
    pub bank: sov_bank::Bank<S>,
    pub accounts: sov_accounts::Accounts<S>,
    // ... other modules ...
    
    // Add the value_setter module here
    pub value_setter: ValueSetter<S>,
}
```

## Step 3: Configure Genesis State

When your rollup is first launched, it populates its initial state from a `genesis.json` file. You need to add a configuration entry for your module so the runtime knows how to initialize its state.

In the previous section, we defined this `Config` struct for the `ValueSetter`:
```rust
pub struct ValueSetterConfig<S: Spec> {
    pub initial_value: u32,
    pub admin: S::Address,
}
```
Now, we provide the corresponding values in `genesis.json`. The key in the JSON object (`"value_setter"`) must exactly match the snake-cased field name in the `Runtime` struct.

```json
// In sov-rollup-starter/configs/{da-layer}/genesis.json

{
  "bank": { ... },
  "sequencer_registry": { ... },
  "accounts": { ... },
  "value_setter": {
    "admin": "sov1h6t82p7my6k02tcz9a27nnsyd9chg490rwn3r09e8k02y5qew2qsl2a4vg",
    "initial_value": 123
  }
}
```

## Step 4: Build and Run

With everything configured, you can build and run your rollup:

```bash
# From the root of sov-rollup-starter
cargo run --bin node -- run
```

Your rollup is now live with the `ValueSetter` module fully integrated! You can send it transactions, query its state via the REST API, and listen for its events.

## Your Module is Live!

Congratulations! You have successfully navigated the complete development lifecycle, from implementation and testing to deployment on your local machine.

You've built the core logic, but now the crucial question is: how do users actually interact with it? How do they create accounts, manage keys, and sign transactions to call your new module's methods?

The next section, **"Wallets and Accounts,"** will bridge this gap. We'll explore how to leverage the SDK's Ethereum-compatible account system and use client-side tooling to sign and submit transactions to your rollup, bringing your application to life for end-users.