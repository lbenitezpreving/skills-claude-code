# Configuración Vitest + Coverage + Stryker

## vite.config.ts — configuración actual del proyecto

```ts
import { defineConfig } from 'vite';
import react from '@vitejs/plugin-react';

export default defineConfig({
  plugins: [react()],
  server: {
    port: 3000,
    proxy: {
      '/api': {
        target: 'http://localhost:8000',
        changeOrigin: true,
        rewrite: (path) => path.replace(/^\/api/, ''),
      },
    },
  },
  test: {
    globals: true,
    environment: 'jsdom',
    setupFiles: './src/test/setup.ts',
    exclude: ['node_modules', 'e2e'],
    coverage: {
      provider: 'v8',
      reporter: ['text', 'html'],
      reportsDirectory: './coverage',
      exclude: [
        'node_modules/',
        'src/test/',
        'e2e/',
        '**/*.test.{ts,tsx}',
        '**/*.spec.{ts,tsx}',
      ],
    },
  },
});
```

## Añadir threshold de cobertura (80%)

```ts
coverage: {
  provider: 'v8',
  reporter: ['text', 'html'],
  reportsDirectory: './coverage',
  thresholds: {
    lines: 80,
    functions: 80,
    branches: 80,
    statements: 80,
  },
  exclude: [
    'node_modules/',
    'src/test/',
    'e2e/',
    '**/*.test.{ts,tsx}',
    '**/*.spec.{ts,tsx}',
    'src/main.tsx',      // entrypoint — no testear
    'src/vite-env.d.ts', // declaraciones de tipo
  ],
},
```

## Instalar dependencias de testing

```bash
# Core de Testing Library
npm install -D @testing-library/react @testing-library/user-event @testing-library/jest-dom

# Cobertura
npm install -D @vitest/coverage-v8

# Mutation testing
npm install -D @stryker-mutator/core @stryker-mutator/vitest-runner
```

## Archivo src/test/setup.ts

```ts
import '@testing-library/jest-dom';
```

## Stryker: stryker.config.mjs

```js
// @ts-check
/** @type {import('@stryker-mutator/api/core').PartialStrykerOptions} */
const config = {
  testRunner: 'vitest',
  reporters: ['html', 'clear-text', 'progress'],
  coverageAnalysis: 'perTest',
  vitest: {
    configFile: 'vite.config.ts',
  },
  mutate: [
    'src/**/*.{ts,tsx}',
    '!src/**/*.test.{ts,tsx}',
    '!src/**/*.spec.{ts,tsx}',
    '!src/test/**',
    '!src/main.tsx',
  ],
  thresholds: {
    high: 80,
    low: 60,
    break: 50,
  },
};

export default config;
```

## Comandos de ejecución

```bash
# Tests con cobertura
npx vitest run --coverage

# Tests en modo watch (desarrollo)
npx vitest

# Solo un archivo
npx vitest run src/components/KanbanBoard/KanbanBoard.test.tsx

# Mutation testing (tarda más)
npx stryker run

# Ver reporte de cobertura
open coverage/index.html
```
