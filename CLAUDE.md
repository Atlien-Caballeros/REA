# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

@AGENTS.md

The repository invariants above are authoritative; this file adds orientation only.

## What REA is

REA ("Reverse Engineer Anything", npm package `rea-agents`) is a TypeScript/Node ESM tool exposing one set of reverse-engineering workflows through two adapters: a one-shot CLI and an MCP stdio server. `scripts/rea.mjs` is the executable dispatcher and routes `rea mcp` to the server and everything else to the CLI. Deep analysis is delegated to bring-your-own engines (Hopper, Ghidra, IDA) plus auxiliary providers (JADX/Android, .NET metadata, firmware via Binwalk/Unblob, browser/CDP, V8 Inspector, EVM, WASM, pwntools/pwndbg).

## Commands

Use the Node version from `.nvmrc` and npm from `package.json#packageManager`.

```bash
npm ci                                         # install locked deps + Husky hooks (no build)
npm run build:cached                           # compile runtime + packaged skill (Turbo-cached); `npm run build` forces rerun
npm run check:fast                             # cached typecheck + lint (also the pre-push hook)
npm run check                                  # typecheck, lint, format:check, knip, metadata:check, architecture guards
npm run test:local -- src/config.test.ts       # source tests, no build
npm run test:focused -- tests/acceptance/applications/runtime.test.ts   # exact files; builds first if a compiled boundary needs it
npm run test:changed -- --dry-run              # preview tests affected since merge base with origin/main
npm run docs:check                             # validate generated docs/metadata
npm run format                                 # oxfmt
```

- Lint is `oxlint` plus `verify:module-boundaries` (resolved import boundaries). Run `npm run verify:test-discovery` after adding or moving tests.
- Vitest projects (see `vitest.config.ts`): `domain`, `services`, `adapters`, `composition`, `conformance`, `evaluation` (source-only, run by `test:fast`), and `boundary`, `cli-boundary`, `process-boundary`, `mcp-boundary`, `process-global`, `acceptance` (compiled runtime). Target one with `npx vitest run --project <name>`.
- Changed-test selection follows the import graph and can miss runtime registration, bridges, shell entrypoints and generated data. Select relevant boundary tests explicitly. An empty selection does not mean the change is verified.
- After a package, lockfile or managed-skill version change, run `npm run metadata:generate` before building. Packaging or setup changes also need `npm run verify:package` and `npm pack --dry-run`.
- Real-provider lanes such as `npm run verify:hopper` and `verify:hopper:linux` are separate from the deterministic suite. See `docs/testing.md` for prerequisites.
- Build and generation commands hold SQLite locks in `.cache/rea-command-locks/`. Never delete them while commands are running.
- Tests must use `createTestTempDirectory` from `tests/fixtures/temporaryDirectory.ts`. Never glob-clean shared `/tmp/rea-*`. CLI module mocks belong in `tests/process-global/`.
- Logging goes to stderr only, controlled by `REA_LOG_LEVEL`. Stdout carries only MCP protocol messages or command results.

## Architecture

`docs/architecture.mermaid` is the canonical map; update it when you change composition. Layers, outermost to innermost:

- **Entry adapters**: `src/cli.ts` + `src/cli/` (CLI) and `src/main.ts` + `src/main/` (MCP server lifetime, config, session). `src/server/` translates MCP requests and registers tools.
- **Composition** (`src/composition/`): production factories that build fresh, typed providers and session factories. They do not start engines or acquire targets. If an optional adapter fails, its successful peers must still load.
- **Application** (`src/application/`): workflows shared by the CLI and MCP adapters.
  - `SessionProviderRouter`/`BinarySession` give each target one immutable deep-provider binding, with disjoint routing for auxiliary providers.
  - `AnalysisProviderRegistry` selects deterministically from sorted candidates and never falls back.
  - `src/application/investigation/` owns `InvestigationRecords`, the `EvidenceLedger` and Unknowns, exposed through narrow ports.
  - `BinarySessionRecords` and `AnalysisSnapshotCache` hold target and profile snapshots.
- **Providers**: one directory per engine or format (`src/hopper/`, `src/ghidra/`, `src/ida/`, `src/android/`, `src/dotnet/`, `src/firmware/`, `src/browser/`, `src/inspector/`, `src/evm/`, `src/wasm/`, `src/native/`, `src/artifacts/`, `src/javascript/`, `src/reference/`).
  - In-engine code lives in `bridge/`, e.g. `bridge/hopper_bridge.py` runs inside Hopper and `bridge/ghidra/*.java` are packaged headless scripts.
  - The IDA adapter wraps an external `ida-pro-mcp`.
- **Process foundation**: `src/process/` owns process groups, private roots, deadlines, bounded reads and cleanup. Reuse it for any child process. `src/windows/` and the Node-API code in `native/windows/src/` hold the Windows-specific host pieces.
- **Contracts** (`src/contracts/`): caller-visible Zod schemas, including shared error schemas in `errorSchemas.ts`.
  - The generated error catalog is `docs/error-contract.schema.json`, refreshed with `npm run docs:error-schema`.
  - MCP tool changes must follow `docs/tool-design.md` and `docs/mcp-contracts.md`.
- **Domain** (`src/domain/`): pure, side-effect-free modules for evidence, comparisons and application graphs. All other layers depend inward on it.
- **Config** (`src/config/`): Zod-validated parsing of environment variables.
- **Evaluation** (`src/evaluation/`): agent release evaluation and transcript scoring.

Source unit tests sit next to their code (`src/**/*.test.ts`). Boundary, acceptance and fixture tests, including the fake launcher, bridge and CDP seams, live under `tests/`.

## Generated vs. authored files

- Edit skill instructions in `.agents/skills/reverse-engineer-anything/`. Never edit the ignored `skills/` output.
- `src/generatedPackageMetadata.ts` and `docs/error-contract.schema.json` are tracked but generated. Regenerate them rather than hand-editing; freshness checks enforce this.
- The release flow uses Release Please and Conventional Commits; see `docs/releasing.md`.
