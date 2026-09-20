# Documentation

Start with the [first model](getting-started.md), then choose the guide for the
operation you need. Every guide links to executable examples.

| I want to… | Read | Run |
| --- | --- | --- |
| Install the package and save my first row | [Getting started](getting-started.md) | [basic.zolo](../examples/basic.zolo) |
| Define defaults, columns and indexes | [Models and writes](models-and-writes.md) | [dx.zolo](../examples/dx.zolo) |
| Insert or update a unique record | [Atomic upserts](models-and-writes.md#atomic-upserts) | [upsert.zolo](../examples/upsert.zolo) |
| Filter, page or select specific columns | [Queries](queries.md) | [expressions.zolo](../examples/expressions.zolo) |
| Load children and save related work | [Relations and transactions](relations-and-transactions.md) | [relationships.zolo](../examples/relationships.zolo) |
| Evolve a persistent database | [Migrations](migrations.md) | [Migration demo](../examples/migration_demo/README.md) |
| Handle absence and constraint failures | [Errors](errors-and-compatibility.md#errors) | [error_handling.zolo](../examples/error_handling.zolo) |
| Check support or solve a setup problem | [Compatibility](errors-and-compatibility.md#compatibility) | [Example catalog](../examples/README.md) |

Guides use the User model from [basic.zolo](../examples/basic.zolo) unless
another model is shown. Short code blocks are excerpts; linked `.zolo` files
include imports, declarations, a connection and an entry point.

## Three distinctions to keep in mind

- **Absent, NULL and unchanged:** omitting a create argument uses its default;
  passing `nil` writes NULL; omitting a patch setter leaves stored data unchanged.
- **Insert and conflict:** upsert input supplies the new row; its explicit patch
  supplies the conflict update. An empty patch returns `nil` on conflict.
- **Loading and integrity:** `belongs_to` enables a loader; `foreign_key: true`
  additionally asks the database to enforce the relationship.

For internals, see [Architecture](../ARCHITECTURE.md). For a sequence of
commands with expected output, see [Examples](../examples/README.md).
