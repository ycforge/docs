# Собственный AES-провайдер

ORM требует только провайдер, реализующий интерфейс `YdbEncryptionProvider`. Ничто не мешает написать собственный — например, AES-256-GCM на базе `node:crypto`. Подключается он в модуль так же, как любой другой провайдер.

## Пример провайдера

```ts
import { createCipheriv, createDecipheriv, randomBytes } from 'node:crypto';
import type { YdbEncryptionProvider, YdbEncryptionContext } from '@ycforge/ydb-orm';

export class AesGcmEncryptionProvider implements YdbEncryptionProvider {
  private readonly key: Buffer;

  constructor(encryptionKey: string) {
    // base64-encoded 32-байтный ключ
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

Сгенерировать ключ:

```bash
openssl rand -base64 32
```

{% note warning %}

Ключ передавайте через переменную окружения — не хардкодьте его в коде и не коммитьте в репозиторий.

{% endnote %}

## Подключение в forRoot

```ts
import { Module } from '@nestjs/common';
import { YdbOrmModule } from '@ycforge/ydb-orm';
import { KmsBlindIndexProvider } from '@ycforge/orm-security-providers/hmac-bi';
import { AesGcmEncryptionProvider } from './aes-gcm-encryption.provider';

@Module({
  imports: [
    YdbOrmModule.forRoot({
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

## Использование с сущностями

```ts
import { YdbEntity, YdbBaseEntity, YdbColumn, YdbPrimaryColumn, YdbEncrypted } from '@ycforge/ydb-orm';

@YdbEntity('users')
export class UserEntity extends YdbBaseEntity {
  @YdbPrimaryColumn('Uuid')
  uuid: string;

  // Доступно для поиска: провайдер blind index хеширует значение в `email_bi`
  @YdbEncrypted({ blindIndex: true })
  @YdbColumn('Bytes')
  email: string;
}
```

{% note info %}

AES-провайдер отвечает только за шифрование. Для поиска по зашифрованным полям (`blindIndex: true`) нужно также зарегистрировать провайдер blind index — в примере выше используется `KmsBlindIndexProvider`, см. [HMAC-SHA256 blind index](../blind-index/hmac-bi.md).

{% endnote %}