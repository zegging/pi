# Provider 与 Model 建模设计

> 基于 `packages/ai/src/models.ts` 中 `createProvider` 的入参、出参和类型定义，结合所有已知 provider 实现，对 `Provider<TApi>` 和 `Model<TApi>` 的类型层级、多态实现、运行时协作、调用链和设计模式进行全景分析。

## 目录

- [1. 核心类型层级](#1-核心类型层级)
- [2. Model — 纯数据抽象](#2-model--纯数据抽象)
- [3. Provider — 运行时服务抽象](#3-provider--运行时服务抽象)
- [4. `CreateProviderOptions` — 入参设计](#4-createprovideroptions--入参设计)
- [5. `createProvider` 工厂函数](#5-createprovider-工厂函数)
- [6. `Provider<TApi>` — 出参接口](#6-providertapi--出参接口)
  - [6.1 `stream` 的泛型设计](#61-stream-的泛型设计)
  - [6.2 `stream` vs `streamSimple`](#62-stream-vs-streamsimple)
- [7. 多态维度：同一接口的不同实现](#7-多态维度同一接口的不同实现)
  - [7.1 单一 API 提供者](#71-单一-api-提供者)
  - [7.2 多 API 提供者（路由表模式）](#72-多-api-提供者路由表模式)
  - [7.3 Ambient 认证提供者](#73-ambient-认证提供者)
  - [7.4 自定义 ProviderEnv 认证](#74-自定义-providerenv-认证)
  - [7.5 OAuth 提供者](#75-oauth-提供者)
  - [7.6 对比总结](#76-对比总结)
- [8. `Models` 集合 —— 运行时编排层](#8-models-集合--运行时编排层)
- [9. 调用链全景](#9-调用链全景)
  - [9.1 compat 路径（coding-agent 默认）](#91-compat-路径coding-agent-默认)
  - [9.2 Models 路径（AgentHarness 使用）](#92-models-路径agentharness-使用)
- [10. 设计模式分析](#10-设计模式分析)
- [11. 设计约束与权衡](#11-设计约束与权衡)
- [12. 相关源文件](#12-相关源文件)

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

## 2. Model — 纯数据抽象

**定义位置**：`packages/ai/src/types.ts:666-696`

```typescript
export interface Model<TApi extends Api> {
    id: string;                    // 如 "claude-opus-4-5"
    name: string;                 // 展示名
    api: TApi;                    // 使用的 API 协议（如 "anthropic-messages"）
    provider: ProviderId;         // 归属的 provider（如 "anthropic"）
    baseUrl: string;              // API 端点
    reasoning: boolean;           // 是否支持推理
    thinkingLevelMap?: ThinkingLevelMap;
    input: ("text" | "image")[];  // 支持的输入模态
    cost: { input, output, cacheRead, cacheWrite }; // 每百万 token 价格
    contextWindow: number;        // 上下文窗口大小
    maxTokens: number;            // 最大输出 token
    headers?: Record<string, string>;
    compat?: ...;                 // API 兼容性配置
}
```

**关键特征**：
- `Model` 没有方法，不执行任何 I/O——它是一个纯数据对象（value object）
- `api` 字段是**字符串标签**，标记"调用这个模型时应该用哪个 API 协议"
- `provider` 字段是**字符串标签**，标记"这个模型属于哪个 Provider"
- `TApi` 泛型参数在类型层面确保 `api` 字段与 `compat` 类型一致

**Model 的来源**：
- **静态模型**：代码生成在 `*.models.ts` 文件中（如 `packages/ai/src/providers/anthropic.models.ts`）
- **动态模型**：通过 `refreshModels()` 从 Provider API 运行时获取
- **自定义模型**：通过 `models.json` 或 Extension `registerProvider()` 接入

---

## 3. Provider — 运行时服务抽象

**定义位置**：`packages/ai/src/models.ts:32-72`

```typescript
export interface Provider<TApi extends Api = Api> {
    readonly id: string;                  // 如 "anthropic"
    readonly name: string;                // 如 "Anthropic"
    readonly baseUrl?: string;            // 默认 API 端点
    readonly headers?: ProviderHeaders;
    readonly auth: ProviderAuth;          // 认证策略（强制必填）

    getModels(): readonly Model<TApi>[];   // 获取模型列表（同步）
    refreshModels?(): Promise<void>;       // 动态刷新模型列表

    stream<T extends TApi>(               // 协议感知的流式请求
        model: Model<T>,
        context: Context,
        options?: ApiStreamOptions<T>,
    ): AssistantMessageEventStream;

    streamSimple(                          // 协议无关的流式请求
        model: Model<TApi>,
        context: Context,
        options?: SimpleStreamOptions,
    ): AssistantMessageEventStream;
}
```

**关键特征**：
- `Provider` 是**有行为的对象**，封装了模型目录、认证、流式请求
- `getModels()` 返回其拥有的 `Model[]`——Provider 是 Model 的容器
- `stream()` / `streamSimple()` 接受 `Model` 作为参数——Model 是 Provider 执行请求的"钥匙"
- `auth` 字段**强制必填**：即使本地 keyless 服务（如 Ollama）也必须提供 `apiKey` auth 来报告是否已配置

**一句话区别**：

| 概念 | 本质 | 类比 |
|------|------|------|
| **`Model`** | 数据对象（value object），描述一个 LLM 模型的静态属性 | 菜单上的菜品条目 |
| **`Provider`** | 运行时服务对象（runtime service），拥有模型列表、认证方式、执行流式请求的能力 | 餐厅（有菜单、有厨房、能上菜） |

---

## 4. `CreateProviderOptions` — 入参设计

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

### 各字段语义

| 字段 | 必填 | 说明 |
|------|------|------|
| `id` | ✅ | 全局唯一的 provider 标识符 |
| `name` | 否 | 展示用名，默认 fallback 到 `id` |
| `baseUrl` | 否 | 该 provider 所有模型的默认 API 端点。可在 `Model.baseUrl` 层面覆盖 |
| `headers` | 否 | 该 provider 所有请求的默认 HTTP 头 |
| `auth` | ✅ | **强制要求**。即使本地 keyless 服务（如 Ollama）也必须提供 `apiKey` auth 来报告是否已配置 |
| `models` | ✅ | 静态初始模型列表。纯动态 provider 传空数组 `[]` |
| `refreshModels` | 否 | 仅动态 provider 实现。返回新的模型列表，并发调用共享同一请求 |
| `api` | ✅ | 单一实现或按 API 分派的路由表 |

### `api` 字段的双态设计

`api` 字段接受两种形态，这是设计的核心灵活点：

**形态 1：单一 `ProviderStreams` 实例**

```ts
// 所有模型使用同一种 API 协议
api: anthropicMessagesApi()   // 类型: ProviderStreams
```

`createProvider` 内部通过 `typeof (input.api).stream === "function"` 判断是否为单一实现。

**形态 2：`Partial<Record<TApi, ProviderStreams>>` 路由表**

```ts
// 不同模型使用不同 API 协议
api: {
    "anthropic-messages": anthropicMessagesApi(),
    "openai-completions": openAICompletionsApi(),
    "openai-responses": openAIResponsesApi(),
}
```

`apiFor()` 根据 `model.api` 查表分派。若模型声明的 `api` 在路由表中无对应条目，`dispatch()` 返回一个立即以 `ModelsError("stream")` 终止的错误流——不会抛同步异常，符合"流错误编码在流内部"的契约。

---

## 5. `createProvider` 工厂函数

```ts
// 文件: packages/ai/src/models.ts:323-369
export function createProvider<TApi extends Api = Api>(
    input: CreateProviderOptions<TApi>
): Provider<TApi> {
```

### 内部状态

```ts
let models = input.models;                          // 可变模型列表（动态刷新时更新）
let inflightRefresh: Promise<void> | undefined;     // 去重：并发 refresh 共享一个 Promise
const single = typeof (input.api).stream === "function"
    ? (input.api as ProviderStreams)
    : undefined;                                    // 形态 1: 单一实现
const byApi = single ? undefined
    : (input.api as Partial<Record<string, ProviderStreams>>);  // 形态 2: 路由表
```

### `apiFor` 分派逻辑

```ts
const apiFor = (model: Model<Api>): ProviderStreams | undefined =>
    single ?? byApi?.[model.api];
```

`single` 优先——若为单一实现则无视 `model.api` 的值，所有模型走同一实现。

### `dispatch` 错误处理

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

不抛同步异常，而是通过 `lazyStream` 返回异步错误流。这是 `StreamFunction` 的核心契约：所有错误（包括配置错误）都编码在流内部。

### `refreshModels` 的去重机制

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

- 首次调用设置 `inflightRefresh`，后续并发调用返回同一个 Promise
- 成功时更新 `models` 变量
- 失败时 `inflightRefresh` 在 `finally` 中清除，下次调用会重试
- 失败时 `models` 保持上次已知状态不变

---

## 6. `Provider<TApi>` — 出参接口

### 6.1 `stream` 的泛型设计

`stream<T extends TApi>` 是 `Provider` 最复杂的泛型签名。它允许：
- 调用方传入 `Model<TApi>` 的子类型 `Model<T>`（`T extends TApi`）
- 对应的 `options` 自动解析为 `ApiStreamOptions<T>`（即具体 API 的完整选项类型）

`ApiStreamOptions<TApi>` 通过 `ApiOptionsMap` 条件类型映射到各协议的具体选项：

```
"anthropic-messages"    → AnthropicOptions      (thinkingEnabled, thinkingBudgetTokens, effort...)
"openai-responses"      → OpenAIResponsesOptions (reasoningEffort, reasoningSummary, serviceTier...)
"google-generative-ai"  → GoogleOptions          (google 特有的 thinking 参数)
"bedrock-converse-stream" → BedrockOptions       (AWS 特有的 thinking 参数)
...
```

这些类型彼此不兼容——`effort` 是 Anthropic 概念，`reasoningEffort` 是 OpenAI 概念。只有 `stream` 的调用方（知道自己用哪个协议）才能正确填充。

### 6.2 `stream` vs `streamSimple`

核心区别：**调用方是否知道自己在跟哪个 API 协议对话。**

| | `stream` | `streamSimple` |
|---|---|---|
| **调用方知识** | 知道自己用的是哪个协议 | 只知道"我要推理" |
| **options 类型** | `ApiStreamOptions<TApi>` — 协议特有参数 | `SimpleStreamOptions` — 跨协议统一 |
| **泛型** | `stream<T extends TApi>`，与 model 联动 | `streamSimple(model: Model<TApi>)`，不引入额外泛型 |
| **典型参数** | `effort`, `thinkingBudgetTokens`, `reasoningEffort`... | `reasoning: "high"` |

#### `streamSimple` 的实现：始终委托给 `stream`

`streamSimple` 从不直接调用 SDK。它是对 `stream` 的适配层，唯一职责是将统一的 `ThinkingLevel` 翻译为对应协议的参数，然后调用 `stream`。

以 Anthropic 为例（`packages/ai/src/api/anthropic-messages.ts:767-807`）：

```ts
export const streamSimple = (model, context, options) => {
    if (!options?.reasoning) {
        return stream(model, context, { thinkingEnabled: false });
    }

    if (model.compat?.forceAdaptiveThinking) {
        const effort = mapThinkingLevelToEffort(model, options.reasoning);
        return stream(model, context, { thinkingEnabled: true, effort });
    }

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

**所有 9 个已知 API 的 `streamSimple` 都遵循同一模式：先翻译参数，再调用 `stream`。** 翻译逻辑因协议而异，但对外暴露的 `SimpleStreamOptions` 完全一致。

#### 为什么需要两层

1. **知道协议细节的代码**（如 agent 的 model selector 想给 Claude 传 `effort: "xhigh"`）→ 用 `stream`，完全控制
2. **不知道协议细节的代码**（如 coding-agent 的 `--thinking` flag，用户只选了 `high`）→ 用 `streamSimple`，pi 负责把 `"high"` 翻译成协议特定参数

如果没有 `streamSimple`，每个调用方都得自己写一份 `ThinkingLevel → 协议参数` 的映射逻辑。

---

## 7. 多态维度：同一接口的不同实现

### 7.1 单一 API 提供者

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

特点：泛型 `TApi` 约束为单一已知 API 类型，所有模型共享同一个 `api` 字段值。

### 7.2 多 API 提供者（路由表模式）

**代表：GitHub Copilot（3 种 API），OpenCode Zen（4 种 API）**

```ts
// 示例: packages/ai/src/providers/github-copilot.ts
export function githubCopilotProvider():
    Provider<"anthropic-messages" | "openai-completions" | "openai-responses"> {
    return createProvider({
        id: "github-copilot",
        name: "GitHub Copilot",
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

特点：`TApi` 为联合类型，运行时 `apiFor()` 根据 `model.api` 查表分派。解决的核心问题：某些平台在不同模型背后使用不同的后端协议，但对用户应该表现为**同一个 provider**。

### 7.3 Ambient 认证提供者

**代表：Amazon Bedrock**

**Ambient 认证**：不需要用户向 pi 显式提供 API key，而是依赖运行环境中已经存在的凭据。pi 代码不持有、不传递 key，只负责检测"是否已配置"。

```ts
// packages/ai/src/providers/amazon-bedrock.ts:6-25
const bedrockAuth: ApiKeyAuth = {
    name: "AWS credentials",
    resolve: async ({ ctx, credential }) => {
        if (credential?.key) return { auth: { apiKey: credential.key }, source: "stored credential" };
        if (await ctx.env("AWS_BEARER_TOKEN_BEDROCK")) return { auth: {}, source: "AWS_BEARER_TOKEN_BEDROCK" };
        if (await ctx.env("AWS_PROFILE")) return { auth: {}, source: "AWS_PROFILE" };
        // ... 检测 ECS task role, web identity token 等
        return undefined;  // 未配置
    },
};
```

所有命中都返回 `{ auth: {} }`——空对象。因为 AWS SDK 的 credential chain 会在发出请求时自己完成 SigV4 签名。但 `resolve()` 返回 `{ auth: {} }` 仍然表示"已配置"，和返回 `undefined`（未配置）有本质区别。

**与标准 `envApiKeyAuth` 的差异**：

| | 标准 `envApiKeyAuth` | Ambient（Bedrock） |
|---|---|---|
| 凭据形式 | key 字符串 | 环境变量 / 文件 / IAM 角色 |
| pi 是否持有 key | 是，读取后注入请求头 | 否，SDK 内部签名 |
| `login()` | 有 | 无 |
| `resolve()` 返回 | `{ auth: { apiKey: "sk-..." } }` | `{ auth: {} }` |

### 7.4 自定义 ProviderEnv 认证

**代表：Cloudflare Workers AI, Cloudflare AI Gateway**

```ts
// packages/ai/src/providers/cloudflare-auth.ts
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

特点：`login()` 收集多个字段，`ApiKeyCredential.env` 承载额外配置，`resolve()` 返回的 `ModelAuth.baseUrl` 是动态的。

### 7.5 OAuth 提供者

**代表：Anthropic（Claude Pro/Max 订阅），GitHub Copilot，OpenAI Codex**

```ts
// 示例: packages/ai/src/providers/anthropic.ts
auth: {
    apiKey: envApiKeyAuth("Anthropic API key", ["ANTHROPIC_OAUTH_TOKEN", "ANTHROPIC_API_KEY"]),
    oauth: lazyOAuth({ name: "Anthropic (Claude Pro/Max)", load: loadAnthropicOAuth }),
}
```

特点：`ProviderAuth` 同时声明 `apiKey` 和 `oauth` 两种认证方式。同一 provider 支持两类用户——API key 用户（按量付费）和订阅用户（Claude Pro/Max）。`lazyOAuth()` 延迟加载 OAuth 实现，避免将所有 OAuth 代码打包进 core bundle。

### 7.6 对比总结

| 维度 | 单一 API（OpenAI） | 多 API（GitHub Copilot） | Ambient（Bedrock） | 自定义 Env（Cloudflare） | OAuth（Anthropic 订阅） |
|------|-------------------|-------------------------|-------------------|------------------------|------------------------|
| `api` 字段 | `ProviderStreams` | `Partial<Record<TApi, ProviderStreams>>` | `ProviderStreams` | `ProviderStreams` | `ProviderStreams` |
| `TApi` 泛型 | 单一类型 | 联合类型 | 单一类型 | 单一类型 | 单一类型 |
| `auth` | `{ apiKey }` | `{ apiKey, oauth }` | `{ apiKey }`（无 `login`） | `{ apiKey }`（自定义 `login`） | `{ apiKey, oauth }` |
| `baseUrl` | 静态 | 静态 | 无（SDK 内部） | 动态（占位符替换） | 静态 |
| `login()` | envApiKeyAuth 标准 | envApiKeyAuth + OAuth | 无 | 多字段 prompt | OAuth 设备码流 |
| `resolve()` 返回 | `{ apiKey: "sk-..." }` | 按 credential 类型 | `{ auth: {} }` → SDK 自签名 | `{ apiKey, baseUrl }` + `env` | OAuth `toAuth()` |

---

## 8. `Models` 集合 —— 运行时编排层

### `ModelsImpl` 的核心职责

```ts
// 文件: packages/ai/src/models.ts:142-289
class ModelsImpl implements MutableModels {
    private providers = new Map<string, Provider>();
    private credentials: CredentialStore;
    private authContext: AuthContext;
}
```

`ModelsImpl` 是 `Provider` 的运行时容器，负责：

1. **Provider 管理**：`setProvider` / `deleteProvider` / `clearProviders`
2. **模型查找**：`getModels(provider?)` / `getModel(provider, id)` —— 都是同步操作，best-effort
3. **模型刷新**：`refresh(provider?)` —— 单 provider 失败抛 `ModelsError("model_source")`，全量刷新用 `allSettled` 不抛
4. **Auth 解析**：`getAuth(model)` —— 委托给 `resolveProviderAuth()`
5. **流式调用**：`stream` / `streamSimple` / `complete` / `completeSimple`

### `applyAuth` 的合并策略

```ts
// 文件: packages/ai/src/models.ts:230-256
private async applyAuth(model, options) {
    const resolution = await resolveProviderAuth(...);
    const requestModel = auth.baseUrl ? { ...model, baseUrl: auth.baseUrl } : model;
    const apiKey = options?.apiKey ?? auth.apiKey;
    const headers = auth.headers || options?.headers
        ? { ...auth.headers, ...options?.headers } : undefined;
    const env = resolution.env || options?.env
        ? { ...(resolution.env ?? {}), ...(options?.env ?? {}) } : undefined;
}
```

合并优先级（从高到低）：`options.apiKey` > `auth.apiKey`，`options.headers` > `auth.headers`，`options.env` > `resolution.env`，`auth.baseUrl` > `model.baseUrl`。

### `MutableModels` 扩展

```ts
export interface MutableModels extends Models {
    setProvider(provider: Provider): void;
    deleteProvider(id: string): void;
    clearProviders(): void;
}
```

支持动态添加/移除 provider——`builtinProviders()` 批量注册、`models.json` 自定义 provider 注入、测试时替换。

---

## 9. 调用链全景

代码库中存在**两条流式调用路径**。

### 9.1 compat 路径（coding-agent 默认）

这是 `coding-agent` 当前实际使用的路径。入口是 `compat` 模块的 `streamSimple` 函数。

**启动时**：`coding-agent` 的 SDK 层创建 `Agent` 时覆盖 `streamFn`：

```typescript
// packages/coding-agent/src/core/sdk.ts:301-331
agent = new Agent({
    streamFn: async (model, context, options) => {
        const auth = await modelRegistry.getApiKeyAndHeaders(model);
        return streamSimple(model, context, {
            ...options,
            apiKey: auth.apiKey,
            env,
            timeoutMs, maxRetries, maxRetryDelayMs,
            headers: mergeProviderAttributionHeaders(...),
        });
    },
});
```

**调用链**：

```
Agent.streamFn (被 AgentLoop 调用)
  → compat.streamSimple(model, context, options)
    → shouldUseBuiltinModels(model)?
      ├── YES → compatModels.streamSimple(model, ...)
      │         → ModelsImpl.streamSimple(model, ...)
      │           → lazyStream(model, async () => {
      │               provider = requireProvider(model)  // 按 model.provider 查找
      │               { requestModel, requestOptions } = await applyAuth(model, options)
      │               return provider.streamSimple(requestModel, context, requestOptions)
      │             })
      │
      └── NO  → resolveApiProvider(model.api)  // 通过 apiProviderRegistry 查找
                → provider.streamSimple(model, context, withEnvApiKey(model, options))
```

**关键组件**：
- `compatModels`：一个 `ModelsImpl` 实例，包含所有内置 Provider
- `apiProviderRegistry`：一个全局 Map，按 API 类型注册 ProviderStreams（用于自定义 Provider）
- `ModelRegistry`（coding-agent 层）：封装了 compat 路径，不直接使用 `Models`/`Provider` 接口

**注意**：SDK 层的 `streamFn` 自己注入了 `apiKey`，然后调用 `compat.streamSimple`，后者内部又走 `ModelsImpl.streamSimple`，后者**又做了一次** `applyAuth`。虽然第二次不会重复注入（因为 options 中已有 apiKey），但逻辑上存在两层认证处理，职责模糊。

### 9.2 Models 路径（AgentHarness 使用）

`AgentHarness` 直接依赖 `Models` 接口，不经过 compat 层。

```typescript
// packages/agent/src/harness/agent-harness.ts:359-385
private createStreamFn(getTurnState): StreamFn {
    return async (model, context, streamOptions) => {
        const turnState = getTurnState();
        const requestOptions = await this.emitBeforeProviderRequest(model, turnState.sessionId, ...);
        return this.models.streamSimple(model, context, {
            cacheRetention: requestOptions.cacheRetention,
            headers: requestOptions.headers,
            maxRetries: requestOptions.maxRetries,
            reasoning: streamOptions?.reasoning,
            signal: streamOptions?.signal,
            sessionId: turnState.sessionId,
        });
    };
}
```

**两路径对比**：

| 维度 | compat 路径 | Models 路径 |
|------|-----------|-----------|
| 入口 | `compat.streamSimple()` | `Models.streamSimple()` |
| 使用方 | coding-agent SDK | AgentHarness |
| 认证注入 | 调用方自行注入 apiKey | `ModelsImpl.applyAuth()` 统一注入 |
| Provider 查找 | 通过 `apiProviderRegistry` 或 `compatModels` | 通过 `ModelsImpl.providers` Map |
| 状态 | 旧路径，逐步迁移 | 新路径，推荐使用 |

### 概念层次图

```
┌──────────────────────────────────────────────────────┐
│                    Agent Loop                        │
│  持有: Model<Api> + StreamFn                         │
│  调用: streamFn(model, context, options)              │
│  职责: 管理对话循环、工具调用、消息转换                 │
└────────────────────┬─────────────────────────────────┘
                     │
                     ▼
┌──────────────────────────────────────────────────────┐
│                StreamFn 实现                          │
│  compat 路径: streamSimple(model, ...)                │
│  Models 路径: models.streamSimple(model, ...)         │
│  职责: 认证注入 (apiKey/headers/env/timeout/retry)    │
└────────────────────┬─────────────────────────────────┘
                     │ model.provider → 查找 Provider
                     ▼
┌──────────────────────────────────────────────────────┐
│                 Provider<T>                           │
│  职责: 模型目录 + 认证声明 + 流式路由                   │
│  streamSimple(model, ctx, opts)                       │
│    → dispatch(model, streams)                         │
│      → model.api → 选择 ProviderStreams 实现           │
│  来源: createProvider() 工厂函数                       │
└────────────────────┬─────────────────────────────────┘
                     │ model.api → 选择 API 实现
                     ▼
┌──────────────────────────────────────────────────────┐
│            ProviderStreams (API 实现)                  │
│  一个模块对应一种 LLM 协议                              │
│  streamSimple(model, ctx, opts)                       │
│    → 翻译 SimpleStreamOptions → stream(model, ...)     │
│  stream(model, ctx, apiSpecificOpts)                  │
│    → 实际 HTTP/WebSocket 调用 + 流事件解析              │
│  返回: AssistantMessageEventStream                     │
└──────────────────────────────────────────────────────┘
```

**核心关系总结**：
- **`Model`** 是被动的数据——描述"什么模型"
- **`Provider`** 是主动的服务——回答"谁提供这个模型"以及"怎么调用它"
- **`Model.api`** 是连接 Model 和 Provider 内部 API 路由的桥梁
- **`Model.provider`** 是连接 Model 和 Provider 实例的桥梁（在 `Models` 集合中按字符串 ID 查找）
- Agent 层只持有 `Model` + `StreamFn`，不直接接触 `Provider`，实现了关注点分离
- `streamSimple` 是 Agent 层和 API 实现之间的翻译层，将"推理级别"翻译成 API 特定参数

---

## 10. 设计模式分析

### 这不是某个经典设计模式

`Provider<TApi>` + `Model<TApi>` 的结构不是 GoF 23 种模式中任何一个的直接应用，而是**三个模式的混合体，加上一个迁移中途的遗留架构**。

### 其中包含的模式

#### Strategy 模式（干净部分）

`Provider` 持有 `ProviderStreams`（即 `stream` / `streamSimple`），运行时按 `model.api` 选择策略。上下文是 Provider，策略是 API 实现。这是策略模式的标准应用。

#### Bridge 模式（API 层）

`streamSimple` → `stream` 的翻译是桥接模式：将抽象（推理级别 off/minimal/low/medium/high/xhigh）与实现（API 特定的 `thinkingBudgetTokens`、`effort`、`reasoning_effort` 等参数）分离。

#### Service Locator 的变体（别扭的根源）

`Model` 上同时挂了 `provider` 和 `api` 两个字符串标签。运行时，调用方持有 `Model` 对象，然后用 `model.provider` 查找 Provider 实例，用 `model.api` 在 Provider 内部查找 API 实现。`Model` 既是数据又是路由键——这违背了单一职责原则。

### 别扭感的来源

#### 问题一：Model 的双重身份

`Model` 应该只是一个值对象，但它同时承担了路由职责。在理想的 Strategy 模式中，客户端应该持有 `(Model, Provider)` 对，而不是让 Model 携带 Provider 的查找键。

#### 问题二：五层委托链

```
Agent
  → streamFn(model, ...)              // 第1层：streamFn 封装认证
    → Models.streamSimple(model)      // 第2层：Models 查找 Provider
      → Provider.streamSimple(model)  // 第3层：Provider dispatch 选择 API
        → ProviderStreams.streamSimple(model)  // 第4层：翻译 + 调用 stream()
          → stream(model, ...)        // 第5层：实际 HTTP 调用
```

每层都有合理的职责，但合在一起产生了过度间接化。

#### 问题三：compat 与 Models 双路径并存

`compat` 模块的注释明确说明这是迁移中途：

```typescript
// packages/ai/src/compat.ts:1-10
/**
 * Temporary compatibility entrypoint preserving the old global pi-ai API
 * surface...
 * This module is deleted with the coding-agent ModelManager migration.
 */
```

旧 compat 路径使用全局注册表模式（`apiProviderRegistry`），新 `Models` 路径使用依赖注入模式。两条路径做同一件事，增加了理解成本。

### 为什么设计成这样

**务实原因**：`Model.provider` 字符串标签让 Model 成为"可路由的值对象"，可以用作查找键、比较键、持久化键和用户输入解析。

**架构原因**：同一运行时中 30+ 个 Provider 共存，Agent 需要动态切换模型。通过让 Model 携带 Provider ID，切换模型只需替换 Model 对象，所有路由逻辑自动跟随。

**历史原因**：旧的全局注册表 API 不能一次砍掉，`compat` 模块在保持向后兼容的同时，逐步将逻辑迁移到新的 `createModels()` + `createProvider()` 架构。

### 总结

| 方面 | 实际模式 | 评价 |
|------|---------|------|
| Provider 持有 API 实现 | Strategy 模式 | 干净 |
| streamSimple → stream | Bridge 模式 | 干净 |
| Model 携带 provider/api 标签 | Service Locator 变体 | 务实但破坏单一职责 |
| 5 层委托链 | 过度间接化 | 每层职责合理但合计臃肿 |
| compat + Models 双路径 | 迁移中途 | 临时状态，compat 将移除 |

**整体定性**：一个在**迁移中途的、务实但过度分层的 Provider-Strategy 混合体**。随着 compat 模块被移除，结构会简化，但 `Model` 作为路由键的定位不会改变——那是这个架构的核心设计决策。

---

## 11. 设计约束与权衡

### 同步 `getModels()` 保证

`Provider.getModels()` 和 `Models.getModels()` 均为同步方法。调用方（UI、CLI）可以随时同步获取模型列表进行渲染。动态 provider 返回的是上次刷新结果，刷新是异步的、明确的。

### 流内部错误编码

`ProviderStreams` 的契约要求所有错误编码在流中，不抛同步异常。`createProvider` 的 `dispatch` 函数在找不到 API 实现时也遵循此契约——通过 `lazyStream` 返回一个立即以错误终止的流。

### Auth 的"未配置"语义

- `undefined` → UI 可以隐藏该 provider 或显示"请配置"
- `ModelsError("oauth")` → token 刷新失败，保留 credential 供重试
- `ModelsError("auth")` → credential store 故障

### `lazyApi` 的按需加载

所有 API 实现模块通过 `lazyApi()` 包装，延迟到首次 `stream()` 调用时才加载。加载 30+ provider 的 `builtinProviders()` 不会引入所有 SDK 的 bundle。

### `ProviderImages` 与 `Provider` 的平行设计

`ImagesProvider`（`packages/ai/src/images-models.ts`）是 `Provider` 的图像生成侧平行接口，复用 `ProviderAuth`，但有自己的 `ImagesModels` 集合和 `generateImages()` 方法（非流式）。

---

## 12. 相关源文件

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
| `packages/coding-agent/src/core/extensions/types.ts:1386` | `ProviderConfig` 接口定义 |
| `packages/coding-agent/src/core/model-registry.ts:828` | `registerProvider` 实现 |
| `packages/coding-agent/src/core/sdk.ts:301` | compat 路径的 streamFn 创建 |
| `packages/agent/src/harness/agent-harness.ts:359` | Models 路径的 streamFn 创建 |