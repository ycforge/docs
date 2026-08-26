# NestJS schema sync

In NestJS applications, schema sync is configured via `YdbCoreModule.forRootAsync()` and the syncer is available via DI.

## Enabling

Set `sync: true` in the `forRootAsync` options:

```ts
YdbCoreModule.forRootAsync({
  useFactory: () => ({
    endpoint: '...',
    auth: createAuth(authKeyFromFile('./authorized_key.json')),
    sync: true, // dev only!
  }),
});
```

{% note warning %}

In production, use [migrations](../migrations.md) instead of `sync: true`.

{% endnote %}

## Manual invocation

The `YDB_SCHEMA_SYNC` provider is exported by the root module. You can verify or apply the schema manually.

```ts
import { Inject, Injectable } from '@nestjs/common';
import { YDB_SCHEMA_SYNC, YdbSchemaSyncer } from '@ycforge/ydb-orm/nest';
import { UserEntity, PostEntity } from './entities';

@Injectable()
export class SchemaService {
  constructor(
    @Inject(YDB_SCHEMA_SYNC)
    private readonly syncer: YdbSchemaSyncer,
  ) {}

  // Verify without changes
  async check() {
    const issues = await this.syncer.verify([UserEntity, PostEntity]);
    console.log('Schema issues:', issues);
  }

  // Force sync
  async apply() {
    await this.syncer.sync([UserEntity, PostEntity]);
  }
}
```

## What sync does

On application startup, after the driver is created, the ORM does the following for every registered entity:

- **no table** → `CREATE TABLE`;
- **no columns** → `ALTER TABLE ADD COLUMN`;
- **extra columns** → warning in the log only (data is not deleted);
- **column type or PK mismatch** → error (YDB cannot alter a column type or PK; a migration is required).

Table description is obtained via the Table service `DescribeTable` (the query service does not return column metadata). Synthetic `{field}_bi` columns for blind index and many-to-many join tables are created too.

## Standalone alternative

If you don't use NestJS, see the [standalone schema sync](../schema-sync.md) guide with `new YdbSchemaSyncer(executor)`.
