# API reference

## Core modules

| Export | Type | Description |
| --- | --- | --- |
| `YdbCoreModule` | class | NestJS root module: `forRootAsync()` |
| `YdbModule` | class | NestJS module: `forRoot()`, `forFeature([...])` |
| `YdbBaseEntity` | class | Active Record base entity class |
| `YdbQueryBuilder` | class | chainable query builder |
| `YdbRepository<T>` | class | DI repository for an entity |
| `YdbEntityManager` | class | repository factory |

## Decorators

| Export | Type | Description |
| --- | --- | --- |
| `YdbEntity(tableName)` | class | table name + registry registration |
| `YdbColumn(type)` | property | YDB column type |
| `YdbPrimaryColumn(type)` | property | primary key column (composite PK supported) |
| `YdbEncrypted(options?)` | property | encrypted field (`blindIndex`, `aadOverride`, `lazy`) |
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
| `BeforeRemove` | method | remove lifecycle hook |
| `getLifecycleHooks(target)` | function | lifecycle hook metadata of a class |

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

Instance methods: `loadRelations([...])`, `decryptField(name)`, `decryptLazyFields()`, `toJSON()`.

## Repository / EntityManager

| Export | Description |
| --- | --- |
| `YdbRepository<T>` | DI repository for an entity (CRUD + relations) |
| `YdbEntityManager` | repository factory (`getRepository(Entity)`) |
| `InjectRepository(Entity)` | decorator to inject a repository in NestJS |
| `getRepositoryToken(Entity)` | DI token for a repository |
| `getOrCreateRepository(Entity)` | get/create repository from runtime deps |

## Persistence / Relations (ORM core)

| Export | Description |
| --- | --- |
| `YdbEntityPersistence<T>` | CRUD, encryption, lifecycle hooks, enum/timestamp conversion |
| `YdbEntityRelations<T>` | eager loading, lazy `loadRelations`, many-to-many |

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
| `YdbEntityConstructor<T>` | entity constructor type |
| `PersistenceDeps` | persistence dependencies (providers, uuid generator) |
| `RelationsDeps` | relations dependencies (encryption providers) |
| `LifecycleHooks` | lifecycle hook metadata type |

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
| `validateYdbModuleOptions(opts)` | fail-fast validation of module options |
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

{% note info %}

Production-ready encryption and blind index providers (Yandex Cloud KMS, HMAC-SHA256) live in the separate package `@ycforge/orm-security-providers`.

{% endnote %}