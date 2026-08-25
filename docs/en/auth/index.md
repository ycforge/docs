# @ycforge/auth

Shared authentication entry point for Yandex Cloud APIs and YDB. The package was extracted from `@ycforge/yorm` and `@ycforge/orm-security-providers` so the same auth configuration can be reused across multiple consumers.

- Zero runtime dependencies in the core (Node.js builtins + global `fetch`).
- Token caching with 60-second leeway and single-flight refresh.
- Built-in exponential-backoff retry.
- Capability matrix: every token request is validated against the target usage.
- Optional adapters: `@ydbjs/auth` and NestJS.

Requires Node.js ≥ 22 (ESM).

## Installation

```bash
yarn add @ycforge/auth
# optional, for the YDB adapter:
yarn add @ydbjs/auth
# optional, for the NestJS module:
yarn add @nestjs/common reflect-metadata
```

## Quick start

```ts
import { createAuth, YCLOUD_AUTH_USAGE } from '@ycforge/auth';

const auth = createAuth({ type: 'metadata' });
const token = await auth.getToken(YCLOUD_AUTH_USAGE);
```

## Strategies

### `iam_token` — static IAM token

```ts
import { createAuth } from '@ycforge/auth';

const auth = createAuth({
  type: 'iam_token',
  token: process.env.IAM_TOKEN!,
  // optional: after this point getToken() throws instead of returning a dead token
  expiresAt: '2026-09-01T00:00:00Z',
});
```

### `metadata` — VM metadata service

```ts
const auth = createAuth({
  type: 'metadata',
  // defaults:
  endpoint:
    'http://169.254.169.254/computeMetadata/v1/instance/service-accounts/default/token',
  flavor: 'Google',
});
```

### `auth_key` — service account authorized key

Signs a PS256 JWT and exchanges it for an IAM token at `https://iam.api.cloud.yandex.net/iam/v1/tokens`.

```ts
import { createAuth, authKeyFromFile } from '@ycforge/auth';

const auth = createAuth(authKeyFromFile('./authorized_key.json'));
// or inline:
const auth2 = createAuth({
  type: 'auth_key',
  keyId: 'aje...',
  serviceAccountId: 'sa-...',
  privateKey: process.env.PRIVATE_KEY!,
});
```

### `access_token` / `anonymous` / `static` (YDB only)

```ts
import { createAuth } from '@ycforge/auth';

createAuth({ type: 'access_token', token: '...' });
createAuth({ type: 'anonymous' }); // empty token
createAuth({ type: 'static', username: 'user', password: 'pass' });
```

The `static` strategy cannot fetch a token without a YDB gRPC endpoint. Its core `getToken()` throws with a hint to use the `@ycforge/auth/ydb` adapter, which delegates to `StaticCredentialsProvider` from `@ydbjs/auth`.

## Capability matrix

| Strategy       | `YCLOUD_AUTH_USAGE` | `YDB_AUTH_USAGE` |
| -------------- | ------------------- | ---------------- |
| `iam_token`    | ✅                  | ✅               |
| `metadata`     | ✅                  | ✅               |
| `auth_key`     | ✅                  | ✅               |
| `access_token` | ❌                  | ✅               |
| `anonymous`    | ❌                  | ✅               |
| `static`       | ❌                  | ✅               |

Requesting a token for an unsupported usage throws `UnsupportedAuthMethodError`.

## YDB adapter (`@ycforge/auth/ydb`)

The adapter turns an `AuthManager` into a `CredentialsProvider` expected by `@ydbjs/auth`.

### Why endpoint / secure?

Most strategies do **not** need a YDB endpoint. The adapter only requires it for the `static` strategy, because `StaticCredentialsProvider` performs a gRPC `Login` call. For all other strategies the parameters are ignored.

### Standalone example (without NestJS)

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

### Username / password

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

## NestJS module (`@ycforge/auth/nestjs`)

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

The manager is provided under the `YCFORGE_AUTH` symbol.

### Example: sharing one `AuthManager` with `@ycforge/yorm`

```ts
import { Module } from '@nestjs/common';
import { YdbModule } from '@ycforge/yorm';
import { YCFORGE_AUTH, YcAuthModule } from '@ycforge/auth/nestjs';
import type { AuthManager } from '@ycforge/auth';

@Module({
  imports: [
    YcAuthModule.forRoot({
      config: { type: 'metadata' },
      global: true,
    }),
    YdbModule.forRootAsync({
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

The ORM will request tokens with `YDB_AUTH_USAGE`.

### Example: sharing one `AuthManager` with `@ycforge/orm-security-providers`

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

The provider will request IAM tokens with `YCLOUD_AUTH_USAGE`.

## Multiple configurations

One `createAuth` call is one config. For independent configs, create multiple managers:

```ts
const ydbAuth = createAuth({ type: 'metadata' });
const cloudAuth = createAuth(authKeyFromFile('./authorized_key.json'));
```

In NestJS, use one `YcAuthModule` per application scope, or pass the specific `AuthManager` directly when configurations differ.
