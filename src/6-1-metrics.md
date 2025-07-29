# Metrics

The SDK includes a custom metrics system called [`sov-metrics`](https://github.com/Sovereign-Labs/sovereign-sdk/tree/nightly/crates/full-node/sov-metrics) designed specifically for rollup monitoring. It uses the [Telegraf line protocol](https://docs.influxdata.com/influxdb/cloud/reference/syntax/line-protocol/) format and integrates with Telegraf through socket listeners for efficient data collection. Metrics are automatically timestamped and sent to your configured Telegraf endpoint, which typically forwards them to InfluxDB for storage and Grafana for visualization. Metrics can only be tracked in **native mode** (not in zkVM).

**Important**: Metrics are emitted immediately when tracked and are NOT rolled back if a transaction reverts. This means failed transactions will still have their metrics recorded, which can be useful for debugging and monitoring error rates.

## Basic Example

```rust
#[cfg(feature = "native")]
use sov_metrics::{track_metrics, start_timer, save_elapsed};

impl<S: Spec> MyModule<S> {
    fn process_batch(&self, items: Vec<Item>) -> Result<()> {
        // Time the operation using the provided macros
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

## Tracking Custom Metrics

To track custom metrics, implement the `Metric` trait:

```rust
// Implement your custom metric in a file of your own choosing...
#![cfg(feature = "native")]
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
}

impl Metric for TransferMetric {
    fn measurement_name(&self) -> &'static str {
        "mymodule_transfers"
    }
    
    fn serialize_for_telegraf(&self, buffer: &mut Vec<u8>) -> std::io::Result<()> {
        // Format: measurement_name,tag1=value1,tag2=value2 field1=value1,field2=value2
        write!(
            buffer,
            "{},from={},to={},token_id={} amount={},duration_ms={}",
            self.measurement_name(),
            self.from,
            self.to,
            self.token_id,
            self.amount,
            self.duration_ms
        )
    }
}

// In your module file...
#[cfg(feature = "native")]
use sov_metrics::{track_metrics, start_timer, save_elapsed};
#[cfg(feature = "native")]
use my_custom_metrics::TransferMetric;

// Adapted from Bank module 
impl<S: Spec> Bank<S> {
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
}
```

## Best Practices

Note: While the SDK provides comprehensive metrics infrastructure, individual modules in the SDK don't currently use metrics directly. Most metrics are tracked at the system level (runner, sequencer, state transitions). The examples here show how you *could* add metrics to your custom modules.

1. **Always gate with `#[cfg(feature = "native")]`** - Metrics are not available in zkVM
2. **Use meaningful measurement names** 
    - A lot of the packages that Sovereign SDK runs under the hood emit metrics. 
    To make it easy to discern that the metrics come from a Sovereign SDK component, we 
    follow the pattern of `sov_` in our metric names. We recommend following the 
    pattern `sov_user_module_name_metric_type` so that it's easy to discern user level
    metric types.
3. **Separate tags and fields properly**:
   - [Telegraf line protocol discerns between Tags and Fields by separating them with a single whitespace](https://docs.influxdata.com/influxdb/cloud/reference/syntax/line-protocol/#elements-of-line-protocol). Make sure to write your metrics accordingly. 
   - Tags: Categorical values for filtering (types, status, enum variants), both their keys and values can only be strings
   - Fields: Numerical values you want to aggregate (counts, durations, amounts), their keys can be strings, and values can be one of: floats, integers, unsigned integers, strings, and booleans
4. **Track business-critical metrics**:
   - Transaction volumes and types
   - Processing times for key operations
   - Error rates and types
5. **Avoid high-cardinality tags** - Don't use unique identifiers like transaction hashes as tags