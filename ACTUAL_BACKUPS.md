# Actual backups

Actual uses the [common backup role](../roles/backup/README.md) once per configured
budget Sync ID. Each has independent scheduling, staging, retention, and a restic
repository at `rclone:r2:backups/restic/actual/SYNC_ID`. One budget's export failure
cannot prevent another budget from being backed up or advance its retention.
The old ZIPs at `r2:backups/actual` are untouched.

**Timers are disabled and stopped by default.** The current development server
image is ahead of the pinned 26.9.0 API, with a known incompatible migration.
This change preserves both versions. Once a compatible API/server version is in
place, run each backup manually, verify it, then set `actual_backup_enabled: true`
and deploy the Actual role. Do not downgrade an already migrated budget simply
to match the older API.

## Export and validation

`actual-backup-export EMPTY_DIRECTORY SYNC_ID` runs the pinned `@actual-app/api`
in the existing Node image. Server and optional budget encryption passwords are
passed through environment variables, not process arguments. Each job uses its
own temporary directory and Docker container. Export failures and cancellation
remove the temporary container and API working files.

The API writes its native ZIP export. The exporter checks the ZIP layout and
unpacks `db.sqlite` and `metadata.json` into stable filenames, verifying CRCs while
reading. Unexpected archive contents fail rather than being silently discarded.
`backup-info.json` records the Sync ID, API version, and configured server image.
Only the unpacked tree enters restic; the transient ZIP and API cache are removed.

`actual-backup-validate DIRECTORY` checks required files, JSON objects, SQLite
integrity, and the presence of database tables. Restic encrypts and deduplicates
the validated export and applies the same success-gated retention as Forgejo.
The API export should contain everything required to import that budget; server
login configuration and other server-wide state are not part of these exports.

`actual_backup_password` remains the server login password.
`actual_backup_restic_password` is a separate SOPS-encrypted repository password
in `group_vars/actual/backup.secrets.sops.yaml`. It is shared by the independent
budget repositories. Keep access to SOPS recovery keys outside the server.

## Operation

Replace `SYNC_ID` below with an ID from `group_vars/actual/main.yaml`. As the
configured login user:

```sh
actual-SYNC_ID-backup
source ~/.config/actual-backup/SYNC_ID/restic.env
restic snapshots --tag complete
restic check --read-data
```

`actual-backup` remains a manual convenience command: it attempts every configured
budget and returns nonzero if any failed. `actual-backup maintenance` does the
same for maintenance. Manual commands work even while timers are disabled.

The corresponding units are `actual-SYNC_ID-backup.service` and
`actual-SYNC_ID-backup-maintenance.service`, each with a timer. The previous
aggregate `actual-backup.timer` is stopped, disabled, and removed on deployment.
Timers for removed Sync IDs are disabled; their repositories and local data are
preserved for recovery. A timer already in flight may finish its current service.

Exclude the entire restic prefix from R2 object-expiration policies. Retention
keeps 7 daily, 4 weekly, and 6 monthly selections by default; weekly maintenance
reclaims unreferenced data and reads a random 10% sample for integrity checking.
Override the `actual_backup_keep_*`, schedule, and timeout defaults as needed.

## Restore

Select a complete snapshot, then restore it to a new directory:

```sh
source ~/.config/actual-backup/SYNC_ID/restic.env
restic restore SNAPSHOT_ID --target /var/tmp/actual-restore
```

With default paths, its payload is under
`/var/tmp/actual-restore/var/tmp/actual-backup-stage/SYNC_ID/payload`.
Validate it and recreate the native import ZIP:

```sh
cd /var/tmp/actual-restore/var/tmp/actual-backup-stage/SYNC_ID/payload
/usr/local/libexec/actual-backup-validate .
python3 -m zipfile -c /var/tmp/actual-restored-budget.zip db.sqlite metadata.json
```

Import that ZIP into an isolated Actual instance with a compatible version, using
Actual's native import flow. `backup-info.json` is recovery context and is not
included in the import ZIP. Verify accounts, balances, and recent transactions
before using the restored budget in production. The restored archive is plaintext;
handle it with the same care as the original financial data.

Local tests exercise the ZIP-to-tree export, validation failures, independent
budget jobs, and a real restic restore/repack cycle with synthetic data. They do
not demonstrate that the currently mismatched API can export your live budgets.
