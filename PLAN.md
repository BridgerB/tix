# tix: TypeScript Package Builder - Implementation Plan

## Context

Build "tix" — a simplified Nix replacement written in TypeScript. Uses a declarative API (Nix-style `mkDerivation`) to fetch, build, and install packages into a content-addressed store. First milestone: successfully build GNU Hello.

## Runtime: Node (native TS)

Node runs `.ts` files directly (native type stripping, no flags needed). Uses Node built-ins: `node:crypto` for hashing, `node:child_process` for spawning builds, `node:fs/promises` for I/O, global `fetch()` for downloads. Only dev dep: `typescript` (for type checking).

## Project Structure

```
tix/
├── package.json
├── tsconfig.json
├── src/
│   ├── index.ts          # Public API re-exports
│   ├── cli.ts            # CLI: `tix build <file.ts>`
│   ├── types.ts          # All core types
│   ├── derivation.ts     # mkDerivation()
│   ├── fetcher.ts        # fetchurl() + download/verify
│   ├── store.ts          # Store path management (~/.tix/store/)
│   ├── hash.ts           # SRI parsing, file verify, input hashing
│   ├── builder.ts        # Phase execution engine
│   ├── phases.ts         # Default phase scripts + override logic
│   ├── mirrors.ts        # mirror:// URL resolution
│   └── log.ts            # Colored terminal output
└── packages/
    └── hello.ts          # GNU Hello package definition
```

## Core Design

### Declarative API (user-facing)

```typescript
import { mkDerivation, fetchurl } from "tix";

export default mkDerivation({
  pname: "hello",
  version: "2.12.2",
  src: fetchurl({
    url: "mirror://gnu/hello/hello-2.12.2.tar.gz",
    hash: "sha256-WpqZbcKSzCTc9BHO6H6S9qrluNE72caBm0x6nc4IGKs=",
  }),
  doCheck: true,
  meta: { description: "...", license: "gpl3Plus", mainProgram: "hello" },
});
```

### Build flow

1. `tix build packages/hello.ts` — CLI dynamically imports the file
2. `fetchurl()` returns a lazy `FetchSpec` (no download yet)
3. `mkDerivation()` resolves phases, computes input hash, returns a `Derivation` object
4. CLI checks if store path exists → skip if already built
5. `realizeFetch()` downloads source via `fetch()`, verifies SRI hash with `node:crypto`, caches in `~/.tix/cache/`
6. Builder creates temp dir, runs phases in order via `child_process.spawn("bash", ["-c", script])`
7. After unpack, detects extracted source dir and uses it as cwd for remaining phases
8. On success: writes `.tix-info.json` to store path. On failure: cleans up output dir.

### Standard phases (each a bash script via `child_process.spawn`)

| Phase | Default | Override |
|-------|---------|----------|
| unpack | `tar xf "$src"` | `unpackPhase: string \| null` |
| patch | skip unless `patches` provided | `patchPhase: string \| null` |
| configure | `./configure --prefix="$out"` | `configurePhase: string \| null` or `dontConfigure: true` |
| build | `make -j$(nproc)` | `buildPhase: string \| null` or `dontBuild: true` |
| check | skip unless `doCheck: true` | `checkPhase: string \| null` |
| install | `make install` | `installPhase: string \| null` |

Each phase runs with `set -euo pipefail`. Environment includes `$out`, `$src`, `$pname`, `$version`.

### Content-addressing

Input hash = SHA-256 of: pname + version + source hash + all resolved phase scripts + flags + env overrides. Store path = `~/.tix/store/<hash32>-<name>-<version>/`. If inputs unchanged, build is skipped.

### Store layout

```
~/.tix/
├── store/     # Built packages: <hash>-<name>/bin/, lib/, etc.
├── cache/     # Downloaded source tarballs
└── tmp/       # Temporary build directories (cleaned up)
```

Configurable via `TIX_STORE_DIR` env var.

## Implementation Order

### Step 1: Project skeleton
- `package.json` (name, bin, scripts, `typescript` dev dep)
- `tsconfig.json` (ESNext, strict, NodeNext module resolution)
- `npm install`

### Step 2: Types + utilities
- `src/types.ts` — `SRIHash`, `FetchSpec`, `DerivationArgs`, `Derivation`, `ResolvedPhases`, `StoreConfig`, `BuildResult`, `Meta`
- `src/hash.ts` — `parseSRI()`, `verifySRI()` (using `node:crypto.createHash`), `computeInputHash()`, `computeStorePath()`
- `src/log.ts` — colored `info`, `phase`, `skip`, `success`, `warn`, `error`
- `src/mirrors.ts` — `resolveMirrors()` with GNU mirror set

### Step 3: Store + fetcher
- `src/store.ts` — `getStoreConfig()`, `initStore()`, `storePathExists()`, `registerStorePath()`
- `src/fetcher.ts` — `fetchurl()` (lazy), `realizeFetch()` (download via `fetch()` + verify + cache)

### Step 4: Build engine
- `src/phases.ts` — `resolvePhases()` with defaults for each phase, flag/override handling
- `src/derivation.ts` — `mkDerivation()` ties phases + hash together
- `src/builder.ts` — `build()` orchestrates fetch → unpack → detect source dir → run phases → register (using `node:child_process.spawn`)

### Step 5: CLI + first package
- `src/cli.ts` — `tix build <file>` command, dynamic import, store init, build invocation
- `src/index.ts` — re-exports `mkDerivation`, `fetchurl`, types
- `packages/hello.ts` — GNU Hello definition

## Verification

1. `node src/cli.ts build packages/hello.ts` completes without error
2. `~/.tix/store/<hash>-hello-2.12.2/bin/hello` exists and prints "Hello, world!"
3. Running the build again prints "Already built" and exits immediately (cache hit)
