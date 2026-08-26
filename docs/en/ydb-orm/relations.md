# Relations

The ORM supports four types of relations between entities: `@OneToMany`, `@ManyToOne`, `@OneToOne`, and `@ManyToMany`.

## Relation types

| Decorator | Description |
| --- | --- |
| `@OneToMany(target, joinColumn)` | one entity → many entities |
| `@ManyToOne(target, joinColumn)` | many entities → one entity |
| `@OneToOne(target, joinColumn)` | one to one |
| `@ManyToMany(target, inverseSide?)` | many to many (requires `@JoinTable`) |

`joinColumn` is the field name that holds the foreign key on the entity. It can be a string or an arrow function `(entity) => entity.field` — the function syntax is more convenient because it protects against typos.

## One-to-Many / Many-to-One

Example: a user has many posts, and a post has one author.

```ts
@YdbEntity('users')
export class UserEntity extends YdbBaseEntity {
  @YdbPrimaryColumn('Uuid')
  uuid: string;

  @YdbColumn('Utf8')
  name: string;

  // join column — FK on the PostEntity.user_uuid side
  @OneToMany(() => PostEntity, (post) => post.user_uuid)
  posts?: PostEntity[];
}

@YdbEntity('posts')
export class PostEntity extends YdbBaseEntity {
  @YdbPrimaryColumn('Uuid')
  uuid: string;

  @YdbColumn('Utf8')
  title: string;

  // join column — FK on the current entity: PostEntity.user_uuid
  @ManyToOne(() => UserEntity, (post) => post.user_uuid)
  user?: UserEntity;

  @YdbColumn('Uuid')
  user_uuid: string;
}
```

The FK column (`user_uuid`) is a regular column declared with `@YdbColumn('Uuid')`. It does not have to be a primary key.

## One-to-One

Example: a user has a profile. The owning side holds the FK.

```ts
@YdbEntity('users')
export class UserEntity extends YdbBaseEntity {
  @YdbPrimaryColumn('Uuid')
  uuid: string;

  // FK to Profile lives in UserEntity.profile_uuid
  @YdbColumn('Uuid')
  profile_uuid: string;

  @OneToOne(() => ProfileEntity, (user) => user.profile_uuid)
  profile?: ProfileEntity;
}

@YdbEntity('profiles')
export class ProfileEntity extends YdbBaseEntity {
  @YdbPrimaryColumn('Uuid')
  uuid: string;

  // inverse description (back-reference) — descriptive, not used for loading
  @OneToOne(() => UserEntity, (user) => user.profile_uuid)
  user?: UserEntity;

  @YdbColumn('Utf8')
  bio: string;
}
```

## Many-to-Many

The owning side is marked with `@JoinTable('join_table_name')`. The join table is included in schema sync and migrations automatically.

```ts
@YdbEntity('photos')
export class PhotoEntity extends YdbBaseEntity {
  @YdbPrimaryColumn('Uuid')
  uuid: string;

  @YdbColumn('Utf8')
  title: string;

  @ManyToMany(() => TagEntity, (tag) => tag.photos)
  @JoinTable('photo_tag')
  tags: TagEntity[];
}

@YdbEntity('tags')
export class TagEntity extends YdbBaseEntity {
  @YdbPrimaryColumn('Uuid')
  uuid: string;

  @YdbColumn('Utf8')
  name: string;

  // inverse side: does not create a separate join table
  @ManyToMany(() => PhotoEntity, (photo) => photo.tags)
  photos: PhotoEntity[];
}
```

By default the join table columns are `{ownerTable}_uuid` and `{inverseTable}_uuid`. They can be overridden in `@JoinTable` options:

```ts
@ManyToMany(() => TagEntity)
@JoinTable('photo_tag', {
  joinColumn: 'photo_id',
  inverseJoinColumn: 'tag_id',
})
tags: TagEntity[];
```

## Loading relations

### Eager loading

`@EagerLoad([...])` loads relations automatically on `find` / `findAll` with a single batch query `WHERE column IN (...)` — no N+1.

```ts
@YdbEntity('users')
@EagerLoad(['posts', 'profile'])
export class UserEntity extends YdbBaseEntity {
  // ...
}

const user = await UserEntity.find({ uuid });
// user.posts and user.profile are already populated, no extra queries
```

Eager loading supports **nested paths** (issue #16): a dot-separated path loads each relation level in batches, carrying the primary keys from the previous level forward. There is no N+1 at any depth — for N root entities and depth D the ORM runs one batch query per relation level.

```ts
@YdbEntity('photos')
@EagerLoad(['tags.owner'])
export class PhotoEntity extends YdbBaseEntity {
  @ManyToMany(() => TagEntity, (t) => t.photos)
  @JoinTable('photo_tag')
  tags: TagEntity[];

  // ...
}

const photo = await PhotoEntity.find({ uuid });
// photo.tags are populated, and every tag.owner is populated too
```

Each path segment must be a declared relation (`@OneToMany` / `@ManyToOne` / `@OneToOne` / `@ManyToMany`). An unknown segment throws before any SQL is executed. `afterFind` fires exactly once per hydrated entity, deepest level first.

### Lazy loading

The instance method `loadRelations([...])` loads relations manually.

```ts
const post = await PostEntity.find({ uuid });
await post.loadRelations(['user']);
console.log(post.user?.name);
```

## Filtering by related entities (#17)

In a WHERE clause you can use a relation property instead of a column, with an object of conditions on the related entity's columns. Root rows are filtered by the existence of at least one matching related row (`EXISTS` semantics):

```ts
// Users who have a 'Hello' post (one-to-many)
await UserEntity.findAll({ posts: { title: 'Hello' } });

// Multiple related predicates + plain root conditions are combined with AND
await UserEntity.findAll({
  name: 'Ivan',
  posts: { title: { $like: '%yql%' } },
});

// Logical groups can mix root columns and relations
await UserEntity.findAll({
  $or: [
    { uuid: someUuid },
    { posts: { user_uuid: otherUuid } },
  ],
});
```

The generated YQL is a semi-join via `IN` with an uncorrelated subquery (YQL core does not support correlated subqueries). Duplicate root rows are impossible, so no `JOIN`/`DISTINCT` is required:

| relation | condition |
| --- | --- |
| one-to-many | `root.pk IN (SELECT child.fk FROM target WHERE pred)` |
| many-to-one / one-to-one | `root.fk IN (SELECT target.pk FROM target WHERE pred)` |
| many-to-many | `root.pk IN (SELECT jt.owner FROM jt WHERE jt.inverse IN (SELECT target.pk FROM target WHERE pred))` |

An empty predicate `{ posts: {} }` means "has at least one related row". Relations can be nested: `{ user: { profile: { bio: 'dev' } } }`.

{% note warning %}

Filtering by related entities is allowed **only for non-encrypted columns**: `@YdbEncrypted` fields (including their blind-index `{field}_bi` columns) are forbidden in related predicates.

{% endnote %}

Limitations:

- paths are resolved strictly from relation metadata; arbitrary SQL fragments cannot be passed;
- unknown relation or column, undeclared join column, incompatible join column types, composite primary key on the join side and a missing `@JoinTable` for many-to-many are rejected with a clear error **before executing SQL**;
- all values are bound as query parameters;
- the filter works in every method sharing the WHERE pipeline (`find`, `findAll`, `count`, `updateBy`, `deleteBy`) and in the [QueryBuilder](query-builder.md), including `{ trx }` and ambient transactions.

## Saving relations

The ORM does not manage cascades automatically — relations are saved through FK columns. Typically all records are created in a single [transaction](transactions.md):

```ts
await txManager.runInTransaction(async (trx) => {
  const opts = { trx };

  const user = new UserEntity();
  user.name = 'John';
  await UserEntity.save(user, opts);

  const post = new PostEntity();
  post.title = 'Post';
  post.user_uuid = user.uuid; // FK
  await PostEntity.save(post, opts);
});
```