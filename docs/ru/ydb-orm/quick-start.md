# Быстрый старт

Быстрый старт для обычного Node.js-приложения — без фреймворков. Для NestJS см. [Быстрый старт (NestJS)](nest/quick-start.md).

## 1. Создайте проект

```bash
mkdir myapp && cd myapp
yarn init -y
yarn add @ycforge/ydb-orm reflect-metadata
```

Пакет — ESM-only (`"type": "module"`), требуется Node.js >= 22. Убедитесь, что в `package.json` есть `"type": "module"`, а в `tsconfig.json` включены декораторы:

```json
{
  "compilerOptions": {
    "target": "ES2023",
    "module": "nodenext",
    "moduleResolution": "nodenext",
    "experimentalDecorators": true,
    "emitDecoratorMetadata": true,
    "strict": true
  }
}
```

`reflect-metadata` импортируется один раз в точке входа приложения.

## 2. Определите сущность

Каждая сущность — класс, наследующий `YdbBaseEntity` и декорированный `@YdbEntity('имя_таблицы')`.

```ts
// src/user.entity.ts
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

## 3. Подключитесь к YDB и настройте сущности

```ts
// src/db.ts
import { createDriver, createExecutor, configureEntities } from '@ycforge/ydb-orm';
import { createAuth, authKeyFromFile } from '@ycforge/auth';
import { UserEntity } from './user.entity.js';

export async function initDb() {
  const options = {
    endpoint: 'grpcs://ydb.serverless.yandexcloud.net:2135/?database=/ru-central1/b1g.../ydb',
    auth: createAuth(authKeyFromFile('./authorized_key.json')),
  };

  const driver = await createDriver(options);
  const executor = createExecutor(driver, options);

  // Пробрасываем executor (и опционально провайдеры шифрования/валидации) в сущности.
  // После этого доступны статические методы Active Record.
  configureEntities([UserEntity], { executor });

  return { driver, executor };
}
```

## 4. Используйте Active Record

```ts
// src/main.ts
import 'reflect-metadata';
import { initDb } from './db.js';
import { UserEntity } from './user.entity.js';

const { driver } = await initDb();

try {
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
} finally {
  driver.close();
}
```

Запуск:

```bash
node --experimental-strip-types src/main.ts
# или скомпилируйте через tsc и запустите dist/main.js
```

## 5. Schema sync (для разработки)

В разработке ORM может создавать недостающие таблицы и колонки автоматически:

```ts
import { YdbSchemaSyncer } from '@ycforge/ydb-orm';

const syncer = new YdbSchemaSyncer(executor);
await syncer.sync([UserEntity]);
```

{% note warning %}

Для продакшена используйте [миграции](migrations.md) вместо schema sync.

{% endnote %}

## 6. Репозиторий (опционально)

`getOrCreateRepository` создаёт `YdbRepository<Entity>` из настроенных runtime-зависимостей. Он собирает логику доступа к данным в одном месте и проще мокается в тестах.

```ts
import { getOrCreateRepository, YdbEntityManager } from '@ycforge/ydb-orm';

const users = getOrCreateRepository(UserEntity);
const user = await users.findOneBy({ uuid });

// QueryBuilder через репозиторий
const recent = await users.query()
  .orderBy('name')
  .limit(20)
  .getMany();

// Или фабрика для работы с несколькими сущностями:
const manager = new YdbEntityManager();
const postsRepo = manager.getRepository(PostEntity);
```

Полный список методов — в разделе [Репозиторий](repository.md).

## Следующие шаги

- [Сущности и декораторы](entity.md) — все декораторы и типы колонок
- [Active Record](active-record.md) — статические методы, QueryOptions, ленивая дешифровка
- [Репозиторий](repository.md) — паттерны репозитория
- [Транзакции](transactions.md) — атомарные операции и их опции
- [AbortSignal](abort-signal.md) — отмена запросов и таймауты
- [Особые случаи](edge-cases.md) — подмена драйвера, логирование, ретраи
- [CLI](cli.md) — миграции и генерация кода из командной строки
- [Быстрый старт (NestJS)](nest/quick-start.md) — если вы используете NestJS
