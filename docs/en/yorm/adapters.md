# Database adapters

The ORM core is not tied to a specific database: persistence, relations, QueryBuilder, migrations, and transactions rely on the executor interfaces (`YdbExecutor`/`YdbQuery`), while all database specifics — the driver, value mapping, error classification, retry policy, DDL and schema introspection — are encapsulated in an **adapter** implementing the `OrmAdapter` interface.

The built-in **YDB adapter** (`ydbAdapter`) is used by default, so working with YDB requires no extra configuration.

## The OrmAdapter interface

```ts
import type { OrmAdapter } from '@ycforge/yorm';
```

Adapter fields:

| Field | Purpose |
|-------|---------|
| `name` | Adapter name (e.g. `'ydb'`) — used in diagnostics and logs |
| `validateModuleOptions(opts, injected?)` | Fail-fast validation of module options before the driver is created |
| `resolveCredentialsProvider(opts, injected?)` | Resolves the `CredentialsProvider` by source priority |
| `createDriver(opts, credentialsProvider?)` | Creates a connected driver |
| `createExecutor(driver, opts)` | Creates an executor (`YdbExecutor`) on top of the driver |
| `mapValue(type, value, field?)` | Converts a JS value to a database value (counterpart of `mapToYdb`) |
| `classifyError(error)` | Structural error classification: `'transient'` / `'fatal'` |
| `isTransientError(error)` | Whether the error is transient (retryable) |
| `resolveRetryPolicy(input)` | Resolves the retry policy (`false`/`true`/object → config or `null`) |
| `withRetryPolicy(executor, policyInput?)` | Wraps an executor with the retry policy |
| `createSchemaSyncer(driver, executor)` | Schema synchronizer (DDL + table introspection) |

## Plugging in an adapter

The adapter is selected via the optional `adapter` field in `YdbModuleOptions`. Without it, `ydbAdapter` is used:

```ts
import { createAuth } from '@ycforge/auth';
import { YdbCoreModule } from '@ycforge/yorm';
import { ydbAdapter } from '@ycforge/yorm/ydb';

@Module({
  imports: [
    YdbCoreModule.forRootAsync({
      useFactory: () => ({
        endpoint: 'grpcs://ydb.serverless.yandexcloud.net:2135/ru-central1/...',
        auth: createAuth(...),
        adapter: ydbAdapter, // optional: this is the default anyway
      }),
    }),
  ],
})
export class AppModule {}
```

A custom adapter (e.g. a wrapper adding metrics, or a stub for tests) is passed through the same `adapter` field.

## The `@ycforge/yorm/ydb` subpath export

The YDB adapter implementation is available via a dedicated subpath import:

```ts
import { ydbAdapter, createDriver, createExecutor, mapToYdb } from '@ycforge/yorm/ydb';
```

It also exports `YdbSchemaSyncer`, the DDL generators (`generateCreateTableYql` and others), the retry utilities (`runWithRetry`, `withRetryPolicy`, `resolveYdbRetryPolicy`), and the related types. All of these are available from the package root as well — the subpath exists for working with the YDB adapter explicitly.

{% note info %}

Public API names (`YdbEntity`, `YdbRepository`, `mapToYdb`, etc.) do not depend on the adapter and were not renamed. The internal paths (`core/driver.js`, `core/mapper.js`, `core/retry.js`, `schema/schema-sync.js`) are kept as re-export shims for backward compatibility.

{% endnote %}

## Other databases

Adapters for other databases (PostgreSQL, etc.) will ship in future releases — the `OrmAdapter` interface already separates their future implementations from the ORM core.
