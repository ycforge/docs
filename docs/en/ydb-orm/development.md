# Development

This section is for those developing the `@ycforge/ydb-orm` library itself. Sources live in the `../ydb-orm` directory.

## Requirements

- Node.js ≥ 22, yarn.
- The package builds to ESM (`dist/`).

## Commands

```bash
yarn install       # install dependencies
yarn build         # tsc -p tsconfig.build.json → dist/ (ESM + .d.ts)
yarn test          # all tests (unit + NestJS integration)
yarn test:unit     # unit tests only
yarn test:e2e      # e2e tests (require a database)
yarn test:cov      # tests with coverage
yarn lint          # eslint --fix (src, test, examples)
yarn format        # prettier --write
```

## Source structure

- `src/core/` — types, option interfaces, DI tokens, TS→YDB mapper, SQL utilities.
- `src/decorators/` — all entity decorators.
- `src/entity/` — `YdbBaseEntity` (Active Record) and runtime dependencies.
- `src/encryption/` — encryption provider interfaces.
- `src/metadata/` — metadata collection from Reflect, global entity registry.
- `src/module/` — NestJS integration (`YdbCoreModule`, `YdbModule`, repository-factory).
- `src/schema/` — schema sync and DDL generators.
- `src/transaction/` — `YdbTransactionManager`.
- `src/credentials/` — credentials provider for auth_key.
- `src/migrations/` — migration runner, loader, generator.
- `src/cli/` — the `ydb-orm` binary.
- `examples/` — usage examples (01-basic-crud, 02-relations, 03-encryption, 04-schema-sync).

## Tests

- Unit tests live next to the code (`src/**/*.spec.ts`).
- Integration tests are in `test/nestjs/` (use `Test.createTestingModule`, override `YDB_DRIVER`/`YDB_QUERY` via `overrideProvider`; no network).
- Fixtures are in `test/fixtures/` (entities with relations, encryption, composite PK).
- In ESM mode, jest `jest` is imported from `@jest/globals`.

## Publishing

npm publishing: `prepublishOnly` runs `yarn build`. Only `dist/` + `README.md` + `LICENSE` are included in the package. The package is public (`publishConfig.access: public`).

## Conventions

- Comments and docs are in Russian; identifiers, error messages, and logs are in English.
- Prettier: single quotes, `trailingComma: "all"`.
- Relative imports use the `.js` extension (ESM/`nodenext`).
- YDB queries are parameterized (`query.parameter(...)`) — never concatenate values into SQL.
- Add any new consumer entity to `YdbModule.forFeature([...])`.
- Do not commit or expose secrets (`authorized_key.json`, `.env`).