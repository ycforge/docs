# ydb-orm

**@ycforge/ydb-orm** is a TypeORM-like ORM for [YDB (Yandex Database)](https://ydb.tech/) written in TypeScript.

The library provides Active Record, relations, field encryption with blind index, schema sync, transactions, and NestJS integration.

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
- **NestJS integration** — `YdbCoreModule.forRootAsync()` + `YdbOrmModule.forFeature()`.
- **Standalone usage** — `createDriver` / `createExecutor` / `configureEntities` for scripts and CLIs.

## Principles

- **Convenience** — TypeORM-like API: if you know TypeORM, ydb-orm will feel familiar.
- **Minimalism** — optimized for memory and CPU.
- **Functionality** — typical YDB scenarios are covered out of the box.

## Technologies

- Runtime: **Node.js ≥ 22**, ESM (`"type": "module"`).
- Driver: `@ydbjs/*` (the new-generation YDB SDK).
- Integration: NestJS (`@nestjs/common`, `@nestjs/core` — peer dependencies).

## Repository

- GitHub: [github.com/ycforge/ydb-orm](https://github.com/ycforge/ydb-orm)
- npm: [npmjs.com/package/@ycforge/ydb-orm](https://www.npmjs.com/package/@ycforge/ydb-orm)
- License: MIT

## Getting started

1. [Installation](installation.md)
2. [Quick start](quick-start.md)
3. [Entities & decorators](entity.md)
4. [Active Record](active-record.md)