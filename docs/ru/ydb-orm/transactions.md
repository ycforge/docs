# Транзакции

ORM предоставляет `YdbTransactionManager` для атомарных операций. Транзакции гарантируют, что несколько записей либо все применятся, либо ни одна.

## Использование

```ts
import { YdbTransactionManager } from '@ycforge/ydb-orm';
import type { QueryOptions } from '@ycforge/ydb-orm';
import { UserEntity } from './user.entity';

const txManager = new YdbTransactionManager(executor);

await txManager.runInTransaction(async (trx) => {
  const opts: QueryOptions = { trx };

  const from = await UserEntity.find({ uuid: fromId }, opts);
  const to = await UserEntity.find({ uuid: toId }, opts);

  from.balance -= 100;
  to.balance += 100;

  await UserEntity.save(from, opts);
  await UserEntity.save(to, opts);
  // Обе записи применяются атомарно: либо обе, либо ни одна
});
```

Внутри колбэка `trx` — executor транзакции. Он передаётся во все методы сущностей через `{ trx }` в `QueryOptions`. Также можно использовать [QueryBuilder](query-builder.md) через `options({ trx })`.

## QueryBuilder в транзакции

```ts
await txManager.runInTransaction(async (trx) => {
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

DDL (CREATE TABLE, ALTER TABLE, DROP TABLE) в YDB не транзакционен — выполнение миграций последовательное. Используйте [миграции](migrations.md) для изменения схемы.

## Следующие шаги

- [Транзакции (NestJS)](nest/transactions.md) — если вы используете NestJS, см. инжектируемый паттерн
