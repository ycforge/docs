# YCForge

YCForge — набор open-source инструментов для разработчиков. Это не платформа и не фреймворк под конкретный вендор: инструменты самостоятельны и не привязаны напрямую к Яндексу или Yandex Cloud.

## Проекты

### auth

Единая авторизация для API Яндекс Облака и YDB. Один `AuthManager` можно переиспользовать в `@ycforge/yorm` и `@ycforge/orm-security-providers`.

- [Документация](auth/index.md)
- GitHub: [github.com/ycforge/auth](https://github.com/ycforge/auth)
- npm: [npmjs.com/package/@ycforge/auth](https://www.npmjs.com/package/@ycforge/auth)

### yorm

TypeORM-like ORM для YDB: Active Record, связи (relations), шифрование полей с blind index, schema sync, миграции, транзакции и интеграция с NestJS.

- [Документация](yorm/index.md)
- GitHub: [github.com/ycforge/yorm](https://github.com/ycforge/yorm)
- npm: [npmjs.com/package/@ycforge/yorm](https://www.npmjs.com/package/@ycforge/yorm)

### orm-security-providers

Провайдеры шифрования и blind index для yorm: Yandex Cloud KMS и HMAC-SHA256. Документация — в разделе «Криптопровайдеры» документации yorm.

- [Документация](yorm/crypto-providers/encryption/yandex-kms.md)
- GitHub: [github.com/ycforge/orm-security-providers](https://github.com/ycforge/orm-security-providers)
- npm: [npmjs.com/package/@ycforge/orm-security-providers](https://www.npmjs.com/package/@ycforge/orm-security-providers)