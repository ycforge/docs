# Репозиторий (DI-вариант)

Помимо Active Record (`UserEntity.find(...)`) `YdbModule.forFeature([...])` регистрирует инжектируемый `YdbRepository<Entity>`. Это удобнее в NestJS-сервисах и не требует обращения к глобальным статическим методам.

## Базовое использование

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

`YdbRepository` содержит те же методы, что и Active Record:

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

`YdbEntityManager` — фабрика репозиториев. Удобен, когда сервису нужны несколько репозиториев или их набор заранее неизвестен.

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

{% note info %}

`YdbEntityManager` не регистрируется автоматически — предоставьте его как provider в модуле, если хотите инжектировать.

{% endnote %}

## Что выбрать

| Подход | Когда использовать |
|--------|-------------------|
| **Active Record** | быстрые скрипты, простые сервисы, привычный TypeORM-стиль |
| **YdbRepository + DI** | NestJS-сервисы, тестируемость, явные зависимости |

Active Record остаётся полностью работоспособным: статические методы `UserEntity.find(...)` — тонкий фасад, делегирующий вызовы в тот же `YdbRepository`.
