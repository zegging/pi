# 扩展系统：自定义 Provider

> 如何在 pi 中接入自定义 LLM provider——从声明式配置到完整流式协议实现。

## 两种接入方式

pi 提供两种方式注册自定义 provider，不需要修改 `packages/ai` 源码：

| | `models.json` | Extension `registerProvider()` |
|---|---|---|
| 需要写代码 | 否 | 是 |
| 自定义流式协议 | 否（只能用内置 API） | 是（`streamSimple`） |
| 自定义 API 标识符 | 否 | 是 |
| OAuth 支持 | 否 | 是 |
| 多后端模型路由 | 否 | 是 |
| 安装/分发 | 文件复制 | Extension 包管理 |

---

## 方式一：声明式 `models.json`（零代码）

pi 启动时从 `~/.pi/agent/models.json` 和 `<project>/.pi/models.json` 加载自定义 provider。

```json
{
  "my-proxy": {
    "baseUrl": "https://api.example.com/v1",
    "apiKey": "$MY_API_KEY",
    "api": "openai-completions",
    "models": [
      {
        "id": "gpt-4o",
        "name": "GPT-4o (proxy)",
        "reasoning": true,
        "input": ["text", "image"],
        "cost": { "input": 2.5, "output": 10, "cacheRead": 0, "cacheWrite": 0 },
        "contextWindow": 128000,
        "maxTokens": 16384
      }
    ]
  }
}
```

关键约束：
- `apiKey` 支持 `$ENV_VAR` 和 `${ENV_VAR}` 语法引用环境变量
- `api` 必须是内置已知 API 之一（如 `"openai-completions"`、`"anthropic-messages"`），由 `packages/ai` 提供流式实现
- 不需要写代码，适合纯代理/转发场景

**也可以仅覆盖已有 provider 的 URL**，不定义新模型：

```json
{
  "anthropic": {
    "baseUrl": "https://my-proxy.example.com"
  }
}
```

此时 Anthropic 的所有内置模型继续可用，但 API 请求发往代理地址。

---

## 方式二：Extension 调用 `pi.registerProvider()`

### 入口

Extension 的 factory 函数接收 `ExtensionAPI` 对象，调用其 `registerProvider(name, config)` 方法：

```ts
export default function (pi: ExtensionAPI) {
  pi.registerProvider("my-provider", {
    // ProviderConfig
  });
}
```

### `ProviderConfig` 接口

定义在 `packages/coding-agent/src/core/extensions/types.ts:1386`：

```ts
interface ProviderConfig {
  /** 展示名称 */
  name?: string;
  /** API 端点 URL */
  baseUrl?: string;
  /** API key 字面量或 "$ENV_VAR" 引用 */
  apiKey?: string;
  /** API 类型（已知 API 或自定义字符串） */
  api?: Api;
  /** 自定义 HTTP 头 */
  headers?: Record<string, string>;
  /** 自动添加 Authorization: Bearer 头 */
  authHeader?: boolean;
  /** 模型列表（提供则替换该 provider 的所有已有模型） */
  models?: ProviderModelConfig[];
  /** 自定义流式实现 */
  streamSimple?: (
    model: Model<Api>,
    context: Context,
    options?: SimpleStreamOptions,
  ) => AssistantMessageEventStream;
  /** OAuth 支持 */
  oauth?: {
    name: string;
    login(callbacks: OAuthLoginCallbacks): Promise<OAuthCredentials>;
    refreshToken(credentials: OAuthCredentials): Promise<OAuthCredentials>;
    getApiKey(credentials: OAuthCredentials): string;
    modifyModels?(models: Model<Api>[], credentials: OAuthCredentials): Model<Api>[];
  };
}
```

### `registerProvider` 的三种行为

实现位于 `packages/coding-agent/src/core/model-registry.ts:828`：

1. **有 `models`**：替换该 provider 的所有已有模型，全新注册。要求同时提供 `baseUrl` 和 `apiKey`（或 `oauth`）
2. **仅有 `baseUrl`/`headers`**：覆盖现有模型的 URL，不改模型列表
3. **有 `oauth`**：注册 OAuth provider，支持 `/login` 交互

---

## 示例一：从零实现 Anthropic 流式协议

**源码**：`packages/coding-agent/examples/extensions/custom-provider-anthropic/index.ts`（~600 行）

这个示例展示了最完整的自定义 provider 实现——自定义 API 标识符、完整的 SSE 流式解析、OAuth，以及 Claude Code stealth mode。

### 注册

```ts
export default function (pi: ExtensionAPI) {
  pi.registerProvider("custom-anthropic", {
    baseUrl: "https://api.anthropic.com",
    apiKey: "$CUSTOM_ANTHROPIC_API_KEY",
    api: "custom-anthropic-api",  // 自定义 API 标识符
    models: [ /* ... */ ],
    oauth: {
      name: "Custom Anthropic (Claude Pro/Max)",
      login: loginAnthropic,
      refreshToken: refreshAnthropicToken,
      getApiKey: (cred) => cred.access,
    },
    streamSimple: streamCustomAnthropic,
  });
}
```

### `streamSimple` 实现要点

签名遵循 `SimpleStreamOptions` 接口：

```ts
function streamCustomAnthropic(
  model: Model<Api>,
  context: Context,
  options?: SimpleStreamOptions,
): AssistantMessageEventStream
```

实现流程：

1. 创建 `AssistantMessageEventStream`（通过 `createAssistantMessageEventStream()`）
2. 异步启动，通过 `stream.push()` 推送事件
3. 从 `options?.apiKey` 获取已解析的 API key
4. 根据 key 类型（OAuth vs API key）配置不同的 Anthropic client
5. 调用 `client.messages.stream()` 获取 SSE 流
6. 遍历 SSE 事件，映射为 pi 的标准事件类型：
   - `message_start` → `{ type: "start" }`
   - `content_block_start`（text/thinking/tool_use）→ `{ type: "text_start" | "thinking_start" | "toolcall_start" }`
   - `content_block_delta` → `{ type: "text_delta" | "thinking_delta" | "toolcall_delta" }`
   - `content_block_stop` → `{ type: "text_end" | "thinking_end" | "toolcall_end" }`
   - `message_delta` → 更新 usage + stop reason
7. 异常时推送 `{ type: "error" }` 事件

### OAuth 特殊处理

OAuth token（`sk-ant-oat` 前缀）需要：

- 使用 `authToken` 而非 `apiKey` 初始化 Anthropic client
- 注入 `claude-code-20250219` 和 `oauth-2025-04-20` beta 头
- 工具名映射为 Claude Code 工具名（`Read`/`Write`/`Bash`/`Grep` 等），绕过订阅限制
- 系统提示前缀 `"You are Claude Code, Anthropic's official CLI for Claude."`

---

## 示例二：GitLab Duo——复用内置 API 实现

**源码**：`packages/coding-agent/examples/extensions/custom-provider-gitlab-duo/index.ts`（~400 行）

这个示例展示了**不需要自己实现流式协议**的场景——直接复用 pi 内置的 `anthropicMessagesApi()` 和 `openAIResponsesApi()`。

### 注册

```ts
export default function (pi: ExtensionAPI) {
  pi.registerProvider("gitlab-duo", {
    baseUrl: "https://cloud.gitlab.com",
    apiKey: "$GITLAB_TOKEN",
    api: "gitlab-duo-api",
    models: MODELS.map(({ id, name, reasoning, ... }) => ({ id, name, reasoning, ... })),
    oauth: {
      name: "GitLab Duo",
      login: loginGitLab,
      refreshToken: refreshGitLabToken,
      getApiKey: (cred) => cred.access,
    },
    streamSimple: streamGitLabDuo,
  });
}
```

### `streamSimple` 实现要点

核心思路：**作为代理层，在调用内置 API 之前注入 GitLab 的 direct access token 和 headers**。

```ts
function streamGitLabDuo(model, context, options) {
  // 1. 获取 GitLab direct access token
  const directAccess = await getDirectAccessToken(gitlabAccessToken);

  // 2. 根据模型后端选择内置 API 实现
  const innerStream = cfg.backend === "anthropic"
    ? anthropicMessagesApi().streamSimple(modelWithBaseUrl, context, streamOptions)
    : openAIResponsesApi().streamSimple(modelWithBaseUrl, context, streamOptions);

  // 3. 转发内置流的所有事件
  for await (const event of innerStream) stream.push(event);
}
```

关键点：
- `getDirectAccessToken()` 通过 GitLab API 换取短期 token（`DIRECT_ACCESS_TTL = 25min`），使用内存缓存
- 模型定义包含 `backend` 字段（`"anthropic"` | `"openai"`），运行时决定走哪条内置 API 路径
- 不需要解析 SSE 事件、不需要处理 thinking/tool_call 映射——所有这些由内置 API 实现完成
- 只需要做 token 注入和 URL 路由

### 与示例一的对比

| | 示例一（custom-anthropic） | 示例二（gitlab-duo） |
|---|---|---|
| 流式实现 | 自己实现完整 SSE 解析 | 复用内置 API 实现 |
| 代码量 | ~600 行 | ~400 行 |
| 适用场景 | 非标准 API 协议 | 标准 API 协议 + 自定义认证层 |
| 关键复杂度 | 事件映射、OAuth stealth mode | Direct access token 缓存、多后端路由 |

---

## `ProviderModelConfig` 接口

```ts
interface ProviderModelConfig {
  id: string;              // 模型 ID
  name: string;            // 展示名称
  api?: Api;               // 覆盖 provider 级别的 api
  baseUrl?: string;        // 覆盖 provider 级别的 baseUrl
  reasoning: boolean;      // 是否支持 thinking
  thinkingLevelMap?: Model<Api>["thinkingLevelMap"];
  input: ("text" | "image")[];
  cost: { input: number; output: number; cacheRead: number; cacheWrite: number };
  contextWindow: number;
  maxTokens: number;
  headers?: Record<string, string>;
  compat?: Model<Api>["compat"];
}
```

---

## 相关源码

| 文件 | 说明 |
|------|------|
| `packages/coding-agent/src/core/extensions/types.ts:1386` | `ProviderConfig` 接口定义 |
| `packages/coding-agent/src/core/extensions/loader.ts:197` | `registerProvider` 在 ExtensionAPI 上的绑定 |
| `packages/coding-agent/src/core/extensions/runner.ts:365` | Provider 注册队列的刷新 |
| `packages/coding-agent/src/core/model-registry.ts:828` | `registerProvider` 的实际实现 |
| `packages/coding-agent/examples/extensions/custom-provider-anthropic/` | 示例一：完整自定义流式协议 |
| `packages/coding-agent/examples/extensions/custom-provider-gitlab-duo/` | 示例二：复用内置 API 实现 |