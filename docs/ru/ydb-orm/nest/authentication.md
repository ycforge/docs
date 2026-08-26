# Аутентификация (NestJS)

В NestJS-приложениях аутентификация настраивается через опции `YdbCoreModule.forRootAsync()`.

## Передача AuthManager в модуль

Опция `auth` принимает готовый `AuthManager`:

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

ORM внутри вызывает `createYdbCredentialsProvider(auth, YDB_AUTH_USAGE, { endpoint, secure })` из `@ycforge/auth/ydb`. Область `YDB_AUTH_USAGE` обязательна — это позволяет адаптеру проверить, что выбранная стратегия совместима с YDB.

## Передача готового CredentialsProvider

Для передачи кастомного провайдера (например, в тестах) используйте опцию `credentialsProvider`:

```ts
YdbCoreModule.forRootAsync({
  useFactory: () => ({
    endpoint: 'grpcs://ydb.serverless.yandexcloud.net:2135/?database=/ru-central1/.../...',
    credentialsProvider: myTestProvider,
  }),
})
```

Одновременная установка `credentialsProvider` и `auth` (или `driverOptions.credentialsProvider`) — ошибка конфигурации.

## Поддерживаемые стратегии

Все стратегии настраиваются через `@ycforge/auth`:

| Стратегия | Описание |
| --- | --- |
| `iam_token` | Статический IAM-токен |
| `metadata` | IAM-токен через metadata-сервис VM (только Yandex Cloud) |
| `auth_key` | Ключ сервисного аккаунта → JWT → IAM-токен |
| `access_token` | Статический access-токен (только YDB) |
| `anonymous` | Анонимный доступ (локальный YDB) |
| `static` | Имя пользователя/пароль (только YDB) |

Подробнее о стратегиях — в документации [@ycforge/auth](../../auth/index.md).

## Standalone-альтернатива

Если вы не используете NestJS, см. [standalone-руководство по аутентификации](../authentication.md) с `createDriver`.
