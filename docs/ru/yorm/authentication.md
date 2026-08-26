# Аутентификация

`@ycforge/ydb-orm` не хранит собственных способов авторизации. Вместо этого он принимает готовый `AuthManager` из пакета `@ycforge/auth` и сам адаптирует его в `CredentialsProvider` для `@ydbjs/auth`.

Если вам нужно переиспользовать одну конфигурацию авторизации для YDB и для Yandex Cloud API (например, KMS), создайте `AuthManager` один раз и передайте его в оба модуля.

## Поддерживаемые способы

Все стратегии задаются через `@ycforge/auth`:

| Стратегия | Описание |
| --- | --- |
| `iam_token` | Статический IAM-токен |
| `metadata` | IAM-токен из сервиса метаданных ВМ (только в Yandex Cloud) |
| `auth_key` | Авторизованный ключ сервисного аккаунта → JWT → IAM-токен |
| `access_token` | Статический access-токен (YDB only) |
| `anonymous` | Анонимный доступ (локальная YDB) |
| `static` | Логин/пароль (YDB only) |

Подробнее о стратегиях, capability-матрице и YDB-адаптере см. в документации [@ycforge/auth](../auth/index.md).

## Передача AuthManager в модуль

Опция `auth` принимает готовый `AuthManager`:

```ts
import { Module } from '@nestjs/common';
import { YdbCoreModule } from '@ycforge/ydb-orm';
import { createAuth, authKeyFromFile } from '@ycforge/auth';

@Module({
  imports: [
    YdbCoreModule.forRootAsync({
      useFactory: () => ({
        endpoint: 'grpcs://ydb.serverless.yandexcloud.net:2135/?database=/ru-central1/.../...',
        auth: createAuth(authKeyFromFile('./authorized_key.json')),
        sync: true, // только для dev!
      }),
    }),
  ],
})
export class AppModule {}
```

ORM внутри вызывает `createYdbCredentialsProvider(auth, YDB_AUTH_USAGE, { endpoint, secure })` из `@ycforge/auth/ydb`. Указание скоупа `YDB_AUTH_USAGE` обязательно — так адаптер проверяет, что выбранная стратегия совместима с YDB (например, `static` нельзя использовать для `YCLOUD_AUTH_USAGE`).

## Передача готового CredentialsProvider

Если нужно передать собственный провайдер (например, в тестах), используйте опцию `credentialsProvider`:

```ts
YdbCoreModule.forRootAsync({
  useFactory: () => ({
    endpoint: 'grpcs://ydb.serverless.yandexcloud.net:2135/?database=/ru-central1/.../...',
    credentialsProvider: myTestProvider,
  }),
})
```

Нельзя одновременно задавать `credentialsProvider` и `auth` (или `driverOptions.credentialsProvider`) — это ошибка конфигурации.

## Авторизация вне NestJS

См. раздел [Без NestJS (standalone)](standalone.md).
