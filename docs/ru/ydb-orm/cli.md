# CLI

Пакет ставит бинарь `ydb-orm`: миграции, проверка схемы и генерация кода из командной строки.

## Команды

```bash
ydb-orm migration:create CreateUsers      # пустая миграция ./migrations/<ts>-CreateUsers.ts
ydb-orm migration:generate AddPhotos      # миграция по diff сущностей и БД
ydb-orm migration:run                     # применить все новые миграции
ydb-orm migration:revert                  # откатить последнюю
ydb-orm migration:show                    # статус миграций (алиас — migration:status)
ydb-orm migration:check                   # проверка готовности для CI (exit != 0, если не готово)
ydb-orm migration:repair <name> --as-applied|--as-reverted   # разрешить прерванную миграцию
ydb-orm schema:verify                     # сверить схему БД с метаданными сущностей
ydb-orm entity:create UserProfile         # сущность ./src/user-profile.entity.ts
ydb-orm metadata:dump                     # метаданные сущностей в JSON (stdout, без БД)
ydb-orm entity:diagram                    # Mermaid ER-диаграмма по метаданным (stdout/--output, без БД)
ydb-orm completion bash                   # скрипт shell-автодополнения (bash|zsh|fish)
```

Глобальные опции:

- `--dir <path>` — директория миграций (по умолчанию `./migrations`; для `entity:create` — `./src`).
- `--config <path>` — путь к конфигу.
- `--output <file>` — файл вывода для `entity:diagram` (существующий файл не перезаписывается).
- `--json` — машинный вывод для `migration:show`, `migration:status`, `migration:check`.
- `--verbose` — полный стек ошибки и цепочка cause при сбое.

Неизвестные флаги и пустые значения опций считаются ошибкой.

## Конфиг

Файл `./ydb-orm.config.ts` (или `.mts`/`.mjs`/`.js`; ищется в текущей директории, в `./src/` и выше до корня ФС; поддерживаются default и именованный экспорт):

```ts
import { createAuth, authKeyFromFile } from '@ycforge/auth';
import { UserEntity } from './src/user.entity.js';

export default {
  endpoint: process.env.YDB_ENDPOINT!,
  auth: createAuth(authKeyFromFile('./authorized_key.json')),
  entities: [UserEntity],        // нужно для migration:generate
  migrationsDir: './migrations', // опционально
};
```

Без конфига CLI читает env `YDB_ENDPOINT` (или `YDB_CONNECTION_STRING`), но для задания `auth` всё равно потребуется файл конфигурации.

## Миграции

### migration:create / migration:generate

`migration:create` пишет пустой файл `<timestamp>-<Name>.ts`. `migration:generate` строит diff по всем `entities` из конфига:

- нет таблицы → `CREATE TABLE` (+ `DROP TABLE` в `down`);
- нет колонок → `ADD COLUMN` (+ `DROP COLUMN` в `down`);
- расхождения типа/PK и лишние колонки не меняются автоматически — попадают в миграцию как `WARNING`-комментарии.

**Подсказки о переименовании**: когда diff выглядит как rename колонки (ровно одна лишняя колонка БД и одна новая колонка той же типа, без участия PK/индексов/TTL/blind-index), ADD/DROP для пары подавляются и добавляется комментарий-подсказка:

```ts
// SUGGESTION (not applied automatically): possible column rename detected.
// YQL has no ALTER TABLE RENAME COLUMN yet — verify the data and migrate manually:
//   ALTER TABLE `photos` RENAME COLUMN `label` TO `title`;
```

Подсказка никогда не выполняется автоматически; при неоднозначности поведение прежнее: `ADD COLUMN` + `WARNING`.

Обе команды печатают цветной diff расхождений, сгруппированный по таблицам. Цвет определяется по потоку вывода и отключается вне TTY или по `NO_COLOR`.

### migration:run / migration:revert / migration:repair

Порядок выполнения — по имени файла (`<timestamp>-<Name>.ts`). Применённые миграции хранятся в таблице `ydb_migrations` (создаётся автоматически). Идентичность — SHA-256 содержимого файла: переименование не вызывает повторного применения, изменение применённой миграции — ошибка.

DDL в YDB не транзакционен, поэтому перед каждым `up()`/`down()` пишется маркер `state='started'`, заменяемый на `'applied'` только после успеха. Если запуск умер посреди миграции:

- повторный `migration:run` не начнёт её заново вслепую;
- `migration:revert` откажется её откатывать;
- разрешите явно: `ydb-orm migration:repair <name> --as-applied` (изменения дозаведены вручную) или `--as-reverted` (изменения откачены вручную).

Параллельные запуски сталкиваются на детерминированном INSERT-claim (PRIMARY KEY) — двойное применение между процессами невозможно.

### Проверка готовности: migration:check / show / status

Read-only workflow: команды только читают состояние (`DescribeTable` для `ydb_migrations` + голый `SELECT`; для сущностей — `DescribeTable`) и ничего не меняют — в частности, таблица учёта не создаётся и не изменяется.

Состояния и exit-коды:

| Состояние | Exit-код | Значение |
| --- | --- | --- |
| готово | 0 | все миграции применены; схема совпадает, если проверялась |
| pending | 1 | есть неприменённые миграции |
| interrupted | 2 | есть прерванные миграции (`state='started'`) |
| schema-drift | 3 | схема БД расходится с метаданными сущностей (проверяется при заданном `entities` в конфиге) |
| modified | 4 | содержимое применённой миграции изменилось после применения |
| ошибка команды | 5 | не удалось подключиться/прочитать состояние/неожиданный сбой |

Если таблица учёта ещё не существует (свежая база), считается, что не применено ничего: при наличии файлов миграций это pending (exit 1), без них — готово (exit 0); в `--json` различается полем `bookkeeping: {exists: false}`.

Несколько состояний сразу → приоритет: `interrupted` → `modified` → `pending` → `schema-drift`. Orphan-записи (файл удалён после применения) показываются в отчёте, но сами по себе готовность не ломают.

Текстовый режим: сводка/список — в stdout, проблемы и diff схемы — в stderr, итоговая строка начинается с `Up to date:` или `Not ready:`. Для машинного разбора используйте `--json`: весь отчёт в stdout со стабильной схемой (`ready`, `state`, `states`, `exitCode`, списки `pending`/`interrupted`/`modified`/`orphaned`, детальный массив `migrations` и блок `schema`).

## Генерация кода

### entity:create

В TTY запускается интерактивный мастер: имя таблицы → цикл колонок (имя → тип YDB → PK → encrypted/blind index → enum значения + хранилище) → авто-timestamps для date-колонок → опциональный TTL (ISO 8601 duration) → предпросмотр и подтверждение записи.

Гарантии:

- каждое определение валидируется **до** записи файла;
- существующий файл никогда не перезаписывается (коллизия падает до первого вопроса);
- Ctrl+C / EOF — чистый выход (код 130) без записи;
- никаких обращений к БД и DDL — только локальная генерация;
- вне TTY ввод не читается вовсе: детерминированно создаётся шаблон по умолчанию (`uuid` PK + `name`), команда не зависает.

Программный API: `createEntityFileFromSpec(dir, spec)` (+ `validateEntitySpec`, `renderEntityFile`, `buildDefaultEntitySpec`, `runEntityCreateCommand`).

### metadata:dump

Выгружает канонические метаданные сущностей из конфига в детерминированный JSON — **без подключения к БД**. Для каждой сущности: имена класса/таблицы, колонки с YDB-типами (включая synthetic `{field}_bi`), порядок PK, связи всех типов (+ физические `joinTables`), индексы, TTL, декларативные флаги шифрования (без секретов никогда), enum'ы, JSON-колонки, eager-связи. Стабильный порядок — побайтово одинаковый вывод между запусками. Программный API: `buildMetadataDump(entities)`.

### entity:diagram

Рендерит те же канонические метаданные в Mermaid ER-диаграмму — без подключения к БД:

```bash
ydb-orm entity:diagram                            # Mermaid-текст в stdout
ydb-orm entity:diagram --output docs/schema.mmd   # в файл (перезапись запрещена)
```

PK-колонки идут первыми в порядке объявления с маркером `PK`; FK-колонки помечены `FK`; many-to-many рисуется через физическую join-таблицу; порядок блоков/линий детерминирован. Программный API: `buildEntityDiagram(entities)`, `writeDiagramFile(path, diagram)`.

## Автодополнение шелла

```bash
# bash
ydb-orm completion bash | sudo tee /etc/bash_completion.d/ydb-orm
# zsh
ydb-orm completion zsh > ~/.zsh/completions/_ydb-orm
# fish
ydb-orm completion fish > ~/.config/fish/completions/ydb-orm.fish
```
