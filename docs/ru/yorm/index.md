# yorm

**@ycforge/yorm** — TypeORM-like ORM для [YDB (Yandex Database)](https://ydb.tech/) на TypeScript.

Библиотека предоставляет Active Record, отношения (relations), шифрование полей с blind index, schema sync, транзакции и интеграцию с NestJS.

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
- **NestJS-интеграция** — `YdbCoreModule.forRootAsync()` + `YdbModule.forFeature()`.
- **Использование без NestJS** — `createDriver` / `createExecutor` / `configureEntities` для скриптов и CLI.

## Принципы

- **Удобство** — API в стиле TypeORM: если вы знаете TypeORM, yorm покажется знакомым.
- **Минимализм** — оптимизация по памяти и CPU.
- **Функциональность** — покрытие типовых сценариев работы с YDB из коробки.

## Технологии

- Runtime: **Node.js ≥ 22**, ESM (`"type": "module"`).
- Драйвер: `@ydbjs/*` (новое поколение SDK YDB).
- Интеграция: NestJS (`@nestjs/common`, `@nestjs/core` — peer-зависимости).

## Репозиторий

- GitHub: [github.com/ycforge/yorm](https://github.com/ycforge/yorm)
- npm: [npmjs.com/package/@ycforge/yorm](https://www.npmjs.com/package/@ycforge/yorm)
- Лицензия: MIT

## С чего начать

1. [Установка](installation.md)
2. [Быстрый старт](quick-start.md)
3. [Сущности и декораторы](entity.md)
4. [Active Record](active-record.md)