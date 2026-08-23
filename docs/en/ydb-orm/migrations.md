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
ydb-orm migration:show                    # migration status (alias — migration:status)
ydb-orm migration:check                   # CI readiness check (non-zero exit when not ready)
ydb-orm schema:verify                     # verify DB schema against entity metadata
ydb-orm entity:create UserProfile         # entity ./src/user-profile.entity.ts
ydb-orm completion bash                   # shell completion script (bash|zsh|fish)
```

Options:

- `--dir <path>` — migrations directory (default `./migrations`; for `entity:create` — `./src`).
- `--config <path>` — path to config.
- `--json` — machine-readable output for `migration:show`, `migration:status` and `migration:check`.

### Readiness check

`migration:check`, `migration:status` and `migration:show` are **read-only**: they only inspect the current state (`SELECT` from `ydb_migrations` + `DescribeTable` for entities) and never change anything. States and exit codes:

| State | Exit code | Meaning |
| --- | --- | --- |
| ready | 0 | all migrations applied; schema matches if it was checked |
| pending | 1 | there are unapplied migrations |
| interrupted | 2 | there are interrupted migrations (`state='started'`): a previous run stopped mid-way, the database may be partially migrated |
| schema-drift | 3 | the database schema differs from entity metadata (checked only when the `entities` array is set in the config) |
| modified | 4 | content of an applied migration changed after it was applied |
| command error | 5 | failed to connect, read state, or an unexpected failure |

Interrupted and modified migrations are explicitly not treated as successfully applied; orphan records (the file was deleted after applying) are shown in the report but do not break readiness on their own. When several states are detected at once, the exit code follows the priority: `interrupted` → `modified` → `pending` → `schema-drift`. Resolve an interrupted migration with `migration:repair`.

Text mode: summary or migration list goes to stdout, problems and the schema diff go to stderr; the final line starts with `Up to date:` or `Not ready:`. Diff colors follow the real output stream and are disabled outside a TTY and by `NO_COLOR`. For CI parsing use `--json`: the whole report is printed to stdout with a stable schema — `ready`, `state`, `states`, `exitCode`, the `pending`/`interrupted`/`modified`/`orphaned` lists, the `migrations` array with per-migration flags, and the `schema` block with found discrepancies.

`migration:generate` and `schema:verify` print a colored diff of "entities vs database" discrepancies, grouped by table. Colors are disabled when output is not a TTY or via the `NO_COLOR` environment variable.

## Shell completion

```bash
# bash
ydb-orm completion bash | sudo tee /etc/bash_completion.d/ydb-orm
# zsh
ydb-orm completion zsh > ~/.zsh/completions/_ydb-orm
# fish
ydb-orm completion fish > ~/.config/fish/completions/ydb-orm.fish
```

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

### Rename hints

When a diff looks like a column rename — exactly one extra DB column and exactly one new entity column with the same type, with no PK, index, TTL or blind-index (`_bi`) involvement — the command does not silently emit `ADD COLUMN` + `DROP COLUMN`. Instead, ADD/DROP are suppressed for that pair and a comment hint is added to `up()`/`down()`:

```ts
// SUGGESTION (not applied automatically): possible column rename detected.
// YQL has no ALTER TABLE RENAME COLUMN yet — verify the data and migrate manually:
//   ALTER TABLE `photos` RENAME COLUMN `label` TO `title`;
```

The hint is never executed automatically: YQL does not support `RENAME COLUMN` yet (only tables can be renamed in YDB), so applying it is always a manual decision after verifying the data. If the situation is ambiguous (multiple candidates, key/indexed/TTL columns, encryption metadata), the behavior stays as before: `ADD COLUMN` + `WARNING`. The same hint appears in the colored diff of `schema:verify`.

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