# CLI

The package installs the `ydb-orm` binary: migrations, schema verification and code generation from the command line.

## Commands

```bash
ydb-orm migration:create CreateUsers      # empty migration ./migrations/<ts>-CreateUsers.ts
ydb-orm migration:generate AddPhotos      # migration by entity↔DB diff
ydb-orm migration:run                     # apply all new migrations
ydb-orm migration:revert                  # revert the last one
ydb-orm migration:show                    # migration status (alias — migration:status)
ydb-orm migration:check                   # CI readiness check (non-zero exit when not ready)
ydb-orm migration:repair <name> --as-applied|--as-reverted   # resolve an interrupted migration
ydb-orm schema:verify                     # verify DB schema against entity metadata
ydb-orm entity:create UserProfile         # entity ./src/user-profile.entity.ts
ydb-orm metadata:dump                     # entity metadata as JSON (stdout, no DB)
ydb-orm entity:diagram                    # Mermaid ER diagram from metadata (stdout/--output, no DB)
ydb-orm completion bash                   # shell completion script (bash|zsh|fish)
```

Global options:

- `--dir <path>` — migrations directory (default `./migrations`; for `entity:create` — `./src`).
- `--config <path>` — path to config.
- `--output <file>` — output file for `entity:diagram` (existing files are never overwritten).
- `--json` — machine-readable output for `migration:show`, `migration:status`, `migration:check`.
- `--verbose` — full error stack and cause chain on failure.

Unknown flags and empty option values are treated as errors.

## Config file

File `./ydb-orm.config.ts` (or `.mts`/`.mjs`/`.js`; also searched in `./src/` and upward to the FS root; both default and named exports are supported):

```ts
import { createAuth, authKeyFromFile } from '@ycforge/auth';
import { UserEntity } from './src/user.entity.js';

export default {
  endpoint: process.env.YDB_ENDPOINT!,
  auth: createAuth(authKeyFromFile('./authorized_key.json')),
  entities: [UserEntity],        // required for migration:generate
  migrationsDir: './migrations', // optional
};
```

Without a config the CLI reads env `YDB_ENDPOINT` (or `YDB_CONNECTION_STRING`), but setting up `auth` still requires a config file.

## Migration commands

### migration:create / migration:generate

`migration:create` writes an empty `<timestamp>-<Name>.ts` file. `migration:generate` builds a diff across all `entities` from the config:

- no table → `CREATE TABLE` (+ `DROP TABLE` in `down`);
- no columns → `ADD COLUMN` (+ `DROP COLUMN` in `down`);
- type/PK mismatches and extra columns are not changed automatically — they appear in the migration as `WARNING` comments.

**Rename hints**: when a diff looks like a column rename — exactly one extra DB column and exactly one new entity column with the same type, with no PK/index/TTL/blind-index involvement — ADD/DROP are suppressed for the pair and a comment hint is added:

```ts
// SUGGESTION (not applied automatically): possible column rename detected.
// YQL has no ALTER TABLE RENAME COLUMN yet — verify the data and migrate manually:
//   ALTER TABLE `photos` RENAME COLUMN `label` TO `title`;
```

The hint is never executed automatically; if ambiguous, behavior stays `ADD COLUMN` + `WARNING`.

Both commands print a colored diff of discrepancies grouped by table. Colors follow the output stream and are disabled outside a TTY or by `NO_COLOR`.

### migration:run / migration:revert / migration:repair

Execution order is by file name (`<timestamp>-<Name>.ts`). Applied migrations are tracked in the `ydb_migrations` table (created automatically). Identity is SHA-256 of the file content: renaming a file does not re-apply it; changing an applied migration is an error.

DDL in YDB is not transactional, so before each `up()`/`down()` a `state='started'` marker is written and replaced with `'applied'` only after success. If a run dies mid-migration:

- repeated `migration:run` will not blindly restart it;
- `migration:revert` refuses to roll it back;
- resolve explicitly: `ydb-orm migration:repair <name> --as-applied` (changes kept manually) or `--as-reverted` (changes rolled back manually).

Parallel runs collide on a deterministic INSERT claim (PRIMARY KEY) — double application is impossible across processes.

### Readiness check: migration:check / show / status

Read-only workflow: commands inspect state (`DescribeTable` for `ydb_migrations` + bare `SELECT`; `DescribeTable` for entities) and never change anything — in particular the bookkeeping table is neither created nor altered.

States and exit codes:

| State | Exit code | Meaning |
| --- | --- | --- |
| ready | 0 | all migrations applied; schema matches if it was checked |
| pending | 1 | there are unapplied migrations |
| interrupted | 2 | there are interrupted migrations (`state='started'`) |
| schema-drift | 3 | DB schema differs from entity metadata (checked when `entities` set in config) |
| modified | 4 | content of an applied migration changed after it was applied |
| command error | 5 | failed to connect, read state, or unexpected failure |

If the bookkeeping table does not exist yet (fresh database), nothing is considered applied: with migration files present that is pending (exit 1), without them ready (exit 0); in `--json` distinguished by `bookkeeping: {exists: false}`.

Multiple states at once → priority: `interrupted` → `modified` → `pending` → `schema-drift`. Orphan records (file deleted after applying) are shown but do not break readiness by themselves.

Text mode: summary/list → stdout, problems and schema diff → stderr, final line starts with `Up to date:` or `Not ready:`. For CI parsing use `--json`: the whole report goes to stdout with a stable schema (`ready`, `state`, `states`, `exitCode`, `pending`/`interrupted`/`modified`/`orphaned`, per-migration flags in `migrations`, and the `schema` block).

## Code generation

### entity:create

In a TTY, starts an interactive wizard: table name → columns loop (name → YDB type → PK → encrypted/blind index → enum values + storage) → auto timestamps for date-like columns → optional TTL (ISO 8601 duration) → preview and write confirmation.

Guarantees:

- every definition is validated **before** any file is written;
- an existing file is never overwritten (collision fails before the first question);
- Ctrl+C / EOF exit cleanly (code 130) without writing anything;
- no DB access and no DDL — purely local file generation;
- outside a TTY input is never read: a deterministic default template (`uuid` PK + `name`) is generated and the command does not hang.

Programmatic API: `createEntityFileFromSpec(dir, spec)` (+ `validateEntitySpec`, `renderEntityFile`, `buildDefaultEntitySpec`, `runEntityCreateCommand`).

### metadata:dump

Dumps canonical metadata of the entities from the config as deterministic JSON — **without connecting to the database**. Per entity: class/table names, columns with YDB types (including synthetic `{field}_bi`), PK order, relations of all types (+ physical `joinTables`), indexes, TTL, declarative encryption flags (no secrets ever), enums, JSON columns, eager relations. Stable ordering → byte-for-byte identical output across runs. Programmatic API: `buildMetadataDump(entities)`.

### entity:diagram

Renders the same canonical metadata into a Mermaid ER diagram — no DB connection:

```bash
ydb-orm entity:diagram                            # Mermaid text to stdout
ydb-orm entity:diagram --output docs/schema.mmd   # to a file (overwriting forbidden)
```

PK columns come first in declaration order marked with `PK`; FK columns are marked `FK`; many-to-many is drawn via the physical join table; block/line ordering is deterministic. Programmatic API: `buildEntityDiagram(entities)`, `writeDiagramFile(path, diagram)`.

## Shell completion

```bash
# bash
ydb-orm completion bash | sudo tee /etc/bash_completion.d/ydb-orm
# zsh
ydb-orm completion zsh > ~/.zsh/completions/_ydb-orm
# fish
ydb-orm completion fish > ~/.config/fish/completions/ydb-orm.fish
```
