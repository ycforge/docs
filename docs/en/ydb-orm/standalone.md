# Standalone usage

You can use the ORM in scripts, Lambda functions, or CLI utilities without NestJS. A set of functions for creating a connection and configuring entities is exported for that.

## Creating a connection

```ts
import { createDriver, createExecutor, configureEntities } from '@ycforge/ydb-orm';
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

// Configure entities: executor + encryption providers
configureEntities([UserEntity, PostEntity], {
  executor,
  // encryptionProvider: myEncryptionProvider,
  // blindIndexProvider: myBlindIndexProvider,
});

// Now Active Record is available
const user = await UserEntity.find({ email: 'john@example.com' });

// Close the driver when done
driver.close();
```

Functions:

- `createDriver(opts, credentialsProvider?)` — creates and connects a driver (`driver.ready()`).
- `createExecutor(driver, opts)` — creates an executor on top of the driver (honors `poolOptions` and `logQueries`).
- `createCredentialsProvider(opts)` — creates a credentials provider by `auth_type`.
- `configureEntities(entities, { executor, encryptionProvider?, blindIndexProvider? })` — sets the executor and providers on entities.

## Direct queries

You can run arbitrary YQL queries via the executor:

```ts
const rows = await executor`
  SELECT * FROM users WHERE name = $name
  LIMIT 10
`.parameter('name', 'John');
```

## Transactions without NestJS

```ts
await executor.transaction().execute(async (trx) => {
  const opts = { trx };
  await UserEntity.save(user, opts);
  await PostEntity.save(post, opts);
});
```

## Migrations in a script

```ts
import { YdbMigrationRunner, loadMigrationsFromDir } from '@ycforge/ydb-orm';

const migrations = await loadMigrationsFromDir('./migrations');
const runner = new YdbMigrationRunner(executor);
await runner.run();
```

See also [Migrations](migrations.md) and the programmatic API.

## Query logging

Pass `logQueries: true` or a `QueryLogger` instance in the options:

```ts
const executor = createExecutor(driver, {
  endpoint: '...',
  auth_type: 'anonymous',
  authOptions: {},
  logQueries: true, // console logger by default
});
```

A custom logger implements the `QueryLogger` interface — it receives a `QueryLogEntry` with SQL, masked parameters, and duration.