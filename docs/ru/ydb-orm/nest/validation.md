# Валидация (NestJS)

Провайдер валидации объявляется один раз в опциях модуля — `forFeature` сам пробрасывает его в каждую зарегистрированную сущность через `setValidationProvider`.

## Объявление в опциях модуля

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

`ClassValidatorProvider` использует `class-validator` (опциональная peer-зависимость). Провайдер также доступен для инъекции через DI-токен `YDB_VALIDATION_PROVIDER`.

## Объявление правил

Правила — стандартные декораторы `class-validator` на полях сущности:

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

## Кастомный провайдер как DI-сервис

Реализуйте `YdbValidationProvider` и зарегистрируйте в DI; сославшись на него из фабрики:

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

Валидация выполняется перед шифрованием и записью:

- `save()` — перед вставкой и обновлением;
- `insertMany()` — для каждого элемента.

При ошибке бросается исключение со списком ошибок:

```
Validation failed for UserEntity: name should not be empty; email must be an email
```

## Standalone-альтернатива

Без NestJS вызывайте `UserEntity.setValidationProvider(...)` вручную после `configureEntities` — см. [Валидация](../validation.md).
