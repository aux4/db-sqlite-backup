#### Description

The `backup` command writes a consistent, compacted copy of a SQLite database using `VACUUM INTO`. Because SQLite performs this as an online backup, other connections can keep reading and writing while it runs — unlike a plain file copy, which can capture a torn page and miss data still held in the write-ahead log.

No system dependencies are required: the backup runs through the `aux4/db-sqlite` driver, so there is no `sqlite3` CLI to install.

The destination is given as `--path`, or as `--dir` plus `--file`. The destination directory is created if it does not exist, and an existing file at that path is replaced. When the resolved path has no extension, `.db` is appended — this lets `aux4/backup` pass an extension-less base path and leave artifact naming to the provider.

On success a result manifest is printed to stdout, which is what `aux4/backup` records in its catalog. If `VACUUM INTO` fails, the command reports the SQLite error, removes any partial file, and exits non-zero.

#### Usage

```bash
aux4 db sqlite backup --database <file> --path <file>
aux4 db sqlite backup --database <file> --dir <dir> --file <name>
```

--database  Path to the SQLite database file to back up
--path      Full path to write the backup file (takes precedence over --dir/--file)
--dir       Directory to write the backup file (combined with --file)
--file      File name for the backup (combined with --dir)

#### Example

```bash
aux4 db sqlite backup --database ./app.db --path ./backups/app-2026-07-28.db
```

```text
{"path":"./backups/app-2026-07-28.db","bytes":8192,"checksum":"5f2b8c...","status":"success","format":"sqlite-vacuum"}
```
