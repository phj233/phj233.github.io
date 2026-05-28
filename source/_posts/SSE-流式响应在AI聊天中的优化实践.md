---
title: SSE 流式响应在 AI 聊天中的优化实践
date: 2026-05-28 12:00:00
categories:
- Program
tags:
- SSE
- 流式
- Vue3
- TypeScript
- 后端
- 性能优化
---

AI 聊天需要流式输出——用户提问后，模型逐字生成回应的过程通过 SSE 实时推送到前端。这里记录一下前后端各做了什么优化。

## 一、前端 SSE 解析

前端 `useSseChat` composable 负责 SSE 流的消费：

```ts
import { ref } from 'vue'
import { readAuthToken } from '../utils/authSession'

type ChatCompletionStreamPayload = {
  choices?: Array<{
    delta?: {
      content?: string
      reasoning_content?: string
      reasoningContent?: string
    }
  }>
}

/**
 * SSE data 行可能出现双重 `data:data:` 前缀，需要递归去除。
 */
function normalizeSseLine(raw: string) {
  let line = raw.trim()
  while (line.startsWith('data:')) {
    line = line.slice(5).trimStart()
  }
  // 过滤 SSE 控制行（注释、event、id、retry）
  if (line.startsWith(':') || line.startsWith('event:')
      || line.startsWith('id:') || line.startsWith('retry:')) {
    return ''
  }
  return line.trim()
}

export function useSseChat() {
  const streaming = ref(false)
  let abortController: AbortController | null = null

  function stop() {
    abortController?.abort()
    abortController = null
    streaming.value = false
  }

  async function stream(
    model: string,
    messages: ChatMessage[],
    onDelta: (value: string) => void,
    options: StreamOptions = {},
  ) {
    stop()
    abortController = new AbortController()
    streaming.value = true

    try {
      const response = await fetch('/api/v1/chat/completions', {
        method: 'POST',
        headers: {
          'Content-Type': 'application/json',
          Accept: 'text/event-stream',
          token: readAuthToken(),
        },
        body: JSON.stringify({
          model,
          messages,
          stream: true,
          conversationUid: options.conversationUid || undefined,
          projectUid: options.projectUid || undefined,
          clientMessageId: options.clientMessageId || undefined,
          assistantClientMessageId: options.assistantClientMessageId || undefined,
        }),
        signal: abortController.signal,
      })

      if (!response.ok) {
        let message = `HTTP ${response.status}`
        try {
          const payload = await response.json()
          if (payload?.msg) message = payload.msg
        } catch { /* 非 JSON 响应体，保持 HTTP status 为错误提示 */ }
        throw new Error(message)
      }

      if (!response.body) throw new Error('当前浏览器不支持流式响应')

      const reader = response.body.getReader()
      const decoder = new TextDecoder('utf-8')
      let remainder = ''
      let currentEvent = ''

      const handleLine = (raw: string) => {
        const trimmed = raw.trim()

        // 空行 = 事件分隔符，重置当前事件名
        if (!trimmed) { currentEvent = ''; return }

        // event: 行 —— 记录自定义事件名
        if (trimmed.startsWith('event:')) {
          currentEvent = trimmed.slice(6).trim()
          return
        }

        // 忽略 SSE 标准注释/控制行
        if (trimmed.startsWith(':') || trimmed.startsWith('id:')
            || trimmed.startsWith('retry:')) return

        // 既不是 data: 也不是 JSON 开头的行，不是标准 SSE 数据行
        if (!trimmed.startsWith('data:') && !trimmed.startsWith('{')
            && trimmed !== '[DONE]') return

        const line = normalizeSseLine(raw)
        if (!line || line === '[DONE]') return

        // 自定义 error 事件 —— 抛出让外层 catch 处理
        if (currentEvent === 'error') {
          throw new Error(line)
        }

        // 自定义 metadata 事件 —— 传递会话/消息 UID
        if (currentEvent === 'metadata') {
          try { options.onMetadata?.(JSON.parse(line)) } catch { /* ... */ }
          return
        }

        // 标准 data 行 —— 解析 delta
        let payload: ChatCompletionStreamPayload
        try {
          payload = JSON.parse(line)
        } catch {
          throw new Error(line)
        }

        const choice = payload.choices?.[0]
        const reasoningDelta =
          choice?.delta?.reasoning_content
          || choice?.delta?.reasoningContent
          || ''
        const delta = choice?.delta?.content || ''

        if (reasoningDelta) options.onReasoningDelta?.(reasoningDelta)
        if (delta) onDelta(delta)
      }

      // 流式读取循环
      while (true) {
        const { value, done } = await reader.read()
        if (done) break
        remainder += decoder.decode(value, { stream: true })
        const lines = remainder.split('\n')
        remainder = lines.pop() || ''
        for (const raw of lines) {
          handleLine(raw)
        }
      }
      // 处理最后一行残留
      remainder += decoder.decode()
      if (remainder) handleLine(remainder)
    } finally {
      streaming.value = false
      abortController = null
    }
  }

  return { streaming, stream, stop }
}
```

### 设计要点

**1. 双重 data: 前缀兼容**。有些上游 SSE 会写出 `data:data:{...}`，`normalizeSseLine` 用 while 循环递归去掉所有 `data:` 前缀。

**2. 自定义事件**。除了标准 SSE `data:` 行，还支持自定义事件：
- `event: metadata` — 后端返回会话 UID、消息 UID，前端用于后续关联
- `event: error` — 流中发生错误时抛出，终止解析

**3. remainder 行残留处理**。SSE 流是分 chunk 到达的，`\n` 可能出现在 chunk 中间。用 `remainder` 变量缓存最后不完整的行，下个 chunk 来时拼接。

**4. AbortController 取消**。调用 `stop()` 时中止 fetch，后端检测到客户端断开后关闭上游流，释放连接资源。

---

## 二、聊天 Composable 编排

`useChatConversation` 在 `useSseChat` 之上做会话层编排：

```ts
export function useChatConversation(options: UseChatConversationOptions) {
  const chatStore = useChatStore()
  const sseChat = useSseChat()
  const activeStreamConversationId = ref('')
  const activeAssistantMessageId = ref('')

  async function sendMessage() {
    // 如果正在流式输出，点发送 = 停止生成
    if (chatStore.streaming) {
      await stopCurrentGeneration()
      return
    }

    const text = options.prompt.value.trim()
    if (!text || text.length > chatStore.maxChars) return

    const modelCode = chatStore.currentModel
    const upstream = buildUpstreamMessages()

    // 1. 添加用户消息到本地
    const user = chatStore.appendMessage({ role: 'user', content: text, modelCode })

    // 2. 添加占位的 assistant 消息（空内容、streaming 状态）
    const assistant = chatStore.appendMessage({
      role: 'assistant',
      content: '',
      modelCode,
      status: 'streaming',
    })

    // 3. 流式消费
    let buffer = ''
    let reasoningBuffer = ''
    try {
      await sseChat.stream(
        modelCode,
        upstream,
        (delta) => {
          buffer += delta
          chatStore.updateConversationMessage(
            activeStreamConversationId.value,
            assistant.id,
            { content: buffer, status: 'streaming' },
          )
          void options.scrollToBottom()
        },
        {
          conversationUid: chatStore.activeConversation?.serverSynced
            ? chatStore.activeConversation.id : null,
          projectUid: chatStore.activeProjectId || null,
          clientMessageId: user.id,
          assistantClientMessageId: assistant.id,
          onMetadata: (metadata) => {
            chatStore.applyServerMetadata({ ...metadata,
              localConversationId, localUserMessageId: user.id,
              localAssistantMessageId: assistant.id })
          },
          onReasoningDelta: (delta) => {
            reasoningBuffer += delta
            chatStore.updateConversationMessage(
              activeStreamConversationId.value,
              assistant.id,
              { reasoningContent: reasoningBuffer, status: 'streaming' })
          },
        },
      )
      // 流结束，标记完成
      chatStore.updateConversationMessage(activeStreamConversationId.value, assistant.id, {
        content: buffer.trim() ? buffer : '生成完成，但没有返回内容。',
        status: 'completed',
      })
    } catch (error) {
      if (error instanceof DOMException && error.name === 'AbortError') {
        // 用户主动停止
        await chatStore.cancelServerMessage(
          activeStreamConversationId.value, assistant.id,
          buffer, reasoningBuffer)
      } else {
        // 异常失败
        chatStore.updateConversationMessage(activeStreamConversationId.value, assistant.id, {
          content: `请求失败：${error instanceof Error ? error.message : '网络错误'}`,
          status: 'failed',
        })
      }
    } finally {
      chatStore.setStreaming(false)
      void options.scrollToBottom()
    }
  }
}
```

### 状态管理关键点

- **两阶段写入策略**：前端先写 user 消息和空 assistant 占位消息；流式过程中只更新 assistant 的内容；流结束/失败/取消时写终态状态
- **reasoning_content 独立处理**：深度思考模型的推理文本走独立的 `onReasoningDelta` 回调，和正文 `content` 分离存储
- **AbortError 判断**：`error instanceof DOMException && error.name === 'AbortError'` 区分"用户主动停止"和"网络异常"

---

## 三、后端优化要点

### 有界线程池

```java
// AsyncConfig.java
// chatStreamExecutor 使用有界线程池 + 零队列 + AbortPolicy
// 容量不足时返回 429 而不是让请求无限排队
```

### 客户端断开时传播取消

```java
// 关键逻辑
// 客户端断开或超时 → 后端关闭上游 SSE 响应流
// → 中断工作线程 → 释放后端连接和上游资源
```

### 归一化上游 SSE payload

不同供应商（OpenAI、DeepSeek、通义、Kimi）的 SSE 格式不完全相同。后端统一归一化为格式：

```
data: {"choices":[{"delta":{"content":"xxx"}}]}

event: metadata
data: {"conversationUid":"xxx","userMessageUid":"xxx"}

event: error
data: 上游服务响应超时

data: [DONE]
```

前端只需要处理一种格式。

### 会话落库策略

```java
// ChatConversationService
// 写入策略：
// 请求开始时 → 写 user 占位消息 + assistant 占位消息
// 流式过程中 → 只在内存累积，不逐 token 写库
// 流结束/失败/取消/超时 → 一次性更新 assistant 消息的 content
//
// 原因：逐 token 写库会显著放大数据库压力
```

### reasoning_content 保存规则

```java
// 只保存模型显式返回且允许展示的 reasoning_content
// 不保存原始 chunk
// 不参与后续上下文拼接
// 最长保留 50000 字符（防止极长推理撑爆数据库）
```

### 上下文窗口裁剪

```java
// 每次构建上游请求时：
// - 有服务端会话的：取最近 40 条 user/assistant 消息
// - 新会话的：只取本次请求消息
// - 加上 system 消息
// 不能把全量历史都发给上游
```

---

## 四、不可破坏边界

1. **流式格式必须兼容现有前端**。不能改 SSE payload 的 JSON 结构
2. **用户主动断开时不能继续无意义消耗上游资源**。后端必须关闭上游 SSE 流
3. **token 计费仍按上游 usage 后扣费**。缺少 usage 时不额外猜算
4. **会话保存不能逐 token 写库**。长文本输出下，逐 token 写库会放大 DB 压力
5. **所有会话/消息读写必须绑定 account_id**。不能仅凭前端传入的 uid 操作

当前主链路（前端 SSE 解析、后端流式代理、会话持久化）已线上验证通过，后续只做观察和小修，不做大改。
