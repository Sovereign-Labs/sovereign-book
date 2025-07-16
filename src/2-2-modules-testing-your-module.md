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

// Generate a runtime with core modules + your custom module
generate_optimistic_runtime!(
    TestRuntime <=  // Your test runtime name
    my_module: MyModule<S> // Your custom module
);
```

### 2. Define Your Genesis Configuration

Set up the initial state with test users:

```rust
use sov_test_utils::{HighLevelOptimisticGenesisConfig, TestRunner};

fn setup() -> (Vec<TestUser<TestSpec>>, TestRunner<TestRuntime, TestSpec>) {
    // Create genesis config with test users
    let genesis_config = HighLevelOptimisticGenesisConfig::generate()
        // Add 3 users with default gas balance
        .add_accounts_with_default_balance(3);
    
    // Extract users from genesis for later use
    let users = genesis_config.additional_accounts().to_vec();
    
    // Create runner with genesis
    let runner = TestRunner::new_with_genesis(
        genesis_config.into_genesis_params(),
        TestRuntime::default()
    );
    
    (users, runner)
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
        // Setup runner and get test users from genesis
        let (users, mut runner) = setup();
        let user = &users[0];
        
        // Execute a transaction
        runner.execute_transaction(TransactionTestCase {
            input: user.create_plain_message::<TestRuntime, MyModule>(
                CallMessage::SetValue { value: 42 }
            ),
            assert: Box::new(|result, state| {
                // Verify the transaction succeeded
                assert!(result.tx_receipt.is_successful());
                
                // Query and verify state
                let current_value = MyModule::default()
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
    let (users, mut runner) = setup();
    let sender = &users[0];
    let receiver = &users[1];
    
    runner.execute_transaction(TransactionTestCase {
        input: sender.create_plain_message::<TestRuntime, Bank>(
            CallMessage::Transfer {
                to: receiver.address(),
                coins: Coins { 
                    amount: 999_999_999_999, // More than they have
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

Verify your module emits the correct events. Note that the event enum name (e.g., `TestRuntimeEvent`) is automatically generated based on your runtime name in the `generate_optimistic_runtime!` macro:

```rust
#[test]
fn test_event_emission() {
    let (users, mut runner) = setup();
    let user = &users[0];
    
    runner.execute_transaction(TransactionTestCase {
        input: user.create_plain_message::<TestRuntime, MyModule>(
            CallMessage::CreateItem { name: "Test".into() }
        ),
        assert: Box::new(move |result, _state| {
            assert!(result.tx_receipt.is_successful());
            assert_eq!(result.events.len(), 1);
            
            assert_eq!(
                result.events[0],
                TestRuntimeEvent::MyModule(my_module::Event::ItemCreated {
                    creator: user.address(),
                    name: "Test".into()
                })
            );
        }),
    });
}
```

### 3. Time-Based Testing

Test operations that depend on blockchain progression:

```rust
#[test]
fn test_time_delayed_operation() {
    let (users, mut runner) = setup();
    let user = &users[0];
    
    // Initiate a time-locked operation - transaction test case elided here for brevity
    runner.execute_transaction(start_timelock);
    
    // Try to complete immediately - should fail
    runner.execute_transaction(TransactionTestCase {
        input: user.create_plain_message::<TestRuntime, MyModule>(
            CallMessage::CompleteTimelock { id: timelock_id }
        ),
        assert: Box::new(|result, _| {
            assert!(result.tx_receipt.is_reverted());
            
            if let TxEffect::Reverted(contents) = &result.tx_receipt.tx_effect {
                assert!(contents.reason.to_string().contains("Not ready"));
            }
        }),
    });
    
    // Advance blockchain time
    runner.advance_slots(100); // Advance 100 slots
    
    // Now the operation should succeed
    runner.execute_transaction(complete_timelock);
}
```

### 4. State Queries

Query state without executing transactions:

```rust
#[test]
fn test_state_queries() {
    let (users, mut runner) = setup();
    let user = &users[0];
    
    // Execute some state changes
    runner.execute_transaction(create_item);
    
    // Query current state
    runner.query_visible_state(|state| {
        let item_count = MyModule::default()
            .get_item_count(state)
            .unwrap_infallible()
            .unwrap();
        assert_eq!(item_count, 1);
        
        // Query from other modules
        let user_balance = Bank::default()
            .get_balance_of(
                user.address(),
                config_gas_token_id(),
                state
            )
            .unwrap_infallible()
            .unwrap();
        assert!(user_balance > 0);
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
        MyModuleConfig {
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

For more advanced testing scenarios and APIs, refer to the [`TestRunner`](fix-link) struct documentation in the SDK. The TestRunner provides many additional methods for:

- Executing batches of transactions
- Querying archival state at specific heights
- Setting up REST API servers for testing
- Custom gas configurations
- And much more

The test utilities in the [`sov-test-utils`](fix-link) crate provide comprehensive tools for testing every aspect of your module's behavior.