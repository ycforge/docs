# Authentication

Authentication is configured with the `auth_type` option in the connection parameters.

## Authentication methods

| Value | Description |
| --- | --- |
| `meta` | IAM token from the metadata service (inside Yandex Cloud) |
| `auth_key` | authorized key JSON of a service account (`authOptions.authorized_key_path`); JWT exchange is implemented with `fetch`, no heavy SDKs |
| `anonymous` | anonymous access (local YDB) |

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

`authorized_key.json` is a secret file. Do not commit it to the repository or expose its contents in code, logs, or API responses.

{% endnote %}

### anonymous

```ts
{
  endpoint: 'grpc://localhost:2136/local',
  auth_type: 'anonymous',
  authOptions: {},
}
```

## Errors

An invalid `auth_type` value → `Invalid YDB auth type` error. A missing `authorized_key_path` with `auth_key` → `Authorized key path not provided` error.

## Passing a credentials provider

To pass a ready-made credentials provider (for example, in tests), use `createDriver(opts, credentialsProvider)` — see [Standalone usage](standalone.md).