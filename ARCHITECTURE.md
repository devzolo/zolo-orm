# Architecture

This document describes how the package turns a struct into SQL, what the
generated code owns, and where the boundaries with `std::database` and the
compiler are. Read the [README](README.md) first for the user-facing API.

## Code generation

`Model` is a compile-time derive. For each annotated struct it emits:

- the `CREATE TABLE` DDL and the static column metadata,
- a `<Model>Query` struct with the `where_*`, `order_by_*`, `filter`, `select`
  and execution methods,
- `create`, `find`, `changes`, `insert_many`, `delete_all` and the instance
  `insert`/`update`/`delete` methods,
- a row decoder that builds the struct from a driver row,
- one `load_<field>` function per `belongs_to` relation.

The derive works on structured `quote` blocks and `syntax.ident`, so the
generated code keeps the original type references. There is no runtime model
registry and no per-field reflection: everything the generated code needs is
resolved when the model is compiled.

Generated helper names are chosen so that a user field cannot shadow them.
Creation defaults and returned-row decoding run in schema helpers outside the
named-parameter scope, which is why a model may have fields called `db`,
`Statement`, `Result` or `OrmError` without breaking its own generated code.
The `hygiene` example exercises this.

## Query plans

A query is an immutable plan. Each `where_*` or `filter` call creates a new
plan node that links back to the previous one, so extending a query shares its
earlier predicates instead of copying them. Rendering walks the links once, in
insertion order, and hands the predicates, projection, ordering and pagination
to the zolo-sql AST builder.

Generated methods validate column names and value types before anything
reaches the builder. The low-level plan methods are public so that other
libraries can build on them, but they accept raw SQL fragments and give none of
the guarantees of the generated API.

Filter and select lambdas are not executed. The compiler captures the lambda,
checks it against the model's schema metadata, and rewrites it into structured
predicates or a column list plus a typed decoder. Unsupported expressions are
rejected at that point.

## Statements and execution

A `Statement` carries the SQL text and its bound parameters separately. It is
executed through the ordinary `std::database` entry points: the same zolo-sql
dialect compilation and the same driver path that a plain `sql"..."` literal
uses. The package adds no driver or native plugin of its own.

`Statement.compile(dialect)` exists for inspection. Execution does not call it;
`Database` compiles the statement itself and caches the compiled SQL by text
and dialect within fixed limits. Bound values and connections are never cached.

Optional values are bound as SQL `NULL` tokens rather than as nil array
elements, because the VM's binding arrays drop a trailing nil. Identifiers are
validated at compile time and always quoted.

## Creation, patches and decoding

`create` builds a named-parameter signature from the model's fields, with the
authored default expressions attached. Nullable parameters distinguish an
omitted argument from an explicit `nil`, which is how a default can coexist
with storing NULL on purpose. The row returned by `INSERT ... RETURNING` is
decoded into the model.

`ModelChanges` stores typed assignments. Setting a field to `nil` and not
setting it are different states, so a patch can write NULL. An empty patch
issues no SQL.

Decoders use complete struct literals, so a database NULL can never be replaced
by the field's construction default. SQLite booleans are accepted as `bool` or
as the integers 0 and 1; other numeric probes use Zolo's checked conversions.
A decode failure carries the model, the field and the expected type, but not
the rejected value.

## Batches

`insert_many` computes the chunk size at compile time from the column count,
capped at 900 bound parameters per statement. It opens one transaction, issues
one multi-row `INSERT` per chunk, and rolls everything back on the first error.
Each chunk uses bounded SQL and binding buffers; the input list stays owned by
the caller. Native execution dispatch is shared by the Cranelift and LLVM
backends, while the VM has its own bridge and rollback path.

## Relations

`belongs_to` generates an explicit loader instead of a lazy property. The
loader deduplicates the parent keys, fetches the children with `IN` queries of
at most 900 keys, and groups them back in the original parent order. It does
not add foreign-key DDL, does not wrap its reads in a transaction, and never
runs when a field is accessed, which rules out hidden N+1 queries.

## Errors

- `OrmError.Database` wraps the driver's `DbError`; SQL builder limits reached
  during execution or `try_statement()` also surface here.
- `OrmError.FieldDecode` and `OrmError.Decode` report rows that do not match the
  model.
- `OrmError.UnsafeMutation` rejects deletes that are ambiguous: no filter, or a
  filter combined with ordering or pagination.
- `OrmError.Unsupported` reports scalar NULL projections and runtimes without
  database support.

Invalid model declarations are compile errors. Invalid pagination or batch
configuration is treated as a programmer error.

## Compiler and editor contract

The derive publishes `sql_schema`, `sql_column` and `sql_query` metadata in a
generic protocol. The compiler uses it to validate `sql"..."` literals against
the visible tables and columns, to capture query lambdas, and to give the
language server field completion, hover and diagnostics. Any library can emit
the same protocol; nothing in the compiler is tied to a package named `orm`.

The `zolo db` commands consume the same metadata to generate migrations, so
typed queries and the migration history are derived from a single source.
