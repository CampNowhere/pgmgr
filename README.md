# Postgres Manager (pgmgr)
[![Build Status](https://travis-ci.org/rnubel/pgmgr.svg?branch=master)](https://travis-ci.org/rnubel/pgmgr)

Utility for web applications to manage their Postgres database in a
reliable, consistent manner. Inspired by [mattes/migrate]
(http://www.github.com/mattes/migrate), but with several benefits:

* Migration version numbers are timestamp-based, not sequential. This saves
  significant headaches when multiple developers are working on the same
  application in parallel. However, `pgmgr` is still compatible with your
  old migrations.
* `pgmgr` can generate database dumps, with seed data included, so that
  there's a single, authoritative source of your database structure. It's
  recommended you regularly check this file into source control.

## Getting Started

First, create a `.pgmgr.json` file in your app, as described below. Then,
generate your first migration:

```
$ pgmgr migration MyFirstMigration

Created migrations/1433277961_MyFirstMigration.up.sql
Created migrations/1433277961_MyFirstMigration.down.sql
```

Flesh it out:
```
$ echo 'CREATE TABLE foos (foo_id INTEGER)' > 1433277961_MyFirstMigration.up.sql
```

Bootstrap your database:
```
$ pgmgr db create
Database pgmgr-test-app created successfully.
```

And apply your migration:
```
$ pgmgr db migrate
== Applying 1433277961_MyFirstMigration.up.sql ==
== Completed in 8 ms ==
```

## Configuration

`pgmgr` supports file-based configuration (useful for checking into your
source code) and environment-based configuration, which always overrides
the former (useful for production deploys, Docker usage, etc).

### .pgmgr.json

By default, `pgmgr` will look for a file named `.pgmgr.json` in your
working directory. You can override the file path with the environment
variable `PGMGR_CONFIG_FILE`. It should look something like:

```
{
  "host": "localhost",
  "port": 5432,
  "username": "test",
  "password": "test",
  "database": "testdb",
  "sslmode": "disable",
  "migration-table": "public.schema_migrations",
  "migration-folder": "db/migrate",
  "dump-file": "db/dump.sql",
  "column-type": "integer",
  "format": "unix",
  "seed-tables": [ "foos", "bars" ]
}
```

The `column-type` option can be `integer` or `string`, and determines
the type of the `schema_migrations.version` column. The `string` column
type will store versions as `CHARACTER VARYING (255)`.

The `format` option can be `unix` or `datetime`. The `unix` format is
the integer epoch time; the `datetime` uses versions similar to ActiveRecord,
such as `20150910120933`. In order to use the `datetime` format, you must
also use the `string` column type.

The `migration-table` option can be used to specify an alternate table name
in which to track migration status. It defaults to the schema un-qualified
`schema_migrations`, which will typically create a table in the `public`
schema unless the database's default search path has been modified. If you
use a schema qualified name, pgmgr will attempt to create the schema first
if it does not yet exist.

`migration-driver`, added in August 2019, allows migrations to be run either
through the Go `pq` library (which runs the migrations as a single multi-statement
command) or through the `psql` command-line utility. The possible options are
`'pq'` or `'psql'`. The default is currently `pq` (subject to change).

### Locking & retry configuration
On a high-traffic database, attempts within your migrations to lock tables for DDL can often run into locks. To ensure that your patches don't inhibit other activity and to allow them to safely get through these lock scenarios, pgmgr comes with a default set of timeouts and a retry loop. Those timeouts are:

* 200ms to obtain any needed locks
* 1000ms to apply any individual statement

If you need longer timeouts, set them within your migration SQL (e.g. `SET statement_timeout TO 100000`).

The retry behavior will retry any migration if it detects it as having failed for a lock-related reason. It will attempts up to 10 additional times with a 5-second sleep between each try by default. To customize that behavior, pass either `--max-retries=N`, `--retry-delay=N`, export `PGMGR_MAX_RETRIES=N`, `PGMGR_RETRY_DELAY=N`, or add a `lock-config` block to your `.pgmgr.json` file:

```json
  "lock-config": {
    "max-retries": 50,
    "retry-delay": 1
  }
```

### Environment variables

The values above map to these environment variables:

* `PGMGR_HOST`
* `PGMGR_PORT`
* `PGMGR_USERNAME`
* `PGMGR_PASSWORD`
* `PGMGR_DATABASE`
* `PGMGR_SSLMODE`
* `PGMGR_DUMP_FILE` (the filepath to dump the database definition out to)
* `PGMGR_SEED_TABLES` (tables to include data with when dumping the database)
* `PGMGR_COLUMN_TYPE`
* `PGMGR_FORMAT`
* `PGMGR_MIGRATION_TABLE`
* `PGMGR_MIGRATION_DRIVER`

If you prefer to use a connection string, you can set `PGMGR_URL` which will supersede the other configuration settings, e.g.:

```
PGMGR_URL='postgres://test:test@localhost/testdb?sslmode=require'
```

Also, for host, port, username, password, and database, if you haven't set a
value via the config file, CLI arguments, or environment variables, pgmgr will
look at the standard Postgres env vars (`PGHOST`, `PGUSERNAME`, etc).

## Usage

```
pgmgr migration MigrationName   # generates files for a new migration
pgmgr db create                 # creates the database if it doesn't exist
pgmgr db drop                   # drop the database
pgmgr db migrate                # apply un-applied migrations
pgmgr db rollback               # reverts the latest migration, if possible.
pgmgr db load                   # loads the schema dump file from PGMGR_DUMP_FILE
pgmgr db dump                   # dumps the database structure & seeds to PGMGR_DUMP_FILE
```
