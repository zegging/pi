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

- [Provider 建模设计深度分析](ai-provider-model.md) — `createProvider` 入参/出参、`Provider<TApi>` 泛型设计、单一 API vs 多 API 路由表、Ambient/OAuth/自定义 Env 认证实现差异、`Models` 运行时编排、调用链全景
- [流事件协议分析](ai-stream-events.md) — `AssistantMessageEvent` 类型定义、事件生命周期、Provider 映射机制、HTTP 传输层关系

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
    validation.ts       TypeBox-based validation
    oauth/              OAuth provider interfaces

  images.ts, images-models.ts, images-api-registry.ts
                        Image generation support
  image-models.generated.ts  Auto-generated image model catalog
  models.generated.ts   Auto-generated text model catalog

  compat.ts             Legacy compatibility layer
  env-api-keys.ts       Environment variable API key resolution
  cli.ts                CLI binary (pi-ai command)
```

## Key entry points

- **`@earendil-works/pi-ai`** → `src/index.ts` — core types, `Models`, auth types, no generated catalogs
- **`@earendil-works/pi-ai/compat`** → `src/compat.ts` — legacy compatibility layer
- **`@earendil-works/pi-ai/bedrock-provider`** → `src/bedrock-provider.ts` — Bedrock-specific entry

## Working with models

The `Models` class is the central coordination point:

```typescript
import { createModels } from "@earendil-works/pi-ai";

const models = createModels();
// Add providers
models.addProvider(anthropicProvider());
// Look up models
const model = models.getModel("claude-sonnet-5-20250929");
// Stream a completion
const stream = models.stream({ model, messages, tools });
```

`Models` handles:
- Model lookup by ID (with fuzzy matching)
- Auth resolution (lazy, only when streaming)
- Provider selection and delegation
- Stream lifecycle management

## Adding a new provider

1. Create `packages/ai/src/providers/<name>.models.ts` with the static model catalog
2. Create `packages/ai/src/providers/<name>.ts` with the provider factory
3. Register in `packages/ai/src/providers/all.ts`
4. Update `packages/ai/scripts/generate-models.ts` if needed and regenerate

## Recent changes

- Bedrock prompt caching for Claude 5 enabled
- DS4 context overflow error detection added
- Copilot device-code token polling delay
- Fireworks GLM 5.2 Fast alignment with GLM 5.2
- Copilot Claude Sonnet 5 added

## Related docs

- [packages/coding-agent/docs/providers.md](../../packages/coding-agent/docs/providers.md) — end-user provider setup
- [packages/coding-agent/docs/models.md](../../packages/coding-agent/docs/models.md) — custom model configuration
- [packages/coding-agent/docs/custom-provider.md](../../packages/coding-agent/docs/custom-provider.md) — implementing custom providers