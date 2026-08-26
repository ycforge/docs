# Транзакции

ORM предоставляет `YdbTransactionManager` для атомарных операций. Транзакции гарантируют, что несколько записей либо все применятся, либо ни одна.

## Использование

```ts
import { YdbTransactionManager } from '@ycforge/ydb-orm';
import type { QueryOptions } from '@ycforge/ydb-orm';
import { UserEntity } from './user.entity';

const txManager = new YdbTransactionManager(executor);

await txManager.runInTransaction(async (trx) => {
  const opts: QueryOptions = { trx };

  const from = await UserEntity.find({ uuid: fromId }, opts);
  const to = await UserEntity.find({ uuid: toId }, opts);

  from.balance -= 100;
  to.balance += 100;

  await UserEntity.save(from, opts);
  await UserEntity.save(to, opts);
  // Обе записи применяются атомарно: либо обе, либо ни одна
});
```

Внутри колбэка `trx` — executor транзакции. Он передаётся во все методы сущностей через `{ trx }` в `QueryOptions`. Также можно использовать [QueryBuilder](query-builder.md) через `options({ trx })`.

## Опции исполнения

`runInTransaction(fn, options?)` принимает опции, которые пробрасываются в вызов транзакции SDK:

```ts
await txManager.runInTransaction(
  async (trx, signal) => {
    await OrderEntity.save(order, { trx });
  },
  {
    isolation: 'snapshotReadWrite', // serializableReadWrite (по умолчанию) | snapshotReadOnly | snapshotReadWrite
    timeout: 5_000,                 // таймаут НА КАЖДУЮ ПОПЫТКУ (см. ниже)
    signal: controller.signal,      // ГЛОБАЛЬНАЯ отмена: вся операция, все попытки
    idempotent: true,               // см. «Retry-семантика» ниже
  },
);
```

| Опция | Описание |
| --- | --- |
| `isolation` | уровень изоляции YDB: `serializableReadWrite` (по умолчанию), `snapshotReadOnly`, `snapshotReadWrite` |
| `timeout` | таймаут на каждую попытку в миллисекундах; каждая retry-попытка получает свежее окно |
| `signal` | глобальный `AbortSignal`: отменяет операцию целиком, включая все попытки |
| `idempotent: true` | разрешает SDK повторить колбэк при retryable-ошибках (колбэк обязан быть устойчивым к повтору) |
| `retry` | политика ретраев тела, управляемая ORM (`true` или объект политики); несовместима с `reuse` |
| `reuse: true` | присоединиться к активной транзакции вместо ошибки о вложенности |
| `ambient: true` | пробросить транзакцию в ambient-контекст: операции без явного `{ trx }` идут в неё |

Опции валидируются fail-fast: неизвестный ключ (опечатка), невалидный уровень изоляции, неположительный `timeout`, не-`AbortSignal` — ошибка сразу.

**Семантика отмены при `idempotent: true`** — у `signal` и `timeout` разный охват:

- `signal` — **глобальный**: пробрасывается в SDK как есть и отменяет операцию целиком, включая все retry-попытки;
- `timeout` — **на каждую попытку**: SDK может повторить колбэк заново (`idempotent: true`), и каждая попытка получает свежее окно таймаута — retry никогда не стартует с уже истёкшим дедлайном первой попытки. Сигнал, который получает колбэк (`fn(trx, signal)`), объединяет сигнал попытки от SDK и `AbortSignal.timeout(timeout)` этой попытки.

Полный дедлайн на всю операцию задаётся явно через пользовательский сигнал:

```ts
await txManager.runInTransaction(fn, {
  idempotent: true,
  signal: AbortSignal.timeout(30_000), // общий лимит на все попытки
});
```

Подробнее об отмене — в разделе [AbortSignal](abort-signal.md).

## Retry-политика

При `idempotent: true` SDK повторяет **весь колбэк** при retryable-ошибках (сбой сети, смерть сессии):

- побочные эффекты колбэка выполняются повторно;
- lifecycle hooks (`@BeforeInsert`, `@AfterInsert`, ...) срабатывают больше одного раза;
- каждая попытка получает новую сессию/транзакцию (новый `trx`).

Чтобы владеть циклом повторов самостоятельно (ограниченное число попыток), передайте политику `retry`:

```ts
await txManager.runInTransaction(
  async (trx) => {
    await OrderEntity.save(order, { trx });
  },
  {
    idempotent: true,   // колбэк обязан быть устойчивым к повтору
    retry: true,        // или объект политики; нельзя совмещать с reuse
    maxAttempts: 5,
  },
);
```

При заданной политике повторами тела владеет ORM: между попытками bounded backoff с jitter, повторяются только транзитные статусы (`ABORTED`, `UNAVAILABLE`, `OVERLOADED`), `timeout` по-прежнему действует на каждую попытку, глобальный `signal` и `idempotent` пробрасываются в SDK как раньше. Без опции `retry` тело ретраит только SDK.

## Вложенные транзакции

Вложенный `runInTransaction()` по умолчанию **запрещён**: второй вызов откроет независимую транзакцию на другой сессии, что почти всегда ошибка. Чтобы присоединиться к активной транзакции (коммит/откат остаются у внешнего вызова), передайте `{ reuse: true }`:

```ts
await txManager.runInTransaction(async () => {
  await txManager.runInTransaction(async (trx2) => {
    // Error: Nested runInTransaction() detected ...
  });
});

await txManager.runInTransaction(async () => {
  await txManager.runInTransaction(async (sameTrx) => {
    // та же транзакция, что и снаружи
  }, { reuse: true });
});
```

Вложенность определяется по AsyncLocalStorage-цепочке и только для того же executor'а БД; вложенные транзакции на другом драйвере/базе считаются независимыми.

## Ambient-контекст (opt-in)

Один пропущенный `{ trx }` — и запрос молча уйдёт вне транзакции. Ambient-режим решает это: операции репозиториев без явного `{ trx }` автоматически выполняются в активной транзакции.

```ts
// Глобально для процесса — опция модуля NestJS:
YdbCoreModule.forRootAsync({
  useFactory: () => ({
    // ...
    transactions: { ambient: true },
  }),
});

// Или точечно, на один вызов:
await txManager.runInTransaction(async () => {
  await OrderEntity.save(order);          // уйдёт в транзакцию автоматически
  await OrderEntity.save(other, { trx }); // явный trx тоже работает
}, { ambient: true });
```

Правила безопасности:

- если при активной ambient-транзакции явно передан **другой** `{ trx }` — ошибка смешивания, а не молчаливое расхождение данных;
- после commit/rollback контекст очищается;
- параллельные транзакции не перетекают друг в друга;
- ambient выключен по умолчанию: явный `{ trx }` работает как раньше.

## Запросы вне транзакции

Для отладки можно включить предупреждение о каждом запросе вне какой бы то ни было транзакции:

```ts
transactions: { warnOutsideTransaction: true } // console.warn на каждый такой запрос
```

По умолчанию выключено.

## Массовые операции

`insertMany`, `updateBy` и `deleteBy` также принимают `{ trx }` и выполняются внутри транзакции.

## Что не покрывают транзакции

DDL (CREATE TABLE, ALTER TABLE, DROP TABLE) в YDB не транзакционен — выполнение миграций последовательное. Используйте [миграции](migrations.md) для изменения схемы.

## Следующие шаги

- [AbortSignal](abort-signal.md) — отмена запросов и таймауты
- [Транзакции (NestJS)](nest/transactions.md) — инжектируемый паттерн для NestJS
