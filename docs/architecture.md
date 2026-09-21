# Architecture

FIQ is a control plane, not a storage engine and not a Spark replacement. Its job is to keep the
maintenance decision, approval, execution, and evidence in one place while leaving table data in
the user's platform.

![FIQ architecture](assets/fiq-architecture.png)

## Runtime boundary

The deployable FIQ runtime contains two services:

- `fiq-server`, which also serves the React SPA
- PostgreSQL, which stores control-plane state

Delta Kernel runs inside `fiq-server`; it is a library, not a separate service. Object storage,
Hive Metastore, Spark, and Livy are connection-scoped integrations. The demo Compose overlay
provides local versions of those systems for evaluation, but they are not required for the
minimal runtime to start.

This boundary matters operationally. FIQ can start with an empty Connections page and no data
platform attached. It reaches external systems only when a connection is tested, tables are
discovered or assessed, or an operation is executed.

## Request and maintenance flow

1. The React SPA calls workspace-scoped REST endpoints in `fiq-server`.
2. The server stores connections, assessment evidence, policies, plans, operation state, and
   audit records in PostgreSQL.
3. PATH discovery and metadata assessment use the embedded Delta Kernel. HMS discovery and
   runtime-only evidence are collected through the configured Spark/Livy environment.
4. A policy evaluates stored facts. Assessment does not contain policy thresholds or create a
   recommendation by itself.
5. An eligible plan moves through preflight and, when configured, human approval.
6. `fiq-engine-spark` submits a typed maintenance request to Livy. The Scala 2.13
   `fiq-spark-job` checks the planned Delta version and calls supported Delta/Spark APIs.
7. The job writes a structured `result.json` and SHA-256 manifest to the FIQ result prefix.
8. FIQ refreshes the table assessment and compares before/after evidence before marking the
   operation successful.

Livy logs are diagnostic output, not the canonical result. A Spark command completing is not by
itself proof that maintenance achieved the expected effect.

## Target handling

Execution targets stay typed from discovery through verification:

- `PathTarget` resolves with `DeltaTable.forPath`.
- `CatalogTarget` resolves with `DeltaTable.forName`.

There is no fallback between them. FIQ never writes `_delta_log` files directly. Unknown Delta
features and unsupported runtimes fail closed or leave the table read-only rather than guessing
that mutation is safe.

## Assessment model

Each assessment records six independent dimensions:

- `FILE_LAYOUT`
- `DELETION_VECTORS`
- `TRANSACTION_LOG`
- `RETENTION`
- `CLUSTERING`
- `PROTOCOL`

Every dimension carries completeness, provenance, observed version, timestamp, and an incomplete
reason when needed. Missing evidence remains `PARTIAL` or `UNKNOWN`; it is not converted to zero.
A bounded 1 MiB histogram records file-size distribution without sorting every active file in the
server. Policies query that stored evidence later.

## Modules

| Module | Responsibility |
|---|---|
| `fiq-domain` | Domain types, policy evaluation, capability rules, lifecycle, and RBAC model |
| `fiq-delta` | Classic Delta metadata reading and assessment |
| `fiq-engine-spark` | Livy protocol and execution adapter |
| `fiq-spark-job` | Spark/Delta discovery, preflight, mutation, and result writing |
| `fiq-server` | REST API, persistence, scheduling, authentication, verification, and SPA hosting |
| `fiq-ui` | React operator interface |

## Operation lifecycle

The intended lifecycle is:

```text
PLANNED → AWAITING_APPROVAL → QUEUED → RUNNING → terminal
                                      ↘ CANCELLING ↗
```

Terminal outcomes include `SUCCEEDED`, `FAILED`, `SKIPPED`, and `CANCELLED`. A stale planned
Delta version is skipped rather than executed. If mutation may have happened but verification
fails, the operation records that ambiguity so an operator does not blindly replay it.

PostgreSQL claims queued work with `FOR UPDATE SKIP LOCKED`, and a partial unique index limits a
table to one active queued/running/cancelling operation. Policy fires use durable keys to avoid
intentionally creating the same scheduled run twice.

## Current engineering boundary

The architecture has a functioning end-to-end core, but several guarantees are not complete
enough for a hostile or large shared deployment:

- OIDC users are not yet backed by a complete workspace-membership model.
- An approved plan does not yet persist an immutable copy of every execution setting.
- External Livy submission and the following database state change are not one atomic action.
- Cancellation records the request before Spark termination has been independently confirmed.
- Audit writes and every related state mutation are not yet covered by a single transaction.
- Discovery, assessment, and driver-side result collection have not been qualified at large-fleet
  scale.

These are the main reasons Phase 1 is described as a Classic Delta Maintenance Core rather than a
production-ready multi-tenant control plane. The exact tested baseline and deferred features are
kept in [`phase-status.md`](phase-status.md).
