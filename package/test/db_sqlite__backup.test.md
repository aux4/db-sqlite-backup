# db sqlite backup and restore

Provider commands (contributed by `aux4/db-sqlite-backup`) that wrap SQLite's
`VACUUM INTO` online backup. The database is supplied via `--database` (or a
`config.yaml` profile) and the destination via `--path` (or `--dir` + `--file`).
`backup` prints a result manifest to stdout; `restore` replaces the database
with a backup file after verifying its integrity. No system dependencies are
needed — the backup runs through the `aux4/db-sqlite` driver.

```file:config.yaml
config:
  test:
    database: ./bkptest.db
```

```beforeAll
aux4 db sqlite execute --database ./bkptest.db --query "CREATE TABLE items (id INTEGER PRIMARY KEY, name TEXT, qty INTEGER)"
```

```beforeAll
aux4 db sqlite execute --database ./bkptest.db --query "INSERT INTO items (id, name, qty) VALUES (1, 'apple', 10), (2, 'pear', 20), (3, 'plum', 30)"
```

```afterAll
rm -rf bkptest.db bkptest.db-wal bkptest.db-shm backups fresh.db
```

## backup with --database and --path

### should write the backup and print a manifest

```timeout
120000
```

```execute
aux4 db sqlite backup --database ./bkptest.db --path ./backups/full.db
```

```expect:regex
\{"bytes":"\d+","checksum":"[a-f0-9]{64}","format":"sqlite-vacuum","path":"\./backups/full\.db","status":"success"\}
```

### should create a readable SQLite backup

```timeout
120000
```

```execute
aux4 db sqlite execute --database ./backups/full.db --query "SELECT COUNT(*) AS n FROM items" | jq -c .
```

```expect
[{"n":3}]
```

## backup with a config profile

### should resolve the database from the config profile

```timeout
120000
```

```execute
aux4 db sqlite backup --configFile config.yaml --config test --path ./backups/fromconfig.db | jq -r .status
```

```expect
success
```

## backup path resolution

### should resolve the path from dir + file

```timeout
120000
```

```execute
aux4 db sqlite backup --database ./bkptest.db --dir ./backups --file dirfile.db | jq -r .path
```

```expect
./backups/dirfile.db
```

### should append a .db extension when the path has none

```timeout
120000
```

```execute
aux4 db sqlite backup --database ./bkptest.db --path ./backups/noext | jq -r .path
```

```expect
./backups/noext.db
```

### should replace an existing backup file

```timeout
120000
```

```execute
aux4 db sqlite backup --database ./bkptest.db --path ./backups/full.db | jq -r .status
```

```expect
success
```

## backup with no path

### should fail fast when neither path nor dir/file is given

```timeout
120000
```

```execute
aux4 db sqlite backup --database ./bkptest.db
```

```error:partial
Error: provide --path, or --dir and --file
```

## backup with a missing database

### should fail instead of backing up an empty database

```timeout
120000
```

```execute
aux4 db sqlite backup --database ./does-not-exist.db --path ./backups/missing.db
```

```error:partial
Error: database file not found
```

### should not leave an artifact behind

```timeout
120000
```

```execute
test -e ./backups/missing.db && echo leftover || echo "cleaned up"
```

```expect
cleaned up
```

## restore into the same database

### should restore the backup and print an outcome

```timeout
120000
```

```execute
aux4 db sqlite execute --database ./bkptest.db --query "DELETE FROM items" >/dev/null
aux4 db sqlite restore --database ./bkptest.db --path ./backups/full.db
```

```expect
{"action":"restore","database":"./bkptest.db","path":"./backups/full.db","status":"success"}
```

### should bring the rows back

```timeout
120000
```

```execute
aux4 db sqlite execute --database ./bkptest.db --query "SELECT * FROM items ORDER BY id" | jq -c .
```

```expect
[{"id":1,"name":"apple","qty":10},{"id":2,"name":"pear","qty":20},{"id":3,"name":"plum","qty":30}]
```

## restore into a fresh database

### should restore the backup into a different file

```timeout
120000
```

```execute
aux4 db sqlite restore --database ./fresh.db --path ./backups/full.db | jq -r .status
```

```expect
success
```

### should have the rows in the fresh database

```timeout
120000
```

```execute
aux4 db sqlite execute --database ./fresh.db --query "SELECT * FROM items ORDER BY id" | jq -c .
```

```expect
[{"id":1,"name":"apple","qty":10},{"id":2,"name":"pear","qty":20},{"id":3,"name":"plum","qty":30}]
```

## restore with a missing file

### should fail fast when the backup file does not exist

```timeout
120000
```

```execute
aux4 db sqlite restore --database ./bkptest.db --path ./does-not-exist.db
```

```error:partial
Error: backup file not found: ./does-not-exist.db
```

## restore with a corrupt backup

### should refuse to restore a file that is not a SQLite database

```timeout
120000
```

```execute
mkdir -p backups
echo "this is not a database" > ./backups/corrupt.db
aux4 db sqlite restore --database ./bkptest.db --path ./backups/corrupt.db
```

```error:partial
failed PRAGMA integrity_check
```

### should leave the existing database untouched

```timeout
120000
```

```execute
aux4 db sqlite execute --database ./bkptest.db --query "SELECT COUNT(*) AS n FROM items" | jq -c .
```

```expect
[{"n":3}]
```
