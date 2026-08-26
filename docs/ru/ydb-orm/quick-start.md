# Быстрый старт

## 1. Определите сущность

Каждая сущность — класс, наследующий `YdbBaseEntity` и декорированный `@YdbEntity('имя_таблицы')`.

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

## 2. Создайте драйвер и executor

```ts
import { createDriver, createExecutor, configureEntities } from '@ycforge/ydb-orm';
import { createAuth, authKeyFromFile } from '@ycforge/auth';

const driver = await createDriver({
  endpoint: 'grpcs://ydb.serverless.yandexcloud.net:2135/?database=/ru-central1/b1g.../ydb',
  auth: createAuth(authKeyFromFile('./authorized_key.json')),
});

const executor = createExecutor(driver);
```

## 3. Настройте сущности

```ts
configureEntities([UserEntity], { executor });
```

После `configureEntities` статические методы Active Record доступны на каждой сущности.

## 4. Используйте Active Record

```ts
import { UserEntity } from './user.entity';

// Вставка (uuid генерируется автоматически)
const user = new UserEntity();
user.name = 'Иван';
user.email = 'ivan@example.com';
await UserEntity.save(user);

// Чтение
const found = await UserEntity.find({ uuid: user.uuid });
console.log(found?.name); // 'Иван'

// Обновление (save с заполненным uuid)
user.name = 'Иван Петров';
await UserEntity.save(user);

// Список с пагинацией
const page = await UserEntity.findAll({}, { limit: 50, offset: 0 });

// Количество
const total = await UserEntity.count({});

// Удаление
await UserEntity.delete(user.uuid);
```

## 5. Используйте репозиторий

`getOrCreateRepository` создаёт `YdbRepository<Entity>` из настроенных runtime-зависимостей. Он собирает логику доступа к данным в одном месте и проще мокать в тестах.

```ts
import { getOrCreateRepository } from '@ycforge/ydb-orm';
import { UserEntity } from './user.entity';

const users = getOrCreateRepository(UserEntity);

// CRUD
const user = await users.findOneBy({ uuid });
const all = await users.findAll({ name: 'Иван' }, { limit: 50 });
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

`YdbEntityManager` — фабрика репозиториев. Удобен, когда нужно работать с разными сущностями через единый интерфейс:

```ts
import { YdbEntityManager } from '@ycforge/ydb-orm';

const manager = new YdbEntityManager();
const userRepo = manager.getRepository(UserEntity);
const postRepo = manager.getRepository(PostEntity);

await userRepo.save(user);
const posts = await postRepo.findBy({ author_uuid: user.uuid });
```

{% note info %}

Репозиторий — необязательный паттерн: статические методы `UserEntity` можно вызывать из любого места напрямую (см. [Active Record](active-record.md)). Подробнее о методах репозитория читайте в разделе [Репозиторий](repository.md).

{% endnote %}

## 6. Запустите schema sync (для разработки)

При старте с опцией `sync: true` ORM создаст недостающие таблицы и колонки автоматически. Для продакшена используйте [миграции](migrations.md) вместо `sync`.

```ts
import { YdbSchemaSyncer } from '@ycforge/ydb-orm';

const syncer = new YdbSchemaSyncer(executor);
await syncer.sync([UserEntity]);
```

## 7. Закройте драйвер по завершении

```ts
driver.close();
```

## Следующие шаги

- [Сущности и декораторы](entity.md) — все доступные декораторы и типы колонок
- [Active Record](active-record.md) — статические методы, QueryOptions, ленивая дешифровка
- [Репозиторий](repository.md) — standalone и NestJS DI паттерны
- [Транзакции](transactions.md) — атомарные операции
- [Быстрый старт (NestJS)](nest/quick-start.md) — если вы используете NestJS, см. NestJS-руководство
