# Миграции

Миграции — способ управлять схемой БД в продакшене. По аналогии с TypeORM: миграция — это класс с методами `up`/`down`, получающими `YdbExecutor`.

## Структура миграции

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

Применённые миграции хранятся в таблице `ydb_migrations` (создаётся автоматически). Порядок выполнения — по имени файла (`<timestamp>-<Name>.ts`).

{% note warning %}

Node ≥ 22.18 импортирует `.ts` напрямую, отдельный ts-node не нужен. Из-за нативного стриппинга типов типы (`YdbMigration`, `YdbExecutor`) импортируйте через `import type` — обычный именованный импорт типа упадёт в рантайме.

{% endnote %}

## CLI

Пакет устанавливает бинарь `ydb-orm`:

```bash
ydb-orm migration:create CreateUsers      # пустая миграция ./migrations/<ts>-CreateUsers.ts
ydb-orm migration:generate AddPhotos      # миграция по diff сущностей и БД
ydb-orm migration:run                     # применить все новые миграции
ydb-orm migration:revert                  # откатить последнюю
ydb-orm migration:show                    # статус миграций
ydb-orm migration:check                   # проверить, что все миграции применены
ydb-orm schema:verify                     # проверить схему БД против метаданных сущностей
ydb-orm entity:create UserProfile         # сущность ./src/user-profile.entity.ts
```

Опции:

- `--dir <path>` — директория миграций (по умолчанию `./migrations`; для `entity:create` — `./src`).
- `--config <path>` — путь к конфигу.
- `--json` — machine-readable вывод для `migration:show` и `migration:check`.

## Конфиг CLI

Файл `./ydb-orm.config.ts` (или `.mts`/`.mjs`/`.js`; также ищется в `./src/`):

```ts
import { UserEntity } from './src/user.entity.js';

export default {
  endpoint: process.env.YDB_ENDPOINT!,
  auth_type: 'auth_key',
  authOptions: { authorized_key_path: './authorized_key.json' },
  entities: [UserEntity],        // нужно для migration:generate
  migrationsDir: './migrations', // опционально
};
```

Без конфига CLI читает env: `YDB_ENDPOINT` (или `YDB_CONNECTION_STRING`), `YDB_AUTH_TYPE` (по умолчанию `anonymous`), `YDB_AUTHORIZED_KEY_PATH`.

## migration:generate

Команда строит diff по всем `entities` из конфига:

- нет таблицы → `CREATE TABLE` (+ `DROP TABLE` в `down`);
- нет колонок → `ADD COLUMN` (+ `DROP COLUMN` в `down`);
- расхождение типа/PK и лишние колонки не меняются автоматически — попадают в миграцию как `WARNING`-комментарии.

```bash
ydb-orm migration:generate AddPhotos
```

## Программный API

Можно встроить миграции в свой пайплайн:

```ts
import {
  YdbMigrationRunner,
  loadMigrationsFromDir,
  planMigration,
  executeSql,
} from '@ycforge/ydb-orm';

const migrations = await loadMigrationsFromDir('./migrations');
const runner = new YdbMigrationRunner(executor);

await runner.run();       // применить новые
await runner.revert();    // откатить последнюю
const status = await runner.status(); // статус
```

`planMigration(expected, existing)` — чистая функция, строит план миграции по diff между ожидаемыми схемами сущностей и текущим состоянием БД.