# YCForge

YCForge is a set of open-source tools for developers. It is not a platform or a vendor-specific framework: the tools are independent and not directly tied to Yandex or Yandex Cloud.

## Projects

### auth

Shared authentication for Yandex Cloud APIs and YDB. One `AuthManager` can be reused across `@ycforge/yorm` and `@ycforge/orm-security-providers`.

- [Documentation](auth/index.md)
- GitHub: [github.com/ycforge/auth](https://github.com/ycforge/auth)
- npm: [npmjs.com/package/@ycforge/auth](https://www.npmjs.com/package/@ycforge/auth)

### yorm

TypeORM-like ORM for YDB: Active Record, relations, field encryption with blind index, schema sync, migrations, transactions, and NestJS integration.

- [Documentation](yorm/index.md)
- GitHub: [github.com/ycforge/yorm](https://github.com/ycforge/yorm)
- npm: [npmjs.com/package/@ycforge/yorm](https://www.npmjs.com/package/@ycforge/yorm)

### orm-security-providers

Encryption and blind index providers for yorm: Yandex Cloud KMS and HMAC-SHA256. Documentation lives in the "Crypto providers" section of the yorm docs.

- [Documentation](yorm/crypto-providers/encryption/yandex-kms.md)
- GitHub: [github.com/ycforge/orm-security-providers](https://github.com/ycforge/orm-security-providers)
- npm: [npmjs.com/package/@ycforge/orm-security-providers](https://www.npmjs.com/package/@ycforge/orm-security-providers)