# Yandex Cloud KMS

`KmsEncryptionProvider` encrypts and decrypts entity fields through the **Yandex Cloud KMS** SymmetricCrypto REST API. Part of the `@ycforge/orm-security-providers` package, subpath `yandex-kms`.

The AAD string is passed to KMS as `aadContext` — the ciphertext is cryptographically bound to the values of the `@YdbSecurityAAD()` fields, protecting against tampering with related data.

## Install

```bash
yarn add @ycforge/orm-security-providers
```

The package requires `@ycforge/yorm` as a peer dependency.

## Config in forRoot

Pass the provider in the module options under `encryptionProvider`. KMS authorization is an IAM token; get it through `@ycforge/auth` and pass a ready `AuthManager`:

```ts
import { Module } from '@nestjs/common';
import { YdbModule } from '@ycforge/yorm';
import { KmsEncryptionProvider } from '@ycforge/orm-security-providers/yandex-kms';
import { createAuth, authKeyFromFile } from '@ycforge/auth';

@Module({
  imports: [
    YdbModule.forRoot({
      useFactory: () => ({
        endpoint: process.env.YDB_ENDPOINT!,
        auth: createAuth(authKeyFromFile('./authorized_key.json')),
        encryptionProvider: new KmsEncryptionProvider({
          keyId: process.env.KMS_KEY_ID!,
          auth: createAuth(authKeyFromFile('./authorized_key.json')),
        }),
      }),
    }),
  ],
})
export class AppModule {}
```

{% note info %}

`YdbCoreModule.forRootAsync(...)` is an equivalent form of the same configuration if you use the core module directly.

{% endnote %}

To share one `AuthManager` between YDB and KMS, register `@ycforge/auth/nestjs` once and inject it via `YCFORGE_AUTH` — see the example in the [@ycforge/auth](../../../auth/index.md) docs.

## Options

| Option | Type | Required | Default | Description |
|--------|------|----------|---------|-------------|
| `keyId` | `string` | yes | — | Symmetric KMS key ID |
| `auth` | `AuthManager` | yes | — | Ready auth manager from `@ycforge/auth` |
| `apiEndpoint` | `string` | no | `https://kms.yandex` | KMS API base URL |

{% note warning %}

The default endpoint is `https://kms.yandex`. The legacy `kms.api.cloud.yandex.net` returns 404 on crypto operations — do not use it.

{% endnote %}

## Authentication

The provider obtains an IAM token through the passed `AuthManager` by calling `auth.getToken(YCLOUD_AUTH_USAGE)`. All IAM strategies from `@ycforge/auth` are available:

| Mode | Description | Options |
|------|-------------|---------|
| `metadata` | VM metadata service (Yandex Cloud only) | — |
| `auth_key` | Service-account JSON key → JWT → IAM token | `authKeyFromFile` |
| `iam_token` | Static IAM token (no auto-refresh) | `token` |

### auth_key (recommended for local and CI)

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
  auth: createAuth(authKeyFromFile('./authorized_key.json')),
});
```

### metadata (production on a VM)

```ts
import { createAuth } from '@ycforge/auth';

new KmsEncryptionProvider({
  keyId: process.env.KMS_KEY_ID!,
  auth: createAuth({ type: 'metadata' }),
});
```

### iam_token (quick test)

```ts
new KmsEncryptionProvider({
  keyId: process.env.KMS_KEY_ID!,
  auth: createAuth({ type: 'iam_token', token: process.env.IAM_TOKEN! }),
});
```

## Usage with entities

```ts
import { YdbEntity, YdbBaseEntity, YdbColumn, YdbPrimaryColumn, YdbEncrypted, YdbSecurityAAD } from '@ycforge/yorm';

@YdbEntity('users')
export class UserEntity extends YdbBaseEntity {
  @YdbPrimaryColumn('Uuid')
  uuid: string;

  @YdbSecurityAAD()
  @YdbColumn('Utf8')
  organization: string;

  // Stored encrypted, searchable via the synthetic `email_bi` column
  @YdbEncrypted({ blindIndex: true })
  @YdbColumn('Bytes')
  email: string;

  // Encrypted but not searchable
  @YdbEncrypted({ blindIndex: false })
  @YdbColumn('Bytes')
  government_id: string;
}
```

```ts
// NestJS: add the entity to forFeature
@Module({
  imports: [YdbModule.forFeature([UserEntity])],
})
export class UsersModule {}

// Create and read — encryption/decryption happens transparently
const user = await UserEntity.save({ email: 'john@example.com' });
const found = await UserEntity.find({ email: 'john@example.com' });
```

For searchable fields, blind index must be enabled; pass the hash provider in `blindIndexProvider` — see [HMAC-SHA256 blind index](../blind-index/hmac-bi.md).
