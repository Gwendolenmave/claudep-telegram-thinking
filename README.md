# Claude Code CLI → Telegram：可折叠 Thinking 教程

把 `claude -p` / Claude Code CLI 在 `stream-json` 模式下暴露的结构化 `thinking` block，与最终回复分开解析，并显示为 Telegram 中可点击展开的折叠区域。

```text
Bot is thinking…
[点击展开原始 thinking]

正常最终回复
```

> 本教程面向这类链路：`Telegram Bot → 自建后端 → Claude Code CLI / claude -p`。
>
> 它讨论的是 **Claude Code CLI 暴露的结构化 thinking 内容**，不声称获得完整、永久稳定的隐藏思维链。不同 Claude Code 版本、模型和运行模式的事件结构可能变化，因此第一步永远是能力探测。

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

### 6. 增加简单开关

推荐只做三个命令：

```text
/think auto
/think off
/think status
```

- `auto`：有真实 thinking 就展示，没有就只显示正文
- `off`：永远只显示正文
- `status`：显示 runner、输出格式、能力状态和当前模式

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

## 最小测试清单

- 只有 `result`，没有 thinking
- 一个 thinking block + result
- 多个 thinking block
- malformed NDJSON line
- stdout / stderr 分离
- thinking 中包含 `<`、`>`、`&`
- 超长 thinking 被截断，正文完整
- `/think auto`、`off`、`status`
- 模式重启后保留
- Telegram 富文本失败时正文只发送一次
- thinking 不进入 history、memory、summary 和 embedding
- 非 Claude Code provider 行为不受影响

## 可直接复制给 Codex / Claude Code 的施工 Prompt

<details>
<summary><strong>展开并复制完整 Prompt</strong></summary>

```text
请为当前“Claude Code CLI → Telegram Bot”项目实现真实 Thinking 的折叠展示。

目标效果：

Bot is thinking…
[Telegram 中默认折叠，点击后展开原始 thinking]

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

发送降级顺序：

1. expandable blockquote
2. spoiler
3. 普通 blockquote
4. 原始纯正文

降级重试必须保证正文只发送一次。

五、模式设置

实现：

/think auto
/think off
/think status

auto：有真实 thinking 时展示；没有时只显示正文。
off：只显示正文。
status：显示当前模式、Claude runner、输出格式、是否支持真实 thinking、上轮是否检测到 thinking、language policy=preserve。

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
- /think auto/off/status
- 设置持久化
- Telegram 四级降级
- 正文不会重复发送
- thinking 不进入 history/memory
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
