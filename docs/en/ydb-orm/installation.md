# Installation

## Requirements

- **Node.js** ≥ 22 (ESM, `"type": "module"` in `package.json`).
- **yarn** or **npm** — any package manager.

## Installing the package

```bash
yarn add @ycforge/ydb-orm
```

or

```bash
npm install @ycforge/ydb-orm
```

## Peer dependencies

The core package has no required peer dependencies. For [NestJS integration](nest/overview.md), add:

```bash
yarn add @nestjs/common @nestjs/core reflect-metadata rxjs
```

{% note info %}

Decorators require `reflect-metadata`. Import it once in the application entry point.

{% endnote %}

## tsconfig

Since the package is ESM (`module: nodenext`), make sure your `tsconfig.json` has:

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

The mandatory options for decorators are `experimentalDecorators` and `emitDecoratorMetadata`.

## Verify the installation

```bash
node -e "import('@ycforge/ydb-orm').then(m => console.log(Object.keys(m).length + ' exports'))"
```

If the command prints the number of exports, the package is installed correctly.