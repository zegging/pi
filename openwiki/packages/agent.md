# packages/agent — `@earendil-works/pi-agent-core`

Agent runtime with tool calling, state management, session persistence, and context compaction. Built on top of `@earendil-works/pi-ai`.

## Two layers

The agent package has two distinct layers:

### 1. Core Agent (`src/agent.ts`, `src/agent-loop.ts`)

The low-level agent loop. Works with `AgentMessage` (a flexible, app-specific message type) throughout, only converting to LLM-compatible `Message[]` at the boundary.

**`Agent` class** — stateful wrapper that:
- Owns the transcript (messages with usage tracking)
- Emits lifecycle events (`AgentEvent`)
- Executes tool calls with before/after hooks
- Exposes queueing APIs for steering and follow-up messages

**`agentLoop()`** — processes a prompt, streams LLM responses, executes tool calls, handles inner/outer loops for tool chaining. Returns an `EventStream<AgentEvent, AgentMessage[]>`.

**`agentLoopContinue()`** — continues an existing session without adding a new prompt (used for retries, follow-up without user input).

Key design: the optional `transformContext` hook runs before each LLM call, allowing context pruning, injection, or modification.

### 2. Agent Harness (`src/harness/`)

A higher-level, opinionated framework for building durable agent applications:

| Module | Purpose |
|--------|---------|
| `AgentHarness` | Main class — wraps the core agent loop with session management, compaction, hooks, skills, prompt templates, and resource management |
| `Session` | Session state: messages, metadata, branches, parent links |
| `compaction/` | Context window management — token estimation, cut points, summarization, branch summaries |
| `env/nodejs.ts` | `NodeExecutionEnv` — filesystem and execution environment for Node.js |
| `skills.ts` | Skill loading, formatting, and invocation |
| `prompt-templates.ts` | Prompt template formatting |
| `system-prompt.ts` | System prompt generation |

## Tool execution

Tools are defined with:

- **Schema** — TypeBox schema for argument validation
- **Description** — Natural language description for the LLM
- **Execution mode** — `"sequential"` or `"parallel"` (controls whether multiple tool calls from one assistant message run in sequence or concurrently)
- **Hooks** — `beforeToolCall` and `afterToolCall` hooks for blocking, modifying, or reacting to tool execution

The `QueueMode` setting (`"all"` or `"one-at-a-time"`) controls how queued user messages are injected at drain points.

## Session management

Sessions are stored as JSONL files in `~/.pi/agent/sessions/`. Each session:

- Has a unique UUIDv7 ID
- Contains a sequence of entries (messages, tool calls, tool results, system messages)
- Supports branching (fork from any point)
- Tracks metadata (created, updated, model, provider, branch info)

The `SessionManager` (in `packages/coding-agent`) builds on these primitives to provide session listing, switching, forking, and tree navigation.

## Compaction

When context windows fill up, the compaction system:

1. **Estimates** token usage for each message
2. **Finds cut points** — where to truncate the conversation
3. **Generates summaries** — compresses older turns into a summary injected into the system prompt
4. **Branch summarization** — when forking, generates a summary of the parent branch

Compaction is configurable via `DEFAULT_COMPACTION_SETTINGS` and can be tuned per session.

## Package structure

```
packages/agent/src/
  index.ts              Public API surface
  node.ts               Node.js entry point (+ NodeExecutionEnv)
  types.ts              Core types: AgentMessage, AgentTool, AgentEvent, AgentContext, etc.
  agent.ts              Agent class — stateful wrapper
  agent-loop.ts         Agent loop implementations (agentLoop, runAgentLoop, continue)
  proxy.ts              Stream proxy for routing LLM calls through a server

  harness/              Higher-level agent framework
    agent-harness.ts    AgentHarness class
    types.ts            Harness types: Session, FileSystem, ExecutionEnv, Skill, Result<T>
    messages.ts         Custom message types (bashExecution, custom, branchSummary, etc.)
    session/            Session persistence
      session.ts        Session class
      jsonl-storage.ts  JSONL file storage
      jsonl-repo.ts     JSONL repository
      memory-storage.ts In-memory storage
      memory-repo.ts    In-memory repository
      uuid.ts           UUIDv7 generation
      repo-utils.ts     Repository utilities
    compaction/         Context window management
      compaction.ts     Token estimation, cut points, summarization
      branch-summarization.ts  Branch summary generation
      utils.ts          Compaction utilities
    env/                Execution environments
      nodejs.ts         NodeExecutionEnv + NodeFileSystem
    skills.ts           Skill loading and invocation
    prompt-templates.ts Prompt template formatting
    system-prompt.ts    System prompt generation
    utils/              shell-output.ts, truncate.ts
```

## Key types

- **`AgentMessage`** — flexible message type (user, assistant, toolResult, system, custom) — app-specific, not directly LLM-compatible
- **`AgentTool`** — tool definition with TypeBox schema, description, execution mode, hooks
- **`AgentEvent`** — discriminated union of lifecycle events (turnStart, turnUpdate, turnEnd, toolCallStart, toolCallEnd, error, etc.)
- **`StreamFn`** — function signature for streaming LLM calls; `Models.streamSimple` satisfies this

## Related docs

- [packages/coding-agent/docs/sessions.md](../../packages/coding-agent/docs/sessions.md) — end-user session management
- [packages/coding-agent/docs/compaction.md](../../packages/coding-agent/docs/compaction.md) — end-user compaction guide
- [packages/coding-agent/docs/session-format.md](../../packages/coding-agent/docs/session-format.md) — JSONL session format spec