# Queries

[Documentation](README.md) · [Runnable query expressions](../examples/expressions.zolo)

The snippets use the User model in [basic.zolo](../examples/basic.zolo).

`User::query()` returns a generated `UserQuery`. Every method returns a new
query and leaves the original untouched, so a base query can be reused.

## Filters

```rust
User::query().filter(|u| u.active && (u.id >= 10 || u.email.contains("@example.")))
```

Filter lambdas support comparisons on scalar fields, `bool` fields on their
own, `&&`, `||` and `!` with grouping, comparisons of optional fields with
`nil`, and the string methods `starts_with`, `ends_with` and `contains`. String
matching compiles to `LIKE`, so it follows the database's collation rules; the
`%`, `_` and `!` characters in the argument are escaped. Anything else, such as
calling a function inside the lambda, is a compile error. Compute the value
first and compare against it.

The lambda is SQL, not a callback: it runs in the database with SQL semantics,
including three-valued NULL logic.

Generated methods cover the common cases without a lambda:

| Method | SQL |
| --- | --- |
| `.where_email(value)` / `.where_email_not(value)` | `=` / `<>` |
| `.where_id_in([1, 2])` | `IN`; an empty list matches no rows |
| `.where_id_gt(10)` / `.where_id_lt(20)` | `>` / `<`, generated for non-bool scalar fields without codecs |
| `.where_note(nil)` / `.where_note_not(nil)` | `IS NULL` / `IS NOT NULL` |

Chained filters combine with `AND`. [Codec fields](domain-codecs.md) accept
domain values for equality, inequality and generated IN filters; range, string
matching, truthiness and implicit domain ordering are not supported.

## Ordering and reading

```rust
User::query()
  .where_active(true)
  .order_by_id(descending: true)
  .limit(20)
  .offset(40)
  .all(db)?
```

Choose the read that matches the caller's intent:

| Method | Result |
| --- | --- |
| `.all(db)` | `Result<[User], OrmError>`; every row in the requested page. |
| `.first(db)` | `Result<User?, OrmError>`; the first row, or `nil`. |
| `.first_row(db)` | `Result<Row<User>?, OrmError>`; a wrapper with `value`, or `nil` when absent. |
| `.first_or_error(db)` | `Result<User, OrmError>`; the first row, or `NotFound("User")`. |
| `.count(db)` | `Result<int, OrmError>`; matching records before pagination. |
| `.exists(db)` | `Result<bool, OrmError>`; whether any record matches, before pagination. |

`first_or_error` honors ordering, offset and a zero limit, just like `first`.
It does not require the query to match exactly one row. An offset requires a
limit when fetching rows, and a limit of zero returns no rows.

`exists` selects a constant with `LIMIT 1` and performs no model decoding. Like
`count`, it ignores ordering, limit and offset: even `.limit(0).exists(db)`
returns `true` when records match the filters. It still propagates SQL builder
and database errors.

```rust
let active = User::query().where_active(true)
let has_active_users = active.exists(db)?
let first_active = active.order_by_id().first_or_error(db)?
```

See [required_reads.zolo](../examples/required_reads.zolo) for absence handling,
reused queries, pagination and malformed database rows.

## Projections

`select` takes one field or a tuple of fields; only those columns are fetched
and each value goes through the field's typed decoder.

```rust
let pairs = User::query().select(|u| (u.id, u.email)).all(db)?
```

Optional scalar projections return arrays such as `[str?]`. SQL NULL is a
present `nil` element: leading, middle and trailing NULL values retain their
positions, including when every value is NULL. An empty result is `[]`.

`first(db)` returns `T?`, so a NULL scalar and no row both produce `nil`.
Use `first_row` to distinguish them without selecting an extra column:

```rust
let result = User::query().where_id(1).select(|user| user.note).first_row(db)?
if let row = result {
  // The row exists. row.value has type str? and may be nil (SQL NULL).
  print(row.value ?? "No note")
}
```

`first_row` returns `Result<Row<T>?, OrmError>`. Only absence makes the wrapper
`nil`; the value inside keeps the projection's exact type. `first_or_error` is
also available on projections: a present NULL scalar returns `Ok(nil)`, while
absence returns `NotFound` with the source model name. Both honor ordering,
offset and zero limits and propagate decoder failures.

See [nullable_projections.zolo](../examples/nullable_projections.zolo) and
[pagination.zolo](../examples/pagination.zolo) for present NULL values,
absence and LEFT joins.

## Numbered pages

```rust
let users = User::query().where_active(true).order_by_name()
let page = users.page(db, number: 2, size: 20)?
let emails = users.select(|user| user.email).page(db, number: 1, size: 10)?
```

`page` is available on model queries, projections and INNER/LEFT joins. It
returns `Result<Page<T>, OrmError>`:

| Field | Meaning |
| --- | --- |
| `items: [T]` | Decoded rows, preserving NULL projection slots. |
| `number: int` / `size: int` | Requested page number and size; defaults are 1 and 20. |
| `total: int` | Matching rows before pagination. |
| `total_pages: int` | Rounded-up page count, or 0 for an empty result. |
| `next_page: int?` | The next number when it does not exceed `total_pages`; otherwise nil. |
| `previous_page: int?` | `number - 1` when `number > 1`; otherwise nil, including for empty results. |

The primary key is appended to the requested ordering to break ties. Without
an explicit order, pages follow ascending primary keys. Joined pages append
every source's key, and count joined rows, including multiple matches per
left row. Projections retain this ordering without adding columns to their
returned values. The original query remains reusable.

A request beyond the last page returns empty `items` and retains the total.
Page numbers and sizes must be positive and their offset must fit in `int`.
Combining `page` with an existing `limit` or a nonzero `offset` returns
`InvalidPagination`, including `limit(0)`.

Each page performs a count and a select. It starts no transaction; use
`orm::transaction` around the call when both statements must observe the same
SQLite snapshot. Separate page requests do not freeze changing data.

## Cursor pages

Use cursor pagination when a primary-key continuation is enough and totals
are unnecessary:

```rust
let users = User::query().where_active(true)
let first = users.cursor_page(db, size: 20)?
if first.has_next {
  let second = users.cursor_page(db, size: 20, after: first.next_cursor)?
}
```

The generated method is available for model queries whose primary key is
`int` or `str`. Its result is `CursorPage<Model, Key>`, with `items`,
`has_next: bool` and `next_cursor: Key?`. The cursor is the last emitted key
only when another page exists; otherwise it is `nil`. A nil `after` starts
from the beginning. A non-nil cursor is an exclusive, bound SQL value and
does not have to identify a currently existing row.

The default is ascending key order. Pass `descending: true` on every request
to walk backward. Keep filters and direction the same across requests.
The method uses one query and requests one extra row to detect continuation;
it decodes only the rows being returned.

A positive size must leave room for that extra row. Existing ordering,
limits, nonzero offsets or joins return `InvalidPagination`. Cursor methods
are not generated for projections, joined queries or float/bool primary keys.
The cursor is a typed key, not an encoded token or a database snapshot.

See [pagination.zolo](../examples/pagination.zolo) for both styles, string
keys, stable joins, invalid requests and query reuse.

## Typed relation joins

A `belongs_to` field generates one INNER and one LEFT join method when its
target is a model with a nonoptional primary key. The method uses the logical
field name. For `@model(belongs_to: Department) department_id: int?`:

```rust
let rows = Worker::query()
  .left_join_department_id()
  .filter(|worker, department| worker.id >= minimum && department?.name != nil)
  .select(|worker, department| (worker.name, department?.name))
  .all(db)?
```

The two lambda parameters identify the two sources independently. Mapped SQL
names are resolved from model metadata and each source receives its own SQL
alias, including self joins. INNER exposes a required right schema; LEFT
exposes an optional right schema, so use `department?.name`. A required access
such as `department.name` in a LEFT filter or projection is a compile error.

Without `select`, `all` returns `[Joined<Worker, Department>]` for INNER and
`[Joined<Worker, Department?>]` for LEFT, wrapped in `Result`. Each item has
`left` and `right`. LEFT identifies an absent row by the right model's
nonoptional primary key. A matched row with a nullable field remains a real
model, and malformed required fields still return a decoding error.

Add conditions to the relationship's ON clause with `on`:

```rust
let prefix = "Eng"
let workers = Worker::query()
  .left_join_department_id()
  .on(|worker, department| department.name.starts_with(prefix))
  .order_by_left_id()
  .all(db)?
```

An ON predicate runs before LEFT produces absent right rows, so both parameters
are required there. A WHERE predicate added with `filter` runs after that step.
Filtering right columns in WHERE can remove unmatched rows; filtering them in
ON preserves the left rows. Both support compatible column-to-column comparisons
and captured scalar values. Parameters are bound in statement order: ON first,
then WHERE, preserving the order within each clause.

Use `order_by_left_FIELD` and `order_by_right_FIELD` to choose a source.
Ordering calls append criteria. Filters, ordering and pagination applied before
the join remain part of the resulting query; pagination is applied to joined
rows, without introducing a hidden subquery. `count` counts joined rows and
`exists` probes that same relation, both ignoring pagination and ordering.
No implicit distinct count is added.

Joined queries also expose `first_row`, `first_or_error` and numbered `page`.
A LEFT row with no right match remains a present row with `right == nil`.
Required-read errors name the left model, and projected nullable fields follow
the presence contract described above.

These helpers expose one `belongs_to` edge. For several sources, relation chains
or repeated models, declare a [composed query view](composed-queries.md) with
`@derive(Query)`. It gives each source a name and produces that view as its result.
[Computed projections and aggregates](aggregates.md) use the same source lambdas.
Joined mutations and RIGHT/FULL joins still require explicit SQL. See [joins.zolo](../examples/joins.zolo) for the single-edge API,
imports, reexports, self joins, bound ON conditions and decoding failures.

## Deleting

```rust
User::query().where_id(1).delete(db)?
User::delete_all(db)?
```

A filtered delete rejects ordering and pagination. Deleting without a filter is
only possible through the explicit `delete_all`.

## Inspecting SQL

`statement()` returns the SQL text and its bound parameters, and
`statement().compile("postgres")` renders it for another dialect. Execution and
`try_statement()` report SQL builder limits through `Result`; `statement()` is
the convenience form that unwraps them. The builder accepts identifiers up to
1,024 bytes, nesting up to depth 64 and up to 16,384 nodes or parameters.

For count or existence SQL, use the public plan and choose its statement kind:

```rust
let statement = User::query().where_active(true).plan.try_statement("exists")?
print(statement.text) // SELECT 1 AS "orm_exists" FROM ... WHERE ... LIMIT 1
```

`plan.exists(db)` executes the same existence plan. Bound values stay separate
from SQL. For joined queries ON bindings precede WHERE bindings; each clause preserves filter order.

## Raw SQL

`User::from_sql(db, sql"SELECT ...")` decodes rows from a hand-written query
into the model. The query must return every model column under its mapped SQL
name.

The derive also publishes the schema to the compiler, so `sql"..."` literals
are checked against the visible tables, columns and parameter types at compile
time. The same metadata drives editor completion and diagnostics. Codec columns
use their physical storage type for raw parameters; raw SQL does not encode
domain values automatically, while `from_sql` still decodes them.

## Capture values before filtering

The lambda describes SQL. Compute application values outside it, then capture
those values:

```rust
let prefix = "Ana"
let active = User::query().where_active(true)
let page = active.filter(|user| user.name.starts_with(prefix)).order_by_id().limit(20)
let rows = page.all(db)?
let total = active.count(db)?
```

Each builder call returns a new query. Here `active` stays reusable and
`total` counts all active users, not the page. Use deterministic ordering when
paging. [Computed projections and aggregates](aggregates.md) extend selections
with SQL arithmetic, grouping and HAVING. [Advanced cursors](advanced-cursors.md)
continue typed orderings across model, joined and projected queries.

Scalar and tuple projections preserve NULL values and row count. Optional
fields retain their optional types, including fields from the right side of a
LEFT join; assigning `[str?]` to `[str]` is a compile error.

Next: [absence and database errors](errors-and-compatibility.md#errors).
