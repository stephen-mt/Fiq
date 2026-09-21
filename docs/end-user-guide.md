# End-user guide

This guide starts from the perspective of an operator who has not used FIQ before. It covers two
paths:

1. Run the self-contained demo to learn the workflow.
2. Connect FIQ to an existing MinIO/S3, Spark, Livy, Delta Lake, and optional Hive Metastore.

## What FIQ does

FIQ does not store table data and does not replace Spark or a catalog. It is the maintenance
control plane:

```text
discover tables
  → measure Delta facts
  → evaluate your policy
  → explain the decision
  → preflight
  → ask Spark to OPTIMIZE or VACUUM FULL
  → independently verify before/after state
```

FIQ itself runs as `server + PostgreSQL`. Delta Kernel is embedded in the server. Object storage,
catalog, and Spark/Livy are integrations supplied by your platform.

## Path A: learn with the local demo

Requirements: Docker Compose, Java 21, `curl`, and `jq`.

```bash
make demo-up
make demo-ps
```

The first start can take several minutes while Spark downloads Delta/Hive dependencies. A ready
demo has `server`, `postgres`, `minio`, `hms-postgres`, `hms`, and `livy` running. `minio-init` and
`sample-init` should show `Exited (0)` because they are successful one-shot initializers.

Open <http://localhost:9091>. Sample PATH and HMS connections and tables appear automatically.
Follow the workflow under [Use the UI](#use-the-ui).

The acceptance script mutates only labeled sample fixtures:

```bash
make demo-test
```

To erase only FIQ demo data and begin again:

```bash
make demo-reset
make demo-up
```

Never point `scripts/acceptance-core.sh` at non-sample tables.

## Path B: use an existing Delta platform

### 1. Check compatibility

Maintenance mutation is currently qualified only for:

- Spark 4.0.1.
- Delta Lake 4.0.1.
- Scala 2.13 Spark applications.
- Livy as the execution adapter.
- Classic Delta tables addressed by PATH or Hive Metastore.

An unknown runtime or Delta feature may remain discoverable/readable, but FIQ fails closed and
does not mutate it.

Collect these values from the platform operator:

```text
Object-storage endpoint     http://minio.example:9000
Bounded Delta root          s3a://lakehouse/delta/production
Livy endpoint               http://livy.example:8998
FIQ result prefix           s3a://fiq-system/runs
Optional Spark catalog      spark_catalog
Optional HMS URI            thrift://hms.example:9083
```

Addresses are resolved from containers, not from the browser. `localhost:8998` inside the FIQ
container means the FIQ container itself. Use a routable hostname, Docker network alias,
`host.docker.internal`, or platform service DNS as appropriate.

### 2. Prepare Spark and Livy

Build the FIQ Spark application:

```bash
make build
```

Publish this artifact somewhere every Livy-submitted Spark driver can read:

```text
fiq-spark-job/build/libs/fiq-spark-job-0.1.0-SNAPSHOT-all.jar
```

For a file installed on each Livy host, the FIQ job URI is typically:

```text
local:///opt/fiq/fiq-spark-job.jar
```

Spark needs Delta and S3A configuration equivalent to:

```properties
spark.sql.extensions=io.delta.sql.DeltaSparkSessionExtension
spark.sql.catalog.spark_catalog=org.apache.spark.sql.delta.catalog.DeltaCatalog
spark.jars.packages=io.delta:delta-spark_2.13:4.0.1,org.apache.hadoop:hadoop-aws:3.4.1

spark.hadoop.fs.s3a.endpoint=http://minio.example:9000
spark.hadoop.fs.s3a.path.style.access=true
spark.hadoop.fs.s3a.connection.ssl.enabled=false
```

For HMS discovery, the same Spark/Livy runtime must also be configured with:

```properties
spark.sql.catalogImplementation=hive
spark.hadoop.hive.metastore.uris=thrift://hms.example:9083
```

Give Spark/Livy storage access through workload identity, its environment, Kubernetes Secrets, or
the platform credential chain. FIQ never sends raw access keys in Spark arguments.

### 3. Prepare object storage for FIQ

Create a system prefix such as `s3a://fiq-system/runs`. Spark writes structured operation results
there; FIQ reads and verifies their SHA-256 manifest. Both identities need access:

- Spark/Livy: read/write result objects and maintain the target Delta table.
- FIQ server: read table metadata and structured results.

For a local S3-compatible deployment, export runtime values before starting FIQ:

```bash
export FIQ_S3_ENDPOINT='http://minio.example:9000'
export FIQ_RESULTS_PREFIX='s3a://fiq-system/runs'
export FIQ_LIVY_JOB_JAR='local:///opt/fiq/fiq-spark-job.jar'
export AWS_ACCESS_KEY_ID='replace-with-runtime-secret'
export AWS_SECRET_ACCESS_KEY='replace-with-runtime-secret'

make up
```

The Compose file passes these optional variables into the FIQ container but does not store them in
Git or PostgreSQL. In production, prefer workload identity, Kubernetes Secrets, or another
orchestrator-native secret injection mechanism.

Open <http://localhost:9091>. An empty Connections page is the expected first-run state.

## Use the UI

### 1. Create a PATH connection

Open **Connections → New connection** and enter:

```text
Name                 Production Delta PATH
Source type          PATH
Explicit root URI    s3a://lakehouse/delta/production
Warehouse URI        leave blank
Livy URI             http://livy.example:8998
Secret reference     optional label, for example env:MINIO_PRODUCTION
```

The root must be bounded. Do not select an unrestricted whole bucket. FIQ searches down at most
four levels and recognizes directories containing a valid `_delta_log`.

`Secret reference` is only an identifier. It is not a password field and does not inject a secret;
the FIQ and Spark runtimes must already have credentials.

Click **Test capabilities**, then **Discover**. A successful discovery reports the number of Delta
tables found. Open **Tables** and confirm that the execution target is `PATH`; FIQ will mutate it
through `DeltaTable.forPath`.

### 2. Create an HMS connection

The Spark/Livy runtime must already know the HMS URI. In FIQ enter:

```text
Name                 Production HMS
Source type          Hive Metastore
Catalog name         spark_catalog
Warehouse URI        s3a://lakehouse/warehouse (optional)
Livy URI             http://livy.example:8998
Secret reference     optional descriptive reference
```

Click **Test capabilities** and **Discover**. FIQ asks the connection's Spark catalog to enumerate
namespaces/tables and retains only provider `delta`. In **Tables**, confirm execution target
`CATALOG`; mutation uses `DeltaTable.forName` and never silently falls back to a path.

### 3. Assess a table

Open **Tables → table → Refresh assessment**. Review all six dimensions:

- `FILE_LAYOUT`: active file count, bytes, histogram, and partition skew.
- `DELETION_VECTORS`: DV files/bytes/deleted rows when observable.
- `TRANSACTION_LOG`: current version, checkpoint, and log facts.
- `RETENTION`: tombstone count, bytes, and age distribution when measurable.
- `CLUSTERING`: partition/liquid-clustering facts and eligibility.
- `PROTOCOL`: Delta protocol, features, runtime, and mutation capability.

Check `completeness`, `provenance`, `observedVersion`, and `observedAt`. `UNKNOWN` means FIQ did
not measure the fact; it never means zero. Assessment records table facts and is independent of
policy thresholds.

### 4. Create an OPTIMIZE policy

Open **Policies → Create policy**. A conservative first manual policy is:

```text
Operation                       OPTIMIZE BINPACK
Small-file threshold            128 MiB
Minimum small files             20
Minimum small-file ratio        30%
Minimum rewrite bytes           0 for evaluation; raise for production
Target file size                1 GiB
Minimum expected reduction      0 for evaluation; raise after observing workloads
Automatic                       Off
Approval required               Off or On according to team practice
```

These are decision thresholds, not measurement settings. The same assessment can be evaluated by
multiple policies with different thresholds.

From the table's **Policies** tab, create a plan. Before queueing it, answer:

1. Is the decision `ELIGIBLE`, `NOT_ELIGIBLE`, or `BLOCKED`?
2. Which observed values passed or failed each threshold?
3. Is the exact target PATH or CATALOG correct?
4. Which Delta version was assessed?
5. How many bytes are expected to be rewritten?

Queue or approve the plan, then follow **Operations**:

```text
ASSESS → PREFLIGHT → SUBMIT → EXECUTE → VERIFY
```

Spark command success alone is not FIQ success. `SUCCEEDED` requires a fresh assessment and
verified improvement. Review version, active files, small files, and ratio before/after.

### 5. Create a VACUUM FULL policy

Start with normal retention and manual approval:

```text
Operation                       VACUUM FULL
Retention                       168 hours
Minimum candidate count         1
Minimum reclaimable bytes       choose a workload-appropriate value
Automatic                       Off
Approval required               On
```

FIQ runs a real `DRY RUN`, canonicalizes/sorts candidates, and binds approval to target, version,
retention, count, and candidate hash. Immediately before mutation it repeats the dry run and
rejects incompatible changes. After mutation it verifies that approved candidates disappeared.

Do not use zero-hour retention on real data. FIQ permits it only on isolated tables labeled
`sample=true`, and approval is still mandatory. VACUUM can remove files needed for old versions or
long-running readers when retention is unsafe.

### 6. Enable scheduling only after manual proof

Once manual plans produce expected before/after results, edit/create a policy with:

- Correct selector for environment/catalog/namespace/table/tags.
- Cron and workspace timezone.
- Maintenance window.
- `Automatic` enabled.
- Per-policy concurrency and byte budget.
- Approval retained for operations that need it.

Repeated scheduler ticks use a durable fire key and do not intentionally create duplicate runs.

## Understand common states

| State/result | Meaning | Action |
|---|---|---|
| `NOT_ELIGIBLE` | Facts do not cross your policy thresholds. | No action; adjust only if policy intent is wrong. |
| `BLOCKED` | Required evidence/capability is incomplete or unsafe. | Fix runtime, permissions, or missing evidence. |
| `AWAITING_APPROVAL` | Preflight succeeded but human approval is required. | Review target and evidence hash before approval. |
| `SKIPPED / FIQ_PLAN_STALE` | Table version changed after planning. | Refresh, reassess, and create a new plan. |
| `FAILED, maintenanceApplied=false` | Mutation was not proven to have run. | Inspect steps/logs, then retry safely through replan. |
| `FAILED, maintenanceApplied=true` | Mutation may have run but verification failed. | Inspect table state; never blindly replay the old job. |
| `SUCCEEDED` | Mutation and post-operation verification both passed. | Review before/after evidence and audit history. |

## Daily commands

Minimal runtime:

```bash
make up
make ps
make logs
make down
```

Demo:

```bash
make demo-up
make demo-ps
make demo-logs
make demo-test
make demo-down
```

Run `make help` for scoped restart and cleanup commands. FIQ cleanup never requires broad Docker
prune commands.

## Current limits

- Mutation-qualified only on Spark 4.0.1 + Delta Lake 4.0.1.
- PATH and HMS only; Glue and Unity Catalog are deferred.
- OPTIMIZE BINPACK and VACUUM FULL only.
- Livy is the only execution adapter implemented.
- `FIQ_S3_ENDPOINT` is currently deployment-level for embedded Kernel access to an
  S3-compatible endpoint; per-connection storage endpoints are not fully exposed in the UI.
- Other Spark and Delta versions require their own compatibility qualification.
