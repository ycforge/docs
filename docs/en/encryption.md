# Field encryption

The ORM supports encrypting individual fields with the ability to search encrypted values via a blind index.

## How it works

1. A field is marked with `@YdbEncrypted({ blindIndex })`.
2. Before writing, the ORM calls the encryption provider: `encrypt(plaintext, aad, context)` → `Uint8Array`.
3. The ciphertext is stored in a `Bytes` column (raw bytes — no base64, ~33% savings compared to `Utf8`).
4. If `blindIndex: true`, a deterministic hash is also computed and stored in the synthetic `{field}_bi` column (`Utf8`).
5. After reading, the ORM calls `decrypt(ciphertext, aad, context)` → plaintext.

## Configuring providers

Providers are passed in the module options:

```ts
YdbCoreModule.forRootAsync({
  useFactory: () => ({
    endpoint: '...',
    auth_type: 'auth_key',
    authOptions: { authorized_key_path: './authorized_key.json' },
    encryptionProvider: myEncryptionProvider,
    blindIndexProvider: myBlindIndexProvider,
  }),
});
```

### Interfaces

```ts
interface YdbEncryptionProvider {
  encrypt(
    plaintext: string,
    aad: string,
    context: YdbEncryptionContext,
  ): Promise<Uint8Array>;

  decrypt(
    ciphertext: Uint8Array,
    aad: string,
    context: YdbEncryptionContext,
  ): Promise<string>;
}

interface YdbBlindIndexProvider {
  hash(plaintext: string, context: YdbEncryptionContext): Promise<string>;
}
```

`YdbEncryptionContext` contains: `entityName`, `tableName`, `fieldName`, `primaryKeyValue`, `aadFields`.

{% note warning %}

Providers ship in a **separate package** `@ycforge/orm-security-providers` (AES-256-GCM, HMAC-SHA256, KMS providers). `@ycforge/ydb-orm` only ships the `Base64TestEncryptionProvider` stub — it is fine for tests but provides no real cryptography.

{% endnote %}

### Example provider (AES-256-GCM + HMAC-SHA256)

```ts
import { createCipheriv, createDecipheriv, createHmac, randomBytes } from 'node:crypto';
import type { YdbEncryptionProvider, YdbBlindIndexProvider, YdbEncryptionContext } from '@ycforge/ydb-orm';

export class AesGcmEncryptionProvider
  implements YdbEncryptionProvider, YdbBlindIndexProvider
{
  private readonly key: Buffer;
  private readonly hmacKey: Buffer;

  constructor(encryptionKey: string, blindIndexKey: string) {
    this.key = Buffer.from(encryptionKey, 'base64');
    this.hmacKey = Buffer.from(blindIndexKey, 'base64');
  }

  async encrypt(plaintext: string, aad: string): Promise<Uint8Array> {
    const iv = randomBytes(12);
    const cipher = createCipheriv('aes-256-gcm', this.key, iv);
    cipher.setAAD(Buffer.from(aad, 'utf8'));
    const encrypted = Buffer.concat([cipher.update(plaintext, 'utf8'), cipher.final()]);
    return Buffer.concat([iv, cipher.getAuthTag(), encrypted]);
  }

  async decrypt(ciphertext: Uint8Array, aad: string): Promise<string> {
    const data = Buffer.from(ciphertext);
    const decipher = createDecipheriv('aes-256-gcm', this.key, data.subarray(0, 12));
    decipher.setAAD(Buffer.from(aad, 'utf8'));
    decipher.setAuthTag(data.subarray(12, 28));
    return Buffer.concat([decipher.update(data.subarray(28)), decipher.final()]).toString('utf8');
  }

  async hash(plaintext: string): Promise<string> {
    return createHmac('sha256', this.hmacKey).update(plaintext).digest('base64');
  }
}
```

{% note warning %}

Pass keys via environment variables — do not hardcode them in code or commit them to the repository.

{% endnote %}

## Searching encrypted fields

### With blind index

```ts
@YdbEntity('users')
export class UserEntity extends YdbBaseEntity {
  @YdbPrimaryColumn('Uuid')
  uuid: string;

  // In DB: ciphertext + email_bi column
  @YdbEncrypted({ blindIndex: true })
  email: string;
}

// Works: the ORM hashes the value and compares hashes
const user = await UserEntity.find({ email: 'john@example.com' });
```

### Without blind index

Searching by such a field is impossible — the ORM raises an error.

```ts
@YdbEncrypted({ blindIndex: false })
government_id: string;

// Error: Cannot search by encrypted field "government_id" without blind index
await UserEntity.find({ government_id: '1234567890' });
```

## AAD (Additional Authenticated Data)

Fields marked with `@YdbSecurityAAD()` are included in the AAD when encrypting other fields. If such a field changes, existing ciphertexts stop decrypting — protection against tampering with related data.

```ts
@YdbEntity('users')
export class UserEntity extends YdbBaseEntity {
  @YdbPrimaryColumn('Uuid')
  uuid: string;

  @YdbSecurityAAD()
  @YdbColumn('Utf8')
  organization: string;

  @YdbEncrypted()
  email: string;
}
```

The AAD string is built from fields in lexicographic order: `organization=acme;...`. For a bulk `updateBy` on an encrypted field, an explicit `aadOverride` is required, otherwise the ORM raises an error — AAD cannot be reconstructed without the full row.

## Notes

- The entity object stores **plaintext**. The ORM encrypts a copy before UPSERT and does not mutate the original object — otherwise a repeated `save()` would re-encrypt the ciphertext.
- `null` / `undefined` values are not encrypted.
- On read, the ORM decrypts fields automatically; synthetic `{field}_bi` columns do not appear on the instance and are excluded from `toJSON()`.