# NestJS-адаптер

`@ycforge/ydb-orm` поставляет NestJS-адаптер через подпакет `@ycforge/ydb-orm/nest`. Он предоставляет DI-интеграцию: регистрацию модулей, инжектируемые репозитории, менеджер транзакций и провайдер schema sync.

## Установка peer-зависимостей

```bash
yarn add @nestjs/common @nestjs/core reflect-metadata rxjs
```

## Структура модулей

| Модуль | Назначение |
| --- | --- |
| `YdbCoreModule.forRootAsync(...)` | корневой модуль: драйвер, executor, auth, провайдеры шифрования |
| `YdbOrmModule.forFeature([...entities])` | feature-модуль: регистрация инжектируемых репозиториев |

## DI-токены

| Токен | Что предоставляет |
| --- | --- |
| `YDB_DRIVER` | драйвер YDB |
| `YDB_QUERY` | executor запросов |
| `YDB_OPTIONS` | опции модуля |
| `YDB_CREDENTIALS_PROVIDER` | credentials provider |
| `YDB_ENCRYPTION_PROVIDER` | провайдер шифрования |
| `YDB_BLIND_INDEX_PROVIDER` | провайдер blind index |
| `YDB_SCHEMA_SYNC` | schema syncer |

## Что меняется по сравнению со standalone

| Аспект | Standalone | NestJS |
| --- | --- | --- |
| Подключение | `createDriver` + `createExecutor` | `YdbCoreModule.forRootAsync(...)` |
| Настройка сущностей | `configureEntities([Entity], { executor })` | `YdbOrmModule.forFeature([Entity])` |
| Репозиторий | `getOrCreateRepository(Entity)` | `@InjectRepository(Entity)` |
| Транзакции | `new YdbTransactionManager(executor)` | инжектируемый `YdbTransactionManager` |
| Schema sync | `new YdbSchemaSyncer(executor)` | `@Inject(YDB_SCHEMA_SYNC)` |

## Пример: минимальное NestJS-приложение

```ts
import { Module } from '@nestjs/common';
import { YdbCoreModule, YdbOrmModule } from '@ycforge/ydb-orm/nest';
import { createAuth, authKeyFromFile } from '@ycforge/auth';
import { UserEntity } from './user.entity';

@Module({
  imports: [
    YdbCoreModule.forRootAsync({
      useFactory: () => ({
        endpoint: 'grpcs://ydb.serverless.yandexcloud.net:2135/?database=/ru-central1/.../...',
        auth: createAuth(authKeyFromFile('./authorized_key.json')),
        sync: true,
      }),
    }),
    YdbOrmModule.forFeature([UserEntity]),
  ],
})
export class AppModule {}
```

## Следующие шаги

- [Быстрый старт (NestJS)](quick-start.md) — пошаговое руководство для NestJS
- [Инъекция репозитория (NestJS)](repository.md) — паттерн DI-репозитория
- [Транзакции (NestJS)](transactions.md) — инжектируемый менеджер транзакций
- [Schema sync (NestJS)](schema-sync.md) — DI-инъекция schema syncer
- [Шифрование (NestJS)](encryption.md) — передача провайдеров через опции модуля
- [Аутентификация (NestJS)](authentication.md) — auth через опции модуля
