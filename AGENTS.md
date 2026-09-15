# AGENTS.md — motion-canvas

Monorepo: npm workspaces (`packages/*`) + Lerna. Run all commands from this dir
(the monorepo root). CI uses Node 24.20.0 (`verify.yml`); `HUSKY: 0` in CI.
Note: `@commitlint/cli@21` and `lerna@10` require Node ≥22 — covered by the CI
pin and the root `engines` floor (`>=24.20.0`).

## Setup & build order

- `npm install` then `npx lerna run build` — full build is prerequisite for
  tests, e2e, examples, docs.
- Dev loop for editor: `npm run template:dev` (Vite watches `core`, `2d`, `ui`,
  `vite-plugin`; `packages/template` is the sample project).
- Player dev requires a prior build: `npm run template:build`, then
  `npm run player:dev`.
- Docs build requires both:
  `npx lerna run build && npx lerna run bundle && npm run docs:build` with
  `NODE_OPTIONS=--max-old-space-size=8192`.
- Docs toolchain: Docusaurus 3.10.2 + React 19. React copies are pinned
  repo-wide via root `package.json` `overrides` — don't remove them. `typedoc`
  (0.25.x) and `rollup` are root devDeps on purpose: typedoc must resolve the
  root TypeScript 5.x (running it against a newer TS breaks it). The docs
  workspace keeps a nested `typescript@~6.0.2` used by its `typecheck` script
  only.

## Package boundaries

- `core`: animation runtime (signals, flow, scenes, threading). Build:
  `tspc -p tsconfig.build.json`; bundle: `rollup -c rollup.config.mjs`.
- `2d`: renderer + editor panels. Split build: `build-lib`
  (`tspc -p src/lib/tsconfig.build.json`) + `build-editor`
  (`rollup -c rollup.editor.mjs`). Unit tests cover only `src/lib/**/*.test.*`.
  The code editor runs CodeMirror (`@codemirror/language@^6.12.4`, shared
  `@lezer/common@^1.5.0`); the docs fiddle uses
  `@codemirror/lang-javascript@^6.2.5`.
- `ui`: editor shell (Preact + `@preact/signals`, Vite). `build` =
  `tsc && vite build`; `type` = `tsc -w`.
- `vite-plugin`: plain `tsc` build, peer `vite 4.x || 5.x`. Its `skipLibCheck`
  is not accidental — keep it.
- `ffmpeg`: dual `client/tsconfig.json` + `server/tsconfig.json` builds; license
  GPLv3 (others MIT).
- `player`: Vite web-component consumer of built packages.
- `internal`: private build helpers only — includes `vite/markdown-literals`
  plugin required by `core`/`2d` vitest configs.
- `docs`: private, not published. Docusaurus 3.10.2 site (React 19, MDX v3,
  `plugin-svgr`); the `typedoc.js` plugin regenerates `src/generated` during
  production builds.
- `e2e` / `examples` / `template`: private, not published.

## Verify (mirrors `verify.yml`)

- Lint: `npx eslint "**/*.ts?(x)"` — quote the glob. Flat config lives in
  `eslint.config.mjs` (eslint 10; eslintrc is not supported). Companion pins:
  `@typescript-eslint/*` v8, `eslint-plugin-tsdoc` 0.5.x, `@eslint/js` v10,
  `globals` v17.
- Style check: `npm run prettier` (`--check`); fix with `npm run prettier:fix` /
  `npm run eslint:fix`.
- Unit: `npx lerna run build && npx lerna run test`, or per package:
  `npm run core:test`, `npm run 2d:test`.
- Single test: `npx vitest run <path>` from `packages/core` or `packages/2d`
  (jsdom env; `core` uses `vitest.setup.ts`).
- E2E: `npm run e2e:test` (Playwright **Firefox** headless +
  `jest-image-snapshot@^6.5.2`; spins up Vite server itself). Failure diffs:
  `packages/e2e/src/__image_snapshots__/__diff_output__`. In containers set
  `HOME=/root` (see `verify.yml`). The matcher is consumed only by
  `packages/e2e`; 6.5.2 drops the `rimraf` runtime dep (uses native `fs.rmSync`)
  and adds `runtimeHooksPath` / `maxChildProcessBufferSizeInBytes` plus base64 /
  TypedArray input support.
- UI types: `npm run ui:type`.
- Docs types: `npm run typecheck -w packages/docs` (docs-nested TS ~6.0.2).
- Docs build: `npm run docs:build` (expensive; needs a prior
  `npx lerna run build && npx lerna run bundle`).

## Conventions

- Conventional Commits enforced by commitlint + Husky. Scope must be one of:
  `2d, core, create, docs, e2e, examples, ffmpeg, legacy, player, ui, vite-plugin`.
- ESLint extras that bite: `explicit-member-accessibility` (always write
  `public/private`), `grouped-accessor-pairs: getBeforeSet`, `eqeqeq` (except
  `null`), TSDoc `tsdoc/syntax` errors, strict `naming-convention` (camelCase
  vars/fns, PascalCase types/enums, `T`-prefixed type params, unused params must
  be `_`-prefixed).
- Prettier: `singleQuote`, `bracketSpacing: false`, 80 col, `organize-imports`
  plugin auto-sorts imports — don't hand-order them.
- Ignored by lint/style: `**/*.js`, `**/*.d.ts`, `packages/template`,
  `packages/create/template-*`.
- Never edit generated output: `packages/*/lib|dist|build`,
  `packages/2d/editor`, `packages/docs/src/generated`.
- `core`/`2d` manifests pin `types: ["node"]` and `lib` includes ES2022: hoisted
  `@types/*` packages from the docs workspace (e.g. `@types/mdx`) must not leak
  into library builds, and `Array.prototype.at` typing comes from `lib`, not
  from `@types/node` polyfills.
