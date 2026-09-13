# zolo-orm

Typed models and SQL queries for Zolo, generated at compile time.

Annotate a struct with `@derive(Model)` and the package generates the table
DDL, a query builder, create/update/delete helpers, batch inserts and relation
loaders for it. Query lambdas are captured by the compiler and translated to
SQL, so a misspelled field or a filter with the wrong type is a compile error,
not a runtime surprise. SQLite is the supported database.

Requires Zolo 0.1.8-alpha or newer. The package is not published to a registry
yet; depend on it straight from the Git repository.

## Installation

```toml
[dependencies]
orm = { git = "https://github.com/devzolo/zolo-orm.git", rev = "main" }
```

Or let the CLI write that line for you:

```sh
zolo add orm --git https://github.com/devzolo/zolo-orm.git --rev main
```

`rev` accepts a branch, a tag or a commit hash; pin a commit for anything
you deploy. Run `zolo install` and import what you need:

```rust
use orm::{Model, OrmError, transaction}
```

The `examples/` directory is a runnable project:

```sh
cd examples
zolo install
zolo run basic.zolo
```

## Quick start

```rust
use std::database::Database
use orm::{Model, OrmError}

@derive(Model)
@model(table: "users")
struct User {
  @model(primary_key: true, generated: true)
  id: int,
  @model(unique: true)
  email: str,
  active: bool = true,
  note: str?,
}

fn example(db: Database) -> Result<int, OrmError> {
  User::create_table(db)?
  let user = User::create(db, email: "ana@example.com")?
  User::changes().set_note("Hello").apply(db, user.id)?
  let users = User::query()
    .filter(|u| u.active && u.email.starts_with("ana"))
    .order_by_email()
    .limit(20)
    .select(|u| (u.id, u.email))
    .all(db)?
  return Result::Ok(users.len())
}

let db = Database.open("sqlite://:memory:").unwrap()
defer db.close()
example(db).unwrap()
```

## Models

A model is an ordinary struct. Fields may be `int`, `float`, `str`, `bool` or
the optional form of any of them. Every model declares exactly one primary key,
and it cannot be optional.

```rust
@derive(Model)
@model(table: "inventory")
pub struct Product {
  @model(primary_key: true, generated: true, column: "product_id")
  pub id: int,
  pub name: str,
  pub available: bool = default_availability(),
}
```

Field options:

| Option | Meaning |
| --- | --- |
| `primary_key: true` | The row key. Required on exactly one field. |
| `generated: true` | Let SQLite allocate the key. Integer keys only. |
| `unique: true` | Adds a UNIQUE constraint to the column. |
| `column: "name"` | Maps the field to a different SQL column. |
| `belongs_to: "Parent"` | Declares a relation. See [Relations](#relations). |

Without `generated: true` the application supplies every key. Generated keys use
SQLite's rowid allocation, so the id of a deleted row may be reused later.

Default expressions run when a row is created and are evaluated in the scope of
the declaring module, so a model can call private helper functions.

`User::create_table(db)` runs the initial `CREATE TABLE`. It does not detect a
table that already exists with an older shape; use [migrations](#migrations)
for that. `User::schema_sql()` returns the DDL as text.

## Creating and updating rows

`create` takes the model's fields as named arguments, applies the declared
defaults, runs `INSERT ... RETURNING` and returns the stored row. Generated keys
are not part of its signature.

```rust
let user = User::create(db, email: "ana@example.com")?
```

For an optional field with a default, omitting the argument applies the default
and passing `nil` stores SQL NULL.

Complete, caller-keyed records use the instance methods:

```rust
user.insert(db)?
user.update(db)?
user.delete(db)?
```

Partial updates go through a patch. Each `set_FIELD` call returns a new patch,
a repeated setter replaces the earlier value, and fields that were never set
are left untouched. `set_note(nil)` writes NULL. Applying an empty patch does
nothing and returns zero. Primary keys have no setter.

```rust
User::changes().set_note(nil).set_active(false).apply(db, user.id)?
```

`User::find(db, key)` returns `Result<User?, OrmError>`. When absence is a bug
in the caller, unwrap it explicitly:

```rust
let user = User::find(db, 1)? ?? panic("missing user")
```

## Queries

`User::query()` returns a generated `UserQuery`. Every method returns a new
query and leaves the original untouched, so a base query can be reused.

### Filters

```rust
User::query().filter(|u| u.active && (u.id >= 10 || u.email.contains("@example.")))
```

Filter lambdas support comparisons on scalar fields, `bool` fields on their
own, `&&`, `||` and `!` with grouping, comparisons of optional fields with
`nil`, and the string methods `starts_with`, `ends_with` and `contains`. String
matching compiles to `LIKE`, so it follows the database's collation rules; the
`%`, `_` and `!` characters in the argument are escaped. Anything else, such as
calling a function inside the lambda, is a compile error. Compute the value
first and compare against it.

The lambda is SQL, not a callback: it runs in the database with SQL semantics,
including three-valued NULL logic.

Generated methods cover the common cases without a lambda:

| Method | SQL |
| --- | --- |
| `.where_email(value)` / `.where_email_not(value)` | `=` / `<>` |
| `.where_id_in([1, 2])` | `IN`; an empty list matches no rows |
| `.where_id_gt(10)` / `.where_id_lt(20)` | `>` / `<`, generated for non-bool fields |
| `.where_note(nil)` / `.where_note_not(nil)` | `IS NULL` / `IS NOT NULL` |

Chained filters combine with `AND`.

### Ordering, pagination and reading

```rust
User::query()
  .where_active(true)
  .order_by_id(descending: true)
  .limit(20)
  .offset(40)
  .all(db)?
```

`all`, `first` and `count` execute the query. `count` counts matches before
pagination is applied. An offset requires a limit, and a limit of zero returns
no rows.

### Projections

`select` takes one field or a tuple of fields; only those columns are fetched
and each value goes through the field's typed decoder.

```rust
let pairs = User::query().select(|u| (u.id, u.email)).all(db)?
```

Select optional fields as part of a tuple. A scalar projection whose value is
NULL returns `OrmError.Unsupported`, because Zolo arrays cannot yet hold a nil
slot; rows are never dropped silently.

### Deleting

```rust
User::query().where_id(1).delete(db)?
User::delete_all(db)?
```

A filtered delete rejects ordering and pagination. Deleting without a filter is
only possible through the explicit `delete_all`.

### Inspecting SQL

`statement()` returns the SQL text and its bound parameters, and
`statement().compile("postgres")` renders it for another dialect. Execution and
`try_statement()` report SQL builder limits through `Result`; `statement()` is
the convenience form that unwraps them. The builder accepts identifiers up to
1,024 bytes, nesting up to depth 64 and up to 16,384 nodes or parameters.

## Batches and transactions

`insert_many` inserts a list of rows in one transaction using multi-row
`INSERT` statements. The chunk size is computed at compile time from the number
of columns and capped at 900 bound parameters per statement. Any failure rolls
back the whole batch, and an empty list opens no transaction.

```rust
User::insert_many(db, users, batch_size: 128)?
```

`insert_many_in(tx, users)` does the same chunking inside a transaction the
caller already owns.

`transaction` runs a closure and returns a single `Result` layer; returning
`Err` rolls back.

```rust
let saved = transaction(db, |tx| User::create(tx, email: "bia@example.com"))?
```

The package never opens or closes the connection it is given. Close it at the
application boundary.

## Relations

Relations are explicit. Nothing is loaded lazily and no property access issues
a query.

```rust
@derive(Model)
struct Post {
  @model(primary_key: true, generated: true)
  id: int,
  @model(belongs_to: "User")
  user_id: int,
  title: str,
}

let users = User::query().all(db)?
for related in Post::load_user_id(db, users)? {
  print("{related.parent.email}: {related.items.len()} posts")
}
```

`load_user_id` returns one entry per parent, in the order the parents were
given, each with `.parent` and `.items`. Parent keys are deduplicated and
fetched with `IN` queries of at most 900 keys. `references: "other_key"` points
the relation at a parent field other than the primary key. Declaring a relation
does not add a foreign-key constraint to the DDL.

## Raw SQL

`User::from_sql(db, sql"SELECT ...")` decodes rows from a hand-written query
into the model. The query must return every model column under its mapped SQL
name.

The derive also publishes the schema to the compiler, so `sql"..."` literals
are checked against the visible tables, columns and parameter types at compile
time. The same metadata drives editor completion and diagnostics.

## Migrations

The model metadata feeds `zolo db`. Each migration is a timestamped directory
with `up.sql`, `down.sql`, a `schema.json` snapshot and a checksum manifest,
all meant to be reviewed and committed.

```sh
zolo db generate initial
zolo db status --url sqlite://app.db
zolo db migrate --url sqlite://app.db
zolo db check --url sqlite://app.db
zolo db rollback --url sqlite://app.db
```

The default directory is `migrations/`; override it with `--migrations <dir>`.
Generation is additive: changed columns and drops must be written by hand, and
a generated drop additionally requires `--allow-destructive`. Apply and rollback
run the SQL and the history update in one transaction. The runner refuses edited
migration files, gaps in the history and unknown migrations. `check` compares
the last applied snapshot with the live catalog without changing anything, and
`baseline` adopts an existing database by verifying its tables and recording a
replayable initial migration.

## Errors

`OrmError` has five variants:

| Variant | When |
| --- | --- |
| `Database(DbError)` | The driver or SQL builder failed. |
| `Decode(str)` | A row could not be turned into the model. |
| `FieldDecode(DecodeError)` | A field had an unexpected type. Carries model, field and expected type, never the value. |
| `UnsafeMutation(str)` | A delete without a filter, or with ordering/pagination. |
| `Unsupported(str)` | A scalar NULL projection, or a runtime without database support. |

Invalid model declarations, unknown fields, wrong filter types, duplicate
columns, oversized identifiers and ambiguous generated method names are all
compile errors. For example, fields named `id` and `id_not` would both generate
`where_id_not`; rename the field and keep the column with `@model(column: "id_not")`.

## Limitations

- SQLite is the only database that executes queries and migrations. Statements
  can be compiled for PostgreSQL and MySQL for inspection, but those drivers do
  not bind parameters yet, and the migration runner rejects their URLs.
- Composite keys, indexes, foreign-key DDL, joins, upserts and computed
  projections are not generated.
- Only `int`, `float`, `str` and `bool` columns are supported. Decimal,
  timestamp and blob codecs are planned.
- Scalar projections cannot return NULL; select a tuple instead.
- No streaming results, async execution, prepared-statement cache or
  connection pool.
- The package does not run on the wasm target, which has no database runtime.

## Development

```sh
zolo run scripts/test.zolo
zolo run scripts/test_backends.zolo
zolo run scripts/benchmark.zolo --backend vm
zolo run scripts/benchmark.zolo --backend native
```

`test.zolo` type-checks the library, runs every example, runs the integration
tests and checks that each program under `tests/` fails to compile with the
expected diagnostic. `test_backends.zolo` builds the examples with the
Cranelift and LLVM backends and executes them.

`benchmark.zolo` inserts 2,000 rows once with individual `INSERT` calls and
once with chunked multi-row statements, both inside a single transaction. It
runs one warm-up process and, by default, five measured processes, then writes
every sample and the medians to `target/benchmarks/<backend>.json`. The numbers
depend on the machine and toolchain; use them to compare the two strategies,
not as a throughput figure.

See [ARCHITECTURE.md](ARCHITECTURE.md) for how the pieces fit together.

## License

MIT. See [LICENSE](LICENSE).
