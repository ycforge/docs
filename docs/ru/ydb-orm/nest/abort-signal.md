# AbortSignal (NestJS)

Отмена и таймауты работают так же, как в standalone — через `QueryOptions.signal` / `timeout`. В NestJS типичный источник сигнала — скоуп HTTP-запроса.

## Отмена в скоупе запроса

Отменяйте работу с БД при отключении клиента (в Express 5 / Fastify доступен `req.signal`; иначе соберите контроллер из событий жизненного цикла запроса):

```ts
import { Body, Controller, Post, Req } from '@nestjs/common';
import type { Request } from 'express';
import { UserEntity } from './user.entity';

@Controller('users')
export class UsersController {
  @Post()
  create(@Body() body: CreateUserDto, @Req() req: Request) {
    return UserEntity.save(buildUser(body), { signal: req.signal });
  }
}
```

С ручным контроллером (любой HTTP-фреймворк):

```ts
const controller = new AbortController();
res.on('close', () => controller.abort());

const users = await UserEntity.findAll(
  { role: 'admin' },
  { signal: controller.signal },
);
```

## Таймауты в сервисах

```ts
@Injectable()
export class ReportsService {
  constructor(private readonly txManager: YdbTransactionManager) {}

  async heavyReport() {
    // Дедлайн на один запрос
    const rows = await OrderEntity.findAll({}, { timeout: 10_000 });

    await this.txManager.runInTransaction(
      async (trx) => {
        await OrderEntity.save(order, { trx });
      },
      {
        timeout: 5_000,                      // на каждую попытку
        signal: AbortSignal.timeout(30_000), // общий дедлайн на все попытки
      },
    );
  }
}
```

В транзакциях охваты различаются:

- `signal` — **глобальный**: отменяет всю операцию, включая все retry-попытки;
- `timeout` — **на каждую попытку**: при `idempotent: true` каждая попытка получает свежее окно; колбэк получает объединённый сигнал вторым аргументом.

## Interceptor, ограничивающий время запросов

NestJS-interceptor может задать дедлайн для всего, что выполняется внутри обработчика:

```ts
@Injectable()
export class TimeoutInterceptor implements NestInterceptor {
  intercept(context: ExecutionContext, next: CallHandler) {
    const controller = new AbortController();
    const timer = setTimeout(() => controller.abort(), 15_000);

    return next.handle().pipe(
      tap({
        complete: () => clearTimeout(timer),
        error: () => clearTimeout(timer),
      }),
      // передайте controller.signal в сервисы через request-scoped провайдер
    );
  }
}
```

{% note info %}

Отмена никогда не превращается в повтор [retry-политикой](../transactions.md#retry-политика) ORM — операция завершается причиной отмены (`signal.reason`).

{% endnote %}

## Standalone-альтернатива

Фреймворк-независимые паттерны — в разделе [AbortSignal](../abort-signal.md).
