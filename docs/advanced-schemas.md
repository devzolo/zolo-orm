# Advanced schemas

[Documentation](README.md) · [Example](../examples/advanced_schemas.zolo)

Declare an ordered primary key with field references. A composite key generates a named value type:

```zolo
@derive(Model)
@model(primary_key: [Account.tenant_id, Account.id])
struct Account {
  tenant_id: int,
  id: int,
  name: str,
}

let key = AccountKey { tenant_id: 42, id: 7 }
let account = Account::find_or_error(db, key).unwrap()
Account::changes(name: "Updated").apply(db, key).unwrap()
account.delete(db).unwrap()
```

`find`, `find_or_error`, `changes(...).apply`, `update`, `delete` and `upsert_by_key` use every key component. `account.key()` returns `AccountKey`. The order in `primary_key` controls DDL, bindings and pagination tie-breakers. Primary fields cannot appear in a patch. Key fields must be required scalar values; codec domains cannot be keys.

A one-field `primary_key: [Account.id]` keeps the existing scalar key parameter. The field-level `@model(primary_key: true)` form remains supported for one key. Do not combine it with the type-level list. Generated identity is available only for a single integer primary key.

## Composite foreign keys

The fields and references lists are ordered and have equal lengths:

```zolo
@model(
  primary_key: [Membership.tenant_id, Membership.id],
  foreign_keys: #{
    account: #{
      fields: [Membership.tenant_id, Membership.account_id],
      references: [Account.tenant_id, Account.id],
      on_delete: .Cascade,
    },
  },
)
```

Each target list must be one complete primary key, a column marked `unique`, or one declared `unique_indexes` key, in its declared order. One field from a composite PK is not independently unique. Partial and expression indexes cannot establish FK target uniqueness. `advanced_indexes` is not used as an FK uniqueness declaration; use `unique_indexes` for an ordinary column key.

All local fields belong to the declaring model; all target fields belong to the same parent. Storage types must agree. Codec fields are rejected. `SetNull` requires every local field to be optional. `SetDefault` requires a SQL default on each required field. Application defaults do not count as database defaults.

The map name, such as `account`, identifies the descriptor in diagnostics. DDL emits an unnamed SQL constraint. Migrations preserve the grouped fields, order, target and actions; physical SQL constraint names are unsupported.

A composite FK does not generate a scalar `belongs_to` loader. Compose a query view with matching typed lists:

```zolo
@derive(Query)
struct Directory {
  membership: Membership,
  @join(
    from: Directory.membership,
    keys: [Membership.tenant_id, Membership.account_id],
    reference_keys: [Account.tenant_id, Account.id],
  )
  account: Account,
}
```

`Account?` produces a LEFT join. The existing singular `key`/`references` pair remains available; do not mix it with `keys`/`reference_keys`. All source PK columns become pagination tie-breakers.

## Directional, expression and partial indexes

Index descriptors use checked records with contextual enum completion:

```zolo
@model(advanced_indexes: #{
  active_email: #{
    unique: true,
    terms: [
      #{field: Account.tenant_id},
      #{field: Account.email, transform: .Lower, direction: .Desc},
    ],
    predicate: [#{field: Account.active, kind: .IsTrue}],
  },
})
```

Terms default to `transform: .Identity` and `direction: .Asc`. The supported transforms are `Identity`, `Lower`, `Upper`, `Length` and `Abs`. Text transforms require a string field; `Abs` requires a numeric field. Transforms cannot be applied to codec domains.

`predicate` defaults to an empty list. Available predicates are `IsNull`, `IsNotNull`, `IsTrue` and `IsFalse`; multiple entries are joined with SQL `AND`. Boolean predicates require a boolean field. Fields remain typed references, and mapped physical column names are honored. These descriptors do not accept raw SQL.

Ordinary unique column indexes can generate `upsert_by_index_NAME`. A unique advanced index only generates that method when every term is `Identity` and no predicate exists. Partial or expression indexes have no generated upsert method: a column-only conflict target would be incorrect. `upsert_by_key` remains available.

The descriptor evaluator checks unknown, duplicate and missing required record fields. Nested `TypeRef` and `FieldRef` values resolve in the consumer scope, while descriptor defaults resolve in the producer scope. Rename updates logical field references and leaves index labels and physical strings intact.

## Schema inspection and migrations

Snapshots store composite PK order, grouped FK columns and references, index direction, supported expression transforms and partial predicates. SQLite inspection uses PK ordinals and FK sequence numbers, and checks index expressions against catalog SQL. Drift reports changes in these definitions instead of reducing them to plain column names.

Automatic rebuilds reject primary-key identity changes, including reordered components. Unsupported expressions, collations, named constraints and FK timing options fail explicitly. Binary columns retain BLOB storage; automatic scalar-to-binary or binary-to-scalar conversion requires an explicit reviewed migration. Existing scalar snapshots remain readable.

See [advanced_schemas.zolo](../examples/advanced_schemas.zolo) for a complete model, query and CRUD example. This source batch has not been compiled or run; executable validation remains pending.
