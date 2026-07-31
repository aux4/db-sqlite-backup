# aux4/db-sqlite-backup

Backup and restore commands for SQLite databases, contributed to the `db:sqlite` profile.

This package extends `aux4/db-sqlite` with two sibling subcommands, `backup` and `restore`, that follow the backup provider contract used by `aux4/backup`. It can be used on its own, or registered as a provider so the orchestrator schedules, catalogs and prunes the backups for you.

Unlike the MySQL provider, this package needs **no system dependencies** — SQLite's online backup is performed with `VACUUM INTO`, which runs through the existing `aux4/db-sqlite` driver. There is no `sqlite3` CLI to install.

## Installation

```bash
aux4 aux4 pkger install aux4/db-sqlite-backup
```

## Why not just copy the file?

A SQLite database is a single file, so copying it looks tempting — but a plain `cp` of a live database can capture a torn, half-written page and misses anything still sitting in the write-ahead log. The result is a backup that may not open.

`VACUUM INTO` takes a consistent snapshot while other connections keep reading and writing, and compacts the result as it goes. That is what this package uses.

## Usage

```bash
# Back up a database
aux4 db sqlite backup --database ./app.db --path ./backups/app-2026-07-28.db

# Split the destination into directory + file
aux4 db sqlite backup --database ./app.db --dir ./backups --file app.db

# Restore, replacing the target database
aux4 db sqlite restore --database ./app.db --path ./backups/app-2026-07-28.db
```

`backup` prints a result manifest to stdout:

```json
{
  "path": "./backups/app-2026-07-28.db",
  "bytes": 8192,
  "checksum": "5f2b...",
  "status": "success",
  "format": "sqlite-vacuum"
}
```

## Commands

| Command | Description |
|---------|-------------|
| `db sqlite backup` | Write a compacted, consistent copy of the database using `VACUUM INTO` |
| `db sqlite restore` | Replace a database with a backup file, after checking its integrity |

### backup

```bash
aux4 db sqlite backup --database <file> --path <file>
aux4 db sqlite backup --database <file> --dir <dir> --file <name>
```

| Option | Description |
|--------|-------------|
| `--database` | Path to the SQLite database file to back up |
| `--path` | Full path to write the backup to (takes precedence over `--dir`/`--file`) |
| `--dir` | Directory to write the backup into (combined with `--file`) |
| `--file` | File name for the backup (combined with `--dir`) |

The destination directory is created if missing. When the resolved path has no extension, `.db` is added — so an orchestrator can pass an extension-less base path and let this package name the artifact for its own format. An existing file at the destination is replaced.

### restore

```bash
aux4 db sqlite restore --database <file> --path <file> [--integrityCheck false]
```

| Option | Description |
|--------|-------------|
| `--database` | Path to the database file to restore into (replaced by the backup) |
| `--path` | Full path to the backup file (takes precedence over `--dir`/`--file`) |
| `--dir` | Directory containing the backup file (combined with `--file`) |
| `--file` | File name of the backup (combined with `--dir`) |
| `--integrityCheck` | Run `PRAGMA integrity_check` on the backup before replacing anything (default: `true`) |

Restore replaces the target database with the backup file, then removes any stale `-wal` and `-shm` sidecar files so the restored database is not mixed with the previous database's journal.

Because restore overwrites the target, the backup is verified first: if `PRAGMA integrity_check` does not return `ok`, the restore is aborted and the existing database is left untouched. Pass `--integrityCheck false` to skip that check on very large databases.

## Use with aux4/backup

Register the database as a backup target and the orchestrator handles scheduling, cataloging, retention and restores:

```bash
aux4 backup register app \
  --command "aux4 db sqlite" \
  --config app \
  --configFile ./db.yaml \
  --dir /var/backups/app \
  --retain 7

aux4 backup run app --wait true
aux4 backup verify app
aux4 backup restore app
```

Connection settings come from a `config.yaml` profile, so no credentials or paths are stored in the backup catalog:

```yaml
config:
  app:
    database: /srv/app/app.db
```

## License

Apache-2.0
