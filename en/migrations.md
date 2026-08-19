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

## CLI

The package installs the `ydb-orm` binary:

```bash
ydb-orm migration:create CreateUsers      # empty migration ./migrations/<ts>-CreateUsers.ts
ydb-orm migration:generate AddPhotos      # migration by entity↔DB diff
ydb-orm migration:run                     # apply all new migrations
ydb-orm migration:revert                  # revert the last one
ydb-orm migration:show                    # migration status
ydb-orm entity:create UserProfile         # entity ./src/user-profile.entity.ts
```

Options: `--dir <path>` (migrations directory, default `./migrations`; for `entity:create` — `./src`), `--config <path>`.

## CLI config

File `./ydb-orm.config.ts` (or `.mts`/`.mjs`/`.js`; also searched in `./src/`):

```ts
import { UserEntity } from './src/user.entity.js';

export default {
  endpoint: process.env.YDB_ENDPOINT!,
  auth_type: 'auth_key',
  authOptions: { authorized_key_path: './authorized_key.json' },
  entities: [UserEntity],        // required for migration:generate
  migrationsDir: './migrations', // optional
};
```

Without a config, the CLI reads env: `YDB_ENDPOINT` (or `YDB_CONNECTION_STRING`), `YDB_AUTH_TYPE` (default `anonymous`), `YDB_AUTHORIZED_KEY_PATH`.

## migration:generate

The command builds a diff across all `entities` from the config:

- no table → `CREATE TABLE` (+ `DROP TABLE` in `down`);
- no columns → `ADD COLUMN` (+ `DROP COLUMN` in `down`);
- type/PK mismatches and extra columns are not changed automatically — they appear in the migration as `WARNING` comments.

```bash
ydb-orm migration:generate AddPhotos
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