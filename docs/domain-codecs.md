# Domain codecs

[Documentation](README.md) · [Complete example](../examples/domain_codecs.zolo) · [Model declarations](../examples/domain_models.zolo)

Keep domain types in application code while storing ordinary SQL scalars.
A field opts into a codec through a real type reference:

```rust
use orm::Model

enum State { Active, Disabled }

struct StateCodec {}
impl StateCodec {
  pub fn encode(value: State) -> str {
    return match value { .Active => "active", .Disabled => "disabled" }
  }

  pub fn decode(value: str) -> Result<State, str> {
    return match value {
      "active" => Result::Ok(State::Active),
      "disabled" => Result::Ok(State::Disabled),
      _ => Result::Err("unrecognized stored state"),
    }
  }
}

@derive(Model)
struct Account {
  @model(primary_key: true)
  id: int,
  @model(codec: StateCodec)
  state: State = .Active,
  @model(codec: StateCodec)
  previous_state: State?,
}
```

The model, creation arguments, insert input, patch setters and projections all
keep `State` or `State?`. Passing a plain string to a typed state parameter is
a compile error. The codec may remain private in the model's declaring module;
consumers only need the public model and the domain types they use.

Across module boundaries, use concrete type aliases or spell nominal generic
instances directly. Private aliases with their own type parameters are not yet
supported by the SQL interface snapshot. Inferred proxy returns also require
generic arguments to be recoverable; phantom arguments with no representation
in the inferred type remain unresolved instead of becoming dynamic.


## Storage and conversions

`encode` accepts the nonoptional domain value and returns its storage scalar.
`decode` accepts that scalar and returns `Result<Domain, Error>`. It may
reject malformed stored data. Generated methods check these types.

Domain fields must be scalars or named types (such as structs, enums or newtypes),
optionally nullable. Wrap a collection in a newtype before giving it a codec;
bare lists, maps, unions and dynamic values do not have the nominal contract
needed by named patches.


| `storage` | Zolo storage type | SQL column type |
| --- | --- | --- |
| `.Text` (default) | `str` | `TEXT` |
| `.Integer` | `int` | `BIGINT` |
| `.Real` | `float` | `DOUBLE PRECISION` |
| `.Boolean` | `bool` | `BOOLEAN` |
| `.Binary` | `bytes` | `BLOB` |

For example, the [shared model](../examples/domain_models.zolo) stores
`Email(str)` with a text codec and `Credits(int)` with
`@model(codec: CreditsCodec, storage: .Integer)`. `storage` requires `codec`.

Optional fields bypass both functions for `nil`: explicit nil writes SQL NULL
and a stored NULL decodes to nil. Construction defaults only apply during
construction. A read never replaces a stored NULL with a default.

All writes use the encoder: `create`, instance `insert`/`update`, `insert_many`,
named patches, setters and upserts. Full reads and selected columns use the
decoder. A rejected value becomes `OrmError.FieldDecode` with the model,
field and expected logical type; stored data and the codec's error payload
are not included.

## Queries and indexes

```rust
let active = State::Active
let accounts = Account::query().filter(|account| account.state == active)
let rows = accounts.select(|account| account.state).all(db)?
Account::changes(state: State::Disabled, previous_state: nil).apply(db, 1)?
```

Captured equality and inequality encode values before binding them. The
generated `where_FIELD`, `where_FIELD_not` and `where_FIELD_in` methods also
accept domain values. NULL checks keep their usual behavior.

Two codec columns may be compared when their logical type, codec declaration
and storage type match. Aliases keep declaration identity; unrelated same-name
types or codecs do not become compatible.

A codec does not promise that storage ordering matches domain ordering.
Generated range/order methods are therefore omitted for these fields, and
captured range, string-pattern and boolean-truthiness operations are rejected.
Compare or transform domain values in application code when needed.

Ordinary and UNIQUE indexes may include codec fields; their equality operates
on encoded values. Choose a stable, deterministic encoding suitable for that
contract. Primary keys, generated keys and relation keys cannot use codecs.

## Schema and raw SQL

DDL and migrations see physical storage types. Changing only an encoder's
representation does not change the column type and cannot automatically migrate
existing values; plan a data migration for such a change.

Raw `sql"..."` parameters use physical storage types. They do not invoke domain
encoders automatically. `Model::from_sql` still decodes returned model columns
through their codecs.

[Built-in codecs](builtin-codecs.md) provide validated dates and UTC timestamps,
exact decimal text and binary blobs. Application codecs can use any storage type
above. Binary values use the database bytes API; arrays and strings are not
implicitly treated as binary data. `Sql::count_value` can count presence on a
codec field, but other aggregates do not infer logical ordering or arithmetic.
