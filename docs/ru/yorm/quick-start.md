# Быстрый старт

## 1. Определите сущность

Каждая сущность — класс, наследующий `YdbBaseEntity` и декорированный `@YdbEntity('имя_таблицы')`.

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

## 2. Подключите корневой модуль

В корневом модуле приложения подключите `YdbCoreModule.forRootAsync()` с параметрами подключения. Feature-модули, регистрирующие сущности, подключаются следующим шагом.

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
        sync: true, // как synchronize в TypeORM — только для dev!
      }),
    }),
  ],
})
export class AppModule {}
```

`forRootAsync` поддерживает `useFactory`, `useClass` и `useExisting` — так же, как в NestJS.

## 3. Подключите forFeature в UserModule

Каждый feature-модуль регистрирует сущности, с которыми работает, через `YdbModule.forFeature([...])`. Это инжектирует в сущность executor (и опционально провайдеры шифрования), делая её статические методы доступными.

```ts
import { Module } from '@nestjs/common';
import { YdbModule } from '@ycforge/yorm';
import { UserEntity } from './user.entity';

@Module({
  imports: [YdbModule.forFeature([UserEntity])],
})
export class UserModule {}
```

Импортируйте `UserModule` в корневой модуль:

```ts
@Module({
  imports: [
    YdbCoreModule.forRootAsync({ ... }), // из шага 2
    UserModule,
  ],
})
export class AppModule {}
```

{% note warning %}

`YdbModule.forFeature([...])` обязателен для каждой сущности: без него статические методы сущности упадут с ошибкой «YDB executor not set».

{% endnote %}

## 4. Используйте Active Record

```ts
import { Injectable } from '@nestjs/common';
import { UserEntity } from './user.entity';

@Injectable()
export class UsersService {
  async demo() {
    // Вставка (uuid генерируется автоматически)
    const user = new UserEntity();
    user.name = 'Иван';
    user.email = 'ivan@example.com';
    await UserEntity.save(user);

    // Чтение
    const found = await UserEntity.find({ uuid: user.uuid });
    console.log(found?.name); // 'Иван'

    // Обновление (save с заполненным uuid)
    user.name = 'Иван Петров';
    await UserEntity.save(user);

    // Список с пагинацией
    const page = await UserEntity.findAll({}, { limit: 50, offset: 0 });

    // Количество
    const total = await UserEntity.count({});

    // Удаление
    await UserEntity.delete(user.uuid);
  }
}
```

## 5. Используйте репозиторий через DI

`YdbModule.forFeature([...])` автоматически регистрирует `YdbRepository<Entity>`. Внедряйте его через `@InjectRepository()` в сервисы — так не приходится обращаться к глобальным статическим методам сущности.

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

Опубликуйте его через контроллер:

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

Зарегистрируйте сервис и контроллер в `UserModule` из шага 3 — его `imports` остаются без изменений:

```ts
@Module({
  // ...imports: [YdbModule.forFeature([UserEntity])] — как в шаге 3
  providers: [UsersService],
  controllers: [UsersController],
})
export class UserModule {}
```

{% note info %}

Репозиторий — необязательный паттерн: статические методы `UserEntity` можно вызывать из любого места напрямую (см. [Active Record](active-record.md)). `YdbRepository` собирает логику доступа к данным в одном месте и проще мокать в тестах. Подробнее о методах репозитория читайте в разделе [Репозиторий](repository.md).

{% endnote %}

## 6. Запустите приложение

```bash
yarn start
```

При старте с опцией `sync: true` ORM создаст недостающие таблицы и колонки автоматически. Для продакшена используйте [миграции](migrations.md) вместо `sync`.

## 7. Проверьте приложение через curl

```bash
# Создайте пользователя
curl -s -X POST http://localhost:3000/users \
  -H 'Content-Type: application/json' \
  -d '{"name":"Иван","email":"ivan@example.com"}'

# Получите пользователя по uuid
curl -s http://localhost:3000/users/<uuid>
```

Подставьте вместо `<uuid>` значение, возвращённое запросом `POST`.