# Logging

The SDK uses the `tracing` crate for structured logging, providing rich context and efficient filtering.

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

Spans provide hierarchical context for complex operations:

```rust
use tracing::{instrument, Instrument};

#[instrument(
    skip(self, state),
    fields(
        module = "my_module",
        operation = "batch_process"
    )
)]
fn process_batch(&self, batch_id: BatchId, items: Vec<Item>, state: &mut impl TxState<S>) -> Result<()> {
    info!(%batch_id, item_count = items.len(), "Starting batch processing");
    
    for (idx, item) in items.iter().enumerate() {
        // Create a child span for each item
        let span = tracing::span!(
            tracing::Level::DEBUG,
            "process_item",
            %batch_id,
            item_index = idx,
            item_id = %item.id
        );
        
        let _enter = span.enter();
        trace!("Processing item");
        self.process_single_item(item, state)?;
        trace!("Item processed");
    }
    
    info!(%batch_id, "Batch processing completed");
    Ok(())
}
```

## Structured Error Context

Use `anyhow` for rich error context:

```rust
use anyhow::{Context, Result};

fn validate_transfer(&self, from: &S::Address, to: &S::Address, amount: u64, state: &impl StateAccessor<S>) -> Result<()> {
    let balance = self.balances
        .get(from, state)
        .context("Failed to read sender balance")?
        .unwrap_or(0);
    
    if balance < amount {
        error!(
            %from,
            %balance,
            requested_amount = %amount,
            "Insufficient balance for transfer"
        );
        
        return Err(anyhow::anyhow!("Insufficient balance"))
            .with_context(|| format!(
                "Account {} has balance {} but attempted to transfer {}",
                from, balance, amount
            ));
    }
    
    trace!(%from, %to, %amount, "Transfer validated");
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

3. **Log at operation boundaries**:
   - Start and end of major operations
   - Before and after state changes
   - Error conditions with full context

4. **Use conditional logging for expensive operations**:
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