# Errors and compatibility

[Documentation](README.md) · [Runnable error handling](../examples/error_handling.zolo)

## Errors

`OrmError` has six variants:

| Variant | When |
| --- | --- |
| `Database(DbError)` | The driver or SQL builder failed. |
| `NotFound(str)` | A required read found no record. Carries the model name, never the key or filter values. |
| `Decode(str)` | A row could not be turned into the model. |
| `FieldDecode(DecodeError)` | A field had an unexpected type. Carries model, field and expected type, never the value. |
| `UnsafeMutation(str)` | A delete without a filter, or with ordering/pagination. |
| `Unsupported(str)` | A runtime without database support. |

For SQLite constraint failures, match `OrmError.Database(cause)` and use
`cause.is(DbErrorKind::UniqueViolation)` or `match cause.kind()`. Import
`DbErrorKind` from `std::database`; no driver-message parsing is needed.
The available constraint kinds are `UniqueViolation`, `ForeignKeyViolation`,
`NotNullViolation`, `CheckViolation` and the general `ConstraintViolation`.
Other SQL execution failures remain `QueryFailed`.

[error_handling.zolo](../examples/error_handling.zolo) shows a registration flow
that turns a duplicate email into an `EmailTaken` application outcome. It
inserts related work inside `orm::transaction`, verifies that a duplicate
rolls it back, and propagates other database failures. The transaction returns
one `Result` layer, so the caller handles the application outcome directly.

Handle expected absence without a panic:

```rust
fn describe_user(db: Database, requested_id: int) -> Result<str, OrmError> {
  return match User::find_or_error(db, requested_id) {
    .Ok(user) => Result::Ok(user.email),
    .Err(error) => match error {
      .NotFound(model) => Result::Ok("{model} was not found"),
      _ => Result::Err(error),
    },
  }
}
```

Invalid model declarations, unknown fields, wrong filter types, duplicate
columns, oversized identifiers and ambiguous generated method names are all
compile errors. For example, fields named `id` and `id_not` would both generate
`where_id_not`; rename the field and keep the column with `@model(column: "id_not")`.

## Compatibility

| Surface | Supported scope |
| --- | --- |
| Zolo VM + SQLite | ORM queries, writes, transactions and migrations. |
| Native / Cranelift + SQLite | Runnable package examples and the same generated API. |
| LLVM + SQLite | Runnable package examples and the same generated API. |
| PostgreSQL / MySQL SQL rendering | Statement inspection; bound ORM execution and migrations are not supported. |
| Browser Playground | The underlying in-memory SQLite bridge, SQL builder and reflection are tested. This does not establish standalone ORM package/project support in the Playground. |
| wasm-aot | Language/reflection tests exist; full database-backed ORM execution is not supported. |

The compiler must support structured query capture, `exists`, index/upsert SQL,
generated sibling imports and consumer-visible derive reflection. Use a
matching CLI, runtime and language server. A newer package checkout cannot
add those capabilities to an older compiler.

The package is consumed through Git or a local path, not a package registry.
The checked-out examples use the local path. A Git dependency on `main` only
contains what has been published to that branch.

## Current boundaries

- One nonoptional primary key per model. Composite primary/foreign keys,
  chained/arbitrary joins and computed projections are not supported. One
  typed INNER or LEFT belongs_to join per query is supported.
- Columns support `int`, `float`, `str`, `bool` and their optional forms.
  Decimal, timestamp and blob codecs are not provided.
- Composite UNIQUE indexes and conflict targets are supported; expression,
  partial and per-column directional ORM indexes are not generated.
- Scalar projections preserve NULL slots. `first` on a nullable scalar returns
  nil for both a NULL value and no row; project a required key with it to distinguish them.
- No streaming API, async execution, prepared-statement cache or connection pool.
- Reviewed SQLite rebuilds require `--allow-rebuild` and preserve primary-key
  identity and compatible data. Lossy casts and unsupported schema objects are
  rejected; discarding columns still requires `--allow-destructive`. Version 3
  migration policy is checksummed. See the
  [migration change table](migrations.md#supported-changes-and-explicit-boundaries).

## Troubleshooting

| Symptom | Check |
| --- | --- |
| `orm` cannot be resolved | Run `zolo install` from the project containing its dependency; check the path or Git revision and lock. |
| A generated input or method is missing | Confirm both the ORM revision and compatible compiler; rerun with `--no-cache`. |
| CLI succeeds but editor reports missing generated types | Refresh the matching language server/extension, then reload the editor window. |
| A filter expression is rejected | Use supported comparisons/string operations and compute other values outside the lambda. |
| `NotFound` when no record is expected | Use `find`/`first` for optional results; reserve `*_or_error` for a required row. |
| Upsert returns `nil` | An empty patch plus a matching key means DO NOTHING; use a nonempty patch if an update is intended. |
| A UNIQUE error survives upsert | It may come from another key or the patch; only the named conflict target is handled. |
| A foreign-key error occurs | Create the parent first and check the key/action. Loading metadata alone is not enforcement. |
| An existing table did not change | `create_table` does not migrate it; generate and apply reviewed migrations. |
| A migration rejects the live catalog | Inspect drift or unsupported schema details; baseline does not erase mismatches. |
