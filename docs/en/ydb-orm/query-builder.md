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
| `where(criteria)` | equality conditions (AND), merged with previous ones |
| `andWhere(criteria)` | alias of `where()` for readable chains |
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

- Conditions support equality only (`=`), joined with `AND`. A repeated field in `andWhere` overwrites the previous value.
- Fields in `WHERE` and `ORDER BY` are validated against the entity metadata — an unknown field raises an error.
- Encrypted fields with blind index are supported just like in `find`/`findAll`.