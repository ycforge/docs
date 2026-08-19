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

### Lazy loading

The instance method `loadRelations([...])` loads relations manually.

```ts
const post = await PostEntity.find({ uuid });
await post.loadRelations(['user']);
console.log(post.user?.name);
```

## Saving relations

The ORM does not manage cascades automatically — relations are saved through FK columns. Typically all records are created in a single [transaction](transactions.md):

```ts
await this.trxManager.runInTransaction(async (trx) => {
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