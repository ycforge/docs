# Транзакции (NestJS)

В NestJS-приложениях `YdbTransactionManager` доступен через DI при импорте `YdbCoreModule.forRootAsync()`.

## Использование

```ts
import { Injectable } from '@nestjs/common';
import { YdbTransactionManager } from '@ycforge/ydb-orm/nest';
import type { QueryOptions } from '@ycforge/ydb-orm';
import { UserEntity } from './user.entity';

@Injectable()
export class UsersService {
  constructor(private readonly txManager: YdbTransactionManager) {}

  async transfer() {
    await this.txManager.runInTransaction(async (trx) => {
      const opts: QueryOptions = { trx };

      const from = await UserEntity.find({ uuid: fromId }, opts);
      const to = await UserEntity.find({ uuid: toId }, opts);

      from.balance -= 100;
      to.balance += 100;

      await UserEntity.save(from, opts);
      await UserEntity.save(to, opts);
      // Обе записи применяются атомарно: либо обе, либо ни одна
    });
  }
}
```

Внутри колбэка `trx` — executor транзакции. Он передаётся во все методы сущностей через `{ trx }` в `QueryOptions`. Также можно использовать [QueryBuilder](../query-builder.md) через `options({ trx })`.

## QueryBuilder в транзакции

```ts
await this.txManager.runInTransaction(async (trx) => {
  const posts = await PostEntity.query()
    .where({ author_uuid: userId })
    .options({ trx })
    .getMany();
});
```

## Вложенные транзакции

Вызов `runInTransaction` внутри транзакции выполняет запросы в той же транзакции (транзакции YDB не вкладываются).

## Массовые операции

`insertMany`, `updateBy` и `deleteBy` также принимают `{ trx }` и выполняются внутри транзакции.

## Что не покрывают транзакции

DDL (CREATE TABLE, ALTER TABLE, DROP TABLE) в YDB не транзакционен — выполнение миграций последовательное. Используйте [миграции](../migrations.md) для изменения схемы.

## Standalone-альтернатива

Если вы не используете NestJS, см. руководство [standalone-транзакций](../transactions.md) с `new YdbTransactionManager(executor)`.
