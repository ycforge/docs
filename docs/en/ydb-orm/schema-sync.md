# Schema sync

`schema sync` is the equivalent of `synchronize` in TypeORM: the ORM adjusts the database schema to match the metadata of all registered entities.

## Standalone usage

```ts
import { YdbSchemaSyncer } from '@ycforge/ydb-orm';
import { UserEntity, PostEntity } from './entities';

const syncer = new YdbSchemaSyncer(executor);

// Verify without changes
const issues = await syncer.verify([UserEntity, PostEntity]);
console.log('Schema issues:', issues);

// Apply changes (CREATE TABLE, ALTER TABLE ADD COLUMN)
await syncer.sync([UserEntity, PostEntity]);
```

{% note warning %}

In production, use [migrations](migrations.md) instead of `sync`.

{% endnote %}

## What sync does

After the driver is created, the ORM does the following for every registered entity:

- **no table** → `CREATE TABLE`;
- **no columns** → `ALTER TABLE ADD COLUMN`;
- **extra columns** → warning in the log only (data is not deleted);
- **column type or PK mismatch** → error (YDB cannot alter a column type or PK; a migration is required).

Table description is obtained via the Table service `DescribeTable` (the query service does not return column metadata). Synthetic `{field}_bi` columns for blind index and many-to-many join tables are created too.

## DDL generators

DDL generators are available in the public API — you can use them to build migrations manually:

```ts
import {
  buildExpectedTableSchema,
  generateCreateTableYql,
  generateAddColumnsYql,
  checkTableSchema,
} from '@ycforge/ydb-orm';

const expected = buildExpectedTableSchema(UserEntity);
const createYql = generateCreateTableYql(expected);
console.log(createYql);
```

## Requirements

Every entity must have a PK column (`@YdbPrimaryColumn` or `uuid`). Otherwise sync fails with a clear error.

## Next steps

- [NestJS schema sync](nest/schema-sync.md) — if you use NestJS, see the DI-injected pattern
