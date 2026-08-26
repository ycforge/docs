# AbortSignal

Every query and transaction can be cancelled or time-limited via `AbortSignal`.

## Cancelling a query

Pass `signal` in `QueryOptions` — the second argument of all Active Record methods, repository methods, and QueryBuilder's `options()`:

```ts
const controller = new AbortController();

setTimeout(() => controller.abort(new Error('Client shutdown')), 5_000);

const users = await UserEntity.findAll(
  { role: 'admin' },
  { signal: controller.signal }, // query is cancelled after 5 s
);
```

On cancellation the promise rejects with the signal's reason (`controller.abort(reason)`); the query does not continue on the server beyond the transport cancel.

## Timeouts

`timeout` in `QueryOptions` sets a per-query deadline in milliseconds:

```ts
// Equivalent to passing AbortSignal.timeout(5000)
const user = await UserEntity.find({ uuid }, { timeout: 5_000 });
```

A total deadline via an explicit signal:

```ts
const user = await UserEntity.find(
  { uuid },
  { signal: AbortSignal.timeout(30_000) },
);
```

`AbortSignal.timeout(ms)` fires automatically after `ms` — no controller or cleanup needed.

## QueryBuilder

```ts
const photos = await PhotoEntity.query()
  .where({ is_public: true })
  .options({ signal: controller.signal, timeout: 10_000 })
  .getMany();
```

## Raw executor queries

The executor query object exposes `.signal()`:

```ts
await executor`
  SELECT * FROM users WHERE name = $name
`
  .parameter('name', 'John')
  .signal(controller.signal)
  .execute();
```

## Transactions

In `runInTransaction(fn, options)` the scopes differ:

- `signal` — **global**: cancels the whole operation including all retry attempts;
- `timeout` — **per attempt**: with retries (`idempotent: true`) each attempt gets a fresh window; the callback receives a merged signal as the second argument.

```ts
import { YdbTransactionManager } from '@ycforge/ydb-orm';

const txManager = new YdbTransactionManager(executor);

await txManager.runInTransaction(
  async (trx, signal) => {
    // signal merges the attempt signal + this attempt's timeout
    await OrderEntity.save(order, { trx });
    if (signal.aborted) throw new Error('Cancelled');
  },
  {
    timeout: 5_000,
    signal: AbortSignal.timeout(30_000), // total deadline for all attempts
  },
);
```

With the ORM [retry policy](transactions.md#retry-policy), cancellation is never converted into a retry — the operation ends with the cancellation reason (`signal.reason`).

## Bulk operations

`insertMany`, `updateBy`, `deleteBy` accept the same `QueryOptions`:

```ts
await UserEntity.insertMany(users, {
  timeout: 10_000,
  signal: req.signal, // e.g. incoming HTTP request aborted
});
```

## Practical patterns

**HTTP request scope** (cancel DB work when the client disconnects):

```ts
app.post('/users', async (req, res) => {
  const user = await UserEntity.save(buildUser(req.body), { signal: req.signal });
  res.json(user);
});
```

**Racing against business logic:**

```ts
const controller = new AbortController();
const work = exportReport({ signal: controller.signal });
const done = await Promise.race([
  work,
  new Promise((_, reject) =>
    setTimeout(() => controller.abort(), 60_000),
  ),
]);
```

{% note info %}

Signals/timeout flags are recorded per query by the driver; a timed-out query surfaces as a YDB timeout error rather than hanging forever. Marking a query `{ idempotent: true }` allows retries only for transient statuses — timeouts of deterministic queries are not retried silently.

{% endnote %}

## Next steps

- [Transactions](transactions.md) — execution options and retry semantics
- [NestJS AbortSignal](nest/abort-signal.md) — request-scoped cancellation if you use NestJS
