# Architecture

This document describes how the package turns a struct into SQL, what the
generated code owns, and where the boundaries with `std::database` and the
compiler are. Start with the [README](README.md) and [user guides](docs/README.md) for the public API.

## Responsibilities

| Layer | Owns |
| --- | --- |
| Model declarations | Field types, defaults, keys, indexes and relation intent. |
| zolo-orm | Typed helpers, immutable query plans, patches and row decoding. |
| Compiler / language server | Derives, generated imports, query capture, reflection and editor analysis. |
| zolo-sql | Structured SQL construction, parsing and dialect rendering. |
| std::database / zolodb | Connections, bound execution, transactions and structured driver failures. |
| zolo db | Schema snapshots, migration planning, history and drift checks. |

The application owns connection lifetime and chooses transaction boundaries.
Queries execute when an execution method is called; accessing a field never
triggers a hidden query. Model metadata feeds query checking and migrations,
so there is no separately maintained mapping schema.

For observable contracts, start with [basic](examples/basic.zolo),
[upsert](examples/upsert.zolo) and [relationships](examples/relationships.zolo).
The larger schema/foreign-key examples retain edge-case regression coverage.

## Code generation

`Model` is a compile-time derive. For each annotated struct it emits:

- the `CREATE TABLE`/`CREATE INDEX` DDL and the static column metadata,
- a `<Model>Query` struct with the `where_*`, `order_by_*`, `filter`, `select`
  and execution methods,
- `create`, `find`, `find_or_error`, `changes`, `insert_many`, `delete_all` and the instance
  `insert`/`update`/`delete` methods,
- a `<Model>Insert` input type and typed upsert/statement methods for each
  declared primary/unique conflict target,
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
libraries can build on them. Their runtime column names and structured
predicates still pass through the SQL builder, but lack the field-name and
value-type checks of the generated API.

Filter and select lambdas are not executed. The compiler captures the lambda,
checks it against the model's schema metadata, and rewrites it into structured
predicates or a column list plus a typed decoder. Unsupported expressions are
rejected at that point.

`first_or_error` delegates to `first`, so it preserves the query's ordering and
offset, its zero-limit fast path, and its database/decoding failures. Only an
absent row becomes `NotFound` with the source model name. `find_or_error`
builds a primary-key predicate and uses the same required-read path.

`count` and `exists` operate on matching records independently of ordering and
pagination. The `exists` statement kind renders `SELECT 1 AS "orm_exists" FROM
... WHERE ... LIMIT 1`. It ignores the model projection and tests whether the
driver returned a row; it never invokes the model decoder. This keeps the work
bounded to one match and allows an existence check even when an unrelated
model field cannot be decoded.

## Statements and execution

A `Statement` carries the SQL text and its bound parameters separately. It is
executed through the ordinary `std::database` entry points: the same zolo-sql
dialect compilation and the same driver path that a plain `sql"..."` literal
uses. The package adds no driver or native plugin of its own.

`Statement.compile(dialect)` exists for inspection. Execution does not call it;
`Database` compiles the statement itself and caches the compiled SQL by text
and dialect within fixed limits. Bound values and connections are never cached.

Optional writes use SQL `NULL` tokens without consuming a bound parameter.
Identifiers are validated at compile time and always quoted. Projection arrays
keep presence separately from values, so SQL NULL occupies a real element
through the VM, native/LLVM runtimes and plugin bridges.

## Creation, patches and decoding

`create` builds a named-parameter signature from the model's fields, with the
authored default expressions attached. Nullable parameters distinguish an
omitted argument from an explicit `nil`, which is how a default can coexist
with storing NULL on purpose. The row returned by `INSERT ... RETURNING` is
decoded into the model.

`ModelChanges` stores typed assignments. Setting a field to `nil` and not
setting it are different states, so a patch can write NULL. Applying an empty
patch directly with `apply` issues no SQL.

Decoders use complete struct literals, so a database NULL can never be replaced
by the field's construction default. SQLite booleans are accepted as `bool` or
as the integers 0 and 1; other numeric probes use Zolo's checked conversions.
A decode failure carries the model, the field and the expected type, but not
the rejected value.

## Indexes, foreign keys and upsert

Type-level `indexes` and `unique_indexes` are maps from a logical name to an
ordered field list. Field-level `index` is shorthand for one such index. The
derive validates names, duplicate fields and field existence before converting
logical fields to quoted SQL columns. Schema statements are deterministic and
shared by `schema_sql`, `schema_statements`, `create_table` and the compiler's
`@sql_schema(ddl)` metadata. Migrations consume the same DDL.

Foreign keys are opt-in on `belongs_to`. Generic `TypeRef` and `FieldRef`
attribute values resolve against the consumer's visible type catalog and retain
canonical declaration identity, imported binding syntax and reflected metadata.
The ORM checks that an explicit field belongs to the same target declaration;
aliases of that declaration remain equivalent. `ForeignKeyAction` enum values
encode referential actions. This preserves the producer's executable lexical
scope. The ORM reads the
referenced model's raw decorators to resolve its canonical table, column and
unique-key status. It validates the field types and referential actions before
emitting REFERENCES. A loader alone retains its earlier behavior and produces
no foreign-key DDL.

Each upsert target is generated from a primary key, UNIQUE field or named
unique index. `<Model>Insert` supplies insert values/defaults; generated keys
are optional inputs. `<Model>Changes` supplies only explicit update assignments.
The shared statement helper appends patch bindings after insert bindings and
emits one parameterized ON CONFLICT statement with RETURNING. It never copies
insert defaults into the UPDATE clause.

An empty patch emits DO NOTHING and a conflict returns nil. No read/modify/write
race, follow-up SELECT or synthetic UPDATE is needed. Constraints unrelated to
the named target remain database errors, and the returned row always passes
through the normal model decoder. Statement-inspection helpers use the exact
same construction path as execution. Generated method and helper-name
collisions are checked before publishing the API.

## Batches

`insert_many` computes the chunk size at compile time from the column count,
capped at 900 bound parameters per statement. It opens one transaction, issues
one multi-row `INSERT` per chunk, and rolls everything back on the first error.
Each chunk uses bounded SQL and binding buffers; the input list stays owned by
the caller. Native execution dispatch is shared by the Cranelift and LLVM
backends, while the VM has its own bridge and rollback path.

## Relations

`belongs_to` generates an explicit loader instead of a lazy property. The
loader deduplicates non-null parent keys, fetches the children with `IN` queries
of at most 900 keys, and groups them back in the original parent order. A parent
with a nil key receives an empty group. The loader does not add foreign-key DDL
unless `foreign_key: true` is requested, does not wrap its reads in a transaction,
and never runs when a field is accessed, which rules out hidden N+1 queries.

## Errors

- `OrmError.Database` wraps the driver's `DbError`; SQL builder limits reached
  during execution or `try_statement()` also surface here. SQLite constraint
  kinds such as `UniqueViolation` survive this wrapper and transactional
  rollback; callers branch on `DbError.kind()` or `.is()` without parsing text.
- `OrmError.NotFound` identifies the model when a required read has no row; it
  never includes the requested key or filter values.
- `OrmError.FieldDecode` and `OrmError.Decode` report rows that do not match the
  model.
- `OrmError.UnsafeMutation` rejects deletes that are ambiguous: no filter, or a
  filter combined with ordering or pagination.
- `OrmError.Unsupported` reports runtimes without database support.

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

## Relation query sources

Generated INNER/LEFT queries opt into the generic `sql_query(sources: [...])`
protocol. Each source names a typed schema, SQL alias and exported reader
methods. Nullable sources additionally name a required presence field. LEFT
projection decoders select this field under an internal alias to distinguish
an absent row from a malformed matched row. Readers accept the selected column
alias while preserving the original model/field error context.

A relation view uses `sql_schema(query_only: true)`. Its columns participate
in query validation and editor tooling, but it contributes no physical table
or DDL to migration discovery. The compiler does not depend on this package's
name. Imported projections call reader methods on the real exported model,
never the synthetic schema names used only during analysis.

`QueryPlan` stores relation predicates and WHERE filters independently.
The shared SQL AST builder quotes source aliases and column identifiers
separately, renders ON before WHERE, and emits placeholders in that order.
Count and existence retain the joined relation while discarding ordering and
pagination. A joined mutation is rejected rather than inferred.
