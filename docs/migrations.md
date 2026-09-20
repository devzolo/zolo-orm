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

## Supported changes and explicit boundaries

| Change | Behavior |
| --- | --- |
| New models with foreign keys | Creates parents before children; rollback drops children first. |
| Index addition, removal or replacement | Generates reversible SQL. |
| DROP TABLE or DROP COLUMN | Requires `--allow-destructive` when generating. |
| Required column without a SQL default | Refused. |
| Adding a primary-key or UNIQUE column | Refused. |
| Existing column, foreign key or table UNIQUE constraint changes | Requires an explicit table rebuild; not generated automatically. |
| Cyclic initial foreign-key graph | Refused by automatic generation. |

A rebuild needs a complete, reviewed migration and a snapshot that describes
its result. This is not an instruction to edit an already checksummed migration:
the runner detects edited files. The automatic generator does not yet provide
a general rebuild workflow.

New snapshots use version 2 and retain compatibility with version 1 snapshots
and their original checksums. Inspection includes index keys, composite
uniqueness and foreign-key actions. It refuses schema details it cannot
represent faithfully, including partial/expression/custom-collation indexes,
deferred foreign keys and directional primary/UNIQUE constraints.

Migrations always enable foreign-key enforcement before their transaction and
check for orphaned rows. Scripts must not contain `BEGIN`, `COMMIT`,
`ROLLBACK` or other transaction-control statements: the runner owns the
transaction around SQL and history updates.
