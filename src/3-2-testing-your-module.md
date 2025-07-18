# Testing Your Module

Testing is crucial for building reliable modules. The SDK provides a comprehensive testing framework that makes it easy to write thorough tests for your modules.

## Test Infrastructure Overview

There are several key components for testing:

- **TestRunner**: A stateful test harness that manages your runtime environment  
- **TestUser**: Test accounts with preconfigured balances
- **TransactionTestCase**: A structured way to define test scenarios with assertions
- **Runtime Generation Macros**: Automatically include all core modules

## Setting Up Your Test Environment

### 1. Create Your Test Runtime

The `generate_optimistic_runtime!` macro automatically includes all core modules (Bank, Accounts, SequencerRegistry, etc.), so you only need to add your custom modules:

```rust
use sov_test_utils::runtime::optimistic::generate_optimistic_runtime;

// Your module's crate
use your_module::YourModule;

// Generate a runtime with core modules + your custom module
generate_optimistic_runtime!(
    TestRuntime <=  // Your test runtime name
    your_module: YourModule<S> // Your custom module
);
```

### 2. Define Your Genesis Configuration

Set up the initial state with test users:

```rust
use sov_test_utils::{HighLevelOptimisticGenesisConfig, TestRunner, TestUser};

pub struct TestData<S: Spec> {
    pub admin: TestUser<S>,
    pub user1: TestUser<S>,
    pub user2: TestUser<S>,
}

pub fn setup() -> (TestData<TestSpec>, TestRunner<TestRuntime, TestSpec>) {
    let genesis_config = HighLevelOptimisticGenesisConfig::generate()
        .add_accounts_with_default_balance(3);
    
    let mut users = genesis_config.additional_accounts().to_vec();
    let test_data = TestData {
        user2: users.pop().unwrap(),
        user1: users.pop().unwrap(),
        admin: users.pop().unwrap(),
    };
    
    let runner = TestRunner::new_with_genesis(/* ... */);
    
    (test_data, runner)
}
```

## Writing Your First Test

Here's a complete example testing a simple module operation:

```rust
#[cfg(test)]
mod tests {
    use super::*;
    use sov_test_utils::{TestRunner, TransactionTestCase};
    
    #[test]
    fn test_module_operation() {
        // Setup runner and get test user 
        let (test_data, mut runner) = setup();
        let user = &test_data.user1;
        
        // Execute a transaction
        runner.execute_transaction(TransactionTestCase {
            input: user.create_plain_message::<TestRuntime, YourModule>(
                CallMessage::SetValue { value: 42 }
            ),
            assert: Box::new(|result, state| {
                // Verify the transaction succeeded
                assert!(result.tx_receipt.is_successful());
                
                // Query and verify state
                let current_value = YourModule::default()
                    .get_value(state)
                    .unwrap_infallible()  // State access can't fail in tests
                    .unwrap();            // Handle the Option
                assert_eq!(current_value, 42);
            }),
        });
    }
}
```

## Running Your Tests

Execute your tests from your module's root directory using standard Rust commands:

```bash
# Navigate to your module directory 
cd your-module/

# Run all tests in your module
cargo test

# Run specific test
cargo test test_module_operation

# Run with output for debugging
cargo test -- --nocapture
```

## Testing Patterns

### 1. Error Scenario Testing

Test that your module handles errors correctly:

```rust
#[test]
fn test_insufficient_balance() {
    let (test_data, mut runner) = setup();
    let sender = &test_data.user1;
    let receiver = &test_data.user2;
    
    runner.execute_transaction(TransactionTestCase {
        input: sender.create_plain_message::<TestRuntime, Bank>(
            CallMessage::Transfer {
                to: receiver.address(),
                coins: Coins { 
                    amount: 999_999_999_999, // More than the sender has 
                    token_id: config_gas_token_id() 
                },
            }
        ),
        assert: Box::new(|result, _state| {
            // Verify the transaction reverted
            assert!(result.tx_receipt.is_reverted());
            
            // Check the specific error message
            if let TxEffect::Reverted(contents) = &result.tx_receipt.tx_effect {
                assert!(contents.reason.to_string().contains("Insufficient balance"));
            }
        }),
    });
}
```

### 2. Event Testing

Verify that your module emits the correct events. Note that the event enum name (e.g., `TestRuntimeEvent`) is automatically generated based on your runtime name.

```rust
#[test]
fn test_event_emission() {
    let (test_data, mut runner) = setup();
    let user = &test_data.user1;
    
    runner.execute_transaction(TransactionTestCase {
        input: user.create_plain_message::<TestRuntime, YourModule>(
            CallMessage::CreateItem { name: "Test".into() }
        ),
        assert: Box::new(move |result, _state| {
            assert!(result.tx_receipt.is_successful());
            assert_eq!(result.events.len(), 1);
            
            assert_eq!(
                result.events[0],
                TestRuntimeEvent::YourModule(your_module::Event::ItemCreated {
                    creator: user.address(),
                    name: "Test".into()
                })
            );
        }),
    });
}
```

### 3. Time-Based Testing

Test operations that depend on blockchain progression by advancing slots:

```rust
#[test]
fn test_time_delayed_operation() {
    let (users, mut runner) = setup();

    // 1. Initiate a time-locked operation (e.g., a vesting schedule)

    // 2. Advance blockchain time
    runner.advance_slots(100); // Advance 100 slots

    // 3. Now, the second part of the operation should succeed
    runner.execute_transaction(/* ... complete the operation ... */);
}
```

### 4. Standalone State Queries

While you can query state within a transaction's assert block, you can also query the latest visible state at any point using `runner.query_visible_state`. This is useful for verifying the initial genesis state or checking state after non-transaction events like advancing slots. This can be useful if you especially have custom [`hooks`](3-5-advanced.md#hooks):

```rust
#[test]
fn test_state_queries() {
    let (test_data, mut runner) = setup();
    let admin = &test_data.admin;

    // Query the initial genesis state before any transactions
    runner.query_visible_state(|state| {
        // Query a value from your module
        let item_count = YourModule::<S>::default()
            .get_item_count(state)
            .unwrap_infallible()
            .unwrap();
        assert_eq!(item_count, 0);

    });

    // 2. Execute a transaction that changes state
    runner.execute_transaction(TransactionTestCase {
        input: admin.create_plain_message::<TestRuntime, YourModule>(
            CallMessage::CreateItem { name: "Test".into() }
        ),
        assert: |result, _| assert!(result.tx_receipt.is_successful()),
    });

    // Query again to see the new state
    runner.query_visible_state(|state| {
        let item_count = YourModule::<S>::default()
            .get_item_count(state)
            .unwrap_infallible()
            .unwrap();
        assert_eq!(item_count, 1);
    });
}
```

### 5. Custom Module Genesis Configuration

If your module requires initialization parameters in genesis (like an admin address or initial values), you'll need to provide a custom configuration:

```rust
use sov_test_utils::{GenesisConfig};

fn setup_with_config() -> (TestUser<TestSpec>, TestRunner<TestRuntime, TestSpec>) {
    let genesis_config = HighLevelOptimisticGenesisConfig::generate()
        .add_accounts_with_default_balance(1);
    
    // Get the admin user
    let admin = genesis_config
        .additional_accounts()
        .first()
        .unwrap()
        .clone();
    
    // Create genesis with your module's configuration
    let genesis = GenesisConfig::from_minimal_config(
        genesis_config.into(),
        YourModuleConfig {
            admin: admin.address(),
            initial_value: 1000,
            // Other module-specific parameters
        },
    );
    
    let runner = TestRunner::new_with_genesis(
        genesis.into_genesis_params(),
        TestRuntime::default()
    );
    
    (admin, runner)
}
```

## Additional Resources

For more advanced testing scenarios, the [`sov-test-utils` crate](fix-link-https://docs.rs/sov-test-utils) is your primary resource. It contains all the testing components covered in this guide and much more.

We highly recommend exploring the documentation for the **[`TestRunner`](fix-link-https://docs.rs/sov-test-utils/latest/sov_test_utils/runtime/struct.TestRunner.html)** struct, which provides methods for more complex scenarios, including:

*   Executing and asserting on batches of transactions.
*   Querying historical state at specific block heights.
*   Customizing gas and fee configurations.
*   Running an integrated REST API server for off-chain testing.

The `sov-test-utils` crate provides a comprehensive toolkit for testing every aspect of your module's behavior.

### Ready for Primetime

With a thoroughly tested module, you can be confident in your logic's correctness and robustness. It's now time to bring your module to life by integrating it into a live rollup runtime.

In the next section, **"Integrating Your Module,"** we'll guide you through adding your module to the Runtime struct, configuring its genesis state, and making it a live component of your application.