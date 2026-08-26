# NestJS logging

Query logging is enabled by the `logQueries` option of `YdbCoreModule.forRootAsync()`.

## Enabling

```ts
import { Module } from '@nestjs/common';
import { YdbCoreModule } from '@ycforge/ydb-orm/nest';
import { createAuth, authKeyFromFile } from '@ycforge/auth';

@Module({
  imports: [
    YdbCoreModule.forRootAsync({
      useFactory: () => ({
        endpoint: '...',
        auth: createAuth(authKeyFromFile('./authorized_key.json')),
        logQueries: true, // console logger by default
        // logQueries: myLogger, // or a custom QueryLogger
      }),
    }),
  ],
})
export class AppModule {}
```

- `logQueries: true` — uses `ConsoleQueryLogger`, prints `[YDB] QUERY <ms>` with SQL and masked parameters.
- `logQueries: <QueryLogger>` — custom logger instance.

## Custom logger backed by Nest Logger

Implement `QueryLogger` and delegate to the framework logger:

```ts
import { Injectable, Logger } from '@nestjs/common';
import type { QueryLogger, QueryLogEntry } from '@ycforge/ydb-orm';

@Injectable()
export class YdbQueryLogger implements QueryLogger {
  private readonly logger = new Logger('YDB');

  log(entry: QueryLogEntry): void {
    // entry.sql, entry.paramNames, entry.maskedParams,
    // entry.durationMs, entry.error?
    this.logger.log(`${entry.durationMs}ms ${entry.sql}`);
  }
}

// In the module:
YdbCoreModule.forRootAsync({
  useFactory: (logger: YdbQueryLogger) => ({
    endpoint: '...',
    auth: createAuth(authKeyFromFile('./authorized_key.json')),
    logQueries: logger,
  }),
  inject: [YdbQueryLogger],
})
```

## What is logged

Each `QueryLogEntry` contains:

| Field | Description |
| --- | --- |
| `sql` | the YQL statement |
| `paramNames` | parameter names |
| `maskedParams` | parameter values after masking |
| `durationMs` | execution time |
| `error?` | error, if the query failed |

**Parameter masking** protects secrets/PII: values of parameters named like `password`, `token`, `secret`, `authorization`, `email`, `credential`, `phone`, `card`, blind-index columns `{field}_bi`, etc. are replaced with `<redacted>`; binary/ciphertext values are logged as `<bytes:N>`; other long strings are truncated to 64 characters.

Every query inside `runInTransaction` is logged as well.

## Standalone alternative

Without NestJS pass `logQueries` to `createExecutor(driver, opts)` or wrap an executor with `wrapExecutorWithLogging(executor, logger)` — see [Edge cases](../edge-cases.md#query-logging).
