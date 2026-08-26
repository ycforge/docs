# Transactions

The ORM provides `YdbTransactionManager` for atomic operations. Transactions ensure that multiple writes either all succeed or all fail.

## Usage

```ts
import { YdbTransactionManager } from '@ycforge/ydb-orm';
import type { QueryOptions } from '@ycforge/ydb-orm';
import { UserEntity } from './user.entity';

const txManager = new YdbTransactionManager(executor);

await txManager.runInTransaction(async (trx) => {
  const opts: QueryOptions = { trx };

  const from = await UserEntity.find({ uuid: fromId }, opts);
  const to = await UserEntity.find({ uuid: toId }, opts);

  from.balance -= 100;
  to.balance += 100;

  await UserEntity.save(from, opts);
  await UserEntity.save(to, opts);
  // Both records are applied atomically: either both or none
});
```

Inside the callback, `trx` is the transaction executor. It is passed to all entity methods via `{ trx }` in `QueryOptions`. You can also run a [QueryBuilder](query-builder.md) this way via `options({ trx })`.

## Execution options

`runInTransaction(fn, options?)` accepts options that are forwarded into the SDK transaction call:

```ts
await txManager.runInTransaction(
  async (trx, signal) => {
    await OrderEntity.save(order, { trx });
  },
  {
    isolation: 'snapshotReadWrite', // serializableReadWrite (default) | snapshotReadOnly | snapshotReadWrite
    timeout: 5_000,                 // timeout PER ATTEMPT (see below)
    signal: controller.signal,      // GLOBAL cancellation: the whole operation, all attempts
    idempotent: true,               // see "Retry semantics" below
  },
);
```

| Option | Description |
| --- | --- |
| `isolation` | YDB isolation level: `serializableReadWrite` (default), `snapshotReadOnly`, `snapshotReadWrite` |
| `timeout` | per-attempt timeout in milliseconds; each retry attempt gets a fresh window |
| `signal` | global `AbortSignal`: cancels the entire operation including all attempts |
| `idempotent: true` | allows the SDK to re-run the callback on retryable errors (callback must tolerate re-execution) |
| `retry` | ORM-owned retry policy for the body (`true` or a policy object); cannot be combined with `reuse` |
| `reuse: true` | join the currently active transaction instead of throwing on nesting |
| `ambient: true` | expose the transaction to the ambient context: operations without explicit `{ trx }` join it |

Options are validated fail-fast: unknown keys (typos), invalid isolation levels, non-positive `timeout`, non-`AbortSignal` values error immediately.

**Cancellation semantics with `idempotent: true`** — `signal` and `timeout` have different scopes:

- `signal` is **global**: passed to the SDK as-is, cancels the operation entirely, including all retries;
- `timeout` is **per attempt**: the SDK may re-run the callback (`idempotent: true`), and each attempt gets a fresh timeout window — a retry never starts with the first attempt's deadline already expired. The signal the callback receives (`fn(trx, signal)`) merges the SDK's per-attempt signal with that attempt's `AbortSignal.timeout(timeout)`.

A total deadline for the whole operation is set explicitly through a user signal:

```ts
await txManager.runInTransaction(fn, {
  idempotent: true,
  signal: AbortSignal.timeout(30_000), // shared limit across all attempts
});
```

See [AbortSignal](abort-signal.md) for cancellation details.

## Retry policy

With `idempotent: true` the SDK re-runs the **entire callback** on retryable errors (network failures, session death):

- side effects of the callback run again;
- lifecycle hooks (`@BeforeInsert`, `@AfterInsert`, ...) fire more than once;
- each attempt gets a new session/transaction (a new `trx`).

To own the retry loop yourself (bounded attempts), pass a `retry` policy:

```ts
await txManager.runInTransaction(
  async (trx) => {
    await OrderEntity.save(order, { trx });
  },
  {
    idempotent: true,   // the callback must tolerate re-execution
    retry: true,        // or a policy object; cannot be combined with reuse
    maxAttempts: 5,
  },
);
```

With a policy, the ORM owns body retries: bounded backoff with jitter between attempts, only transient statuses retried (`ABORTED`, `UNAVAILABLE`, `OVERLOADED`), `timeout` still applies per attempt, global `signal` and `idempotent` pass through to the SDK. Without `retry`, only the SDK retries the body.

## Nested transactions

By default a nested `runInTransaction()` is **forbidden**: a second call would open an independent transaction on another session, which is almost always a bug. To join the active transaction (commit/rollback stay with the outer call), pass `{ reuse: true }`:

```ts
await txManager.runInTransaction(async () => {
  await txManager.runInTransaction(async (trx2) => {
    // Error: Nested runInTransaction() detected ...
  });
});

await txManager.runInTransaction(async () => {
  await txManager.runInTransaction(async (sameTrx) => {
    // same transaction as outside
  }, { reuse: true });
});
```

Nesting is detected via the AsyncLocalStorage chain and only for the same DB executor; nested transactions on another driver/database are considered independent.

## Ambient context (opt-in)

One missed `{ trx }` — and a query silently runs outside the transaction. Ambient mode fixes this: repository operations without an explicit `{ trx }` automatically execute inside the active transaction.

```ts
// Globally for the process — NestJS module option:
YdbCoreModule.forRootAsync({
  useFactory: () => ({
    // ...
    transactions: { ambient: true },
  }),
});

// Or per call:
await txManager.runInTransaction(async () => {
  await OrderEntity.save(order);          // joins the transaction automatically
  await OrderEntity.save(other, { trx }); // explicit trx works too
}, { ambient: true });
```

Safety rules:

- if a **different** `{ trx }` is explicitly passed while an ambient transaction is active — a mixing error is thrown instead of silent data divergence;
- after commit/rollback the context is cleared;
- parallel transactions do not leak into each other;
- ambient is off by default: explicit `{ trx }` behaves as before.

## Queries outside transactions

For debugging you can enable a warning for every query running outside any transaction:

```ts
transactions: { warnOutsideTransaction: true } // console.warn per such query
```

Off by default.

## Bulk operations

`insertMany`, `updateBy`, and `deleteBy` also accept `{ trx }` and run inside a transaction.

## What transactions do not cover

DDL (CREATE TABLE, ALTER TABLE, DROP TABLE) in YDB is not transactional — migration execution is sequential. Use [migrations](migrations.md) to change the schema.

## Next steps

- [AbortSignal](abort-signal.md) — cancellation and timeouts in queries
- [NestJS transactions](nest/transactions.md) — DI-injected pattern if you use NestJS
