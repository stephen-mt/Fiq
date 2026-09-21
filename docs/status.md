# Delivery status

## Implemented

- Typed `PathTarget`/`CatalogTarget` contracts persisted from discovery through verification, with
  tests proving no `forPath`/`forName` fallback.
- Bounded PATH discovery and Spark/HMS Delta-provider discovery, idempotent upsert, missing-table
  state, canonical fingerprints, and sample labels.
- Policy-independent six-dimension assessments with per-dimension completeness/provenance,
  bounded file-size histogram, real transaction-log/tombstone facts, and UNKNOWN preservation.
- User-configurable OPTIMIZE/VACUUM policies and deterministic structured decisions; a debt score
  is UI summary only and is not the planner source of truth.
- Qualified OPTIMIZE BINPACK and VACUUM FULL execution for Spark 4.0.1 + Delta 4.0.1, version
  checks, structured result + SHA-256 manifest, explicit steps, and independent verification.
- VACUUM canonical preflight/approval evidence and pre-mutation revalidation. Zero-hour retention
  is restricted to isolated sample tables and approval.
- Durable cron/window/selector policy scheduler with fire idempotency and concurrency controls.
- Local PostgreSQL/MinIO/HMS/Livy/Spark sample topology and automatic connection/discovery
  bootstrap with no manual database insertion or setup curl.
- A minimal product runtime containing only FIQ server/UI and PostgreSQL. MinIO, HMS, Livy/Spark,
  storage credentials, and sample generation live exclusively in the local demo overlay.
- Minimal UI controls for connection, discovery, six-dimension health, policy creation, planning,
  approval/execution, and before/after evidence. Playwright covers the core mocked API workflow at
  390/768/1440 plus dark theme.

## Qualification evidence in this checkout

The following checks passed on 2026-08-24:

- `./gradlew ci :fiq-server:quarkusBuild :fiq-spark-job:shadowJar --no-daemon`: 39 Gradle tasks,
  including all backend/domain tests, formatting checks, the production Quarkus build, and Spark
  shadow JAR.
- UI typecheck, three Vitest cases, production Vite build, and nine Playwright cases across the
  mobile/tablet/desktop projects.
- Both minimal and merged demo Compose configurations validate, `bash -n
  scripts/acceptance-core.sh` passes, and `make help` documents scoped lifecycle commands.
- A clean minimal Compose project started through `make up` with only PostgreSQL and FIQ. FIQ
  readiness was `UP`, the UI returned HTTP 200, and empty connection/table lists remained usable
  without MinIO, HMS, Livy, storage configuration, or sample data.
- A separate clean demo Compose project with new PostgreSQL, MinIO, HMS, and Ivy volumes started
  through `make demo-up`. Sample generation, connection bootstrap, PATH discovery, and HMS
  discovery completed without manual database insertion or setup curl.
- Real PATH OPTIMIZE used a `PathTarget`, advanced Delta version 1 to 2, and reduced active/small
  files from 96 to 1.
- Real HMS OPTIMIZE used a `CatalogTarget`, advanced Delta version 1 to 2, and reduced active/small
  files from 64 to 1.
- Real isolated zero-hour VACUUM FULL required approval, preserved a preflight hash for eight
  candidates (7,541 measured bytes), applied the mutation, and verified zero approved candidates
  remaining.

The executable acceptance is [`scripts/acceptance-core.sh`](../scripts/acceptance-core.sh). It is
destructive only to the explicitly marked sample fixtures and must never be pointed at production
tables.

## Not currently mutation-qualified

VACUUM LITE, Z-order, liquid-clustering maintenance, REORG, inventory vacuum, and Glue remain
visible only for compatibility. Unity Catalog, catalog-managed mutation, generic table formats,
HA/DR qualification, enterprise webhook/secret frameworks, and 100k-table scale qualification
are deferred.

Known limitations remain: first clean startup downloads a large Spark/Hive dependency graph;
Playwright uses deterministic API fixtures while the separate Compose acceptance exercises the
real backend; and Glue is not implemented. Broad security, scale, accessibility, and multi-runtime
compatibility qualification has not been completed.
