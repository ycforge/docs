# Custom AES provider

The ORM only requires a provider that implements `YdbEncryptionProvider`. Nothing stops you from writing your own — for example, an AES-256-GCM provider on top of `node:crypto`. It connects to the module the same way as any other provider.

## Example provider

```ts
import { createCipheriv, createDecipheriv, randomBytes } from 'node:crypto';
import type { YdbEncryptionProvider, YdbEncryptionContext } from '@ycforge/ydb-orm';

export class AesGcmEncryptionProvider implements YdbEncryptionProvider {
  private readonly key: Buffer;

  constructor(encryptionKey: string) {
    // base64-encoded 32-byte key
    this.key = Buffer.from(encryptionKey, 'base64');
  }

  async encrypt(plaintext: string, aad: string, _context: YdbEncryptionContext): Promise<Uint8Array> {
    const iv = randomBytes(12);
    const cipher = createCipheriv('aes-256-gcm', this.key, iv);
    cipher.setAAD(Buffer.from(aad, 'utf8'));
    const encrypted = Buffer.concat([cipher.update(plaintext, 'utf8'), cipher.final()]);
    return Buffer.concat([iv, cipher.getAuthTag(), encrypted]);
  }

  async decrypt(ciphertext: Uint8Array, aad: string, _context: YdbEncryptionContext): Promise<string> {
    const data = Buffer.from(ciphertext);
    const decipher = createDecipheriv('aes-256-gcm', this.key, data.subarray(0, 12));
    decipher.setAAD(Buffer.from(aad, 'utf8'));
    decipher.setAuthTag(data.subarray(12, 28));
    return Buffer.concat([decipher.update(data.subarray(28)), decipher.final()]).toString('utf8');
  }
}
```

Generate a key:

```bash
openssl rand -base64 32
```

{% note warning %}

Pass the key via an environment variable — do not hardcode it in code or commit it to the repository.

{% endnote %}

## Config in forRoot

```ts
import { Module } from '@nestjs/common';
import { YdbModule } from '@ycforge/ydb-orm';
import { KmsBlindIndexProvider } from '@ycforge/orm-security-providers/hmac-bi';
import { AesGcmEncryptionProvider } from './aes-gcm-encryption.provider';

@Module({
  imports: [
    YdbModule.forRoot({
      useFactory: () => ({
        endpoint: process.env.YDB_ENDPOINT!,
        auth_type: 'auth_key',
        authOptions: { authorized_key_path: './authorized_key.json' },
        encryptionProvider: new AesGcmEncryptionProvider(process.env.ENCRYPTION_KEY!),
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

```ts
import { YdbEntity, YdbBaseEntity, YdbColumn, YdbPrimaryColumn, YdbEncrypted } from '@ycforge/ydb-orm';

@YdbEntity('users')
export class UserEntity extends YdbBaseEntity {
  @YdbPrimaryColumn('Uuid')
  uuid: string;

  // Searchable: the blind index provider hashes the value into `email_bi`
  @YdbEncrypted({ blindIndex: true })
  @YdbColumn('Bytes')
  email: string;
}
```

{% note info %}

The AES provider handles encryption only. For searchable encrypted fields (`blindIndex: true`) you must also register a blind index provider — the example above uses `KmsBlindIndexProvider`, see [HMAC-SHA256 blind index](../blind-index/hmac-bi.md).

{% endnote %}