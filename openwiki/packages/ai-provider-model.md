# Provider 建模设计深度分析

> 基于 `packages/ai/src/models.ts` 中 `createProvider` 的入参、出参和相关类型定义，结合所有已知 provider 实现，对统一概念下的不同实现路径进行逐层分析。

## 目录

- [1. 核心类型层级](#1-核心类型层级)
- [2. `CreateProviderOptions` — 入参设计](#2-createprovideroptions--入参设计)
- [3. `createProvider` 工厂函数](#3-createprovider-工厂函数)
- [4. `Provider<TApi>` — 出参接口](#4-providertapi--出参接口)
  - [4.1 `stream` 的泛型设计](#41-stream-的泛型设计)
  - [4.2 `stream` vs `streamSimple` —— 协议感知 vs 协议无关](#42-stream-vs-streamsimple--协议感知-vs-协议无关)
- [5. 多态维度：同一接口的不同实现](#5-多态维度同一接口的不同实现)
  - [5.1 单一 API 提供者](#51-单一-api-提供者)
  - [5.2 多 API 提供者（路由表模式）](#52-多-api-提供者路由表模式)
  - [5.3 Ambient 认证提供者](#53-ambient-认证提供者)
  - [5.4 自定义 ProviderEnv 认证提供者](#54-自定义-providerenv-认证提供者)
  - [5.5 OAuth 提供者](#55-oauth-提供者)
  - [5.6 对比总结](#56-对比总结)
- [6. `Models` 集合 —— 运行时编排层](#6-models-集合--运行时编排层)
- [7. 调用链全景](#7-调用链全景)
- [8. 设计约束与权衡](#8-设计约束与权衡)

---

## 1. 核心类型层级

```
               Api (string 联合类型)
               ├── KnownApi          (9 种已知 API)
               │   ├── "openai-completions"       (Chat Completions 协议)
               │   ├── "openai-responses"         (Responses API 协议)
               │   ├── "openai-codex-responses"   (Codex 专用)
               │   ├── "azure-openai-responses"   (Azure + Responses)
               │   ├── "anthropic-messages"       (Anthropic Messages)
               │   ├── "bedrock-converse-stream"  (AWS Bedrock)
               │   ├── "google-generative-ai"     (Gemini API)
               │   ├── "google-vertex"            (Vertex AI)
               │   └── "mistral-conversations"    (Mistral)
               └── (string & {})         (自定义/未知 API，保留扩展点)

     ProviderAuth                   (联合 auth 策略)
     ├── apiKey?: ApiKeyAuth        (API key 认证，含 ambient)
     └── oauth?: OAuthAuth          (OAuth 认证)

     ProviderStreams                (API 实现模块的约定接口)
     ├── stream(model, ctx, opts?)  → AssistantMessageEventStream
     └── streamSimple(model, ctx, opts?) → AssistantMessageEventStream

     Model<TApi>                    (单个模型描述)
     ├── id, name, api, provider, baseUrl
     ├── reasoning, contextWindow, maxTokens
     ├── cost, input[], thinkingLevelMap?
     └── compat? (API 特定的兼容覆写)
```

关键设计动机：**`TApi` 泛型是类型系统层面的"API 路由键"**。`Provider<TApi>` 通过泛型声明其模型使用哪些 API，`createProvider` 利用此泛型确保 `api` 字段与 `models` 的 `api` 字段类型一致。`Models` 集合内部擦除泛型为 `Provider<Api>`，对外暴露 `Model<Api>`，调用方通过 `hasApi()` 类型守卫恢复具体类型。

---

## 2. `CreateProviderOptions` — 入参设计

```ts
// 文件: packages/ai/src/models.ts:295-315
export interface CreateProviderOptions<TApi extends Api = Api> {
    id: string;
    name?: string;                         // 默认: id
    baseUrl?: string;
    headers?: ProviderHeaders;
    auth: ProviderAuth;                    // 必填 — 每个 provider 都有 auth 语义
    models: readonly Model<TApi>[];        // 初始模型列表
    refreshModels?: () => Promise<readonly Model<TApi>[]>;
    api: ProviderStreams | Partial<Record<TApi, ProviderStreams>>;
}
```

### 2.1 各字段的语义与约束

| 字段 | 必填 | 说明 |
|------|------|------|
| `id` | ✅ | 全局唯一的 provider 标识符，对应 `KnownProvider` 联合类型中的值 |
| `name` | 否 | 展示用名，默认 fallback 到 `id` |
| `baseUrl` | 否 | 该 provider 所有模型的默认 API 端点。可在 `Model.baseUrl` 层面覆盖 |
| `headers` | 否 | 该 provider 所有请求的默认 HTTP 头。可在流式调用时通过 `options.headers` 覆盖/合并 |
| `auth` | ✅ | **强制要求**。即使本地 keyless 服务（如 Ollama）也必须提供 `apiKey` auth 来报告是否已配置 |
| `models` | ✅ | 静态初始模型列表。纯动态 provider 传空数组 `[]` |
| `refreshModels` | 否 | 仅动态 provider 实现。返回新的模型列表，`createProvider` 内部用 `inflightRefresh` 保证并发调用共享同一请求 |
| `api` | ✅ | 单一实现或按 API 分派的路由表 |

### 2.2 `api` 字段的双态设计

`api` 字段接受两种形态，这是设计的核心灵活点：

**形态 1：单一 `ProviderStreams` 实例**

```ts
// 示例: Anthropic, OpenAI, Google, OpenRouter 等
// 所有该 provider 的模型使用同一种 API 协议
api: anthropicMessagesApi()   // 类型: ProviderStreams
```

在 `createProvider` 内部，通过运行时检测 `typeof (input.api).stream === "function"` 来判断是否为单一实现。此时 `byApi` 为 `undefined`，`apiFor()` 始终返回该单一实例。

**形态 2：`Partial<Record<TApi, ProviderStreams>>` 路由表**

```ts
// 示例: GitHub Copilot, OpenCode Zen
// 不同模型使用不同 API 协议
api: {
    "anthropic-messages": anthropicMessagesApi(),
    "openai-completions": openAICompletionsApi(),
    "openai-responses": openAIResponsesApi(),
}
```

此时 `apiFor()` 根据 `model.api` 查表分派。若模型声明的 `api` 在路由表中无对应条目，`dispatch()` 返回一个立即以 `ModelsError("stream")` 终止的错误流——不会抛同步异常，符合"流错误编码在流内部"的契约。

---

## 3. `createProvider` 工厂函数

```ts
// 文件: packages/ai/src/models.ts:323-369
export function createProvider<TApi extends Api = Api>(
    input: CreateProviderOptions<TApi>
): Provider<TApi> {
```

### 3.1 内部状态

```ts
let models = input.models;                          // 可变模型列表（动态刷新时更新）
let inflightRefresh: Promise<void> | undefined;     // 去重：并发 refresh 共享一个 Promise
const single = typeof (input.api).stream === "function"
    ? (input.api as ProviderStreams)
    : undefined;                                    // 形态 1: 单一实现
const byApi = single ? undefined
    : (input.api as Partial<Record<string, ProviderStreams>>);  // 形态 2: 路由表
```

### 3.2 `apiFor` 分派逻辑

```ts
const apiFor = (model: Model<Api>): ProviderStreams | undefined =>
    single ?? byApi?.[model.api];
```

`single` 优先——若为单一实现则无视 `model.api` 的值，所有模型走同一实现。仅在 `single` 为 `undefined` 时才查路由表。

### 3.3 `dispatch` 错误处理

```ts
const dispatch = (model, run) => {
    const streams = apiFor(model);
    if (!streams) {
        return lazyStream(model, async () => {
            throw new ModelsError("stream",
                `Provider ${input.id} has no API implementation for "${model.api}"`);
        });
    }
    return run(streams);
};
```

注意：当找不到对应 API 实现时，**不抛同步异常**，而是通过 `lazyStream` 返回一个异步错误流。这是 `StreamFunction` 的核心契约：所有错误（包括配置错误）都编码在流内部。

### 3.4 `refreshModels` 的去重机制

```ts
refreshModels: refreshModels
    ? () => {
        inflightRefresh ??= (async () => {
            try {
                models = await refreshModels();
            } finally {
                inflightRefresh = undefined;
            }
        })();
        return inflightRefresh;
    }
    : undefined,
```

关键行为：
- 首次调用设置 `inflightRefresh`，后续并发调用返回同一个 Promise
- 成功时更新 `models` 变量（`getModels()` 闭包捕获的变量）
- 失败时 `inflightRefresh` 也在 `finally` 中清除，下次调用会重试
- 失败时 `models` 保持上次已知状态不变

---

## 4. `Provider<TApi>` — 出参接口

```ts
// 文件: packages/ai/src/models.ts:32-72
export interface Provider<TApi extends Api = Api> {
    readonly id: string;
    readonly name: string;
    readonly baseUrl?: string;
    readonly headers?: ProviderHeaders;
    readonly auth: ProviderAuth;
    getModels(): readonly Model<TApi>[];
    refreshModels?(): Promise<void>;
    stream<T extends TApi>(model: Model<T>, context: Context, options?: ApiStreamOptions<T>): AssistantMessageEventStream;
    streamSimple(model: Model<TApi>, context: Context, options?: SimpleStreamOptions): AssistantMessageEventStream;
}
```

### 4.1 `stream` 的泛型设计

`stream<T extends TApi>` 是 `Provider` 最复杂的泛型签名。它允许：
- 调用方传入 `Model<TApi>` 的子类型 `Model<T>`（`T extends TApi`）
- 对应的 `options` 自动解析为 `ApiStreamOptions<T>`（即具体 API 的完整选项类型）

这要求 `Provider<"anthropic-messages">` 的 `stream` 方法接受 `Model<"anthropic-messages">` 并给出 `AnthropicOptions` 类型的选项。`createProvider` 内部通过 `dispatch` + `run` 闭包类型推导实现此约束。

### 4.2 `stream` vs `streamSimple` —— 协议感知 vs 协议无关

核心区别：**调用方是否知道自己在跟哪个 API 协议对话。**

| | `stream` | `streamSimple` |
|---|---|---|
| **调用方知识** | 知道自己用的是哪个协议 | 只知道"我要推理" |
| **options 类型** | `ApiStreamOptions<TApi>` — 协议特有参数 | `SimpleStreamOptions` — 跨协议统一 |
| **泛型** | `stream<T extends TApi>`，与 model 联动 | `streamSimple(model: Model<TApi>)`，不引入额外泛型 |
| **典型 options** | `effort`, `thinkingBudgetTokens`, `reasoningEffort`, `serviceTier`... | `reasoning?: ThinkingLevel`, `thinkingBudgets?` |

`ApiStreamOptions<TApi>` 通过 `ApiOptionsMap` 条件类型映射到各协议的具体选项：

```
"anthropic-messages"    → AnthropicOptions      (thinkingEnabled, thinkingBudgetTokens, effort...)
"openai-responses"      → OpenAIResponsesOptions (reasoningEffort, reasoningSummary, serviceTier...)
"google-generative-ai"  → GoogleOptions          (google 特有的 thinking 参数)
"bedrock-converse-stream" → BedrockOptions       (AWS 特有的 thinking 参数)
...
```

这些类型彼此不兼容——`effort` 是 Anthropic 概念，`reasoningEffort` 是 OpenAI 概念。只有 `stream` 的调用方（知道自己用哪个协议）才能正确填充。

#### `streamSimple` 的实现：始终委托给 `stream`

`streamSimple` 从不直接调用 SDK。它是对 `stream` 的适配层，唯一职责是将统一的 `ThinkingLevel` 翻译为对应协议的参数，然后调用 `stream`。

以 Anthropic 为例（`packages/ai/src/api/anthropic-messages.ts:767-807`）：

```ts
export const streamSimple = (model, context, options) => {
    // 没要 reasoning → 关掉 thinking
    if (!options?.reasoning) {
        return stream(model, context, { thinkingEnabled: false });
    }

    // 自适应 thinking 模型（Opus 4.6+）→ ThinkingLevel 映射为 effort
    if (model.compat?.forceAdaptiveThinking) {
        const effort = mapThinkingLevelToEffort(model, options.reasoning);
        return stream(model, context, { thinkingEnabled: true, effort });
    }

    // 旧模型（token budget 模式）→ 计算 thinkingBudgetTokens
    const adjusted = adjustMaxTokensForThinking(
        base.maxTokens, model.maxTokens, options.reasoning, options.thinkingBudgets
    );
    return stream(model, context, {
        thinkingEnabled: true,
        thinkingBudgetTokens: adjusted.thinkingBudget,
        maxTokens: adjusted.maxTokens,
    });
};
```

**所有 9 个已知 API 的 `streamSimple` 实现都遵循同一模式：先翻译参数，再调用 `stream`。** 翻译逻辑因协议而异，但对外暴露的 `SimpleStreamOptions` 完全一致。

#### 为什么需要两层

调用链中有两类消费者：

1. **知道协议细节的代码**（如 agent 的 model selector 想给 Claude 传 `effort: "xhigh"`）→ 用 `stream`，完全控制
2. **不知道协议细节的代码**（如 coding-agent 的 `--thinking` flag，用户只选了 `high`）→ 用 `streamSimple`，pi 负责把 `"high"` 翻译成 Anthropic 的 `effort: "high"` 或 OpenAI 的 `reasoningEffort: "high"`

如果没有 `streamSimple`，每个调用方都得自己写一份 `ThinkingLevel → 协议参数` 的映射逻辑，而且很容易写错（比如把 `effort` 传给 OpenAI）。

---

## 5. 多态维度：同一接口的不同实现

### 5.1 单一 API 提供者

**代表：Anthropic, OpenAI, Google, OpenRouter, DeepSeek, Groq, xAI, Cerebras...**

```ts
// 示例: packages/ai/src/providers/openai.ts
export function openaiProvider(): Provider<"openai-responses"> {
    return createProvider({
        id: "openai",
        name: "OpenAI",
        baseUrl: "https://api.openai.com/v1",
        auth: { apiKey: envApiKeyAuth("OpenAI API key", ["OPENAI_API_KEY"]) },
        models: Object.values(OPENAI_MODELS),
        api: openAIResponsesApi(),   // ← 单一 ProviderStreams
    });
}
```

特点：
- 泛型 `TApi` 被约束为单一已知 API 类型
- 所有模型共享同一个 `api` 字段值
- `api` 参数为 `ProviderStreams`（单一实例）
- 工厂函数返回类型明确：`Provider<"openai-responses">`

### 5.2 多 API 提供者（路由表模式）

**代表：GitHub Copilot（3 种 API），OpenCode Zen（4 种 API）**

```ts
// 示例: packages/ai/src/providers/github-copilot.ts
export function githubCopilotProvider():
    Provider<"anthropic-messages" | "openai-completions" | "openai-responses"> {
    return createProvider({
        id: "github-copilot",
        name: "GitHub Copilot",
        baseUrl: "https://api.individual.githubcopilot.com",
        auth: {
            apiKey: envApiKeyAuth("GitHub Copilot token", ["COPILOT_GITHUB_TOKEN"]),
            oauth: lazyOAuth({ name: "GitHub Copilot", load: loadGitHubCopilotOAuth }),
        },
        models: Object.values(GITHUB_COPILOT_MODELS),
        api: {
            "anthropic-messages": anthropicMessagesApi(),
            "openai-completions": openAICompletionsApi(),
            "openai-responses": openAIResponsesApi(),
        },
    });
}
```

特点：
- `TApi` 为联合类型（如 `"anthropic-messages" | "openai-completions" | "openai-responses"`）
- 每个模型在 `*.models.ts` 中声明自己的 `api` 字段
- `api` 参数为 `Partial<Record<TApi, ProviderStreams>>`（路由表）
- 运行时 `apiFor()` 根据 `model.api` 查表分派
- 若模型的 `api` 值不在路由表中，返回错误流

**多 API 提供者解决的核心问题**：某些平台（如 GitHub Copilot）在不同模型背后使用不同的后端协议，但对用户应该表现为**同一个 provider**。用户只需配置一次 auth，模型选择由 provider 内部路由。

### 5.3 Ambient 认证提供者

**代表：Amazon Bedrock**

**Ambient 认证**是指：不需要用户向 pi 显式提供 API key，而是依赖运行环境中已经存在的凭据。pi 代码不持有、不传递 key，只负责检测"是否已配置"。

**以 Bedrock 为例**（`packages/ai/src/providers/amazon-bedrock.ts:6-25`）：

```ts
const bedrockAuth: ApiKeyAuth = {
    name: "AWS credentials",
    // 注意: 无 login() 方法! 用户无法通过 pi 交互式配置
    resolve: async ({ ctx, credential }) => {
        // 按优先级检测环境中已有的 AWS 凭据:
        if (credential?.key) return { auth: { apiKey: credential.key }, source: "stored credential" };
        if (await ctx.env("AWS_BEARER_TOKEN_BEDROCK")) return { auth: {}, source: "AWS_BEARER_TOKEN_BEDROCK" };
        if (await ctx.env("AWS_PROFILE")) return { auth: {}, source: "AWS_PROFILE" };
        if ((await ctx.env("AWS_ACCESS_KEY_ID")) && (await ctx.env("AWS_SECRET_ACCESS_KEY"))) {
            return { auth: {}, source: "AWS access keys" };
        }
        if (await ctx.env("AWS_CONTAINER_CREDENTIALS_RELATIVE_URI")) return { auth: {}, source: "ECS task role" };
        if (await ctx.env("AWS_CONTAINER_CREDENTIALS_FULL_URI")) return { auth: {}, source: "ECS task role" };
        if (await ctx.env("AWS_WEB_IDENTITY_TOKEN_FILE")) return { auth: {}, source: "web identity token" };
        return undefined;  // 未配置
    },
};
```

**关键行为**：所有命中都返回 `{ auth: {} }`——空对象。因为 AWS SDK 的 credential chain 会在发出请求时自己完成 SigV4 签名，pi 不需要把 key 注入到 HTTP 请求中。但 `resolve()` 返回 `{ auth: {} }` 仍然表示"已配置"，和返回 `undefined`（未配置）有本质区别。

**为什么叫 ambient**：凭据是"环境本身就有的"——env vars、`~/.aws/credentials` 文件、ECS metadata service、IAM 角色。pi 代码不收集、不存储、不传递它们。pi 只负责回答一个布尔问题："能不能用？"

**与标准 `envApiKeyAuth` 的根本差异**：

| | 标准 `envApiKeyAuth` | Ambient（Bedrock） |
|---|---|---|
| 凭据形式 | key 字符串 | 环境变量 / 文件 / IAM 角色 |
| pi 是否持有 key | 是，读取后注入请求头 | 否，SDK 内部签名 |
| `login()` | 有（交互式 prompt） | 无 |
| `resolve()` 返回 | `{ auth: { apiKey: "sk-..." } }` | `{ auth: {} }` |
| 用户操作 | 设置 env var 或 pi login | 配置 AWS 环境（`aws configure` 等） |

虽然 Bedrock 不需要 API key 字符串，但仍在 `ProviderAuth` 中通过 `apiKey` 渠道报告状态。这正是 `ProviderAuth` 强制要求至少提供 `apiKey` 或 `oauth` 之一的原因：**即使"不需要 key"也是一种 auth 策略，需要有渠道报告"是否已配置"。**

### 5.4 自定义 ProviderEnv 认证提供者

**代表：Cloudflare Workers AI, Cloudflare AI Gateway**

```ts
// 示例: packages/ai/src/providers/cloudflare-auth.ts
export function cloudflareWorkersAIAuth(): ApiKeyAuth {
    return {
        name: "Cloudflare API key",
        login: async (callbacks) => {
            const key = await callbacks.prompt({ type: "secret", ... });
            const accountId = await callbacks.prompt({ type: "text", ... });
            return { type: "api_key", key, env: { CLOUDFLARE_ACCOUNT_ID: accountId } };
        },
        resolve: async ({ model, ctx, credential }) => {
            // 需要 API key + account_id 两者都配置
            // 返回的 auth 中包含动态 baseUrl（替换 {CLOUDFLARE_ACCOUNT_ID} 占位符）
        },
    };
}
```

特点：
- `login()` 收集多个字段（不止 api key，还有 account ID）
- `ApiKeyCredential.env` 承载额外配置（`CLOUDFLARE_ACCOUNT_ID`）
- `resolve()` 返回的 `ModelAuth.baseUrl` 是动态的（包含 account ID 替换）
- `AuthResult.env` 额外传出 provider 环境变量供请求时使用

**解决的核心问题**：某些 provider 需要 **API key + 额外配置** 才能工作。`ApiKeyCredential.env` 和 `AuthResult.env` 提供了将额外配置从 credential 层传递到请求层的通道。

### 5.5 OAuth 提供者

**代表：Anthropic（Claude Pro/Max 订阅），GitHub Copilot，OpenAI Codex**

```ts
// 示例: packages/ai/src/providers/anthropic.ts
auth: {
    apiKey: envApiKeyAuth("Anthropic API key", ["ANTHROPIC_OAUTH_TOKEN", "ANTHROPIC_API_KEY"]),
    oauth: lazyOAuth({ name: "Anthropic (Claude Pro/Max)", load: loadAnthropicOAuth }),
}
```

特点：
- `ProviderAuth` 同时声明 `apiKey` 和 `oauth` 两种认证方式
- `resolveProviderAuth` 的优先级：若存在 stored credential，按 credential 类型选择对应 handler；无 stored credential 时走 ambient（apiKey 的 env var 查找）
- `lazyOAuth()` 延迟加载 OAuth 实现，避免将所有 OAuth 代码打包进 core bundle
- OAuth 的 `refresh`/`toAuth` 分离设计：`refresh` 在 credential store 锁内执行 token 刷新，`toAuth` 从 credential 派生请求 auth

**设计原理**：同一 provider 支持两类用户——API key 用户（按量付费）和订阅用户（Claude Pro/Max）。`ProviderAuth` 的 `apiKey` + `oauth` 双通道让两种用户都能使用同一 provider 定义。

### 5.6 对比总结

| 维度 | 单一 API（OpenAI） | 多 API（GitHub Copilot） | Ambient（Bedrock） | 自定义 Env（Cloudflare） | OAuth（Anthropic 订阅） |
|------|-------------------|-------------------------|-------------------|------------------------|------------------------|
| `api` 字段 | `ProviderStreams` | `Partial<Record<TApi, ProviderStreams>>` | `ProviderStreams` | `ProviderStreams` | `ProviderStreams` |
| `TApi` 泛型 | 单一类型 | 联合类型 | 单一类型 | 单一类型 | 单一类型 |
| `auth` | `{ apiKey }` | `{ apiKey, oauth }` | `{ apiKey }`（无 `login`） | `{ apiKey }`（自定义 `login`） | `{ apiKey, oauth }` |
| `baseUrl` | 静态 | 静态 | 无（SDK 内部） | 动态（占位符替换） | 静态 |
| `login()` | envApiKeyAuth 标准 | envApiKeyAuth + OAuth | 无 | 多字段 prompt | OAuth 设备码流 |
| `resolve()` 返回 | `{ apiKey: "sk-..." }` | 按 credential 类型 | `{ auth: {} }` → SDK 自签名 | `{ apiKey, baseUrl }` + `env` | OAuth `toAuth()` |

**额外维度：`stream` vs `streamSimple`**

| | `stream` | `streamSimple` |
|---|---|---|
| 调用方知识 | 知道协议 | 只知道"我要推理" |
| options 类型 | `ApiStreamOptions<TApi>`（协议特有） | `SimpleStreamOptions`（跨协议统一） |
| 实现方式 | 直接调 SDK | 翻译参数后调 `stream` |
| 典型参数 | `effort`, `thinkingBudgetTokens`, `reasoningEffort`... | `reasoning: "high"`

---

## 6. `Models` 集合 —— 运行时编排层

### 6.1 `ModelsImpl` 的核心职责

```ts
// 文件: packages/ai/src/models.ts:142-289
class ModelsImpl implements MutableModels {
    private providers = new Map<string, Provider>();
    private credentials: CredentialStore;
    private authContext: AuthContext;
    // ...
}
```

`ModelsImpl` 是 `Provider` 的运行时容器，负责：

1. **Provider 管理**：`setProvider` / `deleteProvider` / `clearProviders`
2. **模型查找**：`getModels(provider?)` / `getModel(provider, id)` —— 都是同步操作，best-effort
3. **模型刷新**：`refresh(provider?)` —— 调用动态 provider 的 `refreshModels()`，单 provider 失败抛 `ModelsError("model_source")`，全量刷新用 `allSettled` 不抛
4. **Auth 解析**：`getAuth(model)` —— 委托给 `resolveProviderAuth()`，处理 credential store 读锁和 OAuth 刷新
5. **流式调用**：`stream` / `streamSimple` / `complete` / `completeSimple`

### 6.2 `applyAuth` 的合并策略

```ts
// 文件: packages/ai/src/models.ts:230-256
private async applyAuth(model, options) {
    const resolution = await resolveProviderAuth(...);
    // 1. baseUrl: auth 层面可覆盖（如 Cloudflare 的动态 URL）
    const requestModel = auth.baseUrl ? { ...model, baseUrl: auth.baseUrl } : model;
    // 2. apiKey: 显式请求选项 > auth 解析结果
    const apiKey = options?.apiKey ?? auth.apiKey;
    // 3. headers: auth 层 + 请求层合并，请求层 override
    const headers = auth.headers || options?.headers
        ? { ...auth.headers, ...options?.headers }
        : undefined;
    // 4. env: auth 层 + 请求层合并，请求层 override
    const env = resolution.env || options?.env
        ? { ...(resolution.env ?? {}), ...(options?.env ?? {}) }
        : undefined;
}
```

合并优先级（从高到低）：
- `options.apiKey` > `auth.apiKey`（显式覆盖）
- `options.headers` > `auth.headers`（逐个 key 合并）
- `options.env` > `resolution.env`（逐个 key 合并）
- `auth.baseUrl` > `model.baseUrl`（仅当 auth 返回时才覆盖）

### 6.3 `MutableModels` 扩展

```ts
export interface MutableModels extends Models {
    setProvider(provider: Provider): void;
    deleteProvider(id: string): void;
    clearProviders(): void;
}
```

`createModels()` 返回 `MutableModels`，允许调用方动态添加/移除 provider。这支持：
- `builtinProviders()` 批量注册所有内置 provider
- `models.json` 自定义 provider 动态注入
- 测试时替换 provider

---

## 7. 调用链全景

```
用户代码
  │
  ├─ createModels({ credentials, authContext })
  │     └─ new ModelsImpl(options)
  │
  ├─ models.setProvider(anthropicProvider())
  │     └─ createProvider({ id, auth, models, api })
  │           ├─ 闭包捕获 models 列表
  │           ├─ 推断 single/byApi 分派模式
  │           └─ 返回 Provider<TApi> 对象
  │
  ├─ models.getModel("anthropic", "claude-sonnet-4-5")
  │     └─ 同步查 Map → getModels() → .find()
  │
  ├─ await models.getAuth(model)
  │     └─ resolveProviderAuth(provider, model, credentials, authContext)
  │           ├─ 有 stored credential → 按类型路由
  │           │    ├─ oauth → resolveStoredOAuth (含刷新锁)
  │           │    └─ api_key → resolveApiKey
  │           └─ 无 stored → ambient (env vars)
  │
  └─ models.stream(model, context, options)
        └─ lazyStream(model, async () => {
              ├─ applyAuth(model, options)     // 异步 auth 解析
              │    ├─ resolveProviderAuth()
              │    └─ 合并 apiKey/headers/env/baseUrl
              └─ provider.stream(requestModel, context, requestOptions)
                    └─ dispatch(model, streams => streams.stream(...))
                          ├─ apiFor(model) → 查 single 或 byApi 路由表
                          └─ 找到 → streams.stream(model, ctx, opts)
                          └─ 未找到 → lazyStream(throw ModelsError)
           })
```

---

## 8. 设计约束与权衡

### 8.1 同步 `getModels()` 保证

`Provider.getModels()` 和 `Models.getModels()` 均为同步方法。这是有意为之的约束：
- 调用方（UI、CLI）可以随时同步获取模型列表进行渲染
- 动态 provider 返回的是**上次刷新结果**，可能为空或过时
- 刷新是异步的、明确的：`await models.refresh()` 或 `await models.refresh("anthropic")`

### 8.2 流内部错误编码

`ProviderStreams` 的契约要求所有错误编码在流中，不抛同步异常。`createProvider` 的 `dispatch` 函数在找不到 API 实现时也遵循此契约——通过 `lazyStream` 返回一个立即以错误终止的流。这保证了调用方不需要额外的 try/catch 包裹 `stream()` 调用。

### 8.3 Auth 的"未配置"语义

`Models.getAuth()` 返回 `undefined` 表示 provider 不适用（未配置），与 `ModelsError` 抛出（auth 系统故障）有明确区分：
- `undefined` → UI 可以隐藏该 provider 或显示"请配置"
- `ModelsError("oauth")` → token 刷新失败，保留 credential 供重试
- `ModelsError("auth")` → credential store 故障

### 8.4 `lazyApi` 的按需加载

所有 API 实现模块通过 `lazyApi()` 包装，延迟到首次 `stream()` 调用时才加载。这意味着：
- 加载 30+ provider 的 `builtinProviders()` 不会引入所有 SDK 的 bundle
- 仅实际使用到的 provider 的 SDK 才会被加载
- `lazyOAuth()` 同理：OAuth 实现仅在用户触发登录流程时加载

### 8.5 `ProviderImages` 与 `Provider` 的平行设计

`ImagesProvider`（`packages/ai/src/images-models.ts`）是 `Provider` 的图像生成侧平行接口：
- 复用了 `ProviderAuth`（auth 层完全共享）
- 有自己的 `ImagesModels` 集合（平行于 `Models`）
- `ImagesProvider` 没有 `stream`/`streamSimple`，而是 `generateImages()`（返回 `Promise`，非流式）
- `ImagesProvider` 也没有泛型 API 路由表模式——目前只有 `openrouter-images` 一种图像 API

这体现了"相同模式、不同协议"的设计哲学：auth 和模型管理逻辑复用，但生成接口根据领域特征（流式 vs 一次性）独立定义。

---

## 相关源文件

| 文件 | 内容 |
|------|------|
| `packages/ai/src/models.ts` | `Provider`, `Models`, `createProvider`, `createModels`, `hasApi`, `calculateCost` |
| `packages/ai/src/types.ts` | `Api`, `KnownApi`, `ProviderId`, `Model`, `StreamOptions`, `ApiStreamOptions`, `ProviderStreams` |
| `packages/ai/src/auth/types.ts` | `ProviderAuth`, `ApiKeyAuth`, `OAuthAuth`, `AuthResult`, `CredentialStore`, `Credential` |
| `packages/ai/src/auth/resolve.ts` | `resolveProviderAuth`, `ModelsError`, `AuthResolutionOverrides` |
| `packages/ai/src/auth/helpers.ts` | `envApiKeyAuth`, `lazyOAuth` |
| `packages/ai/src/api/lazy.ts` | `lazyStream`, `lazyApi` |
| `packages/ai/src/images-models.ts` | `ImagesProvider`, `ImagesModels`（平行接口） |
| `packages/ai/src/providers/all.ts` | `builtinProviders()` 聚合 |
| `packages/ai/src/providers/anthropic.ts` | 单一 API + 双 auth 示例 |
| `packages/ai/src/providers/github-copilot.ts` | 多 API 路由表 + 双 auth 示例 |
| `packages/ai/src/providers/amazon-bedrock.ts` | Ambient 认证示例 |
| `packages/ai/src/providers/cloudflare-auth.ts` | 自定义 ProviderEnv 认证示例 |
| `packages/ai/src/providers/opencode.ts` | 多 API 路由表（4 种 API）示例 |
| `packages/ai/src/providers/faux.ts` | 测试用假 provider |