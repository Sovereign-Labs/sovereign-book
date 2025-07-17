# Adding Your Module to Your Runtime

Once you've built and tested your module, the final step is to integrate it into your rollup runtime. This section will walk you through the process of adding your module to a new rollup project based on our rollup starter template.

## Step 1: Add Your Module as a Dependency

First, add your module to the workspace dependencies in the root `Cargo.toml`:

```toml
[workspace.dependencies]
# ... existing dependencies ...

# Add your module here
your-module = { path = "../path/to/your-module" }
# Or if published:
# your-module = { version = "0.1.0" }
# Or if the module is available on Github:
# your-module = { git = "https://github.com/your-github/your-module", rev = "dfd0624c32f5fb363c2190e9d911605663f7d693" }
```

Then add it to your STF crate's dependencies in `crates/stf/Cargo.toml` (where your `Runtime` is typically located in):

```toml
[dependencies]
# ... existing dependencies ...

your-module = { workspace = true }

[features]
default = []
native = [
    # ... existing native features ...
    "your-module/native",
]
```

## Step 2: Add Your Module to the Runtime Struct

The central piece of your rollup's logic is the [`Runtime` struct](fix-link), usually found in `crates/stf/src/runtime.rs`. This struct lists all the modules that compose your rollup. To integrate your module, simply add it as a new field to this struct.

The Runtime struct uses several derive macros (`#[derive(Genesis, DispatchCall, ...)]`) that automatically generate the boilerplate code for state initialization, transaction dispatching, and message encoding.

```rust
use your_module::YourModule;

#[derive(Genesis, Hooks, DispatchCall, Event, MessageCodec, RuntimeRestApi)]
pub struct Runtime<S: Spec> {
    /// The bank module is responsible for managing tokens
    pub bank: sov_bank::Bank<S>,
    
    /// The accounts module manages user accounts and addresses
    pub accounts: sov_accounts::Accounts<S>,
    
    // ... other modules ...
    
    /// Your custom module
    pub your_module: YourModule<S>,
}
```

## Step 3: Configure Genesis State

When your rollup is first launched, it populates its initial state from a `genesis.json` file. You need to tell the rollup how to initialize your module by adding a corresponding entry to this file.

### Understanding `genesis.json`

The `genesis.json` is a simple key-value store where each key is the snake-case name of a module in your `Runtime` struct, and the value is the initial configuration for that module.

When the rollup starts, it deserializes this JSON into your module's `Config` struct (which you define in your module) and passes it to your module's `genesis()` method.

You will find this file in the root of your rollup project: `your-rollup/{DA_LAYER_NAME}/genesis.json`.

### Adding Your Module's Configuration

There are two cases:

**1. Your Module Requires No Initial Configuration:**

If your module's `Config` is an empty struct (e.g., `pub struct MyModuleConfig {}`) or can be created with `Default::default()`, you just need to add its name to `genesis.json` with an empty JSON object `{}`.

```json
// In your-rollup/genesis.json
{
  "bank": { ... },
  "sequencer_registry": { ... },
  "accounts": { ... },
  "my_awesome_module": {}
}
```

**2. Your Module Requires Initial Configuration:**

If your module needs initial parameters (like an admin address or an initial value), you must provide them in the JSON object. The JSON fields must exactly match the fields of your module's `Config` struct.

For example, if your module has this `Config` struct:
```rust
// In modules/my-awesome-module/src/lib.rs
#[derive(serde::Deserialize, serde::Serialize)]
pub struct MyAwesomeModuleConfig {
    pub admin_address: S::Address,
    pub initial_counter: u64,
}
```

Your `genesis.json` entry would look like this:
```json
// In your-rollup/genesis.json
{
  // ... other modules
  "my_awesome_module": {
    "admin_address": "0x633dD354F65261d7a64E10459508F8713a537149",
    "initial_counter": 100
  }
}
```

## Step 4: Configure Rollup Constant

The `constants.toml` file in your rollup's root directory allows you to configure chain-level parameters that don't change often. You should update this file to reflect your rollup's identity.

```toml
# Change these to make your chain unique
CHAIN_ID = 12345  # Your unique chain ID
CHAIN_NAME = "my-awesome-rollup"

# Gas configuration
GAS_TOKEN_NAME = "GAS"

# Other compile time parameters...
```
These values are compiled into your rollup binary.

## Step 5: Build and Run

With everything configured, you can run your rollup with your module:

```bash
# Run the node
cargo run --bin node
```

Your rollup is now operational! You can:
- Send transactions to your module
- Query its state via the REST API
- See events in transaction receipts

## Troubleshooting

### Common Issues

**Module not found in genesis**
- Ensure the module name in `genesis.json` matches the field name in your Runtime struct

**Serialization errors**
- Verify your genesis configuration matches your module's `Config` type
- Check that all addresses use the correct format (0x-prefixed hex)

**Build errors**
- Ensure all feature flags are properly configured
- Check that your module exports all required types

### Your Module is Live!

Congratulations! Your module is now a fully integrated part of a running rollup. You have successfully navigated the complete development lifecycle, from implementation and testing to deployment on your local machine.

You've built the core logic, but now the crucial question is: how do users actually interact with it? How do they create accounts, manage keys, and sign transactions to call your new module's methods?

The next section, "Wallets and Accounts," will bridge this gap. We'll explore how to leverage the SDK's Ethereum-compatible account system and use client-side tooling to sign and submit transactions to your rollup, bringing your application to life for end-users.