# Instrumenting Your Rollup

Proper instrumentation is essential for monitoring, debugging, and optimizing your rollup in production. The Sovereign SDK provides comprehensive observability tools that help you understand your rollup's behavior and performance.

This section covers:
- **[Metrics](/instrumenting/metrics.md)** - Track performance indicators and business metrics
- **[Logging](/instrumenting/logging.md)** - Debug and monitor your rollup's execution

[TODO: Insert section on spinning up Grafana dashboards to monitor your rollup seamlessly]

## Important: Native-Only Features

All instrumentation code must be gated with `#[cfg(feature = "native")]` to ensure it only runs on full nodes, not in the zkVM during proof generation. This allows you to instrument generously without affecting proof generation performance.