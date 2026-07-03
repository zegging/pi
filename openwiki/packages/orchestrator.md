# packages/orchestrator — `@earendil-works/pi-orchestrator`

**Experimental** orchestrator for managing long-running coding agent (pi) processes via IPC. Spawns, monitors, and routes RPC commands to agent instances.

This is a separate process that acts as a supervisor for pi agent instances, enabling remote access and multi-instance management.

## Architecture

### IPC over Unix sockets

The orchestrator uses a JSONL-based protocol over a Unix domain socket (`orchestrator.sock` in `~/.pi/orchestrator/`). The protocol is defined in `src/ipc/protocol.ts`.

### Supervisor pattern

`OrchestratorSupervisor` (`src/supervisor.ts`) manages live instances:
- Spawns and stops agent processes
- Routes RPC commands to the correct instance
- Manages event subscriptions (fan-out to subscribers)
- Recovers instances after orchestrator restart

### RPC process model

Each instance is an `RpcProcessInstance` (`src/rpc-process.ts`) — a child process running pi in RPC mode, communicating via JSONL over stdio. The orchestrator bridges this to its Unix socket IPC.

### Radius integration

Optional cloud presence system (`src/radius.ts`, ~13KB):
- Registers machines and instances with a remote Radius API
- Sends heartbeats with exponential backoff (1s base, 30s max)
- Enables remote access to local instances

### Streaming mode

`rpc_stream` upgrades a connection to bidirectional streaming for real-time agent interaction. The client sends JSONL `RpcCommand` or `extension_ui_response` messages on stdin and receives responses on stdout.

## CLI

```
orchestrator serve                   Start the IPC server
orchestrator list                    List all instances
orchestrator spawn [--cwd <path>] [--label <label>]  Create a new instance
orchestrator status <instance-id>    Get instance status
orchestrator stop <instance-id>      Stop an instance
orchestrator rpc <instance-id> <json-command>  Send a one-shot RPC command
orchestrator rpc-stream <instance-id>  Start bidirectional RPC streaming
```

## Session metadata refresh

The supervisor only refreshes persisted session metadata after commands that can change session identity: `new_session`, `switch_session`, `fork`, `clone`, `set_session_name`, `prompt`. This avoids wasted IO on transient RPC commands.

## Package structure

```
packages/orchestrator/
  src/
    cli.ts              CLI entry point with subcommands
    serve.ts            Server startup, recovery, Radius init
    supervisor.ts       OrchestratorSupervisor — instance lifecycle
    rpc-process.ts      RpcProcessInstance — child process management
    handler.ts          Request handler dispatching commands
    config.ts           Configuration (socket path, version, Bun detection)
    storage.ts          Persistence (machine.json, instances.json)
    types.ts            Shared types (InstanceStatus, MachineRecord, InstanceRecord)
    radius.ts           Radius cloud integration (~13KB)
    ipc/
      protocol.ts       IPC message types and JSONL encoding/decoding
      server.ts         Unix socket server
      client.ts         Unix socket client
```

## Relationship to other packages

- Depends on `@earendil-works/pi-coding-agent` for RPC types (`RpcCommand`, `RpcResponse`, `AgentSessionEvent`, etc.)
- Spawns pi as a child process running in RPC mode
- Independent of the TUI — runs headless