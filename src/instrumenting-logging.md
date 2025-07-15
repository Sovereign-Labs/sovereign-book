# Logging

The SDK uses the [`tracing`](https://docs.rs/tracing/latest/tracing/) crate for structured logging, providing rich context and efficient filtering.

## Basic Logging Patterns

```rust
// Adapted from the `Bank` module
use tracing::trace;

impl<S: Spec> MyModule<S> {
    pub(crate) fn freeze(
        &mut self,
        token_id: TokenId,
        context: &Context<S>,
        state: &mut impl TxState<S>,
    ) -> Result<()> {
        // Logging at the start of operation
        trace!(freezer = %sender, "Freeze token request");

        // Redundant code elided here...

        token
            .freeze(sender)
            .with_context(|| format!("Failed to freeze token_id={}", &token_id))?;

        self.tokens.set(&token_id, &token, state)?;

        // Logging at the end of operation
        trace!(
            freezer = %sender,
            %token_id,
            "Successfully froze tokens"
        );

        Ok(())
    }
}
```

## Using Spans for Context

Spans are like invisible context that gets automatically attached to every log line within their scope. Instead of passing context like `batch_id` or `user_id` through every function call just so you can log it, you create a span at the top level and all logs within that span automatically include that context.

Think of spans as a way to say "everything that happens from here until the span ends is part of this operation." This is especially useful when debugging - you can filter logs by span fields to see everything that happened during a specific batch process or user request.

Here's how you can use spans to provide hierarchical context for complex operations:

```rust
use tracing::instrument;

// Example 1: Using the #[instrument] macro (easiest way)
#[instrument(skip(self, state, items))]  // skip large/non-Debug types
fn process_batch(&self, batch_id: BatchId, items: Vec<Item>, state: &mut impl TxState<S>) -> Result<()> {
    // The #[instrument] macro automatically adds all function parameters (except skipped ones) to the span
    // So batch_id is automatically included in all logs within this function
    info!(item_count = items.len(), "Starting batch processing");
    
    for (idx, item) in items.iter().enumerate() {
        // This log will show: batch_id=123 item_id=456 "Processing item"
        trace!(item_index = idx, item_id = %item.id, "Processing item");
        self.process_single_item(item, state)?;
    }
    
    info!("Batch processing completed");
    Ok(())
}

// Example 2: Creating spans manually (when you need more control)
fn process_user_request(&self, user_id: UserId, request: Request) -> Result<()> {
    // Create a span with context that will be included in all logs
    let span = tracing::span!(
        tracing::Level::INFO,
        "user_request", // span name
        %user_id,
        request_type = %request.request_type()
    );
    
    // Enter the span - all logs from here will include user_id and request_type
    let _enter = span.enter();
    
    debug!("Validating request");
    self.validate_request(&request)?;
    
    debug!("Processing request");
    self.process(&request)?;
    
    info!("Request completed successfully");
    Ok(())
}
```

## Log Levels

- `error!` - Unrecoverable errors that affect module operation
- `warn!` - Recoverable issues or unusual conditions
- `info!` - High-level operations (tx processing, module lifecycle)
- `debug!` - Detailed operational data (state changes, intermediate values)
- `trace!` - Very detailed execution flow

## Best Practices

1. **Structure your logs**:
   ```rust
   // Good - structured, filterable
   debug!(user = %address, action = "deposit", amount = %value, "Processing deposit");
   
   // Avoid - unstructured string interpolation
   debug!("Processing deposit for {} of amount {}", address, value);
   ```

2. **Include relevant context**:
   - Transaction/operation IDs
   - User addresses (when relevant)
   - Amounts and values
   - Error details
   - State transitions

3. **Use conditional logging for expensive operations**:
   ```rust
   #[cfg(feature = "native")]
   fn debug_state(&self, state: &impl StateAccessor<S>) {
       if tracing::enabled!(tracing::Level::TRACE) {
           let total_accounts = self.count_accounts(state);
           let total_balance = self.calculate_total_balance(state);
           trace!(
               %total_accounts,
               %total_balance,
               "Module state snapshot"
           );
       }
   }
   ```

## Configuration

Configure logging output in your node initialization:

```rust
// In your node initialization
tracing_subscriber::fmt()
    .with_env_filter(EnvFilter::from_default_env())
    .with_target(false)
    .with_thread_ids(true)
    .with_level(true)
    .json()  // For production, use JSON format
    .init();
```

Set log levels via environment variables:
```bash
RUST_LOG=info,my_module=debug cargo run
```

## Integration with Log Aggregation

For production deployments:
- Use JSON formatting for structured logs
- Ship logs to aggregation services (Vector, Fluentd, Logstash)
- Query and analyze with Elasticsearch, Loki, or CloudWatch
- Set up alerts for error patterns