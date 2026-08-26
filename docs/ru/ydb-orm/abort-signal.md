# AbortSignal

Каждый запрос и транзакцию можно отменить или ограничить по времени через `AbortSignal`.

## Отмена запроса

Передайте `signal` в `QueryOptions` — второй аргумент всех методов Active Record, репозитория и `options()` QueryBuilder:

```ts
const controller = new AbortController();

setTimeout(() => controller.abort(new Error('Клиент отключился')), 5_000);

const users = await UserEntity.findAll(
  { role: 'admin' },
  { signal: controller.signal }, // запрос отменится через 5 секунд
);
```

При отмене промис реджектится с причиной из сигнала (`controller.abort(reason)`); запрос не продолжается на сервере дальше транспортной отмены.

## Таймауты

`timeout` в `QueryOptions` задаёт дедлайн на один запрос в миллисекундах:

```ts
// Эквивалентно передаче AbortSignal.timeout(5000)
const user = await UserEntity.find({ uuid }, { timeout: 5_000 });
```

Общий дедлайн через явный сигнал:

```ts
const user = await UserEntity.find(
  { uuid },
  { signal: AbortSignal.timeout(30_000) },
);
```

`AbortSignal.timeout(ms)` срабатывает автоматически через `ms` — контроллер и очистка не нужны.

## QueryBuilder

```ts
const photos = await PhotoEntity.query()
  .where({ is_public: true })
  .options({ signal: controller.signal, timeout: 10_000 })
  .getMany();
```

## Прямые запросы через executor

Объект запроса executor'а предоставляет `.signal()`:

```ts
await executor`
  SELECT * FROM users WHERE name = $name
`
  .parameter('name', 'Иван')
  .signal(controller.signal)
  .execute();
```

## Транзакции

В `runInTransaction(fn, options)` охваты различаются:

- `signal` — **глобальный**: отменяет всю операцию, включая все retry-попытки;
- `timeout` — **на каждую попытку**: при ретраях (`idempotent: true`) каждая попытка получает свежее окно; колбэк получает объединённый сигнал вторым аргументом.

```ts
import { YdbTransactionManager } from '@ycforge/ydb-orm';

const txManager = new YdbTransactionManager(executor);

await txManager.runInTransaction(
  async (trx, signal) => {
    // signal объединяет сигнал попытки + таймаут этой попытки
    await OrderEntity.save(order, { trx });
    if (signal.aborted) throw new Error('Отменено');
  },
  {
    timeout: 5_000,
    signal: AbortSignal.timeout(30_000), // общий дедлайн на все попытки
  },
);
```

При [retry-политике](transactions.md#retry-политика) ORM отмена никогда не превращается в повтор — операция завершается причиной отмены (`signal.reason`).

## Массовые операции

`insertMany`, `updateBy`, `deleteBy` принимают те же `QueryOptions`:

```ts
await UserEntity.insertMany(users, {
  timeout: 10_000,
  signal: req.signal, // например, abort входящего HTTP-запроса
});
```

## Практические паттерны

**Скоуп HTTP-запроса** (отмена работы с БД при отключении клиента):

```ts
app.post('/users', async (req, res) => {
  const user = await UserEntity.save(buildUser(req.body), { signal: req.signal });
  res.json(user);
});
```

**Гонка с бизнес-логикой:**

```ts
const controller = new AbortController();
const work = exportReport({ signal: controller.signal });
const done = await Promise.race([
  work,
  new Promise((_, reject) =>
    setTimeout(() => controller.abort(), 60_000),
  ),
]);
```

{% note info %}

Сигналы/таймауты фиксируются драйвером на каждый запрос; истёкший таймаут проявляется как ошибка YDB timeout, а не вечное зависание. Пометка `{ idempotent: true }` разрешает ретраи только для транзитных статусов — таймауты детерминированных запросов молча не повторяются.

{% endnote %}

## Следующие шаги

- [Транзакции](transactions.md) — опции исполнения и retry-семантика
- [AbortSignal (NestJS)](nest/abort-signal.md) — отмена в скоупе HTTP-запроса для NestJS
