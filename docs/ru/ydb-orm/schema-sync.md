# Schema sync

`schema sync` — аналог `synchronize` в TypeORM: ORM.adjustирует схему БД под метаданные всех зарегистрированных сущностей.

## Standalone-использование

```ts
import { YdbSchemaSyncer } from '@ycforge/ydb-orm';
import { UserEntity, PostEntity } from './entities';

const syncer = new YdbSchemaSyncer(executor);

// Проверка без изменений
const issues = await syncer.verify([UserEntity, PostEntity]);
console.log('Проблемы схемы:', issues);

// Применение изменений (CREATE TABLE, ALTER TABLE ADD COLUMN)
await syncer.sync([UserEntity, PostEntity]);
```

{% note warning %}

В продакшене используйте [миграции](migrations.md) вместо `sync`.

{% endnote %}

## Что делает sync

После создания драйвера ORM выполняет следующее для каждой зарегистрированной сущности:

- **нет таблицы** → `CREATE TABLE`;
- **нет колонок** → `ALTER TABLE ADD COLUMN`;
- **лишние колонки** → только предупреждение в логе (данные не удаляются);
- **несовпадение типа колонки или PK** → ошибка (YDB не может изменить тип колонки или PK; необходима миграция).

Описание таблицы получается через Table service `DescribeTable` (query service не возвращает метаданные колонок). Synthetic-колонки `{field}_bi` для blind index и join-таблицы many-to-many также создаются.

## Генераторы DDL

Генераторы DDL доступны в публичном API — можно использовать для ручного создания миграций:

```ts
import {
  buildExpectedTableSchema,
  generateCreateTableYql,
  generateAddColumnsYql,
  checkTableSchema,
} from '@ycforge/ydb-orm';

const expected = buildExpectedTableSchema(UserEntity);
const createYql = generateCreateTableYql(expected);
console.log(createYql);
```

## Требования

Каждая сущность обязана иметь PK-колонку (`@YdbPrimaryColumn` или `uuid`). Иначе sync падает с понятной ошибкой.

## Следующие шаги

- [Schema sync (NestJS)](nest/schema-sync.md) — если вы используете NestJS, см. инжектируемый паттерн
