# HMAC-SHA256 blind index

`KmsBlindIndexProvider` вычисляет детерминированные blind index для поиска по зашифрованным полям. Входит в пакет `@ycforge/orm-security-providers`, подпуть `hmac-bi`.

Yandex Cloud KMS не предоставляет нативной хеш-функции, поэтому blind index реализован через **HMAC-SHA256** с отдельным ключом. Результат детерминирован: одно и то же значение всегда даёт один и тот же хеш, что позволяет ORM искать по зашифрованным полям через сравнение хешей.

{% note info %}

`context` в методе `hash()` принимается, но игнорируется — хеш зависит только от ключа и значения.

{% endnote %}

## Установка

```bash
yarn add @ycforge/orm-security-providers
```

Пакет требует `@ycforge/yorm` как peer-зависимость.

## Опции

| Опция | Тип | Обязательно | Описание |
|-------|-----|-------------|----------|
| `blindIndexKey` | `string` | да | Base64-encoded HMAC-ключ, минимум 32 байта (256 бит) |

Сгенерировать ключ:

```bash
openssl rand -base64 32
```

{% note warning %}

Ключ храните в переменной окружения — хеш однонаправленный, но любой, кто владеет ключом, может вычислить хеши произвольных значений. Не хардкодьте и не коммитьте его.

{% endnote %}

## Подключение в forRoot

Провайдер передаётся в опции модуля под ключом `blindIndexProvider`:

```ts
import { Module } from '@nestjs/common';
import { YdbModule } from '@ycforge/yorm';
import { KmsBlindIndexProvider } from '@ycforge/orm-security-providers/hmac-bi';

@Module({
  imports: [
    YdbModule.forRoot({
      useFactory: () => ({
        endpoint: process.env.YDB_ENDPOINT!,
        auth_type: 'auth_key',
        authOptions: { authorized_key_path: './authorized_key.json' },
        encryptionProvider: /* любой YdbEncryptionProvider, напр. KmsEncryptionProvider */,
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

Пометите поле `@YdbEncrypted({ blindIndex: true })` — ORM сохраняет хеш в synthetic-колонку `{field}_bi` (`Utf8`):

```ts
import { YdbEntity, YdbBaseEntity, YdbColumn, YdbPrimaryColumn, YdbEncrypted } from '@ycforge/yorm';

@YdbEntity('users')
export class UserEntity extends YdbBaseEntity {
  @YdbPrimaryColumn('Uuid')
  uuid: string;

  // В БД: ciphertext + колонка `email_bi`
  @YdbEncrypted({ blindIndex: true })
  @YdbColumn('Bytes')
  email: string;
}
```

```ts
// NestJS: добавьте сущность в forFeature
@Module({
  imports: [YdbModule.forFeature([UserEntity])],
})
export class UsersModule {}

// Работает: ORM хеширует значение и сравнивает хеши
const user = await UserEntity.find({ email: 'john@example.com' });
```

{% note info %}

Blind index однонаправленный: восстановить plaintext из хеша нельзя. Он даёт только поиск по равенству для зашифрованных значений.

{% endnote %}
