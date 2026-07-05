# AI 流事件协议分析

本文档深入分析 `@earendil-works/pi-ai` 中流式事件（stream events）的类型定义、设计原因、Provider 映射机制以及 HTTP 传输层的关系。

## 事件类型定义位置

所有流事件类型统一在 `/packages/ai/src/types.ts` 第 453 行定义，类型名为 `AssistantMessageEvent`：

```typescript
// packages/ai/src/types.ts:453-465
export type AssistantMessageEvent =
  | { type: "start"; partial: AssistantMessage }
  | { type: "text_start"; contentIndex: number; partial: AssistantMessage }
  | { type: "text_delta"; contentIndex: number; delta: string; partial: AssistantMessage }
  | { type: "text_end"; contentIndex: number; content: string; partial: AssistantMessage }
  | { type: "thinking_start"; contentIndex: number; partial: AssistantMessage }
  | { type: "thinking_delta"; contentIndex: number; delta: string; partial: AssistantMessage }
  | { type: "thinking_end"; contentIndex: number; content: string; partial: AssistantMessage }
  | { type: "toolcall_start"; contentIndex: number; partial: AssistantMessage }
  | { type: "toolcall_delta"; contentIndex: number; delta: string; partial: AssistantMessage }
  | { type: "toolcall_end"; contentIndex: number; toolCall: ToolCall; partial: AssistantMessage }
  | { type: "done"; reason: Extract<StopReason, "stop" | "length" | "toolUse">; message: AssistantMessage }
  | { type: "error"; reason: Extract<StopReason, "aborted" | "error">; error: AssistantMessage };
```

## 为什么这样定义

### 1. 统一的跨 Provider 标准化层

不同 LLM Provider 的流式协议各不相同：

| Provider | 原生流格式 |
|----------|-----------|
| OpenAI | SSE (Server-Sent Events)，每个 chunk 是 `ChatCompletionChunk` |
| Anthropic | SSE，事件类型包括 `message_start`、`content_block_start`、`content_block_delta`、`content_block_stop`、`message_delta`、`message_stop` 等 |
| Google | Gemini 流式响应格式 |
| Mistral | 自己的 SSE 格式 |
| Bedrock | AWS Converse Stream 格式 |

Pi 的设计目标是将所有这些异构格式**归一化**为单一的事件协议，让上层消费者（agent 运行时、TUI、coding-agent CLI）无需关心底层 Provider 的差异。

### 2. 生命周期模式

每个内容块类型（text / thinking / toolCall）都遵循统一的生命周期：

```
_start → _delta (零次或多次) → _end
```

整个流本身也遵循生命周期：

```
start → (任意数量的内容块事件) → done / error
```

`done` 和 `error` 是**终止事件**——它们触发 `EventStream` 内部的 `isComplete` 检查（见 `/packages/ai/src/utils/event-stream.ts:72`），终结异步迭代并 resolve 最终的 `result()` Promise。

### 3. `contentIndex` 与事件交错

不同类型的内容块（文本、思考、工具调用）可能在同一批上游 chunk 中交错出现。文档明确说明（`/packages/ai/README.md:596`）：

> 不同内容块的流事件**不保证连续**。Provider 可能在同一个上游 chunk 中交错发送 text、thinking 和 tool call 的 delta。消费者必须使用 `contentIndex` 将每个 delta/end 事件关联到对应的块。

因此所有 `_start`、`_delta`、`_end` 事件都携带 `contentIndex`，消费者可以据此维护每个内容块的独立累加器。

## EventStream 架构

### 核心类：`/packages/ai/src/utils/event-stream.ts`

```typescript
// 泛型 EventStream 基类
export class EventStream<T, R = T> implements AsyncIterable<T> {
  private queue: T[] = [];
  private waiting: ((value: IteratorResult<T>) => void)[] = [];
  private done = false;
  private finalResultPromise: Promise<R>;
  private resolveFinalResult!: (result: R) => void;
  private isComplete: (event: T) => boolean;
  private extractResult: (event: T) => R;

  push(event: T): void { ... }           // 生产者推送事件
  async *[Symbol.asyncIterator]() { ... }  // 消费者异步迭代
  result(): Promise<R> { ... }            // 等待最终结果

  end(result?: R): void { ... }           // 强制终止流
}

// 专用子类
export class AssistantMessageEventStream extends EventStream<AssistantMessageEvent, AssistantMessage> {
  constructor() {
    super(
      (event) => event.type === "done" || event.type === "error",  // 终止条件
      (event) => {
        if (event.type === "done") return event.message;
        if (event.type === "error") return event.error;
        throw new Error("Unexpected event type for final result");
      },
    );
  }
}
```

关键设计点：
- `EventStream` 是一个**纯内存队列**，不涉及任何 HTTP I/O
- `push()` 由 Provider 实现调用，将事件推入队列
- `[Symbol.asyncIterator]()` 让消费者可以用 `for await...of` 迭代
- `result()` 返回一个 Promise，在 `done`/`error` 事件到来时 resolve——这就是 `models.complete()` 的实现方式

## Provider 映射机制

### ProviderStreams 接口

每个 Provider 必须实现 `ProviderStreams` 接口（`/packages/ai/src/types.ts:222`）：

```typescript
export interface ProviderStreams {
  stream(model: Model<Api>, context: Context, options?: StreamOptions): AssistantMessageEventStream;
  streamSimple(model: Model<Api>, context: Context, options?: SimpleStreamOptions): AssistantMessageEventStream;
}
```

### 映射表：以 Anthropic 为例

`/packages/ai/src/api/anthropic-messages.ts` 第 546 行开始，`for await` 循环将 Anthropic 原生事件映射为 Pi 规范事件：

| Anthropic 原生事件 | Pi 规范事件 | 源码行 |
|---|---|---|
| `message_start` | 更新 usage 元数据（不推事件） | 547-559 |
| `content_block_start` (type: `text`) | `text_start` | 568 |
| `content_block_start` (type: `thinking`) | `thinking_start` | 587 |
| `content_block_start` (type: `tool_use`) | `toolcall_start` | 600 |
| `content_block_delta` (type: `text_delta`) | `text_delta` | 608-613 |
| `content_block_delta` (type: `thinking_delta`) | `thinking_delta` | 620-625 |
| `content_block_delta` (type: `input_json_delta`) | `toolcall_delta` | 633-638 |
| `content_block_delta` (type: `signature_delta`) | 仅更新签名（不推事件） | 640-646 |
| `content_block_stop` (text) | `text_end` | 654-658 |
| `content_block_stop` (thinking) | `thinking_end` | 661-665 |
| `content_block_stop` (tool_use) | `toolcall_end` | 672-677 |
| `message_delta` | 更新 stop_reason 和 usage | 680-713 |
| 流正常结束 | `done` | 725 |
| 流异常/中断 | `error` | 735 |

### 映射表：以 OpenAI 为例

`/packages/ai/src/api/openai-completions.ts` 第 317 行开始，`for await (const chunk of openaiStream)` 循环将 OpenAI 的 `ChatCompletionChunk` 映射为 Pi 规范事件：

| OpenAI 原生 chunk 内容 | Pi 规范事件 | 源码行 |
|---|---|---|
| 流开始 | `start` | 198 |
| `choice.delta.content` 有值 | `text_delta` | 356-361 |
| `choice.delta.reasoning_content` 等有值 | `thinking_delta` | 380+ |
| `choice.delta.tool_calls` 有新调用 | `toolcall_start` | 300-304 |
| `choice.delta.tool_calls` 有 function.arguments | `toolcall_delta` | 类似 |
| `choice.finish_reason` 出现 | `text_end` / `thinking_end` / `toolcall_end` | 220-246 |
| 流正常结束 | `done` | 类似 725 |
| 流异常/中断 | `error` | 类似 735 |

## HTTP 传输层：一个请求，一个响应，多个事件

**关键结论：每个 Pi 事件 ≠ 一个 HTTP 请求/响应。**

一次 `models.stream()` 调用只发起**一次 HTTP 请求**，Provider 返回一个**流式 HTTP 响应**（SSE 流或 chunked transfer）。Pi 在这个单一的响应体上逐个解析 chunk，每解析到一个有意义的 chunk 就推送一个或多个 Pi 规范事件到 `EventStream` 中。

```
models.stream(model, context)
  │
  └─► 1 次 HTTP POST（带 stream: true）
       │
       └─► 1 个流式响应体（SSE chunks 持续到达）
            │
            ├─► chunk 1 到达 → push({ type: "start", ... })
            ├─► chunk 2 到达 → push({ type: "text_delta", ... })
            ├─► chunk 3 到达 → push({ type: "thinking_delta", ... })
            ├─► chunk 4 到达 → push({ type: "toolcall_start", ... })
            ├─► ...
            └─► 流结束      → push({ type: "done", ... })
```

### 源码证据

**Anthropic**（`/packages/ai/src/api/anthropic-messages.ts:539`）：
```typescript
// 一次 HTTP 请求
const response = await client.messages.create({ ...params, stream: true }, requestOptions).asResponse();

stream.push({ type: "start", partial: output });

// 在同一个响应体上迭代
for await (const event of iterateAnthropicEvents(response, options?.signal)) {
    // 每个 Anthropic 原生事件 → 1 个 Pi 规范事件
    if (event.type === "content_block_delta") {
        stream.push({ type: "text_delta", ... });
    }
}
```

**OpenAI**（`/packages/ai/src/api/openai-completions.ts:194-317`）：
```typescript
// 一次 HTTP 请求
const { data: openaiStream, response } = await client.chat.completions
    .create(params, requestOptions).withResponse();

stream.push({ type: "start", partial: output });

// 在同一个 SSE 流上迭代
for await (const chunk of openaiStream) {
    // 每个 ChatCompletionChunk → 1 个或多个 Pi 规范事件
    if (choice.delta.content) {
        stream.push({ type: "text_delta", ... });
    }
}
```

### 事件类型与 HTTP 的对应关系

| Pi 事件 | HTTP 层面的含义 |
|---------|----------------|
| `start` | HTTP 响应头到达后**立即推送一次**，表示流已建立。不是独立的 HTTP 请求。 |
| `text_start` / `thinking_start` / `toolcall_start` | Provider 的 SSE 流中某个内容块开始的信号。 |
| `text_delta` / `thinking_delta` / `toolcall_delta` | 每个 SSE chunk 中携带的增量内容。OpenAI 场景下就是字面意义上的一个 `ChatCompletionChunk`。 |
| `text_end` / `thinking_end` / `toolcall_end` | Provider 信号该内容块已完成。 |
| `done` | 流式响应体**完全消费完毕**后推送。标志着单次 HTTP 事务的结束。 |
| `error` | HTTP 请求失败、流被中断、或 Provider 返回错误。也是终止事件。 |

## LazyStream 包装层

`/packages/ai/src/api/lazy.ts` 提供了 `lazyStream` 函数，用于处理异步初始化（auth 解析、懒加载模块）：

```typescript
export function lazyStream(
  model: Model<Api>,
  setup: () => Promise<AsyncIterable<AssistantMessageEvent>>,
): AssistantMessageEventStream {
  const outer = new AssistantMessageEventStream();
  setup()
    .then((inner) => forwardStream(outer, inner))  // 将内部流事件转发到外部流
    .catch((error) => {
      // 初始化失败直接推送 error 事件
      outer.push({ type: "error", reason: "error", error: message });
      outer.end(message);
    });
  return outer;
}
```

这确保了 `Models.stream()` 可以**同步返回**一个 `AssistantMessageEventStream`，而异步的 auth 解析和模块加载在后台进行。如果初始化失败，消费者会收到一个 `error` 事件。

## Faux Provider 的模拟实现

`/packages/ai/src/providers/faux.ts` 的 `streamWithDeltas` 函数（第 308 行）展示了即使在**无网络**的测试场景下，事件协议也保持一致：

```typescript
stream.push({ type: "start", partial: { ...partial } });

for (let index = 0; index < message.content.length; index++) {
  if (block.type === "thinking") {
    stream.push({ type: "thinking_start", contentIndex: index, ... });
    for (const chunk of splitStringByTokenSize(...)) {
      stream.push({ type: "thinking_delta", contentIndex: index, delta: chunk, ... });
    }
    stream.push({ type: "thinking_end", contentIndex: index, ... });
  }
  // ... 类似处理 text 和 toolCall
}
stream.push({ type: "done", reason: message.stopReason, message });
```

## 相关源码文件

| 文件 | 职责 |
|------|------|
| `/packages/ai/src/types.ts:453-465` | `AssistantMessageEvent` 类型定义 |
| `/packages/ai/src/utils/event-stream.ts` | `EventStream` 和 `AssistantMessageEventStream` 类 |
| `/packages/ai/src/api/lazy.ts` | `lazyStream` 包装器，处理异步初始化 |
| `/packages/ai/src/models.ts:258-267` | `Models.stream()` 实现，auth 解析 + 委托 Provider |
| `/packages/ai/src/api/anthropic-messages.ts:468-741` | Anthropic 完整映射实现 |
| `/packages/ai/src/api/openai-completions.ts:152-488` | OpenAI Completions 完整映射实现 |
| `/packages/ai/src/providers/faux.ts:308-401` | 测试用 Faux Provider 的事件模拟 |
| `/packages/ai/README.md:577-596` | 官方事件参考文档 |