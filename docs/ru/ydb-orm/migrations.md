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
ydb-orm metadata:dump                     # метаданные сущностей в JSON (stdout, без БД)
ydb-orm completion bash                   # скрипт shell-автодополнения (bash|zsh|fish)
```

Опции:

- `--dir <path>` — директория миграций (по умолчанию `./migrations`; для `entity:create` — `./src`).
- `--config <path>` — путь к конфигу.
- `--json` — машинночитаемый вывод для `migration:show`, `migration:status` и `migration:check`.

### Интерактивный entity:create

В TTY команда `ydb-orm entity:create <name>` запускает мастер генерации сущности:

1. имя таблицы (по умолчанию — snake_case от имени сущности, валидируется);
2. колонки в цикле «имя → тип YDB → PK → encrypted/blind index → enum (значения + хранилище Utf8/Int32)»;
3. для date-like колонок (`Date`/`Datetime`/`Timestamp`) — автопростановка времени создания/обновления (`@YdbCreateDateColumn`/`@YdbUpdateDateColumn`);
4. опциональный TTL (`@YdbTtl`, ISO 8601 duration, например `PT2H`) по выбранной date-like колонке;
5. предпросмотр сгенерированного файла и подтверждение записи.

Гарантии:

- все введённые определения валидируются **до** записи файла: имя таблицы и свойства (идентификаторы `[A-Za-z_][A-Za-z0-9_]*`, без коллизий с членами `YdbBaseEntity`), наличие PK, уникальность колонок и значений enum, типы date-колонок, интервал TTL;
- существующий файл никогда не перезаписывается: коллизия обнаруживается до старта вопросов и завершается ошибкой;
- отмена (Ctrl+C) и EOF (Ctrl+D) — чистый выход (exit-код 130), файл не создаётся;
- никаких обращений к БД и DDL — только локальная генерация файла;
- вне TTY (CI, скрипты, закрытый stdin) ввод не читается вовсе: детерминированно создаётся шаблон по умолчанию (`uuid` PK типа `Uuid` + колонка `name`), команда не зависает.

Пример сгенерированной сущности:

```ts
import {
  YdbBaseEntity,
  YdbColumn,
  YdbCreateDateColumn,
  YdbEntity,
  YdbEnum,
  YdbPrimaryColumn,
  YdbTtl,
} from '@ycforge/ydb-orm';

export enum StatusEnum {
  ACTIVE = 'active',
  NEW_ORDER = 'new_order',
}

@YdbEntity('orders')
@YdbTtl({ interval: 'PT30D', column: 'expires_at' })
export class Order extends YdbBaseEntity {
  @YdbPrimaryColumn('Uuid')
  uuid: string;

  @YdbColumn('Utf8')
  @YdbEnum({ values: Object.values(StatusEnum), storage: 'Utf8' })
  status: StatusEnum;

  @YdbCreateDateColumn()
  @YdbColumn('Timestamp')
  created_at: Date;

  @YdbColumn('Timestamp')
  expires_at: Date;
}
```

Для скриптов и инструментов генерация доступна программно:

```ts
import { createEntityFileFromSpec } from '@ycforge/ydb-orm';

const created = createEntityFileFromSpec('./src', {
  className: 'OrderEntity',
  tableName: 'orders',
  columns: [
    { name: 'uuid', type: 'Uuid', primary: true },
    { name: 'status', type: 'Utf8', enumValues: ['active'], enumStorage: 'Utf8' },
    { name: 'created_at', type: 'Timestamp', createDate: true },
  ],
});
```

Также экспортируются `validateEntitySpec`, `renderEntityFile`, `buildDefaultEntitySpec`, а интерактивный мастер можно запускать над произвольными потоками через `runEntityCreateCommand` / `runEntityCreateWizard`.

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

### Экспорт метаданных: metadata:dump

Read-only команда выгружает метаданные сущностей из конфига (`entities`, как у `migration:generate`) в детерминированный JSON — **без подключения к БД**: ни драйвер, ни executor, ни DDL не задействуются. Команда по природе JSON-only: весь дамп приходит в stdout, отдельного текстового режима нет.

```bash
ydb-orm metadata:dump
```

Формат версионируется полями `format`/`version`; для каждой сущности выгружаются:

- имя класса и таблицы; колонки с YDB-типами (включая synthetic `{field}_bi` blind-index-колонки) и первичный ключ с порядком колонок;
- связи всех типов: тип связи, целевая сущность/таблица, join-колонка, обратное свойство (`inverseProperty`), для many-to-many — ссылка на join-таблицу; физические описания join-таблиц идут отдельным списком `joinTables` (колонки и их типы из фактических PK, владелец связи);
- индексы (имя, колонки с учётом порядка, unique) и TTL (колонка, ISO 8601 interval, unit);
- шифрование без секретов: только декларативные флаги полей (blind index + имя `_bi`-колонки, lazy, aadOverride) и AAD-поля PK; провайдеры, ключи и runtime-материал не экспортируются никогда;
- enum-метаданные (значения в семантическом порядке, storage `Utf8`/`Int32`), JSON-колонки и eager-связи.

Детерминированность: стабильный порядок сущностей (по имени таблицы), колонок, индексов, связей и ключей JSON — повторный запуск даёт побайтово одинаковый вывод. Наследование следует правилам #92/#107: собственные `@YdbIndex`/`@YdbTtl` не наследуются, колонки/PK/шифрование/eager наследуются. Невалидные метаданные (класс без `@YdbEntity`, дубликат имени таблицы, отсутствие PK, конфликтующие join-таблицы, невалидный селектор join-колонки, несовместимый TTL) роняют команду с понятной ошибкой до какого-либо вывода.

Для внешних инструментов доступен программный API: `buildMetadataDump(entities)` и типы `MetadataDump`, `DumpedEntity` и другие экспортируются из пакета.

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