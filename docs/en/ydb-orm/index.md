# ydb-orm

**@ycforge/ydb-orm** is a TypeORM-like ORM for [YDB (Yandex Database)](https://ydb.tech/) written in TypeScript.

The library provides Active Record, relations, field encryption with blind index, schema sync, transactions, and a NestJS adapter.

{% note info %}

Русская версия документации доступна через переключатель языков в шапке сайта.

{% endnote %}

## Features

- **Active Record** — entities extend `YdbBaseEntity` and get static methods `find`, `findAll`, `count`, `save`, `insertMany`, `delete`, and more.
- **Decorators** — `@YdbEntity`, `@YdbColumn`, `@YdbPrimaryColumn`, `@YdbEncrypted`, relations, `@EagerLoad`, `@YdbIndex`, `@YdbEnum`, `@YdbTtl`, lifecycle hooks.
- **Field encryption** — encryption providers with blind index support for searching encrypted values and AAD binding to related fields.
- **Database schema** — `schema sync` (like `synchronize` in TypeORM) and DDL generators for migrations.
- **Migrations** — classes with `up`/`down`, CLI commands `migration:create|generate|run|revert|show`.
- **Transactions** — `YdbTransactionManager.runInTransaction()`.
- **NestJS adapter** — `YdbCoreModule.forRootAsync()` + `YdbOrmModule.forFeature()` with DI injection (see [NestJS docs](nest/overview.md)).
- **Standalone usage** — `createDriver` / `createExecutor` / `configureEntities` for scripts, Lambdas, and CLIs.

## Principles

- **Convenience** — TypeORM-like API: if you know TypeORM, ydb-orm will feel familiar.
- **Minimalism** — optimized for memory and CPU.
- **Functionality** — typical YDB scenarios are covered out of the box.

## Technologies

- Runtime: **Node.js >= 22**, ESM (`"type": "module"`).
- Driver: `@ydbjs/*` (the new-generation YDB SDK).
- Optional: NestJS integration via `@ycforge/ydb-orm/nest` (peer dependencies: `@nestjs/common`, `@nestjs/core`).

## Repository

- GitHub: [github.com/ycforge/ydb-orm](https://github.com/ycforge/ydb-orm)
- npm: [npmjs.com/package/@ycforge/ydb-orm](https://www.npmjs.com/package/@ycforge/ydb-orm)
- License: MIT

## Getting started

1. [Installation](installation.md)
2. [Quick start](quick-start.md) (standalone)
3. [NestJS quick start](nest/quick-start.md) (if you use NestJS)
4. [Entities & decorators](entity.md)
5. [Active Record](active-record.md)