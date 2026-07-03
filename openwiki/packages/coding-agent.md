# packages/coding-agent — `@earendil-works/pi-coding-agent`

Interactive coding agent CLI. The primary user-facing package. Exposes the `pi` binary and provides the full agent experience: tools, sessions, extensions, themes, keybindings, and more.

## Modes

The coding agent runs in three modes:

| Mode | Entry point | Description |
|------|------------|-------------|
| **Interactive (TUI)** | `src/modes/interactive/` | Full terminal UI with differential rendering, slash commands, inline editing |
| **Print** | `src/modes/print-mode.ts` | Non-interactive, single-prompt execution with JSON event output |
| **RPC** | `src/modes/rpc/` | JSON-RPC over stdin/stdout for programmatic control |

The `AgentSession` class (`src/core/agent-session.ts`, ~106KB) is the shared core across all modes. It encapsulates agent state, event subscription, model management, compaction, bash execution, and session switching/branching.

## Tools

Built-in tools are defined in `src/core/tools/`:

| Tool | File | Description |
|------|------|-------------|
| `read` | `read.ts` | Read files with optional offset/limit |
| `write` | `write.ts` | Create or overwrite files |
| `edit` | `edit.ts` | Patch files with old/new string replacement |
| `edit-diff` | `edit-diff.ts` | Diff-based file editing |
| `bash` | `bash.ts` | Run shell commands |
| `grep` | `grep.ts` | Search files with regex |
| `find` | `find.ts` | Find files by glob pattern |
| `ls` | `ls.ts` | List directory contents |

Additional read-only tools (`grep`, `find`, `ls`) are available through tool options. Tools use TypeBox schemas for argument validation. The `tool-definition-wrapper.ts` wraps tool definitions for consistent behavior.

## Extension system

The extension system (`src/core/extensions/`) allows third-party TypeScript modules to extend pi:

- **Tools** — add custom tools that the agent can call
- **Commands** — add slash commands (`/mycommand`)
- **Hooks** — hook into lifecycle events (before agent start, before provider request, after tool calls)
- **UI** — add custom UI components, autocomplete providers, entry renderers
- **Keybindings** — add custom keybindings

Extensions are loaded from:
- `~/.pi/agent/extensions/` (global)
- `<project>/.pi/extensions/` (project-local)
- Pi packages that bundle extensions

Key source files:
- `types.ts` (57KB) — all extension API types
- `loader.ts` — extension discovery and loading
- `runner.ts` — extension execution and lifecycle
- `index.ts` — extension system entry point

## Session management

The `SessionManager` (`src/core/session-manager.ts`, ~49KB) handles:
- Session listing, creation, switching, and deletion
- Branching and tree navigation
- Session metadata (name, model, provider, timestamps)
- Session export (HTML, JSONL)

Sessions are persisted as JSONL files in `~/.pi/agent/sessions/`. The format is documented in `packages/coding-agent/docs/session-format.md`.

## Configuration

Configuration is managed by `src/config.ts` (~19KB) and `src/core/settings-manager.ts` (~38KB):

- **Global settings** — `~/.pi/agent/settings.json`
- **Project settings** — `<project>/.pi/settings.json`
- **Auth** — `~/.pi/agent/auth.json` (managed by `src/core/auth-storage.ts`)
- **Keybindings** — `~/.pi/agent/keybindings.json`

## Package manager

The `PackageManager` (`src/core/package-manager.ts`, ~80KB) and `package-manager-cli.ts` (~24KB) handle pi packages — bundles of extensions, skills, prompts, themes, and keybindings distributed via npm.

## Key subsystems

| Module | File | Purpose |
|--------|------|---------|
| Model registry | `src/core/model-registry.ts` (34KB) | Model discovery, listing, selection |
| Model resolver | `src/core/model-resolver.ts` (24KB) | Resolves model aliases, defaults, provider routing |
| Resource loader | `src/core/resource-loader.ts` (37KB) | Loads AGENTS.md, CLAUDE.md, .pi/ files, skills |
| Prompt templates | `src/core/prompt-templates.ts` | `/template` slash command expansion |
| System prompt | `src/core/system-prompt.ts` | Generates system prompt from settings, skills, context |
| Skills | `src/core/skills.ts` (14KB) | Agent Skills — reusable on-demand capabilities |
| Compaction | `src/core/compaction/` | Session compaction (context window management) |
| Bash executor | `src/core/bash-executor.ts` | Shell command execution with timeout/PTY support |
| Trust manager | `src/core/trust-manager.ts` | Project trust on first run |
| Event bus | `src/core/event-bus.ts` | Internal event system |
| Telemetry | `src/core/telemetry.ts` | Usage telemetry |
| HTTP dispatcher | `src/core/http-dispatcher.ts` | HTTP request handling for tools |
| Auth storage | `src/core/auth-storage.ts` | Credential persistence |
| Migrations | `src/migrations.ts` | Data migration between versions |

## Package structure

```
packages/coding-agent/
  src/
    cli.ts               CLI entry point
    main.ts              Main orchestration (~30KB)
    index.ts             Public API surface
    config.ts            Configuration paths and constants
    rpc-entry.ts         RPC mode entry point
    package-manager-cli.ts  Package management CLI
    migrations.ts        Data migration logic

    cli/                 CLI argument parsing, startup UI, config selection
    core/                Core system
      agent-session.ts    AgentSession class (~106KB)
      agent-session-runtime.ts  Runtime for agent sessions
      agent-session-services.ts Services for agent sessions
      session-manager.ts  Session management (~49KB)
      package-manager.ts  Package management (~80KB)
      settings-manager.ts Settings management (~38KB)
      model-registry.ts   Model registry (~34KB)
      model-resolver.ts   Model resolution (~24KB)
      resource-loader.ts  Resource loading (~37KB)
      sdk.ts              SDK for extensions (~14KB)
      skills.ts           Skills system (~14KB)
      tools/              Tool implementations
      compaction/         Session compaction
      extensions/         Extension system
      export-html/        HTML export
      keybindings.ts      Keybinding config
      bash-executor.ts    Bash execution
      auth-storage.ts     Auth persistence
      prompt-templates.ts Prompt templates
      system-prompt.ts    System prompt generation
      slash-commands.ts   Slash command handling
      trust-manager.ts    Project trust
      http-dispatcher.ts  HTTP dispatch
      event-bus.ts        Event system
      telemetry.ts        Usage telemetry
    modes/               Operating modes
      interactive/       TUI mode
      print-mode.ts      Print mode
      rpc/               RPC mode
    utils/               Utilities (images, git, shell, syntax highlighting, etc.)
    bun/                 Bun-specific platform glue

  docs/                  Extensive user-facing documentation (~30 docs)
  examples/              Extension examples
  test/                  ~120+ test files
```

## CLI entry points

- **`pi`** — interactive TUI mode (default)
- **`pi -p "prompt"`** — print mode (non-interactive)
- **`pi --rpc`** — RPC mode (JSON-RPC over stdin/stdout)
- **`pi --model <id>`** — select model
- **`pi --list-models`** — list available models
- **`pi --list-sessions`** — list sessions
- **`pi --resume <id>`** — resume a session

## Existing documentation

The coding-agent package has extensive end-user documentation in `packages/coding-agent/docs/`. Key pages:

- [quickstart.md](../../packages/coding-agent/docs/quickstart.md) — install and first session
- [usage.md](../../packages/coding-agent/docs/usage.md) — interactive mode, slash commands, CLI reference
- [providers.md](../../packages/coding-agent/docs/providers.md) — provider setup and auth
- [extensions.md](../../packages/coding-agent/docs/extensions.md) — extension development guide
- [sdk.md](../../packages/coding-agent/docs/sdk.md) — Node.js SDK
- [rpc.md](../../packages/coding-agent/docs/rpc.md) — RPC mode spec
- [sessions.md](../../packages/coding-agent/docs/sessions.md) — session management
- [compaction.md](../../packages/coding-agent/docs/compaction.md) — context compaction
- [settings.md](../../packages/coding-agent/docs/settings.md) — configuration reference
- [themes.md](../../packages/coding-agent/docs/themes.md) — terminal themes
- [keybindings.md](../../packages/coding-agent/docs/keybindings.md) — keyboard shortcuts
- [skills.md](../../packages/coding-agent/docs/skills.md) — Agent Skills
- [security.md](../../packages/coding-agent/docs/security.md) — security model

## Recent changes

- Skip unauthenticated default model (prevents errors when no auth configured)
- Abort stuck context hooks
- Set `executionMode: sequential` on question example tool
- Surface auth storage save failures
- Serialize split-turn compaction summaries
- Add entry renderers for session entries
- Expose model resolution helpers
- Reject non-positive and oversized bash timeouts