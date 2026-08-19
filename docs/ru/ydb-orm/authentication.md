# Аутентификация

Аутентификация настраивается опцией `auth_type` в параметрах подключения.

## Способы аутентификации

| Значение | Описание |
| --- | --- |
| `meta` | IAM-токен из metadata-сервиса (внутри Yandex Cloud) |
| `auth_key` | authorized key JSON сервисного аккаунта (`authOptions.authorized_key_path`); JWT-обмен реализован на `fetch`, без тяжёлых SDK |
| `anonymous` | анонимный доступ (локальная YDB) |

### meta

```ts
{
  endpoint: 'grpcs://ydb.serverless.yandexcloud.net:2135/?database=/ru-central1/...',
  auth_type: 'meta',
  authOptions: {},
}
```

### auth_key

```ts
{
  endpoint: 'grpcs://ydb.serverless.yandexcloud.net:2135/?database=/ru-central1/...',
  auth_type: 'auth_key',
  authOptions: { authorized_key_path: './authorized_key.json' },
}
```

{% note warning %}

`authorized_key.json` — секретный файл. Не коммитьте его в репозиторий и не выводите содержимое в код, логи или ответы API.

{% endnote %}

### anonymous

```ts
{
  endpoint: 'grpc://localhost:2136/local',
  auth_type: 'anonymous',
  authOptions: {},
}
```

## Ошибки

Невалидное значение `auth_type` → ошибка `Invalid YDB auth type`. Отсутствующий `authorized_key_path` при `auth_key` → ошибка `Authorized key path not provided`.

## Передача провайдера credentials

Если нужно передать готовый credentials provider (например, для тестов), используйте `createDriver(opts, credentialsProvider)` — см. [Использование без NestJS](standalone.md).