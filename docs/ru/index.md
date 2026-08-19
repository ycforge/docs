# YCForge

YCForge — набор open-source инструментов для разработчиков. Это не платформа и не фреймворк под конкретный вендор: инструменты самостоятельны и не привязаны напрямую к Яндексу или Yandex Cloud.

## Проекты

### ydb-orm

TypeORM-like ORM для YDB: Active Record, связи (relations), шифрование полей с blind index, schema sync, миграции, транзакции и интеграция с NestJS.

- [Документация](ydb-orm/index.md)
- GitHub: [github.com/ycforge/ydb-orm](https://github.com/ycforge/ydb-orm)
- npm: [npmjs.com/package/@ycforge/ydb-orm](https://www.npmjs.com/package/@ycforge/ydb-orm)

### orm-security-providers

Провайдеры шифрования и blind index для ydb-orm: Yandex Cloud KMS и HMAC-SHA256. Документация — в разделе «Криптопровайдеры» документации ydb-orm.

- [Документация](ydb-orm/crypto-providers/encryption/yandex-kms.md)
- GitHub: [github.com/ycforge/orm-security-providers](https://github.com/ycforge/orm-security-providers)
- npm: [npmjs.com/package/@ycforge/orm-security-providers](https://www.npmjs.com/package/@ycforge/orm-security-providers)