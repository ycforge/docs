# Schema sync

`schema sync` — аналог `synchronize` в TypeORM: при старте приложения ORM подстраивает схему БД под метаданные всех зарегистрированных сущностей.

## Включение

Задайте `sync: true` в опциях `forRootAsync`:

```ts
YdbCoreModule.forRootAsync({
  useFactory: () => ({
    endpoint: '...',
    auth_type: 'auth_key',
    authOptions: { authorized_key_path: './authorized_key.json' },
    sync: true, // только для dev!
  }),
});
```

{% note warning %}

В продакшене используйте [миграции](migrations.md) вместо `sync: true`.

{% endnote %}

## Что делает синхронизация

При старте приложения, после создания драйвера, ORM для каждой зарегистрированной сущности:

- **нет таблицы** → `CREATE TABLE`;
- **нет колонок** → `ALTER TABLE ADD COLUMN`;
- **лишние колонки** → только предупреждение в лог (данные не удаляются);
- **расхождение типа колонки или PK** → ошибка (YDB не позволяет менять тип колонки или PK, нужна миграция).

Описание таблицы получается через Table service `DescribeTable` (query service метаданные колонок не отдаёт). Создаются также synthetic-колонки `{field}_bi` для blind index и join-таблицы many-to-many.

## Ручной вызов

Провайдер `YDB_SCHEMA_SYNC` экспортируется корневым модулем. Можно проверить или применить схему вручную.

```ts
import { Inject, Injectable } from '@nestjs/common';
import { YDB_SCHEMA_SYNC, YdbSchemaSyncer } from '@ycforge/yorm';
import { UserEntity, PostEntity } from './entities';

@Injectable()
export class SchemaService {
  constructor(
    @Inject(YDB_SCHEMA_SYNC)
    private readonly syncer: YdbSchemaSyncer,
  ) {}

  // Проверка без изменений
  async check() {
    const issues = await this.syncer.verify([UserEntity, PostEntity]);
    console.log('Проблемы схемы:', issues);
  }

  // Принудительная синхронизация
  async apply() {
    await this.syncer.sync([UserEntity, PostEntity]);
  }
}
```

## DDL-генераторы

Генераторы DDL доступны в публичном API — их можно использовать для построения миграций вручную:

```ts
import {
  buildExpectedTableSchema,
  generateCreateTableYql,
  generateAddColumnsYql,
  checkTableSchema,
} from '@ycforge/yorm';

const expected = buildExpectedTableSchema(UserEntity);
const createYql = generateCreateTableYql(expected);
console.log(createYql);
```

## Требования

Каждая сущность обязана иметь PK-колонку (`@YdbPrimaryColumn` или `uuid`). Иначе sync упадёт с понятной ошибкой.