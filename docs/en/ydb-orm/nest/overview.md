# NestJS adapter

`@ycforge/ydb-orm` ships a NestJS adapter via the `@ycforge/ydb-orm/nest` subpackage. It provides DI integration: module registration, injectable repositories, transaction manager, and schema sync provider.

## Install peer dependencies

```bash
yarn add @nestjs/common @nestjs/core reflect-metadata rxjs
```

## Module structure

| Module | Purpose |
| --- | --- |
| `YdbCoreModule.forRootAsync(...)` | Root module: driver, executor, auth, encryption providers |
| `YdbOrmModule.forFeature([...entities])` | Feature module: registers injectable repositories |

## DI tokens

| Token | What it provides |
| --- | --- |
| `YDB_DRIVER` | YDB driver |
| `YDB_QUERY` | query executor |
| `YDB_OPTIONS` | module options |
| `YDB_CREDENTIALS_PROVIDER` | credentials provider |
| `YDB_ENCRYPTION_PROVIDER` | encryption provider |
| `YDB_BLIND_INDEX_PROVIDER` | blind index provider |
| `YDB_SCHEMA_SYNC` | schema syncer |

## What changes vs standalone

| Concern | Standalone | NestJS |
| --- | --- | --- |
| Connection | `createDriver` + `createExecutor` | `YdbCoreModule.forRootAsync(...)` |
| Entity setup | `configureEntities([Entity], { executor })` | `YdbOrmModule.forFeature([Entity])` |
| Repository | `getOrCreateRepository(Entity)` | `@InjectRepository(Entity)` |
| Transaction | `new YdbTransactionManager(executor)` | injected `YdbTransactionManager` |
| Schema sync | `new YdbSchemaSyncer(executor)` | `@Inject(YDB_SCHEMA_SYNC)` |

## Example: minimal NestJS app

```ts
import { Module } from '@nestjs/common';
import { YdbCoreModule, YdbOrmModule } from '@ycforge/ydb-orm/nest';
import { createAuth, authKeyFromFile } from '@ycforge/auth';
import { UserEntity } from './user.entity';

@Module({
  imports: [
    YdbCoreModule.forRootAsync({
      useFactory: () => ({
        endpoint: 'grpcs://ydb.serverless.yandexcloud.net:2135/?database=/ru-central1/.../...',
        auth: createAuth(authKeyFromFile('./authorized_key.json')),
        sync: true,
      }),
    }),
    YdbOrmModule.forFeature([UserEntity]),
  ],
})
export class AppModule {}
```

## Next steps

- [NestJS quick start](quick-start.md) — step-by-step NestJS guide
- [NestJS repository injection](repository.md) — DI repository pattern
- [NestJS transactions](transactions.md) — injected transaction manager
- [NestJS schema sync](schema-sync.md) — DI schema syncer
- [NestJS encryption](encryption.md) — passing providers via module options
- [NestJS authentication](authentication.md) — auth via module options
