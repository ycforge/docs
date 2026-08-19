# Шифрование полей

ORM поддерживает шифрование отдельных полей с поиском по зашифрованным значениям через blind index.

## Как это работает

1. Поле помечается `@YdbEncrypted({ blindIndex })`.
2. Перед записью ORM вызывает провайдер шифрования: `encrypt(plaintext, aad, context)` → `Uint8Array`.
3. Ciphertext хранится в колонке `Bytes` (raw bytes — без base64, экономия ~33% по сравнению с `Utf8`).
4. Если `blindIndex: true`, дополнительно вычисляется детерминированный хеш и сохраняется в synthetic-колонку `{field}_bi` (`Utf8`).
5. После чтения ORM вызывает `decrypt(ciphertext, aad, context)` → plaintext.

## Подключение провайдеров

Провайдеры передаются в опции модуля:

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

### Интерфейсы

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

`YdbEncryptionContext` содержит: `entityName`, `tableName`, `fieldName`, `primaryKeyValue`, `aadFields`.

{% note warning %}

Провайдеры входят в **отдельный пакет** `@ycforge/orm-security-providers` (AES-256-GCM, HMAC-SHA256, KMS-провайдеры). В составе `@ycforge/ydb-orm` есть только заглушка `Base64TestEncryptionProvider` — она подходит для тестов, но не обеспечивает реальной криптографии.

{% endnote %}

### Пример провайдера (AES-256-GCM + HMAC-SHA256)

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

Ключи передавайте через переменные окружения — не хардкодьте их в коде и не коммитьте в репозиторий.

{% endnote %}

## Поиск по зашифрованным полям

### С blind index

```ts
@YdbEntity('users')
export class UserEntity extends YdbBaseEntity {
  @YdbPrimaryColumn('Uuid')
  uuid: string;

  // В БД: ciphertext + колонка email_bi
  @YdbEncrypted({ blindIndex: true })
  email: string;
}

// Работает: ORM хеширует значение и сравнивает хеши
const user = await UserEntity.find({ email: 'ivan@example.com' });
```

### Без blind index

Поиск по такому полю невозможен — ORM бросит ошибку.

```ts
@YdbEncrypted({ blindIndex: false })
government_id: string;

// Ошибка: Cannot search by encrypted field "government_id" without blind index
await UserEntity.find({ government_id: '1234567890' });
```

## AAD (Additional Authenticated Data)

Поля, помеченные `@YdbSecurityAAD()`, включаются в AAD при шифровании других полей. Если такое поле изменится, существующие ciphertext'ы перестанут расшифровываться — защита от подмены связанных данных.

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

AAD-строка строится из полей в лексикографическом порядке: `organization=acme;...`. Для массового `updateBy` по зашифрованному полю нужен явный `aadOverride`, иначе ORM бросит ошибку — AAD нельзя пересобрать без полной строки.

## Особенности

- Объект сущности хранит **plaintext**. ORM шифрует копию перед UPSERT и не мутирует исходный объект — иначе повторный `save()` зашифровал бы ciphertext повторно.
- `null` / `undefined` не шифруются.
- При чтении ORM расшифровывает поля автоматически; synthetic-колонки `{field}_bi` в инстанс не попадают и исключаются из `toJSON()`.