# NestJS AbortSignal

Cancellation and timeouts work the same as standalone — via `QueryOptions.signal` / `timeout`. In NestJS the typical source of a signal is the HTTP request scope.

## Request-scoped cancellation

Cancel DB work when the client disconnects (Express 5 / Fastify expose `req.signal`; otherwise build a controller from request lifecycle events):

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

With a manual controller (any HTTP framework):

```ts
const controller = new AbortController();
res.on('close', () => controller.abort());

const users = await UserEntity.findAll(
  { role: 'admin' },
  { signal: controller.signal },
);
```

## Timeouts in services

```ts
@Injectable()
export class ReportsService {
  constructor(private readonly txManager: YdbTransactionManager) {}

  async heavyReport() {
    // Per-query deadline
    const rows = await OrderEntity.findAll({}, { timeout: 10_000 });

    await this.txManager.runInTransaction(
      async (trx) => {
        await OrderEntity.save(order, { trx });
      },
      {
        timeout: 5_000,                      // per attempt
        signal: AbortSignal.timeout(30_000), // total deadline for all attempts
      },
    );
  }
}
```

In transactions the scopes differ:

- `signal` — **global**: cancels the whole operation including all retry attempts;
- `timeout` — **per attempt**: with `idempotent: true` each attempt gets a fresh window; the callback receives a merged signal as the second argument.

## Interceptor that bounds query time

A NestJS interceptor can enforce a deadline for everything executed inside a request handler:

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
      // pass controller.signal into services via a request-scoped provider
    );
  }
}
```

{% note info %}

Cancellation is never converted into a retry by the ORM [retry policy](../transactions.md#retry-policy) — the operation ends with the cancellation reason (`signal.reason`).

{% endnote %}

## Standalone alternative

See [AbortSignal](../abort-signal.md) for the framework-agnostic patterns.
