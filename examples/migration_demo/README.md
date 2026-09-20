# Migration demo

A separate project that creates its schema through `zolo db`. Its
[models](models.zolo) define a note and an index; [main.zolo](main.zolo) only
reads and writes the migrated database.

From the ORM repository root:

```sh
cd examples/migration_demo
zolo install
zolo db generate initial
zolo db status --url sqlite://tutorial.db
```

Review the generated `migrations/<timestamp>_initial/` files, then run:

```sh
zolo db migrate --url sqlite://tutorial.db
zolo db check --url sqlite://tutorial.db
zolo run main.zolo --no-cache
```

Expected program output:

```text
First note: Created after migration
Notes: 1
```

A fixed primary key makes repeated runs keep one row. The application opens
`sqlite://tutorial.db` directly; keep that URL in the commands above.
Its data is independent of all in-memory examples.

To exercise rollback on this disposable database:

```sh
zolo db rollback --url sqlite://tutorial.db
zolo db migrate --url sqlite://tutorial.db
zolo db check --url sqlite://tutorial.db
zolo run main.zolo --no-cache
```

Rollback drops the table and its data; the last run inserts the demo row again.
On subsequent visits, use `status` and `migrate`; generating an unchanged schema
reports that it is already up to date.

See the [migration guide](../../docs/migrations.md) for configuration, baselines,
drift checks and unsupported schema changes, or return to the
[example catalog](../README.md).

## Preserve data through a rebuild

After the initial migration and application run above, use the alternate
[models_rebuilt.zolo](models_rebuilt.zolo). It adds title uniqueness and a SQL
default for the body. `--schema` selects this file without replacing the
original model:

```sh
zolo run inspect.zolo --no-cache
zolo db generate tighten_notes --schema models_rebuilt.zolo --allow-rebuild
```

Review the new migration's copy SQL and its inverse, then run:

```sh
zolo db migrate --url sqlite://tutorial.db
zolo db check --schema models_rebuilt.zolo --url sqlite://tutorial.db
zolo run inspect.zolo --no-cache
zolo db rollback --url sqlite://tutorial.db
zolo run inspect.zolo --no-cache
```

Each inspection prints `migration data preserved: ok`. Unlike the application
entry point, [inspect.zolo](inspect.zolo) performs no inserts or updates, so it
cannot hide missing rows. Rollback returns to the initial schema and leaves
the rebuild pending; applying it again exercises the same data-preserving
transition. Use the alternate `--schema` for checks after reapplying it.

Required values, duplicates or a lossy type conversion cause the rebuild to
fail atomically. See the [migration guide](../../docs/migrations.md) for the
catalog restrictions and the difference between rebuild and destructive flags.
