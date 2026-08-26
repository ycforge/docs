# NestJS authentication

In NestJS applications, authentication is configured via `YdbCoreModule.forRootAsync()` options.

## Passing AuthManager to the module

The `auth` option accepts a ready `AuthManager`:

```ts
import { Module } from '@nestjs/common';
import { YdbCoreModule } from '@ycforge/ydb-orm/nest';
import { createAuth, authKeyFromFile } from '@ycforge/auth';

@Module({
  imports: [
    YdbCoreModule.forRootAsync({
      useFactory: () => ({
        endpoint: 'grpcs://ydb.serverless.yandexcloud.net:2135/?database=/ru-central1/.../...',
        auth: createAuth(authKeyFromFile('./authorized_key.json')),
        sync: true,
      }),
    }),
  ],
})
export class AppModule {}
```

The ORM internally calls `createYdbCredentialsProvider(auth, YDB_AUTH_USAGE, { endpoint, secure })` from `@ycforge/auth/ydb`. The `YDB_AUTH_USAGE` scope is mandatory — this lets the adapter verify that the chosen strategy is compatible with YDB.

## Passing a ready CredentialsProvider

To pass a custom provider (for example, in tests), use the `credentialsProvider` option:

```ts
YdbCoreModule.forRootAsync({
  useFactory: () => ({
    endpoint: 'grpcs://ydb.serverless.yandexcloud.net:2135/?database=/ru-central1/.../...',
    credentialsProvider: myTestProvider,
  }),
})
```

Setting `credentialsProvider` together with `auth` (or `driverOptions.credentialsProvider`) is a configuration error.

## Supported strategies

All strategies are configured via `@ycforge/auth`:

| Strategy | Description |
| --- | --- |
| `iam_token` | Static IAM token |
| `metadata` | VM metadata service IAM token (Yandex Cloud only) |
| `auth_key` | Service account authorized key → JWT → IAM token |
| `access_token` | Static access token (YDB only) |
| `anonymous` | Anonymous access (local YDB) |
| `static` | Username/password (YDB only) |

See the [@ycforge/auth](../../auth/index.md) docs for details on strategies.

## Standalone alternative

If you don't use NestJS, see the [standalone authentication guide](../authentication.md) with `createDriver`.
