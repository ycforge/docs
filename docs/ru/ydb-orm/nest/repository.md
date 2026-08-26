# Инъекция репозитория (NestJS)

В NestJS-приложениях `YdbOrmModule.forFeature([...])` регистрирует инжектируемый `YdbRepository<Entity>`. Это удобнее в NestJS-сервисах и не требует обращения к глобальным статическим методам.

## Инъекция репозитория

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

## Инъекция YdbEntityManager

`YdbEntityManager` также доступен для инъекции в NestJS (зарегистрируйте как provider вручную):

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

## DI-токены

| Токен | Описание |
| --- | --- |
| `YDB_DRIVER` | драйвер YDB |
| `YDB_QUERY` | executor запросов |
| `YDB_OPTIONS` | опции модуля |
| `YDB_CREDENTIALS_PROVIDER` | credentials provider |
| `YDB_ENCRYPTION_PROVIDER` | провайдер шифрования |
| `YDB_BLIND_INDEX_PROVIDER` | провайдер blind index |
| `YDB_SCHEMA_SYNC` | schema syncer |

Используйте `@Inject(TOKEN)` для доступа к любому из них в NestJS-сервисах.

## Standalone-альтернатива

Если вы не используете NestJS, см. паттерны [standalone-репозитория](../repository.md) с `getOrCreateRepository` и `YdbEntityManager`.
