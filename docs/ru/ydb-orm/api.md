# Справочник API

## Основные модули

| Экспорт | Тип | Описание |
| --- | --- | --- |
| `YdbCoreModule` | класс | корневой модуль NestJS: `forRootAsync()` |
| `YdbOrmModule` | класс | модуль NestJS: `forRoot()`, `forFeature([...])` |
| `YdbBaseEntity` | класс | базовый класс Active Record сущности |
| `YdbQueryBuilder` | класс | цепочный построитель запросов |
| `YdbRepository<T>` | класс | DI-репозиторий для сущности |
| `YdbEntityManager` | класс | фабрика репозиториев |

## Декораторы

| Экспорт | Тип | Описание |
| --- | --- | --- |
| `YdbEntity(tableName)` | класс | имя таблицы + регистрация в реестре |
| `YdbColumn(type)` | свойство | YDB-тип колонки |
| `YdbPrimaryColumn(type)` | свойство | колонка первичного ключа (составной PK поддерживается) |
| `YdbEncrypted(options?)` | свойство | шифруемое поле (`blindIndex`, `aadOverride`, `lazy`) |
| `YdbSecurityAAD()` | свойство | поле участвует в AAD |
| `OneToMany(target, joinColumn)` | свойство | связь один-ко-многим |
| `ManyToOne(target, joinColumn)` | свойство | связь многие-к-одному |
| `OneToOne(target, joinColumn)` | свойство | связь один-к-одному |
| `ManyToMany(target, inverseSide?)` | свойство | связь многие-ко-многим |
| `JoinTable(tableName, options?)` | свойство | join-таблица для many-to-many |
| `EagerLoad([...])` | класс | eager-загрузка relations без N+1 |
| `YdbIndex(options)` | класс | вторичный индекс (GLOBAL SYNC) |
| `YdbEnum(options)` | свойство | enum-колонка (`Utf8`/`Int32`) |
| `YdbCreateDateColumn()` | свойство | автопроставление даты создания |
| `YdbUpdateDateColumn()` | свойство | автопроставление даты обновления |
| `YdbTtl(options)` | класс | TTL таблицы (один раз на класс) |
| `BeforeInsert` / `AfterInsert` | метод | lifecycle-хуки вставки |
| `BeforeUpdate` | метод | lifecycle-хук обновления |
| `AfterFind` | метод | lifecycle-хук чтения |
| `BeforeRemove` | метод | lifecycle-хук удаления |
| `getLifecycleHooks(target)` | функция | метаданные lifecycle-хуков класса |

## Active Record (статические методы `YdbBaseEntity`)

| Метод | Описание |
| --- | --- |
| `find(where, options?)` | одна запись или `null` |
| `findOneBy(where, options?)` | алиас `find` |
| `findAll(where?, options?)` | список записей |
| `findBy(where?, options?)` | алиас `findAll` |
| `count(where?, options?)` | количество строк |
| `save(entity, options?)` | insert или update по uuid |
| `insertMany(entities, options?)` | массовая вставка батчами по 100 |
| `delete(pkValue, options?)` | удаление по PK (`RETURNING *`) |
| `updateBy(where, patch, options?)` | массовое обновление по условию |
| `deleteBy(where, options?)` | массовое удаление по условию |
| `query()` | создать QueryBuilder |

Методы экземпляра: `loadRelations([...])`, `decryptField(name)`, `decryptLazyFields()`, `toJSON()`.

## Репозиторий / EntityManager

| Экспорт | Описание |
| --- | --- |
| `YdbRepository<T>` | DI-репозиторий сущности (CRUD + relations) |
| `YdbEntityManager` | фабрика репозиториев (`getRepository(Entity)`) |
| `InjectRepository(Entity)` | декоратор инъекции репозитория в NestJS |
| `getRepositoryToken(Entity)` | DI-токен репозитория |
| `getOrCreateRepository(Entity)` | получить/создать репозиторий из runtime deps |

## Persistence / Relations (ядро ORM)

| Экспорт | Описание |
| --- | --- |
| `YdbEntityPersistence<T>` | CRUD, шифрование, lifecycle hooks, enum/timestamp-конвертация |
| `YdbEntityRelations<T>` | eager loading, lazy `loadRelations`, many-to-many |

## Типы

| Экспорт | Описание |
| --- | --- |
| `YdbPrimitive` | тип YDB-примитива (`Uuid`, `Utf8`, ...) |
| `YdbModuleOptions` | опции подключения и модуля |
| `YdbModuleAsyncOptions` | async-опции (`useFactory`/`useClass`/`useExisting`) |
| `YdbExecutor` | executor запросов (транзакции, параметры) |
| `YdbQuery` | объект запроса (`parameter`, `timeout`, `signal`, `cancel`) |
| `QueryOptions` | `trx`, `timeout`, `signal`, `limit`, `offset`, `select` |
| `YdbEncryptionProvider` | интерфейс шифрования (`encrypt`/`decrypt`) |
| `YdbBlindIndexProvider` | интерфейс blind index (`hash`) |
| `YdbEncryptionContext` | контекст шифрования |
| `YdbValidationProvider` | интерфейс валидации (`validate`) |
| `YdbMigration` / `YdbMigrationClass` | интерфейс миграции |
| `BuiltQuery` / `OrderDirection` | типы QueryBuilder |
| `YdbEntityConstructor<T>` | тип конструктора сущности |
| `PersistenceDeps` | зависимости persistence (провайдеры, uuid-генератор) |
| `RelationsDeps` | зависимости relations (провайдеры шифрования) |
| `LifecycleHooks` | тип метаданных lifecycle-хуков |

## DI-токены

| Токен | Описание |
| --- | --- |
| `YDB_DRIVER` | драйвер YDB |
| `YDB_QUERY` | executor запросов |
| `YDB_OPTIONS` | опции модуля |
| `YDB_CREDENTIALS_PROVIDER` | credentials provider |
| `YDB_ENCRYPTION_PROVIDER` | провайдер шифрования |
| `YDB_BLIND_INDEX_PROVIDER` | провайдер blind index |
| `YDB_SCHEMA_SYNC` | синхронизатор схемы |

## Подключение без NestJS

| Экспорт | Описание |
| --- | --- |
| `createDriver(opts, credentialsProvider?)` | создаёт и подключает драйвер |
| `createExecutor(driver, opts)` | создаёт executor |
| `createCredentialsProvider(opts)` | credentials provider по `auth_type` |
| `validateYdbModuleOptions(opts)` | fail-fast валидация опций модуля |
| `configureEntities(entities, opts)` | настраивает сущности |
| `AuthKeyCredentialsProvider` | обмен JWT → IAM через `fetch` |

## Миграции

| Экспорт | Описание |
| --- | --- |
| `YdbMigrationRunner` | runner миграций (run/revert/status) |
| `loadMigrationsFromDir(dir)` | загрузка миграций из директории |
| `planMigration(expected, existing)` | чистый план миграции по diff |
| `renderMigrationFile(...)` | рендер файла миграции |
| `executeSql(executor, sql)` | выполнить SQL-выражение |
| `MIGRATIONS_TABLE` | имя таблицы учёта (`ydb_migrations`) |

## Schema sync

| Экспорт | Описание |
| --- | --- |
| `YdbSchemaSyncer` | синхронизатор схемы (`verify`/`sync`) |
| `buildExpectedTableSchema(entity)` | ожидаемая схема по сущности |
| `generateCreateTableYql(schema)` | DDL CREATE TABLE |
| `generateAddColumnsYql(table, cols)` | DDL ALTER TABLE ADD COLUMN |
| `checkTableSchema(expected, actual)` | проверка схемы |

## Прочее

| Экспорт | Описание |
| --- | --- |
| `mapToYdb(type, value)` | маппинг значения в тип YDB |
| `quoteIdentifier(name)` | экранирование идентификатора |
| `validateIdentifier(name)` | проверка идентификатора |
| `getYdbEntityMetadata(entity)` | метаданные сущности |
| `validateEntityMetadata(entity, ctx)` | валидация метаданных сущности |
| `getRegisteredYdbEntities()` | зарегистрированные сущности |
| `registerYdbEntity(entity)` | регистрация сущности вручную |
| `ConsoleQueryLogger` | консольный логер запросов |
| `wrapExecutorWithLogging(exec, logger)` | обёртка executor с логированием |
| `Base64TestEncryptionProvider` | тестовый провайдер шифрования (заглушка) |
| `ClassValidatorProvider` | валидатор через class-validator |

{% note info %}

Готовые провайдеры шифрования и blind index для production (Yandex Cloud KMS, HMAC-SHA256) находятся в отдельном пакете `@ycforge/orm-security-providers`.

{% endnote %}