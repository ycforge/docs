# Quick start

A quick start for a regular Node.js application — no frameworks required. For NestJS see [NestJS quick start](nest/quick-start.md).

## 1. Create a project

```bash
mkdir myapp && cd myapp
yarn init -y
yarn add @ycforge/ydb-orm reflect-metadata
```

The package is ESM-only (`"type": "module"`), Node.js >= 22 is required. Make sure `package.json` contains `"type": "module"`, and `tsconfig.json` enables decorators:

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

Import `reflect-metadata` once at the application entry point.

## 2. Define an entity

Each entity is a class extending `YdbBaseEntity` and decorated with `@YdbEntity('table_name')`.

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

## 3. Connect to YDB and configure entities

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

  // Wire executor (and optional encryption/validation providers) into entities.
  // After this call Active Record static methods are available.
  configureEntities([UserEntity], { executor });

  return { driver, executor };
}
```

## 4. Use Active Record

```ts
// src/main.ts
import 'reflect-metadata';
import { initDb } from './db.js';
import { UserEntity } from './user.entity.js';

const { driver } = await initDb();

try {
  // Insert (uuid is generated automatically)
  const user = new UserEntity();
  user.name = 'John';
  user.email = 'john@example.com';
  await UserEntity.save(user);

  // Read
  const found = await UserEntity.find({ uuid: user.uuid });
  console.log(found?.name); // 'John'

  // Update (save with a filled uuid)
  user.name = 'John Doe';
  await UserEntity.save(user);

  // List with pagination
  const page = await UserEntity.findAll({}, { limit: 50, offset: 0 });

  // Count
  const total = await UserEntity.count({});

  // Delete
  await UserEntity.delete(user.uuid);
} finally {
  driver.close();
}
```

Run it:

```bash
node --experimental-strip-types src/main.ts
# or compile with tsc and run dist/main.js
```

## 5. Schema sync (development)

During development the ORM can create missing tables and columns automatically:

```ts
import { YdbSchemaSyncer } from '@ycforge/ydb-orm';

const syncer = new YdbSchemaSyncer(executor);
await syncer.sync([UserEntity]);
```

{% note warning %}

For production use [migrations](migrations.md) instead of schema sync.

{% endnote %}

## 6. Repository (optional)

`getOrCreateRepository` creates a `YdbRepository<Entity>` from the configured runtime dependencies. It keeps data-access logic in one place and is easier to mock in tests.

```ts
import { getOrCreateRepository, YdbEntityManager } from '@ycforge/ydb-orm';

const users = getOrCreateRepository(UserEntity);
const user = await users.findOneBy({ uuid });

// QueryBuilder through the repository
const recent = await users.query()
  .orderBy('name')
  .limit(20)
  .getMany();

// Or a factory for working with multiple entities:
const manager = new YdbEntityManager();
const postsRepo = manager.getRepository(PostEntity);
```

See [Repository](repository.md) for the full method list.

## Next steps

- [Entities & decorators](entity.md) — all decorators and column types
- [Active Record](active-record.md) — static methods, QueryOptions, lazy decryption
- [Repository](repository.md) — repository patterns
- [Transactions](transactions.md) — atomic operations and their options
- [AbortSignal](abort-signal.md) — cancellation and timeouts
- [Edge cases](edge-cases.md) — driver substitution, query logging, retries
- [CLI](cli.md) — migrations and code generation from the command line
- [NestJS quick start](nest/quick-start.md) — if you use NestJS
