# Change Log

All notable changes to this project will be documented in this file.
See [Conventional Commits](https://conventionalcommits.org) for commit guidelines.

# 4.0.0 (2026-09-15)

### Features

* **ffmpeg:** skip encoding when extracting frames ([#1095](https://github.com/motion-canvas/motion-canvas/issues/1095)) ([8b0d274](https://github.com/motion-canvas/motion-canvas/commit/8b0d27499c664272271ccc3fd45cbd4561768ea7))
* programmable sounds ([#1082](https://github.com/motion-canvas/motion-canvas/issues/1082)) ([a40d50b](https://github.com/motion-canvas/motion-canvas/commit/a40d50b87f5a2940e81d2fd0af4acb6d19202260))
* sound waveforms ([#1158](https://github.com/motion-canvas/motion-canvas/issues/1158)) ([158125a](https://github.com/motion-canvas/motion-canvas/commit/158125afa5642e342cd8c16173a698f13fe24651))


## Unreleased

### Build System

* upgrade `@types/fluent-ffmpeg` from `^2.1.21` to `^2.1.28`

  The typings for `server/FFmpegExporterServer.ts` move to the
  registry `latest`. 2.1.25 added typed `on()` event overloads
  (`start`, `progress`, `stderr`, `codecData`, `error`, `filenames`,
  `end`), 2.1.26 the `filenames` event, 2.1.27 a `codecData` fix, and
  2.1.28 `esModuleInterop`-compatible imports. The typed `end`
  listener signature required one adjustment: `this.command.on('end',
  resolve)` became `this.command.on('end', () => resolve())`, since
  the listener now receives `(stdout, stderr)` and `resolve` only
  accepts `void | PromiseLike<void>`. The runtime
  `fluent-ffmpeg@^2.1.3` is already latest and unchanged. Verified
  with the client + server `tsc` builds, the lerna build, the unit
  suites, eslint and prettier.

## [3.17.2](https://github.com/motion-canvas/motion-canvas/compare/v3.17.1...v3.17.2) (2024-12-14)

**Note:** Version bump only for package @motion-canvas/ffmpeg





# [3.17.0](https://github.com/motion-canvas/motion-canvas/compare/v3.16.0...v3.17.0) (2024-08-13)

**Note:** Version bump only for package @motion-canvas/ffmpeg





# Change Log

All notable changes to this project will be documented in this file. See
[Conventional Commits](https://conventionalcommits.org) for commit guidelines.

# 1.1.0 (2023-05-10)

### Features

- initial commit
  ([7a01fd5](https://github.com/motion-canvas/exporters/commit/7a01fd5614f2d62b4bd6e24c1096706f5dbf218b))
