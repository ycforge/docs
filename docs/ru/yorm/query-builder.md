# QueryBuilder

Цепочный построитель запросов поверх Active Record. Позволяет собирать SELECT-запросы с фильтрацией, сортировкой и пагинацией.

```ts
const photos = await PhotoEntity.query()
  .where({ is_public: true })
  .andWhere({ author_email: 'a@b.c' }) // encrypted + blind index — как в find
  .orderBy('rating', 'DESC')
  .addOrderBy('title')
  .limit(20)
  .offset(10)
  .getMany();
```

## Методы

| Метод | Описание |
| --- | --- |
| `where(criteria)` | условия (AND), объединяются с предыдущими |
| `andWhere(criteria)` | синоним `where()` для читаемости цепочек |
| `orWhere(criteria)` | объединяет критерий с предыдущими через `OR` |
| `orderBy(field, dir?)` | сортировка (`ASC`/`DESC`), заменяет предыдущую |
| `addOrderBy(field, dir?)` | дополнительная сортировка (для составной) |
| `select(columns)` | конкретные колонки вместо `SELECT *` |
| `limit(n)` | максимум строк (по умолчанию 100, максимум 1000) |
| `offset(n)` | смещение |
| `options(QueryOptions)` | `trx`, `signal`, `timeout` |
| `getMany()` | выполнить запрос, вернуть сущности (с eager relations) |
| `execute()` | алиас `getMany()` |
| `getOne()` | первая запись или `null` |
| `getCount()` | `COUNT(*)` по тем же условиям (без limit/offset/order) |
| `andWhereJsonExists(column, path)` | условие `JSON_EXISTS(column, path)` |
| `andWhereJsonValue(column, path, value)` | условие `JSON_VALUE(column, path) = value` (сравнение как строка) |
| `toYql()` | собрать SQL и значения параметров **без** выполнения |

## Примеры

### Фильтрация и сортировка

```ts
const recent = await PostEntity.query()
  .where({ is_published: true })
  .orderBy('created_at', 'DESC')
  .limit(10)
  .getMany();
```

### Пагинация

```ts
const page = await PostEntity.query()
  .where({ author_uuid: userId })
  .orderBy('rating', 'DESC')
  .addOrderBy('created_at', 'DESC')
  .limit(20)
  .offset(40)
  .getMany();
```

### Просмотр SQL без выполнения

```ts
const { sql, values } = await PostEntity.query()
  .where({ is_public: true })
  .toYql();

console.log(sql);   // SELECT * FROM `photos` WHERE `is_public` = $is_public LIMIT 100 OFFSET 0
console.log(values); // { is_public: true }
```

### Подсчёт

```ts
const count = await PhotoEntity.query().where({ is_public: true }).getCount();
```

### Операторы сравнения

В `where()`/`andWhere()` можно использовать объекты-операторы:

```ts
const users = await UserEntity.query()
  .where({ age: { $gte: 18 } })
  .andWhere({ status: { $in: ['active', 'pending'] } })
  .andWhere({ name: { $like: 'Ivan%' } })
  .getMany();
```

Поддерживаемые операторы:

| Оператор | Описание |
| --- | --- |
| `$eq` | равенство (используется по умолчанию) |
| `$ne` | не равно, `null` превращается в `IS NOT NULL` |
| `$gt`, `$gte`, `$lt`, `$lte` | сравнения |
| `$like` | `LIKE` (только `Utf8`-колонки) |
| `$in` | `IN (...)` |
| `$between` | `BETWEEN lo AND hi` |

### Логические группы `$or` и `$and`

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

`$and` и `$or` принимают массив вложенных объектов; группы можно вкладывать друг в друга. Через `orWhere()` можно добавить одно условие, объединённое с предыдущими через `OR`:

```ts
const users = await UserEntity.query()
  .where({ is_active: true })
  .orWhere({ role: 'admin' })
  .getMany();
```

### JSON-условия

```ts
const events = await EventEntity.query()
  .andWhereJsonExists('metadata', '$.settings.theme')
  .andWhereJsonValue('metadata', '$.role', 'admin')
  .getMany();
```

Те же JSON-проверки можно записать через объектные операторы:

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

### Внутри транзакции

```ts
await this.trxManager.runInTransaction(async (trx) => {
  const photos = await PhotoEntity.query()
    .where({ is_public: true })
    .options({ trx })
    .getMany();
});
```

## Ограничения

- Условия на одном уровне объединяются через `AND`. Повторное поле в `andWhere` перезаписывает предыдущее значение.
- Поля в `WHERE` и `ORDER BY` валидируются по метаданным сущности — неизвестное поле вызовет ошибку.
- Поддержка зашифрованных полей с blind index — такая же, как в `find`/`findAll`: доступно только равенство (`$eq` / прямое значение); остальные операторы на зашифрованном поле вызовут ошибку.