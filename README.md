# zolo-orm

Typed SQLite models and queries for Zolo, generated from ordinary structs.

Declare a model once, then use typed creation, partial updates, atomic upserts,
query projections and explicit relation loading. The same metadata produces
indexes, foreign keys, typed INNER/LEFT joins and migrations. Field names and
value types are checked by the compiler; query values are bound separately
from SQL. Optional scalar projections preserve NULL positions.

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
  User::changes().set_note("Welcome").apply(db, ana.id)?

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
| Insert or update a unique record | [Atomic upserts](docs/models-and-writes.md#atomic-upserts) | [upsert.zolo](examples/upsert.zolo) |
| Filter, project, page or require a row | [Queries](docs/queries.md) | [required_reads.zolo](examples/required_reads.zolo) |
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
