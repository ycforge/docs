# Установка

## Требования

- **Node.js** ≥ 22 (ESM, `"type": "module"` в `package.json`).
- **yarn** или **npm** — любой пакетный менеджер.

## Установка пакета

```bash
yarn add @ycforge/yorm
```

или

```bash
npm install @ycforge/yorm
```

## Peer-зависимости

Для NestJS-интеграции добавьте peer-зависимости:

```bash
yarn add @nestjs/common @nestjs/core reflect-metadata rxjs
```

{% note info %}

Для работы декораторов нужен `reflect-metadata`. Импортируйте его один раз в точке входа приложения (`main.ts`).

{% endnote %}

## Включение в tsconfig

Так как пакет — ESM (`module: nodenext`), убедитесь, что в вашем `tsconfig.json`:

```json
{
  "compilerOptions": {
    "target": "ES2023",
    "module": "nodenext",
    "moduleResolution": "nodenext",
    "experimentalDecorators": true,
    "emitDecoratorMetadata": true,
    "esModuleInterop": true,
    "strict": true
  }
}
```

Обязательные опции для работы декораторов — `experimentalDecorators` и `emitDecoratorMetadata`.

## Проверка установки

```bash
node -e "import('@ycforge/yorm').then(m => console.log(Object.keys(m).length + ' экспортов'))"
```

Если команда вывела число экспортов — пакет установлен корректно.