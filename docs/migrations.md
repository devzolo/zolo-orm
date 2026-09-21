# Migrations

[Documentation](README.md) · [Runnable migration demo](../examples/migration_demo/README.md)

Migrations turn model changes into reviewed SQL and keep an applied history.
Use them for persistent databases; `create_table` only creates missing objects
and does not reconcile changes to an existing table or index.

## Configure a schema entry

The [demo manifest](../examples/migration_demo/zolo.toml) separates the schema
from the application entry point:

```toml
[database]
schema = "models.zolo"
migrations = "migrations"
dialect = "sqlite"
url_env = "DATABASE_URL"
```

`schema` points to the models, or to an entry that imports them. The CLI
collects generated DDL without running application code. Schema and migration
paths are resolved from the project; `--schema` and `--migrations` override
them. `--url` selects the connection explicitly; otherwise the CLI reads the
configured environment variable.

## Create and apply the first migration

From the repository root:

```sh
cd examples/migration_demo
zolo install
zolo db generate initial
zolo db status --url sqlite://tutorial.db
```

Generation writes a timestamped directory under `migrations/` containing
`up.sql`, `down.sql`, `schema.json` and a checksum manifest. Review those files:

```sh
zolo db migrate --url sqlite://tutorial.db
zolo db check --url sqlite://tutorial.db
zolo run main.zolo --no-cache
```

The model generates the `notes` table and a title index. The program creates
no tables: it writes to the migrated schema and prints:

```text
First note: Created after migration
Notes: 1
```

Running it again updates the same demo row instead of adding another one.
Commit migration files with their model change. Keep the database file local;
this repository ignores `*.db` and its SQLite sidecar files.

## Evolve and inspect

After changing a model, run `zolo db generate <name>`, inspect both SQL
directions, then migrate and check again. Adding `@model(index: true)` to a
field can produce a reversible index migration. With no difference, the
current CLI reports that the schema is already up to date and returns a
nonzero status; no migration is created.

| Command | Purpose |
| --- | --- |
| `generate <name>` | Diff model metadata against the latest migration snapshot. |
| `status` | List applied and pending migrations. |
| `migrate` | Apply pending migrations with their history changes atomically. |
| `check` | Check source/history consistency; with a URL, inspect database drift too. |
| `rollback [N]` | Revert the latest applied migration(s), defaulting to one. |
| `baseline` | Verify an existing matching database and record its initial history. |

Commands also accept `--json` for structured output. `baseline` adopts a
database that already matches the models and has no migration history.
It does not fix a mismatch or import arbitrary SQL.

## Try rollback in the disposable demo

The first migration's down SQL drops the demo table and its rows.
In this scratch database, verify the complete round trip:

```sh
zolo db rollback --url sqlite://tutorial.db
zolo db migrate --url sqlite://tutorial.db
zolo db check --url sqlite://tutorial.db
zolo run main.zolo --no-cache
```

The final run recreates the same single demo row. For application data, review
down SQL before rollback: reverting a schema can discard the data it owns.

## Rebuild a table while preserving its rows

SQLite needs a table copy for changes to existing column definitions and
foreign-key or UNIQUE constraints. Opt in when generating the reviewed plan:

```sh
zolo db generate tighten_notes --allow-rebuild
zolo db migrate --url sqlite://tutorial.db
zolo db check --url sqlite://tutorial.db
```

The generator writes both directions: create a replacement table, copy the
shared columns, replace the original, then recreate indexes. The ordered primary-key
columns, their types and generated identity must stay unchanged. A UNIQUE constraint
checks existing duplicates; making a column required checks existing NULLs.
A SQL default applies to newly added columns and future inserts; it does not
silently repair existing NULLs.

Before copying, the runner checks that the current catalog matches the
previous snapshot, including SQL default values and generated keys. Type
changes use explicit casts and must round-trip without changing the original
value. For example, text `'42'` can become an integer, while `'042'`, fractional
numbers converted to integers and integers that lose precision as floats are
refused. A rollback repeats these checks against the current data: values
written after migration can make a reversal unsafe.

Rebuilds use a dedicated connection. Foreign-key enforcement is suspended
before the transaction, all pending migrations and their history changes stay
atomic, orphan checks run before commit, and enforcement is restored afterward.
An error rolls back the whole batch. Custom data-changing migrations cannot
share that batch: apply them first, then generate the rebuild, so CASCADE and
other referential actions keep their expected behavior.

Rebuild manifests use format 3 and bind the rebuilt-table policy to the
checksum. Their SQL must match the generated schema plan. Keep all four files
together; removing or downgrading the manifest cannot turn the copy into a
legacy migration. Ordinary changes still use format 2, and existing format 1/2
checksums remain compatible. Never edit an applied migration.

The [migration demo](../examples/migration_demo/README.md#preserve-data-through-a-rebuild)
includes an alternate model and a read-only program that checks the same row
before migration, after migration and after rollback.

## Supported changes and explicit boundaries

| Change | Behavior |
| --- | --- |
| New models with foreign keys | Creates parents before children; rollback drops children first. |
| Index addition, removal or replacement | Generates reversible SQL for supported plain, directional, expression and partial indexes. |
| Nullable column, or required column with a SQL default | Adds the column; UNIQUE additions require `--allow-rebuild`. |
| Existing column type, nullability, SQL default or uniqueness | Requires `--allow-rebuild` and compatible existing data. |
| Foreign-key actions or table UNIQUE constraints | Requires `--allow-rebuild`; validates the resulting relations. |
| DROP TABLE or DROP COLUMN | Requires `--allow-destructive`; constrained columns also require a rebuild. |
| Dropping a required column without a SQL default | Refused because its definition cannot be restored for existing rows. |
| New required column without a SQL default | Refused; provide a backfill default first. |
| Changing primary-key columns, order, types or generated identity | Refused by automatic rebuild. |
| Cyclic foreign-key graph | Refused by automatic generation. |

`--allow-destructive` permits removing data; rollback restores a dropped
column with NULL or its SQL default, not its old values. Dropped tables are
recreated empty. Review both SQL directions and keep backups appropriate to
your application.

Automatic rebuilds refuse custom views and triggers, generated SQL columns,
CHECK constraints, custom collations, AUTOINCREMENT, STRICT/WITHOUT ROWID,
named constraints and other catalog details the snapshot cannot preserve.
Inspection preserves the supported index expressions and predicates described
in [advanced schemas](advanced-schemas.md), along with each term's direction.
Expressions outside that vocabulary, custom-collation indexes, deferred foreign
keys and directional primary/UNIQUE constraints remain unsupported. Use an
explicit, reviewed migration workflow for those schemas.

Composite primary and foreign keys retain their declared order in snapshots
and catalog checks. Existing scalar-key snapshots remain readable. A composite
foreign key references one complete primary or UNIQUE key; an expression or
partial UNIQUE index cannot serve as that target. Rebuilds preserve these
constraints and recreate supported indexes after copying rows.

Scripts must not contain `BEGIN`, `COMMIT`, `ROLLBACK` or other
transaction-control statements: the runner owns the transaction around SQL
and history updates. Migration execution currently certifies SQLite.
