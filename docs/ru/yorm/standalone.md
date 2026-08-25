# Использование без NestJS

ORM можно использовать в скриптах, Lambda-функциях или CLI-утилитах без NestJS. Для этого экспортируется набор функций для создания подключения и настройки сущностей.

## Создание подключения

```ts
import { createDriver, createExecutor, configureEntities } from '@ycforge/yorm';
import { UserEntity, PostEntity } from './entities/index.js';

const driver = await createDriver({
  endpoint: 'grpc://localhost:2136/local',
  auth_type: 'anonymous',
  authOptions: {},
});

const executor = createExecutor(driver, {
  endpoint: 'grpc://localhost:2136/local',
  auth_type: 'anonymous',
  authOptions: {},
});

// Настраиваем сущности: executor + провайдеры шифрования
configureEntities([UserEntity, PostEntity], {
  executor,
  // encryptionProvider: myEncryptionProvider,
  // blindIndexProvider: myBlindIndexProvider,
});

// Теперь можно использовать Active Record
const user = await UserEntity.find({ email: 'ivan@example.com' });

// Обязательно закрыть драйвер по завершении
driver.close();
```

Функции:

- `createDriver(opts, credentialsProvider?)` — создаёт и подключает драйвер (`driver.ready()`).
- `createExecutor(driver, opts)` — создаёт executor поверх драйвера (с учётом `poolOptions` и `logQueries`).
- `createCredentialsProvider(opts)` — создаёт credentials provider по `auth_type`.
- `configureEntities(entities, { executor, encryptionProvider?, blindIndexProvider? })` — устанавливает executor и провайдеры на сущности.

## Запросы напрямую

Можно выполнять произвольные YQL-запросы через executor:

```ts
const rows = await executor`
  SELECT * FROM users WHERE name = $name
  LIMIT 10
`.parameter('name', 'Иван');
```

## Транзакции без NestJS

```ts
await executor.transaction().execute(async (trx) => {
  const opts = { trx };
  await UserEntity.save(user, opts);
  await PostEntity.save(post, opts);
});
```

## Миграции в скрипте

```ts
import { YdbMigrationRunner, loadMigrationsFromDir } from '@ycforge/yorm';

const migrations = await loadMigrationsFromDir('./migrations');
const runner = new YdbMigrationRunner(executor);
await runner.run();
```

См. также [Миграции](migrations.md) и программный API.

## Логирование запросов

Передайте `logQueries: true` или экземпляр `QueryLogger` в опции:

```ts
const executor = createExecutor(driver, {
  endpoint: '...',
  auth_type: 'anonymous',
  authOptions: {},
  logQueries: true, // консольный логер по умолчанию
});
```

Кастомный логер реализует интерфейс `QueryLogger` — получает `QueryLogEntry` с SQL, замаскированными параметрами и длительностью.