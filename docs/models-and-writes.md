# Models and writes

[Documentation](README.md) · [Runnable upsert](../examples/upsert.zolo)

The snippets extend the User model in [basic.zolo](../examples/basic.zolo).

## Models

A model is an ordinary struct. Fields may be `int`, `float`, `str`, `bool` or
the optional form of any of them. Every model declares exactly one primary key,
and it cannot be optional.

```rust
fn default_availability() -> bool { true }

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
| `unique: true` | Adds a UNIQUE constraint and a typed `upsert_by_FIELD` method. |
| `index: true` | Adds a named index on this field. |
| `column: "name"` | Maps the field to a different SQL column. |
| `belongs_to: Parent` | Declares a relation to a visible model type, including an imported alias. |
| `references: Parent.field` | A field reference on the same parent type; defaults to `id`. |
| `foreign_key: true` | Adds an enforced foreign key for the declared relation. |
| `on_delete` / `on_update` | `.NoAction`, `.Restrict`, `.Cascade`, `.SetNull` or `.SetDefault`; require `foreign_key: true`. |
| `default_sql: "literal"` | A database default used by schema tooling; construction defaults remain application-side. |

Without `generated: true` the application supplies every key. Generated keys use
SQLite's rowid allocation, so the id of a deleted row may be reused later.

Default expressions run when a row is created and are evaluated in the scope of
the declaring module, so a model can call private helper functions.

`User::create_table(db)` runs `CREATE TABLE IF NOT EXISTS` and the declared
`CREATE INDEX IF NOT EXISTS` statements. It does not alter an existing table
or index with an older shape; use [migrations](migrations.md) for that.
`User::schema_sql()` returns all DDL separated by semicolons, and
`User::schema_statements()` returns the individual statements. Creation uses
the supplied connection and can run inside an existing transaction.

## Indexes and conflict targets

A field can opt into `@model(index: true)`. For composite indexes, declare
named lists of logical fields on the model:

```rust
@model(
  table: "members",
  indexes: #{by_name: ["name", "tenant_id"]},
  unique_indexes: #{tenant_email: ["tenant_id", "email"]},
)
```

The derive converts those fields through their `column` mappings and quotes
every SQL identifier. Index names are `zolo_<table-length>_<table>_<name>_idx` or
`zolo_<table-length>_<table>_<name>_uidx`; the length prefix keeps table/name
boundaries unambiguous. A field's shorthand uses its logical field name.
Empty lists, unknown/repeated fields, invalid identifiers and duplicate index
names are compile errors. Named maps are emitted in key order.

Primary keys, `unique: true` fields and `unique_indexes` declare the only
available typed upsert targets. A normal index does not make a field unique.
See [schema_writes.zolo](../examples/schema_writes.zolo) for a complete model with
mapped columns, ordinary indexes and a composite unique key.

## Creating and updating rows

`create` takes the model's fields as named arguments, applies the declared
defaults, runs `INSERT ... RETURNING` and returns the stored row. Generated keys
are not part of its signature.

```rust
let user = User::create(db, name: "Ana", email: "ana@example.com")?
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

`User::find(db, key)` returns `Result<User?, OrmError>`. Use
`find_or_error` when the next operation requires the row; it returns
`Result<User, OrmError>` and reports absence as `OrmError.NotFound("User")`.
Both methods keep database and decoding errors intact.

```rust
let user = User::find_or_error(db, 1)?
User::changes().set_note("Reviewed").apply(db, user.id)?
```

## Defaults, NULL and unchanged fields

| Intent | Create / insert input | Patch |
| --- | --- | --- |
| Use the construction default | Omit the field/argument. | Defaults never apply to a patch. |
| Store SQL NULL in an optional field | Pass `nil`. | Call `set_FIELD(nil)`. |
| Preserve the stored value | Not an insertion operation. | Do not call that field's setter. |

An optional field with no explicit default starts as NULL. `default_sql`
declares a database default for schema tooling and referential actions; it does
not replace the Zolo default expression used by `create`. If both paths need
the same default, declare both deliberately.

The generated `<Model>Insert` type follows the same omission/NULL rules as
`create`. Application defaults are evaluated where the model is declared,
including when an input is imported from another module.

## Atomic upserts

Every model generates an insert-input type, normally `<Model>Insert`, with the
model's field types and defaults. A generated integer key becomes optional:
omitting it or passing `nil` asks SQLite to allocate the key, while an explicit
value supports a primary-key upsert. The derive chooses a suffixed input type
name if a model field would shadow it, just as it does for schema helpers.

```rust
let input = UserInsert { name: "Ana", email: "ana@example.com" }
let patch = User::changes().set_note("Seen today")
let stored = User::upsert_by_email(db, input, patch)?
```

`upsert_by_FIELD` exists only for primary-key and UNIQUE fields. A named
composite unique index such as `tenant_email` generates
`upsert_by_index_tenant_email`. The insert input and patch are specific to the
model, and unknown targets or a different model's input/patch fail to compile.

The operation is one `INSERT ... ON CONFLICT (target) ... RETURNING` statement.
The insert path uses the input and its defaults. The conflict path applies only
the explicit patch: omitted setters leave existing data intact, and a setter
with `nil` writes SQL NULL. Primary keys have no patch setter. Constraint
failures outside the chosen target, and failures caused by the patch itself,
remain typed database errors.

The return type is `Result<Model?, OrmError>`: inserts and updates return the
stored row. An empty patch means `DO NOTHING`; when the chosen key conflicts,
no UPDATE or follow-up SELECT runs and the result is `nil`. An empty patch can
still insert a new row. A nullable UNIQUE key follows SQLite's semantics:
NULL does not conflict with another NULL, so repeated nil keys insert separate
rows.

Inspect the exact SQL and ordered bindings before execution:

```rust
let statement = User::upsert_by_email_statement(input, patch)
print(statement.text)
```

See [schema_writes.zolo](../examples/schema_writes.zolo) for default preservation,
explicit NULL patches, composite/nullable targets and empty-patch behavior.

## Models in another module

Export the model and its fields from the declaring module. Import generated
input types alongside it:

```rust
use models::{Product, ProductInsert}
```

Generated inputs participate in imports and aliases.
[dx_imports.zolo](../examples/dx_imports.zolo) uses
[models.zolo](../examples/models.zolo), including a private default helper and
a primary-key upsert.

Next: [queries](queries.md), [relations](relations-and-transactions.md), or
[persistent schema migrations](migrations.md).
