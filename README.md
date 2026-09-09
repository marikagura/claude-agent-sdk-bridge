# 用 Claude Agent SDK 将订阅额度与完整 harness 接入自建聊天前端

> 本文记录一条自 2026-07-28 起持续运行的链路：手机上的聊天页 → 自建网关 → 家用 Mac 上的 Agent SDK → Claude Code 的 CLI 子进程。
> 计费走 Claude 订阅额度；运行环境是 Claude Code 的完整 harness（CLAUDE.md、MCP 工具、记忆库、session 续接、自动 compact）。
> 文中数字来自该链路的日志、转写与测量脚本，均标注日期；未测量的部分标注「未验证」。
> 基准版本：`@anthropic-ai/claude-agent-sdk` 0.3.258（内置 Claude Code 2.1.258），macOS，Node 22。

---

## 1. 概述

本实现的目标是在自建前端里得到与终端 Claude Code 相同的模型行为：同一份系统提示、同一份 CLAUDE.md、同一套 MCP 工具、同一种 session 与压缩机制，并且由订阅额度计费。实现方式是在本机以 Agent SDK 的 `query()` 逐轮启动 CLI 子进程，把事件流转发到前端。

全文分三部分：第 2、3 节说明 Agent SDK 与 `claude -p` 的关系、认证与计费的前提；第 4 至 9 节是实现本身（架构、`query()` 参数、事件流、权限、固定前缀、提示缓存）；第 10 至 14 节是运行期的事项（登录态、部署位置、上下文窗口、故障对照、版本）。

## 2. Agent SDK 与 `claude -p` 的关系与差别

以下内容核对自 SDK 包内的 `sdk.mjs`（0.3.258）与一次实测。

**同一个二进制。** SDK 包依赖一个平台包（macOS arm64 为 `@anthropic-ai/claude-agent-sdk-darwin-arm64`），其中是一份 Claude Code 二进制，版本与 SDK 对应（0.3.258 对应 2.1.258）。`query()` 启动的就是这份二进制；`pathToClaudeCodeExecutable` 可以指向另一份。它与终端里安装的 CLI 是两份文件，版本可能不同（本机终端为 2.1.266）。

**固定参数，不含 `-p`。** `query()` 对每个子进程固定传入：

```
--output-format stream-json --verbose --input-format stream-json
```

参数列表里没有 `-p` / `--print`。实测（2.1.258，stdin 为管道，仅上述三项加 `--model haiku --max-turns 1 --no-session-persistence --tools ""`）：进程以非交互方式完成一轮后退出，`system/init` 事件报 `apiKeySource: none`，即使用本机登录态。非交互模式由双向 stream-json 与非 TTY 的标准输入输出决定。

**事件格式相同。** `claude -p --output-format stream-json --verbose` 输出的每一行 JSON，与 SDK `for await` 得到的每一条消息是同一份数据；SDK 只做了按行解析与 TypeScript 类型。转写文件、session 目录、自动 compact、后台任务的退出等待、hook 超时等运行期行为也相同，因为它们都在 CLI 内。

**选项到命令行参数的映射。** 大多数选项是一对一的命令行参数：

| SDK 选项 | 命令行参数 |
| --- | --- |
| `model` / `effort` | `--model` / `--effort` |
| `tools: []` / `tools: [...]` / `tools: { type: "preset" }` | `--tools ""` / `--tools A,B` / `--tools default` |
| `allowedTools` / `disallowedTools` | `--allowedTools` / `--disallowedTools` |
| `settingSources` | `--setting-sources=user,project`；省略时不传，CLI 取默认（全部来源） |
| `mcpServers` | `--mcp-config <JSON 字符串>`；进程内 server 在其中记为 `{ "type": "sdk", "name": … }` |
| `strictMcpConfig` / `permissionMode` | `--strict-mcp-config` / `--permission-mode` |
| `canUseTool` | `--permission-prompt-tool stdio` |
| `permissionPromptToolName` | `--permission-prompt-tool <MCP 工具名>`，与 `canUseTool` 互斥 |
| `resume` / `continue` / `sessionId` | `--resume=<id>` / `--continue` / `--session-id=<id>` |
| `forkSession` / `resumeSessionAt` / `resumeDropsTurn` | `--fork-session` / `--resume-session-at=` / `--resume-drops-turn=` |
| `persistSession: false` | `--no-session-persistence` |
| `thinking: { type: "enabled", budgetTokens: N }` | `--max-thinking-tokens N` |
| `thinking: { type: "enabled" }`（无预算）或 `{ type: "adaptive" }` | `--thinking adaptive` |
| `thinking: { type: "disabled" }` | `--thinking disabled` |
| `thinking.display` | `--thinking-display summarized` 或 `omitted` |
| `includePartialMessages` / `includeHookEvents` | `--include-partial-messages` / `--include-hook-events` |
| `maxTurns` / `maxBudgetUsd` / `fallbackModel` / `betas` / `additionalDirectories` / `outputFormat`（json_schema） | `--max-turns` / `--max-budget-usd` / `--fallback-model` / `--betas` / `--add-dir` / `--json-schema` |

**不走命令行的部分，即 SDK 与 `-p` 的实质差别。** 子进程启动后，SDK 与 CLI 之间在同一对 stdin / stdout 上另有一层控制协议（`control_request` / `control_response` / `control_cancel_request`），以下功能都在这层上：

| 功能 | `-p` 的做法 | SDK 的做法 |
| --- | --- | --- |
| 提示词 | argv 或 stdin | 固定使用 stream-json 输入，字符串 prompt 也包装为一条 `user` 消息写入 stdin；`AsyncIterable` 形式可在一次 session 内持续追加 |
| 系统提示 | `--system-prompt` / `--system-prompt-file` / `--append-system-prompt` | 随 `initialize` 控制请求发送（字段 `systemPrompt`、`appendSystemPrompt`、`excludeDynamicSections`、`systemPromptSnapshot`），不经过 argv，无长度限制、不需临时文件。省略 `systemPrompt` 时发送空串，保留 CLI 默认；`{ type: "preset", append }` 只发送 `appendSystemPrompt` |
| 权限裁决 | `--allowedTools` 白名单、`--permission-mode`，或 `--permission-prompt-tool` 指向一个 MCP 工具 | `--permission-prompt-tool stdio`：CLI 发出 `can_use_tool` 控制请求（含工具名与入参），进程内的 `canUseTool` 回调返回 allow / deny |
| hooks | settings 文件中的命令行 hook | 进程内函数，随 `initialize` 登记，CLI 以 `hook_callback` 控制请求回调 |
| MCP | 独立的 server 进程或远端 URL | 除以上两种外，`createSdkMcpServer` 定义的进程内 server：`--mcp-config` 中记为 `type: "sdk"`，JSON-RPC 经 `mcp_message` 控制请求往返 |
| 运行中的控制 | 无现成客户端 | `interrupt()`、`setModel()`、`setPermissionMode()`、`setMaxThinkingTokens()`、`getContextUsage()`、`mcpServerStatus()`、`supportedModels()` 等，均为控制请求 |

控制协议由 CLI 实现，SDK 是它的客户端。因此用 `-p --input-format stream-json` 自行实现这层协议在理论上可行，但没有现成实现。

**进程环境。** `env` 选项整体替换子进程环境（类型文件注释原文：「REPLACES the subprocess environment entirely」），因此需要以 `{ ...process.env }` 为基础再增删。SDK 另外写入 `CLAUDE_CODE_ENTRYPOINT=sdk-ts` 与 `CLAUDE_AGENT_SDK_VERSION`，并删除 `NODE_OPTIONS`。

## 3. 认证与计费的前提

**凭据优先级**（官方 authentication 页）：云服务商凭据 > `ANTHROPIC_AUTH_TOKEN` > `ANTHROPIC_API_KEY` > `apiKeyHelper` > `CLAUDE_CODE_OAUTH_TOKEN` > Anthropic profile > `/login` 的订阅 OAuth。非交互模式下 `ANTHROPIC_API_KEY` 存在即被使用，不询问。因此走订阅的第一条前提是子进程环境里没有前四种凭据；本实现在启动每个子进程前删除 `ANTHROPIC_API_KEY` 与 `ANTHROPIC_AUTH_TOKEN`。

**登录态的位置。** macOS 为钥匙串条目 `Claude Code-credentials`（钥匙串不可写时，例如 SSH 会话，回落到 `~/.claude/.credentials.json`，权限 0600）；Linux 为 `~/.claude/.credentials.json`。SDK 子进程直接复用这份登录态。

**不能使用 `--bare`。** 官方 headless 页：「In bare mode, Claude Code never reads OAuth credentials or the system keychain」，并且「`--bare` … will become the default for `-p` in a future release」。当前 0.3.258 的 SDK 不传 `--bare`；升级 SDK 后需重新确认。

**政策状态**（2026-09-09 核对）。官方支持文章《Use the Claude Agent SDK with your Claude plan》最后更新于 2026-06-16：「nothing has changed: Claude Agent SDK, `claude -p`, and third-party app usage still draw from your subscription's usage limits」；原定 2026-06-15 生效的月度 credit 计划已暂缓。Agent SDK 概览页另有一条限制：「Unless previously approved, Anthropic does not allow third party developers to offer claude.ai login or rate limits for their products, including agents built on the Claude Agent SDK.」本文的范围是个人订阅在个人设备上的自用。

**安装后的自检。** 判据有两个：子进程环境无 API key，且模型确实返回了文本。`result.total_cost_usd` 在订阅轮次里也报折算金额，不能作为判据。

```js
import { query } from "@anthropic-ai/claude-agent-sdk";

const env = { ...process.env };
delete env.ANTHROPIC_API_KEY;
delete env.ANTHROPIC_AUTH_TOKEN;

let text = "";
for await (const m of query({
  prompt: "用一句话回答: 现在这个目录下有哪些文件? 如果你没有工具就直接说没有。",
  options: {
    model: "claude-opus-4-6",
    cwd: process.cwd(),
    settingSources: [],          // 不加载 CLAUDE.md，只验证通路
    tools: [],                   // 内置工具全部关闭
    includePartialMessages: true,
    env,
    canUseTool: async () => ({ behavior: "deny", message: "selfcheck" }),
  },
})) {
  if (m.type === "stream_event" && m.event?.delta?.type === "text_delta") text += m.event.delta.text;
  if (m.type === "result") console.log(m.subtype, JSON.stringify(m.usage));
}
console.log(text);
```

## 4. 架构

```
手机聊天页 ─POST /api/p─▶ Vercel 转发函数 ─submit─▶ 网关 (VPS，内存态 job 队列)
     ▲                          │ SSE                          ▲ pull (长轮询 25 s)  │ chunk / done / beat
     └── 断线后带 cursor 重连 ◀──┘                              │                      ▼
                                                    Mac: daemon (launchd 常驻)
                                                          └─ Agent SDK query() ─▶ CLI 子进程 ─▶ 订阅
```

- **出站长轮询。** Mac 没有入站端口，daemon 每 25 秒向网关领取一次任务，每 20 秒发送一次心跳（附带订阅额度读数）。网关 90 秒未收到心跳即把该模型标为「离线」：前端的模型切换器上该项可见但不可选。
- **三枚 token 分立。** Vercel 侧的 token 只能调用 submit / poll，daemon 侧的只能调用 pull / chunk / done / beat，网关主 key 不参与这组接口。任一未配置或两枚同值时整条链路返回 503。
- **SSE 断开不终止任务。** 手机锁屏或 Vercel 函数超时都不影响这一轮完成；整段回复由网关在任务结束时写入数据库。Vercel 函数上限 60 秒，转发路由在 50 秒时主动发送 `cut` 并结束，客户端带 cursor 重新连接。主动切断的原因是被平台终止的流没有结束标记，客户端无法区分「已说完」与「被中断」。
- **并发。** 同一 thread 串行（同一 session 的 resume 不能并发），不同 thread 并行，daemon 默认并发 2。串行由网关保证：同一 thread 的第二个任务在前一个完成前不会被分发。
- **session 对应。** 一个 thread 对应一个 CLI session id，内存中保存，并写入一条数据库事件作为网关重启后的恢复来源。
- **凭据文件。** 网关地址与 token 放在 `~/.p-bridge.env`（权限 600），不写入 launchd 的 plist；plist 会进入 Time Machine 等备份。

## 5. `query()` 的参数

daemon 中每轮调用的形状如下（省略了系统提示附加文本与日志）：

```js
const env = { ...process.env };
delete env.ANTHROPIC_API_KEY;
delete env.ANTHROPIC_AUTH_TOKEN;
env.CLAUDE_CODE_DISABLE_AUTO_MEMORY = "1";                              // 第 8 节
if (!env.MAX_MCP_OUTPUT_TOKENS) env.MAX_MCP_OUTPUT_TOKENS = "100000";  // 第 12 节

for await (const m of query({
  prompt,                                          // 字符串；带图时为 AsyncIterable
  options: {
    model,
    effort,
    cwd: CWD,
    settingSources: ["user", "project"],
    systemPrompt: { type: "preset", preset: "claude_code", append: APPEND },
    ...(sid ? { resume: sid } : {}),
    ...(fork ? { resumeSessionAt: fork.at, resumeDropsTurn: fork.drops } : {}),
    thinking: { type: "enabled", budgetTokens: 10000, display: "summarized" },
    includePartialMessages: true,
    abortController: abort,                        // 15 分钟上限
    env,
    tools: ["Bash", "Read", "Write", "Edit", "WebSearch", "WebFetch", "ToolSearch", "TaskOutput", "TaskStop"],
    mcpServers: { kimi: { ...fromMcpJson, url: fromMcpJson.url + "?profile=chat" }, ...inProcessServers },
    strictMcpConfig: true,
    canUseTool: async (name, input) =>
      toolAllowed(name)
        ? { behavior: "allow", updatedInput: input }
        : { behavior: "deny", message: `此配置不开放 ${name}` },
  },
})) { /* 第 6 节 */ }
```

| 参数 | 取值 | 说明 |
| --- | --- | --- |
| `model` | 模型 id | 前端与网关之间只传别名（`opus46`、`fable51` 等），别名到 id 的表只在 daemon 一处。版本号写入别名，不使用「最新版本」这类会随时间变化的默认 |
| `effort` | 按模型的可用档位取值，默认 `max` 或 `xhigh` | 每轮的固定开销（启动进程、加载 CLAUDE.md 与 MCP、订阅侧排队）已经存在，思考深度取高档。模型不支持的档位会被上游拒绝 |
| `cwd` | 项目目录 | 决定 CLAUDE.md 与 `.mcp.json` 的读取位置，以及转写文件所在的目录 `~/.claude/projects/<目录名>/`。第 9 节的缓存问题与此相关 |
| `settingSources` | `["user", "project"]` | 省略等于全部加载（含 `local`），`[]` 为隔离模式；`project` 是 CLAUDE.md 的开关。不含 `local` 的原因见第 7 节 |
| `systemPrompt` | `preset: "claude_code"` 加 `append` | 系统提示保持 Claude Code 自身那份；`append` 只写这一出口特有的规则（对话而非终端输出的格式、动作描写的排版、思考文本的写法）。自定义人设的写法是 `{ type: "custom", prompt }`，对照见第 8 节末尾 |
| `resume` | 上一轮的 session id | 一个 thread 一条 session，转写由本机 CLI 保存 |
| `resumeSessionAt` + `resumeDropsTurn` | 仅「重新生成」那一轮 | 在用户上一句之前分叉，上一版回答不进入上下文；不加 `forkSession`，session id 不变 |
| `forkSession: true` | 仅缓存保活 | 第 9 节 |
| `thinking` | `enabled` + 固定预算 + `display: "summarized"` | 预算是上限而非下限，极短的轮次仍可能不思考。`display` 独立于预算：Opus 4.6 默认返回摘要文本，4.7 起默认 `omitted`，此时思考照常发生（usage 中有 `thinking_tokens`，转写中有签名）但文本为空串 |
| `includePartialMessages` | `true` | 逐字 `text_delta` 与 `thinking_delta` 的前提 |
| `tools` | 九件白名单 | `preset` 会把 35 件内置工具的 schema 全部放入每轮固定前缀。统计四条线的全部转写，这一出口调用过的只有 `Bash`、`Read`、`WebSearch`、`WebFetch`、`ToolSearch` 五件；`Artifact`、计划模式、worktree、Cron、Workflow 等未被调用。数字见第 8 节 |
| `mcpServers` + `strictMcpConfig` | 代码内组装的显式清单 | 不修改 `.mcp.json`（终端窗口也读取它）。远端 server 的 URL 附加 `?profile=chat`，由服务端只登记聊天常用的工具，其余通过一个按名称调用的兜底工具访问 |
| `canUseTool` | 按前缀判断 | 第 7 节 |
| `abortController` | 15 分钟 | 网关另有 240 秒无增量的失联判定，第 6 节的心跳与之配合 |

两项不宜删减：`ToolSearch` 需要保留，MCP 工具数量较多时 CLI 会把一部分工具的 schema 延迟加载，只保留名称，模型通过 `ToolSearch` 取得 schema；删除它后这些工具无法调用。白名单替代 `preset` 之后，CLI 新增的内置工具不会自动进入清单，需要手工补充。

进程内 MCP server 的定义方式（用于只存在于本机的资源，如专用浏览器、本地文件、本机持有的密钥）：

```js
import { createSdkMcpServer, tool } from "@anthropic-ai/claude-agent-sdk";
import { z } from "zod";

export const watch = createSdkMcpServer({
  name: "watch",
  tools: [
    tool("preview", "预看一支视频并生成观看笔记", { url: z.string() }, async ({ url }) => ({
      content: [{ type: "text", text: await previewNote(url) }],
    })),
  ],
});
// 挂载：mcpServers: { kimi: {...}, watch }，与远端 server 在同一张表
```

## 6. 事件流的处理

| 消息 | 使用的字段 | 处理 |
| --- | --- | --- |
| `system` / `init` | `session_id`、`tools`、`mcp_servers` | 记录 session id 并回传网关；`tools.length` 用于核对目录件数 |
| `system` / `compact_boundary` | `compact_metadata.pre_tokens`、`post_tokens`、`trigger` | 向前端发送一条说明（例如「压缩：512K → 48K」）。压缩期间 SDK 不发送其他事件 |
| `system` / `api_retry` | `error`（字符串枚举）、`error_status`、`attempt`、`retry_delay_ms` | 向前端说明限流或上游错误及重试计划。`error` 是字符串，不是对象 |
| `stream_event` → `content_block_delta` | `delta.type` 为 `text_delta` 或 `thinking_delta` | 正文与思考各自作为增量转发 |
| `assistant` | `content[]` 中的 `tool_use` | 转发工具名与入参前 300 字符；同一条消息中若有整块 `thinking` 且之前没有收到过 `thinking_delta`，补发一次（重连或 SDK 整块返回时增量流为空） |
| `user` | `tool_result` | 按 `tool_use_id` 配对，转发结果预览；正文以 `Stop hook feedback:` 开头的是 Stop hook 的驳回，此时向前端发送 `reset`，清除已经流出的那一版回复 |
| `result` | `usage`、`subtype`、`num_turns` | 记账并发送一条 `usage` 增量 |

**上下文读数取三项之和。** `input_tokens + cache_read_input_tokens + cache_creation_input_tokens` 是本轮实际送入的量。同一 thread 连续两轮的实测：第一轮 in 1,226 + cache_creation 52,441 = 53,667；第二轮 in 169 + cache_read 52,441 + cache_creation 1,262 = 53,872。只读 `input_tokens` 会把第二轮显示为 169。

**多工具轮次的 usage 是各子轮的累加。** 一轮分块读取记忆共 9 个子轮时，`result.usage` 报 93.7 万，实际窗口 19.5 万。窗口读数取最后一个子轮的三项之和；输出与思考 token 仍取顶层总和。

**增量转发的节奏。** 每 200 毫秒或每 1,500 字符合并一次 POST 到网关。任务运行期间每 60 秒发送一次空 `parts` 以刷新网关侧的心跳时间：长 session 的冷 resume 在 prefill 阶段没有任何事件（2026-08-17 实测 195 秒后才出现第一个字），失联判定应只覆盖进程死亡的情形。静默超过 45 秒时向前端发送一条状态说明（例如「正在执行 /compact，已 120 秒」），最多四条。

**图片**通过流式输入发送，`content` 为数组，图片块在前：

```js
const prompt = (async function* () {
  yield {
    type: "user",
    message: { role: "user", content: [
      { type: "image", source: { type: "base64", media_type: "image/jpeg", data: b64 } },
      { type: "text", text },
    ] },
    parent_tool_use_id: null,
    session_id: "",
  };
})();
```

空的 `text` 块会被上游拒绝，只发图片时数组里只放图片块。图片像素不写入历史，数据库中的 user 事件只记录文字与一条「发送了一张图片」的标记。

## 7. 无人值守下的权限判定

`-p` 与 SDK 都没有权限弹窗。本实现的判定集中在 `canUseTool` 一处。

**allow 必须携带 `updatedInput`。** 类型声明中 `updatedInput?` 为可选，CLI 侧的运行时 schema 要求它是一个对象。返回 `{ behavior: "allow" }` 会使整个返回体不通过 union 校验（`ZodError: invalid_union`），该工具调用失败并重试一次。传入 `{}` 会清空入参。正确写法是原样带回：

```js
canUseTool: async (name, input) =>
  allowed(name) ? { behavior: "allow", updatedInput: input }
                : { behavior: "deny", message: "…" }
```

**`settingSources` 不含 `local`。** `.claude/settings.local.json` 中的 `permissions.allow` 规则在 settings 层放行工具，被放行的调用不会到达 `canUseTool`。该文件是交互窗口减少弹窗的配置，与本出口无关；本实现的规则是回调为唯一裁判。

**判据为前缀。** 内置工具已经由 `tools` 限定，回调对不带 `mcp__` 前缀的名称一律放行；MCP 工具按 server 前缀判断（`mcp__kimi__`、`mcp__watch__` 等）。SDK 新增内置工具时不需要修改回调。

**拒绝理由可以包含正确的调用方式。** 某个工具的全量返回约 100KB，超过 CLI 的字节上限后会被存为文件而无法读取（第 12 节）。回调拒绝不带分页参数的调用，理由中写明分页方式，模型按理由重新调用：

```js
if (name === "mcp__kimi__reentry" && input.coreOnly !== true && typeof input.offset !== "number") {
  return { behavior: "deny", message:
    "全量返回约 100KB 会被存为文件而无法读取。请分块：先 offset=0、limit=80000，再按块尾提示的 offset 连续读取到 [完]。" };
}
```

**安全边界。** 该配置包含 `Bash`、`Write`、`Edit`，工作目录是整个仓库，同时开放 `WebSearch` 与 `WebFetch`。用户粘贴的外部文本与模型抓取的网页属于同一类输入；其中的指令与 shell 之间只隔一层模型判断，没有人工确认。临时收回的方式是把内置工具集设为 `[]`。

**其他配置方式。** 本项目另一条只读的 SDK 线路使用 `tools: ["WebSearch", "WebFetch"]`、`allowedTools` 白名单、`permissionMode: "dontAsk"`，并以一个 `PreToolUse` hook 限制工具调用次数。Claude Code 2.1.259 起另有 `--permission-prompts none`，对会触发询问的调用一律拒绝并告知模型不要重试；本实现未使用，SDK 侧对应选项未查证。

## 8. 固定前缀的构成与削减

每轮请求都携带的固定部分由系统提示、内置工具 schema、CLAUDE.md、harness 自动记忆、MCP 工具目录组成。以下为三次测量。

**2026-08-21，逐项减法**（Opus 4.6，`result.usage` 三项之和，每组只改一处）：

| 配置 | 一轮输入 | 工具件数 |
| --- | --- | --- |
| A 当时的配置：preset 系统提示 + CLAUDE.md + 全部 MCP | 67,352 | 117 |
| B 替换 preset 系统提示 | 59,841 | 117 |
| C 去除工具目录 | 21,647 | 0 |
| D `settingSources: []`（无 CLAUDE.md、无 MCP） | 15,723 | 3 |
| E 新配置：MCP 走 chat 档，不挂外设工具 | 51,874 | 48 |

工具目录占 68%，系统提示约 7.5k。件数减少 59% 只带来 23% 的减量，因为被去除的工具 schema 较短，而保留的工具平均每件约 630 token。

**2026-09-07，内置工具目录**（Opus 4.6，同一方法）：

| 内置工具集 | 目录件数 | 一轮输入 |
| --- | --- | --- |
| `claude_code` preset | 35（总 83） | 50,220 |
| 九件白名单 | 9（总 57） | 29,545 |
| 仅五件曾调用的工具 | 5（总 53） | 28,959 |

减少 20,675（41%）。`Write`、`Edit`、`TaskOutput`、`TaskStop` 四件合计 586，予以保留。

**2026-09-09，分类读数**（Fable 5.1 tokenizer，`/context` 方法）：27.7k = 系统提示 5.7k + 内置九件 3.1k + 项目 CLAUDE.md 9.7k + harness 自动记忆 MEMORY.md 8.5k + compact 预留 3k。MCP 78 件共 31.6k 全部为延迟加载：窗口中只有名称，schema 在模型调用 `ToolSearch` 时才进入。设置 `CLAUDE_CODE_DISABLE_AUTO_MEMORY=1` 后为 18.4k；该 MEMORY.md 是终端窗口的工程索引，聊天出口不需要。

**测量方法。** 减法测量需要多轮真实调用。更直接的方法是使用 CLI 的 `/context` 命令：以流式输入先发送一条内容为 `/context` 的 user 消息并保持连接，随后的 assistant 消息带有 `context_usage` 字段（`SDKContextUsage`：按分类、每个 MCP 工具、memory 文件、skills 的 token 数，延迟加载的项单独标注）。该命令只做 token 计数，不生成文本。`getContextUsage()` 在 `/context` 之后调用会报 transport not ready，读消息上的字段即可。

```js
async function* input() {
  yield { type: "user", message: { role: "user", content: "/context" }, parent_tool_use_id: null, session_id: "" };
  await new Promise(() => {});   // 保持输入流，读到结果后 close
}
for await (const m of query({ prompt: input(), options })) {
  if (m.type === "assistant" && m.context_usage) { console.log(m.context_usage); break; }
}
```

**工具数量与窗口。** 官方文档所述工具超过 30 至 50 件时准确率下降，针对的是 schema 全部在窗口内的情形；延迟加载后每轮只携带名称。曾试验 `ENABLE_TOOL_SEARCH=auto` 使 79 件 schema 全部进入窗口，固定前缀增至 76k，随即撤回。工具可发现性的问题另行处理：在 CLAUDE.md 中按任务分组列出全部工具名称，配合 `ToolSearch` 使用。

**`Skill` 工具。** 白名单不含 `Skill` 时 skills 目录不进入前缀；加入后前缀增加 672（21,683 → 22,355），其中四分之三来自 CLI 自带的十二个 skill。本实现改为在 CLAUDE.md 中指明路径，需要时用 `Read` 读取对应的 SKILL.md。

**自定义系统提示的对照。** 本项目另一条 SDK 线路使用 `settingSources: []`、`systemPrompt: <人设字符串>`、`strictMcpConfig: true`，前缀中没有 CLAUDE.md 与 harness 系统提示。该线路曾省略 `settingSources`，结果每次启动都先执行 CLAUDE.md 中面向交互窗口的开场步骤，在 15 分钟上限内超时。省略等于全部加载，隔离需要显式写 `[]`。

## 9. 提示缓存

订阅侧同样命中提示缓存。2026-09-09 的连续几轮：

```
success · 117 字 · thinking 420 tok · ctx 191825 (cache_read 191528) · 21 s
success · 140 字 · thinking 292 tok · ctx 192506 (cache_read 191822) · 15 s
success ·  74 字 · thinking 169 tok · ctx 193050 (cache_read 192503) · 10 s
success ·  15 字 · thinking  57 tok · ctx 193953 (cache_read 193397) ·  5 s
```

19 万 token 的上下文，每轮新写入约一千，其余从缓存读取。维持这一状态涉及三点。

**系统提示包含 git 状态。** `claude_code` preset 的系统提示中有一段「This is the git status at the start of the conversation」（分支、工作树状态、最近三条 commit）。每个任务启动新的 CLI 进程时重新计算这一段，因此 `cwd` 所指仓库中的任何 commit 或文件变动都会使下一轮从第一个字节开始缓存未命中。2026-09-05 深夜的一次 commit 之后，下一轮 `cache_read 0 / cache_creation 713,349`，耗时 40 秒；订阅侧的代价是延迟，按 API 价格计算约 3.9 美元。对策是链路活跃期间不改动该仓库（测量脚本放在仓库之外）。`systemPrompt.excludeDynamicSections: true`（对应 `--exclude-dynamic-system-prompt-sections`）按文档会把「working directory, environment info, memory paths, git-repo flag」移入首条 user 消息；是否包含 git 状态段落未验证。

**缓存保活。** 缓存一小时过期，过期后下一轮需要重写整段上下文。本实现每 50 分钟以 `forkSession: true` 分叉一个临时 session 读取一次前缀：

```js
for await (const m of query({
  prompt: "(缓存保活，回复「·」一个字符即可，不需要思考)",
  options: { model, cwd, resume: sid, forkSession: true, maxTurns: 1, effort: "max",
             tools: [], settingSources: [], env },
})) { /* 按 message.id 去重后累计 cache_read / cache_creation */ }
```

- `forkSession` 不可省略。不加时是在原 session 上续写，保活消息会留在用户的对话记录中。分叉产生的转写（每次约 3.7MB）用后删除。
- 上限 12 次。缓存写入价格为读取的 12.5 倍（写 1.25、读 0.1），超过 12.5 次保活的费用高于一次重写。这项机制在费用上接近持平，作用是把长时间空闲后第一轮的等待从约三分钟降到秒级。
- 忙碌判定以最近 30 分钟为窗口。2026-08-27 白天 8 次保活中 6 次未命中：保活时机器空闲，写入后终端窗口的其他 session 把前缀挤出缓存，下一次再次全量写入；此类循环占两天保活费用的约 78%。判据改为「最近 30 分钟内是否有其他 session 写过转写」，有则跳过（跳过是本地扫描，不消耗 token 与次数配额）。
- 记账按 `message.id` 去重。同一次 API 调用的 assistant 内容分多条流式消息发送（thinking 一条、正文一条），每条携带同一份 usage 快照，逐条累加会重复计数。保活消息需写明期望的回复：发送单个「·」时模型会推断其含义并常返回空文本，CLI 随即注入「no visible output」重试，一次保活变成两轮。

**冷 resume 的等待。** 第 6 节的 60 秒心跳与 45 秒静默说明都针对这一情形：prefill 阶段 SDK 没有事件，网关侧的失联判定只能依赖心跳。

## 10. 登录态：存储、过期、续期与重新登录

**存储内容。** 钥匙串条目 `Claude Code-credentials` 的值是 JSON，`claudeAiOauth` 下有 `accessToken`、`refreshToken`、`expiresAt`、`refreshTokenExpiresAt`、`scopes`、`subscriptionType`、`rateLimitTier`。

**两个有效期。** access token 约 8 小时（2026-09-09 的一次续期：22:44 续至次日 06:44）。refresh token 另有到期时间（同日读数距当前约六天）。access token 到期后，CLI 在下一次请求时用 refresh token 换取新的一对并写回；refresh token 到期后无法续期，需要重新登录。

**只有 CLI 自身会续期。** 桌面应用与网页版使用各自的 OAuth，不读写 CLI 的钥匙串条目。若长时间没有任何 CLI 进程运行（本实现曾有连续十二天未启动 CLI 的情形），refresh token 到期，之后每次请求返回「Login expired · Please run /login」（2.1.206 起的报法），`/status` 的 Login 行显示「Expired」（2.1.210 起）。CLI 启动时距到期三天内会显示「Your login expires in 3 days · run /login to renew」（2.1.203 起）。

**重新登录。** 在终端运行 `claude`，执行 `/login` 完成浏览器授权；没有浏览器的环境（SSH 会话、容器）在浏览器中完成授权后把显示的 code 粘贴回终端。钥匙串在 SSH 会话中被锁定时，CLI 会把登录态写到 `~/.claude/.credentials.json`。重新登录后 daemon 不需要重启：每个任务启动时都从钥匙串读取当前值。

**本实现的自动续期。** daemon 在 access token 到期前 30 分钟，以终端 CLI 发起一次最小请求，由 CLI 自行完成刷新并写回：

```
claude -p ok --model haiku --max-turns 1 --no-session-persistence
```

`--no-session-persistence` 使该次请求不写转写、不可 resume。判定标准是钥匙串中 `expiresAt` 是否前移，进程退出码 0 不作为判定（CLI 只在到达自身的刷新时点后才刷新，未到时点时正常完成请求即退出）。失败按连续次数指数退避，1 分钟起、30 分钟封顶；未到刷新时点时退避到到期前两分钟。daemon 不直接使用 refresh token：刷新时该令牌会轮换，两个进程同时刷新会使其中一方持有已作废的令牌。

**额度读数。** 订阅额度只能由持有登录态的 Mac 上的 daemon 读取：`GET https://api.anthropic.com/api/oauth/usage`，请求头 `Authorization: Bearer <accessToken>` 与 `anthropic-beta: oauth-2025-04-20`。以 `limits[]` 数组为准（`kind=weekly_scoped` 为按模型的周额度），顶层的 `seven_day_opus` 实测为 null。每五分钟读取一次，每次重读钥匙串；失败按连续次数指数退避（1 分钟起、30 分钟封顶）。

**长期令牌。** 无法进行浏览器登录的环境可使用 `claude setup-token` 生成一年期的 OAuth 令牌，设为 `CLAUDE_CODE_OAUTH_TOKEN`。官方说明：该令牌只能发起模型请求，不能建立 Remote Control 会话或获取 claude.ai connectors，本地配置的 MCP server 不受影响；bare 模式不读取该变量。

## 11. 部署位置：Mac 与 VPS

本实现全部运行在一台 Mac 上，以下几处依赖这一点：

- 登录态在 macOS 钥匙串中，续期由本机的终端 CLI 完成；
- daemon 是 gui 级 LaunchAgent，图形登录之后才启动。开启 FileVault 且未设自动登录的机器在重启后停在解锁界面，agent 不会启动，链路处于离线状态直到有人解锁；
- 进程内 MCP server 使用的资源在本机（专用浏览器实例、本地文件、本机持有的密钥）；
- 转写文件在本机 `~/.claude/projects/` 下，resume 只能在这台机器上进行；
- 额度读数使用钥匙串中的 access token。

在 VPS 上运行同一 daemon 时，以上各项需要改写。本项目尚未部署 VPS 版的桥；下面是已经确定的差异（其中认证方式一项已在本项目 VPS 上的另几条 SDK 线路中运行）：

| 项 | Mac | VPS |
| --- | --- | --- |
| 认证 | 钥匙串登录态 | 二选一：`CLAUDE_CODE_OAUTH_TOKEN`（`claude setup-token` 生成，一年期，仍计入订阅额度），或 `ANTHROPIC_API_KEY`（按量计费） |
| 续期 | 需要（第 10 节） | 不需要：长期令牌一年内有效，API key 不过期 |
| 额度读数 | `/api/oauth/usage` | 长期令牌能否调用该接口未验证；API key 模式改为按 `usage` 逐轮记账 |
| 缓存保活 | 订阅侧费用接近持平 | API 模式下每次保活按读取计费，默认应关闭 |
| 进程内 MCP | 按本机资源挂载 | 只挂载 VPS 上存在的资源 |
| 转写与 session | 本机 | VPS 本地；两台机器的 session 不互通，同一 thread 不能在两端交替 resume，前端需为两端分别维护 thread |
| 运行用户 | 当前用户 | 不以 root 运行：该配置含 `Bash`，应使用独立用户、独立的仓库 clone 与独立的凭据文件 |
| 任务分发 | 单一 daemon | 网关需按 daemon 标识与模型别名分发任务，避免两端互相领取 |

本项目 VPS 上的另几条 SDK 线路（夜间自主运行的只读线路）的认证写法：启动前删除 `ANTHROPIC_API_KEY`、`ANTHROPIC_AUTH_TOKEN`、`OPENROUTER_API_KEY`，然后要求 `CLAUDE_CODE_OAUTH_TOKEN` 或 `~/.claude/.credentials.json` 存在，否则直接报错退出，不回落到其他计费方式。这些线路使用 `settingSources: []` 与自定义系统提示（第 8 节末尾）。

API key 模式下另需三项：按 `usage` 逐轮计算费用并设每日上限（超过时拒绝并说明，不静默）；上下文超过阈值时在下一轮前执行 `/compact`；费用记录落库，daemon 重启后不丢失。

## 12. 上下文窗口与工具返回的上限

**窗口以实际拒绝为准。** 订阅侧 Opus 4.6 的裸 id 上限为 200k：一条 thread 在 19.5 万 token 时被拒绝「Prompt is too long」，且该模型此时无法完成 `/compact`；`claude-opus-4-6[1m]` 变体实测 236k 一次通过。Fable 5.1 原生 1M，裸 id 实测 257k 通过。接近上限时的处理：切换到窗口更大的模型对同一 session 发送 `/compact`（session id 不变），完成后切回。斜杠命令作为普通 prompt 发送，由 CLI 识别。

**MCP 工具返回有两道上限。** `MAX_MCP_OUTPUT_TOKENS` 提高到 100k 之后，另有一道约 100KB 的字节上限不受环境变量影响（98.8KB 内联返回，100.4KB 被存为文件；100k 与 200k 两档相同）。存为文件的内容是单行 JSON，`Read` 按行分页无法读取一行 8 万 token。处理方式是服务端分块：工具接受 `offset/limit` 字符窗口，每块头部报告进度与下一块的 offset，客户端每块 80,000 字符读取到 `[完]`；实测全量 101,878 字符分两块读完。第 7 节的拒绝理由用于引导模型使用分块调用。

**回退到压缩前的状态。** `resumeSessionAt` 加 `forkSession` 无法越过压缩层（分叉得到的仍是摘要视图）。可行的方法是转写文件操作：截取原转写至 `compact_boundary` 前一行，逐行改写 `sessionId` 为新 uuid，写回 `~/.claude/projects/<目录名>/`，把 thread 指向新 id。实测 6,359 行 / 14.3MB 恢复成功，前提是模型窗口足够。

## 13. 故障对照

| 现象 | 原因 | 处理 |
| --- | --- | --- |
| 每轮报错，日志 `401 OAuth access token has expired` | 登录态过期 | 终端运行 `claude` 重新登录；daemon 不需重启 |
| 工具调用失败，日志 `ZodError: invalid_union` | `canUseTool` 的 allow 未携带 `updatedInput` | `{ behavior: "allow", updatedInput: input }` |
| 新增的 MCP 工具被拒绝，旧工具正常 | 旧工具被 settings 层的 allow 规则放行，新工具才到达回调 | `settingSources` 去掉 `local`，使回调成为唯一裁判 |
| 某模型返回 `400 … does not support this model; version X or newer is required`，usage 全零 | SDK 内置的 CLI 版本过旧；`result.subtype` 仍为 `success`，错误文本在正文中 | 升级 `@anthropic-ai/claude-agent-sdk`；探测时以是否返回文本为准 |
| 思考面板为空 | usage 中 `thinking_tokens` 为 0 时模型未思考；大于 0 而文本为空时是 `display` 默认 `omitted`（4.7 起） | `thinking.display: "summarized"`；转写中的整块 thinking 可作补充来源 |
| 一轮数分钟无输出 | 冷 resume 的 prefill、compact，或上游 `api_retry` | 先查日志中的 `api_retry`，再查转写末条 usage 的 `cache_read` 是否为 0 |
| 一次 commit 之后下一轮延迟约 40 秒 | 系统提示中的 git 状态变化，整段缓存未命中 | 第 9 节 |
| 保活日志的 cache_read / cache_creation 为整数倍 | 同一轮多条流式消息的 usage 快照重复累加 | 按 `message.id` 去重 |
| 回复出现两遍并首尾相接 | Stop hook 驳回后模型重写，第一版已经流出 | 收到 `Stop hook feedback:` 的 user 消息时发送 `reset` |
| 工具返回被存为文件，模型无法读取 | 约 100KB 字节上限 | 服务端分块；回调拒绝不带分页参数的调用 |
| 上下文读数波动 | 多工具轮次的 `result.usage` 为子轮累加 | 取末轮三项之和 |
| 日志持续出现额度读取 429 | 读取失败无退避 | 指数退避；连续 429 时先检查退避逻辑 |
| 续期探针失败，stderr 为 `no stdin data received in 3s` 或 settings 中的 permission 规则报错 | 探针进程的 stdin 是未关闭的管道；探针加载了项目与 local 设置，新版 CLI 在启动时校验 allow 规则 | 探针以 `stdio: ["ignore", …]` 启动，并加 `--setting-sources user` |

## 14. 版本

- SDK 内置的 CLI 与终端安装的 CLI 是两份文件（0.3.258 内置 2.1.258，本机终端 2.1.266）。新模型 id 需要新版 CLI（`claude-fable-5-1` 要求 ≥ 2.1.251），升级对象是 npm 包，不是 `claude update`。
- `--bare` 将成为 `-p` 的默认，且 bare 模式不读取 OAuth 登录态与 `CLAUDE_CODE_OAUTH_TOKEN`。当前 SDK 不传该参数；升级 SDK 后先运行第 3 节的自检。
- 白名单替代 `preset` 后新的内置工具不会自动加入；升级后核对 `system/init` 的 `tools` 列表。
- 升级前以 `persistSession: false` 开一条测试 session，核对事件解析、thinking、权限回调、MCP 连接，通过后再切换。0.3.233 对未知模型 id 返回 400 而 `subtype` 仍为 `success` 的情况即在此类测试中发现。

## 15. 小结

Agent SDK 是 Claude Code 的库形态：`query()` 启动包内的 CLI 二进制，固定以双向 stream-json 通信，不传 `-p`；系统提示、hooks、进程内 MCP 与权限回调经同一通道上的控制协议传递，其余选项一对一映射为命令行参数。订阅计费的前提是子进程环境不含 API key 且不使用 `--bare`。保留 harness 的配置是 `preset: "claude_code"` 加 `settingSources: ["user", "project"]`；固定前缀的削减对象主要是工具目录（内置工具白名单、MCP 按档登记、schema 延迟加载）；无人值守的权限判定集中在 `canUseTool`（allow 携带 `updatedInput`）；提示缓存的维持依赖仓库静止、`forkSession` 保活与按 `message.id` 记账。登录态由 CLI 自身续期，长期无 CLI 进程运行时需要重新登录；VPS 部署需改用长期令牌或 API key，并重做续期、额度、session 归属与任务分发。

## 参考文档

| 主题 | 出处 |
| --- | --- |
| Agent SDK 概览（与 CLI 的关系、第三方限制） | code.claude.com/docs/en/agent-sdk/overview |
| TypeScript SDK 参考（`Options`、`SDKContextUsage`、`PermissionResult`） | code.claude.com/docs/en/agent-sdk/typescript；包内 `sdk.d.ts` |
| 非交互模式（`--bare`、`system/api_retry`、`--permission-prompts`） | code.claude.com/docs/en/headless |
| 认证（凭据优先级、存储位置、续期、`claude setup-token`） | code.claude.com/docs/en/authentication |
| CLI 参数 | code.claude.com/docs/en/cli-reference |
| 订阅政策 | support.claude.com/en/articles/15036540 |
| 提示缓存 | platform.claude.com/docs/en/build-with-claude/prompt-caching |
| thinking 的 `display` 字段与跨轮保留 | platform.claude.com/docs/en/build-with-claude/thinking |

## 许可

本仓库（正文与代码）以 MIT 许可发布，见 [LICENSE](LICENSE)。
