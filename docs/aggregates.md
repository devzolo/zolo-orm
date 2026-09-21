# Computed projections and aggregates

[Documentation](README.md) · [Example](../examples/aggregates.zolo)

Use ordinary scalar arithmetic in a captured selection. Values from application
code stay bound parameters:

```rust
let multiplier = 2
let amounts = Sale::query().select(|sale| sale.units * multiplier + 1).all(db)?
```

Supported operators are `+`, `-`, `*` and `/` on numeric columns/values.
Division produces a nullable real value: a zero divisor produces SQL NULL.
Nullable operands propagate NULL. Integer overflow is not silently converted
into a truncated application value: execution or decoding can return an error.
Remainder is not captured because Zolo and SQLite use different negative-value
semantics. Codec domains require explicit application-side conversions; their
stored scalars do not imply arithmetic on the logical type.

## Aggregate expressions

Import `Sql` from `orm`. It declares functions that are captured into SQL inside
query lambdas:

| Expression | Result |
| --- | --- |
| `Sql::count()` | Number of rows, including rows with NULL fields. |
| `Sql::count_value(value)` | Number of non-NULL values. |
| `Sql::sum(number)` | Numeric sum, nullable when there are no non-NULL inputs. |
| `Sql::avg(number)` | Nullable real average. |
| `Sql::min(value)` / `Sql::max(value)` | Nullable minimum/maximum of compatible scalar values. |

```rust
let totals = Sale::query().select(|sale|
  (Sql::count(), Sql::sum(sale.units), Sql::avg(sale.price))
).first_or_error(db)?
```

An ungrouped aggregate returns one result row even for an empty input:
`count` is zero and `sum`/`avg`/`min`/`max` are NULL. Aggregate calls may appear
inside scalar calculations, but cannot contain another aggregate.
They are SQL expression markers, not ordinary functions for in-memory lists.

## Groups and HAVING

`group_by` accepts a column or tuple of scalar columns. `filter` applies before
grouping; `having` applies to groups:

```rust
let minimum = 2
let report = Sale::query()
  .filter(|sale| sale.units > 0)
  .group_by(|sale| sale.category)
  .having(|sale| Sql::count() >= minimum)
  .select(|sale| (sale.category, Sql::sum(sale.units)))
let page = report.page(db, size: 20)?
```

Every nonaggregate column in a grouped selection or HAVING condition must be
a grouping key. This prevents SQLite from choosing an arbitrary row's value.
Use `try_statement` to inspect plan-validation failures as a Result.

Numbered pages count result groups, and append grouping keys to make their
ordering deterministic. Ungrouped aggregate pages describe the single aggregate
row. `seek_page` rejects grouped and aggregate selections.

The same capture protocol applies to relation joins and composed views.
Lambda parameters retain source order and LEFT-join nullability. Bound values
are collected in SQL order: SELECT, ON, WHERE, then HAVING. Grouping keys do not
contain bound values.
