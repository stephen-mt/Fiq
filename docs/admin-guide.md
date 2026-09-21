# Administration

Create workspaces and environments before exposing connections. Assign the smallest role:
`VIEWER` reads fleet state, `OPERATOR` assesses/plans/executes, `APPROVER` decides gated work, and
`ADMIN` manages connections and access. The backend checks every resource; hiding a UI control is
not an authorization boundary.

Policies select environment, catalog, namespace, table, and tags. Currently only OPTIMIZE BINPACK
and VACUUM FULL can mutate tables. OPTIMIZE thresholds include small-file size/count/ratio,
rewrite bytes, target size, and expected reduction. VACUUM thresholds include retention,
candidate count, and measured reclaimable bytes. Values such as 128 MiB, 20 files, and 30% are
editable templates—not hidden assessment logic.

VACUUM always performs a real DRY RUN. Approval retains the target/version, canonical sorted
candidate hash, candidate count, retention, and planning time. Retention below 168 hours is not
qualified except zero-hour on a table marked `sample=true`; that isolated case still always needs
approval. Never enable retention-check bypass globally.

Use UTC in persistence. The UI presents the workspace timezone. Audit rows are protected by a
database trigger against updates and deletes. Database superusers can still bypass application
controls, so restrict that role and export audit data to immutable retention storage.
