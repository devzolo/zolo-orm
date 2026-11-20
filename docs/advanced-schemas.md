# Advanced schemas

[Documentation](README.md) · [Example](../examples/advanced_schemas.zolo)

Declare an ordered primary key with field references. A composite key generates a named value type:

```zolo
@key([.tenant_id, .id])
struct Account deriving Model {
  tenant_id: int,
  id: int,
  name: str,
}

let key = AccountKey { tenant_id: 42, id: 7 }
let account = Account::find_or_error(db, key).unwrap()
Account::changes(name: "Updated").apply(db, key).unwrap()
account.delete(db).unwrap()
```

`find`, `find_or_error`, `changes(...).apply`, `update`, `delete` and `upsert_by_key` use every key component. `account.key()` returns `AccountKey`. The order in `@key([...])` controls DDL, bindings and pagination tie-breakers. Primary fields cannot appear in a patch. Key fields must be required scalar values; codec domains cannot be keys.

A one-field `@key([.id])` keeps the existing scalar key parameter. The field-level `@key` form remains supported for one key. Do not combine it with the list on the model. Generated identity (`@key(generated: true)`) is available only for a single integer primary key. A key that names a field of another model is refused by the compiler where it is written.

## Composite foreign keys

The fields and references lists are ordered and have equal lengths:

```zolo
@key([.tenant_id, .id])
@foreign_key(
  "account",
  fields: [.tenant_id, .account_id],
  references: [Account.tenant_id, Account.id],
  on_delete: .Cascade,
)
```

Each target list must be one complete primary key, a column marked `@unique`, or one unique `@index` over fields, in its declared order. One field from a composite PK is not independently unique. Partial and expression indexes cannot establish FK target uniqueness. An index written with `terms` or `predicate` is not used as an FK uniqueness declaration; use a unique `@index` over fields for an ordinary column key.

All local fields belong to the declaring model; all target fields belong to the same parent. Storage types must agree. Codec fields are rejected. `SetNull` requires every local field to be optional. `SetDefault` requires a SQL default on each required field. Application defaults do not count as database defaults.

The first argument, such as `"account"`, names the constraint and identifies it in diagnostics. The `fields` list must name fields of the declaring model. DDL emits an unnamed SQL constraint. Migrations preserve the grouped fields, order, target and actions; physical SQL constraint names are unsupported.

A composite FK does not generate a scalar `belongs_to` loader. Compose a query view with matching typed lists:

```zolo
struct Directory deriving Query {
  membership: Membership,
  @join(
    from: .membership,
    keys: [Membership.tenant_id, Membership.account_id],
    references: [Account.tenant_id, Account.id],
  )
  account: Account,
}
```

`Account?` produces a LEFT join. `from` names an earlier source with a leading dot, and `keys` and `references` are lists of equal length, even for a single pair. All source PK columns become pagination tie-breakers.

## Directional, expression and partial indexes

Index descriptors use checked records with contextual enum completion:

```zolo
@index(
  "active_email",
  unique: true,
  terms: [
    .{ field: .tenant_id },
    .{ field: .email, transform: .Lower, direction: .Desc },
  ],
  predicate: [.{ field: .active, kind: .IsTrue }],
)
struct Account deriving Model { ... }
```

Terms default to `transform: .Identity` and `direction: .Asc`. The supported transforms are `Identity`, `Lower`, `Upper`, `Length` and `Abs`. Text transforms require a string field; `Abs` requires a numeric field. Transforms cannot be applied to codec domains.

`predicate` defaults to an empty list. Available predicates are `IsNull`, `IsNotNull`, `IsTrue` and `IsFalse`; multiple entries are joined with SQL `AND`. Boolean predicates require a boolean field. Fields remain typed references, and mapped physical column names are honored. These descriptors do not accept raw SQL.

Ordinary unique column indexes can generate `upsert_by_index_NAME`. A unique advanced index only generates that method when every term is `Identity` and no predicate exists. Partial or expression indexes have no generated upsert method: a column-only conflict target would be incorrect. `upsert_by_key` remains available.

The descriptor evaluator checks unknown, duplicate and missing required record fields. Nested `TypeRef` and `FieldRef` values resolve in the consumer scope, while descriptor defaults resolve in the producer scope. Rename updates logical field references and leaves index labels and physical strings intact.

## Schema inspection and migrations

Snapshots store composite PK order, grouped FK columns and references, index direction, supported expression transforms and partial predicates. SQLite inspection uses PK ordinals and FK sequence numbers, and checks index expressions against catalog SQL. Drift reports changes in these definitions instead of reducing them to plain column names.

Automatic rebuilds reject primary-key identity changes, including reordered components. Unsupported expressions, collations, named constraints and FK timing options fail explicitly. Binary columns retain BLOB storage; automatic scalar-to-binary or binary-to-scalar conversion requires an explicit reviewed migration. Existing scalar snapshots remain readable.

See [advanced_schemas.zolo](../examples/advanced_schemas.zolo) for a complete model, query and CRUD example. This source batch has not been compiled or run; executable validation remains pending.
