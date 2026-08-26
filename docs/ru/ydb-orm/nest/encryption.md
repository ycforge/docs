# Шифрование (NestJS)

В NestJS-приложениях провайдеры шифрования и blind index передаются через опции `YdbCoreModule.forRootAsync()`.

## Настройка провайдеров

```ts
import { Module } from '@nestjs/common';
import { YdbCoreModule } from '@ycforge/ydb-orm/nest';
import { myEncryptionProvider, myBlindIndexProvider } from './providers';

@Module({
  imports: [
    YdbCoreModule.forRootAsync({
      useFactory: () => ({
        endpoint: '...',
        auth: createAuth(authKeyFromFile('./authorized_key.json')),
        encryptionProvider: myEncryptionProvider,
        blindIndexProvider: myBlindIndexProvider,
      }),
    }),
  ],
})
export class AppModule {}
```

## Интерфейсы провайдеров

См. [standalone-руководство по шифрованию](../encryption.md#интерфейсы) для интерфейсов `YdbEncryptionProvider` и `YdbBlindIndexProvider`, примера AES-256-GCM-провайдера и документации по AAD.

## Доступ к провайдерам через DI

Провайдеры шифрования и blind index доступны как DI-токены:

```ts
import { Inject, Injectable } from '@nestjs/common';
import { YDB_ENCRYPTION_PROVIDER, YDB_BLIND_INDEX_PROVIDER } from '@ycforge/ydb-orm/nest';
import type { YdbEncryptionProvider, YdbBlindIndexProvider } from '@ycforge/ydb-orm';

@Injectable()
export class CryptoService {
  constructor(
    @Inject(YDB_ENCRYPTION_PROVIDER)
    private readonly encryption: YdbEncryptionProvider,
    @Inject(YDB_BLIND_INDEX_PROVIDER)
    private readonly blindIndex: YdbBlindIndexProvider,
  ) {}
}
```

## Standalone-альтернатива

Если вы не используете NestJS, см. [standalone-руководство по шифрованию](../encryption.md) с `configureEntities`.
