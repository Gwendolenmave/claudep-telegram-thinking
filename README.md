# Claude Code CLI → Telegram：可折叠 Thinking 与 Little Thought 教程

把 `claude -p` / Claude Code CLI 在 `stream-json` 模式下暴露的结构化 `thinking` block，与最终回复分开解析，并显示为 Telegram 中可点击展开的折叠区域。

```text
Bot is thinking…
[点击展开原始 thinking]

正常最终回复
```

> 本教程面向这类链路：`Telegram Bot → 自建后端 → Claude Code CLI / claude -p`。
>
> 它讨论的是 **Claude Code CLI 暴露的结构化 thinking 内容**，不声称获得完整、永久稳定的隐藏思维链。不同 Claude Code 版本、模型和运行模式的事件结构可能变化，因此第一步永远是能力探测。

## 相关教程

- [Delos Relay：让规划型 AI 与 Coding Agent 不再靠人肉复制粘贴](./guides/delos-relay.md)

## 两种展示：Raw Thinking vs Little Thought

两者都从 `stream-json` 的独立 thinking block 出发，但产品目的完全不同：

| | Raw Thinking | Little Thought |
|---|---|---|
| 内容 | 原始 thinking 的完整或截断文本 | 只允许公开的一句微型念头 |
| 适用场景 | 本地调试、能力验证 | 面向用户的日常界面 |
| 默认状态 | 建议关闭，仅 owner 临时开启 | 建议作为 companion UI 的默认模式 |
| 额外延迟 | 无 | 确定性选择器为零；模型改写器应异步 |
| 持久化 | 不进入历史或记忆 | 可只记 hash/长度，正文仍独立持久化 |

### 对比例子

下面是脱敏重制的示意内容，不来自私人聊天原文。

**Raw Thinking（诊断模式）**

```text
Amelia is thinking…
┌─────────────────────────────────────────────┐
│ Let me inspect the earlier context carefully.│
│ The user asked ...                           │
│ Previous messages show ...                   │
│ I need to decide how to respond ...          │
└─────────────────────────────────────────────┘

最终回复
```

**Little Thought（推荐的日常模式）**

```text
┌─────────────────────────────────────────────┐
│ I should answer simply and stay present.     │
└─────────────────────────────────────────────┘
最终回复
```

Little Thought 不是“公开一份更短的隐藏思维链”，而是从内部 processing 中生成或选择的 **public-facing micro-thought**。它必须能够安全地直接给用户阅读；拿不准时就不展示。

## 原理

普通调用通常使用：

```bash
claude -p "Your prompt" --output-format json
```

这种模式一般只留下最终 `result`，中间的结构化 thinking 不会出现在程序可消费的输出中。

改为：

```bash
claude -p "Your prompt" \
  --output-format stream-json \
  --verbose
```

stdout 会变成 NDJSON：每一行都是一个 JSON 事件。实际可用版本中，thinking 和最终答案会分别出现，例如：

```json
{"type":"assistant","message":{"content":[
  {"type":"thinking","thinking":"Reasoning text...","signature":"..."}
]}}
```

最终正文仍来自终结事件：

```json
{"type":"result","subtype":"success","result":"Final answer"}
```

因此核心只有两步：

```text
thinking = 所有 type == "thinking" block 的 thinking 字段
text     = 成功 result 事件的 result 字段
```

然后把二者沿应用链路分开传递：

```text
Claude Code CLI
→ NDJSON parser
→ { text, thinking? }
→ Telegram renderer
```

## 先分清两种看起来很像的 Thinking

### 1. 模型写进最终正文的 `<details>`

```html
<details><summary>Thinking</summary>
Keep it short and teasing.
</details>

最终回复正文
```

这只是普通 final text。程序只能靠格式猜测或正则拆分，不能据此认定存在独立 thinking 通道。

### 2. `stream-json` 中的独立 thinking block

```json
{
  "type": "assistant",
  "message": {
    "content": [
      {
        "type": "thinking",
        "thinking": "Reasoning text...",
        "signature": "opaque-value"
      }
    ]
  }
}
```

它与最终答案在数据流中有独立边界，可以可靠地：

- 折叠展示
- 保留原始语言
- 与最终正文分开存储
- 排除出历史、记忆、摘要和 embedding
- 避免用正则猜测正文边界

## 实现步骤

### 1. 在隔离环境中做能力探测

先查看当前版本：

```bash
claude --version
claude --help
```

然后执行一个最小测试：

```bash
claude -p "A farmer has 17 sheep. All but 9 run away. How many remain?" \
  --output-format stream-json \
  --verbose \
  > /tmp/claude-stream.ndjson
```

查看 assistant 和 result 事件：

```bash
jq -c 'select(.type == "assistant" or .type == "result")' \
  /tmp/claude-stream.ndjson
```

只查看 thinking block：

```bash
jq -c '
  select(.type == "assistant")
  | .message.content[]?
  | select(.type == "thinking")
' /tmp/claude-stream.ndjson
```

确认能看到独立的 `type: "thinking"` 后再继续施工。没有看到时，先检查实际调用参数、Claude Code 版本与模型行为，不要用普通正文伪造“真实 Thinking”。

### 2. 将调用改为 `stream-json --verbose`

旧调用：

```bash
claude -p "..." --output-format json
```

新调用：

```bash
claude -p "..." --output-format stream-json --verbose
```

已有的认证方式、system prompt、工具限制、安全参数和 session 配置保持原样；只替换输出格式相关参数。

### 3. 实现 NDJSON parser

下面是一个最小 TypeScript 版本：

```ts
export interface ParsedClaudeResponse {
  text: string;
  thinking?: {
    source: "reasoning";
    text: string;
  };
}

export function parseClaudeStreamJson(
  stdout: string,
): ParsedClaudeResponse {
  const thinkingParts: string[] = [];
  let finalText: string | undefined;

  for (const rawLine of stdout.split(/\r?\n/)) {
    const line = rawLine.trim();
    if (!line) continue;

    let event: unknown;
    try {
      event = JSON.parse(line);
    } catch {
      // 可记录诊断信息，但不要把坏行发给 Telegram。
      continue;
    }

    if (!isRecord(event)) continue;

    if (event.type === "assistant" && isRecord(event.message)) {
      const content = event.message.content;

      if (Array.isArray(content)) {
        for (const block of content) {
          if (
            isRecord(block) &&
            block.type === "thinking" &&
            typeof block.thinking === "string" &&
            block.thinking.trim()
          ) {
            thinkingParts.push(block.thinking);
          }
        }
      }
    }

    if (
      event.type === "result" &&
      event.subtype === "success" &&
      typeof event.result === "string"
    ) {
      finalText = event.result;
    }
  }

  if (!finalText) {
    throw new Error("Claude stream ended without a successful result event.");
  }

  const thinkingText = thinkingParts.join("\n\n").trim();

  return {
    text: finalText,
    ...(thinkingText
      ? {
          thinking: {
            source: "reasoning" as const,
            text: thinkingText,
          },
        }
      : {}),
  };
}

function isRecord(value: unknown): value is Record<string, any> {
  return typeof value === "object" && value !== null;
}
```

关键规则：

- 最终正文以成功 `result.result` 为准
- 多个 thinking block 按原顺序合并
- `signature` 是不透明字段，不展示、不解析
- stderr、system、status、tool 和 debug 事件不进入正文
- 不要把所有 stdout 行直接 `join()` 成回复

### 4. 扩展内部输出结构

原来：

```ts
type ModelResult = {
  text: string;
};
```

改为：

```ts
type ModelResult = {
  text: string;
  thinking?: {
    source: "reasoning";
    text: string;
  };
};
```

沿现有调用链传递：

```text
Claude provider
→ chat service
→ Telegram runtime
```

Telegram 层只消费统一结构，不直接理解 Claude Code 原始事件。

### 5. 用 Telegram 原生折叠块展示

Telegram HTML parse mode 支持：

```html
<b>Bot is thinking…</b>
<blockquote expandable>
Thinking content
</blockquote>

Final answer
```

最小 renderer：

```ts
function escapeTelegramHtml(text: string): string {
  return text
    .replaceAll("&", "&amp;")
    .replaceAll("<", "&lt;")
    .replaceAll(">", "&gt;")
    .replaceAll('"', "&quot;");
}

export function renderThinkingMessage(
  text: string,
  thinking?: string,
  maxThinkingChars = 3500,
): string {
  if (!thinking?.trim()) {
    return escapeTelegramHtml(text);
  }

  const clipped =
    thinking.length > maxThinkingChars
      ? `${thinking.slice(0, maxThinkingChars)}\n\n… [thinking truncated]`
      : thinking;

  return [
    "<b>Bot is thinking…</b>",
    `<blockquote expandable>${escapeTelegramHtml(clipped)}</blockquote>`,
    "",
    escapeTelegramHtml(text),
  ].join("\n");
}
```

发送时：

```ts
await sendMessage({
  chat_id: chatId,
  text: renderThinkingMessage(output.text, output.thinking?.text),
  parse_mode: "HTML",
});
```

生产环境建议准备降级阶梯：

```text
expandable blockquote
→ spoiler
→ 普通 blockquote
→ 纯正文
```

任何富文本错误都不能导致最终正文丢失或重复发送。

## 从 Raw Thinking 制作 Little Thought

生产环境的目标不是“尽量展示一点 raw”，而是满足以下门禁：

- 一句话，通常不超过 240–280 字符
- 第一人称、可直接面向用户阅读
- 不复述 system prompt、policy、memory、tool、消息编号或上下文清单
- 不泄露路径、token、内部状态、其他用户信息或逐步推理
- 没有安全候选时返回 `undefined`，只发送正文
- raw thinking 不落入 transcript、history、memory、summary 或 outbox

### 方案 A：确定性选择器（推荐基线，零额外模型调用）

将 raw thinking 视为不可信输入：切句、拒绝内部元叙述、限制长度，只选择天然已经像公开念头的一句。**绝不能用 `raw.slice(0, n)` 作为 fallback**，因为前 n 个字符往往正是上下文复盘或内部指令。

下面是最小参考实现；真实项目还应按自己的语言和隐私边界补规则：

```ts
const PRIVATE_OR_META = [
  /^[-*]\s/,
  /\b(system|prompt|policy|safety|context|memory|token|tool|json)\b/i,
  /\b(the user|user said|earlier messages?|previous messages?)\b/i,
  /\bI need to (answer|respond|avoid|follow|check)\b/i,
  /\[[0-9, -]+\]/,
];

export function makeLittleThought(
  rawThinking: string,
  maxChars = 280,
): string | undefined {
  const candidates = rawThinking
    .replace(/\r/g, "")
    .split(/(?<=[.!?])\s+|\n+/)
    .map((part) => part.trim())
    .filter(Boolean);

  return candidates.find((sentence) =>
    sentence.length >= 12 &&
    sentence.length <= maxChars &&
    /\b(I|I'm|I'll|me|my)\b/i.test(sentence) &&
    !PRIVATE_OR_META.some((pattern) => pattern.test(sentence))
  );
}
```

这个函数的失败结果是“不展示 Little Thought”，不是退回 raw thinking。规则是隐私门，不是安全证明；上线前仍需用真实但脱敏的 adversarial fixtures 做测试。

### 方案 B：模型改写器（可选，优先异步）

如果希望把复杂 processing 改写成更自然的一句，可以调用一个独立的小模型，但不要让它阻塞正文：

```text
正文先正常发送
→ bounded/redacted input 交给 compactor
→ 输出恰好一句 public-facing thought
→ 再做 deterministic privacy gate
→ 通过后异步补充；超时或失败则什么都不做
```

同步 compactor 会把模型启动和网络延迟加到每轮回复上；短 timeout 又容易导致几乎每次都进入 fallback。若产品不需要异步编辑，优先使用方案 A。

### 推荐 Telegram renderer

日常 Little Thought 不需要加粗标题，也不需要 expandable：只保留 Telegram 原生紫色 blockquote，并用一个必要换行紧接正文。

```ts
export function renderLittleThoughtMessage(
  text: string,
  littleThought?: string,
): string {
  if (!littleThought?.trim()) {
    return escapeTelegramHtml(text);
  }

  return [
    `<blockquote>${escapeTelegramHtml(littleThought)}</blockquote>`,
    escapeTelegramHtml(text),
  ].join("\n");
}
```

Raw Thinking 若保留为 owner-only 诊断模式，仍可使用 `<blockquote expandable>`，并明确标注它是 raw/debug 内容。

### 状态与测试

推荐模式：

```text
/think raw       # owner-only，完整诊断视图
/think compact   # Little Thought，推荐默认
/think off       # 只显示正文
/think status    # runner、能力、模式、持久化策略
```

至少覆盖：meta/context 清单拒收、消息编号拒收、HTML 转义、无安全候选、长度上界、raw 不落盘、Little Thought 与正文只发送一次，以及 compactor timeout 不延迟或吞掉正文。

### 为什么不需要编辑消息？

确定性选择器可以在第一次 Telegram `sendMessage` 前完成，因此 Little Thought 与正文一次性发送，不需要 `editMessageText`。只有选择异步模型改写方案时，才需要决定是编辑原消息、补发一条，还是直接放弃迟到的 thought。


### 6. 增加简单开关

推荐提供四个命令：

```text
/think raw
/think compact
/think off
/think status
```

- `raw`：仅 owner 临时查看完整原始 thinking
- `compact`：展示通过门禁的 Little Thought；没有安全候选时只显示正文
- `off`：永远只显示正文
- `status`：显示 runner、输出格式、能力状态、当前模式与 persistence policy

模式应复用项目现有 settings/state 层持久化。

### 7. 将 Thinking 排除出历史和记忆

最终正文继续进入原有 transcript/history/memory 流程。

thinking 不应进入：

- 上下文历史
- memory extraction
- summary
- embedding
- 人格或关系上下文
- 崩溃恢复正文

若需要记录诊断事件，推荐默认只保存：

```ts
type ThinkingMetadata = {
  messageId: string;
  source: "reasoning";
  chars: number;
  sha256: string;
};
```

当轮 Telegram 仍可展示全文，但不必长期保存原文。


> **隐私提醒：**公开 README 不应使用真实私人对话截图来演示 raw thinking。原始 thinking 可能复述聊天历史、记忆卡、内部策略或敏感事实；请使用人工重制、完全脱敏的 fixture。

## 最小测试清单

- 只有 `result`，没有 thinking
- 一个 thinking block + result
- 多个 thinking block
- malformed NDJSON line
- stdout / stderr 分离
- thinking 中包含 `<`、`>`、`&`
- 超长 thinking 被截断，正文完整
- `/think raw`、`compact`、`off`、`status`
- compact 拒绝 meta/context 清单且无安全候选时不显示
- 模式重启后保留
- Telegram 富文本失败时正文只发送一次
- raw thinking 不进入 history、memory、summary、embedding 或 outbox
- 非 Claude Code provider 行为不受影响

## 可直接复制给 Codex / Claude Code 的施工 Prompt

<details>
<summary><strong>展开并复制完整 Prompt</strong></summary>

```text
请为当前“Claude Code CLI → Telegram Bot”项目实现真实 Thinking 的折叠展示。

目标效果：

raw（owner-only 诊断）：
Bot is thinking…
[Telegram 中默认折叠，点击后展开原始 thinking]

compact（日常默认）：
[无标题的紫色 Little Thought blockquote]
正常最终回复正文

请先理解当前代码结构和真实调用链，再自主完成实现、测试和集成准备。

一、能力验证

找到当前调用 Claude Code / claude -p 子进程的位置和实际命令。

在不影响 live bot 的隔离环境中，用相同模型和主要参数分别测试：

--output-format json

以及：

--output-format stream-json --verbose

捕获完整 stdout 和 stderr。

确认 stream-json 中是否存在独立事件：

{
  "type": "assistant",
  "message": {
    "content": [
      {
        "type": "thinking",
        "thinking": "..."
      }
    ]
  }
}

同时确认最终正文来自成功 result 事件的 .result 字段。

只有确认真实、独立的 thinking block 存在后才继续实现。

二、修改 provider

将现有 Claude Code invocation 的输出格式从 json 改为：

--output-format stream-json --verbose

其余已有安全参数、模型配置、工具限制和认证方式保持不变。

实现 NDJSON parser：

- 每行解析一个 JSON event
- 最终正文取成功 result 事件的 .result
- thinking 取 assistant.message.content 中所有 type == "thinking" block 的 .thinking
- 多个 thinking block 按原顺序用空行连接
- 不展示或解析 signature
- stderr、system、status、tool 和 debug 信息不进入正文或 thinking
- 没有成功 result 时按现有错误流程失败

三、统一输出结构

扩展现有 ModelResult / TurnOutcome，而不是平行创建重复架构：

{
  text: string,
  thinking?: {
    source: "reasoning",
    text: string
  }
}

text 只包含最终正文。
thinking 只包含真实 thinking。

四、Telegram 展示

增加独立 renderer。

有 thinking 时使用 HTML：

<b>Bot is thinking…</b>
<blockquote expandable>
原始 thinking
</blockquote>

最终正文

要求：

- thinking 保留原始语言
- HTML 转义 &, <, > 和引号
- thinking 只放在第一条分片消息中
- 默认最大显示 3000–3500 字符
- thinking 超长时截断，但正文必须完整
- 使用 parse_mode="HTML"

另实现 Little Thought：

- 对 raw thinking 切句并执行 deterministic privacy gate
- 拒绝 system/prompt/policy/safety/context/memory/tool、消息编号、用户事实清单和“I need to respond”式内部元叙述
- 只选择一句 12–280 字符、第一人称、可直接公开阅读的候选
- 没有安全候选时只发送正文，绝不 raw.slice() 或回退展示原始 thinking
- compact renderer 只使用普通 <blockquote>，不加粗标题
- blockquote 后只留一个必要换行，紧接正文
- 本阶段不增加 secondary-model 调用；若以后增加模型改写，必须异步且失败不影响正文

发送降级顺序：

1. expandable blockquote
2. spoiler
3. 普通 blockquote
4. 原始纯正文

降级重试必须保证正文只发送一次。

五、模式设置

实现：

/think raw
/think compact
/think off
/think status

raw：仅 owner 临时查看 expandable 原始 thinking。
compact：展示通过门禁的 Little Thought；没有安全候选时只显示正文。
off：只显示正文。
status：显示当前模式、Claude runner、输出格式、是否支持真实 thinking、上轮是否检测到 thinking、persistence policy 与 language policy。

模式使用项目现有 settings/state 层持久化，重启后保留。

六、历史与记忆隔离

thinking 与最终 assistant message 分开处理。

最终正文继续进入现有 transcript/history/memory 流程。

thinking 不进入：

- 上下文历史
- memory extraction
- summary
- embedding
- 人格上下文
- 崩溃恢复正文

若需要记录 thinking 事件，默认只保存：

- messageId
- source
- chars
- sha256

当轮 Telegram 展示全文不受影响。

七、测试

覆盖：

- text-only result
- 单个和多个 thinking block
- NDJSON 多行解析
- malformed line
- stderr 隔离
- HTML 转义
- 超长 thinking
- /think raw/compact/off/status
- compact meta/context 拒收、长度上界、无候选省略
- 设置持久化
- Telegram 四级降级
- 正文不会重复发送
- raw thinking 不进入 history/memory/outbox
- 非 Claude Code provider 不受影响

运行完整 typecheck、build 和 test。

八、交付

在独立 branch/worktree 中完成，不直接部署 live bot。

完成后一次性汇报：

1. 真实 thinking 能力验证证据
2. 原调用命令与新调用命令
3. 修改文件
4. parser 和内部输出结构
5. Telegram renderer 与 fallback
6. 历史和记忆隔离方式
7. 测试结果
8. commit 和基线
9. 部署、验证与回滚步骤
```

</details>

## API 接 Telegram 呢？

Telegram renderer、模式开关、fallback 和记忆隔离逻辑都可以复用。

但本教程的上游 adapter 是 **Claude Code CLI 专用**：

```text
claude -p → stream-json → NDJSON parser
```

使用 Anthropic Messages API 时，应改为读取 API 返回的结构化 content blocks 或流式 delta，不需要启动子进程，也不使用本教程的 CLI parser。

## License

教程、Prompt 和代码示例采用 **CC BY-NC-SA 4.0**：允许复制、修改、翻译、分享，以及交给 Codex / Claude Code 等 coding agent 使用；需要署名、禁止商业用途，并按相同许可分享衍生版本。

详见 [`LICENSE.md`](./LICENSE.md) 与 [`RESPONSIBLE_USE.md`](./RESPONSIBLE_USE.md)。

## Credits

Built by Gwendolen with Amelia GPT and Amelia Claude.
