# NestJS repository injection

In NestJS applications, `YdbOrmModule.forFeature([...])` registers an injectable `YdbRepository<Entity>`. This is more convenient in NestJS services and avoids relying on global static methods.

## Injecting a repository

```ts
import { Injectable } from '@nestjs/common';
import { InjectRepository, YdbRepository } from '@ycforge/ydb-orm/nest';
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

## Injecting YdbEntityManager

`YdbEntityManager` can also be injected in NestJS (register it as a provider manually):

```ts
import { Injectable } from '@nestjs/common';
import { YdbEntityManager, YdbRepository } from '@ycforge/ydb-orm/nest';
import { UserEntity } from './user.entity.js';

@Injectable()
export class AdminService {
  private readonly userRepo: YdbRepository<UserEntity>;

  constructor(manager: YdbEntityManager) {
    this.userRepo = manager.getRepository(UserEntity);
  }
}
```

## DI tokens

| Token | Description |
| --- | --- |
| `YDB_DRIVER` | YDB driver |
| `YDB_QUERY` | query executor |
| `YDB_OPTIONS` | module options |
| `YDB_CREDENTIALS_PROVIDER` | credentials provider |
| `YDB_ENCRYPTION_PROVIDER` | encryption provider |
| `YDB_BLIND_INDEX_PROVIDER` | blind index provider |
| `YDB_SCHEMA_SYNC` | schema syncer |

Use `@Inject(TOKEN)` to access any of these in your NestJS services.

## Standalone alternative

If you don't use NestJS, see the [standalone repository](../repository.md) patterns with `getOrCreateRepository` and `YdbEntityManager`.
