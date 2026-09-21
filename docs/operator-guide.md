# Operator guide

For the first-run UI workflow and connection examples, see the
[end-user guide](end-user-guide.md).

## Minimal FIQ runtime

FIQ has one required runtime dependency: PostgreSQL. Start the server, UI, and database with:

```bash
make up
```

Open `http://localhost:9091`. An empty connection/table list is healthy. The server does not need
MinIO, HMS, Livy, or sample data to start and does not probe them at startup. Connection records
carry their own execution endpoint; testing or using a connection is what contacts external
infrastructure.

Use `make ps`, `make logs`, and `make down`. The minimal Compose file is intentionally limited to
`postgres` and `server`.

## Self-contained local demo

Requirements are Java 21, Docker with Compose, Node 24, `curl`, and `jq`. Start the demo with:

```bash
make demo-up
```

This uses `compose.yaml + compose.demo.yaml`. Development auth is disabled. PostgreSQL is on
5432, MinIO on 9000/9001, HMS on 9083, Livy on 8998, and FIQ on 9091. These credentials are
development only. Do not reuse them outside an isolated workstation.

The sample initializer creates PATH small files, an HMS small-file table, deletion-vector and
history/checkpoint fixtures, and an isolated VACUUM fixture. The server automatically registers
the sample PATH/HMS connections and discovers them. The initializer and discovery upserts are
idempotent; `FIQ_SAMPLE_ENABLED` is false outside Compose by default.

The first clean startup can take several minutes because Spark resolves Delta, Hadoop AWS, and
Hive client artifacts into the `fiq-ivy` volume. Keep that volume for normal restarts; subsequent
starts reuse it. Wait for `sample-init` to exit successfully and `server` to start:

```bash
make demo-ps
make demo-logs
```

To keep the stack running after closing the terminal:

```bash
make demo-up
make demo-ps
make demo-logs
```

`Ctrl+C` only stops log following. `make demo-down` preserves demo state. `make demo-reset`
deletes only the selected `fiq-demo` Compose project's containers and volumes; it never prunes
unrelated Docker resources.

## End-user workflow

1. **Connections**: create PATH/HMS, test the qualified runtime, then discover. PATH always needs
   an explicit bounded root; HMS enumerates Delta-provider tables through Spark.
2. **Tables**: confirm PATH or CATALOG execution target and the Sample label. Refresh assessment.
3. **Health**: inspect completeness/provenance for all six dimensions. UNKNOWN means not measured,
   never zero.
4. **Policies**: choose only OPTIMIZE BINPACK or VACUUM FULL and configure thresholds. These values
   evaluate the assessment; they do not affect measurement.
5. **Table → Policies**: create a plan. Review the exact target, observations, threshold
   comparisons, blocker reasons, and VACUUM preflight identity.
6. **Operations**: queue or approve, then inspect ASSESS, PREFLIGHT, EXECUTION, and VERIFICATION.
   A Spark job alone is not success; verified before/after evidence is required.

Run `make demo-test` after startup for the destructive isolated sample acceptance.
It proves PATH `forPath`, HMS `forName`, measurable OPTIMIZE reduction, and VACUUM FULL
dry-run/approval/apply/verify. Never point that script at non-sample data.

The upstream Livy 0.9 distribution is built for Scala 2.12 while Spark 4 uses Scala 2.13. FIQ uses
Livy's batch endpoint, where the submitted application owns its Scala 2.13 classpath, and probes
reachability before use. Production must use a vendor-tested Livy/Spark 4 deployment; do not infer
support from the Livy server version alone.

## Production configuration

Use external PostgreSQL and configure Spark/Livy endpoints on each connection. Set
`FIQ_DATABASE_*` and the OIDC values. Set the optional `FIQ_S3_ENDPOINT` only when embedded Delta
Kernel or structured-result reads need an S3-compatible endpoint such as MinIO. The OIDC client
uses Authorization Code + PKCE and a secure SameSite cookie. Keep `FIQ_AUTH_ENABLED=true` and
`FIQ_COOKIE_SECURE=true`. Connections store only a secret reference; credentials arrive through
workload identity, cloud credential chains, or Kubernetes Secrets.

Run `/q/health/ready` for readiness, `/q/health/live` for liveness, and scrape `/q/metrics`.
Back up PostgreSQL with PITR. Restore into a separate instance, run Flyway validation, then admit
one FIQ replica before scaling out. Spark jobs already accepted by Livy must be reconciled by job
ID; never replay a run by changing its database state.

## Incident actions

- Livy unavailable: queued submissions back off exponentially and fail after the configured cap.
- Lost server replica: leases expire and another replica resumes polling without resubmission.
- Stale table version: the run becomes `SKIPPED`; refresh and create a new plan.
- Unknown Delta feature: the table remains observable but maintenance is read-only.
- VACUUM LITE or another retained operation type: it has no qualified executor and remains
  read-only.
