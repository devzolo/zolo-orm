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
