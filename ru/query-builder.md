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
| `where(criteria)` | условия равенства (AND), объединяются с предыдущими |
| `andWhere(criteria)` | синоним `where()` для читаемости цепочек |
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

- Условия — только равенство (`=`), объединяются через `AND`. Повторное поле в `andWhere` перезаписывает предыдущее значение.
- Поля в `WHERE` и `ORDER BY` валидируются по метаданным сущности — неизвестное поле вызовет ошибку.
- Поддержка зашифрованных полей с blind index — такая же, как в `find`/`findAll`.