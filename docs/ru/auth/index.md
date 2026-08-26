# @ycforge/auth

Единая точка аутентификации для API Яндекс Облака и YDB. Пакет выделен из `@ycforge/ydb-orm` и `@ycforge/orm-security-providers`, чтобы одну и ту же конфигурацию авторизации можно было переиспользовать в нескольких потребителях.

- Нулевое количество runtime-зависимостей в ядре (Node.js builtins + глобальный `fetch`).
- Кэширование токенов с запасом 60 секунд и single-flight обновлением.
- Встроенный retry с экспоненциальным бэкоффом.
- Capability-матрица: каждый запрос токена проверяется на совместимость с целевым использованием.
- Опциональные адаптеры: `@ydbjs/auth` и NestJS.

Требуется Node.js ≥ 22 (ESM).

## Установка

```bash
yarn add @ycforge/auth
# опционально, для YDB-адаптера:
yarn add @ydbjs/auth
# опционально, для NestJS-модуля:
yarn add @nestjs/common reflect-metadata
```

## Быстрый старт

```ts
import { createAuth, YCLOUD_AUTH_USAGE } from '@ycforge/auth';

const auth = createAuth({ type: 'metadata' });
const token = await auth.getToken(YCLOUD_AUTH_USAGE);
```

## Способы авторизации

### `iam_token` — статический IAM-токен

```ts
import { createAuth } from '@ycforge/auth';

const auth = createAuth({
  type: 'iam_token',
  token: process.env.IAM_TOKEN!,
  // опционально: после наступления этого момента getToken() будет бросать ошибку
  expiresAt: '2026-09-01T00:00:00Z',
});
```

### `metadata` — сервис метаданных ВМ

```ts
const auth = createAuth({
  type: 'metadata',
  // значения по умолчанию:
  endpoint:
    'http://169.254.169.254/computeMetadata/v1/instance/service-accounts/default/token',
  flavor: 'Google',
});
```

### `auth_key` — авторизованный ключ сервисного аккаунта

Подписывает JWT с алгоритмом PS256 и обменивает его на IAM-токен по адресу `https://iam.api.cloud.yandex.net/iam/v1/tokens`.

```ts
import { createAuth, authKeyFromFile } from '@ycforge/auth';

const auth = createAuth(authKeyFromFile('./authorized_key.json'));
// или inline:
const auth2 = createAuth({
  type: 'auth_key',
  keyId: 'aje...',
  serviceAccountId: 'sa-...',
  privateKey: process.env.PRIVATE_KEY!,
});
```

### `access_token` / `anonymous` / `static` (только YDB)

```ts
import { createAuth } from '@ycforge/auth';

createAuth({ type: 'access_token', token: '...' });
createAuth({ type: 'anonymous' }); // пустой токен
createAuth({ type: 'static', username: 'user', password: 'pass' });
```

Стратегия `static` (username/password) не может получить токен без YDB gRPC-эндпоинта: в ядре её `getToken()` бросает ошибку с подсказкой использовать адаптер `@ycforge/auth/ydb`, который делегирует вызовы `StaticCredentialsProvider` из `@ydbjs/auth`.

## Capability-матрица

| Стратегия      | `YCLOUD_AUTH_USAGE` | `YDB_AUTH_USAGE` |
| -------------- | ------------------- | ---------------- |
| `iam_token`    | ✅                  | ✅               |
| `metadata`     | ✅                  | ✅               |
| `auth_key`     | ✅                  | ✅               |
| `access_token` | ❌                  | ✅               |
| `anonymous`    | ❌                  | ✅               |
| `static`       | ❌                  | ✅               |

Запрос токена для неподдерживаемого использования бросает `UnsupportedAuthMethodError`.

## YDB-адаптер (`@ycforge/auth/ydb`)

Адаптер превращает `AuthManager` в `CredentialsProvider`, который ожидает `@ydbjs/auth`.

### Зачем нужны `endpoint` / `secure`?

Большинству стратегий YDB-эндпоинт **не нужен**. Адаптер требует его только для стратегии `static`, потому что `StaticCredentialsProvider` выполняет gRPC-вызов `Login` к YDB-эндпоинту. Для остальных стратегий параметры игнорируются.

### Пример без NestJS

```ts
import { createAuth, authKeyFromFile, YDB_AUTH_USAGE } from '@ycforge/auth';
import { createYdbCredentialsProvider } from '@ycforge/auth/ydb';
import { Driver } from '@ydbjs/core';

const auth = createAuth(authKeyFromFile('./authorized_key.json'));
const creds = createYdbCredentialsProvider(auth, YDB_AUTH_USAGE);

const driver = new Driver('grpcs://ydb.serverless.yandexcloud.net:2135/?database=/ru-central1/.../...', {
  credentialsProvider: creds,
});
await driver.ready();
```

### Логин и пароль

```ts
import { createAuth, YDB_AUTH_USAGE } from '@ycforge/auth';
import { createYdbCredentialsProvider } from '@ycforge/auth/ydb';

const auth = createAuth({
  type: 'static',
  username: 'user',
  password: 'pass',
});

const creds = createYdbCredentialsProvider(auth, YDB_AUTH_USAGE, {
  endpoint: 'grpcs://ydb.serverless.yandexcloud.net:2135',
  secure: true,
});
```

## NestJS-модуль (`@ycforge/auth/nestjs`)

```ts
import { Module } from '@nestjs/common';
import { InjectAuth, YCFORGE_AUTH, YcAuthModule } from '@ycforge/auth/nestjs';
import type { AuthManager } from '@ycforge/auth';

@Module({
  imports: [
    YcAuthModule.forRoot({
      config: { type: 'metadata' },
      global: true,
    }),
  ],
})
export class AppModule {}

@Injectable()
export class SomeService {
  constructor(@InjectAuth() private readonly auth: AuthManager) {}
}
```

Менеджер предоставляется под символом `YCFORGE_AUTH`.

### Пример: один `AuthManager` для `@ycforge/ydb-orm`

```ts
import { Module } from '@nestjs/common';
import { YdbOrmModule } from '@ycforge/ydb-orm';
import { YCFORGE_AUTH, YcAuthModule } from '@ycforge/auth/nestjs';
import type { AuthManager } from '@ycforge/auth';

@Module({
  imports: [
    YcAuthModule.forRoot({
      config: { type: 'metadata' },
      global: true,
    }),
    YdbOrmModule.forRootAsync({
      useFactory: (auth: AuthManager) => ({
        endpoint: 'grpcs://ydb.serverless.yandexcloud.net:2135',
        database: '/ru-central1/.../...',
        auth,
      }),
      inject: [YCFORGE_AUTH],
    }),
  ],
})
export class AppModule {}
```

ORM сам запросит токены с использованием `YDB_AUTH_USAGE`.

### Пример: один `AuthManager` для `@ycforge/orm-security-providers`

```ts
import { Module } from '@nestjs/common';
import { YandexKmsModule } from '@ycforge/orm-security-providers';
import { YCFORGE_AUTH, YcAuthModule } from '@ycforge/auth/nestjs';
import type { AuthManager } from '@ycforge/auth';

@Module({
  imports: [
    YcAuthModule.forRoot({
      global: true,
      config: authKeyFromFile('./keys/kms-key.json'),
    }),
    YandexKmsModule.forRootAsync({
      useFactory: (auth: AuthManager) => ({
        keyId: process.env.KMS_KEY_ID!,
        auth,
      }),
      inject: [YCFORGE_AUTH],
    }),
  ],
})
export class AppModule {}
```

Провайдер запросит IAM-токен с использованием `YCLOUD_AUTH_USAGE`.

## Несколько конфигураций

Один вызов `createAuth` — один конфиг. Для нескольких независимых конфигураций создайте несколько менеджеров:

```ts
const ydbAuth = createAuth({ type: 'metadata' });
const cloudAuth = createAuth(authKeyFromFile('./authorized_key.json'));
```

В NestJS используйте один `YcAuthModule` на область приложения или передавайте нужный `AuthManager` напрямую, если конфигурации различаются.
