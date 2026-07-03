# Architecture

This page describes the monorepo structure, build system, dependency graph, and cross-cutting concerns of the pi-mono repository.

## Monorepo structure

Pi-mono is an npm workspaces monorepo. The root `package.json` declares workspaces:

```
packages/*
packages/coding-agent/examples/extensions/with-deps
packages/coding-agent/examples/extensions/custom-provider-anthropic
packages/coding-agent/examples/extensions/custom-provider-gitlab-duo
packages/coding-agent/examples/extensions/sandbox
packages/coding-agent/examples/extensions/gondolin
```

The five core packages are:

```
packages/
  ai/           @earendil-works/pi-ai
  agent/        @earendil-works/pi-agent-core
  coding-agent/  @earendil-works/pi-coding-agent
  tui/          @earendil-works/pi-tui
  orchestrator/  @earendil-works/pi-orchestrator
```

Extension examples live under `packages/coding-agent/examples/extensions/`.

### Package dependency graph

```
tui  (standalone — terminal UI framework)
  ^
  |
ai   (standalone — LLM API abstraction)
  ^
  |
agent  (depends on ai)
  ^
  |
coding-agent  (depends on ai, agent, tui)
  ^
  |
orchestrator  (depends on coding-agent)
```

The orchestrator depends on `coding-agent` for RPC types and spawns it as a child process. All other dependencies are at the code level.

## TypeScript configuration

- **`tsconfig.base.json`** — shared compiler options for all packages
  - Target: ES2022
  - Module: Node16
  - Strict mode, erasable syntax only, declaration maps, source maps
  - `rewriteRelativeImportExtensions: true` — auto-rewrites `.ts` extensions in imports
- **`tsconfig.json`** (root) — references individual package tsconfigs and the root scripts directory

Key constraint: only erasable TypeScript syntax is allowed (no `enum`, `namespace`, parameter properties, `import =`, `export =`). This is enforced by the `erasableSyntaxOnly` compiler option.

## Linting and formatting

Biome (`biome.json`) handles both linting and formatting:

- **Formatter**: tab indent, 3-space width, 120 char line width
- **Linter**: recommended rules, with some relaxations (`noNonNullAssertion: off`, `noExplicitAny: off`)
- **Excluded files**: `models.generated.ts`, `*.models.ts`, `test-sessions.ts`

Run with `npm run check`, which calls `biome check --write --error-on-warnings`.

## Build system

The build script (`npm run build`) builds each package in dependency order using their individual `npm run build` scripts. Each package compiles TypeScript with `tsc` (or uses esbuild for some packages). The root build runs:

1. `packages/tui` → `npm run build`
2. `packages/ai` → `npm run build`
3. `packages/agent` → `npm run build`
4. `packages/coding-agent` → `npm run build`
5. `packages/orchestrator` → `npm run build`

## CI checks (`npm run check`)

The full check pipeline runs:

1. **Biome** — lint and format all TypeScript files
2. **`check:pinned-deps`** — verify direct external dependencies are pinned to exact versions
3. **`check:ts-imports`** — verify TypeScript imports are compatible (no imports from non-exported paths)
4. **`check:shrinkwrap`** — verify `packages/coding-agent/npm-shrinkwrap.json` is up to date
5. **`check:install-lock:coding-agent`** — verify the coding-agent install lock is in sync
6. **`tsgo --noEmit`** — type-check the entire project (uses `@typescript/native-preview`)
7. **`check:browser-smoke`** — verify no browser-incompatible APIs are used in the coding-agent package

## Supply-chain hardening

- Direct external dependencies are **pinned to exact versions** in `package.json`
- `.npmrc` sets `save-exact=true` and `min-release-age=2`
- `package-lock.json` is the dependency ground truth
- Pre-commit hook (via husky) blocks accidental lockfile commits unless `PI_ALLOW_LOCKFILE_CHANGE=1`
- `packages/coding-agent/npm-shrinkwrap.json` is generated from the root lockfile to pin transitive deps for npm users
- Shrinkwrap generation has an explicit allowlist for dependency lifecycle scripts
- CI installs with `npm ci --ignore-scripts`; a scheduled workflow runs `npm audit`

## Package entry points

Each package has a defined public API surface:

| Package | Entry point | Key source |
|---------|------------|------------|
| `ai` | `src/index.ts` | Core types, model collection, auth — no provider factories or generated catalogs |
| `agent` | `src/index.ts` | Agent class, loop functions, harness, session, compaction |
| `agent/node` | `src/node.ts` | Same as `index.ts` + `NodeExecutionEnv` |
| `coding-agent` | `src/index.ts` | AgentSession, config, compaction, extensions, types |
| `tui` | `src/index.ts` | TUI class, Component interface, all components |
| `orchestrator` | `src/index.ts` | Minimal public re-exports |

## Key root-level files

| File | Purpose |
|------|---------|
| `package.json` | Workspace config, scripts, devDependencies |
| `tsconfig.base.json` | Shared TypeScript compiler options |
| `tsconfig.json` | Root project references |
| `biome.json` | Biome linter/formatter config |
| `AGENTS.md` | Development rules for both humans and coding agents |
| `CONTRIBUTING.md` | Contribution guidelines, contributor gate |
| `SECURITY.md` | Security policy and vulnerability reporting |
| `.npmrc` | `save-exact=true`, `min-release-age=2` |
| `.husky/` | Pre-commit hooks |
| `scripts/` | Build, release, check, and profiling scripts |