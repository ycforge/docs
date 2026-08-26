# Repository

`YdbRepository<T>` is the core of the ORM: all CRUD logic lives in it (and in `YdbEntityPersistence`/`YdbEntityRelations` under the hood). You can use it directly — without NestJS and without DI.

## Standalone

After `configureEntities` the repository is created automatically:

```ts
import { getOrCreateRepository } from '@ycforge/ydb-orm';

const repo = getOrCreateRepository(UserEntity);

// CRUD
const user = await repo.findOneBy({ uuid });
const users = await repo.findAll({ name: 'Ivan' }, { limit: 50 });
const count = await repo.count({ is_admin: true });
await repo.save(entity);
await repo.insertMany([u1, u2]);
await repo.updateBy({ status: 'old' }, { status: 'archived' });
await repo.deleteBy({ status: 'deprecated' });

// QueryBuilder
const popular = await repo.query()
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

// CRUD through repos
await userRepo.save(user);
const posts = await postRepo.findBy({ author_uuid: user.uuid });
await postRepo.insertMany([post1, post2, post3]);

// Transactions — pass { trx } to any method
import { YdbTransactionManager } from '@ycforge/ydb-orm';

const txManager = new YdbTransactionManager(executor);
await txManager.runInTransaction(async (trx) => {
  await userRepo.save(user, { trx });
  await postRepo.save(post, { trx });
});
```

`YdbRepository` exposes the same methods as Active Record:

- `find(where, options?)`
- `findOneBy(where, options?)`
- `findAll(where?, options?)`
- `findBy(where, options?)`
- `count(where?, options?)`
- `save(entity, options?)`
- `insertMany(entities, options?)`
- `delete(pkValue, options?)`
- `deleteBy(where, options?)`
- `updateBy(where, patch, options?)`
- `query()`
- `loadRelations(items, relationNames, options?)`

## NestJS DI

In NestJS applications, `YdbOrmModule.forFeature([...])` registers an injectable `YdbRepository<Entity>`. This is more convenient in NestJS services and avoids relying on global static methods.

```ts
import { Injectable } from '@nestjs/common';
import { InjectRepository, YdbRepository } from '@ycforge/ydb-orm';
import { UserEntity } from './user.entity.js';

@Injectable()
export class UserService {
  constructor(
    @InjectRepository(UserEntity)
    private readonly userRepo: YdbRepository<UserEntity>,
  ) {}

  async getByUuid(uuid: string) {
    return this.userRepo.findOneBy({ uuid });
  }

  async create(name: string) {
    const user = new UserEntity();
    user.name = name;
    return this.userRepo.save(user);
  }
}
```

`YdbEntityManager` can also be injected in NestJS (register it as a provider manually):

```ts
import { Injectable } from '@nestjs/common';
import { YdbEntityManager, YdbRepository } from '@ycforge/ydb-orm';
import { UserEntity } from './user.entity.js';

@Injectable()
export class AdminService {
  private readonly userRepo: YdbRepository<UserEntity>;

  constructor(manager: YdbEntityManager) {
    this.userRepo = manager.getRepository(UserEntity);
  }
}
```

## Which to choose

| Approach | When to use |
|----------|-------------|
| **Active Record** | quick scripts, simple services, familiar TypeORM style |
| **Repository (standalone)** | multi-entity scripts, services without DI, programmatic access |
| **Repository + DI** (NestJS) | NestJS services, testability, explicit dependencies |

Active Record remains fully functional: static methods like `UserEntity.find(...)` are a thin facade delegating to the same `YdbRepository`. Both styles can be mixed in one application.
