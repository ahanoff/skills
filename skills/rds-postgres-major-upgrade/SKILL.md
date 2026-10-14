---
name: rds-postgres-major-upgrade
description: |-
  Plan and run a major version upgrade of an Amazon RDS for PostgreSQL instance managed by Pulumi (or Terraform) with an RDS Blue/Green deployment, keeping a real rollback.

  Use when the user wants to upgrade RDS PostgreSQL across major versions (for example 16 to 18), asks about `blueGreenUpdate` / `blue_green_update`, wants to create or switch over an RDS Blue/Green deployment, or needs to reconcile Pulumi state after one.

  Do not use for Aurora clusters, MySQL/MariaDB, or minor version bumps that need no Blue/Green.
compatibility: Documentation-only skill. Drives the aws, pulumi and psql CLIs by command name; no runtime dependencies.
metadata:
  version: "1.0.0"
---

# RDS PostgreSQL major version upgrade with Blue/Green

This skill moves a production RDS for PostgreSQL instance to a new major version through an RDS Blue/Green deployment driven through the RDS API instead of the IaC resource, then brings Pulumi state back in line. It keeps the old instance and its point-in-time recovery (PITR) history as a rollback.

## Ground rules

- **Every step that changes a production database needs the user's explicit go**, inside their approved maintenance window. Prepare commands and run read-only checks ahead of time; execute only when told.
- Never hardcode account IDs or credentials. Ask which AWS profile and region to use, and load secrets (such as a Pulumi passphrase) only inside the command that needs them.
- After every write, read the resource back and confirm the change before reporting it. Record UTC timestamps for the change log.
- **Ask how production changes are made** before preparing anything. Neither CloudFormation nor the Terraform and Pulumi AWS providers have a resource for the Blue/Green deployment itself, so the runbook is always RDS API calls. The commands below use the AWS CLI. Where changes must go through pipelines, approvals and an audit trail, prepare the same calls as pipeline jobs, one per step with a manual approval before the switchover, or as an SSM Automation runbook, instead of running them from a terminal.
- Run database checks read-only unless the step says otherwise: `PGOPTIONS='-c default_transaction_read_only=on'`. If the app's `DATABASE_URL` carries query parameters `psql` rejects (for example `?encoding=utf8`), strip them first.

## Choose the path

| Situation | Path |
|---|---|
| Dev, staging, or a database where a pre-upgrade snapshot is enough rollback | Pulumi `blueGreenUpdate: { enabled: true }` is acceptable. Take a manual snapshot first. |
| Anything that needs the old instance, its PITR history, or a check before switchover | **Blue/Green through the RDS API**, then reconcile state. Follow this skill. |
| Small database and the window allows downtime | In-place upgrade (`engineVersion` bump, `allowMajorVersionUpgrade: true`, `applyImmediately: true`) with application traffic stopped. |

**Don't use `blueGreenUpdate` where the old instance is the rollback.** It is bridged from the Terraform AWS provider's `blue_green_update`, which runs create, switchover and delete in one resource update and:

- deletes the old instance with `SkipFinalSnapshot: true` hard-coded, ignoring the resource's `skipFinalSnapshot` and `finalSnapshotIdentifier`;
- doesn't pass `DeleteAutomatedBackups`, so the RDS API default also deletes the old instance's automated backups and PITR history;
- switches over as soon as green is ready, with no chance to validate it;
- has to finish inside the update timeout (80 minutes by default) and the CI credentials' lifetime. If credentials expire after the switchover, the stack still points at the old instance and the state needs the repair described below.

## Preflight (read-only, days before)

1. **Target version available in the region:** `aws rds describe-db-engine-versions --engine postgres --engine-version <target>`.
2. **Instance facts:** from `aws rds describe-db-instances`, record `EngineVersion`, `MultiAZ`, `AllocatedStorage`, `MaxAllocatedStorage`, `DBParameterGroups[].ParameterApplyStatus`, `BackupRetentionPeriod` (must be above 0) and `DeletionProtection`.
3. **Disk space.**
    - **Autoscaling ceiling.** Creation fails with `InvalidParameterValue ... maximum storage threshold ... at least 10% higher than the target allocated storage` when `MaxAllocatedStorage` is below 1.1 × `AllocatedStorage`. Raise `maxAllocatedStorage` in code before the window; it's an online change.
    - **Free space on blue.** While the deployment exists, a logical replication slot on blue retains every WAL segment green hasn't consumed. Check CloudWatch `FreeStorageSpace` against `TransactionLogsGeneration` at peak; AWS recommends at least a day's worth of WAL as extra free space. If blue doesn't have it, raise allocated storage or the ceiling first. Don't plan to leave a deployment waiting for days before switchover.
4. **Replication workers.** Blue/Green needs at least one logical replication worker per database. Check `max_replication_slots`, `max_wal_senders`, `max_logical_replication_workers` and `max_worker_processes` against the number of databases and any existing replication. Set them in both parameter groups if the defaults aren't enough; they also need the reboot.
5. **Tables without a primary key.** While the deployment exists, all tables are published for logical replication, and PostgreSQL rejects UPDATE and DELETE on tables with no replica identity on the live database. Run per database:

    ```sql
    SELECT n.nspname, c.relname, c.relreplident,
           s.n_live_tup, s.n_tup_upd, s.n_tup_del, pg_get_userbyid(c.relowner) AS owner
    FROM pg_class c
    JOIN pg_namespace n ON n.oid = c.relnamespace
    LEFT JOIN pg_stat_user_tables s ON s.relid = c.oid
    WHERE c.relkind IN ('r','p')
      AND n.nspname NOT IN ('pg_catalog','information_schema')
      AND NOT EXISTS (SELECT 1 FROM pg_constraint k WHERE k.conrelid = c.oid AND k.contype = 'p');
    ```

    Also check large objects (`SELECT count(*) FROM pg_largeobject_metadata`), which aren't replicated, and unlogged tables.
6. **Schema changes.** No migrations or other DDL may run between creating the deployment and switching over; DDL breaks replication and the deployment must be recreated. Schedule application releases with migrations after the switchover.
7. **How applications connect.** Confirm every client uses the instance endpoint hostname, not an IP. List every service that holds a connection pool; all of them will be restarted after the switchover.
8. **Rehearse on a non-production copy** and note how long green creation takes.

## Code changes before the window

Merge and deploy these to production ahead of the window:

- A parameter group for the **current** major with `rds.logical_replication = 1` and `applyMethod: 'pending-reboot'`, attached to the instance.
- A parameter group for the **target** major with the same parameters, including `rds.logical_replication = 1`, created but not attached. It becomes green's group through `--target-db-parameter-group-name` and production's after the switchover. Never let green fall back to `default.postgres<target>`: logical replication would be off on the upgraded instance, and the next Blue/Green would need another failover reboot. Keep the two groups identical apart from `family`.
- A higher `maxAllocatedStorage` or allocated storage if preflight step 3 needs it.
- `engineVersion` left at the current version, and no `blueGreenUpdate`.

The parameter group attaches as `pending-reboot`; the database doesn't change until step 1 below.

## Runbook (in the window)

1. **Snapshot first (optional).** `aws rds create-db-snapshot`. Take it before the reboot: a snapshot right after a Multi-AZ failover is a full copy that can take hours, and `create-blue-green-deployment` fails with `InvalidDBInstanceState` while the instance is `backing-up`.
2. **Reboot with failover** so logical replication takes effect: `aws rds reboot-db-instance --db-instance-identifier <db> --force-failover`. Expect a short burst of dropped connections. Confirm `ParameterApplyStatus` is `in-sync` and `SHOW wal_level` returns `logical`.
3. **Set replica identity** on each table found in preflight step 5. It changes a catalog flag only and doesn't rewrite the table. The connecting role must own the tables.

    ```sql
    SET lock_timeout = '5s';
    SET statement_timeout = '30s';
    ALTER TABLE <schema>.<table> REPLICA IDENTITY FULL;
    ```

    Re-run the preflight query and confirm `relreplident` is `f`.
4. **Create the deployment:**

    ```bash
    aws rds create-blue-green-deployment \
      --blue-green-deployment-name <db>-pg<target> \
      --source arn:aws:rds:<region>:<account>:db:<db> \
      --target-engine-version <target> \
      --target-db-parameter-group-name <target-parameter-group>
    ```

    Poll `aws rds describe-blue-green-deployments` until `Status` is `AVAILABLE`. Its tasks run in order: create the replica, upgrade the engine, configure backups. Green spends much of that time in `storage-initialization`; `describe-db-instances` reports `StorageOperationPercentProgress`. Expect tens of minutes even for a modest database.

    While it runs, and until the switchover, watch blue's `FreeStorageSpace`, `TransactionLogsDiskUsage` and `OldestReplicationSlotLag` in CloudWatch. If free space falls steadily or slot lag keeps growing, tell the user. Deleting the deployment drops the slot and releases the WAL, and is the safe move if the switchover is postponed.
5. **Validate green before the switchover:**
    - green is `available` on the target version, attached to the custom target parameter group (not a `default.*` group) with `ParameterApplyStatus` `in-sync`, and `BackupRetentionPeriod` matches blue;
    - `SHOW rds.logical_replication` on green returns `on`;
    - CloudWatch `OldestReplicationSlotLag` on blue is near zero;
    - exact row counts for every table are identical on blue and green, read-only, queried from the same host.

    Report the results and wait for the user's go.
6. **Switch over:** `aws rds switchover-blue-green-deployment --blue-green-deployment-identifier <id> --switchover-timeout 300`. Poll until `SWITCHOVER_COMPLETED`, usually under a minute. If it can't finish within the timeout, RDS rolls it back and nothing changes.
7. **Restart every application instance** one to two minutes after the switchover, for example `aws ecs update-service --force-new-deployment` per ECS service followed by `aws ecs wait services-stable`, or a rolling restart on Kubernetes. A pool that reconnected while DNS was moving can hold connections to the old instance, which is now read-only. Those connections look healthy and fail only on writes, with errors such as `cannot execute INSERT in a read-only transaction`.
8. **Verify:**
    - the application's hostname resolves to the new instance and `SELECT version()` shows the target version;
    - application hosts appear as `client_addr` in `pg_stat_activity` on the new instance;
    - write counters in `pg_stat_database` rise over 60 seconds;
    - `pg_stat_activity` on `<db>-old1` shows only `rdsadmin` sessions;
    - application logs since the switchover have no read-only or connection errors;
    - whatever tag selects the instance for backup plans is on the new instance. Tags from blue replace green's at switchover, and `-old1` keeps them as well.

## Reconcile Pulumi state

The provider uses the DBI resource ID (`db-…`) as the Pulumi resource ID. After the switchover that ID belongs to `<db>-old1`, so a plain `pulumi refresh` would start tracking the old instance and the next `pulumi up` would try to rename it back. Point the state at the new instance first:

```bash
pulumi stack export --stack <stack> --file before.json   # keep this as a backup

python3 - <<'EOF'
import json
OLD, NEW = "db-OLD...", "db-NEW..."   # DbiResourceId of <db>-old1 and of the new <db>
d = json.load(open("before.json"))
for r in d["deployment"]["resources"]:
    if r["type"] == "aws:rds/instance:Instance" and r["id"] == OLD:
        r["id"] = NEW
        out = r["outputs"]
        out["id"] = NEW
        out["resourceId"] = NEW
        out.pop("__pulumi_raw_state_delta", None)  # stale after the edit; breaks the next read
assert OLD not in json.dumps(d)
json.dump(d, open("fixed.json", "w"), indent=2)
EOF

pulumi stack import --stack <stack> --file fixed.json
pulumi refresh --stack <stack> --target '<rds-instance-urn>' --yes --skip-preview
```

[`scripts/fix-state.ts`](https://github.com/ahanoff/how-to/blob/main/rds-blue-green-pulumi/scripts/fix-state.ts) in the sample code does the same edit and also stops if any other resource still references the old ID.

The refresh should move `engineVersion`, `parameterGroupName` and any setting you changed by hand to the live values.

Then commit the result: `engineVersion` set to the target, the target parameter group attached, no `blueGreenUpdate`, and any storage change. **Its production preview must show no changes.** Anything else means the state or the code is still wrong. Deploy it before any other production release, including the application release that waited for the switchover.

Afterwards, `pulumi refresh --preview-only` across the whole stack is a useful read-only check for unrelated drift. Report drift rather than applying it during the window.

## Rollback

- **Before the switchover:** delete the Blue/Green deployment. Production is untouched, and the replication slot and its retained WAL go away with it.
- **After the switchover:** writes go only to the new instance, and replication doesn't flow back. PITR can't cross a major version. Rolling back means pointing applications at `<db>-old1` (read-only until `default_transaction_read_only` is set to 0 and it's rebooted) or restoring the pre-upgrade snapshot, then re-applying writes made since the switchover by hand. Decide this with the user; never roll back automatically.

## Cleanup (days later, with the user's go)

```bash
aws rds modify-db-instance --db-instance-identifier <db>-old1 --no-deletion-protection --apply-immediately
aws rds delete-db-instance --db-instance-identifier <db>-old1 \
  --final-db-snapshot-identifier <db>-pg<old>-final \
  --no-delete-automated-backups
aws rds delete-blue-green-deployment --blue-green-deployment-identifier <id>
```

`--no-delete-automated-backups` keeps the old version's PITR window until its retention expires. Deleting the deployment record leaves both instances in place. Delete the manual pre-upgrade snapshot when it's no longer needed, and remove the old-major parameter group from code in a follow-up change.
