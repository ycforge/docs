# NestJS validation

The validation provider can be declared once in module options — `forFeature` wires it into every registered entity via `setValidationProvider`.

## Declaring in module options

```ts
import { Module } from '@nestjs/common';
import { YdbCoreModule, YdbOrmModule } from '@ycforge/ydb-orm/nest';
import { ClassValidatorProvider } from '@ycforge/ydb-orm';
import { UserEntity } from './user.entity';

@Module({
  imports: [
    YdbCoreModule.forRootAsync({
      useFactory: () => ({
        endpoint: '...',
        auth: createAuth(authKeyFromFile('./authorized_key.json')),
        validationProvider: new ClassValidatorProvider({ groups: ['create'] }),
      }),
    }),
    YdbOrmModule.forFeature([UserEntity]),
  ],
})
export class AppModule {}
```

`ClassValidatorProvider` uses `class-validator` (an optional peer dependency). The provider is also injectable via the DI token `YDB_VALIDATION_PROVIDER`.

## Declaring rules

Rules are standard `class-validator` decorators on entity fields:

```ts
import { IsEmail, IsNotEmpty, Length } from 'class-validator';
import { YdbBaseEntity, YdbEntity, YdbColumn, YdbPrimaryColumn } from '@ycforge/ydb-orm';

@YdbEntity('users')
export class UserEntity extends YdbBaseEntity {
  @YdbPrimaryColumn('Uuid')
  uuid: string;

  @IsNotEmpty()
  @Length(2, 100)
  @YdbColumn('Utf8')
  name: string;

  @IsEmail()
  @YdbColumn('Utf8')
  email: string;
}
```

## Custom provider as a DI service

Implement `YdbValidationProvider` and register it in DI; reference it from the factory:

```ts
import { Injectable } from '@nestjs/common';
import type { YdbValidationProvider } from '@ycforge/ydb-orm';

@Injectable()
export class MyValidationProvider implements YdbValidationProvider {
  async validate(entity: any): Promise<string[]> {
    const errors: string[] = [];
    if (!entity.name) errors.push('name is required');
    return errors;
  }
}

@Module({
  providers: [MyValidationProvider],
  imports: [
    YdbCoreModule.forRootAsync({
      useFactory: (validator: MyValidationProvider) => ({
        endpoint: '...',
        auth: createAuth(authKeyFromFile('./authorized_key.json')),
        validationProvider: validator,
      }),
      inject: [MyValidationProvider],
    }),
    YdbOrmModule.forFeature([UserEntity]),
  ],
})
export class AppModule {}
```

Validation runs before encryption and the write:

- `save()` — before insert and update;
- `insertMany()` — for each item.

On failure an exception with the error list is thrown:

```
Validation failed for UserEntity: name should not be empty; email must be an email
```

## Standalone alternative

Without NestJS call `UserEntity.setValidationProvider(...)` manually after `configureEntities` — see [Validation](../validation.md).
