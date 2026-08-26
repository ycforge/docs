# Логирование (NestJS)

Логирование запросов включается опцией `logQueries` в `YdbCoreModule.forRootAsync()`.

## Включение

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
        logQueries: true, // консольный логгер по умолчанию
        // logQueries: myLogger, // или свой QueryLogger
      }),
    }),
  ],
})
export class AppModule {}
```

- `logQueries: true` — используется `ConsoleQueryLogger`, вывод `[YDB] QUERY <мс>` с SQL и замаскированными параметрами.
- `logQueries: <QueryLogger>` — собственный экземпляр логгера.

## Собственный логгер на базе Nest Logger

Реализуйте `QueryLogger` и делегируйте во фреймворк-логгер:

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

// В модуле:
YdbCoreModule.forRootAsync({
  useFactory: (logger: YdbQueryLogger) => ({
    endpoint: '...',
    auth: createAuth(authKeyFromFile('./authorized_key.json')),
    logQueries: logger,
  }),
  inject: [YdbQueryLogger],
})
```

## Что попадает в лог

Каждый `QueryLogEntry` содержит:

| Поле | Описание |
| --- | --- |
| `sql` | текст YQL-запроса |
| `paramNames` | имена параметров |
| `maskedParams` | значения параметров после маскирования |
| `durationMs` | время выполнения |
| `error?` | ошибка, если запрос упал |

**Маскирование параметров** защищает секреты/PII: значения параметров с именами вида `password`, `token`, `secret`, `authorization`, `email`, `credential`, `phone`, `card`, blind-index колонок `{field}_bi` и т.п. заменяются на `<redacted>`; бинарные/зашифрованные значения логируются как `<bytes:N>`; остальные длинные строки обрезаются до 64 символов.

Каждый запрос внутри `runInTransaction` также логируется.

## Standalone-альтернатива

Без NestJS передавайте `logQueries` в `createExecutor(driver, opts)` или оборачивайте executor через `wrapExecutorWithLogging(executor, logger)` — см. [Особые случаи](../edge-cases.md#логирование-запросов).
