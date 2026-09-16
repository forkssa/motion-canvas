# AGENTS.md — motion-canvas

Monorepo: npm workspaces (`packages/*`) + Lerna. Run all commands from this dir
(the monorepo root). CI uses Node 24.20.0 (`verify.yml`); `HUSKY: 0` in CI.
Note: `@commitlint/cli@21` and `lerna@10` require Node ≥22 — covered by the CI
pin and the root `engines` floor (`>=24.20.0`). Build/test toolchain: the root
`vite@^8.3.0` (Rolldown + Oxc, ESM-only, Lightning CSS minifier) drives every
Vite workspace (`ui`, `player`, `examples`, `template`, `e2e`); unit tests run
on `vitest@^5.0.1` (`core`, `2d`, `e2e`), whose `vite` peer is
`^6.4.0 || ^7 || ^8` and is satisfied by the hoisted root copy.
`@preact/preset-vite` is `^2.10.6` (first line with a Vite 8-compatible peer
range).

## Setup & build order

- `npm install` then `npx lerna run build` — full build is prerequisite for
  tests, e2e, examples, docs and the editor dev loop.
- npm 11 gates install scripts behind root `allowScripts`:
  `npm install-scripts ls` must report no unreviewed packages. Approve with
  `npm install-scripts approve --all` (writes pinned entries) and run the newly
  approved hook with `npm rebuild <pkg> --foreground-scripts`; drop entries
  whose package left the tree with `npm install-scripts prune`. Currently
  approved (all installed): `core-js`, `ffmpeg-ffprobe-static@6.1.2-rc.1`,
  `nx@23.2.1`, `@parcel/watcher@2.6.0` (optional dep of `chokidar`/sass; its
  `build-from-source.js` hook resolves the prebuilt
  `@parcel/watcher-linux-x64-glibc` binding).
- Dev loop for editor: `npm run template:dev` (Vite watches `core`, `2d`, `ui`
  and the ffmpeg client through the config aliases; `packages/template` is the
  sample project). The config loads the built `@motion-canvas/vite-plugin` /
  `@motion-canvas/ffmpeg` and `editorPlugin` reads `@motion-canvas/ui`'s
  `dist/editor.html`, so keep `npm run vite-plugin:dev` (`tsc -w`) running for
  plugin edits — Vite restarts on `lib` changes.
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
  `tspc -p tsconfig.build.json`; bundle: `rollup -c rollup.config.mjs`. Colors
  wrap chroma-js (`chroma-js@^3.2.0` + `@types/chroma-js@^3.1.2`; previously
  exact-pinned to `2.4.2`/`2.4.4` to dodge the broken `2.5.0-*` pre-releases).
  `@types/chroma-js` 3.x is ESM-style (`export default chroma`), types no named
  value exports, and its internal `chroma` namespace cannot be module-augmented,
  so `src/types/Color.ts` uses the default import, declares a local
  `interface Color extends ChromaColor, Type, WebGLConvertible` paired with
  `export const Color: ColorStatic`, and redeclares chroma's chainable methods
  (parameters via `Parameters<ChromaColor[...]>`) to return the extended
  `Color`. Keep that interface/const pair as-is: exporting an aliased interface
  instead makes declaration emit fail downstream (TS4058 in `2d`). `Color.css()`
  / `serialize()` emit modern space-separated CSS (`rgb(0 0 0)`), which is what
  lands in meta files and the UI color input now.
- `2d`: renderer + editor panels. Split build: `build-lib`
  (`tspc -p src/lib/tsconfig.build.json`) + `build-editor`
  (`rollup -c rollup.editor.mjs`). Unit tests cover only `src/lib/**/*.test.*`.
  The code editor runs CodeMirror (`@codemirror/language@^6.12.4`, shared
  `@lezer/common@^1.5.0`, `@lezer/highlight@^1.2.3`); the docs fiddle uses
  `@codemirror/lang-javascript@^6.2.5` over `@lezer/javascript@^1.5.4`. The
  deprecated `CodeBlock` component and its `code-fns` dependency were removed in
  4.0.0 — `Code` (`src/lib/components/Code.ts`) is the only code node; it keeps
  `  nodeName('CodeBlock')` so older scenes still deserialize. SVG path parsing
  uses `parse-svg-path@^0.2.0`, which ships its own types (default export
  `parse`, `Command` = `[string, ...number[]]` imported as
  `{Command as PathCommand}` in `getPathProfile.ts`) — the old ambient shim
  `src/lib/parse-svg-path.d.ts` was removed in favor of the upstream
  declarations.
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
  tolerate the extra `div`-level `disabled` forwarded by `AudioClip`. The color
  controls import `chroma-js` directly without declaring it — it resolves
  through the workspace-hoisted copy pulled in by `core` and Vite bundles it
  into `dist` (externals stay `@motion-canvas/core` and preact). Since
  `@types/chroma-js` 3.x only types the default export, always use
  `import chroma from 'chroma-js'` (`chroma.hsv(...)`, `chroma.valid(...)`) —
  named value imports no longer type-check. Class names come from `clsx@^2.1.1`
  (default import; a single workspace-hoisted copy shared with `2d`/`docs` —
  `2d` uses the named `{clsx}` import, also valid in 2.x). `sass@^1.104.1` and
  `highlight.js@^11.12.0` are devDeps. Vite 8 drives Dart Sass through the
  modern API, so the `silenceDeprecations: ['legacy-js-api']` workaround was
  dropped in the Vite 8 upgrade (only `packages/2d/rollup.editor.mjs` keeps it —
  `rollup-plugin-postcss` still calls the legacy API). The lib build pins
  `build.lib.cssFileName: 'style'`: Vite 6+ otherwise names the CSS output after
  `build.lib.fileName` (`main.css`), while `editorPlugin` injects
  `dist/style.css` and `packages/docs/editor.js` serves `/editor/style.css`.
  `vite.showcase.ts` pins the same name. Its `vite.config.ts` uses
  `build.rolldownOptions` (the Vite 8 rename) and the package is
  `"type": "module"`, so it loads as ESM.
- `vite-plugin`: plain `tsc` build, peer `vite ^8.0.0`. Its `skipLibCheck` is
  not accidental — keep it. The package stays CommonJS (no `"type": "module"`)
  but its tsconfig uses `module`/`moduleResolution` `node16` and every `vite`
  import is a type-only `import type … with {'resolution-mode': 'import'}` —
  Vite 8's declarations are ESM-only and cannot be resolved, let alone
  value-imported, under legacy `node` resolution (TS1479). The emitted `lib`
  must stay free of `require('vite')`; the one runtime use (`normalizePath` in
  `src/partials/webgl.ts`) was replaced with an inline slash conversion. JSX
  goes through
  `oxc: {jsx: {runtime: 'automatic', importSource: '@motion-canvas/2d/lib'}}`
  (the `esbuild` option is deprecated in Vite 8), the non-editor `build.target`
  is `'baseline-widely-available'` (`'modules'` was removed in Vite 7), and
  build hooks use `rolldownOptions`. The virtual `\0virtual:editor` module emits
  project imports via `path.resolve(filePath)` — Vite 8/Rolldown no longer
  resolves relative specifiers inside virtual modules against the process cwd.
  Project globs (`project` entries such as `src/*.ts`) are expanded with
  `fast-glob@^3.3.3` in `src/utils.ts` (`fg.isDynamicPattern()` +
  `fg.sync(..., {onlyFiles: true})`); that range also keeps the hoisted
  `micromatch` on the patched 4.0.8 line. `mime-types@^3.0.2`
  (`src/partials/exporter.ts`) still ships no types — keep the
  `@types/mime-types@^3.0.1` devDep; 3.x resolves via `mime-db@^1.54` and its
  mime-score conflict resolution, so `mime.extension('image/jpeg')` returns
  `jpg` (JPEG exports are `.jpg`, was `.jpeg` under 2.x) and `lookup('*.wav')`
  is `audio/wav` (was `audio/wave`). `source-map@^0.8.0`
  (`src/partials/webgl.ts`) is the WASM line, but the `SourceNode` /
  `SourceMapGenerator` pair stays synchronous in Node; the
  `declare module 'source-map'` augmentation adding `toJSON(): any` is still
  required for the custom `includeMap` key on the emitted map (0.8's own
  signature returns `RawSourceMap`). `follow-redirects` is `^1.16.0` with
  `@types/follow-redirects@^1.14.4`; npm 11.19.0's workspace uninstall of that
  `@types` package also drops the runtime `follow-redirects` dep from the
  manifest, so re-add it after uninstalling the types.
- `ffmpeg`: dual `client/tsconfig.json` + `server/tsconfig.json` builds; license
  GPLv3 (others MIT). The server tsconfig uses `module`/`moduleResolution`
  `node16` (CommonJS emit preserved) and `server/FFmpegBridge.ts` imports
  `Connect` / `ViteDevServer` from `vite` through a `resolution-mode` type
  import, since Vite 8's declarations are ESM-only. The server's ffmpeg API
  types come from `@types/fluent-ffmpeg@^2.1.28` (devDep) — its typed `on()`
  overloads pass `(stdout, stderr)` to the `end` listener, so resolve with
  `() => resolve()`, never `resolve` directly. The bundled binaries come from
  `ffmpeg-ffprobe-static@^6.1.2-rc.1` (ffmpeg/ffprobe 6.1.2), which downloads
  them in an `install` script: keep the root `allowScripts` pin in step with the
  installed version (npm 11 skips unapproved scripts) and re-check it on every
  bump. 6.1.2-rc.1 dropped the old `postinstall: patch-package` hook, so the
  matching root `overrides` entry was removed too.
- `player`: Vite web-component consumer of built packages. Its devDep
  `sass@^1.104.1` shares the `ui` copy; the config's `legacy-js-api`
  `silenceDeprecations` block was removed with the Vite 8 upgrade (modern API)
  and `build.rollupOptions` became `build.rolldownOptions`. The package is
  `"type": "module"`, so `vite.config.ts` already loads as ESM.
- `internal`: private build helpers only — includes `vite/markdown-literals`
  plugin required by `core`/`2d` vitest configs. `common/marked.js` (shared by
  the `markdown-literals` TS transformer and Vite plugin) runs
  `marked@^18.0.13` + `highlight.js@^11.12.0`; marked is ESM-only since v16 and
  is loaded from this CommonJS file through Node's `require(esm)` support, and
  its renderer overrides must use the v13+ token objects (`link({href, tokens})`
  with `this.parser.parseInline(tokens)`, `code({text, lang})`) — the old
  argument-style signatures are never called.
- `docs`: private, not published. Docusaurus 3.10.2 site (React 19, MDX v3,
  `plugin-svgr`); the `typedoc.js` plugin regenerates `src/generated` during
  production builds. Site components use `clsx@^2.1.1` via the default import
  (the site was the last `^1.2.0` consumer before the repo-wide 2.x dedupe). The
  deprecated-`CodeBlock` doc page (`docs/components/code-block.mdx`) and the
  2.4.0/2.6.0 blog posts (which linked it) were removed in 4.0.0.
- `e2e` / `examples` / `template`: private, not published. The `code-block`
  example project was removed in 4.0.0; adding a new example requires both a
  `src/*.ts` project file (plus its `scenes/*` entry and `.meta`) and a line in
  the `project` list of `packages/examples/vite.config.mts`. Since the editor
  dev loop aliases `@motion-canvas/ui` / `@motion-canvas/2d/editor` to source,
  `packages/template/vite.config.mts` uses the same aliases and imports the
  built `@motion-canvas/vite-plugin` / `@motion-canvas/ffmpeg` packages (the
  only config that used to import their TS sources; importing them made every
  file in those CJS packages trip Vite 8's `configLoader: 'native'` warning).
  All three configs are `.mts`/ESM (the packages are CJS; the planned native
  config loader rejects ESM syntax in CJS-loaded files) and use
  `rolldownOptions`; calls on the CJS plugin packages go through `.default`
  (`motionCanvas.default(...)`, `ffmpeg.default()`) because an ESM-loaded config
  gets the namespace object. `e2e/vite.config.ts` takes `defineConfig` from
  `vitest/config` so its `test` block is typed.

## Verify (mirrors `verify.yml`)

- Lint: `npx eslint "**/*.{ts,tsx,mts}"` — quote the glob. Flat config lives in
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
  invoke bare `vitest`, which drops into watch mode. The `core` / `2d` configs
  are `vitest.config.mts`: the packages emit CommonJS but the configs use ESM
  syntax, and Vitest 5/Vite 8 warns for ESM syntax in a CJS-loaded config
  (`configLoader: 'native'`). The `e2e` config keeps its `test` block in
  `vite.config.ts`.
- Single test: `npx vitest run <path>` from `packages/core` or `packages/2d`
  (jsdom env; `core` uses `vitest.setup.ts`). jsdom is declared as a devDep in
  both `.package.json` files (currently `^30.0.1`) and is only loaded by the
  vitest dom-environment — never import it directly; vitest@5 consumes jsdom
  through its optional peer without conflicts.
- E2E: `npm run e2e:test -- run` (non-interactive; plain `npm run e2e:test`
  drops into vitest watch mode on a TTY) — Playwright **Firefox** headless +
  `jest-image-snapshot@^6.5.2`; spins up Vite server itself
  (`packages/e2e/vite.config.ts` sets `testTimeout`/`hookTimeout` to 60s, as the
  `beforeAll` hook launches the browser and the server). Failure diffs:
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

- Conventional Commits enforced by commitlint + Husky (root devDep `^9.1.7`).
  `prepare` must stay the bare `husky` command — `husky install` is deprecated
  since v9 and prints a warning on every install. Hooks: `.husky/commit-msg`
  (`npx --no -- commitlint --edit $1`) and `.husky/pre-commit`
  (`npx lint-staged`), wired via `core.hooksPath=.husky/_`; never add the legacy
  `#!/usr/bin/env sh` shebang or the `.husky.sh` source line to a hook file
  (husky 9.1.2+ warns they will fail in v10). `lint-staged` (root devDep
  `^17.5.1`; config in root `package.json`) needs Node ≥22.22.1 and Git ≥2.32.0,
  and runs tasks without a shell — no pipes / `bash -c` in the config, and Node
  scripts must be invoked as `node script.js` (`--shell` is gone since v16).
  Scope must be one of:
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
