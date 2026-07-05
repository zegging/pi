# Provider 与 Model 抽象分析

> 从应用启动和 Agent Stream 执行两个方向，对 `Provider` 和 `Model` 这两个抽象类型的定义、交互、协作、调用进行全景分析。

## 目录

- [1. 一句话核心区别](#1-一句话核心区别)
- [2. Model — 纯数据抽象](#2-model--纯数据抽象)
- [3. Provider — 运行时服务抽象](#3-provider--运行时服务抽象)
- [4. 两阶段协作：创建与运行时](#4-两阶段协作创建与运行时)
- [5. 调用链全景](#5-调用链全景)
  - [5.1 路径 A：compat 路径（coding-agent 默认使用）](#51-路径-acompat-路径coding-agent-默认使用)
  - [5.2 路径 B：Models 集合路径（AgentHarness 使用）](#52-路径-bmodels-集合路径agentharness-使用)
- [6. Provider 内部：API 路由机制](#6-provider-内部api-路由机制)
- [7. stream vs streamSimple](#7-stream-vs-streamsimple)
- [8. 概念层次图](#8-概念层次图)

---

## 1. 一句话核心区别

| 概念 | 本质 | 类比 |
|------|------|------|
| **`Model`** | 数据对象（value object），描述一个 LLM 模型的静态属性 | 菜单上的菜品条目 |
| **`Provider`** | 运行时服务对象（runtime service），拥有模型列表、认证方式、执行流式请求的能力 | 餐厅（有菜单、有厨房、能上菜） |

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
- `Model` 没有方法，不执行任何 I/O
- `api` 字段是一个**字符串标签**，标记了"调用这个模型时应该用哪个 API 协议"
- `provider` 字段是**字符串标签**，标记了"这个模型属于哪个 Provider"
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
    readonly auth: ProviderAuth;          // 认证策略

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

**Provider 的创建**：通过 `createProvider()` 工厂函数（`packages/ai/src/models.ts:323-369`），将 `CreateProviderOptions` 打包成 `Provider` 对象。内置 Provider 工厂（如 `anthropicProvider()`）和 `models.json` 自定义 Provider 都走这个函数。

---

## 4. 两阶段协作：创建与运行时

### 阶段一：创建 — Model 嵌入 Provider

每个 Provider 工厂在创建时，将 Model 列表作为静态数据传入：

```typescript
// packages/ai/src/providers/anthropic.ts
export function anthropicProvider(): Provider<"anthropic-messages"> {
    return createProvider({
        id: "anthropic",
        name: "Anthropic",
        baseUrl: "https://api.anthropic.com",
        auth: {
            apiKey: envApiKeyAuth("Anthropic API key", ["ANTHROPIC_OAUTH_TOKEN", "ANTHROPIC_API_KEY"]),
            oauth: lazyOAuth({ name: "Anthropic (Claude Pro/Max)", load: loadAnthropicOAuth }),
        },
        models: Object.values(ANTHROPIC_MODELS),  // ← Model[] 在此注入
        api: anthropicMessagesApi(),               // ← API 实现在此注入
    });
}
```

`createProvider` 内部仅仅是把这些字段打包成 `Provider` 对象：

```typescript
return {
    id, name, baseUrl, headers, auth,
    getModels: () => models,  // 闭包捕获初始（或刷新后的）Model 列表
    stream: (model, context, options) =>
        dispatch(model, (streams) => streams.stream(model, context, options)),
    streamSimple: (model, context, options) =>
        dispatch(model, (streams) => streams.streamSimple(model, context, options)),
};
```

### 阶段二：运行时 — Model 作为路由钥匙

当 Agent Loop 需要调用 LLM 时，它持有 `(model, streamFn)` 对：

```typescript
// packages/agent/src/agent-loop.ts:298-308
const streamFunction = streamFn || streamSimple;
const response = await streamFunction(config.model, llmContext, {
    ...config,
    apiKey: resolvedApiKey,
    signal,
});
```

`config.model` 是一个 `Model<any>` 对象。`streamFunction` 最终通过 `model.provider` 和 `model.api` 字段路由到正确的 Provider 和 API 实现：

- `model.provider` → 在 `Models` 集合中查找对应的 `Provider` 实例
- `model.api` → 在 `Provider` 内部的 API 路由表中选择对应的 `ProviderStreams` 实现

---

## 5. 调用链全景

代码库中存在**两条流式调用路径**。

### 5.1 路径 A：compat 路径（coding-agent 默认使用）

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
    → 判断 shouldUseBuiltinModels(model)?
      ├── YES → compatModels.streamSimple(model, ...)
      │         → ModelsImpl.streamSimple(model, ...)
      │           → lazyStream(model, async () => {
      │               provider = requireProvider(model)  // 按 model.provider 查找
      │               { requestModel, requestOptions } = await applyAuth(model, options)
      │               // applyAuth 调用 resolveProviderAuth 注入 apiKey/headers/baseUrl
      │               return provider.streamSimple(requestModel, context, requestOptions)
      │             })
      │
      └── NO  → resolveApiProvider(model.api)  // 通过 apiProviderRegistry 查找
                → provider.streamSimple(model, context, withEnvApiKey(model, options))
```

**关键组件**：
- `compatModels`：一个 `ModelsImpl` 实例，包含所有内置 Provider
- `apiProviderRegistry`：一个全局 Map，按 API 类型注册 ProviderStreams（用于自定义 Provider）
- `ModelRegistry`（coding-agent 层）：封装了 compat 路径，不直接使用 `Models`/`Provider` 接口，而是通过 `getModels()`/`getProviders()` 和 `apiProviderRegistry` 工作

### 5.2 路径 B：Models 集合路径（AgentHarness 使用）

这是 `AgentHarness` 使用的路径，**直接依赖 `Models` 接口**。

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
            // ...
            reasoning: streamOptions?.reasoning,
            signal: streamOptions?.signal,
            sessionId: turnState.sessionId,
        });
    };
}
```

**调用链**：

```
AgentHarness.createStreamFn()
  → this.models.streamSimple(model, context, options)
    → ModelsImpl.streamSimple(model, ...)
      → lazyStream(model, async () => {
          provider = this.requireProvider(model)  // 按 model.provider 查找
          { requestModel, requestOptions } = await this.applyAuth(model, options)
          return provider.streamSimple(requestModel, context, requestOptions)
        })
```

**两路径对比**：

| 维度 | compat 路径 | Models 路径 |
|------|-----------|-----------|
| 入口 | `compat.streamSimple()` | `Models.streamSimple()` |
| 使用方 | coding-agent SDK | AgentHarness |
| 认证注入 | 调用方自行注入 apiKey | `ModelsImpl.applyAuth()` 统一注入 |
| Provider 查找 | 通过 `apiProviderRegistry` 或 `compatModels` | 通过 `ModelsImpl.providers` Map |
| 状态 | 旧路径，逐步迁移 | 新路径，推荐使用 |

---

## 6. Provider 内部：API 路由机制

`createProvider` 支持两种 API 配置模式，通过 `api` 字段的双态设计实现。

### 单 API Provider

```typescript
// 示例: Anthropic, OpenAI, Google, OpenRouter 等
// 所有模型使用同一种 API 协议
api: anthropicMessagesApi()  // 类型: ProviderStreams
```

### 多 API Provider（路由表模式）

```typescript
// 示例: GitHub Copilot (同时支持 Anthropic 和 OpenAI 协议)
// packages/ai/src/providers/github-copilot.ts
api: {
    "anthropic-messages": anthropicMessagesApi(),
    "openai-completions": openAICompletionsApi(),
    "openai-responses": openAIResponsesApi(),
}
```

### 内部 dispatch 逻辑

`createProvider` 内部通过 `dispatch` 函数，按 `model.api` 选择对应的 `ProviderStreams`：

```typescript
// packages/ai/src/models.ts:327-344
const single = typeof (input.api as ProviderStreams).stream === "function"
    ? (input.api as ProviderStreams) : undefined;
const byApi = single ? undefined : (input.api as Partial<Record<string, ProviderStreams>>);

const apiFor = (model: Model<Api>): ProviderStreams | undefined =>
    single ?? byApi?.[model.api];

const dispatch = (model, run) => {
    const streams = apiFor(model);
    if (!streams) {
        // 返回一个错误流: "Provider has no API implementation for ..."
        return lazyStream(model, async () => {
            throw new ModelsError("stream", `Provider ${input.id} has no API implementation for "${model.api}"`);
        });
    }
    return run(streams);
};
```

---

## 7. stream vs streamSimple

每个 API 实现模块（如 `packages/ai/src/api/anthropic-messages.ts`）导出两个函数：

| 函数 | 选项类型 | 职责 |
|------|---------|------|
| `stream` | API 特定（如 `AnthropicOptions`） | 接受协议特定的参数，直接构建 HTTP 请求 |
| `streamSimple` | 通用（`SimpleStreamOptions`） | 将通用选项翻译成 API 特定选项，然后调用 `stream` |

**`streamSimple` 的核心逻辑**：将 Agent 层的"推理级别"（off/minimal/low/medium/high/xhigh）翻译成 API 特定的参数：

```typescript
// packages/ai/src/api/anthropic-messages.ts:767-807
export const streamSimple = (model, context, options) => {
    const base = buildBaseOptions(model, context, options, options?.apiKey);
    if (!options?.reasoning) {
        return stream(model, context, { ...base, thinkingEnabled: false });
    }
    // 把 reasoning 级别翻译成 thinkingBudgetTokens
    const adjusted = adjustMaxTokensForThinking(base.maxTokens, model.maxTokens, options.reasoning, ...);
    const maxTokens = clampMaxTokensToContext(model, context, adjusted.maxTokens);
    return stream(model, context, {
        ...base, maxTokens, thinkingEnabled: true,
        thinkingBudgetTokens: Math.min(adjusted.thinkingBudget, Math.max(0, maxTokens - 1024)),
    });
};
```

Agent 层始终使用 `streamSimple`，因为它不关心底层 API 细节——它只知道"推理级别"，由 `streamSimple` 负责翻译。

---

## 8. 概念层次图

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

## 9. 设计模式分析

### 9.1 这不是某个经典设计模式

`Provider<TApi>` + `Model<TApi>` 的结构不是 GoF 23 种模式中任何一个的直接应用，而是**三个模式的混合体，加上一个迁移中途的遗留架构**。

### 9.2 其中包含的模式

#### Strategy 模式（干净部分）

`Provider` 持有 `ProviderStreams`（即 `stream` / `streamSimple`），运行时按 `model.api` 选择策略。上下文是 Provider，策略是 API 实现。

```
Provider
  └── dispatch(model, run)
        └── model.api → 选择 ProviderStreams 实现
```

这是策略模式的标准应用，职责清晰。

#### Bridge 模式（API 层）

`streamSimple` → `stream` 的翻译是桥接模式：将抽象（推理级别 off/minimal/low/medium/high/xhigh）与实现（API 特定的 `thinkingBudgetTokens`、`effort`、`reasoning_effort` 等参数）分离。

```
Agent 层 (只关心推理级别)
  → streamSimple (统一选项)
    → stream (API 特定选项)
      → HTTP 调用
```

#### Service Locator 的变体（别扭的根源）

`Model` 上同时挂了 `provider` 和 `api` 两个字符串标签。运行时，调用方持有 `Model` 对象，然后：

1. 用 `model.provider` 去**查找** Provider 实例（类似 Service Locator）
2. 用 `model.api` 去 Provider 内部**查找** API 实现（二级查找）

```typescript
// ModelsImpl.streamSimple 内部
const provider = this.requireProvider(model);         // 按 model.provider 查找
return provider.streamSimple(requestModel, context);  // 委托给 Provider
// Provider.streamSimple 内部
return dispatch(model, (streams) => streams.streamSimple(...)); // 按 model.api 查找
```

### 9.3 别扭感的来源

#### 问题一：Model 的双重身份

`Model` 应该只是一个值对象（描述"什么模型"），但它同时承担了**路由职责**——`model.provider` 用于查找 Provider 实例，`model.api` 用于选择 API 实现。这违背了单一职责原则。

在理想的 Strategy 模式中，客户端应该持有 `(Model, Provider)` 对，而不是让 Model 携带 Provider 的查找键。Model 不需要知道自己的"provider"是谁——那是调用方的职责。

#### 问题二：五层委托链

```
Agent
  → streamFn(model, ...)              // 第1层：streamFn 封装认证
    → Models.streamSimple(model)      // 第2层：Models 查找 Provider
      → Provider.streamSimple(model)  // 第3层：Provider dispatch 选择 API
        → ProviderStreams.streamSimple(model)  // 第4层：翻译 + 调用 stream()
          → stream(model, ...)        // 第5层：实际 HTTP 调用
```

对于一个"用某个模型发请求"的操作，经过了 5 层委托。每层都有合理的职责（认证注入、Provider 查找、API 路由、选项翻译、HTTP 调用），但合在一起产生了**过度间接化**的感觉。

#### 问题三：compat 与 Models 双路径并存

`compat` 模块的注释明确说明这是**迁移中途**：

```typescript
// packages/ai/src/compat.ts:1-10
/**
 * Temporary compatibility entrypoint preserving the old global pi-ai API
 * surface...
 * This module is deleted with the coding-agent ModelManager migration.
 */
```

旧 compat 路径使用全局注册表模式（`apiProviderRegistry`），新 `Models` 路径使用依赖注入模式（`ModelsImpl` 持有 `Map<string, Provider>`）。两条路径做同一件事，增加了理解成本。

更具体地说，`coding-agent` 的 SDK 层在 `streamFn` 中自己注入了 `apiKey`、`headers`、`timeoutMs` 等（`packages/coding-agent/src/core/sdk.ts:301-331`），然后调用 `compat.streamSimple`。但 `compat.streamSimple` 内部又走 `ModelsImpl.streamSimple`，后者**又做了一次** `applyAuth`（`packages/ai/src/models.ts:278-284`）。虽然第二次不会重复注入（因为 options 中已经有 apiKey 了），但逻辑上存在两层认证处理，职责模糊。

### 9.4 为什么设计成这样

#### 务实原因：Model 需要作为路由键

`Model.provider` 字符串标签是**务实的选择**——它让 Model 成为"可路由的值对象"，可以用作：

- **查找键**：`ModelsImpl.providers.get(model.provider)` 找到 Provider
- **比较键**：`modelsAreEqual(a, b)` 比较两个模型是否相同
- **持久化键**：Session 中存储 `{ provider: "anthropic", modelId: "claude-opus-4-5" }` 来恢复模型
- **用户输入解析**：`--model sonnet` → 反向查找 `model.provider === "anthropic" && model.id.includes("sonnet")`

#### 架构原因：多 Provider 共存

同一个运行时中，30+ 个 Provider 共存，Agent 需要动态切换模型。如果每个 Agent 都直接持有 Provider 引用，模型切换会变得复杂。通过让 Model 携带 Provider ID，切换模型只需替换 Model 对象，所有路由逻辑自动跟随。

#### 历史原因：渐进式迁移

旧的全局注册表 API（`registerApiProvider`、`apiProviderRegistry`）不能一次砍掉，因为 `coding-agent` 的 `ModelRegistry` 仍然依赖它。`compat` 模块在保持向后兼容的同时，逐步将逻辑迁移到新的 `createModels()` + `createProvider()` 架构。

### 9.5 理想形态

如果去除历史包袱，一个更清晰的设计可能是：

```
Model —— 纯数据，不携带 provider/api 字符串标签
Provider —— 持有 Model[] + API 实现 + 认证
Client —— 持有 Provider，调用 provider.stream(model, ...)
```

这接近于 **Repository 模式**：Provider 是 Repository，Model 是 Entity，Client 是 Service。Model 不需要知道自己的归属——归属由谁持有它来决定。

但在这个代码库的上下文中，`Model.provider` 标签不会消失，因为它是模型发现、用户输入解析、Session 持久化等其他子系统的基础设施约定。

### 9.6 总结

| 方面 | 实际模式 | 评价 |
|------|---------|------|
| Provider 持有 API 实现 | Strategy 模式 | 干净 |
| streamSimple → stream | Bridge 模式 | 干净 |
| Model 携带 provider/api 标签 | Service Locator 变体 | 务实但破坏单一职责 |
| 5 层委托链 | 过度间接化 | 每层职责合理但合计臃肿 |
| compat + Models 双路径 | 迁移中途 | 临时状态，compat 将移除 |

**整体定性**：一个在**迁移中途的、务实但过度分层的 Provider-Strategy 混合体**。随着 compat 模块被移除，结构会简化，但 `Model` 作为路由键的定位不会改变——那是这个架构的核心设计决策。