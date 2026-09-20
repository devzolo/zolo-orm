# Relations and transactions

[Documentation](README.md) · [Runnable relationship workflow](../examples/relationships.zolo)

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
the relation at a parent field other than the primary key. A nullable parent
key produces an empty child group when its value is nil.

A loading relation alone does not add a constraint. Opt into enforcement:

```rust
@model(
  belongs_to: "User",
  references: "id",
  foreign_key: true,
  on_delete: "cascade",
)
user_id: int,
```

The referenced type must be visible to the model, including through an imported
alias. `references` names its logical field; the derive resolves the actual
parent table and column, including overrides. The child and parent scalar
types must agree; relation keys are `int` or `str`, including optional forms.
The referenced field must be a primary key or a
single-field UNIQUE key, including a declared single-field unique index.

Both actions default to `NO ACTION`. `set_null` requires an optional child
field. `set_default` requires `default_sql` on required fields; an application
default does not count. The resulting value must satisfy the field and
foreign-key constraints. SQLite connections enforce foreign
keys by default, and violations remain `DbErrorKind::ForeignKeyViolation`.
[foreign_keys.zolo](../examples/foreign_keys.zolo) exercises mapped imported
models, cascades, SET NULL, restrictive defaults and nullable parent keys.

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
let saved = transaction(db, |tx| User::create(tx, name: "Bia", email: "bia@example.com"))?
```

The package never opens or closes the connection it is given. Close it at the
application boundary.

## Save a parent and its children together

The [relationships example](../examples/relationships.zolo) declares Author
and Article, adds an indexed foreign key, then passes one transaction
connection to every write:

```rust
let author = transaction(db, |tx| publish(tx))?
```

`publish` returns `Result<Author, OrmError>`; any returned error rolls back its
writes. Use `tx` inside the operation, including `insert_many_in` when batching.
`insert_many` owns a transaction itself; its `_in` variant participates in one
you already own.

Create parent tables before child tables when calling `create_table` manually.
Migrations order an acyclic model graph for you. Reading `related.items`
accesses loaded data and issues no SQL.

For mapped parent keys, aliases, SET NULL, SET DEFAULT and restrictive actions,
continue with [foreign_keys.zolo](../examples/foreign_keys.zolo).
For a recoverable failure that rolls back related work, see
[error_handling.zolo](../examples/error_handling.zolo).
