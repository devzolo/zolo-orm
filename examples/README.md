# Examples

[Package README](../README.md) · [Guides](../docs/README.md)

## Start here

From the repository root:

```sh
cd examples
zolo install
zolo run basic.zolo --no-cache
zolo run upsert.zolo --no-cache
zolo run relationships.zolo --no-cache
```

These three walkthroughs each open a fresh in-memory SQLite database. They
are independent, close their connections and can be repeated. Unexpected
results fail an assertion rather than printing a misleading success message.

### 1. First model

[basic.zolo](basic.zolo): generated key, construction default, create, partial
update, required read and a typed projection.

```text
Created: ana@example.com
Active user: Ana
Matching users: 1
```

### 2. Insert or update

[upsert.zolo](upsert.zolo): one unique email, separate insert input/update patch,
DO NOTHING on conflict and explicit NULL.

```text
Inserted: Ana
Updated: Ana Maria
Conflict with empty patch: skipped
Note: cleared
Rows: 1
```

### 3. Related writes

[relationships.zolo](relationships.zolo): parent and children in one
transaction, explicit relation loading, an index and an enforced delete cascade.

```text
Ana: 2 articles
Articles after deleting author: 0
```

### 4. Persistent schema

[migration_demo/](migration_demo/README.md) is a separate project with a file
database. Follow its commands to generate, inspect, apply, check and roll back
a migration, then run an application that does not create its own tables.
Its repeated upsert keeps one demo row.

## Explore a specific contract

From `examples/`, run `zolo run <file>.zolo --no-cache`.

| Program | What it demonstrates |
| --- | --- |
| [aggregates.zolo](aggregates.zolo) | Computed selections, empty aggregates, grouped totals and group-aware pages. |
| [advanced_cursors.zolo](advanced_cursors.zolo) | Mixed directions, NULL ordering, hidden projection keys and joined continuation. |
| [builtin_codecs.zolo](builtin_codecs.zolo) | Validated dates/UTC timestamps, exact decimals and real binary storage. |
| [advanced_schemas.zolo](advanced_schemas.zolo) | Composite keys, typed foreign-key descriptors and advanced indexes. |
| [named_patches.zolo](named_patches.zolo) | Named changes, omitted fields, explicit NULL, immutable patches and upserts. |
| [domain_codecs.zolo](domain_codecs.zolo) | Enum/newtype fields, private codecs, bound domain filters and contextual decode errors. |
| [composed_queries.zolo](composed_queries.zolo) | Four named sources, chained LEFT/INNER joins, self joins, facade imports and stable pages. |
| [dx.zolo](dx.zolo) | Defaults, NULL, typed patches and query ergonomics. |
| [dx_imports.zolo](dx_imports.zolo) | Imported models and generated inputs, private defaults and SQL schema checking. |
| [expressions.zolo](expressions.zolo) | Captured query expressions, bound values and projections. |
| [typed_indexes.zolo](typed_indexes.zolo) | Typed composite indexes, mapped columns and named unique upserts. |
| [pagination.zolo](pagination.zolo) | Row presence, numbered pages, cursor keys, joined ordering and invalid requests. |
| [required_reads.zolo](required_reads.zolo) | Optional/required reads, existence, pagination and decode errors. |
| [error_handling.zolo](error_handling.zolo) | Recoverable duplicate registration and transaction rollback. |
| [schema_writes.zolo](schema_writes.zolo) | Composite indexes, mapped columns, nullable conflicts, empty patches and trigger behavior. |
| [foreign_keys.zolo](foreign_keys.zolo) | Imported/mapped parent keys, nullable relations and referential actions. |
| [relations.zolo](relations.zolo) | Optional string keys, duplicate parents and stable grouping. |
| [query_plan.zolo](query_plan.zolo) | Lower-level plans and SQL inspection. |
| [integration.zolo](integration.zolo) | CRUD, query and batch integration coverage. |
| [hygiene.zolo](hygiene.zolo) | Fields that resemble generated helper names. |

[models.zolo](models.zolo), [schema_parents.zolo](schema_parents.zolo),
[domain_models.zolo](domain_models.zolo), [aggregate_models.zolo](aggregate_models.zolo),
[composed_models.zolo](composed_models.zolo)
and [composed_views.zolo](composed_views.zolo) are imported fixtures, not
standalone demonstrations.
[benchmark.zolo](benchmark.zolo) is driven by the benchmark script below.

## Run native or LLVM

The three walkthroughs also compile to executable programs. From `examples/`:

```sh
zolo build --emit native upsert.zolo -o ../target/upsert-native
zolo build --emit llvm upsert.zolo -o ../target/upsert-llvm
```

Execute the resulting file; Windows adds `.exe`. For example, in PowerShell:

```powershell
../target/upsert-native.exe
../target/upsert-llvm.exe
```

The output should match the VM output above. Native builds need the Zolo native
toolchain and matching runtime; LLVM additionally needs its configured tools.

## Run the maintained suites

From the repository root:

```sh
zolo run scripts/test.zolo
zolo run scripts/test_backends.zolo
zolo run scripts/benchmark.zolo --backend vm
```

The VM suite includes the walkthroughs, advanced examples, package tests and
compile-fail fixtures. The backend suite includes the walkthroughs and selected
regression examples on native/LLVM. The migration demo is validated separately
through its documented CLI workflow. Benchmark results live in
`target/benchmarks/`; they are measurements, not pass/fail examples.

### Typed joins

Run `zolo run joins.zolo` to exercise INNER/LEFT relation joins with mapped
columns, imports and reexports, self joins, ON/WHERE bindings, pagination and
decode errors. Expected output includes `orm typed joins: ok`.

Optional scalar projections preserve NULL positions; see [nullable projections](nullable_projections.zolo).

### Typed relationship metadata

[typed_relations.zolo](typed_relations.zolo) uses type and field references,
two imported aliases of the same parent, contextual and explicit action enums,
a typed join and cascade updates/deletes. Run it with
`zolo run typed_relations.zolo --no-cache`. Expected output:
`orm typed relation metadata: ok`.

### Typed indexes, presence and pages

[typed_indexes.zolo](typed_indexes.zolo) declares index members as field
references and upserts through a composite unique key. Expected output:
`orm typed indexes: ok`.

[pagination.zolo](pagination.zolo) distinguishes a present NULL from no row,
pages models and projections, breaks ties in joins using both primary keys,
and continues int/string keys in either direction. It also includes malformed
rows and invalid page requests. Expected output:
`orm presence and pagination: ok`.

### Named patches, composed queries and codecs

[named_patches.zolo](named_patches.zolo) expects `orm named patches: ok`.
[composed_queries.zolo](composed_queries.zolo) expects `orm composed queries: ok`.
[domain_codecs.zolo](domain_codecs.zolo) expects `orm domain codecs: ok`.

These examples are registered in the VM/package and native/LLVM suites. They
use local package sources and need a matching compiler with the metadata and
nominal-type support described in [compatibility](../docs/errors-and-compatibility.md#compatibility).

### Advanced queries, cursors, codecs and schemas

[aggregates.zolo](aggregates.zolo) expects `orm aggregates: ok`.
[advanced_cursors.zolo](advanced_cursors.zolo) expects `orm advanced cursors: ok`.
[builtin_codecs.zolo](builtin_codecs.zolo) expects `orm builtin codecs: ok`.
[advanced_schemas.zolo](advanced_schemas.zolo) expects `orm advanced schemas: ok`.

These new regression sources are written and cataloged; execution validation of
the current development batch is pending.
