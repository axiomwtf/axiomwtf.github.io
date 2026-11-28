# honeybun Specification
**Version:** 0.1.0
**Ownership:** honeybun is owned by AXIOM
---

## 1. Overview

honeybun is a single static binary that provides equivalent functionality to npm, npx, esbuild, vite, and vitest in one toolchain, plus TypeScript-to-JavaScript transpilation. It is Node-compatible, TypeScript-native, scriptable, and faster on the dev loop due to a persistent daemon architecture. It is not a tsc replacement: TypeScript is transpiled with the Oxc parser + transformer, no full type-checking is performed (use tsc for that).

**The output is always exactly one file: `honey` by default.** No JARs. No DLLs. No sidecars. No "requires runtime" asterisks. Every command compiles down to a single binary. If `honey` cannot produce one binary, it fails with an error. It does not produce partial output.

No Bun code. No Zig. Rust + upstream V8, newest, unpinned.

---

## 2. Naming

| Name | Role |
|------|------|
| `honey` | CLI binary, daemon, package manager |
| `hxney` | One-off package executor (npx/bunx equivalent), bundled npm (`hxney npm`), interactive JS shell (bare `hxney`) |
| `Cinnamon` | Internal engine: compiler, resolver, bundler, minifier |
| `.cinm` | Module directory (equivalent functionality to node_modules) |
| `node_modules` | Symlink to `.cinm` (interop) |

---

## 3. Architecture

One static binary. No runtime files. No `.so`. No sidecar.

`honey` (CLI, daemon, PM) → `Cinnamon` (engine) → `V8` (upstream, newest) (runtime)

### Components

| Component | Role |
|-----------|------|
| `honey` | CLI, daemon, package manager |
| `hxney` | One-off package executor (npx/bunx equivalent), bundled npm (`hxney npm`), interactive JS shell (bare `hxney`) |
| `Cinnamon` | Compiler, Resolver, Bundler, Minifier, `.cinm` layout |
| `V8 (upstream)` | Runtime (newest crates.io `deno_core` + its matched `v8`) |

---

## 4. Daemon

- Spawned by `honey start`, or auto-spawned by the CLI on first `honey run`
- Listens on `/run/user/$UID/honey.sock` (chmod 0600, removed on exit)
- Holds one warm V8 isolate across runs; IPC round-trip ~0.2ms on top of a ~90µs process floor
- Idle footprint ~20 MB RSS
- Crash recovery: the CLI falls back to in-process execution if the socket is dead. `honey stop` kills the daemon.
- Rebuilds: a running daemon keeps executing the code it was spawned with. After `cargo build`, run `honey stop` once so the next `honey run` auto-spawns the new binary.
- Env knobs: `HONEY_NO_DAEMON=1` (disable), `HONEY_SOCKET` (path override), `HONEY_TRACE=1` (trace, forces cold), `HONEY_DAEMON_LOG` (daemon trace file)

### Protocol

Length-prefixed JSON over the Unix socket:

```
Client -> Server: [4-byte LE length][JSON request]
Server -> Client: [4-byte LE length][JSON response]
```

Request shape:
```json
{ "cmd": "run", "file": "app.ts", "args": [], "cwd": "/home/user/project" }
```

Commands: `run`, `watch`, `test_batch`, `eval`, `build`, `build_watch`, `ping`, `stop`.

The `watch` command opens a long-lived session: the daemon runs the file, streams one response frame, then registers file watches (non-blocking fd; inotify on Linux, kqueue `EVFILT_VNODE` on macOS) on the main file plus every resolved dependency file and its directory from the last run's resolution records. Any event re-runs on the warm engine and streams a new frame; events that land mid-run are consumed by the post-run drain, so one editor save yields exactly one re-run. Detection is event-driven (30-60µs from edit to re-run start), no polling. The session ends when the client disconnects; `Ctrl+C` on the CLI kills the client and the daemon returns to its accept loop.

`test_batch` runs a whole suite sequentially in the warm engine (one Response per test file, engine init once per suite instead of one worker subprocess per file). `eval` evaluates a config-hook script (`honey.config.ts` hooks) as a classic script in the warm engine. `build` runs the bundler in the resident process, so one-shot builds never pay a second binary start. `build_watch` hosts the `honey build --watch` loop entirely in the daemon and streams one captured Response frame per rebuild (the once-only header lines ride in the first frame, so the client's output matches the cold watcher's); the session ends when the client disconnects — the daemon zero-timeout-polls the client socket each 10ms tick, so a killed or Ctrl-C'd client frees the daemon immediately.

Response shape:
```json
{ "exit_code": 0, "stdout": "...", "stderr": "...", "duration_ms": 0.4, "error": null }
```

### Fallback

If the daemon isn't running (crashed, not yet started), the CLI auto-spawns it in the background and waits up to 500ms for the socket; if it never appears, the CLI falls back to in-process mode. `honey` always works.

### Warm State

The daemon keeps one V8 isolate alive across successful runs (exit 0). A failed or crashed run drops the engine and its caches.

- The module cache is keyed by absolute resolved path; every cached specifier also keeps a resolution record (specifier + module dir -> resolved path + mtime snapshot). Warm runs re-served from cache are validated by one `stat` per record (~0.6µs each); a changed file drops both its cache entry and its record, and the run re-resolves and re-evaluates from disk.
- The main module always re-executes in a warm engine.
- ESM main modules are an exception: deno_core permits one registered main module per engine, so after an ESM main the engine is rebuilt from scratch on the next run (CJS mains keep full warm reuse).
- System/`node:` builtins are kept in the engine between runs unconditionally.

---

## 5. Cinnamon (Engine)

### Subsystems

| Subsystem | What it does |
|-----------|-------------|
| Compiler | TS to JS via Oxc parser + transformer. Type-stripping only — no type-checking (use tsc). |
| Resolver | `.cinm` traversal, exports/imports maps, ESM/CJS interop; standard `node_modules/<pkg>` fallback (exports/main/index) for npm-installed trees |
| Bundler | Graph to output. Based on Rolldown. |
| Minifier | oxc codegen minify + compile-time defines (`process.env.NODE_ENV`) with static-if/dead-branch elimination to a fixpoint; sourcemaps via oxc_sourcemap concat. Tree-shake tracked, not landed. |

### `.cinm` Layout

Optimized flat/hard-linked layout (no nested `node_modules`):

```
.cinm/
├── lodash@4.17.21/          # package name + version
│   ├── package.json
│   ├── index.js
│   └── ...
├── typescript@5.6.3/
├── react@19.2.8/
├── @types/                  # scoped packages
│   └── react@19.2.8/
└── ...
```

### Hard-Linking

When the same package version is installed in multiple projects, the files are hard-linked, not copied:

```
Project A/.cinm/lodash@4.17.21/index.js  -> inode 42
Project B/.cinm/lodash@4.17.21/index.js  -> inode 42 (same file)
```

- Disk usage is near-zero for shared dependencies
- Updates are isolated (editing one project's copy doesn't affect another)
- The filesystem handles deduplication transparently

### Interop

`node_modules` is a symlink (Unix) or junction (Windows) pointing to `.cinm`:

```
node_modules -> .cinm
```

This means `require('lodash')` works in any tool that looks for `node_modules/`, but the actual files live in the optimized layout. Conversely, a package installed by plain npm (standard `node_modules/<pkg>` with `package.json` `exports`/`main`, or an `index.*` entry) resolves without ever being installed through honey; when both stores exist, `.cinm` is searched first per ancestor directory and `node_modules` fills the gap. Extensionless imports from `.ts`/`.tsx` files fall back through `.ts`, `.tsx`, `.js`, `.mjs`, `.cjs` and directory-index candidates (bundler-mode rules). Transpiled `.tsx`/`.jsx` output is cached per file mtime *plus* the nearest tsconfig.json's mtime, so flipping `jsx`/`jsxImportSource` settings re-transpiles on the next run even mid-daemon-session.

`.gitignore`: `.cinm/` and `node_modules`

### JS to TS Migration

```bash
honey migrate              # whole project
honey migrate --file x.js  # single file
```

- Infers types from usage patterns
- Adds interface/type declarations for repeated shapes
- Flags inferred types with `// @honey: inferred`
- Incremental support (a `.ts` output newer than its `.js` source is skipped)
- The migrated `.js` source is removed after a successful `.ts` write; parse
  failures keep the original file and are reported per-file

---

## 6. V8 (upstream, newest)

honey runs on upstream V8 from crates.io — never a fork, never a vendored
checkout. The dependency is deliberately unpinned (`deno_core = "0"` in
Cargo.toml), so every build takes the newest `deno_core` release and the V8
binding it pairs with (exposed as `deno_core::v8`; honey never declares its own
`v8` crate dependency — the runtime layer owns that pairing). What honey does
on top of stock V8:

1. **Runtime `v8_flags`** (applied in `src/runtime.rs` `apply_v8_flags()` before
   the first isolate; effective in every shipped binary): Maglev tier threshold
   400→100 invocations, TurboFan tier threshold 3000→750 invocations, initial
   heap → 32MB, stack trace limit = 10 and lazy source positions = on (both
   already V8 defaults, kept explicit).
2. **Build wiring** (`.cargo/config.toml` + `build.rs`): `target-cpu=native` for
   all crates, non-PIE final binaries (`-no-pie`, `-z now`, `-z
   noseparate-code`), LTO + `codegen-units=1` + strip.
3. **Startup context snapshot** (implemented, see below).

Not done: no GN-arg patching (upstream prebuilt archives are used as-is),
no inspector strip (the inspector is present in the binary; it is dormant),
no source patches. Fork source builds (`V8_FROM_SOURCE`, RUSTY_V8_ARCHIVE
against a vendored V8) were tried and removed — the crates.io prebuilt is the
supported path; "newest upstream" is a hard policy.

### Startup Context Snapshot (implemented)

- **Build-time:** `honey --internal-mksnapshot` builds a pristine engine (no honey extension), runs a short warmup script (JSON, Date, closures, promises) to bake IC state into the context, and serializes it to `target/honey_snapshot.bin`. Regenerate the blob (and rebuild) whenever the runtime's context setup changes.
- **Embed:** `build.rs` detects the blob, emits `include_bytes!` glue, the next `cargo build` ships it in-binary (~0.6MB); tracked via `rerun-if-changed` so adding/removing the blob triggers a rebuild
- **Runtime:** `build_runtime()` passes `RuntimeOptions::startup_snapshot`; deno_core restores isolate + context from the blob and re-runs the honey extension's global middleware (require/console/process/Buffer/timers) on the restored global — middleware runs in both init paths upstream
- **Measured:** steady-state engine init 3.56ms -> 1.30ms (~2.7x), A/B inside one binary via `honey --internal-enginetiming` with HONEY_MKSNAPSHOT toggling the path
- **Constraint:** the blob captures the pristine context (isolate creation, builtins, deno_core internals); the extension middleware re-injects globals at engine creation, which is where the remaining ~1.3ms goes. A full globals-in-snapshot design needs the builtin surface moved to `FunctionTemplate`s + ops (honey's node-compat globals are plain `v8::Function` objects holding raw Rust callback addresses, which V8's snapshot serializer refuses to encode).

```rust
// Build step
cargo build -> build.rs detects target/honey_snapshot.bin -> include_bytes! glue
honey --internal-mksnapshot -> pristine JsRuntimeForSnapshot + warmup -> blob

// Runtime
let runtime = JsRuntime::new(RuntimeOptions {
    startup_snapshot: Some(HONEY_SNAPSHOT),  // only when the blob was built in
    ...
})
```

### Node Compatibility

- Full compat layer (`deno_node` equivalent, built on upstream deno_core)
- Target: 95%+ of npm packages at v0.1
- Native addons (`.node`): v0.2 (loading one fails with an explicit message today)
- No Bun code. Upstream deno_core + V8, newest, unpinned. Clean room.

Implemented today (verified against real npm packages — react/react-dom 19
server render, chalk 5 native-ESM chain, zod 4 (CJS + ESM entries), lodash 4,
semver 7.x classes (module cycles), dayjs, ms, picocolors):

- **Builtins**: `path`, `fs` (+`fs/promises`), `os`, `util` (incl. `inspect`,
  `format`, `promisify`, `styleText`, `TextEncoder`/`TextDecoder`),
  `url`, `module`, `events`, `assert` (deep-equal family with cycle guards,
  strict/loose variants, `throws` matching), `worker_threads`, `stream` stubs,
  `child_process`, `tty`, `crypto` (hash/random bytes/UUID), `buffer`,
  `querystring`, `readline`, `perf_hooks` (`performance.now`/`mark`),
  `async_hooks` (synchronous `AsyncLocalStorage`), and `process` (module form
  of the global process object).
- **ESM↔CJS interop**: `require()` of ESM-only packages via an ESM→CJS
  transform (multi-line imports/exports, `export {a as b}` lists, re-exports,
  CJS named-export detection à la cjs-module-lexer for `import {x} from cjs`).
  Explicit `.cjs` files are always CommonJS regardless of package `type`.
  Node's `require(esm)` default-unwrap semantics for `module.exports =`
  reassignments are preserved (interop annotations never decorate a
  reassigned `module.exports`).
- **Subpath `#` imports**: the package.json `imports` field (incl. conditional
  `node` targets) resolves `#foo` specifiers against the nearest package.json —
  chalk's `#ansi-styles` / `#supports-color` vendor-alias layout works.
- **Deno-shaped compat probes**: deno_core registers a `globalThis.Deno` stub
  (its ops object); packages like chalk/has-flag do
  `globalThis.Deno ? Deno.args : process.argv`, so honey mirrors
  `process.argv` into `Deno.args` after engine construction.
- **Scoped packages**: `honey add @scope/name` supports the nested
  `.cinm/@scope/name@x.y.z` layout (store entries stay flat).

---

## 7. Package Manager

- Default registry: npm
  ```bash
  honey install              # install this project's dependencies (honey.lock + .cinm)
  hxney npm <args>           # npm-style package management: hxney proxies a bundled npm
  hxney npm install -g <pkg> # global installs live under ~/.cache/honey/npm-global
  ```
  `hxney npm` bootstraps the npm package itself into the shared store
  (`~/.cache/honey/store/npm@<version>`) on first use and runs its CLI on
  the Node-compatible runner on PATH (node, falling back to bun). Global
  installs live under the hxney-owned prefix (`~/.cache/honey/npm-global`
  when no user prefix is set) and their bin shims are linked into
  `~/.local/bin` so installed commands are immediately runnable; uninstall
  prunes the links. `honey install <pkg>` / `-g` is rejected with a pointer
  to `hxney npm`.
- **Interactive shell**: bare `hxney` opens a JS REPL on the warm engine
  (value echo, persistent session state, multi-line input, `exit` /
  Ctrl-D to leave). Leading `const`/`let`/`class` declarations are
  rewritten to persist across lines (eval scoping).
- Lockfile: `honey.lock` (binary format, fast)
- Can read `package-lock.json` for migration
- Parallel downloads (tokio) — implemented: concurrent BFS waves over a `Semaphore(8)`, shared HTTP client, shared-store dedupe, atomic single-write lockfile
- Hard-linked cache (shared across projects)

### `.cinm` Hard-Link Layout

The `.cinm` directory provides equivalent functionality to `node_modules` with a flat, hard-linked structure:

```
.cinm/
├── lodash@4.17.21/          # package name + version
│   ├── package.json
│   ├── index.js
│   └── ...
├── typescript@5.6.3/
├── react@19.2.8/
├── @types/                  # scoped packages
│   └── react@19.2.8/
└── ...
```

Hard-linking — When the same package version is installed in multiple projects, the files are hard-linked, not copied:

```
Project A/.cinm/lodash@4.17.21/index.js  -> inode 42
Project B/.cinm/lodash@4.17.21/index.js  -> inode 42 (same file)
```

This means:
- Disk usage is near-zero for shared dependencies
- Updates are isolated (editing one project's copy doesn't affect another)
- The filesystem handles deduplication transparently

Interop — `node_modules` is a symlink (Unix) or junction (Windows) pointing to `.cinm`:

```
node_modules -> .cinm
```

This means `require('lodash')` works in any tool that looks for `node_modules/`, but the actual files live in the optimized layout.

Why this matters — pnpm's approach (real `node_modules` made of symlinks into a store) is more complex. This is simpler: one real directory, one symlink. The resolver walks `.cinm` directly, which is faster than traversing nested symlinks.

---

## 8. honey.config.ts

```typescript
export default defineConfig({
  onInstall(ctx) { ... }
  onResolve(ctx) { ... }
  transform(ctx) { ... }
  onBuild(ctx) { ... }
  onRun(ctx) { ... }
  onTest(ctx) { ... }
})
```

- Zero config out of the box
- Fully scriptable when you need it
- Runs in the daemon's warm V8 isolate (no extra engine)
- Script tasks: `honey run <task>` with no file of that name resolves the task from `scripts` in the config — a string is the shell command, a function of ctx `{args}` returns the command; run through `sh -c 'exec <command> "$@"'` with extra CLI args appended. An existing file always wins over a script task.

**v0.1 status (implemented):** `onInstall`, `onBuild`, `onRun`, `onTest` fire around the corresponding commands; `honey.config.ts` (or `.js`) is discovered in the cwd, ESM→CJS-transformed and evaluated, `defineConfig` unwrapped, and the hook called with a JSON ctx object. `onRun` ctx: `{file}`; `onInstall`: `{cmd, production|package, dev}`; `onBuild`: `{cmd}`; `onTest`: `{filter}`. Hooks are advisory-only in v0.1: they can log or mutate config-visible state, hook errors print a warning and never fail the command, `onResolve`/`transform` are accepted but ignored, and the config file cannot yet `require`/import other modules or npm packages.

---

## 9. Test Runner

```bash
honey test                 # discover + run all
honey test --filter "auth" # filter by name
honey test --slow          # include slow tests
honey test --watch         # watch mode
```

- Parallel execution (tokio)
- Snapshots
- Built-in assertions
- Runs on Cinnamon runtime (no separate test framework needed)

**v0.1 status (implemented):** discovery walks `tests/` (falling back to a project-wide walk) for `*.test.js/.ts/.jsx/.tsx` and `*.spec.*` (`.d.ts` excluded, `.slow.` marked); `--filter` is a path substring; parallel execution dispatches files to one-shot worker subprocesses (`honey --internal-test-worker`, stdin JSON protocol, semaphore capped at CPU count) with results printed in path order; built-in assertions via an injected `expect`/`describe`/`it` preload; snapshots persist to `<testfile>.snap` (`HONEY_SNAPSHOT_UPDATE=1` to re-record); `--watch` re-runs on test-file changes via the platform watcher (inotify on Linux, kqueue on macOS). Limitation: parallel first-pass runs pay fresh engine init per worker — the warm-isolate claim applies to the daemon hot path and sequential re-runs, not to the parallel first pass.

---

## 10. E2E Quality Gate

- Two suites: `e2e/js/` (oracle) and `e2e/ts/` (same tests, typed)
- Gate: TS pass rate must be >= 97.25% of JS pass rate
- Below threshold: CI blocks, prints failing test names
- Track gap over time; raise threshold as it shrinks

---

## 11. Benchmarks (claimed)

### Measurement conditions

Benchmarks are taken on the real target class of hardware — an 8GB RAM i5-13420H laptop — at ~75% RAM utilization, under everyday load: browser open with media streaming while scrolling. Not on an idle benchmark rig. All measurements use the standard `cargo build --release` binary with default profile flags (opt-level 3, LTO, strip) — no PGO, no custom tuning. A quiet dedicated machine only improves the ratios; a laptop under real-world memory and multitask pressure is where the dev loop actually lives, so that is where the ~5x nominal and up-to-8.4x peak are claimed from.

| Operation | vs Bun |
|-----------|--------|
| Dev loop (repeated invocations, warm daemon) | 7.1x single-run (warm `honey run hello.js` 0.90ms vs `bun hello.js` 6.41ms), 6.9x node_modules resolution (`require('ms')`: 1.23 vs 8.46ms), 6.3x batched (50-module batch: 1.39 vs 8.79ms), 7.3x on TS (0.90 vs 6.56ms), 3.3x process floor (0.71 vs 2.35ms) — 2026-11-22 battery medians (05:51Z) |
| Test/batch runs (vitest-style suites, one-shot runner) | 2.6x faster (50 files / 150 assertions: honey 7.83 vs Bun 20.05ms median) |
| Per-file spawns (fresh process per file, warm daemon) | 8.4x faster (50 files: honey 52.2ms vs Bun 437.0ms) — the peak ratio in the battery |
| Watch mode (edit-refresh cycle) | 1.7x faster (edit→output vs `bun --watch`: honey ~8.0ms vs Bun ~13.4ms warm over 5 refreshes; re-run itself ~1ms on the warm engine, event-driven file watching — inotify on Linux, kqueue on macOS — no polling) |
| Node startup-bound workloads | up to 31x faster (warm JS run: honey 0.90ms vs Node 28.20ms; process floor honey 0.71ms vs Node 6.82ms) |
| Transpile (TS to JS) | 2.7x faster (1.3MB TS: 18.18 vs 49.03ms) |
| Bundle | 2.0x faster dev (3.70 vs 7.56ms, 100 source modules + CSS; `honey build <entry> [-o dir]`: inlines relative imports and — since the standard `node_modules` fallback — bare npm packages too, resolved via package.json `exports`/`main` from `.cinm` or `node_modules`; CSS imports are extracted to a sibling `.css` (minified in production); a `process` guard is emitted for browser bundles; `node:` builtins and URL specs stay external; tree-shaking: unused-export pruning across the module graph, dead modules and factories dropped) |
| Build watch (3 consecutive source touches) | 1.2x faster (honey 11.73 vs Bun 14.16ms) |
| Install | ~1:1 (network-bound, small packages: 351.1 vs 303.0ms) |
| Type-checking | Not supported (use tsc) |
| Minify | 2.4x faster (3.36 vs 8.16ms, 100-module bundle; oxc codegen minify + `process.env.NODE_ENV` define with static-if/dead-branch elimination to a fixpoint; `-p` prod build 3.15ms). Tree-shaking (used-export analysis: unused export wiring pruned, dead pure modules dropped from the bundle) is landed. |
| Sourcemaps | 2.8x faster external (2.94 vs 8.29ms, 100 modules; per-module oxc maps concatenated at bundle line offsets) — `--sourcemap linked` (default unminified), `external` (default with `--minify`/`-p`), `inline`, or `none`; tokens map back to original TS/JS lines/columns |

Reference points behind the dev-loop row: a two-line `console.log` script runs in ~113µs median end-to-end through the warm daemon (Bun: ~590µs on the same machine, same session); a `require('ms')`-style node_modules-resolution script runs in ~114µs (Bun: ~623µs); a warm `require()` to an already-cached module costs 2-3µs. Behind the batch row: 50 vitest-style test files where Bun pays its per-file runtime startup 50 times and honey's warm daemon pays it once; warm staleness re-validation is one mtime check per cached file (~0.6µs each).

---

## 12. Dependencies (compile-time, linked statically)

```
clap, tokio, reqwest (rustls-tls), serde, serde_json, flate2, tar, anyhow, futures,
deno_core "0" (newest, unpinned — every build takes the latest crates.io release and
its matched v8; currently 0.413 / v8 150.4.0),
oxc_parser/transformer/semantic/codegen/allocator/ast/span (major-matched set, 0.40),
oxc_sourcemap, libc
```

---

## 13. Build

```bash
cargo build --release
```

Only the base system libraries remain dynamic (libgcc, libm, libc plus the loader); everything else is statically linked. No external files, no `node_modules`, no `.so` shims. One file.

Binaries link non-PIE automatically via `build.rs` (`rustc-link-arg-bins=-no-pie`, Linux only) — plain `cargo build --release` is enough; a non-PIE exec saves ~0.4ms of startup on every invocation.

Optional, for the fastest possible engine init (§6): after the first build, run `honey --internal-mksnapshot` once, then rebuild — the blob is picked up automatically and embedded from then on. Regenerate the blob (and rebuild) whenever the runtime's context setup changes.

---

## 14. Notes

- No Bun code. No Zig. Upstream deno_core + V8, newest, unpinned. Clean room.
- The compiler is written in Rust (edition 2024) and lives in `src/`
- The daemon is the single biggest architectural decision in the whole project
- Every `honey run` in the dev loop is an IPC round-trip (~0.2ms on top of a ~90µs process floor) instead of a cold runtime start. That's the up-to-8.4x claim on the dev loop, with the peak on warm-daemon per-file test spawns.
- Axiom philosophy ("no sidecars: companion files and helper processes count as bugs"): the daemon is the one permitted helper process, justified by the up-to-8.4x dev-loop claim. Its socket lives in `$XDG_RUNTIME_DIR` (user tmpfs), is chmod 0600, and is removed on daemon exit. Nothing transient is ever written into a user project: only project data (`.cinm/`, `honey.lock`), like Cargo.lock. Transient helper files (hxney's exec wrapper) go to the system temp dir.
