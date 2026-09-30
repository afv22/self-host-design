# Forgejo backups

Forgejo uses two service-owned helpers and the common backup runner:

1. `forgejo-backup-export DIRECTORY` creates an unpacked dump in an empty directory.
2. `forgejo-backup-validate DIRECTORY SQLITE_RELATIVE_PATH` checks required files,
   the SQLite database's integrity, and the presence of Forgejo's core tables.
3. `forgejo-backup` runs those steps, stores the tree with restic through the
   existing rclone configuration, and applies retention after success.

The [common backup role](../roles/backup/README.md) installs the runner, restic
configuration, and timers. Forgejo supplies its export and validation helpers.
Its existing repository, password, paths, snapshot identity, and unit names are
preserved. To change export or validation, edit the service helper.

## Export and consistency

This implementation is for the current local SQLite deployment. It creates a
one-off container from the running container's exact image, stops Forgejo,
executes `forgejo dump --type tar` against its volumes, and restarts Forgejo
before extracting, validating, or uploading. The dump container has no network.
Forgejo is unavailable while the dump runs; this includes Web and Git access.
The exporter attempts to restart Forgejo on errors and cancellation, and removes
its temporary container. A host crash or forced kill can defeat cleanup: inspect
`forgejo-forge-backup-export`, remove it if necessary, and restart `forgejo-forge`.

The export retains the SQLite database, SQL dump, configuration, repositories,
LFS, attachments, and other included application data. Packages, logs, search
indexes, and generated repository archives are skipped. It does not claim to
exclude all Actions artifacts. The original archive is discarded after extraction;
restic sees individual files and can deduplicate their contents. Git repacking or
large database changes can still cause substantial new data.
Allow local disk space for the uncompressed archive in Docker's storage and,
during extraction, both the archive and extracted tree in the staging directory.

The expected SQLite location inside the dump is `data/gitea.db`, configurable as
`fj_backup_sqlite_path`. A different layout fails validation rather than silently
falling back to the SQL dump. Forgejo's SQL export has known restore caveats;
the SQLite file is the primary database recovery path here.

## Storage and retention

The default repository is `rclone:r2:backups/restic/forgejo`. The old
`r2:backups/forgejo` ZIPs are untouched. **Do not configure R2 object expiration
on the restic prefix**, including any bucket-wide expiration rules. Old data
objects can still be required by recent snapshots. Restic owns deletion.

The role installs distro restic and requires version 0.17 or newer. Rclone must
already be configured by its role (as it is in `site.yaml`). A missing repository
is initialized on the first backup; authentication and connectivity failures do
not trigger initialization.

`fj_backup_password` is encrypted in
`group_vars/forgejo/backup.secrets.sops.yaml` using the repository's age recipient.
The deployed password and `restic.env` live in the login user's
`~/.config/forgejo-backup/`, with restricted permissions. Keep access to the SOPS
key independently of the server; losing the restic password prevents recovery.
Changing the Ansible password does not rotate the repository key: use restic's
key-management commands first if rotation is needed.

Daily backups retain 7 daily, 4 weekly, and 6 monthly restore points, configurable
through `fj_backup_keep_*`. These are overlapping calendar selections, not fixed
age cutoffs or a storage cap. Restic can also retain the oldest snapshot when
there is not yet enough history to satisfy a selection. Weekly maintenance checks the repository, prunes
unreferenced data, and checks a random 10% of stored data. This sample is not a
full read of every object; run a full check periodically or before a restore drill.
Pruning can require temporary storage and additional requests.

Snapshots start with the `pending` tag and are tagged `complete` only after a
successful restic exit. Retention considers only complete snapshots. An unreadable
file (restic exit 3), interrupted run, or failed tagging cannot displace complete
backups. Pending snapshots remain for inspection and can consume storage until
explicitly removed. A failed run retains its local payload for diagnosis; the
next run replaces it. Both timers and manual wrapper runs share a lock.

## Deploy and first verification

Before deploying, confirm that R2 expiration rules exclude the new prefix.
Deploy `playbooks/forgejo.yaml` as usual. Enabling the persistent timer may run
an overdue backup immediately, with the downtime described above.

On the Forgejo host:

```sh
sudo systemctl start forgejo-backup.service
sudo journalctl -u forgejo-backup.service -n 100 --no-pager
```

As the configured login user (so rclone finds the correct credentials):

```sh
source ~/.config/forgejo-backup/restic.env
restic snapshots --tag complete
restic stats --mode raw-data
restic check --read-data
```

Review `result.json` under `fj_backup_stage_dir` after successive successful runs
for restic's bytes processed and data added. Measure a few representative days
before increasing retention. A failed systemd unit is visible in `systemctl
--failed`; this change does not add external failure or missed-backup alerts.

## Restore drill

Use a specific complete snapshot ID from `restic snapshots --tag complete`:

```sh
source ~/.config/forgejo-backup/restic.env
restic restore SNAPSHOT_ID --target /var/tmp/forgejo-restore
```

With the default staging path, the recovered payload is at
`/var/tmp/forgejo-restore/var/tmp/forgejo-backup-stage/payload`:

```sh
/usr/local/libexec/forgejo-backup-validate \
  /var/tmp/forgejo-restore/var/tmp/forgejo-backup-stage/payload data/gitea.db
```

For application verification, use an isolated container and a new data volume:

- Read `image-reference.txt`, `image-id.txt`, and `image-digests.json` to select
  the original image, preferably its recorded registry digest. A local image ID
  identifies the original but is not a pullable registry reference.
- Restore `data/` to the configured `APP_DATA_PATH` (normally `/data/gitea`),
  `repos/` to the configured repository root (normally `/data/git/repositories`),
  and `app.ini` to `/data/gitea/conf/app.ini`. Check paths in the restored config;
  handle `custom/` according to its original location if present. Restore LFS and
  attachments to their configured locations as well.
- Use the restored SQLite database at the configured database `PATH`; preserve
  any associated SQLite WAL files. Do not import `forgejo-db.sql` over it.
- Restore the runtime UID/GID's ownership (currently 1000:1000). Start the same
  Forgejo version with the test volume and isolated networking. Do not publish
  its production Tailscale service or connect it to production runners.
- Verify login, repository browsing and cloning, issues, and any LFS/attachments
  you use. Run `forgejo doctor check --all` against the test instance.

A repository check or SQLite integrity check does not replace this application
restore drill. Only retire legacy ZIPs after the new restore path works.

To inspect interrupted snapshots, run `restic snapshots --tag pending`. Remove
specific unwanted IDs with `restic forget SNAPSHOT_ID` after inspection; weekly
pruning reclaims their unused data. Do not manually delete restic objects with
rclone. Coordinate manual mutations with the timers; direct restic commands do
not acquire the wrapper's local lock.

## Local tests

`tests/test_forgejo_backup.py` exercises validation failures, partial backups,
repository initialization, retention gating, export cleanup, and shell rendering.
It requires Python with Jinja2 and PyYAML:

```sh
python3 -m unittest discover -s tests -v
RESTIC_TEST_BINARY=/path/to/restic python3 -m unittest discover -s tests -v
```

The optional real-restic test backs up twice, verifies deduplication and retention,
restores and validates the files, then runs maintenance against a temporary local
repository. Docker behavior is mocked; this is not a live Forgejo restore test.

References: [Forgejo backup guidance](https://forgejo.org/docs/v16.0/admin/upgrade/#backup),
[restic retention](https://restic.readthedocs.io/en/stable/060_forget.html),
[restic scripting](https://restic.readthedocs.io/en/stable/075_scripting.html).
