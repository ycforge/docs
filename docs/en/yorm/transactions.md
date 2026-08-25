# Transactions

The ORM provides the transaction manager `YdbTransactionManager`, exported by the root module of the NestJS integration.

## Usage

```ts
import { Injectable } from '@nestjs/common';
import { YdbTransactionManager } from '@ycforge/yorm';
import type { QueryOptions } from '@ycforge/yorm';
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
      // Both records are applied atomically: either both or none
    });
  }
}
```

Inside the callback, `trx` is the transaction executor. It is passed to all entity methods via `{ trx }` in `QueryOptions`. You can also run a [QueryBuilder](query-builder.md) this way via `options({ trx })`.

## QueryBuilder in a transaction

```ts
await this.txManager.runInTransaction(async (trx) => {
  const posts = await PostEntity.query()
    .where({ author_uuid: userId })
    .options({ trx })
    .getMany();
});
```

## Nested transactions

Calling `runInTransaction` inside a transaction runs queries within the same transaction (YDB transactions do not nest).

## Bulk operations

`insertMany`, `updateBy`, and `deleteBy` also accept `{ trx }` and run inside a transaction.

## What transactions do not cover

DDL (CREATE TABLE, ALTER TABLE, DROP TABLE) in YDB is not transactional — migration execution is sequential. Use [migrations](migrations.md) to change the schema.