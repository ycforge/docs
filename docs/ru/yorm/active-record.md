# Active Record

Сущности наследуют `YdbBaseEntity` и получают статический API для работы с БД. Все методы — асинхронные.

{% note info %}

Рантайм-зависимости (executor, провайдеры шифрования) хранятся в `WeakMap` по классу — наследники не разделяют состояние родителя.

{% endnote %}

## Основные методы

### find(where, options?)

Возвращает одну запись или `null` (`SELECT ... LIMIT 1`). Требует хотя бы одно условие.

```ts
const user = await UserEntity.find({ uuid: '5ad91505-d4f6-4a81-ab65-9dbc68cf4ed5' });
if (user) {
  console.log(user.name);
}
```

### findAll(where?, options?)

Возвращает список записей. По умолчанию `limit: 100`, максимум `1000`, `offset: 0`.

```ts
const users = await UserEntity.findAll({ name: 'Иван' }, { limit: 50, offset: 0 });
const all = await UserEntity.findAll(); // все, но не больше 100 строк
```

### count(where?, options?)

Возвращает количество строк, удовлетворяющих условию.

```ts
const total = await UserEntity.count({ name: 'Иван' });
```

### save(entity, options?)

Сохраняет сущность:

- без `uuid` — вставка (`UPSERT`), `uuid` генерируется автоматически;
- с `uuid` — обновление (`UPDATE ... RETURNING *`); если строки нет — ошибка.

```ts
const user = new UserEntity();
user.name = 'Иван';
await UserEntity.save(user);

user.name = 'Иван Петров';
await UserEntity.save(user); // обновление по uuid
```

### insertMany(entities, options?)

Массовая вставка батчами по 100 строк (`UPSERT`). Присваивает `uuid`, если его нет.

```ts
const users = [
  new UserEntity(),
  new UserEntity(),
  new UserEntity(),
];
users.forEach((u) => (u.name = 'Иван'));
await UserEntity.insertMany(users);
```

### delete(pkValue, options?)

Удаляет запись по первичному ключу. Возвращает удалённую запись (`RETURNING *`) или `null`, если не найдена.

Для составного PK передайте объект со всеми компонентами ключа:

```ts
// Одиночный PK
const deleted = await UserEntity.delete('5ad91505-d4f6-4a81-ab65-9dbc68cf4ed5');

// Составной PK
const deleted = await UserEntity.delete({ user_uuid: '...', role: 'admin' });
```

### updateBy(where, patch, options?)

Массовое обновление по условию. Возвращает количество затронутых строк.

```ts
const updated = await UserEntity.updateBy({ name: 'Иван' }, { role: 'admin' });
```

### deleteBy(where, options?)

Массовое удаление по условию. Возвращает количество удалённых строк.

```ts
const deleted = await UserEntity.deleteBy({ is_deleted: true });
```

{% note warning %}

`updateBy` и `deleteBy` требуют хотя бы одно условие в `where` — защита от случайного обновления/удаления всей таблицы. Они не вызывают lifecycle-хуки.

{% endnote %}

### findOneBy / findBy

Понятные алиасы для `find` / `findAll`:

```ts
const user = await UserEntity.findOneBy({ email: 'ivan@example.com' });
const admins = await UserEntity.findBy({ role: 'admin' });
```

## QueryOptions

Второй аргумент всех методов — объект `QueryOptions`:

| Поле | Описание |
| --- | --- |
| `trx` | executor транзакции (см. [Транзакции](transactions.md)) |
| `timeout` | таймаут в миллисекундах |
| `signal` | `AbortSignal` для отмены запроса |
| `limit` | максимум строк в SELECT (по умолчанию 100, максимум 1000) |
| `offset` | смещение для SELECT |
| `select` | конкретные колонки для SELECT (вместо `SELECT *`) |

```ts
const users = await UserEntity.findAll(
  { role: 'admin' },
  {
    limit: 20,
    offset: 40,
    select: ['uuid', 'name'],
    timeout: 5000,
  },
);
```

## Поиск по зашифрованным полям

Если поле помечено `@YdbEncrypted({ blindIndex: true })`, поиск по нему работает: ORM хеширует значение и сравнивает со synthetic-колонкой `{field}_bi`.

```ts
const user = await UserEntity.find({ email: 'ivan@example.com' });
```

Поиск по зашифрованному полю **без** blind index — ошибка. Подробнее — в разделе [Шифрование](encryption.md).

## Ленивая дешифровка

Если поле помечено `@YdbEncrypted({ lazy: true })`, при чтении из БД в инстанс записывается ciphertext, а не plaintext. Это экономит CPU, когда значение не требуется.

```ts
@YdbEntity('users')
export class UserEntity extends YdbBaseEntity {
  @YdbEncrypted({ lazy: true })
  largeSecret: string;
}

const user = await UserEntity.find({ uuid });
// user.largeSecret — пока ciphertext

const plaintext = await user.decryptField('largeSecret');
// или дешифровать все lazy-поля сразу:
await user.decryptLazyFields();
```

{% note warning %}

`toJSON()` и `JSON.stringify(entity)` бросают ошибку, пока в инстансе есть недешифрованные lazy-поля. Вызовите `await entity.decryptLazyFields()` перед сериализацией.

{% endnote %}

## Удаление и сериализация

### toJSON()

Сериализует экземпляр в JSON, исключая synthetic-колонки `{field}_bi` и конвертируя `BigInt` в строку.

```ts
const user = await UserEntity.find({ uuid });
await user.decryptLazyFields(); // если есть lazy-поля
console.log(JSON.stringify(user)); // без email_bi, government_id_bi
```

### loadRelations(relations, options?)

Загружает указанные relations на инстансе. Подробнее — в разделе [Связи](relations.md).

```ts
const user = await UserEntity.find({ uuid });
await user.loadRelations(['posts', 'profile']);
```

## Пример: полный CRUD-цикл

```ts
import { UserEntity } from './user.entity';

async function crudDemo() {
  // Create
  const user = new UserEntity();
  user.name = 'Иван';
  user.email = 'ivan@example.com';
  await UserEntity.save(user);

  // Read
  const found = await UserEntity.find({ uuid: user.uuid });

  // Update
  found.name = 'Иван Петров';
  await UserEntity.save(found);

  // Delete
  await UserEntity.delete(found.uuid);
}
```