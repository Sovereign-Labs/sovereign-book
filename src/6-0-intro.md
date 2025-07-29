# Instrumenting Your Rollup

Proper instrumentation is essential for monitoring, debugging, and optimizing your rollup in production. The Sovereign SDK provides comprehensive observability tools that help you understand your rollup's behavior and performance.

## Getting Started with Observability

The [rollup starter](https://github.com/Sovereign-Labs/rollup-starter) repository includes a complete observability stack that gives you instant visibility into your rollup. With a single command, you can spin up a local monitoring environment:

```bash
$ make start-obs
...
Waiting for all services to become healthy...
⏳ Waiting for services... (45 seconds remaining)
✅ All observability services are healthy!
🚀 Observability stack is ready:
   - Grafana:     http://localhost:3000 (admin/admin123)
   - InfluxDB:    http://localhost:8086 (admin/admin123)
```

This command starts all necessary Docker containers and automatically provisions Grafana dashboards specifically designed for rollups. You'll immediately see key metrics like block production rate, transaction throughput, and system performance.

To stop the observability stack:
```bash
make stop-obs
```

For production deployments and advanced configuration, check out our [Observability Tutorial](https://sovlabs.notion.site/Tutorial-Getting-started-with-Grafana-Cloud-17e47ef6566b80839fe5c563f5869017?pvs=74).

## Adding Custom Instrumentation

While the default dashboards provide excellent baseline monitoring, every rollup has unique requirements. You'll want to add custom instrumentation to track:
- Application-specific metrics (e.g., DEX trading volume, NFT mints)
- Performance bottlenecks in your custom modules

This section will teach you how to:
- **[Add Custom Metrics](6-1-metrics.md)** - Track performance indicators and business metrics using the SDK's metrics framework
- **[Implement Structured Logging](6-2-logging.md)** - Debug and monitor your rollup's execution with contextual logs

## Important: Native-Only Features

All instrumentation code must be gated with `#[cfg(feature = "native")]` to ensure it only runs on full nodes, not in the zkVM during proof generation. This critical distinction allows you to instrument generously without affecting proof generation performance or determinism.

Let's dive into the specifics of adding metrics and logging to your rollup.