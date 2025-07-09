# Deploy and Host SigNoz on Railway

**[SigNoz](https://signoz.io)** is an open-source observability platform that enables you to collect, store, and analyze distributed application **traces, metrics, and logs** using the OpenTelemetry standard.

## About Hosting SigNoz

When you deploy SigNoz on Railway, the following core services are provisioned:
- SigNoz
- SigNoz Otel Collector
- ClickHouse
- Zookeeper

The Railway template automatically sets up these services with the necessary environment variables, health checks, and persistent storage. This allows you to quickly go from deployment to creating dashboards. Simply point your application's OpenTelemetry SDK or agent to the provided ingest URL, and SigNoz will immediately begin visualizing service dependencies, latency, and errors.

## Common Use Cases

- **Application Performance Monitoring**: Monitor metrics, logs, and traces across your entire Railway application stack.
- **Debugging and Troubleshooting**: Correlate logs, metrics, and traces to quickly identify and resolve issues.
- **Infrastructure Observability**: Monitor system health, resource usage, and service dependencies in real time.
- **Alerting and Incident Response**: Set up alerts based on metrics and log patterns for proactive incident management.

## Dependencies for SigNoz Hosting

- **Persistent Storage**: Use a Railway volume (or external block storage) for ClickHouse and SigNoz data.
- **Ingest Traffic**: Applications should export OpenTelemetry traces, metrics, or logs over HTTP or gRPC.

### Deployment Dependencies

- [SigNoz Documentation](https://signoz.io/docs/)
- [OpenTelemetry Specification](https://opentelemetry.io/docs/)
- [ClickHouse Server](https://clickhouse.com/docs/en/)

### Implementation Details

To run the SigNoz stack on Railway, ensure the following:

#### OpenTelemetry Ingestion
- You may need to configure **Domains / Proxy** settings in Railway for the `signoz-otel-collector` service, depending on your use case.  
- Port **4317** is open for ingestion by default.

#### SigNoz UI
- A public domain is configured automatically in Railway to access the SigNoz dashboard.

#### Schema-Migration Order  
ClickHouse migrations run in the dedicated **`signoz-schema-migrator`** job. As the Railway does not yet offer Docker-style `depends_on`, dependent services can occasionally start before migrations finish and fail on their first boot.  
If that happens, **redeploy these services after the migrator job completes**, in the exact order shown:

1. **signoz-async-schema-migrator**  
2. **signoz** (main application)  
3. **signoz-otel-collector**

After redeploying in this sequence, all components will connect to ClickHouse with the correct schema and operate normally.

## Why Deploy

Railway is a singular platform to deploy your infrastructure stack. Railway will host your infrastructure so you don't have to deal with configuration, while allowing you to vertically and horizontally scale it.

By deploying SigNoz on Railway, you are one step closer to supporting a complete full-stack application with minimal burden. Host your servers, databases, AI agents, and more on Railway.
