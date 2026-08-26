# Schema sync (NestJS)

В NestJS-приложениях schema sync настраивается через `YdbCoreModule.forRootAsync()` и доступен через DI.

## Включение

Установите `sync: true` в опциях `forRootAsync`:

```ts
YdbCoreModule.forRootAsync({
  useFactory: () => ({
    endpoint: '...',
    auth: createAuth(authKeyFromFile('./authorized_key.json')),
    sync: true, // только для dev!
  }),
});
```

{% note warning %}

В продакшене используйте [миграции](../migrations.md) вместо `sync: true`.

{% endnote %}

## Ручной вызов

Провайдер `YDB_SCHEMA_SYNC` экспортируется корневым модулем. Можно проверить или применить схему вручную.

```ts
import { Inject, Injectable } from '@nestjs/common';
import { YDB_SCHEMA_SYNC, YdbSchemaSyncer } from '@ycforge/ydb-orm/nest';
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

## Что делает sync

После создания драйвера ORM выполняет следующее для каждой зарегистрированной сущности:

- **нет таблицы** → `CREATE TABLE`;
- **нет колонок** → `ALTER TABLE ADD COLUMN`;
- **лишние колонки** → только предупреждение в логе (данные не удаляются);
- **несовпадение типа колонки или PK** → ошибка (YDB не может изменить тип колонки или PK; необходима миграция).

Описание таблицы получается через Table service `DescribeTable` (query service не возвращает метаданные колонок). Synthetic-колонки `{field}_bi` для blind index и join-таблицы many-to-many также создаются.

## Standalone-альтернатива

Если вы не используете NestJS, см. руководство [standalone-schema sync](../schema-sync.md) с `new YdbSchemaSyncer(executor)`.
