# Repository (DI variant)

In addition to Active Record (`UserEntity.find(...)`), `YdbModule.forFeature([...])` registers an injectable `YdbRepository<Entity>`. This is more convenient in NestJS services and avoids relying on global static methods.

## Basic usage

```ts
import { Injectable } from '@nestjs/common';
import { InjectRepository, YdbRepository } from '@ycforge/yorm';
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

## EntityManager

`YdbEntityManager` is a repository factory. It is useful when a service needs several repositories or the set is not known in advance.

```ts
import { Injectable } from '@nestjs/common';
import { YdbEntityManager, YdbRepository } from '@ycforge/yorm';
import { UserEntity } from './user.entity.js';

@Injectable()
export class AdminService {
  private readonly userRepo: YdbRepository<UserEntity>;

  constructor(manager: YdbEntityManager) {
    this.userRepo = manager.getRepository(UserEntity);
  }
}
```

{% note info %}

`YdbEntityManager` is not registered automatically — provide it as a provider in your module if you want to inject it.

{% endnote %}

## Which to choose

| Approach | When to use |
|----------|-------------|
| **Active Record** | quick scripts, simple services, familiar TypeORM style |
| **YdbRepository + DI** | NestJS services, testability, explicit dependencies |

Active Record remains fully functional: static methods like `UserEntity.find(...)` are a thin facade delegating to the same `YdbRepository`.
