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
ydb-orm migration:show                    # статус миграций (алиас — migration:status)
ydb-orm migration:check                   # проверка готовности для CI (exit != 0, если не готово)
ydb-orm schema:verify                     # проверить схему БД против метаданных сущностей
ydb-orm entity:create UserProfile         # сущность ./src/user-profile.entity.ts
ydb-orm completion bash                   # скрипт shell-автодополнения (bash|zsh|fish)
```

Опции:

- `--dir <path>` — директория миграций (по умолчанию `./migrations`; для `entity:create` — `./src`).
- `--config <path>` — путь к конфигу.
- `--json` — машинночитаемый вывод для `migration:show`, `migration:status` и `migration:check`.

### Проверка готовности

`migration:check`, `migration:status` и `migration:show` — **read-only**: команды только читают состояние (`DescribeTable` для `ydb_migrations` + голый `SELECT` записей; для сущностей — `DescribeTable`) и ничего не меняют — в частности, таблица учёта не создаётся и не изменяется (никакого `CREATE TABLE`/`ALTER TABLE`). Состояния и exit-коды:

| Состояние | Exit-код | Значение |
| --- | --- | --- |
| готово | 0 | все миграции применены; схема совпадает, если проверялась |
| pending | 1 | есть неприменённые миграции |
| interrupted | 2 | есть прерванные миграции (`state='started'`): прошлый запуск оборвался посреди миграции, БД может быть частично изменена |
| schema-drift | 3 | схема БД расходится с метаданными сущностей (проверяется, только если в конфиге задан массив `entities`) |
| modified | 4 | содержимое применённой миграции изменилось после применения |
| ошибка команды | 5 | не удалось подключиться, прочитать состояние или неожиданный сбой |

Если таблица учёта `ydb_migrations` ещё не существует (свежая база), она не создаётся: считается, что не применено ничего — при наличии файлов миграций это pending (exit 1), без них — готово (exit 0). В `--json` такое состояние различается полем `bookkeeping: {exists: false}`; легаси-таблицы без колонок `hash`/`state` читаются как есть, без ALTER.

Прерванные и изменённые миграции явно не считаются успешно применёнными; orphan-записи (файл удалён после применения) выводятся в отчёте, но сами по себе готовность не ломают. При нескольких состояниях exit-код выбирается по приоритету: `interrupted` → `modified` → `pending` → `schema-drift`. Разрешить прерванную миграцию можно командой `migration:repair`.

Текстовый режим: сводка или список миграций — в stdout, проблемы и diff схемы — в stderr, итоговая строка начинается с `Up to date:` либо `Not ready:`. Цвет diff определяется по реальному потоку вывода и отключается вне TTY и через `NO_COLOR`. Для CI-парсинга используйте `--json`: весь отчёт приходит в stdout со стабильной схемой — `ready`, `state`, `states`, `exitCode`, списки `pending`/`interrupted`/`modified`/`orphaned`, массив `migrations` с флагами каждой миграции и блок `schema` с найденными расхождениями.

`migration:generate` и `schema:verify` выводят цветной diff расхождений «сущности vs БД», сгруппированный по таблицам. Цвета отключаются при выводе не в TTY или через переменную `NO_COLOR`.

## Shell-автодополнение

```bash
# bash
ydb-orm completion bash | sudo tee /etc/bash_completion.d/ydb-orm
# zsh
ydb-orm completion zsh > ~/.zsh/completions/_ydb-orm
# fish
ydb-orm completion fish > ~/.config/fish/completions/ydb-orm.fish
```

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

### Подсказки о переименовании

Если расхождение выглядит как переименование колонки — ровно одна лишняя колонка в БД и ровно одна новая в сущности с тем же типом, без участия PK, индексов, TTL и blind-index (`_bi`) — команда не делает молча `ADD COLUMN` + `DROP COLUMN`. Вместо этого для такой пары ADD/DROP подавляются, а в `up()`/`down()` добавляется комментарий-подсказка:

```ts
// SUGGESTION (not applied automatically): possible column rename detected.
// YQL has no ALTER TABLE RENAME COLUMN yet — verify the data and migrate manually:
//   ALTER TABLE `photos` RENAME COLUMN `label` TO `title`;
```

Подсказка никогда не исполняется автоматически: YQL пока не поддерживает `RENAME COLUMN` (в YDB переименовываются только таблицы), применение — всегда ручное решение после проверки данных. При неоднозначности (несколько кандидатов, ключевые/индексированные/TTL-колонки, метаданные шифрования) поведение прежнее: `ADD COLUMN` + `WARNING`. Та же подсказка видна в цветном diff команды `schema:verify`.

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