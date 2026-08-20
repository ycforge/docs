# Разработка

Раздел для тех, кто развивает саму библиотеку `@ycforge/ydb-orm`. Исходники — в каталоге `../ydb-orm`.

## Требования

- Node.js ≥ 22, yarn.
- Пакет собирается в ESM (`dist/`).

## Команды

```bash
yarn install       # установка зависимостей
yarn build         # tsc -p tsconfig.build.json → dist/ (ESM + .d.ts)
yarn test          # все тесты (unit + NestJS-интеграционные)
yarn test:unit     # только unit-тесты
yarn test:e2e      # e2e-тесты (требуют БД)
yarn test:cov      # тесты с покрытием
yarn lint          # eslint --fix (src, test, examples)
yarn format        # prettier --write
```

## Структура исходников

- `src/core/` — типы, интерфейсы опций, DI-токены, маппер TS→YDB, утилиты SQL, валидация опций.
- `src/decorators/` — все декораторы сущностей.
- `src/entity/` — `YdbBaseEntity` (Active Record фасад) и рантайм-зависимости.
- `src/persistence/` — `YdbEntityPersistence`: CRUD, шифрование, lifecycle hooks.
- `src/relations/` — `YdbEntityRelations`: eager/lazy relations, many-to-many.
- `src/repository/` — `YdbRepository`, `YdbEntityManager`, `InjectRepository`.
- `src/encryption/` — интерфейсы провайдеров шифрования.
- `src/metadata/` — сбор метаданных из Reflect, глобальный реестр сущностей, валидация.
- `src/module/` — NestJS-интеграция (`YdbCoreModule`, `YdbModule`, repository-factory).
- `src/schema/` — schema sync и генераторы DDL.
- `src/transaction/` — `YdbTransactionManager`.
- `src/credentials/` — credentials provider для auth_key.
- `src/migrations/` — runner, loader, generator миграций.
- `src/cli/` — бинарь `ydb-orm` (completion, diff, миграции).
- `examples/` — примеры использования (01-basic-crud, 02-relations, 03-encryption, 04-schema-sync).

## Тесты

- Unit-тесты лежат рядом с кодом (`src/**/*.spec.ts`).
- Интеграционные тесты — в `test/nestjs/` (используют `Test.createTestingModule`, подменяют `YDB_DRIVER`/`YDB_QUERY` через `overrideProvider`, сети нет).
- Фикстуры — в `test/fixtures/` (сущности с relations, encryption, составным PK).
- В ESM-режиме jest `jest` импортируется из `@jest/globals`.

## Публикация

Публикация в npm: `prepublishOnly` запускает `yarn build`. В пакет попадает только `dist/` + `README.md` + `LICENSE`. Пакет публичный (`publishConfig.access: public`).

## Соглашения

- Комментарии и документация — на русском; идентификаторы, сообщения ошибок и логи — на английском.
- Prettier: одинарные кавычки, `trailingComma: "all"`.
- Относительные импорты — с расширением `.js` (ESM/`nodenext`).
- Запросы к YDB параметризованы (`query.parameter(...)`) — не конкатенируйте значения в SQL.
- Новую сущность-потребителя добавляйте в `YdbModule.forFeature([...])`.
- Секреты (`authorized_key.json`, `.env`) не коммитить и не выводить.