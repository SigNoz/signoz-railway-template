# Deploy and Host SigNoz on Railway

[SigNoz](https://signoz.io) is an open-source observability platform that lets you collect, store, and analyze your application's **traces, metrics, and logs** using the OpenTelemetry standard.

> [!IMPORTANT]
> The template layout has changed. New deployments use the new layout automatically. If you deployed this template earlier, do **not** use Railway's "Update" button to move to it, as it will not apply cleanly and can break your install. Follow [MIGRATION.md](./docs/MIGRATION.md) to migrate on your own schedule. The previous layout stays available for now, so existing installs keep running until you choose to migrate.

## About Hosting SigNoz

Deploying this template provisions the SigNoz stack:

- **SigNoz** (UI and query service)
- **SigNoz OpenTelemetry Collector** (ingester)
- **ClickHouse** (telemetry store)
- **ClickHouse Keeper** (coordination)
- **Schema migrator** (one-time job that prepares the ClickHouse schema)

The template wires these services together with the environment variables, health checks, and persistent storage they need. Point your application's OpenTelemetry SDK or agent at the collector's ingest endpoint, and SigNoz begins visualizing your service dependencies, latency, and errors.

## Common Use Cases

- **Application Performance Monitoring**: monitor metrics, logs, and traces across your Railway stack.
- **Debugging and Troubleshooting**: correlate logs, metrics, and traces to find and fix issues quickly.
- **Infrastructure Observability**: track system health, resource usage, and service dependencies.
- **Alerting and Incident Response**: set alerts on metric and log patterns for proactive response.

## Accessing SigNoz

- The SigNoz UI is served on port **8080** and exposed on the SigNoz service's public domain.
- Send telemetry to the collector over OTLP (gRPC on **4317**, HTTP on **4318**). Expose the port you need under the collector service's networking settings.

## Upgrading

For moving an existing install to the current layout, see [MIGRATION.md](MIGRATION.md).

## Why Deploy on Railway

Railway hosts your full infrastructure stack so you can deploy and scale SigNoz without managing the underlying servers. Host your services, databases, and observability stack together on one platform.

## Resources

- [SigNoz Documentation](https://signoz.io/docs/)
- [OpenTelemetry Specification](https://opentelemetry.io/docs/)
- [ClickHouse Documentation](https://clickhouse.com/docs/)
