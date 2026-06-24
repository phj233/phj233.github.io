---
title: 适配 OpenAI 与 Anthropic 的 Response 协议
date: 2026-05-28 14:00:00
categories:
- Program
tags:
- AI
- OpenAI
- Anthropic
- API
- 架构
---

当你的应用需要同时接入 OpenAI 和 Anthropic 两家 AI 服务商时，你会立刻碰到一个现实问题：两家的**请求格式不同**、**响应结构不同**、**流式协议不同**、**错误格式不同**。如果每一处业务代码都直接写两家分支，很快会变成维护噩梦。

这篇文章梳理两家的协议差异，并给出一个统一的适配层设计。

## 一、两家协议的核心差异

### 1.1 请求格式

**OpenAI Chat Completions：**

```json
{
  "model": "gpt-4o",
  "messages": [
    { "role": "system", "content": "You are a helpful assistant." },
    { "role": "user", "content": "Hello" }
  ],
  "stream": true,
  "temperature": 0.7,
  "max_tokens": 4096
}
```

**Anthropic Messages：**

```json
{
  "model": "claude-sonnet-4-20250514",
  "system": "You are a helpful assistant.",
  "messages": [
    { "role": "user", "content": "Hello" }
  ],
  "stream": true,
  "temperature": 0.7,
  "max_tokens": 4096
}
```

差异：
- OpenAI 把 `system` 放在 `messages` 数组里（`role: "system"`）
- Anthropic 把 `system` 作为**顶层字段**，且值可以是 `string` 或 `array`
- Anthropic 不支持 `developer` 角色（OpenAI o 系列用）
- Anthropic 的参数命名是 `max_tokens`（snake_case），OpenAI 也是 `max_tokens`

### 1.2 响应格式（非流式）

**OpenAI：**

```json
{
  "id": "chatcmpl-xxx",
  "object": "chat.completion",
  "created": 1710000000,
  "model": "gpt-4o",
  "usage": {
    "prompt_tokens": 10,
    "completion_tokens": 50,
    "total_tokens": 60
  },
  "choices": [
    {
      "index": 0,
      "message": {
        "role": "assistant",
        "content": "Hello! How can I help you today?"
      },
      "finish_reason": "stop"
    }
  ]
}
```

**Anthropic：**

```json
{
  "id": "msg_xxx",
  "type": "message",
  "role": "assistant",
  "model": "claude-sonnet-4-20250514",
  "content": [
    {
      "type": "text",
      "text": "Hello! How can I help you today?"
    }
  ],
  "stop_reason": "end_turn",
  "stop_sequence": null,
  "usage": {
    "input_tokens": 10,
    "output_tokens": 50
  }
}
```

核心差异：

| 维度 | OpenAI | Anthropic |
|------|--------|-----------|
| 顶层结构 | `choices[0].message.content` | `content[0].text` |
| content 类型 | 纯字符串 | **数组**（`[{type: "text", text: "..."}]`） |
| finish 原因 | `choices[0].finish_reason`（`stop`/`length`/`tool_calls`/`content_filter`） | `stop_reason`（`end_turn`/`max_tokens`/`stop_sequence`/`tool_use`） |
| usage 字段名 | `prompt_tokens` / `completion_tokens` | `input_tokens` / `output_tokens` |
| usage 包含 | 含 `total_tokens` | **不含** `total_tokens`（需自己算） |
| 顶层 role | 嵌套在 `message.role` 里 | 顶层 `role` 字段 |

### 1.3 流式 SSE 格式

**OpenAI SSE：**

```
data: {"id":"chatcmpl-xxx","object":"chat.completion.chunk","choices":[{"index":0,"delta":{"content":"Hello"},"finish_reason":null}]}

data: {"id":"chatcmpl-xxx","object":"chat.completion.chunk","choices":[{"index":0,"delta":{"content":"!"},"finish_reason":null}]}

data: {"id":"chatcmpl-xxx","object":"chat.completion.chunk","choices":[{"index":0,"delta":{},"finish_reason":"stop"}]}

data: [DONE]
```

**Anthropic SSE：**

```
event: message_start
data: {"type":"message_start","message":{"id":"msg_xxx",...}}

event: content_block_start
data: {"type":"content_block_start","index":0,"content_block":{"type":"text","text":""}}

event: content_block_delta
data: {"type":"content_block_delta","index":0,"delta":{"type":"text_delta","text":"Hello"}}

event: content_block_delta
data: {"type":"content_block_delta","index":0,"delta":{"type":"text_delta","text":"!"}}

event: content_block_stop
data: {"type":"content_block_stop","index":0}

event: message_delta
data: {"type":"message_delta","delta":{"stop_reason":"end_turn"},"usage":{"output_tokens":50}}

event: message_stop
data: {"type":"message_stop"}
```

核心差异：

| 维度 | OpenAI | Anthropic |
|------|--------|-----------|
| 事件粒度 | 只有 `data:` 行（隐式 `message` 事件） | 多种结构化事件（`message_start`/`content_block_start`/`content_block_delta`/`message_delta`/`message_stop`） |
| delta 路径 | `choices[0].delta.content` | `delta.text`（在 `content_block_delta` 事件中） |
| usage 传递 | 在最后一个 chunk 中返回 | 在 `message_delta` 事件中返回 |
| 结束标记 | `[DONE]` | `message_stop` 事件 |
| content_block | 不需要（只有一种内容） | 支持多种 content_block（text / tool_use / tool_result） |

### 1.4 多模态（图片）Content 格式

**OpenAI Vision：**

```json
{
  "role": "user",
  "content": [
    { "type": "text", "text": "What's in this image?" },
    {
      "type": "image_url",
      "image_url": {
        "url": "data:image/jpeg;base64,/9j/4AAQ..."
      }
    }
  ]
}
```

**Anthropic Vision：**

```json
{
  "role": "user",
  "content": [
    { "type": "text", "text": "What's in this image?" },
    {
      "type": "image",
      "source": {
        "type": "base64",
        "media_type": "image/jpeg",
        "data": "/9j/4AAQ..."
      }
    }
  ]
}
```

差异：
- OpenAI 用 `{"type": "image_url", "image_url": {"url": "..."}}`，Anthropic 用 `{"type": "image", "source": {"type": "base64", "media_type": "...", "data": "..."}}`
- Anthropic 明确标注 `media_type`，OpenAI 从 data URI 自动推断

### 1.5 错误格式

**OpenAI：**

```json
{
  "error": {
    "message": "You exceeded your current quota",
    "type": "insufficient_quota",
    "code": "insufficient_quota"
  }
}
```

**Anthropic：**

```json
{
  "type": "error",
  "error": {
    "type": "rate_limit_error",
    "message": "This request has been rate limited"
  }
}
```

差异：
- OpenAI 错误在 `error.message`，Anthropic 在 `error.error.message`
- OpenAI 的 `error.type` 和 Anthropic 的 `error.error.type` 错误类型名不同
- Anthropic 多了一层 `"type": "error"` 的外层标识

---

## 二、统一适配层设计

我们的目标是：**上层业务代码只跟一个统一接口打交道，不用关心底层是哪家供应商**。

### 2.1 统一请求模型

```typescript
// 统一的消息格式
interface UnifiedMessage {
  role: 'system' | 'user' | 'assistant'
  content: string | UnifiedContentPart[]
}

interface UnifiedContentPart {
  type: 'text' | 'image'
  text?: string
  imageUrl?: string           // base64 data URI 或 HTTPS URL
  imageMediaType?: string     // image/jpeg, image/png ...
}

// 统一的请求参数
interface UnifiedChatRequest {
  model: string
  messages: UnifiedMessage[]
  stream?: boolean
  temperature?: number
  maxTokens?: number
}

// 统一的响应
interface UnifiedChatResponse {
  id: string
  model: string
  content: string
  finishReason: string
  usage: {
    inputTokens: number
    outputTokens: number
    totalTokens: number
  }
}
```

### 2.2 请求转换层

```typescript
class OpenAIAdapter {
  buildRequest(req: UnifiedChatRequest): Record<string, unknown> {
    const messages = req.messages.map(msg => ({
      role: msg.role,
      content: typeof msg.content === 'string'
        ? msg.content
        : msg.content.map(part =>
            part.type === 'text'
              ? { type: 'text', text: part.text }
              : { type: 'image_url',
                  image_url: { url: part.imageUrl, detail: 'auto' }
                }
          ),
    }))

    return {
      model: req.model,
      messages,
      stream: req.stream ?? false,
      ...(req.temperature !== undefined && { temperature: req.temperature }),
      ...(req.maxTokens !== undefined && { max_tokens: req.maxTokens }),
    }
  }

  parseResponse(body: Record<string, unknown>): UnifiedChatResponse {
    const choice = (body.choices as Array<Record<string, unknown>>)?.[0]
    const message = choice?.message as Record<string, unknown>
    const usage = body.usage as Record<string, number>

    return {
      id: body.id as string,
      model: body.model as string,
      content: (message?.content as string) || '',
      finishReason: (choice?.finish_reason as string) || 'unknown',
      usage: {
        inputTokens: usage?.prompt_tokens || 0,
        outputTokens: usage?.completion_tokens || 0,
        totalTokens: usage?.total_tokens || 0,
      },
    }
  }

  parseError(body: Record<string, unknown>): string {
    const error = body.error as Record<string, string>
    return error?.message || 'OpenAI API error'
  }
}
```

```typescript
class AnthropicAdapter {
  buildRequest(req: UnifiedChatRequest): Record<string, unknown> {
    // 提取 system 消息
    const systemMessages = req.messages.filter(m => m.role === 'system')
    const system = systemMessages.length
      ? systemMessages.map(m =>
          typeof m.content === 'string' ? m.content : ''
        ).join('\n')
      : undefined

    // 其他消息放在 messages 数组中
    const messages = req.messages
      .filter(m => m.role !== 'system')
      .map(msg => ({
        role: msg.role,
        content: typeof msg.content === 'string'
          ? msg.content
          : msg.content.map(part =>
              part.type === 'text'
                ? { type: 'text', text: part.text }
                : {
                    type: 'image',
                    source: {
                      type: 'base64',
                      media_type: part.imageMediaType || 'image/jpeg',
                      data: part.imageUrl?.startsWith('data:')
                        ? part.imageUrl.split(',')[1]
                        : part.imageUrl,
                    },
                  }
            ),
      }))

    const body: Record<string, unknown> = {
      model: req.model,
      messages,
      stream: req.stream ?? false,
      max_tokens: req.maxTokens || 4096,    // Anthropic 必填
    }
    if (system) body.system = system
    if (req.temperature !== undefined) body.temperature = req.temperature

    return body
  }

  parseResponse(body: Record<string, unknown>): UnifiedChatResponse {
    const content = body.content as Array<Record<string, unknown>>
    const text = content
      ?.filter((block: Record<string, unknown>) => block.type === 'text')
      .map((block: Record<string, unknown>) => block.text as string)
      .join('') || ''

    const usage = body.usage as Record<string, number>

    return {
      id: body.id as string,
      model: body.model as string,
      content: text,
      finishReason: body.stop_reason as string,
      usage: {
        inputTokens: usage?.input_tokens || 0,
        outputTokens: usage?.output_tokens || 0,
        totalTokens: (usage?.input_tokens || 0) + (usage?.output_tokens || 0),
      },
    }
  }

  parseError(body: Record<string, unknown>): string {
    const error = body.error as Record<string, Record<string, string>>
    return error?.error?.message || 'Anthropic API error'
  }
}
```

### 2.3 流式适配层

流式适配是最复杂的部分——两家的 SSE 事件模型完全不同：

```typescript
interface StreamCallbacks {
  onContent: (delta: string) => void
  onReasoning?: (delta: string) => void
  onToolUse?: (name: string, input: string) => void
  onComplete: (usage: { inputTokens: number; outputTokens: number }) => void
  onError: (error: string) => void
}

class OpenAIStreamParser {
  parseLine(line: string, callbacks: StreamCallbacks): boolean {
    if (!line.startsWith('data:')) return true
    const data = line.slice('data:'.length).trim()
    if (!data || data === '[DONE]') {
      callbacks.onComplete({ inputTokens: 0, outputTokens: 0 })
      return false
    }

    const chunk = JSON.parse(data)
    const choice = chunk.choices?.[0]
    const delta = choice?.delta

    if (delta?.content) {
      callbacks.onContent(delta.content)
    }
    if (delta?.reasoning_content) {
      callbacks.onReasoning?.(delta.reasoning_content)
    }
    if (delta?.tool_calls) {
      for (const tc of delta.tool_calls) {
        const fn = tc.function
        if (fn?.name) callbacks.onToolUse?.(fn.name, '')
        if (fn?.arguments) callbacks.onToolUse?.(tc.id || fn.name, fn.arguments)
      }
    }
    // usage 在流式模式下通常出现在最后一个非 [DONE] 的 chunk
    if (chunk.usage) {
      callbacks.onComplete({
        inputTokens: chunk.usage.prompt_tokens,
        outputTokens: chunk.usage.completion_tokens,
      })
      return false
    }
    return true
  }
}
```

```typescript
class AnthropicStreamParser {
  private currentEvent = ''
  private textBuffer = ''
  private usageBuffer: { inputTokens: number; outputTokens: number } = {
    inputTokens: 0, outputTokens: 0
  }

  parseLine(raw: string, callbacks: StreamCallbacks): boolean {
    const line = raw.trim()
    if (!line) { this.currentEvent = ''; return true }

    if (line.startsWith('event:')) {
      this.currentEvent = line.slice('event:'.length).trim()
      return true
    }

    if (!line.startsWith('data:')) return true
    const data = line.slice('data:'.length).trim()
    if (!data) return true

    const payload = JSON.parse(data)

    switch (this.currentEvent) {
      case 'content_block_delta': {
        const delta = payload.delta
        if (delta?.type === 'text_delta') {
          this.textBuffer += delta.text
          callbacks.onContent(delta.text)
        } else if (delta?.type === 'input_json_delta') {
          callbacks.onToolUse?.(payload.index?.toString() || '', delta.partial_json)
        } else if (delta?.type === 'thinking_delta') {
          callbacks.onReasoning?.(delta.thinking)
        }
        break
      }

      case 'message_start':
        this.usageBuffer.inputTokens = payload.message?.usage?.input_tokens || 0
        break

      case 'message_delta':
        this.usageBuffer.outputTokens = payload.usage?.output_tokens || 0
        // stop_reason 出现，准备结束
        break

      case 'message_stop':
        callbacks.onComplete(this.usageBuffer)
        return false

      case 'error':
        callbacks.onError(payload.error?.message || 'Anthropic stream error')
        return false
    }

    return true
  }
}
```

### 2.4 统一流式调用入口

```typescript
async function* streamChat(
  adapter: 'openai' | 'anthropic',
  req: UnifiedChatRequest,
  apiKey: string,
): AsyncGenerator<{
  type: 'content' | 'reasoning' | 'tool' | 'complete' | 'error'
  data: string | { inputTokens: number; outputTokens: number }
}> {
  const adapterInstance = adapter === 'openai'
    ? new OpenAIAdapter()
    : new AnthropicAdapter()

  const endpoint = adapter === 'openai'
    ? 'https://api.openai.com/v1/chat/completions'
    : 'https://api.anthropic.com/v1/messages'

  const body = adapterInstance.buildRequest(req)

  const response = await fetch(endpoint, {
    method: 'POST',
    headers: {
      'Content-Type': 'application/json',
      'x-api-key': apiKey,         // Anthropic 用 x-api-key
      'Authorization': `Bearer ${apiKey}`, // OpenAI 用 Bearer
      'anthropic-version': adapter === 'anthropic'
        ? '2023-06-01' : undefined,
    },
    body: JSON.stringify(body),
  })

  if (!response.ok) {
    const errBody = await response.json()
    yield { type: 'error', data: adapterInstance.parseError(errBody) }
    return
  }

  if (!response.body) {
    throw new Error('Streaming not supported')
  }

  const parser = adapter === 'openai'
    ? new OpenAIStreamParser()
    : new AnthropicStreamParser()

  const reader = response.body.getReader()
  const decoder = new TextDecoder()
  let remainder = ''

  while (true) {
    const { value, done } = await reader.read()
    if (done) break

    remainder += decoder.decode(value, { stream: true })
    const lines = remainder.split('\n')
    remainder = lines.pop() || ''

    for (const line of lines) {
      parser.parseLine(line, {
        onContent: (delta) => { /* yield { type: 'content', data: delta } */ },
        onReasoning: (delta) => { /* yield { type: 'reasoning', data: delta } */ },
        onComplete: (usage) => { /* yield { type: 'complete', data: usage } */ },
        onError: (err) => { /* yield { type: 'error', data: err } */ },
      })
    }
  }
}
```

---

## 三、Tool Use 差异

两家都支持 function calling，但实现上有重要差异：

**OpenAI：**
- 请求中声明 `tools` 数组
- 模型返回 `delta.tool_calls` 增量 JSON
- 必须收集拼接完整的 `function.arguments` 再解析
- `tool_choice` 支持 `auto` / `none` / `required` / 指定工具

**Anthropic：**
- 请求中声明 `tools` 数组（结构类似但字段略有不同）
- 模型返回独立的 `content_block`（`type: tool_use`）
- 在 `content_block_start` 事件中拿到 `tool_use.name`，在 `content_block_delta` 中拿到 `input_json_delta`
- **需要前端/中间层缓存 tool_use id**，因为工具结果必须以 `tool_result` content_block 形式传回
- Anthropic 要求 tool_use 和 tool_result 必须交替出现，不能连续两个 tool_use

```typescript
// Anthropic tool_use 处理
case 'content_block_start': {
  if (payload.content_block?.type === 'tool_use') {
    this.activeToolName = payload.content_block.name
    this.activeToolId = payload.content_block.id
  }
  break
}

case 'content_block_delta': {
  if (payload.delta?.type === 'input_json_delta') {
    this.toolInputBuffer += payload.delta.partial_json
  }
  break
}

case 'content_block_stop': {
  if (this.activeToolId) {
    callbacks.onToolUse?.(this.activeToolName, this.toolInputBuffer)
    this.activeToolName = ''
    this.activeToolId = ''
    this.toolInputBuffer = ''
  }
  break
}
```

---

## 四、适配层的设计原则

1. **外部协议向内收敛**。上层业务代码永远操作 `UnifiedMessage` / `UnifiedChatResponse`，不出现 `choices[0].delta.content` 或 `content_block_delta` 这种供应商特有字段。

2. **请求和响应分开适配**。`buildRequest` 和 `parseResponse` 是两个独立方向，不要混在一个方法里。

3. **错误格式独立解析**。两家的错误嵌套深度不同（OpenAI 的 `error.message` vs Anthropic 的 `error.error.message`），各自在 `parseError` 中处理，返回统一错误字符串。

4. **流式解析器保持无状态**（或最小状态）。Anthropic 的 SSE 需要维护事件名状态，但这个状态只在解析器内部，不污染外层。

5. **不要试图统一所有细节**。`max_tokens` 在 OpenAI 不是必填但在 Anthropic 是必填，`system` 消息的结构也不同——适配层处理这些差异，不要强行抹平到毫无区别。

6. **usage 归一化**。OpenAI 有 `total_tokens`、Anthropic 没有——适配层统一计算。

---

## 五、总结

| 层面 | 主要工作量 |
|------|-----------|
| 请求转换 | system 消息位置的差异（顶层 vs messages 数组内）、max_tokens 必填 vs 可选 |
| 响应解析 | content 从 `choices[0].message.content`（string）到 `content[0].text`（array of blocks）的映射 |
| 流式解析 | SSE 事件模型的根本性差异——OpenAI 只有 data 行，Anthropic 有多事件类型的复杂状态机 |
| 多模态 | `image_url` vs `source` 字段差异、`detail` 参数有无 |
| Tool Use | Anthropic 需要缓存 tool_use id 用于 tool_result 回传，OpenAI 的 tool_calls 是增量 JSON |
| 错误处理 | 错误对象在 JSON 中的嵌套深度不同 |

接入两家之后你会意识到，协议差异不是最头疼的——最头疼的是**你在写业务代码时总得想"这段逻辑 O 家支持 A 家不支持怎么办"**。统一适配层的价值正是把这个决策推后，让上层代码先写好统一逻辑，具体底层的"不支持"再由适配层报错或降级。
