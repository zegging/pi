# packages/ai — `@earendil-works/pi-ai`

Unified multi-provider LLM API with automatic model discovery and provider configuration. This is the foundation layer that all other pi packages build on.

## Core concepts

### Provider model

A `Provider<TApi>` is the concrete runtime unit. Each provider declares:

- **id/name** — identity and display name
- **auth** — authentication method (API key, OAuth, ambient credentials)
- **getModels()** — returns the current model catalog (sync; static for most providers, dynamic for some)
- **stream()** — delegates to the appropriate API implementation for streaming completions

Providers are PLUGGABLE. The `Models` class manages a collection of providers, handles model lookup, auth resolution, and stream delegation.

### API abstraction

Each LLM protocol is an `Api` implementation. Known APIs:

| API | Protocol |
|-----|----------|
| `openai-completions` | OpenAI Chat Completions |
| `openai-responses` | OpenAI Responses API |
| `openai-codex-responses` | OpenAI Codex Responses |
| `azure-openai-responses` | Azure OpenAI Responses |
| `anthropic-messages` | Anthropic Messages |
| `bedrock-converse-stream` | AWS Bedrock Converse Stream |
| `google-generative-ai` | Google Generative AI |
| `google-vertex` | Google Vertex AI |
| `mistral-conversations` | Mistral Conversations |
| `openrouter-images` | OpenRouter Images (images API) |

API implementations handle request construction, streaming, error handling, and provider-specific quirks. They are lazy-loaded to avoid pulling in all SDKs at once.

### Authentication

Auth is flexible and provider-specific:

- **API keys** — environment variables or stored in `~/.pi/agent/auth.json`
- **OAuth** — device-code flow for subscription providers (Claude Pro/Max, ChatGPT Plus/Pro, GitHub Copilot)
- **Ambient credentials** — AWS profiles, ADC files, keyless local servers

The `AuthStorage` class (in `packages/coding-agent`) persists credentials, while `packages/ai` provides the auth resolution logic (`resolveProviderAuth`).

### Model discovery

- **Static providers** — model catalog is code-generated in `*.models.ts` files
- **Dynamic providers** — implement `refreshModels()` for runtime discovery from provider APIs
- **`models.generated.ts`** — auto-generated via `packages/ai/scripts/generate-models.ts`; never edit directly

## Deep dives

- [Provider 与 Model 建模设计](provider-model.md) — `createProvider` 入参/出参、`Provider<TApi>` 泛型、多态实现、`Model` vs `Provider` 协作关系、compat vs Models 双路径、5 层委托链、调用链全景
- [流事件协议](stream-events.md) — `AssistantMessageEvent` 类型定义、事件生命周期、EventStream 架构、Provider 映射机制
- [自定义 Provider 接入](custom-providers.md) — 如何用 `models.json` 或 Extension `registerProvider()` 接入自定义 provider，含完整代码示例

## Notable recent changes

- **Codex WebSocket session rotation** — WebSocket sessions older than 55 minutes are proactively closed and re-established to avoid stale session errors (`openai-codex-responses.ts`).
- **Device-code `slow_down` honors server interval** — GitHub's OAuth device-code flow now reports the required poll interval in the `slow_down` response. The client uses this server-provided value instead of a client-tracked increment, fixing WSL/VM clock drift issues (`device-code.ts`, `github-copilot.ts`).
- **Cloudflare 524 timeout retry** — HTTP 524 (Cloudflare timeout) is now treated as a retryable error (`retry.ts`).

## Package structure

```
packages/ai/src/
  index.ts              Public API surface (core types, no generated catalogs)
  types.ts              Central type definitions (Api, Provider, Model, Context, Usage, etc.)
  models.ts             Models runtime collection, model lookup, auth resolution

  api/                  API implementations (one per protocol)
    anthropic-messages.ts
    bedrock-converse-stream.ts
    google-generative-ai.ts
    google-vertex.ts / google-shared.ts
    mistral-conversations.ts
    openai-completions.ts
    openai-responses.ts / openai-responses-shared.ts
    openai-codex-responses.ts
    azure-openai-responses.ts
    openrouter-images.ts
    cloudflare.ts
    lazy.ts              Lazy loading helpers

  providers/            30+ provider factories
    all.ts               All providers aggregated via builtinProviders()
    anthropic.ts, openai.ts, google.ts, mistral.ts, ...
    xai.ts, groq.ts, deepseek.ts, fireworks.ts, ...
    github-copilot.ts, openrouter.ts, vercel-ai-gateway.ts, ...
    zai.ts, minimax.ts, moonshotai.ts, kimi-coding.ts, ...
    Each provider has a companion .models.ts file

  auth/                 Authentication
    types.ts            Auth types: ProviderAuth, AuthResult, AuthContext, OAuthProvider
    resolve.ts          Auth resolution logic
    credential-store.ts InMemoryCredentialStore
    context.ts          OAuth context (callbacks, state)
    helpers.ts          Auth utility helpers

  utils/                Utilities
    event-stream.ts     Streaming infrastructure
    retry.ts, overflow.ts, abort-signals.ts
    diagnostics.ts      Assistant message diagnostics
    json-parse.ts       Partial JSON parsing
```