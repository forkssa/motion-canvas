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
  root TypeScript (0.25.x only supports ≤5.4.x). The root `typescript` is pinned
  to `~5.4.2` for the same reason — `ts-patch@3.0.2` (`tspc`, the patched `tsc`
  driving the `core`/`2d` builds) can't slice TS ≥5.5 — so the range must not be
  widened until both move. The docs workspace keeps a nested `typescript@~6.0.2`
  used by its `typecheck` script only; the `api-extractor` under
  `vite-plugin-dts` ships its own private `typescript@5.9` (harmless, isolated).

## Package boundaries

- `core`: animation runtime (signals, flow, scenes, threading). Build:
  `tspc -p tsconfig.build.json`; bundle: `rollup -c rollup.config.mjs`.
- `2d`: renderer + editor panels. Split build: `build-lib`
  (`tspc -p src/lib/tsconfig.build.json`) + `build-editor`
  (`rollup -c rollup.editor.mjs`). Unit tests cover only `src/lib/**/*.test.*`.
  The code editor runs CodeMirror (`@codemirror/language@^6.12.4`, shared
  `@lezer/common@^1.5.0`, `@lezer/highlight@^1.2.3`); the docs fiddle uses
  `@codemirror/lang-javascript@^6.2.5` over `@lezer/javascript@^1.5.4`.
- `ui`: editor shell (Preact + `@preact/signals`, Vite). `build` =
  `tsc && vite build`; `type` = `tsc -w`. Its `vite-plugin-dts` stays on
  `^4.5.4` on purpose: 4.5.x dropped the hoisted `vue-tsc`/`@volar` dependency
  stack (which used to collide with the `prettier-plugin-organize-imports` peer
  range), and 5.x turns `@microsoft/api-extractor` into a required peer — take
  it only together with an explicit api-extractor devDep. `rollupTypes: true`
  (CI builds) exercises api-extractor. `@preact/signals` 2.x peers on
  `preact >= 10.25.0` — bump preact alongside it (currently `^10.29.8`). preact
  ≥10.22 ships React-style JSX types: element-specific attribute bases
  (`AnchorHTMLAttributes`, `InputHTMLAttributes`, …) are imported from `preact`
  (not `JSX.HTMLAttributes`), and `Ref`/`RefObject` live in `preact` — `Ref` is
  the `RefObject | RefCallback | null` union, so ref objects accessed via
  `.current` should be typed `RefObject<T>`; JSX spreads (but not literal attrs)
  tolerate the extra `div`-level `disabled` forwarded by `AudioClip`.
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
  `npm run core:test`, `npm run 2d:test`. Always run test commands
  (`lerna run test`, `core:test`, `2d:test`, …) with a 60s timeout (e.g.
  `timeout 60s npm run core:test`): on a TTY lerna's Nx-powered task UI stays
  alive after the run finishes, waiting for `q`, and the per-package scripts
  invoke bare `vitest`, which drops into watch mode.
- Single test: `npx vitest run <path>` from `packages/core` or `packages/2d`
  (jsdom env; `core` uses `vitest.setup.ts`).
- E2E: `npm run e2e:test -- run` (non-interactive; plain `npm run e2e:test`
  drops into vitest watch mode on a TTY) — Playwright **Firefox** headless +
  `jest-image-snapshot@^6.5.2`; spins up Vite server itself. Failure diffs:
  `packages/e2e/src/__image_snapshots__/__diff_output__`. In containers set
  `HOME=/root` (see `verify.yml`). `playwright@^1.63.0` pins its own browser
  builds (`firefox-1543` / `ffmpeg-1011`): run `npx playwright install firefox`
  after any playwright upgrade, and keep `verify.yml`'s e2e container image
  (`mcr.microsoft.com/playwright:v1.63.0-jammy`) in lockstep with the npm
  version. The matcher is consumed only by `packages/e2e`; 6.5.2 drops the
  `rimraf` runtime dep (uses native `fs.rmSync`) and adds `runtimeHooksPath` /
  `maxChildProcessBufferSizeInBytes` plus base64 / TypedArray input support.
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
- Prettier (root devDep, `^3.9.6`, repo-wide check is clean): `singleQuote`,
  `bracketSpacing: false`, 80 col, `organize-imports` plugin (`^4.3.0`)
  auto-sorts imports — don't hand-order them. The sort order follows the root
  TypeScript version (5.4.x sorts case-sensitively: `DetailedError` before
  `beginSlide`; TS ≥5.9 flips to case-insensitive and would flag every barrel
  `index.ts`), which is another reason `typescript` stays pinned to `~5.4.2`.
  3.4+ normalizes TS modifier order (`declare public readonly`, never
  `public declare`) and 3.5+ collapses unions that fit the print width to one
  line — write new code in the post-3.9 style above.
- Ignored by lint/style: `**/*.js`, `**/*.d.ts`, `packages/template`,
  `packages/create/template-*`.
- Never edit generated output: `packages/*/lib|dist|build`,
  `packages/2d/editor`, `packages/docs/src/generated`.
- `core`/`2d` manifests pin `types: ["node"]` and `lib` includes ES2022: hoisted
  `@types/*` packages from the docs workspace (e.g. `@types/mdx`) must not leak
  into library builds, and `Array.prototype.at` typing comes from `lib`, not
  from `@types/node` polyfills.
