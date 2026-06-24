---
title: 模型不支持 Responses 时如何用 Chat Bridge 兼容
date: 2026-06-24 16:20:00
categories:
- Program
tags:
- AI
- OpenAI
- Responses
- Chat
- 架构
- 后端
---

最近在网关里补了一轮 Responses 兼容。最麻烦的点不是接一个原生 `/v1/responses`，而是另一类更现实的问题：模型本身只有 Chat Completions 或 Anthropic Messages 能力，但客户端已经按 OpenAI Responses API 发请求。

如果直接回一句“不支持 Responses”，用户体验很差。很多模型明明可以完成任务，只是上游没有实现 Responses 这层协议。如果把 Responses 请求硬塞到 Chat endpoint，又会遇到 `input`、`previous_response_id`、`function_call_output`、`text.format`、`reasoning` 这些字段都对不上的问题。

所以这次做的是 Chat Bridge：对外仍然接受 Responses 请求，对内按路由能力把它转成 Chat 请求，上游返回后再包回 Responses 形状。听起来像字段转换，实际更像一层小型协议网关。

## 一、为什么需要 Chat Bridge

OpenAI Responses API 的形状和 Chat Completions 不一样：

```json
{
  "model": "deepseek-v4-pro",
  "input": [
    {
      "role": "user",
      "content": [
        { "type": "input_text", "text": "分析这段日志" }
      ]
    }
  ],
  "reasoning": { "effort": "medium" },
  "text": { "format": { "type": "json_object" } },
  "previous_response_id": "resp_tm_xxx",
  "stream": true
}
```

Chat Completions 要的是：

```json
{
  "model": "deepseek-v4-pro-202606",
  "messages": [
    { "role": "user", "content": "分析这段日志" }
  ],
  "reasoning_effort": "medium",
  "response_format": { "type": "json_object" },
  "stream": true
}
```

这里有几个不能偷懒的地方：

- `input` 要拆成 `messages`，不同 content block 要分别处理。
- `max_output_tokens` 要映射成 Chat 侧可用的 `max_completion_tokens`。
- `reasoning.effort` 可能要转成 `reasoning_effort`，也可能因为工具调用被策略删除。
- `text.format` 要转成 `response_format`。
- `tools` 和 `tool_choice` 要转成 Chat 可识别的函数调用格式。
- `previous_response_id` 要恢复上一轮上下文，不能原样转发。
- 上游返回 Chat 结构后，还要重新包装成 Responses 的 `output` 数组。

这不是简单兼容字段。它要保证客户端看到的是 Responses，供应商看到的是 Chat，中间还不能丢上下文、不能乱扣费、不能把坏历史送到上游。

## 二、先把路由能力显式化

第一个决定是：不要靠 `protocol` 猜一切。

有的路由是 OpenAI 原生 Responses，有的路由是 Azure Responses，有的路由只是 OpenAI-compatible Chat，还有的路由只能本地估算 token。后来我加了 `model_routes.endpoint_capabilities`，用 JSON 给每条路由标能力。

一个典型配置长这样：

```json
{
  "version": 1,
  "responses": {
    "create": "bridge_chat",
    "compact": "bridge_chat",
    "input_tokens": "local_estimate"
  },
  "chat": {
    "params": {
      "drop_when_tools": ["reasoning_effort"],
      "drop_disabled": ["thinking"],
      "rewrite": {
        "reasoning.effort": {
          "minimal": "low"
        }
      },
      "reason": "upstream_chat_compat"
    }
  }
}
```

这里把 Responses 的三个 endpoint 分开看：

| endpoint | 含义 | 可选模式 |
|---|---|---|
| `create` | `/v1/responses`，真正生成内容 | `native` / `bridge_chat` / `disabled` |
| `compact` | `/v1/responses/compact`，压缩上下文 | `native` / `bridge_chat` / `disabled` |
| `input_tokens` | `/v1/responses/input_tokens`，计数 | `native` / `local_estimate` / `disabled` |

为什么 `input_tokens` 不允许 `bridge_chat`？因为它不是生成请求。把计数请求桥接到 Chat 生成没有意义，最多只能调用供应商计数接口，或者本地估算。

代码里的模式大概是这几个：

```go
const (
  responsesRouteModeInherit       responsesRouteMode = "inherit"
  responsesRouteModeDisabled      responsesRouteMode = "disabled"
  responsesRouteModeNative        responsesRouteMode = "native"
  responsesRouteModeBridgeChat    responsesRouteMode = "bridge_chat"
  responsesRouteModeLocalEstimate responsesRouteMode = "local_estimate"
)
```

默认规则也要保守：

- OpenAI / Azure Responses 走 `native`。
- Anthropic Messages 的 `create` 可以默认 `bridge_chat`，`input_tokens` 可以走原生 count tokens，`compact` 暂时禁用。
- TokenHub、Vertex Gemini、DeepSeek 这类 OpenAI-compatible Chat 路由，不默认开启 Responses，必须显式配置 `bridge_chat`。
- 图片、视频、音频等非 Chat 协议不能配置 `bridge_chat`。

这个设计救了很多后续问题。路由过滤时只看能力，不在转发阶段临时猜“这个上游应该能不能接”。

## 三、Chat 入口也要识别 Responses 形状

现实里还有一种情况：客户端把 Responses 请求发到了 `/v1/chat/completions`。

如果 body 里有 `input`、`text`、`reasoning`、`previous_response_id`、`include`，但没有 `messages`，继续走 Chat 校验只会报“messages 不能为空”。这对用户没帮助，因为他其实发的是 Responses 形状。

所以网关加了一个保守识别：

```go
var responsesPayloadSignalFieldsForChat = []string{
  "input",
  "text",
  "reasoning",
  "previous_response_id",
  "include",
}
```

命中后复用已经读取的 body，转到 Responses create 链路。注意这里不能二次读取请求体，也不能重复幂等检查，否则中间件和日志会乱。对应本地提交是 `fix(gateway): 兼容 Chat 入口 Responses 请求形状`。

有一个边界必须守住：如果 body 已经有 `messages`，就仍然按 Chat 请求处理。否则某些客户端扩展字段里带了 `text`，会被误判成 Responses。

## 四、Responses input 到 Chat messages

Chat Bridge 的第一步是构造 Chat 请求体。

核心逻辑可以概括成：

```go
chat := map[string]interface{}{
  "model":    upstreamModel,
  "messages": responsesChatMessagesFromInput(data),
}

copyResponsesChatParam(chat, data, "temperature", "temperature")
copyResponsesChatParam(chat, data, "top_p", "top_p")
copyResponsesChatParam(chat, data, "stream", "stream")

if v, ok := data["max_output_tokens"]; ok {
  chat["max_completion_tokens"] = v
}

if effort := responsesReasoningEffort(data["reasoning"]); effort != "" {
  chat["reasoning_effort"] = effort
}

if responseFormat, ok := responsesResponseFormatForChat(data); ok {
  chat["response_format"] = responseFormat
}
```

`input` 的转换最关键。Responses 里可能出现这些形状：

```json
{
  "input": "hello"
}
```

```json
{
  "input": [
    {
      "role": "user",
      "content": [
        { "type": "input_text", "text": "看图回答" },
        {
          "type": "input_image",
          "image_url": "https://example.com/a.png",
          "detail": "high"
        }
      ]
    }
  ]
}
```

桥接到 OpenAI-compatible Chat 后要变成：

```json
{
  "messages": [
    {
      "role": "user",
      "content": [
        { "type": "text", "text": "看图回答" },
        {
          "type": "image_url",
          "image_url": {
            "url": "https://example.com/a.png",
            "detail": "high"
          }
        }
      ]
    }
  ]
}
```

这个图片桥接后来单独修过一次，对应提交是 `fix(responses): 修复图片消息桥接格式`。问题很小，但影响很大：Responses 的 `input_image` 不能直接塞给 Chat，OpenAI-compatible Chat 需要 `image_url`。

## 五、工具调用历史必须先校验

Responses 的工具调用历史长这样：

```json
{
  "input": [
    {
      "type": "function_call",
      "call_id": "call_lookup",
      "name": "lookup_order",
      "arguments": "{\"order_id\":\"123\"}"
    },
    {
      "type": "function_call_output",
      "call_id": "call_lookup",
      "output": { "status": "paid" }
    },
    {
      "role": "user",
      "content": "继续分析"
    }
  ]
}
```

桥接成 Chat 时，要变成 assistant tool_calls 加 tool message：

```json
{
  "messages": [
    {
      "role": "assistant",
      "content": "",
      "tool_calls": [
        {
          "id": "call_lookup",
          "type": "function",
          "function": {
            "name": "lookup_order",
            "arguments": "{\"order_id\":\"123\"}"
          }
        }
      ]
    },
    {
      "role": "tool",
      "tool_call_id": "call_lookup",
      "content": "{\"status\":\"paid\"}"
    },
    {
      "role": "user",
      "content": "继续分析"
    }
  ]
}
```

这里不能只做转换，还要做校验。

如果有 `function_call` 却没有对应的 `function_call_output`，严格模型会报错。更糟的是，错误来自上游，用户看到的会是一段供应商内部文案。后来我在桥接前加了校验：

```go
func validateResponsesChatBridgeToolHistory(body []byte) error {
  if err := validateOpenAIToolResponseOrdering(payload.Messages); err != nil {
    return newValidationError(
      param,
      "missing tool response for Responses Chat bridge",
      "When using Responses with tools on a Chat-bridged model, every function_call must be immediately followed by its matching function_call_output. Pass the original previous_response_id with complete tool outputs, or start a new response.",
    )
  }
}
```

这对应 `fix(gateway): 校验 Responses 桥接工具调用`。后来又补了不完整 tool turn 的清理：如果历史里残留半截工具调用，且后面已经是新的用户请求，就把那段不完整工具 turn 丢掉，避免无效历史拖垮新请求。

## 六、`previous_response_id` 不能只存一个字符串

Responses 的上下文恢复比 Chat 麻烦。

Chat 请求通常由客户端带完整 `messages`。Responses 则允许客户端只带 `previous_response_id`。这要求网关知道这个 id 对应的上游响应、路由、模型、用户和桥接上下文。

本地结构里保存了这些字段：

```go
type responsesIDMapping struct {
  PublicID           string
  UpstreamID         string
  UserID             uint
  ModelID            string
  RequestedModel     string
  ChannelID          uint
  UpstreamModel      string
  Protocol           string
  PoolKeyID          uint
  LastRequestID      string
  OriginalResponseID string
  BridgeMessages     []byte
}
```

如果是原生 Responses，重点是把对外 id 映射回上游 id。比如用户看到 `resp_tm_xxx`，下一次请求进来时要替换成真实上游 id。

如果是 Chat Bridge，就没有上游 Responses 上下文可用。网关必须把已转换出的 Chat messages 存下来。下一次带 `previous_response_id` 时，先恢复 `BridgeMessages`，再把当前 input 追加到后面。

这个恢复还有几个校验：

- `previous_response_id` 必须属于当前 API key 对应的用户。
- 模型必须一致，不能拿 A 模型的上下文继续跑 B 模型。
- 路由要匹配上一次的 channel、protocol、upstream model。
- 找不到上下文时要给明确错误，不能静默退化成空上下文。

这部分对应 `feat(gateway): 归档失败请求并校验 Responses 上下文` 和 `fix(gateway): 优化 Responses 上下文恢复与失败提示`。其中“失败请求也归档”很重要。生成过程中失败，不代表没有产生 response id、usage 或可诊断的上游错误。调试和恢复都需要这些信息。

## 七、compact 也能桥接，但要承认它只是摘要

Responses 有 `/v1/responses/compact`，用于压缩上下文。原生支持时可以直接调用上游。模型不支持 Responses 时，只能用 Chat 生成摘要。

桥接时我没有假装它是原生 compact，而是把它当成“上下文摘要任务”：

```text
You are compacting conversation context for a future model request.

Write a concise but complete continuation summary. Do not answer the user's task.

Preserve:
- the user's current goal and hard constraints
- decisions already made
- unfinished tasks and next steps
- important files, request IDs, error messages, tool results, validation results, and billing or safety constraints
- any opaque or encrypted context only as an explicit note that it exists
```

输入由两部分组成：

- `previous_bridge_context`：上一次保存的桥接上下文。
- `current_compact_input`：本次 compact 请求里新的 input。

如果上下文太长，就按字节上限裁剪旧消息。生成出来的 summary 再存成一条 system message：

```json
[
  {
    "role": "system",
    "content": "Prior conversation compact summary:\n..."
  }
]
```

这不是完美模拟。它丢掉了原生 Responses 可能保留的结构化状态，但比直接拒绝好很多。文章里我要强调这一点：Chat Bridge 的 compact 是“可继续工作的摘要”，不是原生状态迁移。

## 八、encrypted_content 要 fail closed

Responses 里可能出现 `encrypted_content`，通常是 reasoning 或上下文里的不透明状态。

Chat 模型读不懂它。网关也不能解密它。那怎么办？

当前策略是：

- 如果请求里既有可读 input，又有未映射的 `encrypted_content`，可以把不可读部分剥掉，并记录 metadata。
- 如果请求只有 `encrypted_content`，没有任何可读上下文，就不能桥接。否则模型会在没有真实上下文的情况下乱答。
- 如果之前存过 `encrypted_content` 的路由 affinity，后续请求要尽量回到同一路由。
- 找不到 affinity 时 fail closed，返回可诊断错误。

这部分看着保守，但很必要。加密上下文是典型的“看起来像字符串，实际上是协议状态”。把它当普通文本会产生假连续性。

## 九、Chat 响应要重新包装成 Responses

上游 Chat 返回：

```json
{
  "id": "chatcmpl_xxx",
  "choices": [
    {
      "message": {
        "role": "assistant",
        "content": "分析结果如下"
      },
      "finish_reason": "stop"
    }
  ],
  "usage": {
    "prompt_tokens": 100,
    "completion_tokens": 20,
    "total_tokens": 120
  }
}
```

对外要包装成 Responses：

```json
{
  "id": "resp_tm_xxx",
  "object": "response",
  "status": "completed",
  "model": "deepseek-v4-pro",
  "output": [
    {
      "type": "message",
      "status": "completed",
      "role": "assistant",
      "content": [
        {
          "type": "output_text",
          "text": "分析结果如下"
        }
      ]
    }
  ],
  "usage": {
    "input_tokens": 100,
    "output_tokens": 20,
    "total_tokens": 120
  }
}
```

如果 Chat 返回 tool_calls，还要变成 Responses 的 `function_call` output item。流式也一样，不能把 Chat chunk 原样吐给客户端。对外 Response ID 要重写，避免暴露上游 id，也方便下一轮 `previous_response_id` 找回上下文。

这个地方有一个容易忽略的点：usage 的来源要标记清楚。

- 上游给了准确 usage，就用 provider exact。
- 桥接过程中只有文本内容，就用 content estimate。
- compact bridge 可以用本地估算作为预扣来源。
- `input_tokens` endpoint 如果所有路由不可用，可以返回 local estimate，但 metadata 要写明这是估算，不是上游计数。

不标来源，后面排查账单会很痛苦。

## 十、长非流式和计费边界

Responses 很容易被客户端非流式调用，尤其是一些 SDK 默认 `stream=false`。如果用户让模型生成很长内容，边缘层、客户端、网关都有超时风险。

网关内部可以把非流式请求转成流式聚合，保持对上游的长连接，再把最终结果一次性返回。这里分两类：

- `safe_non_stream`：普通请求的内部保护，不应该提高预扣。
- `large_input`、`large_output`、`reasoning`、JSON 输出、模型白名单：真正长请求，可以使用更高的输出 token 预扣预算。

最近的提交 `fix(gateway): 调整长非流式预扣与追扣策略` 加了一个热配置：

```text
gateway_long_non_stream_prededuct_output_tokens = 32768
```

它只用于真正长非流式场景。普通 `safe_non_stream` 仍然走常规预扣，避免低余额用户被内部兼容策略误伤。

超过预扣后的追扣也收紧了。只有这些情况才允许追扣：

- 最终响应已交付给客户端。
- 长非流式已经生成并有可计费内容。
- 客户端取消前，已经有可计费输出写出。

如果只是上游返回了成本、但没有交付证据，不应该向用户追扣。协议层必须把交付证据写进 metadata，钱包层才能做正确判断。

## 十一、这套桥接链路的请求流程

整个链路可以画成这样：

```text
client
  |
  | POST /v1/responses
  v
parse Responses request
  |
resolve model and routes
  |
filter route by endpoint_capabilities
  |
  +-- native ------> upstream /v1/responses
  |
  +-- bridge_chat -> build Chat body
                     normalize tools
                     restore previous bridge messages
                     apply chat param policy
                     upstream /chat/completions or /messages
                     wrap Chat response as Responses
                     store public response id mapping
  |
  +-- local_estimate -> input_tokens only
```

失败分支也要写日志：

- 路由不支持 endpoint，要记录 unsupported / disabled / invalid_config。
- `previous_response_id` 路由不匹配，要记录 mismatch。
- encrypted affinity 找不到，要记录 fail closed。
- 工具调用历史不完整，要返回可读的 400。
- compact 缺少可读上下文，要明确告诉用户缺 `previous_response_id` 或只有 opaque context。

这些日志不是给好看用的。桥接层的问题很容易表现成“模型不可用”或“上游 400”，没有 metadata 就很难定位。

## 十二、测试应该覆盖哪些坑

这次本地测试主要补了这些场景：

- Chat 入口识别 Responses 请求形状，但 `messages` 优先级更高。
- `create` / `compact` / `input_tokens` 的 route mode 解析。
- 非原生路由默认 disabled，显式配置后才允许 `bridge_chat`。
- Responses input 转 Chat messages，包括文本、图片、工具调用。
- `function_call_output` 缺失时提前报错。
- 不完整 tool turn 可以清理，避免污染新请求。
- `previous_response_id` 能恢复桥接上下文，模型和路由不匹配时拒绝。
- compact bridge 生成摘要，只保存 summary，不把整段旧历史无限膨胀。
- encrypted_content 只能作为 opaque context 处理。
- Chat bridge 输出能包装回 Responses，并保留 usage。
- `input_tokens` 原生不可用时返回 local estimate。
- 长非流式只在真实长请求时放大预扣，普通 safe_non_stream 不放大。
- 超过预扣追扣必须依赖交付证据。

这些用例看起来多，但每个都对应生产里会出现的一种怪请求。

## 十三、几个设计教训

第一，桥接不是“降级版 Responses”。对外要尽量像 Responses，对内要诚实承认自己在用 Chat。metadata 里要写 `responses_route_mode=bridge_chat`，否则后面调账、排障都会混。

第二，route capability 必须落在数据层。写死“某协议支持 Responses”很快会失控。同一个协议下，不同供应商、不同模型、不同 endpoint 的能力都可能不同。

第三，`previous_response_id` 是状态，不是字符串。它要绑定用户、模型、路由、上游 id、桥接消息和密钥池信息。

第四，工具历史先验错比上游报错更好。自己能判断的 400，就不要让上游替你判断。

第五，计费要跟交付事实绑定。协议桥接层最清楚哪些内容真的发给了用户，所以交付证据要从这里产生。

最后，Chat Bridge 的目标不是让所有模型“真正支持 Responses”。它只是给不支持 Responses 的模型补一条可用路径。能原生就原生，不能原生再桥接，桥接不了就明确失败。这个顺序不能反。

## 总结

模型不支持 Responses，不代表它完全不能服务 Responses 客户端。只要模型能稳定完成 Chat 请求，就可以通过 Chat Bridge 支持一部分 Responses 能力。

但这层桥接必须足够谨慎。请求形状、路由能力、上下文恢复、工具调用、多模态、响应包装、usage、长非流式和计费边界都要一起考虑。少任何一块，问题都可能变成“偶发 400”“上下文丢失”“重复扣费”或者“模型看起来不稳定”。

我现在更倾向于把它当成一个明确的协议适配层，而不是兼容补丁。补丁会越补越乱，协议层至少知道自己在保护什么。
