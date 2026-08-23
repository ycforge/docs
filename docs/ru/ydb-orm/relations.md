# Связи (relations)

ORM поддерживает четыре типа отношений между сущностями: `@OneToMany`, `@ManyToOne`, `@OneToOne` и `@ManyToMany`.

## Типы отношений

| Декоратор | Описание |
| --- | --- |
| `@OneToMany(target, joinColumn)` | одна сущность → много сущностей |
| `@ManyToOne(target, joinColumn)` | много сущностей → одна сущность |
| `@OneToOne(target, joinColumn)` | один к одному |
| `@ManyToMany(target, inverseSide?)` | многие ко многим (требует `@JoinTable`) |

`joinColumn` — имя поля на сущности, хранящего внешний ключ. Можно передать строкой или стрелочной функцией `(entity) => entity.field` — синтаксис функции удобнее, так как защищает от опечаток.

## One-to-Many / Many-to-One

Пример: пользователь имеет много постов, у поста один автор.

```ts
@YdbEntity('users')
export class UserEntity extends YdbBaseEntity {
  @YdbPrimaryColumn('Uuid')
  uuid: string;

  @YdbColumn('Utf8')
  name: string;

  // join column — FK на стороне PostEntity.user_uuid
  @OneToMany(() => PostEntity, (post) => post.user_uuid)
  posts?: PostEntity[];
}

@YdbEntity('posts')
export class PostEntity extends YdbBaseEntity {
  @YdbPrimaryColumn('Uuid')
  uuid: string;

  @YdbColumn('Utf8')
  title: string;

  // join column — FK на текущей сущности: PostEntity.user_uuid
  @ManyToOne(() => UserEntity, (post) => post.user_uuid)
  user?: UserEntity;

  @YdbColumn('Uuid')
  user_uuid: string;
}
```

FK-колонка (`user_uuid`) — обычная колонка, объявленная через `@YdbColumn('Uuid')`. Она не обязана быть первичным ключом.

## One-to-One

Пример: у пользователя есть профиль. Владеющая сторона хранит FK.

```ts
@YdbEntity('users')
export class UserEntity extends YdbBaseEntity {
  @YdbPrimaryColumn('Uuid')
  uuid: string;

  // FK на Profile живёт в UserEntity.profile_uuid
  @YdbColumn('Uuid')
  profile_uuid: string;

  @OneToOne(() => ProfileEntity, (user) => user.profile_uuid)
  profile?: ProfileEntity;
}

@YdbEntity('profiles')
export class ProfileEntity extends YdbBaseEntity {
  @YdbPrimaryColumn('Uuid')
  uuid: string;

  // inverse-описание (back-reference) — описательное, для загрузки не используется
  @OneToOne(() => UserEntity, (user) => user.profile_uuid)
  user?: UserEntity;

  @YdbColumn('Utf8')
  bio: string;
}
```

## Many-to-Many

Владеющая сторона помечается `@JoinTable('имя_join_таблицы')`. Join-таблица попадает в schema sync и миграции автоматически.

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

  // inverse-сторона: не создаёт отдельной join-таблицы
  @ManyToMany(() => PhotoEntity, (photo) => photo.tags)
  photos: PhotoEntity[];
}
```

По умолчанию колонки join-таблицы: `{ownerTable}_uuid` и `{inverseTable}_uuid`. Их можно переопределить в опциях `@JoinTable`:

```ts
@ManyToMany(() => TagEntity)
@JoinTable('photo_tag', {
  joinColumn: 'photo_id',
  inverseJoinColumn: 'tag_id',
})
tags: TagEntity[];
```

## Загрузка связей

### Eager-загрузка

`@EagerLoad([...])` подгружает relations автоматически при `find` / `findAll` одним batch-запросом `WHERE column IN (...)` — без N+1.

```ts
@YdbEntity('users')
@EagerLoad(['posts', 'profile'])
export class UserEntity extends YdbBaseEntity {
  // ...
}

const user = await UserEntity.find({ uuid });
// user.posts и user.profile уже заполнены, дополнительных запросов не было
```

### Ленивая загрузка

Метод экземпляра `loadRelations([...])` подгружает relations вручную.

```ts
const post = await PostEntity.find({ uuid });
await post.loadRelations(['user']);
console.log(post.user?.name);
```

## Фильтрация по связанным сущностям (#17)

В WHERE вместо колонки можно указать свойство-связь с объектом условий по колонкам связанной сущности. Корень фильтруется по наличию хотя бы одной подходящей связанной строки (семантика `EXISTS`):

```ts
// Пользователи, у которых есть роль 'admin' (one-to-many)
await UserEntity.findAll({ posts: { title: 'Hello' } });

// Несколько related-предикатов + обычные условия корня объединяются через AND
await UserEntity.findAll({
  name: 'Ivan',
  posts: { title: { $like: '%yql%' } }, // many-to-one/one-to-many — любой тип связи
});

// Логические группы смешивают колонки корня и связи
await UserEntity.findAll({
  $or: [
    { uuid: someUuid },
    { posts: { user_uuid: otherUuid } },
  ],
});
```

Генерируемый YQL — полуслияние `IN` с некоррелированным подзапросом (коррелированные подзапросы ядром YQL не поддерживаются). Дубликаты корневых строк при этом невозможны, поэтому `JOIN`/`DISTINCT` не нужны:

| связь | условие |
| --- | --- |
| one-to-many | `root.pk IN (SELECT child.fk FROM target WHERE pred)` |
| many-to-one / one-to-one | `root.fk IN (SELECT target.pk FROM target WHERE pred)` |
| many-to-many | `root.pk IN (SELECT jt.owner FROM jt WHERE jt.inverse IN (SELECT target.pk FROM target WHERE pred))` |

Пустой предикат `{ posts: {} }` означает «есть хотя бы одна связанная строка». Связи можно вкладывать: `{ user: { profile: { bio: 'dev' } } }`.

{% note warning %}

Фильтрация по связанным сущностям разрешена **только для нешифрованных колонок**: поля `@YdbEncrypted` (включая их blind-index-колонки `{field}_bi`) в related-предикатах запрещены.

{% endnote %}

Ограничения:

- пути резолвятся строго по метаданным связей; произвольные SQL-фрагменты передать нельзя;
- неизвестная связь или колонка, необъявленная join-колонка, несовместимые типы join-колонок, составной PK на стороне соединения и отсутствие `@JoinTable` у many-to-many отвергаются понятной ошибкой **до выполнения SQL**;
- все значения биндятся параметрами запроса;
- фильтр работает во всех методах с общим конвейером WHERE (`find`, `findAll`, `count`, `updateBy`, `deleteBy`) и в [QueryBuilder](query-builder.md), включая поддержку `{ trx }` и ambient-транзакций.

## Сохранение связей

ORM не управляет каскадами автоматически — связи сохраняются через FK-колонки. Обычно все записи создаются в одной [транзакции](transactions.md):

```ts
await this.trxManager.runInTransaction(async (trx) => {
  const opts = { trx };

  const user = new UserEntity();
  user.name = 'Иван';
  await UserEntity.save(user, opts);

  const post = new PostEntity();
  post.title = 'Пост';
  post.user_uuid = user.uuid; // FK
  await PostEntity.save(post, opts);
});
```