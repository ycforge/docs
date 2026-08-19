# Entities & decorators

Entities are classes that describe YDB tables. They extend `YdbBaseEntity` and are decorated with a set of decorators. Metadata is collected via `reflect-metadata` and cached.

## YDB column types

The column type is a YDB primitive string:

| Type | Description |
| --- | --- |
| `Uuid` | 128-bit UUID |
| `Utf8` | UTF-8 string |
| `Bytes` | binary data (raw bytes) |
| `Int32` | 32-bit integer |
| `Int64` | 64-bit integer |
| `Bool` | boolean |
| `Double` | double-precision float |
| `Float` | single-precision float |
| `Date` | date |
| `Datetime` | date and time |
| `Timestamp` | timestamp |

## @YdbEntity('table_name')

Class decorator: sets the table name and registers the class in the global entity registry (used by schema sync).

```ts
@YdbEntity('users')
export class UserEntity extends YdbBaseEntity {
  // ...
}
```

## @YdbColumn(type)

Property decorator: sets the YDB column type.

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

Primary key decorator. Composite primary keys are supported — just declare several of these columns.

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

If no primary key is declared, the `uuid` column (Uuid) is used by default. It is generated automatically on insert (UUID v7 by default, configurable via the `uuidVersion` option).

{% endnote %}

## @YdbEncrypted({ blindIndex })

Marks a field as encrypted. The value is encrypted before writing and decrypted after reading.

- `blindIndex: true` (default) — adds a synthetic `{field}_bi` column for searching by value hash.
- `blindIndex: false` — stores only ciphertext; searching by the field is impossible.
- `aadOverride` — a string used as AAD instead of the `@YdbSecurityAAD` fields (needed for bulk `updateBy` operations).

Ciphertext is stored in a `Bytes` column (raw bytes). The type from `@YdbColumn` is ignored for such fields, so there is no need to declare it.

```ts
@YdbEntity('users')
export class UserEntity extends YdbBaseEntity {
  @YdbPrimaryColumn('Uuid')
  uuid: string;

  // In DB: ciphertext + email_bi column (hash for search)
  @YdbEncrypted({ blindIndex: true })
  email: string;

  // In DB: only ciphertext, search is impossible
  @YdbEncrypted({ blindIndex: false })
  government_id: string;
}
```

See [Field encryption](encryption.md) for details.

## @YdbSecurityAAD()

An unencrypted field that participates in AAD (Additional Authenticated Data) when encrypting other entity fields. If such a field changes, existing ciphertexts stop decrypting.

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

Relations are described by decorator pairs on both sides of the association. See [Relations](relations.md) for details.

```ts
// One entity → many entities
@OneToMany(() => PostEntity, (post) => post.user_uuid)
posts?: PostEntity[];

// Many entities → one entity
@ManyToOne(() => UserEntity, (post) => post.user_uuid)
user?: UserEntity;

// One to one
@OneToOne(() => ProfileEntity, (user) => user.profile_uuid)
profile?: ProfileEntity;

// Many to many (requires @JoinTable on the owning side)
@ManyToMany(() => Tag, (tag) => tag.photos)
@JoinTable('photo_tag')
tags: Tag[];
```

## @EagerLoad([...relations])

Class decorator: automatically loads the listed relations on `find` / `findAll` with a single batch query `WHERE column IN (...)` — no N+1.

```ts
@YdbEntity('users')
@EagerLoad(['posts', 'profile'])
export class UserEntity extends YdbBaseEntity {
  // ...
}
```

## @YdbIndex({ columns, name?, unique? })

Class decorator (can be used multiple times) — a declarative secondary index (GLOBAL SYNC). It is included in `CREATE TABLE` during schema sync and in `migration:generate`.

- `columns` — index columns (order matters).
- `name` — explicit name; defaults to `{table}__{col1}_{col2}`.
- `unique` — unique index (default `false`).

```ts
@YdbEntity('photos')
@YdbIndex({ columns: ['author_email_bi'] })
@YdbIndex({ columns: ['is_public', 'rating'], name: 'photos__public_rating' })
export class PhotoEntity extends YdbBaseEntity {
  // ...
}
```

## @YdbEnum({ values, storage })

Enum column decorator: maps an enum to `Utf8` (string value) or `Int32` (ordinal number).

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

## @YdbCreateDateColumn() / @YdbUpdateDateColumn()

Automatically populate timestamps on insert and update.

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

The `@YdbUpdateDateColumn` column is also populated by `updateBy` and `insertMany`.

{% endnote %}

## @YdbTtl({ interval, column? })

Declarative table TTL (YDB table TTL). Generates `TTL = Interval(...) ON column` in `CREATE TABLE`. Can only be applied once per class.

```ts
@YdbEntity('sessions')
@YdbTtl({ interval: 'PT2H' })
export class SessionEntity extends YdbBaseEntity {
  @YdbPrimaryColumn('Uuid')
  uuid: string;
  // ...
}
```

## Lifecycle hooks

Method decorators: `@BeforeInsert`, `@AfterInsert`, `@BeforeUpdate`, `@AfterFind`. Hooks run sequentially and are awaited.

```ts
import {
  YdbBaseEntity,
  YdbEntity,
  YdbPrimaryColumn,
  BeforeInsert,
  AfterFind,
} from '@ycforge/ydb-orm';

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

`@BeforeUpdate` works on an instance, so `updateBy` (bulk operation) does not trigger it.

{% endnote %}

## Field validation

The ORM can validate entities before writing via `class-validator`. See [Validation](validation.md).