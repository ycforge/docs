# API reference

## Core modules

| Export | Type | Description |
| --- | --- | --- |
| `YdbCoreModule` | class | NestJS root module: `forRootAsync()` |
| `YdbModule` | class | NestJS module: `forRoot()`, `forFeature([...])` |
| `YdbBaseEntity` | class | Active Record base entity class |
| `YdbQueryBuilder` | class | chainable query builder |

## Decorators

| Export | Type | Description |
| --- | --- | --- |
| `YdbEntity(tableName)` | class | table name + registry registration |
| `YdbColumn(type)` | property | YDB column type |
| `YdbPrimaryColumn(type)` | property | primary key column (composite PK supported) |
| `YdbEncrypted(options?)` | property | encrypted field (`blindIndex`, `aadOverride`) |
| `YdbSecurityAAD()` | property | field participates in AAD |
| `OneToMany(target, joinColumn)` | property | one-to-many relation |
| `ManyToOne(target, joinColumn)` | property | many-to-one relation |
| `OneToOne(target, joinColumn)` | property | one-to-one relation |
| `ManyToMany(target, inverseSide?)` | property | many-to-many relation |
| `JoinTable(tableName, options?)` | property | join table for many-to-many |
| `EagerLoad([...])` | class | eager-load relations without N+1 |
| `YdbIndex(options)` | class | secondary index (GLOBAL SYNC) |
| `YdbEnum(options)` | property | enum column (`Utf8`/`Int32`) |
| `YdbCreateDateColumn()` | property | auto-fill creation date |
| `YdbUpdateDateColumn()` | property | auto-fill update date |
| `YdbTtl(options)` | class | table TTL (once per class) |
| `BeforeInsert` / `AfterInsert` | method | insert lifecycle hooks |
| `BeforeUpdate` | method | update lifecycle hook |
| `AfterFind` | method | find lifecycle hook |

## Active Record (static methods of `YdbBaseEntity`)

| Method | Description |
| --- | --- |
| `find(where, options?)` | one record or `null` |
| `findOneBy(where, options?)` | alias of `find` |
| `findAll(where?, options?)` | list of records |
| `findBy(where?, options?)` | alias of `findAll` |
| `count(where?, options?)` | row count |
| `save(entity, options?)` | insert or update by uuid |
| `insertMany(entities, options?)` | bulk insert in batches of 100 |
| `delete(pkValue, options?)` | delete by PK (`RETURNING *`) |
| `updateBy(where, patch, options?)` | bulk update by condition |
| `deleteBy(where, options?)` | bulk delete by condition |
| `query()` | create a QueryBuilder |

Instance methods: `loadRelations([...])`, `toJSON()`.

## Types

| Export | Description |
| --- | --- |
| `YdbPrimitive` | YDB primitive type (`Uuid`, `Utf8`, ...) |
| `YdbModuleOptions` | connection and module options |
| `YdbModuleAsyncOptions` | async options (`useFactory`/`useClass`/`useExisting`) |
| `YdbExecutor` | query executor (transactions, parameters) |
| `YdbQuery` | query object (`parameter`, `timeout`, `signal`, `cancel`) |
| `QueryOptions` | `trx`, `timeout`, `signal`, `limit`, `offset`, `select` |
| `YdbEncryptionProvider` | encryption interface (`encrypt`/`decrypt`) |
| `YdbBlindIndexProvider` | blind index interface (`hash`) |
| `YdbEncryptionContext` | encryption context |
| `YdbValidationProvider` | validation interface (`validate`) |
| `YdbMigration` / `YdbMigrationClass` | migration interface |
| `BuiltQuery` / `OrderDirection` | QueryBuilder types |

## DI tokens

| Token | Description |
| --- | --- |
| `YDB_DRIVER` | YDB driver |
| `YDB_QUERY` | query executor |
| `YDB_OPTIONS` | module options |
| `YDB_CREDENTIALS_PROVIDER` | credentials provider |
| `YDB_ENCRYPTION_PROVIDER` | encryption provider |
| `YDB_BLIND_INDEX_PROVIDER` | blind index provider |
| `YDB_SCHEMA_SYNC` | schema syncer |

## Standalone usage

| Export | Description |
| --- | --- |
| `createDriver(opts, credentialsProvider?)` | creates and connects a driver |
| `createExecutor(driver, opts)` | creates an executor |
| `createCredentialsProvider(opts)` | credentials provider by `auth_type` |
| `configureEntities(entities, opts)` | configures entities |
| `AuthKeyCredentialsProvider` | JWT → IAM exchange via `fetch` |

## Migrations

| Export | Description |
| --- | --- |
| `YdbMigrationRunner` | migration runner (run/revert/status) |
| `loadMigrationsFromDir(dir)` | load migrations from a directory |
| `planMigration(expected, existing)` | pure migration plan by diff |
| `renderMigrationFile(...)` | render a migration file |
| `executeSql(executor, sql)` | execute an SQL statement |
| `MIGRATIONS_TABLE` | tracking table name (`ydb_migrations`) |

## Schema sync

| Export | Description |
| --- | --- |
| `YdbSchemaSyncer` | schema syncer (`verify`/`sync`) |
| `buildExpectedTableSchema(entity)` | expected schema from an entity |
| `generateCreateTableYql(schema)` | CREATE TABLE DDL |
| `generateAddColumnsYql(table, cols)` | ALTER TABLE ADD COLUMN DDL |
| `checkTableSchema(expected, actual)` | schema check |

## Misc

| Export | Description |
| --- | --- |
| `mapToYdb(type, value)` | map a value to a YDB type |
| `quoteIdentifier(name)` | quote an identifier |
| `validateIdentifier(name)` | validate an identifier |
| `getYdbEntityMetadata(entity)` | entity metadata |
| `validateEntityMetadata(entity, ctx)` | validate entity metadata |
| `getRegisteredYdbEntities()` | registered entities |
| `registerYdbEntity(entity)` | manually register an entity |
| `ConsoleQueryLogger` | console query logger |
| `wrapExecutorWithLogging(exec, logger)` | wrap an executor with logging |
| `Base64TestEncryptionProvider` | test encryption provider (stub) |
| `ClassValidatorProvider` | validator via class-validator |