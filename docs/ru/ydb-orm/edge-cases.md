# Особые случаи

Уникальные сценарии и расширенная настройка: подмена драйвера для тестов, логирование запросов, retry-политика, семантика лимитов, наследование метаданных.

## Подмена драйвера (тесты)

### Standalone

В standalone-режиме в `configureEntities` можно передать любой объект, реализующий интерфейс `YdbExecutor` — фейк или мок:

```ts
configureEntities([UserEntity], { executor: myFakeExecutor });
```

Минимальный фейк записывает запросы и возвращает подготовленные строки:

```ts
const fakeExecutor = async (strings: TemplateStringsArray, ...values: unknown[]) => {
  return {
    parameter(name: string, value: unknown) {
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

В NestJS-тестах подмените DI-провайдеры `YDB_DRIVER` / `YDB_QUERY` через `overrideProvider` — сеть не используется:

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

Опция `driverFactory` также позволяет подставить собственную фабрику драйвера (тесты, нестандартные транспорты). Такой драйвер считается принадлежащим модулю и закрывается при shutdown; драйверы, переданные через `overrideProvider`, не закрываются.

## Логирование запросов

### Включение

Опция `logQueries` (в опциях модуля NestJS, в опциях `createExecutor` — standalone) включает логирование каждого запроса:

```ts
logQueries: true,        // ConsoleQueryLogger — печатает `[YDB] QUERY <мс>` с SQL
// logQueries: myLogger, // или собственный QueryLogger
```

Standalone-эквивалент:

```ts
const executor = createExecutor(driver, {
  ...options,
  logQueries: true,
});
```

### Собственный логгер

Реализуйте интерфейс `QueryLogger`. В записи — SQL, замаскированные параметры, длительность и опциональная ошибка:

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

### Оборачивание готового executor'а

`wrapExecutorWithLogging(executor, logger)` добавляет логирование к готовому executor'у вручную — она же логирует каждый запрос внутри `runInTransaction`:

```ts
import { wrapExecutorWithLogging } from '@ycforge/ydb-orm';

const logged = wrapExecutorWithLogging(executor, new MyLogger());
```

### Маскирование параметров

Параметры маскируются по имени до попадания в лог: секреты/PII (`password`, `token`, `secret`, `authorization`, `email`, `credential`, `phone`, `card`, blind-index колонки `{field}_bi` и т.п.) заменяются на `<redacted>` независимо от длины; бинарные/зашифрованные значения логируются только длиной (`<bytes:N>`); остальные длинные строки обрезаются до 64 символов.

## Retry-политика по типу ошибки

SDK ретраит внутренние запросы с неограниченным бюджетом. ORM может подключить свою политику с ограничением попыток так, чтобы слои повтора не перемножались. Повторяются только транзитные статусы: `ABORTED`, `UNAVAILABLE`, `OVERLOADED`.

```ts
import { runWithRetry } from '@ycforge/ydb-orm';

// Standalone: на executor'е
const executor = createExecutor(driver, { ...options, retry: { maxAttempts: 5 } });

// Для составных потоков вне транзакций:
const result = await runWithRetry(async () => {
  const user = await UserEntity.find({ uuid }, {});
  const orders = await OrderEntity.findBy({ userId: user.uuid }, { limit: 50 });
  return buildReport(user, orders);
}, { maxAttempts: 5 });
```

{% note warning %}

**Правило идемпотентности**: политика повторяет только запросы, явно помеченные `.idempotent(true)` / `{ idempotent: true }`. Непомеченные запросы (включая все записи по умолчанию) выполняются ровно один раз даже при включённой политике. Не вкладывайте `runWithRetry()` внутрь executor'ов/транзакций, уже покрытых политикой.

{% endnote %}

Дефолты: `maxAttempts: 3`, `baseDelayMs: 100`, `maxDelayMs: 5000`, `jitterRatio: 0.25`. При исчерпании попыток наружу выходит последняя исходная ошибка как есть.

Ретраи на уровне транзакций — см. [Транзакции](transactions.md#retry-политика).

## Семантика LIMIT/OFFSET

Явная семантика без молчаливого clamp:

| Вызов | Итоговый `LIMIT` |
| --- | --- |
| лимит не задан | `100` — защитный дефолт |
| `limit(0)` | `0` — гарантированно пустой результат |
| `limit(n)`, 1 ≤ n ≤ 1000 | `n` |
| `limit(n)`, n > 1000 | `1000` — защитный потолок |
| отрицательное / дробное / неконечное | ошибка `Invalid LIMIT` |

`offset`: не задан → 0, дробное → floor, отрицательное → 0.

## Наследование метаданных

Правила между родительским и дочерним классами:

- **`@YdbEntity` не наследуется.** Подкласс без собственного `@YdbEntity` — не сущность: не наследует tableName родителя, не попадает в реестр и expected-схемы schema sync, а Active Record-вызовы на нём падают с понятной ошибкой.
- **Колонки наследуются** (`@YdbColumn`, `@YdbPrimaryColumn`, `@YdbEncrypted`, `@YdbSecurityAAD`, `@YdbJson`, `@YdbEnum`, timestamp-декораторы, lifecycle-хуки): дочерний класс получает объединение метаданных предков; переопределения copy-on-write не мутируют родителя.
- **`@YdbIndex` и `@YdbTtl` не наследуются** — они привязаны к физической таблице класса. Каждый класс объявляет свои явно, поэтому индексы/TTL родителя никогда не попадут в DDL дочерней таблицы.
- **`@EagerLoad` наследуется объединением**: связи родителя сохраняются, список ребёнка дополняет их без повторов (первое объявление выигрывает).
- **Дубликат tableName у двух разных сущностей** — ошибка `Duplicate table name "..."` при построении схемы.

## Защиты модуля (NestJS)

- **Защита от двойного `forRootAsync`**: повторная инициализация ядра, пока предыдущее приложение не закрыто, падает с ошибкой `Duplicate YDB module initialization`. Последовательные бутстрапы (тесты, hot-restart) после `app.close()` разрешены.
- **Schema sync выполняется в `onApplicationBootstrap`**, а не в DI-фабрике — к этому моменту зарегистрированы все сущности всех модулей; в тестах sync срабатывает после `module.init()` / `app.init()`, а не после `compile()`.
- **Изоляция реестра сущностей**: у каждого экземпляра `YdbCoreModule` (каждого Nest-приложения) свой скоуп сущностей — параллельные тестовые приложения не загрязняют друг друга.
- **Graceful shutdown**: принадлежащий модулю драйвер закрывается в `onApplicationShutdown` (включите `app.enableShutdownHooks()`).

## Примечания

- Версия `@bufbuild/protobuf` запинена ровно на `2.12.0`: на `^` ломается типизация `anyUnpack` из-за расхождения branded-типов с `@ydbjs/*`.
- Все запросы параметризованы (`query.parameter(...)`) — значения никогда не конкатенируются в SQL.
