# Quick start

## 1. Define an entity

Each entity is a class extending `YdbBaseEntity` and decorated with `@YdbEntity('table_name')`.

```ts
import {
  YdbBaseEntity,
  YdbEntity,
  YdbColumn,
  YdbPrimaryColumn,
} from '@ycforge/yorm';

@YdbEntity('users')
export class UserEntity extends YdbBaseEntity {
  @YdbPrimaryColumn('Uuid')
  uuid: string;

  @YdbColumn('Utf8')
  name: string;

  @YdbColumn('Utf8')
  email: string;
}
```

## 2. Connect the core module

In the application's root module, register `YdbCoreModule.forRootAsync()` with the connection options. Feature modules that register entities are connected in the next step.

```ts
import { Module } from '@nestjs/common';
import { YdbCoreModule } from '@ycforge/yorm';
import { createAuth, authKeyFromFile } from '@ycforge/auth';

@Module({
  imports: [
    YdbCoreModule.forRootAsync({
      useFactory: () => ({
        endpoint: 'grpcs://ydb.serverless.yandexcloud.net:2135/?database=/ru-central1/b1g.../ydb',
        auth: createAuth(authKeyFromFile('./authorized_key.json')),
        sync: true, // like synchronize in TypeORM — dev only!
      }),
    }),
  ],
})
export class AppModule {}
```

`forRootAsync` supports `useFactory`, `useClass`, and `useExisting` — exactly like NestJS.

## 3. Connect forFeature in UserModule

Each feature module registers the entities it works with via `YdbModule.forFeature([...])`. This injects the executor (and optional encryption providers) into the entity, making its static methods usable.

```ts
import { Module } from '@nestjs/common';
import { YdbModule } from '@ycforge/yorm';
import { UserEntity } from './user.entity';

@Module({
  imports: [YdbModule.forFeature([UserEntity])],
})
export class UserModule {}
```

Import `UserModule` into the root module:

```ts
@Module({
  imports: [
    YdbCoreModule.forRootAsync({ ... }), // from step 2
    UserModule,
  ],
})
export class AppModule {}
```

{% note warning %}

`YdbModule.forFeature([...])` is required for each entity: without it, static entity methods will fail with a "YDB executor not set" error.

{% endnote %}

## 4. Use Active Record

```ts
import { Injectable } from '@nestjs/common';
import { UserEntity } from './user.entity';

@Injectable()
export class UsersService {
  async demo() {
    // Insert (uuid is generated automatically)
    const user = new UserEntity();
    user.name = 'John';
    user.email = 'john@example.com';
    await UserEntity.save(user);

    // Read
    const found = await UserEntity.find({ uuid: user.uuid });
    console.log(found?.name); // 'John'

    // Update (save with a filled uuid)
    user.name = 'John Doe';
    await UserEntity.save(user);

    // List with pagination
    const page = await UserEntity.findAll({}, { limit: 50, offset: 0 });

    // Count
    const total = await UserEntity.count({});

    // Delete
    await UserEntity.delete(user.uuid);
  }
}
```

## 5. Use a repository with DI

`YdbModule.forFeature([...])` automatically registers a `YdbRepository<Entity>`. Inject it with `@InjectRepository()` in your services so you don't need to call the entity's global static methods.

```ts
import { Injectable } from '@nestjs/common';
import { InjectRepository, YdbRepository } from '@ycforge/yorm';
import { UserEntity } from './user.entity';

@Injectable()
export class UsersService {
  constructor(
    @InjectRepository(UserEntity)
    private readonly users: YdbRepository<UserEntity>,
  ) {}

  async create(name: string, email: string) {
    const user = new UserEntity();
    user.name = name;
    user.email = email;
    return this.users.save(user);
  }

  findByUuid(uuid: string) {
    return this.users.findOneBy({ uuid });
  }

  findAll(limit: number, offset: number) {
    return this.users.findAll({}, { limit, offset });
  }

  async remove(uuid: string) {
    await this.users.delete(uuid);
  }
}
```

Expose it through a controller:

```ts
import { Body, Controller, Get, Param, Post } from '@nestjs/common';
import { UsersService } from './users.service';

@Controller('users')
export class UsersController {
  constructor(private readonly users: UsersService) {}

  @Post()
  create(@Body() body: { name: string; email: string }) {
    return this.users.create(body.name, body.email);
  }

  @Get(':uuid')
  findOne(@Param('uuid') uuid: string) {
    return this.users.findByUuid(uuid);
  }
}
```

Register the service and the controller in the `UserModule` from step 3 — its `imports` stay unchanged:

```ts
@Module({
  // ...imports: [YdbModule.forFeature([UserEntity])] — as in step 3
  providers: [UsersService],
  controllers: [UsersController],
})
export class UserModule {}
```

{% note info %}

The repository pattern is optional — you can call `UserEntity` static methods from anywhere directly (see [Active Record](active-record.md)). `YdbRepository` keeps data-access logic in one place and is easier to mock in tests. See the [Repository](repository.md) section for the full list of repository methods.

{% endnote %}

## 6. Run the application

```bash
yarn start
```

On startup with `sync: true`, the ORM creates missing tables and columns automatically. For production, use [migrations](migrations.md) instead of `sync`.

## 7. Test the application with curl

```bash
# Create a user
curl -s -X POST http://localhost:3000/users \
  -H 'Content-Type: application/json' \
  -d '{"name":"John","email":"john@example.com"}'

# Fetch the user by uuid
curl -s http://localhost:3000/users/<uuid>
```

Replace `<uuid>` with the value returned by the `POST` request.