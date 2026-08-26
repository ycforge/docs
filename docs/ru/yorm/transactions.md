# Транзакции

ORM предоставляет менеджер транзакций `YdbTransactionManager`, который экспортируется корневым модулем NestJS-интеграции.

## Использование

```ts
import { Injectable } from '@nestjs/common';
import { YdbTransactionManager } from '@ycforge/ydb-orm';
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
      // Обе записи применятся атомарно: либо обе, либо ни одной
    });
  }
}
```

Внутри колбэка `trx` — executor транзакции. Он передаётся во все методы сущностей через `{ trx }` в `QueryOptions`. Так же через `options({ trx })` можно выполнить [QueryBuilder](query-builder.md).

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

Вызов `runInTransaction` внутри транзакции выполняет запросы в рамках той же транзакции (YDB-транзакции не вкладываются).

## Массовые операции

`insertMany`, `updateBy` и `deleteBy` тоже принимают `{ trx }` и выполняются в транзакции.

## Что транзакция не покрывает

DDL (CREATE TABLE, ALTER TABLE, DROP TABLE) в YDB не транзакционен — выполнение миграций последовательное. Используйте [миграции](migrations.md) для изменения схемы.