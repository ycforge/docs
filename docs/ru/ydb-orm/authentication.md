# Аутентификация

`@ycforge/ydb-orm` не хранит собственные стратегии аутентификации. Вместо этого он принимает готовый `AuthManager` из `@ycforge/auth` и адаптирует его в `CredentialsProvider` для `@ydbjs/auth`.

Если вы хотите повторно использовать одну конфигурацию аутентификации для YDB и Yandex Cloud API (например, KMS), создайте один `AuthManager` и передайте его в оба модуля.

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

Подробнее о стратегиях — в документации [@ycforge/auth](../auth/index.md).

## Standalone-использование

Передайте опцию `auth` в `createDriver`:

```ts
import { createDriver, createExecutor } from '@ycforge/ydb-orm';
import { createAuth, authKeyFromFile } from '@ycforge/auth';

const driver = await createDriver({
  endpoint: 'grpcs://ydb.serverless.yandexcloud.net:2135/?database=/ru-central1/.../...',
  auth: createAuth(authKeyFromFile('./authorized_key.json')),
});

const executor = createExecutor(driver);
```

ORM внутри вызывает `createYdbCredentialsProvider(auth, YDB_AUTH_USAGE, { endpoint, secure })` из `@ycforge/auth/ydb`. Область `YDB_AUTH_USAGE` обязательна — это позволяет адаптеру проверить, что выбранная стратегия совместима с YDB.

## Передача готового CredentialsProvider

Для передачи кастомного провайдера (например, в тестах) используйте опцию `credentialsProvider`:

```ts
const driver = await createDriver({
  endpoint: 'grpcs://ydb.serverless.yandexcloud.net:2135/?database=/ru-central1/.../...',
  credentialsProvider: myTestProvider,
});
```

Одновременная установка `credentialsProvider` и `auth` — ошибка конфигурации.

## Следующие шаги

- [Аутентификация (NestJS)](nest/authentication.md) — если вы используете NestJS, см. передачу `auth` через опции модуля
