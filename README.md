# zolo-orm

Typed SQLite models and queries for Zolo, generated from ordinary structs.

Declare a model once, then use typed creation, partial updates, atomic upserts,
query projections and explicit relation loading. The same metadata produces
indexes, foreign keys, typed INNER/LEFT joins and migrations. Field names and
value types are checked by the compiler; query values are bound separately
from SQL. Relations reference types and fields directly; referential actions use enums.
Optional scalar projections preserve NULL positions. Typed composite indexes reference
fields directly; presence-aware reads and numbered or cursor pages keep common
query flows explicit. Named patches express partial updates in one call; composed
query views name multiple sources, and codecs preserve domain types at the database
boundary. Computed selections and aggregates support reports; richer cursors
continue ordered projections and joins. Built-in codecs cover dates, UTC instants,
exact decimal text and binary values, and schema descriptors support composite
keys and advanced indexes.

[Get started](docs/getting-started.md) · [Documentation](docs/README.md) ·
[Runnable examples](examples/README.md) · [Architecture](ARCHITECTURE.md)

## Run the first example

From this repository's root, using a compatible Zolo installation:

```sh
cd examples
zolo install
zolo run basic.zolo --no-cache
```

No database server is needed. The example uses an in-memory SQLite database.

```text
Created: ana@example.com
Active user: Ana
Matching users: 1
```

## First model

This is the complete [basic.zolo](examples/basic.zolo) program:

```rust
use std::database::Database
use orm::{Model, OrmError}

@derive(Model)
@model(table: "users")
struct User {
  @model(primary_key: true, generated: true)
  id: int,
  name: str,
  @model(unique: true)
  email: str,
  active: bool = true,
  note: str?,
}

fn example(db: Database) -> Result<int, OrmError> {
  // Convenient for this in-memory demo. Use migrations for a persistent schema.
  User::create_table(db)?
  let ana = User::create(db, name: "Ana", email: "ana@example.com")?
  User::changes(note: "Welcome").apply(db, ana.id)?

  let users = User::query()
    .filter(|user| user.active && user.name.starts_with("A"))
    .order_by_name()
    .select(|user| (user.id, user.name))
    .all(db)?

  assert(users.len() == 1)
  assert(users[0][0] == ana.id)
  let saved = User::find_or_error(db, ana.id)?
  assert(saved.note == "Welcome")
  print("Created: {ana.email}")
  print("Active user: {users[0][1]}")
  return Result::Ok(users.len())
}

let db = Database.open("sqlite://:memory:").unwrap()
defer db.close()
let count = example(db).unwrap()
print("Matching users: {count}")
```

The application owns the connection. The example propagates errors with `?`
and unwraps only at its outer boundary; see
[error handling](docs/errors-and-compatibility.md#errors) for expected absence
and constraint failures.

## Typed relations

Relationship metadata uses the same symbols as application code:

```rust
@model(belongs_to: Team, references: Team.code, foreign_key: true, on_delete: .Cascade)
team_code: str,
```

Omit `references` to use `Team.id`. Imported aliases work, and the
referenced field must belong to the target type. Keep database names such as
`column: "team_code"` as strings. See [typed declarations and upgrading](docs/relations-and-transactions.md#typed-relationship-declarations)
and the runnable [alias example](examples/typed_relations.zolo).

## Typed indexes and pages

Use field references for composite indexes:

```rust
@model(unique_indexes: #{tenant_email: [Member.tenant_id, Member.email]})
```

The fields must belong to the annotated model. Renames remain visible to the
compiler and editor, and mapped columns are resolved automatically. See the
complete [typed index example](examples/typed_indexes.zolo).

```rust
let page = User::query().where_active(true).page(db, number: 1, size: 20)?
let next = User::query().where_active(true).cursor_page(db, size: 20)?

let note = User::query().where_id(1).select(|user| user.note).first_row(db)?
// nil means no row; a present Row with value == nil means SQL NULL.
```

Numbered pages include totals; cursor pages continue from a typed primary key.
Both add deterministic ordering. See [pagination](docs/queries.md#numbered-pages)
for their contracts and [presence-aware reads](docs/queries.md#projections)
for nullable results.

## Named patches and domain types

```rust
User::changes(name: "Ana Maria", note: nil).apply(db, user.id)?
```

Omitted fields stay unchanged; explicit nil clears an optional field. Existing
setters remain available for conditional patch construction. See
[named patches](docs/models-and-writes.md#named-patches).

Use `@model(codec: EmailCodec) email: Email` to keep a domain type in creation,
patches, filters and projections while storing a scalar. Declare a
`@derive(Query)` view to combine multiple model sources with named results.
See [domain codecs](docs/domain-codecs.md) and
[composed queries](docs/composed-queries.md) for full declarations and limits.

## Reports and ordered continuations

```rust
use orm::Sql

let totals = User::query().group_by(|user| user.active)
  .select(|user| (user.active, Sql::count())).all(db)?
let names = User::query().order_by_name().select(|user| user.name)
let first = names.seek_page(db, size: 20)?
let next = names.seek_page(db, size: 20, after: first.next_cursor)?
```

[Computed selections and aggregates](docs/aggregates.md) retain SQL nullability.
[Advanced cursors](docs/advanced-cursors.md) append key tie-breakers and preserve
mixed ordering directions. Use [built-in codecs](docs/builtin-codecs.md) for
common domains and [advanced schemas](docs/advanced-schemas.md) for composite
primary/foreign keys, index expressions, directions and partial predicates.

## Add it to your project

```toml
[dependencies]
orm = { git = "https://github.com/devzolo/zolo-orm.git", rev = "main" }
```

Run `zolo install`, then import `orm::Model`. Pin a tested commit for an
application release. For local development use
`orm = { path = "../zolo-orm" }`, relative to your manifest.

These APIs require a compatible Zolo compiler, runtime and editor server.
A Git dependency uses the published revision; the checked-out examples use
the local ORM source. See [setup and compatibility](docs/errors-and-compatibility.md#compatibility).

## Choose your next step

| Task | Guide | Example |
| --- | --- | --- |
| Define defaults, mapped columns and indexes | [Models and writes](docs/models-and-writes.md) | [dx.zolo](examples/dx.zolo) |
| Change several fields in one call | [Named patches](docs/models-and-writes.md#named-patches) | [named_patches.zolo](examples/named_patches.zolo) |
| Preserve enum and newtype fields | [Domain codecs](docs/domain-codecs.md) | [domain_codecs.zolo](examples/domain_codecs.zolo) |
| Calculate reports and grouped totals | [Aggregates](docs/aggregates.md) | [aggregates.zolo](examples/aggregates.zolo) |
| Continue ordered projections or joins | [Advanced cursors](docs/advanced-cursors.md) | [advanced_cursors.zolo](examples/advanced_cursors.zolo) |
| Store dates, exact decimals and binary data | [Built-in codecs](docs/builtin-codecs.md) | [builtin_codecs.zolo](examples/builtin_codecs.zolo) |
| Define composite keys and advanced indexes | [Advanced schemas](docs/advanced-schemas.md) | [advanced_schemas.zolo](examples/advanced_schemas.zolo) |
| Combine several named model sources | [Composed queries](docs/composed-queries.md) | [composed_queries.zolo](examples/composed_queries.zolo) |
| Insert or update a unique record | [Atomic upserts](docs/models-and-writes.md#atomic-upserts) | [upsert.zolo](examples/upsert.zolo) |
| Filter, project or require a row | [Queries](docs/queries.md) | [required_reads.zolo](examples/required_reads.zolo) |
| Declare composite indexes using fields | [Typed indexes](docs/models-and-writes.md#indexes-and-conflict-targets) | [typed_indexes.zolo](examples/typed_indexes.zolo) |
| Read numbered or cursor pages | [Pagination](docs/queries.md#numbered-pages) | [pagination.zolo](examples/pagination.zolo) |
| Save related rows and load their children | [Relations and transactions](docs/relations-and-transactions.md) | [relationships.zolo](examples/relationships.zolo) |
| Read nullable scalar values without losing rows | [Projections](docs/queries.md#projections) | [nullable_projections.zolo](examples/nullable_projections.zolo) |
| Join related models in a typed query | [INNER and LEFT joins](docs/queries.md#typed-relation-joins) | [joins.zolo](examples/joins.zolo) |
| Evolve a persistent database | [Migrations](docs/migrations.md) | [Migration project](examples/migration_demo/README.md) |
| Handle duplicate keys and rollback | [Errors](docs/errors-and-compatibility.md#errors) | [error_handling.zolo](examples/error_handling.zolo) |

Upsert creation values and update patches are separate: omitted patch fields
stay unchanged, and an empty patch returns `nil` on conflict. A relation
loader is explicit; `foreign_key: true` additionally enforces integrity.
`create_table` creates missing objects; use migrations to evolve existing data.

SQLite execution is supported on VM, native and LLVM backends.
Other dialect rendering does not imply database-driver support. Review the
[capability table and current boundaries](docs/errors-and-compatibility.md#compatibility)
for NULL projections, more complex joins, codecs and WebAssembly.

## Development

Run from the repository root:

```sh
zolo run scripts/test.zolo
zolo run scripts/test_backends.zolo
```

The first command checks the library, runs the listed VM examples and package
tests, and verifies rejected programs in `tests/`. The second builds and runs
the listed native/LLVM examples, including the introductory walkthroughs.
The separate migration project has its own
[apply/rollback walkthrough](examples/migration_demo/README.md).

For benchmarks, run `zolo run scripts/benchmark.zolo --backend vm` or
`--backend native`. The script compares 2,000 individual inserts with chunked
inserts inside transactions, after a warm-up and five measured runs by default.
It writes samples and medians to `target/benchmarks/<backend>.json`; use them
to compare strategies on your machine.

## License

MIT. See [LICENSE](LICENSE).
