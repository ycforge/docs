# Адаптеры СУБД

Ядро ORM не привязано к конкретной базе данных: persistence, relations, QueryBuilder, миграции и транзакции опираются на интерфейсы executor'а (`YdbExecutor`/`YdbQuery`), а вся специфика СУБД — драйвер, маппинг значений, классификация ошибок, retry-политика, DDL и чтение схемы — инкапсулирована в **адаптере**, реализующем интерфейс `OrmAdapter`.

По умолчанию используется встроенный **YDB-адаптер** (`ydbAdapter`), поэтому для работы с YDB ничего настраивать не нужно.

## Интерфейс OrmAdapter

```ts
import type { OrmAdapter } from '@ycforge/ydb-orm';
```

Поля адаптера:

| Поле | Назначение |
|------|-----------|
| `name` | Имя адаптера (например, `'ydb'`) — для диагностики и логов |
| `validateModuleOptions(opts, injected?)` | Fail-fast валидация опций модуля до создания драйвера |
| `resolveCredentialsProvider(opts, injected?)` | Разрешение `CredentialsProvider` по приоритету источников |
| `createDriver(opts, credentialsProvider?)` | Создание подключённого драйвера |
| `createExecutor(driver, opts)` | Создание executor'а (`YdbExecutor`) поверх драйвера |
| `mapValue(type, value, field?)` | Преобразование JS-значения в значение СУБД (аналог `mapToYdb`) |
| `classifyError(error)` | Структурная классификация ошибки: `'transient'` / `'fatal'` |
| `isTransientError(error)` | Признак транзитной (повторяемой) ошибки |
| `resolveRetryPolicy(input)` | Разрешение retry-политики (`false`/`true`/объект → конфигурация или `null`) |
| `withRetryPolicy(executor, policyInput?)` | Обёртка executor'а retry-политикой |
| `createSchemaSyncer(driver, executor)` | Синхронизатор схемы (DDL + описание таблиц) |

## Подключение адаптера

Адаптер выбирается через опциональное поле `adapter` в `YdbModuleOptions`. Без него используется `ydbAdapter`:

```ts
import { createAuth } from '@ycforge/auth';
import { YdbCoreModule } from '@ycforge/ydb-orm';
import { ydbAdapter } from '@ycforge/ydb-orm/ydb';

@Module({
  imports: [
    YdbCoreModule.forRootAsync({
      useFactory: () => ({
        endpoint: 'grpcs://ydb.serverless.yandexcloud.net:2135/ru-central1/...',
        auth: createAuth(...),
        adapter: ydbAdapter, // необязательно: это и есть значение по умолчанию
      }),
    }),
  ],
})
export class AppModule {}
```

Свой адаптер (например, обёртка с метриками или stub для тестов) передаётся тем же полем `adapter`.

## Subpath-экспорт `@ycforge/ydb-orm/ydb`

Реализация YDB-адаптера доступна отдельным subpath-импортом:

```ts
import { ydbAdapter, createDriver, createExecutor, mapToYdb } from '@ycforge/ydb-orm/ydb';
```

Отсюда же экспортируются `YdbSchemaSyncer`, DDL-генераторы (`generateCreateTableYql` и др.), retry-утилиты (`runWithRetry`, `withRetryPolicy`, `resolveYdbRetryPolicy`) и связанные типы. Всё это доступно и из корня пакета — subpath нужен, чтобы явно работать с YDB-адаптером.

{% note info %}

Публичные имена API (`YdbEntity`, `YdbRepository`, `mapToYdb` и т.д.) не зависят от адаптера и не переименовывались. Внутренние пути (`core/driver.js`, `core/mapper.js`, `core/retry.js`, `schema/schema-sync.js`) сохранены как реэкспорты-шимы для обратной совместимости.

{% endnote %}

## Другие СУБД

Адаптеры под другие базы данных (PostgreSQL и т.п.) появятся в следующих версиях — интерфейс `OrmAdapter` уже отделяет их будущую реализацию от ядра ORM.
