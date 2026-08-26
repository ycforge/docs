# Migrations

Migrations are a way to manage the database schema in production. Like in TypeORM, a migration is a class with `up`/`down` methods receiving a `YdbExecutor`.

## Migration structure

```ts
import type { YdbMigration, YdbExecutor } from '@ycforge/ydb-orm';
import { executeSql } from '@ycforge/ydb-orm';

export class CreateUsers1755000000000 implements YdbMigration {
  readonly name = '1755000000000-CreateUsers';

  async up(executor: YdbExecutor): Promise<void> {
    await executeSql(executor, 'CREATE TABLE `users` (`uuid` Uuid, PRIMARY KEY (`uuid`))');
  }

  async down(executor: YdbExecutor): Promise<void> {
    await executeSql(executor, 'DROP TABLE `users`');
  }
}
```

Applied migrations are tracked in the `ydb_migrations` table (created automatically). Execution order is by file name (`<timestamp>-<Name>.ts`).

{% note warning %}

Node ≥ 22.18 imports `.ts` directly; no separate ts-node is needed. Because of native type stripping, import types (`YdbMigration`, `YdbExecutor`) via `import type` — a regular named type import fails at runtime.

{% endnote %}

## Reliability guarantees

- **Stable identity**: each migration gets a SHA-256 of its file content (`migration.hash`). Matching against `ydb_migrations` goes by hash — renaming a file does not re-apply it; changing content of an applied migration is an error.
- **Partial application**: DDL in YDB is not transactional, so a `state='started'` marker is written before `up()`/`down()` and replaced with `'applied'` only after success. A crash mid-migration leaves the marker: repeated `run()` will not blindly restart it, and `revert()` refuses to roll it back until resolved explicitly — `runner.markMigrationApplied(name)` or `runner.removeMigrationRecord(name)` (CLI: `ydb-orm migration:repair <name> --as-applied|--as-reverted`).
- **Parallel runs**: the claim to apply is an INSERT with an id derived from the hash — two processes starting the same migration collide on PRIMARY KEY before `up()` executes.

## Commands

All migration commands (`migration:create|generate|run|revert|show/status|check|repair`), schema verification (`schema:verify`) and code generation (`entity:create`, `metadata:dump`, `entity:diagram`, `completion`) live in the [CLI reference](cli.md).

```bash
ydb-orm migration:create CreateUsers      # empty migration
ydb-orm migration:generate AddPhotos      # migration by entity↔DB diff
ydb-orm migration:run                     # apply all new migrations
ydb-orm migration:revert                  # revert the last one
ydb-orm migration:check                   # CI readiness check
```

## Programmatic API

You can embed migrations into your own pipeline:

```ts
import {
  YdbMigrationRunner,
  loadMigrationsFromDir,
  planMigration,
  executeSql,
} from '@ycforge/ydb-orm';

const migrations = await loadMigrationsFromDir('./migrations');
const runner = new YdbMigrationRunner(executor);

await runner.run();       // apply new
await runner.revert();    // revert the last one
const status = await runner.status(); // status
```

`planMigration(expected, existing)` is a pure function that builds a migration plan from the diff between expected entity schemas and the current database state.
