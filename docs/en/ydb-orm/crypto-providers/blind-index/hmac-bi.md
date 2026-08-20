# HMAC-SHA256 blind index

`KmsBlindIndexProvider` computes deterministic blind indexes for searching encrypted fields. Part of the `@ycforge/orm-security-providers` package, subpath `hmac-bi`.

Yandex Cloud KMS does not provide a native hash function, so the blind index is implemented via **HMAC-SHA256** with a separate key. The result is deterministic: the same value always produces the same hash, allowing the ORM to search encrypted fields by comparing hashes.

{% note info %}

`context` in the `hash()` method is accepted but ignored — the hash depends only on the key and the value.

{% endnote %}

## Install

```bash
yarn add @ycforge/orm-security-providers
```

The package requires `@ycforge/ydb-orm` as a peer dependency.

## Options

| Option | Type | Required | Description |
|--------|------|----------|-------------|
| `blindIndexKey` | `string` | yes | Base64-encoded HMAC key, at least 32 bytes (256 bits) |

Generate a key:

```bash
openssl rand -base64 32
```

{% note warning %}

Store the key in an environment variable — the hash is one-way, but anyone who owns the key can compute hashes for arbitrary values. Do not hardcode or commit it.

{% endnote %}

## Config in forRoot

Pass the provider in the module options under `blindIndexProvider`:

```ts
import { Module } from '@nestjs/common';
import { YdbModule } from '@ycforge/ydb-orm';
import { KmsBlindIndexProvider } from '@ycforge/orm-security-providers/hmac-bi';

@Module({
  imports: [
    YdbModule.forRoot({
      useFactory: () => ({
        endpoint: process.env.YDB_ENDPOINT!,
        auth_type: 'auth_key',
        authOptions: { authorized_key_path: './authorized_key.json' },
        encryptionProvider: /* any YdbEncryptionProvider, e.g. KmsEncryptionProvider */,
        blindIndexProvider: new KmsBlindIndexProvider({
          blindIndexKey: process.env.BLIND_INDEX_KEY!,
        }),
      }),
    }),
  ],
})
export class AppModule {}
```

## Usage with entities

Mark a field with `@YdbEncrypted({ blindIndex: true })` — the ORM will store the hash in the synthetic `{field}_bi` column (`Utf8`):

```ts
import { YdbEntity, YdbBaseEntity, YdbColumn, YdbPrimaryColumn, YdbEncrypted } from '@ycforge/ydb-orm';

@YdbEntity('users')
export class UserEntity extends YdbBaseEntity {
  @YdbPrimaryColumn('Uuid')
  uuid: string;

  // In DB: ciphertext + `email_bi` column
  @YdbEncrypted({ blindIndex: true })
  @YdbColumn('Bytes')
  email: string;
}
```

```ts
// NestJS: add the entity to forFeature
@Module({
  imports: [YdbModule.forFeature([UserEntity])],
})
export class UsersModule {}

// Works: the ORM hashes the value and compares hashes
const user = await UserEntity.find({ email: 'john@example.com' });
```

{% note info %}

A blind index is one-way: plaintext cannot be recovered from the hash. It only supports equality search over encrypted values.

{% endnote %}
