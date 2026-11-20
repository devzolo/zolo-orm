# ERP persistence

[Documentation](README.md) · [Concurrency example](../examples/erp_concurrency.zolo) ·
[Workflow example](../examples/erp_workflows.zolo) · [Financial values](financial-values.md)

This development branch contains source changes and regression cases.
They have **not been compiled or executed**. Use a matching compiler/runtime/LSP;
the installed toolchain has not been refreshed.

## Reserve stock atomically

Use a filtered captured update when the new value depends on the stored value:

```rust
let result = Stock::query()
  .filter(|stock| stock.company_id == company && stock.id == product
    && stock.available >= quantity)
  .update(db, |stock| #{available: stock.available - quantity})?
result.expect_one()?
```

This is one parameterized UPDATE. It avoids reading a quantity and subsequently
overwriting another user's reservation. Assignment values are bound before
predicate values. Unknown fields, type mismatches and assignments to keys,
scope fields or revision fields are rejected. The runtime also checks immutable
physical column names when accepting a constructed plan.

An unfiltered update, joined/grouped/projected write, ordering, limit or offset
is rejected. Empty updates are rejected. A constant-true predicate does not
authorize a whole-table update. An explicit administrative raw Statement remains
available when the application intentionally needs a different operation.

MutationResult.affected is the database's affected-row count. expect_one returns
OrmError::Conflict(count) unless exactly one row changed. Zero rows can mean
absence, an old revision or a failed condition; it does not manufacture a
specific business diagnosis from that count.

## Optimistic revisions

```rust
@revision
revision: int = 0,
```

One nonoptional integer field can be the revision. It is not a key or scope field.
Generated instance update compares the instance's revision and increments it in
the same statement. The instance itself is unchanged: read the returned/latest
row for the new revision.

```rust
Account::changes(name: "Ana")
  .apply_if_version(db, account.id, expected: account.revision)?
```

Blind patch apply on a versioned model returns an error. Revisions cannot be set
through named changes. Captured conditional updates increment revisions for all
affected rows, protecting previously read instances against those writes.
They are conditional business operations, not implicit checks of a previously
read version. Instance deletion also checks the revision.

Versioned upsert allows DO NOTHING but rejects conflict updates with an unknown
revision. Use an explicit versioned update for an existing record.

## Explicit company/branch scope

```rust
@key([.company_id, .id])
@scope([.company_id])
struct Stock deriving Model {
  company_id: int,
  id: int,
  available: int,
  @revision
  revision: int = 0,
}

let stock = Stock::scoped(db, company_id: current_company)
let rows = stock.all()?
let record = stock.find_or_error(StockKey.new(current_company, product))?
stock.apply_if_version(record.key(), Stock::changes(available: 10),
  expected: record.revision)?
```

Scoped repositories add all scope fields to key reads, writes and deletes;
inserts validate their values before I/O. Repository create accepts the normal
named inputs except scope fields, which it supplies itself. Multiple scope fields
are supported.
Scope references are actual fields, checked for owner/type and resolved through
mapped column names. Repository query() starts a normal typed query with the
scope predicates already applied.

Scope is an explicit repository contract, not database row-level security.
Unscoped model methods and raw SQL remain administrative escape hatches. Give
application modules scoped repositories instead of unrestricted Database handles
when they should operate within one company. Authentication and role selection
belong to the application.

Use composite foreign keys containing the scope for cross-model isolation.
Joins and preload helpers do not infer another model's policy from a user role.
Scoped repositories intentionally do not offer an upsert that could match an
unrelated company's global unique key.

## A transaction is a lifetime

```rust
use orm::{transact, Actor}

transact(db, |tx| {
  // Use tx.db() with existing model APIs.
  // tx.emit/audit/next_number share the same underlying transaction.
  return Result::Ok(1)
}, actor: Actor.new(user_id, request_id))?
```

The callback gets a distinct database transaction handle backed by the driver's
transaction on one acquired connection. Completion invalidates that handle;
keeping tx or tx.db() in an outer variable does not extend the transaction.

Returned Err and panics roll back. Existing orm::transaction(fn(Database))
remains available for compatibility. Transactions are explicit and must be
propagated, rather than reopening or silently nesting a transaction.

For row locks, use query.locked(tx, wait: .Wait | .NoWait | .SkipLocked).
It accepts an active Transaction and a single-table query. Unsupported dialects,
including SQLite, return an error instead of discarding the lock clause.

Driver error kinds include SerializationFailure, Deadlock and LockTimeout.
DbError.is_retryable() classifies them; it never retries business operations
automatically. Retry a complete transaction only when the application can repeat
its work and external effects safely.

## Validation with before/after data

Model::change(before, after) creates Change<Model>, with typed optional snapshots
and changed fields. A missing before represents insertion; a missing after
represents deletion.

Model::write_change(tx, change, validator) calls the validator **before writing**.
ValidationErrors accumulates path/code/message entries. The validator returns nil;
returning an ignored Result is not part of its contract. Perform fallible prerequisite
checks before this local validation callback. It rejects replacing
keys, scope or revision during an update; a versioned write still checks the
database's current revision.

```rust
let change = Order::change(previous, proposed)
let saved = Order::write_change(tx, change, |change, errors| {
  let units = change.after?.units ?? 0
  errors.require(units > 0, "units", "positive", "Quantity must be positive")
})?
```

This is an explicit validated write path. Ordinary create/insert/update and raw
SQL do not secretly run application rules. Route imports, jobs and UI commands
through the same service when they must share validation. Business processes
such as confirming or invoicing an order compose their effects in that service.

A field marked @redact still appears as changed, but its before and
after values are absent from FieldChange and audit_json never includes it.
The typed snapshots remain available to domain validation. audit_json also
requires an explicit allowlist of fields.

## Audit and durable events

Install infrastructure_schema() through reviewed schema management, or use
create_infrastructure(db) for an isolated example. emit and audit never create
tables implicitly. The helper schema targets PostgreSQL/SQLite.

```rust
tx.audit(AuditEntry.new(
  audit_id, "Order", order_key, "confirmed",
  change.audit_json(["status", "units"]),
))?
tx.emit(OutboxEvent::json(event_id, "order.confirmed", order_key, payload))?
```

Audit requires an Actor. The audit row and event row use the same transaction
as the business change. IDs are application-assigned and unique; duplicates
fail instead of silently dropping an event.

A publisher reads pending_events(db, limit), sends each event, then calls
acknowledge_event(db, event.id). A crash after sending and before acknowledging
can cause another delivery. Consumers must deduplicate by event ID. These helpers
do not start a worker or send network messages, and do not claim exactly-once
delivery, per-aggregate ordering, automatic leases or dead-letter handling.
Retention and sensitive payload choices belong to the application.

## Business numbering and collection edits

tx.next_number("company:7/orders:2026") atomically allocates from a counter.
It must run in an active PostgreSQL/SQLite transaction; rollback also rolls back
the allocation. A counter key should encode an unambiguous company/series/type.
It is separate from generated technical identity, whose values may have gaps.
Do not promise gap-free fiscal numbering without the application's complete
cancellation and document-issuance policy.

diff_collection(before, after, |item| item.id) returns added, kept and removed
items, rejecting duplicate keys. Kept contains the proposed items; compare or
validate them as needed. The helper performs no I/O: explicitly apply the intended
inserts, updates and deletes inside a transaction. An absent payload item is
never automatically deleted.

## Reports and bounded work

Captured selections accept if/else as SQL CASE and ?? as COALESCE. Existing
grouping, aggregates, named views and nullable decoders compose with them.

query_rows(db, statement, decoder) gives a parameterized raw Statement a typed
result decoder for reports beyond the captured vocabulary. Arbitrary functions,
CTEs and window queries remain explicit SQL; this API does not pretend to
statically validate a caller-written decoder against a live schema.

query.each_chunk(db, visitor, size: 256) traverses bounded keyset pages. Return
false from the visitor to stop. Ordering and keys follow seek_page's contract.
This is pagination, not a server-side cursor or a consistent snapshot by itself;
choose transaction isolation when a stable export is required. Mutating traversal
keys while visiting can skip or revisit rows.

## Low-level construction and validation

Generated patch writes accept only distinct, mutable model columns with a bound
value or NULL. Constructing an Assignment manually does not allow changing a key,
scope or revision through those generated entry points. Query mutation plans
check the original filter before adding internal revision-overflow predicates,
so a constant-true filter still fails.

`Model::write_change` rebuilds the field diff from the supplied before/after
models before calling the validator. Its callback records errors and returns
`nil`; an unexpected non-nil return is rejected even when gradual `Any` inference
prevents a compile-time diagnostic. The supplied before snapshot must still come
from the application's intended read; this API is not a universal row policy.
Raw SQL and the general SQL plan primitives remain explicit low-level APIs.
