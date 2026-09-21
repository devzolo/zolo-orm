# Cursor pagination with several ordering fields

[Documentation](README.md) · [Example](../examples/advanced_cursors.zolo)

Use `seek_page` when a query has several ordering fields, joins, or a projection:

```rust
let query = Event::query()
  .order_by_created_at(descending: true)
  .order_by_title()
let first = query.seek_page(db, size: 20)?
let next = query.seek_page(db, size: 20, after: first.next_cursor)?
```

The existing `cursor_page` remains the scalar primary-key shorthand. Both APIs
return `CursorPage` with `items`, `has_next` and `next_cursor`. A final page has no
continuation token. Changing the requested page size is allowed.

## Deterministic continuation

`seek_page` follows the typed `order_by_FIELD` methods already present on the
query. It appends missing primary-key fields to break ties. Composed views and
relation joins include every source key, and composite keys retain all parts.
Each ordering field can have its own ascending or descending direction.

Ascending fields put SQL NULL first; descending fields put NULL last. Predicates
preserve that rule across page boundaries, including absent LEFT-joined models.
Additional tie-breaking key fields use ascending order. At most 24 ordering
fields, including tie-breakers, are accepted to bound SQL expression depth.

The query selects internal cursor columns even when the projection omits them:

```rust
let names = Event::query().order_by_created_at(descending: true)
  .select(|event| event.title)
let page = names.seek_page(db, size: 20)?
```

Those hidden fields do not change the selected type, and their generated
aliases avoid authored column names. A lookahead row determines `has_next`
without decoding that extra row.

## Token and query lifetime

The continuation is a `QueryCursor` value. Pass it back unchanged; its contents
are an implementation detail. It is bound to the SQL statement shape and a
snapshot of the query's bound values, including binary values. A changed filter,
projection or ordering returns `OrmError.InvalidPagination`.

A cursor is an in-process value, not a serialized or signed token for an HTTP
client. It does not hold a database snapshot. Concurrent inserts/deletes and
changes to ordering fields can change which rows appear; use an appropriate
transaction when a consistent snapshot is required.

`seek_page` rejects an existing limit/offset, invalid size, missing key metadata,
conflicting duplicate orderings and grouped/aggregate queries. Domain codecs do
not automatically define SQL ordering; order by ordinary scalar columns.
