# Validation

The ORM can validate entities before writing. Validation runs before encryption and the write: if there are errors, an exception with their list is thrown.

## Setting up a provider

The validation provider is set on an entity via `setValidationProvider`. The built-in `ClassValidatorProvider` uses `class-validator` (an optional peer dependency).

```ts
import { ClassValidatorProvider } from '@ycforge/yorm';
import { UserEntity } from './user.entity';

UserEntity.setValidationProvider(new ClassValidatorProvider());
```

You can pass groups:

```ts
UserEntity.setValidationProvider(new ClassValidatorProvider({ groups: ['create'] }));
```

## Declaring rules

Rules are declared with `class-validator` decorators on entity fields:

```ts
import { IsEmail, IsNotEmpty, Length, validate } from 'class-validator';
import { YdbBaseEntity, YdbEntity, YdbColumn, YdbPrimaryColumn } from '@ycforge/yorm';

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

## Custom provider

Implement the `YdbValidationProvider` interface:

```ts
import type { YdbValidationProvider } from '@ycforge/yorm';

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

## When validation runs

- `save()` — before insert (`beforeInsert`) and update (`beforeUpdate`);
- `insertMany()` — for each item before insert.

The error looks like:

```
Validation failed for UserEntity: name should not be empty; email must be an email
```