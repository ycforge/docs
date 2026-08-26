# Edge cases

Unique scenarios and advanced configuration: driver substitution for tests, query logging, retry policy, limit semantics, metadata inheritance.

## Substituting the driver (testing)

### Standalone

In standalone mode any object implementing the `YdbExecutor` interface can be passed to `configureEntities` — a fake or a mock:

```ts
configureEntities([UserEntity], { executor: myFakeExecutor });
```

A minimal fake executor records queries and returns prepared rows:

```ts
const calls: Array<{ sql: string; params: Record<string, unknown> }> = [];

const fakeExecutor = async (strings: TemplateStringsArray, ...values: unknown[]) => {
  const sql = strings.join('?');
  return {
    parameter(name: string, value: unknown) {
      values.push(value);
      return this;
    },
    async execute() {
      return [{ id: 'uuid-1', name: 'John' }];
    },
  };
};

configureEntities([UserEntity], { executor: fakeExecutor as never });
```

### NestJS

In NestJS tests, substitute DI providers `YDB_DRIVER` / `YDB_QUERY` via `overrideProvider` — no network access happens:

```ts
import { Test } from '@nestjs/testing';
import { YDB_DRIVER, YDB_QUERY } from '@ycforge/ydb-orm/nest';

const moduleRef = await Test.createTestingModule({
  imports: [AppModule],
})
  .overrideProvider(YDB_DRIVER)
  .useValue(mockDriver)
  .overrideProvider(YDB_QUERY)
  .useValue(mockExecutor)
  .compile();
```

The `driverFactory` option also allows supplying your own driver factory (tests, custom transports). A driver created this way is considered owned by the module and is closed on shutdown; drivers passed via `overrideProvider` are not closed.

## Query logging

### Enabling

The `logQueries` option (module options in NestJS, `createExecutor` options standalone) enables logging of every query:

```ts
logQueries: true,        // ConsoleQueryLogger — prints `[YDB] QUERY <ms>` with SQL
// logQueries: myLogger, // or a custom QueryLogger instance
```

Standalone equivalent:

```ts
const executor = createExecutor(driver, {
  ...options,
  logQueries: true,
});
```

### Custom logger

Implement the `QueryLogger` interface. The entry contains SQL, masked parameters, duration and an optional error:

```ts
import type { QueryLogger, QueryLogEntry } from '@ycforge/ydb-orm';

class MyLogger implements QueryLogger {
  log(entry: QueryLogEntry): void {
    // entry.sql, entry.paramNames, entry.maskedParams,
    // entry.durationMs, entry.error?
    console.log(`[ydb] ${entry.durationMs}ms ${entry.sql}`);
  }
}
```

### Wrapping an existing executor

`wrapExecutorWithLogging(executor, logger)` adds logging to a ready executor manually — it also logs every query inside `runInTransaction`:

```ts
import { wrapExecutorWithLogging } from '@ycforge/ydb-orm';

const logged = wrapExecutorWithLogging(executor, new MyLogger());
```

### Parameter masking

Parameters are masked by name before they reach the log: secrets/PII (`password`, `token`, `secret`, `authorization`, `email`, `credential`, `phone`, `card`, blind index columns `{field}_bi`, etc.) are replaced with `<redacted>` regardless of length; binary/ciphertext values are logged as `<bytes:N>` only; other long strings are truncated to 64 characters.

## Retry policy by error type

The SDK retries internally with an unlimited budget. The ORM can attach its own bounded policy so that retry layers do not multiply. Only transient statuses are retried: `ABORTED`, `UNAVAILABLE`, `OVERLOADED`.

```ts
import { runWithRetry } from '@ycforge/ydb-orm';

// Standalone: on the executor
const executor = createExecutor(driver, { ...options, retry: { maxAttempts: 5 } });

// For composite flows outside transactions:
const result = await runWithRetry(async () => {
  const user = await UserEntity.find({ uuid }, {});
  const orders = await OrderEntity.findBy({ userId: user.uuid }, { limit: 50 });
  return buildReport(user, orders);
}, { maxAttempts: 5 });
```

{% note warning %}

**Idempotency rule**: the policy retries only queries explicitly marked `.idempotent(true)` / `{ idempotent: true }`. Unmarked queries (including all writes by default) run exactly once even with the policy enabled. Do not nest `runWithRetry()` inside executors/transactions already covered by a policy.

{% endnote %}

Defaults: `maxAttempts: 3`, `baseDelayMs: 100`, `maxDelayMs: 5000`, `jitterRatio: 0.25`. On exhaustion the last original error propagates as-is.

See [Transactions](transactions.md#retry-policy) for transaction-level retries.

## LIMIT/OFFSET semantics

Explicit semantics without silent clamping:

| Call | Final `LIMIT` |
| --- | --- |
| no limit set | `100` — safe default |
| `limit(0)` | `0` — guaranteed empty result |
| `limit(n)`, 1 ≤ n ≤ 1000 | `n` |
| `limit(n)`, n > 1000 | `1000` — safe ceiling |
| negative / fractional / non-finite | `Invalid LIMIT` error |

`offset`: unset → 0, fractional → floor, negative → 0.

## Metadata inheritance

Rules between parent and child entity classes:

- **`@YdbEntity` is not inherited.** A subclass without its own `@YdbEntity` is not an entity: it does not inherit the parent's tableName, does not enter the registry or schema-sync expected schemas, and Active Record calls on it fail with a clear error.
- **Columns are inherited** (`@YdbColumn`, `@YdbPrimaryColumn`, `@YdbEncrypted`, `@YdbSecurityAAD`, `@YdbJson`, `@YdbEnum`, timestamps, lifecycle hooks): the child gets the union of ancestors' metadata; overrides use copy-on-write and never mutate the parent.
- **`@YdbIndex` and `@YdbTtl` are not inherited** — they belong to the physical table of the class. Each class declares its own explicitly, so parent indexes/TTLs never leak into child DDL.
- **`@EagerLoad` merges**: parent relations are preserved; the child's list extends them without duplicates (first declaration wins).
- **Duplicate tableName in two different entities** — error `Duplicate table name "..."` during schema building.

## Module protections (NestJS)

- **Double initialization guard**: re-initializing the core while the previous app is still open fails with `Duplicate YDB module initialization`. Sequential bootstraps (tests, hot-restart) after `app.close()` are allowed.
- **Schema sync runs in `onApplicationBootstrap`**, not in the DI factory — all entities of all modules are registered by then; in tests sync fires after `module.init()` / `app.init()`, not after `compile()`.
- **Entity registry isolation**: each `YdbCoreModule` instance (each Nest app) has its own entity scope, so parallel test applications cannot pollute each other.
- **Graceful shutdown**: the module-owned driver is closed in `onApplicationShutdown` (enable `app.enableShutdownHooks()`).

## Notes

- `@bufbuild/protobuf` is pinned to exactly `2.12.0`: on `^` the `anyUnpack` typing breaks due to branded-type divergence with `@ydbjs/*`.
- All queries are parameterized (`query.parameter(...)`) — values are never concatenated into SQL.
