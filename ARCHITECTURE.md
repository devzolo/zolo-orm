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

`first_row` checks the returned row count before wrapping a decoded value in
`Row<T>`. A nullable scalar therefore remains a present row even when its value
is nil. Model `first_or_error` delegates to `first`; projection and joined
required reads use `first_row`. Only absence becomes `NotFound` with the source
model name. These paths preserve ordering, offset, zero limits and decoder
failures. `find_or_error` uses the model's required-read path.

Every plan copy preserves its source model name and pagination keys. Numbered
pages validate positive bounds and checked offset arithmetic, then append key
ordering on a fresh plan. Joined queries supply every source's qualified primary key,
which make ordering deterministic when one left row has several matches.
Projections keep these keys as plan metadata without changing their result
shape. A count and a select produce `Page<T>` without an implicit transaction.

Integer and string primary keys also generate `cursor_page`. It owns primary-key
ordering, adds an exclusive bound and requests one lookahead row. Only emitted
rows are decoded; the lookahead determines continuation without creating a
decoder failure for an item outside the page. `CursorPage<T, K>` keeps the
key's type and exposes no total. Existing order/limit/nonzero offset is rejected
instead of silently replaced.

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

`Model::changes(field: value)` builds that same patch through generated typed
setters. Each argument is a union of its field type with a distinct `Unchanged`
marker, whose default represents omission. Optional nil remains a value to
write, and construction defaults are never evaluated. Positive nominal type
guards narrow domain types before setter calls; aliases retain their declaration
identity. The helper constructing the empty plan lives outside the named
argument scope.


Decoders use complete struct literals, so a database NULL can never be replaced
by the field's construction default. SQLite booleans are accepted as `bool` or
as the integers 0 and 1; other numeric probes use Zolo's checked conversions.
A decode failure carries the model, the field and the expected type, but not
the rejected value.

## Indexes, foreign keys and upsert

Type-level `indexes` and `unique_indexes` are maps from a logical name to an
ordered `FieldRef` list. Field-level `index` is shorthand for one such index.
The derive compares each reference's owner identity against `TypeInfo.identity`,
then validates names and duplicate fields before converting logical fields to
quoted SQL columns. Aliases preserve identity and unrelated same-name
declarations remain distinct. The attribute schema recursively materializes
references inside the maps and arrays, in both `opts` and authored decorators. Schema statements are deterministic and
shared by `schema_sql`, `schema_statements`, `create_table` and the compiler's
`@sql_schema(ddl)` metadata. Migrations consume the same DDL.

Foreign keys are opt-in on `belongs_to`. Generic `TypeRef` and `FieldRef`
attribute values resolve against the consumer's visible type catalog and retain
canonical declaration identity, imported binding syntax and reflected metadata.
The ORM checks that an explicit field belongs to the same target declaration;
aliases of that declaration remain equivalent. `ForeignKeyAction` enum values
encode referential actions. This preserves the producer's executable lexical
scope. The ORM reads the
referenced model's decorators to resolve its canonical table, column and
unique-key status, including a one-field named UNIQUE index. Reflected
decorators resolve in the model's defining scope; references encountered there
carry shallow metadata so self references and relation cycles terminate. It validates the field types and referential actions before
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
- `OrmError.InvalidPagination` reports invalid numbered/cursor page bounds,
  overflow or incompatible query state before database execution.

Invalid model declarations are compile errors. Numbered and cursor page errors
are recoverable. The lower-level negative `limit`/`offset` and invalid batch
configuration remain programmer errors.

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

Query-only schemas contribute no physical table or DDL to migration discovery.
A source may instead name a model through `source`; its canonical published
schema provides the fields, including logical domain types. The outer query's
`source` can identify its result view as an import anchor without claiming
ownership of any table.

Imported projections call reader methods on an exported runtime model or view.
Generated proxy methods forward to the original model in its defining scope,
so private aliases and codecs are never transplanted into the consumer.
Their inferred return signatures carry logical values and error types through
local analysis and imported interfaces. Source metadata includes a reader prefix
and an encoder prefix to keep repeated model sources independent.

`QueryPlan` stores relation predicates and WHERE filters independently.
The shared SQL AST builder quotes source aliases and column identifiers
separately, renders ON before WHERE, and emits placeholders in that order.
Count and existence retain the joined relation while discarding ordering and
pagination. A joined mutation is rejected rather than inferred.

## Composed query views

The separate `Query` derive builds a read-only view with one through sixteen
model sources. The first source is required; each subsequent source declares a
prior view field and two model field references. Canonical owner identity
rejects wrong-source references before SQL emission. Required sources use INNER
joins and optional sources LEFT joins; codec join keys are rejected.

Internal aliases use source ordinals, and projected row aliases use source and
field ordinals. Full reads reconstruct each model's raw row and call its own
decoder. Optional source presence is detected from its nonoptional primary key.
The declared view gives every decoded model a stable result field, including
repeated models in self joins.

Filter/select lambdas use declaration order. ON lambdas require only the final
source: earlier LEFT sources remain optional. The shared plan applies added ON
predicates to the last join and binds them before WHERE. Numbered pages retain
all source keys to order joined rows deterministically.

## Logical types and storage codecs

Codec fields publish their logical type plus physical `storage`, an `encoder`
method name and canonical `codec` identity in `sql_column` metadata.
SQL literals and migration schemas use storage types. Captured queries use the
logical type for argument checking and decoder results, inserting a call to the
exported encoder before binding values. Nullable encoder wrappers bypass nil.

All generated write paths use the same typed encoder wrapper; generated readers
decode a checked scalar and translate codec failures to contextual
`FieldDecode` errors. Codecs can stay private to the model's module. Inferred
proxy signatures and canonical enum/newtype import identities preserve the
public field contract across aliases and facades.

Equality and IN encode domain values. Column comparisons also require matching
logical nominal identity, codec identity and storage. Range/string/truthiness
capture and generated ordering/range methods are unavailable for codec fields.
Primary/relation keys remain ordinary scalars. These rules avoid assuming
an ordering or key representation that the codec contract does not promise.

## Advanced query and cursor protocol

`QueryPlan` retains projection bindings, grouping columns, HAVING predicates and
whether a projection was explicitly selected. Every immutable query operation
preserves these fields. Computed selections lower to the SQL builder's typed
expression tree. The library binds SELECT values before ON/WHERE/HAVING values.
Count/exists wrap grouped/computed selections when necessary so group cardinality
and projection placeholders retain their meaning. Generated public model/view
methods with the `__orm_scalar_` prefix decode computed scalar results without
colliding with ordinary `read_FIELD` readers.

The `Sql` type opts aggregate markers into capture with `sql_aggregates`.
The compiler remains independent of the ORM package name, and scalar reader
owners survive imports/reexports. Division promotes real arithmetic and handles
zero through NULL; the SQL transpiler retains decimal notation for integral real
literals. Group validation rejects free columns outside grouping keys.

`seek_page` reuses `Projection<T>` decoding for models, joins and result views.
`cursor.zolo` canonicalizes column identities, appends all source key parts,
selects hidden order values under collision-free aliases, and builds a null-aware
lexicographic continuation predicate. It snapshots scope bindings, including
copied binary bytes, and compares the statement shape on resume. The token is an
application value, not an external signed token or database snapshot.

## Domains and schema descriptors

`codecs.zolo` supplies validated Date/Timestamp/ExactDecimal domains and codecs;
binary data uses std::database::Blob and physical bytes. Nullable codec fields
bypass encode/decode for SQL NULL. Binary bridges distinguish blobs from text and
arrays and preserve empty data and embedded zero bytes.

Model primary-key lists retain declaration order. Composite keys generate a
ModelKey used by CRUD and patches, and all key parts become page tie-breakers.
Foreign-key descriptors preserve ordered local/reference pairs. Advanced indexes
use typed column references with explicit transforms, directions and NULL/boolean
predicates. Migrations must retain those semantics when reading and comparing
the SQLite catalog, rather than flattening expressions to column names.

This batch is source development only: compiler/LSP/database/migration/ORM
regression sources are written, while compilation, execution, runtime/editor
refresh and publication remain pending.
