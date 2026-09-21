# Composed queries

[Documentation](README.md) · [Complete example](../examples/composed_queries.zolo) · [Declarations](../examples/composed_models.zolo)

Use `@derive(Query)` to name the result of several related model sources.
Each field represents one source; the field's optionality chooses LEFT or
INNER JOIN.

```rust
use orm::Query

@derive(Query)
pub struct Directory {
  pub person: Person,
  @join(from: Directory.person, key: Person.team_id, references: Team.id)
  pub team: Team?,
  @join(from: Directory.team, key: Team.organization_id, references: Organization.id)
  pub organization: Organization?,
  @join(from: Directory.person, key: Person.manager_id, references: Person.id)
  pub manager: Person?,
}
```

This excerpt uses the models in [composed_models.zolo](../examples/composed_models.zolo).
`Directory::query().all(db)` returns `Result<[Directory], OrmError>`. Access
`row.person`, `row.team`, `row.organization` and `row.manager` directly;
no nested pair unpacking is needed. A missing LEFT match is nil.

The declaration is a reusable read view. It creates no table, migration DDL
or write methods. The individual models retain ownership of their schemas.

## Join edges

The first source must be required. Every later source declares three references:

| Option | Meaning |
| --- | --- |
| `from` | An earlier field of this same view. |
| `key` | A field on that earlier source's model. |
| `references` | A field on the model being joined. |
| `keys` / `reference_keys` | Ordered field-reference lists for a composite join; do not mix with scalar key/references. |

Required sources use INNER JOIN; optional sources use LEFT JOIN. Both keys must
have matching scalar types. Codec keys are rejected. Each source must be a model
with nonoptional primary-key fields. Presence uses a required key field, and
page ordering retains every part of every source key.
Join targets need not be unique: multiple matches produce multiple result rows.

Mapped columns are resolved automatically. Repeated models, including the
`person` and `manager` sources above, receive distinct aliases. References to
a different view, wrong-owner keys and forward references are compile errors.
A view supports one through sixteen sources, without construction defaults.

An INNER source following a LEFT source can remove rows whose earlier source
is absent. Use optional sources along a chain when absent parents should keep
the root row in the results.

## Filtering and selecting

Lambda parameters follow source declaration order. Every declared source gets
one parameter, including unused sources:

```rust
let name = "Zolo"
let rows = Directory::query()
  .filter(|person, team, organization, manager| organization?.name == name)
  .order_by_person_name()
  .select(|person, team, organization, manager| (person.name, team?.name, manager?.name))
  .all(db)?
```

Use optional access for LEFT sources. A matched source with a nullable field is
still a real model; malformed required fields return decoding errors.
Projections preserve the field's logical type, including domain codecs.
Use `first_row` when a nullable projection must distinguish SQL NULL from
no result row.

Order methods include the view field: `order_by_person_name()` and
`order_by_manager_name(descending: true)` are unambiguous even for repeated
models. Codec fields do not generate ordering methods.

## Additional ON conditions

`on` adds conditions to the **last join**. Earlier LEFT sources remain optional
inside its lambda; the last source is required because ON runs before that
join produces an absent result:

```rust
let prefix = "Ana"
let query = Directory::query()
  .on(|person, team, organization, manager| manager.name.starts_with(prefix))
```

This can restrict which manager matches while keeping people without a match.
Putting that condition in `filter` instead may remove those people. Repeated
`on` calls combine conditions on the last join with AND. Extending earlier ON
clauses is not exposed by this API.

Bound ON values precede WHERE values in statement order, regardless of the order
in which the builder methods were called. Builder calls return new queries.

## Reads, pages and imports

The query provides `all`, `first`, `first_row`, `first_or_error`, `count`,
`exists`, `limit`, `offset`, `page`, `statement` and `try_statement`.
A required-read error names the view. Count/exists retain every join and filter,
ignore pagination, and count/probe joined rows without an implicit DISTINCT.

```rust
let page = Directory::query().page(db, number: 1, size: 20)?
let person = page.items[0].person
```

Numbered pages append every source's primary key to break ordering ties.
They use a count and a select; wrap them in a transaction when a shared
snapshot is required. [Advanced cursors](advanced-cursors.md) continue ordered
views and projections through `seek_page`.

A public view can be imported or reexported on its own. The compiler follows its
model metadata, while generated readers and encoders stay on the exported view.
The [example](../examples/composed_queries.zolo) imports `Directory as Summary`
through a facade without importing its underlying models.

For a single declared `belongs_to` edge, the existing
[relation join helpers](queries.md#typed-relation-joins) remain convenient.
[Computed projections and aggregates](aggregates.md) use the same typed sources.
RIGHT/FULL joins, joined mutations and replacing the declared key equality with
an arbitrary ON expression remain outside these APIs; use explicit SQL for those shapes.
