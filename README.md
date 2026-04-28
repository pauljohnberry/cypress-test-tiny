# cypress-test-tiny — TS5011 Reproduction

Minimal reproduction for [Cypress e2e failing with TypeScript 6](https://github.com/cypress-io/cypress/issues/33648).

## Bug

Cypress 15.14.1 e2e tests fail with TypeScript 6 when the project tsconfig has `outDir`:

```
Error: Webpack Compilation Error
[tsl] ERROR
      TS5011: The common source directory of 'tsconfig.json' is './e2e'.
      The 'rootDir' setting must be explicitly set to this or another path
      to adjust your output's file layout.
```

## Steps to Reproduce

```bash
npm install
npx cypress run --e2e
```

The test will fail with the TS5011 error shown above.

You can also see the `rootDir: undefined` line in the Cypress binary cache:

```bash
# macOS — adjust path for your OS
TSLOADER="$(npx cypress cache path)/15.14.1/Cypress.app/Contents/Resources/app/packages/server/node_modules/ts-loader/dist/index.js"
grep -n "rootDir" "$TSLOADER"
```

With debug logging enabled, you'll also see the monkey-patching guard skip TS6:

```bash
DEBUG=cypress:* npx cypress run --e2e 2>&1 | grep monkey
# cypress:webpack typescript version 6.0.3 is not supported for monkey-patching
```

## Investigation

The bundled `ts-loader@9.5.2` (in `@cypress/webpack-batteries-included-preprocessor`) calls `transpileModule` with `rootDir: undefined` ([source](https://github.com/TypeStrong/ts-loader/blob/v9.5.2/src/index.ts#L419)). TypeScript 6 now requires explicit `rootDir` when `outDir` is set, so the `undefined` override triggers TS5011.

This was fixed in ts-loader 9.5.7: [TypeStrong/ts-loader#1678](https://github.com/TypeStrong/ts-loader/issues/1678)

## Possible Fix

Bump `ts-loader` from `9.5.2` to `>=9.5.7` in [`npm/webpack-batteries-included-preprocessor/package.json`](https://github.com/cypress-io/cypress/blob/develop/npm/webpack-batteries-included-preprocessor/package.json).

## Workaround

Use `@cypress/webpack-preprocessor` with a project-installed `ts-loader@>=9.5.7`:

```bash
npm install --save-dev @cypress/webpack-preprocessor ts-loader
```

```typescript
// cypress.config.ts
import createBundler from '@cypress/webpack-preprocessor';

export default defineConfig({
  e2e: {
    setupNodeEvents(on) {
      on('file:preprocessor', createBundler({
        webpackOptions: {
          resolve: { extensions: ['.ts', '.js'] },
          module: {
            rules: [{ test: /\.ts$/, exclude: /node_modules/, loader: 'ts-loader',
              options: {
                transpileOnly: true,
                compilerOptions: { outDir: undefined, rootDir: '.' },
              } }]
          }
        }
      }));
    }
  }
});
```

Note: the `compilerOptions` override is needed because `transpileModule` still inherits `outDir` from the tsconfig, and TS6 requires `rootDir` when `outDir` is set.
