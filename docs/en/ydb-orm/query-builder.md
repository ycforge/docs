# QueryBuilder

A chainable query builder on top of Active Record. It lets you compose SELECT queries with filtering, sorting, and pagination.

```ts
const photos = await PhotoEntity.query()
  .where({ is_public: true })
  .andWhere({ author_email: 'a@b.c' }) // encrypted + blind index — as in find
  .orderBy('rating', 'DESC')
  .addOrderBy('title')
  .limit(20)
  .offset(10)
  .getMany();
```

## Methods

| Method | Description |
| --- | --- |
| `where(criteria)` | conditions (AND), merged with previous ones |
| `andWhere(criteria)` | alias of `where()` for readable chains |
| `orWhere(criteria)` | merges criteria with previous ones via `OR` |
| `orderBy(field, dir?)` | ordering (`ASC`/`DESC`), replaces previous |
| `addOrderBy(field, dir?)` | additional ordering (for composite) |
| `select(columns)` | specific columns instead of `SELECT *` |
| `limit(n)` | max rows (default 100, max 1000) |
| `offset(n)` | offset |
| `options(QueryOptions)` | `trx`, `signal`, `timeout` |
| `getMany()` | run the query, return entities (with eager relations) |
| `execute()` | alias of `getMany()` |
| `getOne()` | first record or `null` |
| `getCount()` | `COUNT(*)` with the same conditions (no limit/offset/order) |
| `andWhereJsonExists(column, path)` | `JSON_EXISTS(column, path)` condition |
| `andWhereJsonValue(column, path, value)` | `JSON_VALUE(column, path) = value` condition (string comparison) |
| `toYql()` | build SQL and parameter values **without** executing |

## Examples

### Filtering and ordering

```ts
const recent = await PostEntity.query()
  .where({ is_published: true })
  .orderBy('created_at', 'DESC')
  .limit(10)
  .getMany();
```

### Pagination

```ts
const page = await PostEntity.query()
  .where({ author_uuid: userId })
  .orderBy('rating', 'DESC')
  .addOrderBy('created_at', 'DESC')
  .limit(20)
  .offset(40)
  .getMany();
```

### Inspect SQL without executing

```ts
const { sql, values } = await PostEntity.query()
  .where({ is_public: true })
  .toYql();

console.log(sql);   // SELECT * FROM `photos` WHERE `is_public` = $is_public LIMIT 100 OFFSET 0
console.log(values); // { is_public: true }
```

### Count

```ts
const count = await PhotoEntity.query().where({ is_public: true }).getCount();
```

### Comparison operators

`where()`/`andWhere()` accept operator objects:

```ts
const users = await UserEntity.query()
  .where({ age: { $gte: 18 } })
  .andWhere({ status: { $in: ['active', 'pending'] } })
  .andWhere({ name: { $like: 'Ivan%' } })
  .getMany();
```

Supported operators:

| Operator | Description |
| --- | --- |
| `$eq` | equality (default when using `{ field: value }`) |
| `$ne` | not equal; `null` becomes `IS NOT NULL` |
| `$gt`, `$gte`, `$lt`, `$lte` | comparisons |
| `$like` | `LIKE` (only `Utf8` columns) |
| `$in` | `IN (...)` |
| `$between` | `BETWEEN lo AND hi` |

### Logical groups `$or` and `$and`

```ts
const users = await UserEntity.query()
  .where({
    $or: [
      { balance: { $gte: 100 } },
      { is_admin: true },
    ],
    is_banned: false,
  })
  .getMany();
```

`$and` and `$or` accept an array of nested objects; groups can be nested. `orWhere()` adds a single criterion merged with previous ones via `OR`:

```ts
const users = await UserEntity.query()
  .where({ is_active: true })
  .orWhere({ role: 'admin' })
  .getMany();
```

### JSON conditions

```ts
const events = await EventEntity.query()
  .andWhereJsonExists('metadata', '$.settings.theme')
  .andWhereJsonValue('metadata', '$.role', 'admin')
  .getMany();
```

The same JSON checks can be written with object operators:

```ts
const events = await EventEntity.query()
  .where({
    metadata: { $jsonExists: '$.settings.theme' },
  })
  .andWhere({
    payload: { $jsonValue: { path: '$.role', equals: 'admin' } },
  })
  .getMany();
```

### Inside a transaction

```ts
await this.trxManager.runInTransaction(async (trx) => {
  const photos = await PhotoEntity.query()
    .where({ is_public: true })
    .options({ trx })
    .getMany();
});
```

## Limitations

- Conditions at the same level are joined with `AND`. A repeated field in `andWhere` overwrites the previous value.
- Fields in `WHERE` and `ORDER BY` are validated against the entity metadata — an unknown field raises an error.
- Encrypted fields with blind index are supported just like in `find`/`findAll`: only equality (`$eq` / raw value) is allowed; other operators on an encrypted field raise an error.