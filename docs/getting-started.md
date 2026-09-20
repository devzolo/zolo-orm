# Getting started

[Documentation](README.md) · [Next: models and writes](models-and-writes.md)

## Try the checked-out examples

From the repository root:

```sh
cd examples
zolo install
zolo run basic.zolo --no-cache
```

This opens an in-memory SQLite database and closes it when the program exits.
It needs no database server, environment variable or existing table.

Expected program output:

```text
Created: ana@example.com
Active user: Ana
Matching users: 1
```

The first run may also print a package synchronization message. The complete
[basic.zolo](../examples/basic.zolo) program is shown in the
[root README](../README.md#first-model).

## Add the ORM to your application

Create `main.zolo` using the complete
[first-model program](../README.md#first-model), and add `zolo.toml` alongside it:

```toml
[project]
name = "my-app"
version = "0.1.0"
entry = "main.zolo"

[dependencies]
orm = { git = "https://github.com/devzolo/zolo-orm.git", rev = "main" }
```

Then run:

```sh
zolo install
zolo run main.zolo
```

Pin `rev` to a tested commit when shipping an application. A Git dependency
uses that revision, not uncommitted edits in another checkout. For local
co-development, replace the dependency with the path to your ORM checkout:

```toml
[dependencies]
orm = { path = "../zolo-orm" }
```

Paths are relative to the application's manifest. Run `zolo install` after
changing the dependency, and keep its `zolo.lock` with the application.

Use a compatible compiler and matching editor server. The package relies on
generated types, SQL capture and contextual derive reflection; see
[compatibility and setup](errors-and-compatibility.md#compatibility).

## What the first model generates

`@derive(Model)` generates `User::create`, `User::query`, `User::find_or_error`,
a typed `User::changes()` patch and other helpers from the struct's fields.
A generated integer key is omitted when calling `create`. `active = true`
is a construction default, and the optional `note` starts as NULL.

The query's lambda becomes SQL. Only the selected `id` and `name` columns are
read, and the tuple elements keep their field types. A misspelled field or a
wrong filter value is diagnosed before execution.

Functions return `Result<..., OrmError>` and propagate failures with `?`.
The small examples unwrap at their outer boundary so an unexpected failure
stops the program. Applications should
[handle expected failures](errors-and-compatibility.md#errors).

## Move to a persistent database

Use a file URL such as `sqlite://app.db` to keep data between runs.
The application owns the connection and closes it, usually with `defer db.close()`.

`create_table` is convenient for isolated examples and tests. It does not
upgrade an existing schema. For application data, apply migrations before
starting the program. The [migration demo](../examples/migration_demo/README.md)
shows this workflow without calling `create_table`.

Continue with [upserts](../examples/upsert.zolo),
[relationships](../examples/relationships.zolo), or the
[complete example catalog](../examples/README.md).
