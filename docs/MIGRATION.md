# Migrate your Railway install to the new SigNoz template

Tags: Self-Host, Railway

## What changed

The SigNoz Railway template moved to a new layout. Services are renamed, each service's config moves into its own folder under `deployment/`, ClickHouse coordination moves from ZooKeeper to ClickHouse Keeper, and a single `signoz-telemetrystore-migrator` replaces the old schema-migrator jobs.

Railway's template "Update" only refreshes files inside services that already exist. It cannot rename services, add new ones, or change a service's folder, so this move is done by hand.

| Old service | New service | Root directory | Volume (mount path) |
| --- | --- | --- | --- |
| `clickhouse` | `signoz-telemetrystore-clickhouse` | `/deployment/telemetrystore` | keep, `/var/lib/clickhouse/` |
| `zookeeper` | `signoz-telemetrykeeper-clickhousekeeper` | `/deployment/telemetrykeeper` | new, `/var/lib/clickhouse-keeper/` |
| `signoz` | `signoz-signoz` | `/deployment/signoz` | keep, `/var/lib/signoz/` |
| `signoz-otel-collector` | `signoz-ingester` | `/deployment/ingester` | none |
| `schema-migrator-sync` / `-async` | `signoz-telemetrystore-migrator` | `/deployment/telemetrystore-migrator` | none |

Each service builds from a `Dockerfile` and reads its `railway.json` config from its root directory. Keep the new service names exactly as above; the configs resolve each other over `*.railway.internal` by service name.

## Before you begin

Do not use Railway's "Update available" button to move to this layout. The update only refreshes files inside your existing services, so it cannot make this change and will break the install. Migrate on your own schedule by following the steps below. The old layout stays in the template until it is deprecated, so there is no rush.

Back up your volumes: open the ClickHouse and SigNoz services in Railway and take a volume backup. Deleted volumes are recoverable for 48 hours.

## Steps

### Step 1: Repoint ClickHouse

1. Rename the `clickhouse` service to `signoz-telemetrystore-clickhouse`.
2. In **Settings > Source**, set **Root Directory** to `/deployment/telemetrystore`.
3. Keep its volume attached at `/var/lib/clickhouse/`. Its data stays on disk.

### Step 2: Add ClickHouse Keeper

1. Add a new service from the template repo with **Root Directory** `/deployment/telemetrykeeper`, named `signoz-telemetrykeeper-clickhousekeeper`.
2. Give it a fresh volume mounted at `/var/lib/clickhouse-keeper/`.
3. You can remove the old `zookeeper` service once ClickHouse is healthy on Keeper (Step 6).

### Step 3: Repoint SigNoz

1. Rename the `signoz` service to `signoz-signoz`.
2. Set **Root Directory** to `/deployment/signoz`.
3. Keep its volume mounted at `/var/lib/signoz/`. Your dashboards, alerts, and users live there.
4. Keep the public domain on this service for the UI.

### Step 4: Repoint the collector

1. Rename the `signoz-otel-collector` service to `signoz-ingester`.
2. Set **Root Directory** to `/deployment/ingester`.
3. Under **Settings > Networking**, expose the OTLP port (`4318`, and `4317` if you use gRPC) with a TCP proxy so your applications can reach it.

### Step 5: Add the migrator

1. Add a new service from the template repo with **Root Directory** `/deployment/telemetrystore-migrator`, named `signoz-telemetrystore-migrator` (mind the spelling: `telemetrystore`).
2. It is a run-once job (restart policy `NEVER`) that runs `migrate ready`, `bootstrap`, `sync up`, and `async up`.

### Step 6: Bring ClickHouse data back online

The new Keeper starts empty, so your existing replicated tables come up read-only until their coordination metadata is rebuilt from the data on disk.

1. Start `signoz-telemetrystore-clickhouse` and `signoz-telemetrykeeper-clickhousekeeper`.
2. Connect to ClickHouse and check:

   ```sql
   SELECT database, table FROM system.replicas WHERE is_readonly;
   ```

3. For each read-only table across the SigNoz databases (`signoz_traces`, `signoz_logs`, `signoz_metrics`, `signoz_metadata`, `signoz_meter`):

   ```sql
   SYSTEM RESTORE REPLICA <database>.<table>;
   SYSTEM SYNC REPLICA <database>.<table>;
   ```

Do this before running the migrator, so the migrator applies new migrations on top of healthy, writable tables.

### Step 7: Apply in order

Deploy in this order: ClickHouse and Keeper first, then run the migrator, then SigNoz and the ingester.

### Step 8: Validate

1. The `signoz-telemetrystore-migrator` deployment finishes and exits successfully.
2. `SELECT database, table FROM system.replicas WHERE is_readonly;` returns no rows.
3. Open SigNoz on its public domain: your dashboards and alerts are present and you can log in.
4. Point your applications at the `signoz-ingester` OTLP endpoint (its TCP proxy domain and port) and confirm new data arrives.

## Troubleshooting

### ClickHouse tables are read-only after the move

**Symptoms:** SigNoz shows no data, and `system.replicas` lists tables with `is_readonly = 1`.

**Likely causes:** the move to a fresh Keeper means the replicated tables cannot find their coordination metadata.

**Resolution:** run `SYSTEM RESTORE REPLICA` then `SYSTEM SYNC REPLICA` for each affected table (Step 6).

### The ingester keeps restarting

**Symptoms:** `signoz-ingester` crashes on start and retries.

**Likely causes:** the ingester runs `migrate sync check` at startup. If `signoz-telemetrystore-migrator` has not finished its sync migrations, the check fails and the ingester exits.

**Resolution:** wait for the migrator to finish, then redeploy the ingester.

### SigNoz starts with no dashboards or users

**Symptoms:** SigNoz loads, but your dashboards, alerts, and users are gone.

**Likely causes:** the SigNoz volume is not mounted at `/var/lib/signoz/`, so the metastore is not found.

**Resolution:** set the SigNoz service volume mount path to `/var/lib/signoz/` and redeploy.

### The migrator exits with an error

**Symptoms:** `signoz-telemetrystore-migrator` exits with a non-zero code.

**Likely causes:** it cannot reach ClickHouse, or ClickHouse cannot reach Keeper.

**Resolution:** confirm `signoz-telemetrystore-clickhouse` is healthy and `signoz-telemetrykeeper-clickhousekeeper` is running, then re-run the migrator.
