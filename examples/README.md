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
| [dx.zolo](dx.zolo) | Defaults, NULL, typed patches and query ergonomics. |
| [dx_imports.zolo](dx_imports.zolo) | Imported models and generated inputs, private defaults and SQL schema checking. |
| [expressions.zolo](expressions.zolo) | Captured query expressions, bound values and projections. |
| [required_reads.zolo](required_reads.zolo) | Optional/required reads, existence, pagination and decode errors. |
| [error_handling.zolo](error_handling.zolo) | Recoverable duplicate registration and transaction rollback. |
| [schema_writes.zolo](schema_writes.zolo) | Composite indexes, mapped columns, nullable conflicts, empty patches and trigger behavior. |
| [foreign_keys.zolo](foreign_keys.zolo) | Imported/mapped parent keys, nullable relations and referential actions. |
| [relations.zolo](relations.zolo) | Optional string keys, duplicate parents and stable grouping. |
| [query_plan.zolo](query_plan.zolo) | Lower-level plans and SQL inspection. |
| [integration.zolo](integration.zolo) | CRUD, query and batch integration coverage. |
| [hygiene.zolo](hygiene.zolo) | Fields that resemble generated helper names. |

[models.zolo](models.zolo) and [schema_parents.zolo](schema_parents.zolo)
are imported fixtures, not standalone demonstrations.
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
