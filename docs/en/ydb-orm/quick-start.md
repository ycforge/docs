# Quick start

## 1. Define an entity

Each entity is a class extending `YdbBaseEntity` and decorated with `@YdbEntity('table_name')`.

```ts
import {
  YdbBaseEntity,
  YdbEntity,
  YdbColumn,
  YdbPrimaryColumn,
} from '@ycforge/ydb-orm';

@YdbEntity('users')
export class UserEntity extends YdbBaseEntity {
  @YdbPrimaryColumn('Uuid')
  uuid: string;

  @YdbColumn('Utf8')
  name: string;

  @YdbColumn('Utf8')
  email: string;
}
```

## 2. Create a driver and executor

```ts
import { createDriver, createExecutor, configureEntities } from '@ycforge/ydb-orm';
import { createAuth, authKeyFromFile } from '@ycforge/auth';

const driver = await createDriver({
  endpoint: 'grpcs://ydb.serverless.yandexcloud.net:2135/?database=/ru-central1/b1g.../ydb',
  auth: createAuth(authKeyFromFile('./authorized_key.json')),
});

const executor = createExecutor(driver);
```

## 3. Configure entities

```ts
configureEntities([UserEntity], { executor });
```

After `configureEntities`, Active Record static methods are available on every entity.

## 4. Use Active Record

```ts
import { UserEntity } from './user.entity';

// Insert (uuid is generated automatically)
const user = new UserEntity();
user.name = 'John';
user.email = 'john@example.com';
await UserEntity.save(user);

// Read
const found = await UserEntity.find({ uuid: user.uuid });
console.log(found?.name); // 'John'

// Update (save with a filled uuid)
user.name = 'John Doe';
await UserEntity.save(user);

// List with pagination
const page = await UserEntity.findAll({}, { limit: 50, offset: 0 });

// Count
const total = await UserEntity.count({});

// Delete
await UserEntity.delete(user.uuid);
```

## 5. Use a repository

`getOrCreateRepository` creates a `YdbRepository<Entity>` from the configured runtime dependencies. It keeps data-access logic in one place and is easier to mock in tests.

```ts
import { getOrCreateRepository } from '@ycforge/ydb-orm';
import { UserEntity } from './user.entity';

const users = getOrCreateRepository(UserEntity);

// CRUD
const user = await users.findOneBy({ uuid });
const all = await users.findAll({ name: 'John' }, { limit: 50 });
const count = await users.count({ is_admin: true });
await users.save(entity);
await users.insertMany([u1, u2]);
await users.updateBy({ status: 'old' }, { status: 'archived' });
await users.deleteBy({ status: 'deprecated' });

// QueryBuilder
const popular = await users.query()
  .where({ is_public: true })
  .orderBy('views', 'DESC')
  .limit(20)
  .getMany();
```

`YdbEntityManager` is a repository factory — convenient when you work with multiple entities through a single interface:

```ts
import { YdbEntityManager } from '@ycforge/ydb-orm';

const manager = new YdbEntityManager();
const userRepo = manager.getRepository(UserEntity);
const postRepo = manager.getRepository(PostEntity);

await userRepo.save(user);
const posts = await postRepo.findBy({ author_uuid: user.uuid });
```

{% note info %}

The repository pattern is optional — you can call `UserEntity` static methods from anywhere directly (see [Active Record](active-record.md)). See the [Repository](repository.md) section for the full list of repository methods.

{% endnote %}

## 6. Run schema sync (development)

On startup with `sync: true`, the ORM creates missing tables and columns automatically. For production, use [migrations](migrations.md) instead of `sync`.

```ts
import { YdbSchemaSyncer } from '@ycforge/ydb-orm';

const syncer = new YdbSchemaSyncer(executor);
await syncer.sync([UserEntity]);
```

## 7. Close the driver when done

```ts
driver.close();
```

## Next steps

- [Entities & decorators](entity.md) — all available decorators and column types
- [Active Record](active-record.md) — static methods, QueryOptions, lazy decryption
- [Repository](repository.md) — standalone and NestJS DI patterns
- [Transactions](transactions.md) — atomic operations
- [NestJS quick start](nest/quick-start.md) — if you use NestJS, see the NestJS-specific guide
