#### Description

The `restore` command replaces a SQLite database with a previously taken backup file. The backup is given as `--path`, or as `--dir` plus `--file`; the database to restore into is `--database`.

Because restoring overwrites the target database, the backup is validated first. `PRAGMA integrity_check` is run against the backup file, and the restore is aborted if it does not return `ok` — the existing database is left untouched. This also catches the case where the file is not a SQLite database at all. Pass `--integrityCheck false` to skip the check on very large databases where the scan is too expensive.

After copying the backup into place, any stale `-wal` and `-shm` sidecar files belonging to the previous database are removed, so the restored database is never paired with the old database's journal.

Restore is offline: it replaces the database file, so no other process should be writing to it at the time.

#### Usage

```bash
aux4 db sqlite restore --database <file> --path <file> [--integrityCheck <true|false>]
```

--database        Path to the database file to restore into (replaced by the backup)
--path            Full path to the backup file (takes precedence over --dir/--file)
--dir             Directory containing the backup file (combined with --file)
--file            File name of the backup (combined with --dir)
--integrityCheck  Verify the backup with PRAGMA integrity_check first (default: true)

#### Example

```bash
aux4 db sqlite restore --database ./app.db --path ./backups/app-2026-07-28.db
```

```text
{"path":"./backups/app-2026-07-28.db","database":"./app.db","status":"success","action":"restore"}
```
