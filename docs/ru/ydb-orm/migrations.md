# Миграции

Миграции — способ управления схемой БД в продакшене. Как в TypeORM: миграция — класс с методами `up`/`down`, получающий `YdbExecutor`.

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

Node ≥ 22.18 импортирует `.ts` напрямую; отдельный ts-node не нужен. Из-за нативного стриппинга типов импортируйте типы (`YdbMigration`, `YdbExecutor`) через `import type` — обычный именованный импорт типа упадёт в рантайме.

{% endnote %}

## Гарантии надёжности

- **Стабильная идентичность**: каждая миграция получает SHA-256 содержимого файла. Сопоставление с `ydb_migrations` идёт по хешу — переименование файла не вызывает повторного применения; изменение содержимого применённой миграции — ошибка.
- **Частичное применение**: DDL в YDB не транзакционен, поэтому перед `up()`/`down()` пишется маркер `state='started'`, заменяемый на `'applied'` только после успеха. Падение посреди миграции оставляет маркер: повторный `run()` не начнёт её заново вслепую, а `revert()` откажется откатывать, пока состояние не разрешат явно — `runner.markMigrationApplied(name)` или `runner.removeMigrationRecord(name)` (CLI: `ydb-orm migration:repair <name> --as-applied|--as-reverted`).
- **Параллельные запуски**: claim на применение — INSERT с id, детерминированным из хеша. Два процесса сталкиваются на PRIMARY KEY до выполнения `up()`.

## Команды

Все команды миграций (`migration:create|generate|run|revert|show/status|check|repair`), проверка схемы (`schema:verify`) и генерация кода (`entity:create`, `metadata:dump`, `entity:diagram`, `completion`) описаны в [справочнике CLI](cli.md).

```bash
ydb-orm migration:create CreateUsers      # пустая миграция
ydb-orm migration:generate AddPhotos      # миграция по diff сущностей и БД
ydb-orm migration:run                     # применить все новые миграции
ydb-orm migration:revert                  # откатить последнюю
ydb-orm migration:check                   # проверка готовности для CI
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

`planMigration(expected, existing)` — чистая функция, строящая план миграции по diff между ожидаемыми схемами сущностей и текущим состоянием БД.
