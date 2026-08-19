# Быстрый старт

## 1. Определите сущность

Каждая сущность — класс, наследующий `YdbBaseEntity` и декорированный `@YdbEntity('имя_таблицы')`.

```ts
import {
  YdbBaseEntity,
  YdbEntity,
  YdbColumn,
  YdbPrimaryColumn,
} from '@ycforge/ydb-orm';

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
import { YdbCoreModule } from '@ycforge/ydb-orm';

@Module({
  imports: [
    YdbCoreModule.forRootAsync({
      useFactory: () => ({
        endpoint: 'grpcs://ydb.serverless.yandexcloud.net:2135/?database=/ru-central1/b1g.../ydb',
        auth_type: 'auth_key', // 'meta' | 'auth_key' | 'anonymous'
        authOptions: { authorized_key_path: './authorized_key.json' },
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
import { YdbModule } from '@ycforge/ydb-orm';
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

## 5. Используйте репозиторий

В крупных приложениях операции с сущностью удобно обернуть в `@Injectable()`-класс-репозиторий и инжектировать его туда, где он нужен. Репозиторий просто вызывает статические (Active Record) методы сущности.

```ts
import { Injectable } from '@nestjs/common';
import { UserEntity } from './user.entity';

@Injectable()
export class UsersRepository {
  async create(name: string, email: string) {
    const user = new UserEntity();
    user.name = name;
    user.email = email;
    return UserEntity.save(user);
  }

  findByUuid(uuid: string) {
    return UserEntity.find({ uuid });
  }

  findAll(limit: number, offset: number) {
    return UserEntity.findAll({}, { limit, offset });
  }

  async remove(uuid: string) {
    await UserEntity.delete(uuid);
  }
}
```

Опубликуйте его через контроллер:

```ts
import { Body, Controller, Get, Param, Post } from '@nestjs/common';
import { UsersRepository } from './users.repository';

@Controller('users')
export class UsersController {
  constructor(private readonly users: UsersRepository) {}

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

Зарегистрируйте репозиторий и контроллер в `UserModule` из шага 3 — его `imports` остаются без изменений:

```ts
@Module({
  // ...imports: [YdbModule.forFeature([UserEntity])] — как в шаге 3
  providers: [UsersRepository],
  controllers: [UsersController],
})
export class UserModule {}
```

{% note info %}

Репозиторий — необязательный паттерн: статические методы `UserEntity` можно вызывать из любого места напрямую (см. [Active Record](active-record.md)). Репозиторий просто собирает логику доступа к данным в одном месте.

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