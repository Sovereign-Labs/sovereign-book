# Instrumenting Your Rollup

Proper instrumentation is essential for monitoring, debugging, and optimizing your module in production. The Sovereign SDK provides comprehensive observability tools that work seamlessly with your rollup.

[TODO: Insert section on spinning up Grafana dashboards to monitor your rollup seamlessly]

## Metrics

The SDK provides a flexible and backend-agnostic framework for tracking metrics in rollups called `sov-metrics`. It allows developers to define and record metrics that are serialized in the [Telegraf line protocol](https://docs.influxdata.com/influxdb/cloud/reference/syntax/line-protocol/). Metrics are timestamped automatically and can only be tracked in **native mode**.

### Basic Example

```rust
#[cfg(feature = "native")]
use sov_metrics::{track_metrics, start_timer, save_elapsed};

impl<S: Spec> MyModule<S> {
    fn process_batch(&self, items: Vec<Item>) -> Result<()> {
        // Time the entire operation
        start_timer!(batch_timer);
            
        for item in items {
            self.process_item(item)?;
        }
            
        save_elapsed!(elapsed SINCE batch_timer);

        #[cfg(feature = "native")] 
        {
            // Track batch size
            track_metrics(|tracker| {
                tracker.submit_inline(
                    "mymodule_batch_size",
                    format!("items={}", items.len()),
                );
            });
            
            // Track processing time
            track_metrics(|tracker| {
                tracker.submit_inline(
                    "mymodule_batch_processing_time",
                    format!("duration_ms={}", elapsed.as_millis()),
                );
            });
        }
        
        Ok(())
    }
}
```

### Example: Tracking Custom Metrics

To track custom metrics, implement the `Metric` trait:

```rust
#[cfg(feature = "native")]
use sov_metrics::Metric;
use sov_metrics::{track_metrics, start_timer, save_elapsed};
use std::io::Write;

#[derive(Debug)]
struct TransferMetric {
    from: String,
    to: String,
    token_id: TokenId,
    amount: u64,
    duration_ms: u64,
    gas_used: u64,
}

impl Metric for TransferMetric {
    fn measurement_name(&self) -> &'static str {
        "mymodule_transfers"
    }
    
    fn serialize_for_telegraf(&self, buffer: &mut Vec<u8>) -> std::io::Result<()> {
        // Format: measurement_name,tag1=value1,tag2=value2 field1=value1,field2=value2
        write!(
            buffer,
            "{},from={},to={},token_id={} amount={},duration_ms={},gas_used={}",
            self.measurement_name(),
            self.from,
            self.to,
            self.token_id,
            self.amount,
            self.duration_ms,
            self.gas_used
        )
    }
}

// Usage in your module
fn transfer(&self, from: &S::Address, to: &S::Address, token_id: &TokenId, amount: u64, state: &mut impl TxState<S>) -> Result<()> {
    start_timer!(transfer_timer);
    
    // Perform the transfer
    self.do_transfer(from, to, token_id, amount, state)?;
    
    save_elapsed!(elapsed SINCE transfer_timer);
    
    #[cfg(feature = "native")]
    {
        // Track your custom metric
        track_metrics(|tracker| {
            tracker.submit_metric(TransferMetric {
                from: from.to_string(),
                to: to.to_string(),
                token_id: token_id.clone(),
                amount,
                duration_ms: elapsed.as_millis() as u64,
            });
        });
    }
    
    Ok(())
}
```

### Metrics Best Practices

Note: While the SDK provides comprehensive metrics infrastructure, individual modules in the SDK don't currently use metrics directly. Most metrics are tracked at the system level (runner, sequencer, state transitions). The examples here show how you *could* add metrics to your custom modules.

1. **Always gate with `#[cfg(feature = "native")]`** - Metrics are not available in zkVM
2. **Use meaningful measurement names** - Follow the pattern `module_name_metric_type`
3. **Separate fields and tags properly**:
   - Fields: Numerical values you want to aggregate (counts, durations, amounts)
   - Tags: Categorical values for filtering (types, status, enunm variants)
4. **Track business-critical metrics**:
   - Transaction volumes and types
   - Processing times for key operations
   - Error rates and types
5. **Avoid high-cardinality tags** - Don't use unique identifiers like transaction hashes as tags

## Logging

The SDK uses the `tracing` crate for structured logging, providing rich context and efficient filtering.

### Basic Logging Patterns

```rust
use tracing::{trace, debug, info, warn, error};

impl<S: Spec> MyModule<S> {
    fn process_transaction(&self, tx: Transaction, state: &mut impl TxState<S>) -> Result<()> {
        // Log at transaction boundaries
        info!(tx_id = %tx.id, sender = %tx.sender, "Processing transaction");
        
        // Detailed execution flow
        trace!(tx_id = %tx.id, "Validating transaction");
        
        if let Err(e) = self.validate(&tx) {
            error!(
                tx_id = %tx.id,
                error = ?e,
                "Transaction validation failed"
            );
            return Err(e);
        }
        
        // State changes
        debug!(
            tx_id = %tx.id,
            old_balance = %old_balance,
            new_balance = %new_balance,
            "Balance updated"
        );
        
        // Warnings for unusual conditions
        if tx.amount > self.large_tx_threshold {
            warn!(
                tx_id = %tx.id,
                amount = %tx.amount,
                threshold = %self.large_tx_threshold,
                "Large transaction detected"
            );
        }
        
        info!(tx_id = %tx.id, "Transaction completed successfully");
        Ok(())
    }
}
```

### Using Spans for Context

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
        
        async {
            trace!("Processing item");
            self.process_single_item(item, state)?;
            trace!("Item processed");
            Ok::<_, Error>(())
        }.instrument(span).await?;
    }
    
    info!(%batch_id, "Batch processing completed");
    Ok(())
}
```

### Structured Error Context

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

### Logging Best Practices

1. **Use appropriate log levels**:
   - `error!` - Unrecoverable errors that affect module operation
   - `warn!` - Recoverable issues or unusual conditions
   - `info!` - High-level operations (tx processing, module lifecycle)
   - `debug!` - Detailed operational data (state changes, intermediate values)
   - `trace!` - Very detailed execution flow

2. **Structure your logs**:
   ```rust
   // Good - structured, filterable
   debug!(user = %address, action = "deposit", amount = %value, "Processing deposit");
   
   // Avoid - unstructured string interpolation
   debug!("Processing deposit for {} of amount {}", address, value);
   ```

3. **Include relevant context**:
   - Transaction/operation IDs
   - User addresses (when relevant)
   - Amounts and values
   - Error details
   - State transitions

4. **Log at operation boundaries**:
   - Start and end of major operations
   - Before and after state changes
   - Error conditions with full context

### Debugging Techniques

1. **Conditional logging**:
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

2. **State transition logging**:
   ```rust
   fn update_config(&self, new_config: Config, state: &mut impl TxState<S>) -> Result<()> {
       let old_config = self.config.get(state)?;
       
       debug!(
           ?old_config,
           ?new_config,
           "Updating module configuration"
       );
       
       self.config.set(&new_config, state)?;
       
       info!("Module configuration updated");
       Ok(())
   }
   ```

## Performance Considerations

Remember that all metrics and logging are automatically disabled during zkVM execution, so you can instrument generously without affecting proof generation performance. However, in native execution:

1. **Use trace level for hot paths** - Frequent operations should use `trace!`
2. **Batch metrics submissions** - Group related metrics when possible
3. **Avoid expensive computations in log statements** - They're always evaluated
4. **Use conditional compilation** - Gate expensive debug code with `#[cfg(debug_assertions)]`

## Integration with Monitoring Systems

The SDK's metrics integrate seamlessly with standard monitoring stacks:

1. **Telegraf** → **InfluxDB** → **Grafana** for metrics visualization
2. **Vector** or **Fluentd** for log aggregation
3. **Jaeger** or **Tempo** for distributed tracing (via OpenTelemetry)

Configure your rollup's logging output:

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

This structured approach to instrumentation ensures you have complete visibility into your module's behavior in production while maintaining optimal performance.