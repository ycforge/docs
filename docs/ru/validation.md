# Валидация

ORM может валидировать сущности перед записью. Валидация выполняется перед шифрованием и записью: если есть ошибки, выбрасывается исключение с их списком.

## Подключение провайдера

Провайдер валидации устанавливается на сущность через `setValidationProvider`. Встроенный `ClassValidatorProvider` использует `class-validator` (опциональная peer-зависимость).

```ts
import { ClassValidatorProvider } from '@ycforge/ydb-orm';
import { UserEntity } from './user.entity';

UserEntity.setValidationProvider(new ClassValidatorProvider());
```

Можно передать группы:

```ts
UserEntity.setValidationProvider(new ClassValidatorProvider({ groups: ['create'] }));
```

## Объявление правил

Правила описываются декораторами `class-validator` на полях сущности:

```ts
import { IsEmail, IsNotEmpty, Length, validate } from 'class-validator';
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

## Пользовательский провайдер

Реализуйте интерфейс `YdbValidationProvider`:

```ts
import type { YdbValidationProvider } from '@ycforge/ydb-orm';

class MyValidationProvider implements YdbValidationProvider {
  async validate(entity: any): Promise<string[]> {
    const errors: string[] = [];
    if (!entity.name) errors.push('name is required');
    if (typeof entity.email !== 'string' || !entity.email.includes('@')) {
      errors.push('email must be a valid email');
    }
    return errors;
  }
}

UserEntity.setValidationProvider(new MyValidationProvider());
```

## Когда выполняется валидация

- `save()` — перед вставкой (`beforeInsert`) и обновлением (`beforeUpdate`);
- `insertMany()` — для каждого элемента перед вставкой.

Ошибка выглядит так:

```
Validation failed for UserEntity: name should not be empty; email must be an email
```