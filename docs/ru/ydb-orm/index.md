# ydb-orm

**@ycforge/ydb-orm** — TypeORM-like ORM для [YDB (Yandex Database)](https://ydb.tech/) на TypeScript.

Библиотека предоставляет Active Record, отношения (relations), шифрование полей с blind index, schema sync, транзакции и NestJS-адаптер.

{% note info %}

Документация на английском доступна по переключателю языков в шапке сайта.

{% endnote %}

## Возможности

- **Active Record** — сущности наследуют `YdbBaseEntity` и получают статические методы `find`, `findAll`, `count`, `save`, `insertMany`, `delete` и другие.
- **Декораторы** — `@YdbEntity`, `@YdbColumn`, `@YdbPrimaryColumn`, `@YdbEncrypted`, relations, `@EagerLoad`, `@YdbIndex`, `@YdbEnum`, `@YdbTtl`, lifecycle-хуки.
- **Шифрование полей** — провайдеры шифрования с поддержкой blind index для поиска по зашифрованным значениям и AAD-привязкой к связанным полям.
- **Схема БД** — `schema sync` (аналог `synchronize` в TypeORM) и генераторы DDL для миграций.
- **Миграции** — классы с `up`/`down`, CLI-команды `migration:create|generate|run|revert|show`.
- **Транзакции** — `YdbTransactionManager.runInTransaction()`.
- **NestJS-адаптер** — `YdbCoreModule.forRootAsync()` + `YdbOrmModule.forFeature()` с DI-инжекцией (см. [NestJS-документацию](nest/overview.md)).
- **Использование без NestJS** — `createDriver` / `createExecutor` / `configureEntities` для скриптов, Lambda и CLI.

## Принципы

- **Удобство** — API в стиле TypeORM: если вы знаете TypeORM, ydb-orm покажется знакомым.
- **Минимализм** — оптимизация по памяти и CPU.
- **Функциональность** — покрытие типовых сценариев работы с YDB из коробки.

## Технологии

- Runtime: **Node.js ≥ 22**, ESM (`"type": "module"`).
- Драйвер: `@ydbjs/*` (новое поколение SDK YDB).
- Опционально: интеграция с NestJS через `@ycforge/ydb-orm/nest` (peer-зависимости: `@nestjs/common`, `@nestjs/core`).

## Репозиторий

- GitHub: [github.com/ycforge/ydb-orm](https://github.com/ycforge/ydb-orm)
- npm: [npmjs.com/package/@ycforge/ydb-orm](https://www.npmjs.com/package/@ycforge/ydb-orm)
- Лицензия: MIT

## С чего начать

1. [Установка](installation.md)
2. [Быстрый старт](quick-start.md) (standalone)
3. [Быстрый старт (NestJS)](nest/quick-start.md) (если вы используете NestJS)
4. [Сущности и декораторы](entity.md)
5. [Active Record](active-record.md)