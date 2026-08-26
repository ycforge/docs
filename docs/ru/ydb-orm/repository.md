# Репозиторий

`YdbRepository<T>` — ядро ORM: вся CRUD-логика живёт в нём (и в `YdbEntityPersistence`/`YdbEntityRelations` под капотом). Можно использовать напрямую — без NestJS и без DI.

## Standalone

После `configureEntities` репозиторий создаётся автоматически:

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

`YdbEntityManager` — фабрика репозиториев. Удобен, когда нужно работать с разными сущностями через единый интерфейс:

```ts
import { YdbEntityManager } from '@ycforge/ydb-orm';

const manager = new YdbEntityManager();
const userRepo = manager.getRepository(UserEntity);
const postRepo = manager.getRepository(PostEntity);

// CRUD через репозитории
await userRepo.save(user);
const posts = await postRepo.findBy({ author_uuid: user.uuid });
await postRepo.insertMany([post1, post2, post3]);

// Транзакции — передаём { trx } в любой метод
import { YdbTransactionManager } from '@ycforge/ydb-orm';

const txManager = new YdbTransactionManager(executor);
await txManager.runInTransaction(async (trx) => {
  await userRepo.save(user, { trx });
  await postRepo.save(post, { trx });
});
```

`YdbRepository` содержит те же методы, что и Active Record:

- `find(where, options?)`
- `findOneBy(where, options?)`
- `findAll(where?, options?)`
- `findBy(where?, options?)`
- `count(where?, options?)`
- `save(entity, options?)`
- `insertMany(entities, options?)`
- `delete(pkValue, options?)`
- `deleteBy(where, options?)`
- `updateBy(where, patch, options?)`
- `query()`
- `loadRelations(items, relationNames, options?)`

## NestJS DI

В NestJS-приложениях `YdbOrmModule.forFeature([...])` регистрирует инжектируемый `YdbRepository<Entity>`. Это удобнее в NestJS-сервисах и не требует обращения к глобальным статическим методам.

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

`YdbEntityManager` также доступен для инжекции в NestJS (зарегистрируйте как provider вручную):

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

## Что выбрать

| Подход | Когда использовать |
|--------|-------------------|
| **Active Record** | быстрые скрипты, простые сервисы, привычный TypeORM-стиль |
| **Репозиторий (standalone)** | скрипты с несколькими сущностями, сервисы без DI, программный доступ |
| **Репозиторий + DI** (NestJS) | NestJS-сервисы, тестируемость, явные зависимости |

Active Record остаётся полностью работоспособным: статические методы `UserEntity.find(...)` — тонкий фасад, делегирующий вызовы в тот же `YdbRepository`. Оба стиля можно смешивать в одном приложении.
