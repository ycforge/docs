# Yandex Cloud KMS

`KmsEncryptionProvider` шифрует и дешифрует поля сущностей через **Yandex Cloud KMS** (REST API SymmetricCrypto). Входит в пакет `@ycforge/orm-security-providers`, подпуть `yandex-kms`.

Строка AAD передаётся в KMS как `aadContext` — ciphertext криптографически привязывается к значениям полей с `@YdbSecurityAAD()`, защищая от подмены связанных данных.

## Установка

```bash
yarn add @ycforge/orm-security-providers
```

Пакет требует `@ycforge/ydb-orm` как peer-зависимость.

## Подключение в forRoot

Провайдер передаётся в опции модуля под ключом `encryptionProvider`:

```ts
import { Module } from '@nestjs/common';
import { YdbModule } from '@ycforge/ydb-orm';
import { KmsEncryptionProvider } from '@ycforge/orm-security-providers/yandex-kms';

@Module({
  imports: [
    YdbModule.forRoot({
      useFactory: () => ({
        endpoint: process.env.YDB_ENDPOINT!,
        auth_type: 'auth_key',
        authOptions: { authorized_key_path: './authorized_key.json' },
        encryptionProvider: new KmsEncryptionProvider({
          keyId: process.env.KMS_KEY_ID!,
          auth_type: 'auth_key',
          authOptions: {
            authorized_key_path: './authorized_key.json',
          },
        }),
      }),
    }),
  ],
})
export class AppModule {}
```

{% note info %}

`YdbCoreModule.forRootAsync(...)` — эквивалентная форма той же конфигурации, если вы используете core-модуль напрямую.

{% endnote %}

## Опции

| Опция | Тип | Обязательно | По умолчанию | Описание |
|-------|-----|-------------|--------------|----------|
| `keyId` | `string` | да | — | ID симметричного ключа KMS |
| `auth_type` | `'meta' \| 'auth_key' \| 'iam_token'` | да | — | Способ авторизации |
| `authOptions` | `KmsAuthOptions` | да | — | Параметры авторизации |
| `apiEndpoint` | `string` | нет | `https://kms.yandex` | Базовый URL KMS API |

{% note warning %}

Endpoint по умолчанию — `https://kms.yandex`. Устаревший `kms.api.cloud.yandex.net` возвращает 404 на криптооперациях — не используйте его.

{% endnote %}

## Авторизация

Провайдер получает IAM-токен через `IamTokenManager` (экспортируется из того же подпути). Поддерживается три способа:

| Режим | Описание | Опции |
|-------|----------|-------|
| `meta` | Metadata-сервис виртуальной машины (только в Yandex Cloud) | — |
| `auth_key` | JSON-ключ сервисного аккаунта → JWT → IAM-токен | `authorized_key_path` |
| `iam_token` | Готовый IAM-токен (без автообновления) | `iam_token` |

### auth_key (рекомендуется для локальной разработки и CI)

```bash
yc iam service-account create --name kms-encrypter
yc iam key create --service-account-id <sa_id> --output authorized_key.json
yc kms symmetric-key add-access-binding \
  --id <key_id> \
  --service-account-id <sa_id> \
  --role kms.keys.encrypterDecrypter
```

```ts
new KmsEncryptionProvider({
  keyId: process.env.KMS_KEY_ID!,
  auth_type: 'auth_key',
  authOptions: { authorized_key_path: './authorized_key.json' },
});
```

### meta (продакшен на ВМ)

```ts
new KmsEncryptionProvider({
  keyId: process.env.KMS_KEY_ID!,
  auth_type: 'meta',
  authOptions: {},
});
```

### iam_token (быстрая проверка)

```ts
new KmsEncryptionProvider({
  keyId: process.env.KMS_KEY_ID!,
  auth_type: 'iam_token',
  authOptions: { iam_token: process.env.IAM_TOKEN! },
});
```

## Использование с сущностями

```ts
import { YdbEntity, YdbBaseEntity, YdbColumn, YdbPrimaryColumn, YdbEncrypted, YdbSecurityAAD } from '@ycforge/ydb-orm';

@YdbEntity('users')
export class UserEntity extends YdbBaseEntity {
  @YdbPrimaryColumn('Uuid')
  uuid: string;

  @YdbSecurityAAD()
  @YdbColumn('Utf8')
  organization: string;

  // Хранится зашифрованным, доступно для поиска через synthetic-колонку `email_bi`
  @YdbEncrypted({ blindIndex: true })
  @YdbColumn('Bytes')
  email: string;

  // Зашифровано, но поиск недоступен
  @YdbEncrypted({ blindIndex: false })
  @YdbColumn('Bytes')
  government_id: string;
}
```

```ts
// NestJS: добавьте сущность в forFeature
@Module({
  imports: [YdbModule.forFeature([UserEntity])],
})
export class UsersModule {}

// Создание и чтение — шифрование/дешифрование происходит прозрачно
const user = await UserEntity.save({ email: 'john@example.com' });
const found = await UserEntity.find({ email: 'john@example.com' });
```

Для поиска по зашифрованным полям blind index должен быть включён; провайдер хеша передайте в `blindIndexProvider` — см. [HMAC-SHA256 blind index](../blind-index/hmac-bi.md).
