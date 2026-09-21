# Documentation

Start with the [first model](getting-started.md), then choose the guide for the
operation you need. Every guide links to executable examples.

| I want to… | Read | Run |
| --- | --- | --- |
| Install the package and save my first row | [Getting started](getting-started.md) | [basic.zolo](../examples/basic.zolo) |
| Define defaults, columns and indexes | [Models and writes](models-and-writes.md) | [dx.zolo](../examples/dx.zolo) |
| Update fields with named arguments | [Named patches](models-and-writes.md#named-patches) | [named_patches.zolo](../examples/named_patches.zolo) |
| Store domain types through codecs | [Domain codecs](domain-codecs.md) | [domain_codecs.zolo](../examples/domain_codecs.zolo) |
| Use dates, exact decimals and binary storage | [Built-in codecs](builtin-codecs.md) | [builtin_codecs.zolo](../examples/builtin_codecs.zolo) |
| Define composite keys and advanced indexes | [Advanced schemas](advanced-schemas.md) | [advanced_schemas.zolo](../examples/advanced_schemas.zolo) |
| Read several named model sources | [Composed queries](composed-queries.md) | [composed_queries.zolo](../examples/composed_queries.zolo) |
| Insert or update a unique record | [Atomic upserts](models-and-writes.md#atomic-upserts) | [upsert.zolo](../examples/upsert.zolo) |
| Calculate totals and group results | [Computed selections and aggregates](aggregates.md) | [aggregates.zolo](../examples/aggregates.zolo) |
| Continue ordered projections and joins | [Advanced cursors](advanced-cursors.md) | [advanced_cursors.zolo](../examples/advanced_cursors.zolo) |
| Filter or select specific columns | [Queries](queries.md) | [expressions.zolo](../examples/expressions.zolo) |
| Declare typed composite indexes | [Indexes](models-and-writes.md#indexes-and-conflict-targets) | [typed_indexes.zolo](../examples/typed_indexes.zolo) |
| Distinguish NULL from absence and paginate | [Reads and pages](queries.md#projections) | [pagination.zolo](../examples/pagination.zolo) |
| Load children and save related work | [Relations and transactions](relations-and-transactions.md) | [relationships.zolo](../examples/relationships.zolo) |
| Join related models in a typed query | [INNER and LEFT joins](queries.md#typed-relation-joins) | [joins.zolo](../examples/joins.zolo) |
| Evolve a persistent database | [Migrations](migrations.md) | [Migration demo](../examples/migration_demo/README.md) |
| Handle absence and constraint failures | [Errors](errors-and-compatibility.md#errors) | [error_handling.zolo](../examples/error_handling.zolo) |
| Check support or solve a setup problem | [Compatibility](errors-and-compatibility.md#compatibility) | [Example catalog](../examples/README.md) |

Guides use the User model from [basic.zolo](../examples/basic.zolo) unless
another model is shown. Short code blocks are excerpts; linked `.zolo` files
include imports, declarations, a connection and an entry point.

## Three distinctions to keep in mind

- **Absent, NULL and unchanged:** omitting a create argument uses its default;
  passing `nil` writes NULL; omitting a patch argument or setter leaves stored data unchanged.
  On reads, `first_row` wraps present values, including NULL, separately from absence.
- **Insert and conflict:** upsert input supplies the new row; its explicit patch
  supplies the conflict update. An empty patch returns `nil` on conflict.
- **Loading and integrity:** `belongs_to` enables a loader; `foreign_key: true`
  additionally asks the database to enforce the relationship.

For internals, see [Architecture](../ARCHITECTURE.md). For a sequence of
commands with expected output, see [Examples](../examples/README.md).
