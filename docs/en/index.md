# YCForge

YCForge is a set of open-source tools for developers. It is not a platform or a vendor-specific framework: the tools are independent and not directly tied to Yandex or Yandex Cloud.

## Projects

### auth

Shared authentication for Yandex Cloud APIs and YDB. One `AuthManager` can be reused across `@ycforge/ydb-orm` and `@ycforge/orm-security-providers`.

- [Documentation](auth/index.md)
- GitHub: [github.com/ycforge/auth](https://github.com/ycforge/auth)
- npm: [npmjs.com/package/@ycforge/auth](https://www.npmjs.com/package/@ycforge/auth)

### ydb-orm

TypeORM-like ORM for YDB: Active Record, relations, field encryption with blind index, schema sync, migrations, transactions, and NestJS integration.

- [Documentation](ydb-orm/index.md)
- GitHub: [github.com/ycforge/ydb-orm](https://github.com/ycforge/ydb-orm)
- npm: [npmjs.com/package/@ycforge/ydb-orm](https://www.npmjs.com/package/@ycforge/ydb-orm)

### orm-security-providers

Encryption and blind index providers for ydb-orm: Yandex Cloud KMS and HMAC-SHA256. Documentation lives in the "Crypto providers" section of the ydb-orm docs.

- [Documentation](ydb-orm/crypto-providers/encryption/yandex-kms.md)
- GitHub: [github.com/ycforge/orm-security-providers](https://github.com/ycforge/orm-security-providers)
- npm: [npmjs.com/package/@ycforge/orm-security-providers](https://www.npmjs.com/package/@ycforge/orm-security-providers)