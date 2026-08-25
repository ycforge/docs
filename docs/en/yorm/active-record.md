# Active Record

Entities extend `YdbBaseEntity` and get a static API for working with the database. All methods are async.

{% note info %}

Runtime dependencies (executor, encryption providers) are stored in a `WeakMap` keyed by class — subclasses do not share the parent's state.

{% endnote %}

## Core methods

### find(where, options?)

Returns a single record or `null` (`SELECT ... LIMIT 1`). Requires at least one condition.

```ts
const user = await UserEntity.find({ uuid: '5ad91505-d4f6-4a81-ab65-9dbc68cf4ed5' });
if (user) {
  console.log(user.name);
}
```

### findAll(where?, options?)

Returns a list of records. Default `limit: 100`, max `1000`, `offset: 0`.

```ts
const users = await UserEntity.findAll({ name: 'John' }, { limit: 50, offset: 0 });
const all = await UserEntity.findAll(); // all, but no more than 100 rows
```

### count(where?, options?)

Returns the number of rows matching the condition.

```ts
const total = await UserEntity.count({ name: 'John' });
```

### save(entity, options?)

Saves an entity:

- without `uuid` — insert (`UPSERT`), `uuid` is generated automatically;
- with `uuid` — update (`UPDATE ... RETURNING *`); an error if the row is missing.

```ts
const user = new UserEntity();
user.name = 'John';
await UserEntity.save(user);

user.name = 'John Doe';
await UserEntity.save(user); // update by uuid
```

### insertMany(entities, options?)

Bulk insert in batches of 100 (`UPSERT`). Assigns `uuid` if missing.

```ts
const users = [new UserEntity(), new UserEntity(), new UserEntity()];
users.forEach((u) => (u.name = 'John'));
await UserEntity.insertMany(users);
```

### delete(pkValue, options?)

Deletes a record by primary key. Returns the deleted record (`RETURNING *`) or `null` if not found.

For a composite PK, pass an object with all key components:

```ts
// Single PK
const deleted = await UserEntity.delete('5ad91505-d4f6-4a81-ab65-9dbc68cf4ed5');

// Composite PK
const deleted = await UserEntity.delete({ user_uuid: '...', role: 'admin' });
```

### updateBy(where, patch, options?)

Bulk update by condition. Returns the number of affected rows.

```ts
const updated = await UserEntity.updateBy({ name: 'John' }, { role: 'admin' });
```

### deleteBy(where, options?)

Bulk delete by condition. Returns the number of deleted rows.

```ts
const deleted = await UserEntity.deleteBy({ is_deleted: true });
```

{% note warning %}

`updateBy` and `deleteBy` require at least one `where` condition — a guard against accidentally updating/deleting the whole table. They do not trigger lifecycle hooks.

{% endnote %}

### findOneBy / findBy

Convenient aliases for `find` / `findAll`:

```ts
const user = await UserEntity.findOneBy({ email: 'john@example.com' });
const admins = await UserEntity.findBy({ role: 'admin' });
```

## QueryOptions

The second argument of all methods is a `QueryOptions` object:

| Field | Description |
| --- | --- |
| `trx` | transaction executor (see [Transactions](transactions.md)) |
| `timeout` | timeout in milliseconds |
| `signal` | `AbortSignal` to cancel the query |
| `limit` | max rows in SELECT (default 100, max 1000) |
| `offset` | SELECT offset |
| `select` | specific columns for SELECT (instead of `SELECT *`) |

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

## Searching encrypted fields

If a field is marked `@YdbEncrypted({ blindIndex: true })`, searching by it works: the ORM hashes the value and compares it with the synthetic `{field}_bi` column.

```ts
const user = await UserEntity.find({ email: 'john@example.com' });
```

Searching by an encrypted field **without** blind index is an error. See [Field encryption](encryption.md).

## Lazy decryption

If a field is marked with `@YdbEncrypted({ lazy: true })`, reading from the DB stores the ciphertext in the instance, not the plaintext. This saves CPU when the value is not needed.

```ts
@YdbEntity('users')
export class UserEntity extends YdbBaseEntity {
  @YdbEncrypted({ lazy: true })
  largeSecret: string;
}

const user = await UserEntity.find({ uuid });
// user.largeSecret is still ciphertext

const plaintext = await user.decryptField('largeSecret');
// or decrypt all lazy fields at once:
await user.decryptLazyFields();
```

{% note warning %}

`toJSON()` and `JSON.stringify(entity)` throw if the instance has undecrypted lazy fields. Call `await entity.decryptLazyFields()` before serialization.

{% endnote %}

## Delete and serialization

### toJSON()

Serializes an instance to JSON, excluding synthetic `{field}_bi` columns and converting `BigInt` to string.

```ts
const user = await UserEntity.find({ uuid });
await user.decryptLazyFields(); // if there are lazy fields
console.log(JSON.stringify(user)); // without email_bi, government_id_bi
```

### loadRelations(relations, options?)

Loads the specified relations on the instance. See [Relations](relations.md).

```ts
const user = await UserEntity.find({ uuid });
await user.loadRelations(['posts', 'profile']);
```

## Example: a full CRUD cycle

```ts
import { UserEntity } from './user.entity';

async function crudDemo() {
  // Create
  const user = new UserEntity();
  user.name = 'John';
  user.email = 'john@example.com';
  await UserEntity.save(user);

  // Read
  const found = await UserEntity.find({ uuid: user.uuid });

  // Update
  found.name = 'John Doe';
  await UserEntity.save(found);

  // Delete
  await UserEntity.delete(found.uuid);
}
```