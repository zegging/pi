# Pi Mono — Quickstart

Pi Mono is the monorepo for the **Pi agent harness** — a minimal, extensible coding agent that runs in a terminal. It includes an interactive CLI (`pi`), a multi-provider LLM abstraction layer, an agent runtime, a terminal UI framework, and an orchestrator for managing long-running agent processes.

## What's in this repo

| Package | npm name | Purpose |
|---------|----------|---------|
| `packages/ai` | `@earendil-works/pi-ai` | Unified multi-provider LLM API (OpenAI, Anthropic, Google, Bedrock, 30+ providers) |
| `packages/agent` | `@earendil-works/pi-agent-core` | Agent runtime with tool calling, state management, session persistence, compaction |
| `packages/coding-agent` | `@earendil-works/pi-coding-agent` | Interactive coding agent CLI — the `pi` binary |
| `packages/tui` | `@earendil-works/pi-tui` | Terminal UI framework with differential rendering |
| `packages/orchestrator` | `@earendil-works/pi-orchestrator` | Process supervisor for long-running agent instances (experimental) |

## Quick links

- [Architecture overview](architecture.md) — monorepo structure, build system, dependency graph
- [packages/ai](packages/ai.md) — LLM API abstraction, providers, auth, model discovery
- [packages/agent](packages/agent.md) — Agent loop, sessions, compaction, harness
- [packages/coding-agent](packages/coding-agent.md) — CLI, tools, extensions, configuration
- [packages/tui](packages/tui.md) — Terminal UI framework, components, rendering
- [packages/orchestrator](packages/orchestrator.md) — Process supervision, IPC, Radius
- [Workflows](workflows.md) — build, check, release, supply-chain hardening
- [Testing](testing.md) — test infrastructure, conventions, running tests

## Development setup

```bash
# Install dependencies (no lifecycle scripts)
npm install --ignore-scripts

# Build all packages
npm run build

# Run checks (lint, format, type-check, shrinkwrap, pinned deps)
npm run check

# Run tests (skips LLM-dependent tests without API keys)
./test.sh
```

Requires Node.js >= 22.19.0.

## Build order

The build script (`npm run build`) builds packages in dependency order:

1. `packages/tui` (no internal dependencies)
2. `packages/ai` (no internal dependencies)
3. `packages/agent` (depends on `ai`)
4. `packages/coding-agent` (depends on `ai`, `agent`, `tui`)
5. `packages/orchestrator` (depends on `coding-agent`)

## Key conventions

- **TypeScript** with erasable syntax only (Node strip-only mode, no `enum`, `namespace`, parameter properties)
- **Biome** for linting and formatting (tab indent, 120 char width)
- **Exact version pinning** for external dependencies; ranged versions for internal workspace packages
- **No inline imports** — top-level imports only
- **No `any`** unless absolutely necessary
- **Commit format**: `{feat,fix,docs}[(ai,tui,agent,coding-agent)]: <message>`

## Existing documentation

The `packages/coding-agent/docs/` directory contains extensive end-user documentation for the `pi` CLI (quickstart, usage, providers, extensions, sessions, etc.). See the [coding-agent package page](packages/coding-agent.md) for a guide to those docs.

## Project website

- [pi.dev](https://pi.dev) — project website with demos
- [pi.dev/docs/latest](https://pi.dev/docs/latest) — hosted documentation
- [Discord](https://discord.com/invite/3cU7Bz4UPx) — community