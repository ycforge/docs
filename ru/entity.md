# Сущности и декораторы

Сущности — это классы, которые описывают таблицы YDB. Они наследуют `YdbBaseEntity` и декорируются набором декораторов. Метаданные собираются через `reflect-metadata` и кешируются.

## Типы колонок YDB

Тип колонки задаётся строкой-примитивом YDB:

| Тип | Описание |
| --- | --- |
| `Uuid` | 128-битный UUID |
| `Utf8` | строка UTF-8 |
| `Bytes` | бинарные данные (raw bytes) |
| `Int32` | 32-битное целое |
| `Int64` | 64-битное целое |
| `Bool` | булево значение |
| `Double` | число с плавающей точкой |
| `Float` | число с плавающей точкой (32 бита) |
| `Date` | дата |
| `Datetime` | дата и время |
| `Timestamp` | метка времени |

## @YdbEntity('table_name')

Класс-декоратор: задаёт имя таблицы и регистрирует класс в глобальном реестре сущностей (используется schema sync).

```ts
@YdbEntity('users')
export class UserEntity extends YdbBaseEntity {
  // ...
}
```

## @YdbColumn(type)

Декоратор свойства: задаёт YDB-тип колонки.

```ts
@YdbEntity('posts')
export class PostEntity extends YdbBaseEntity {
  @YdbPrimaryColumn('Uuid')
  uuid: string;

  @YdbColumn('Utf8')
  title: string;

  @YdbColumn('Int32')
  rating: number;

  @YdbColumn('Bool')
  isPublic: boolean;
}
```

## @YdbPrimaryColumn(type)

Декоратор первичного ключа. Поддерживается составной первичный ключ — просто объявите несколько таких колонок.

```ts
@YdbEntity('user_roles')
export class UserRoleEntity extends YdbBaseEntity {
  @YdbPrimaryColumn('Uuid')
  user_uuid: string;

  @YdbPrimaryColumn('Utf8')
  role: string;
}
```

{% note info %}

Если первичный ключ не объявлен, по умолчанию используется колонка `uuid` (Uuid). Она генерируется автоматически при вставке (по умолчанию UUID v7, настраивается опцией `uuidVersion`).

{% endnote %}

## @YdbEncrypted({ blindIndex })

Помечает поле как шифруемое. Значение шифруется перед записью и расшифровывается после чтения.

- `blindIndex: true` (по умолчанию) — добавляет synthetic-колонку `{field}_bi` для поиска по хешу значения.
- `blindIndex: false` — хранится только ciphertext, поиск по полю невозможен.
- `aadOverride` — строка, которая будет использоваться как AAD вместо полей `@YdbSecurityAAD` (нужна при массовых операциях `updateBy`).

Шифротекст хранится в колонке `Bytes` (raw bytes). Тип из `@YdbColumn` для таких полей игнорируется, объявлять его не нужно.

```ts
@YdbEntity('users')
export class UserEntity extends YdbBaseEntity {
  @YdbPrimaryColumn('Uuid')
  uuid: string;

  // В БД: ciphertext + колонка email_bi (hash для поиска)
  @YdbEncrypted({ blindIndex: true })
  email: string;

  // В БД: только ciphertext, поиск невозможен
  @YdbEncrypted({ blindIndex: false })
  government_id: string;
}
```

Подробнее — в разделе [Шифрование полей](encryption.md).

## @YdbSecurityAAD()

Незашифрованное поле участвует в AAD (Additional Authenticated Data) при шифровании других полей сущности. Если такое поле изменится, существующие ciphertext'ы перестанут расшифровываться.

```ts
@YdbEntity('users')
export class UserEntity extends YdbBaseEntity {
  @YdbPrimaryColumn('Uuid')
  uuid: string;

  @YdbSecurityAAD()
  @YdbColumn('Utf8')
  organization: string;

  @YdbEncrypted()
  email: string;
}
```

## Relations

Отношения описываются парами декораторов на обеих сторонах связи. Подробнее — в разделе [Связи](relations.md).

```ts
// Одна сущность → много сущностей
@OneToMany(() => PostEntity, (post) => post.user_uuid)
posts?: PostEntity[];

// Много сущностей → одна сущность
@ManyToOne(() => UserEntity, (post) => post.user_uuid)
user?: UserEntity;

// Один к одному
@OneToOne(() => ProfileEntity, (user) => user.profile_uuid)
profile?: ProfileEntity;

// Многие ко многим (требует @JoinTable на владеющей стороне)
@ManyToMany(() => Tag, (tag) => tag.photos)
@JoinTable('photo_tag')
tags: Tag[];
```

## @EagerLoad([...relations])

Класс-декоратор: автоматически подгружает перечисленные relations при `find` / `findAll` одним batch-запросом `WHERE column IN (...)` — без N+1.

```ts
@YdbEntity('users')
@EagerLoad(['posts', 'profile'])
export class UserEntity extends YdbBaseEntity {
  // ...
}
```

## @YdbIndex({ columns, name?, unique? })

Класс-декоратор (можно несколько) — декларативный вторичный индекс (GLOBAL SYNC). Попадает в `CREATE TABLE` при schema sync и в `migration:generate`.

- `columns` — колонки индекса (порядок важен).
- `name` — явное имя; по умолчанию `{table}__{col1}_{col2}`.
- `unique` — уникальный индекс (по умолчанию `false`).

```ts
@YdbEntity('photos')
@YdbIndex({ columns: ['author_email_bi'] })
@YdbIndex({ columns: ['is_public', 'rating'], name: 'photos__public_rating' })
export class PhotoEntity extends YdbBaseEntity {
  // ...
}
```

## @YdbEnum({ values, storage })

Декоратор enum-колонки: маппинг enum ↔ `Utf8` (строковое значение) или `Int32` (порядковый номер).

```ts
enum Status {
  ACTIVE = 'active',
  INACTIVE = 'inactive',
}

@YdbEntity('sessions')
export class SessionEntity extends YdbBaseEntity {
  @YdbPrimaryColumn('Uuid')
  uuid: string;

  @YdbColumn('Utf8')
  @YdbEnum({ values: Object.values(Status), storage: 'Utf8' })
  status: Status;
}
```

## @YdbJson()

Декоратор свойства: колонка хранится как JSON-строка (`Utf8` в БД), автоматически сериализуется/десериализуется при записи/чтении.

```ts
@YdbEntity('users')
export class UserEntity extends YdbBaseEntity {
  @YdbPrimaryColumn('Uuid')
  uuid: string;

  @YdbJson()
  metadata: { role: string; settings: Record<string, any> };
}

const user = new UserEntity();
user.metadata = { role: 'admin', settings: { theme: 'dark' } };
await UserEntity.save(user);
// В БД: '{"role":"admin","settings":{"theme":"dark"}}'
```

## @YdbCreateDateColumn() / @YdbUpdateDateColumn()

Автоматически проставляют метки времени при вставке и обновлении.

```ts
@YdbEntity('users')
export class UserEntity extends YdbBaseEntity {
  @YdbPrimaryColumn('Uuid')
  uuid: string;

  @YdbCreateDateColumn()
  @YdbColumn('Timestamp')
  created_at: Date;

  @YdbUpdateDateColumn()
  @YdbColumn('Timestamp')
  updated_at: Date;
}
```

{% note info %}

Колонка из `@YdbUpdateDateColumn` проставляется также при `updateBy` и `insertMany`.

{% endnote %}

## @YdbTtl({ interval, column? })

Декларативный TTL таблицы (YDB table TTL). Генерирует `TTL = Interval(...) ON column` в `CREATE TABLE`. Можно применить только один раз на класс.

```ts
@YdbEntity('sessions')
@YdbTtl({ interval: 'PT2H' })
export class SessionEntity extends YdbBaseEntity {
  @YdbPrimaryColumn('Uuid')
  uuid: string;
  // ...
}
```

## Lifecycle-хуки

Метод-декораторы: `@BeforeInsert`, `@AfterInsert`, `@BeforeUpdate`, `@AfterFind`, `@BeforeRemove`. Хуки вызываются последовательно и ожидаются (`await`).

```ts
@YdbEntity('users')
export class UserEntity extends YdbBaseEntity {
  @YdbPrimaryColumn('Uuid')
  uuid: string;

  @BeforeInsert()
  setDefaults() {
    this.created_at = new Date();
  }

  @AfterFind()
  normalize() {
    this.name = this.name.trim();
  }
}
```

{% note warning %}

Хуки `@BeforeUpdate` и `@BeforeRemove` работают на инстансе, поэтому `updateBy` / `deleteBy` (массовые операции) их не вызывают.

{% endnote %}

## Валидация полей

ORM может валидировать сущности перед записью через `class-validator`. Подробнее — в разделе [Валидация](validation.md).