# NestJS encryption

In NestJS applications, encryption and blind index providers are passed via `YdbCoreModule.forRootAsync()` options.

## Configuring providers

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

## Provider interfaces

See the [standalone encryption guide](../encryption.md#interfaces) for the `YdbEncryptionProvider` and `YdbBlindIndexProvider` interfaces, the AES-256-GCM example provider, and AAD documentation.

## Accessing providers via DI

The encryption and blind index providers are available as DI tokens:

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

## Standalone alternative

If you don't use NestJS, see the [standalone encryption guide](../encryption.md) with `configureEntities`.
