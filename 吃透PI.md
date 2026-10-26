# 吃透 PI：源码学习手册

这份文档不是“重点文件列表”，而是带着你从代码里学习 PI 的方法。

你的最终目标不是背 PI 的目录结构，而是能自己做出一个简化版 Coding Agent 简历项目。因此每一章都按同一个节奏来：

1. 先跑一个可观察行为。
2. 再从入口追到核心函数。
3. 在代码里验证一个设计事实。
4. 写出你自己的理解。
5. 抽象成你简历项目里要实现的能力。

## 0. 目标项目

最终做一个 `mini-coding-agent`，第一版只需要完整闭环：

- CLI：`agent run "任务描述"`
- 一个 LLM Provider
- 流式输出
- 工具调用：读文件、列目录、搜索、编辑文件、执行 shell
- JSONL session 记录
- 高风险操作确认
- README、架构图、演示任务

PI 是参考对象，不是照抄对象。学习 PI 时，每看懂一个设计，都要问：我的项目要不要这个能力？要的话能不能简化？

## 1. 先把项目跑起来

如果你直接输入 `pi`，PowerShell 提示 `The term 'pi' is not recognized`，说明系统里没有全局 `pi` 命令。克隆源码不会自动注册命令。

检查：

```powershell
Get-Command pi
```

在源码仓库里学习，优先用：

```powershell
npm install --ignore-scripts
.\pi-test.ps1
```

也可以用：

```powershell
.\pi-test.bat
```

Git Bash、WSL、macOS、Linux 用：

```bash
npm install --ignore-scripts
./pi-test.sh
```

如果你想全局使用发布版：

```powershell
npm install -g --ignore-scripts @earendil-works/pi-coding-agent
```

安装后还找不到命令，检查 npm 全局安装前缀：

```powershell
npm config get prefix
```

Windows 通常把这个目录本身加入 `PATH`；macOS/Linux 通常把该目录下的 `bin` 加入 `PATH`。改完重开终端。

## 2. 学习前的读代码规则

不要一上来逐文件读。每次只追一个用户行为。

推荐你每次读代码时都写四行笔记：

```text
我追的行为：
入口函数：
关键状态：
我自己的项目要怎么实现：
```

遇到看不懂的地方，不要停在局部函数里硬啃。先回答三个问题：

- 这个函数的输入是什么？
- 它改变了什么状态？
- 它把控制权交给谁？

### 2.1 如果你不会打断点

先不用觉得必须会复杂调试。打断点本质上只做一件事：让程序运行到某一行时停住，然后你看变量、看调用栈、一步步往下走。

在 VS Code 里最小操作：

1. 打开一个 `.ts` 文件。
2. 在行号左边点一下，出现红点，这就是断点。
3. 按 `F5` 启动调试。
4. 程序停住后，看左侧 `VARIABLES` 和 `CALL STACK`。
5. `F10` 是执行下一行，但不进入函数内部。
6. `F11` 是进入当前这一行调用的函数内部。
7. `Shift+F11` 是跳出当前函数。
8. 再按 `F5` 是继续运行到下一个断点。

建议先在最容易命中的地方打断点：

- `packages/coding-agent/src/cli.ts` 的 `main(process.argv.slice(2))`
- `packages/coding-agent/src/main.ts` 的 `const parsed = parseArgs(args)`
- `packages/coding-agent/src/main.ts` 的 `let appMode = resolveAppMode(...)`
- `packages/coding-agent/src/modes/print-mode.ts` 的 `await session.prompt(...)`
- `packages/agent/src/agent.ts` 的 `async prompt(...)`
- `packages/agent/src/agent-loop.ts` 的 `export async function runAgentLoop(...)`

如果你不想先配置 VS Code 调试器，也可以用“穷人断点”：

```ts
console.error("到这里了", { args });
```

或者：

```ts
debugger;
```

`debugger;` 只有在调试模式启动时才会停住。普通命令行运行时通常不会停。

### 2.2 给 VS Code 配一个最小调试入口

这个仓库的 `pi-test.ps1` 本质上会执行：

```powershell
node_modules/.bin/tsx.cmd packages/coding-agent/src/cli.ts
```

所以可以在 VS Code 新建 `.vscode/launch.json`，先用 `--help` 练习断点。它不需要 API key，也不会真的请求模型。

```json
{
  "version": "0.2.0",
  "configurations": [
    {
      "type": "node",
      "request": "launch",
      "name": "Debug PI help",
      "runtimeExecutable": "${workspaceFolder}/node_modules/.bin/tsx.cmd",
      "runtimeArgs": ["--tsconfig", "${workspaceFolder}/tsconfig.json"],
      "program": "${workspaceFolder}/packages/coding-agent/src/cli.ts",
      "args": ["--help"],
      "cwd": "${workspaceFolder}",
      "console": "integratedTerminal",
      "skipFiles": ["<node_internals>/**", "${workspaceFolder}/node_modules/**"]
    }
  ]
}
```

第一轮练习：

1. 在 `cli.ts` 的 `main(...)` 行打断点。
2. 选择 `Debug PI help`。
3. 按 `F5`。
4. 停住后看 `process.argv`。
5. 按 `F11` 进入 `main.ts`。

第二轮练习，把 `args` 改成：

```json
"args": ["-p", "Say hello"]
```

这次可能因为没有模型或 API key 失败，但没关系。你的目标是看清楚它如何从 CLI 走到 `runPrintMode()` 和 `session.prompt()`，不是先让模型成功回答。

### 2.3 新手追代码时不要用“猜”

你每次卡住时，优先用这几个搜索命令：

```powershell
rg "functionName" packages
rg "class Agent" packages
rg "session.prompt" packages
rg "runAgentLoop" packages
```

看到一个函数名时，不要只读它附近几行。按这个顺序看：

1. 谁调用它？
2. 它调用谁？
3. 它读了哪些参数？
4. 它改了哪些状态？
5. 它返回什么？

新手最容易犯的错是盯着一个复杂函数从上往下硬读。正确做法是先找主干，再回头看细节。

## 3. 任务卡 1：理解 monorepo 分层

目标：知道 PI 为什么拆成 `ai`、`agent`、`coding-agent`、`tui`。

先看根目录：

```powershell
Get-Content package.json
Get-ChildItem packages
```

你要在代码里确认：

- `packages/ai` 只关心模型、消息、Provider、流式协议。
- `packages/agent` 只关心 Agent 状态机和工具循环。
- `packages/coding-agent` 是具体应用，把模型、工具、session、CLI、TUI 组合起来。
- `packages/tui` 是终端 UI 组件库。

读法：

1. 打开根 `package.json`，看 `workspaces`。
2. 打开每个 package 的 `package.json`，只看 `name`、`description`、`dependencies`。
3. 画出依赖方向：`coding-agent -> agent + ai + tui`。

你要写出的结论：

```text
PI 的核心不是一个大 CLI 文件，而是四层：
LLM 抽象层 -> Agent Runtime -> Coding Agent 应用层 -> TUI 展示层。
我的简历项目第一版只需要前三层，TUI 后置。
```

## 4. 任务卡 2：追 `pi -p "hello"` 的启动链路

目标：知道一个命令从终端输入后，如何进入 Agent。

先跑一个最小行为：

```powershell
.\pi-test.ps1 -p "Say hello"
```

如果你还没有配置模型或 key，可能会失败。失败也没关系，先看它失败到哪里。

追代码顺序：

1. `packages/coding-agent/src/cli.ts`
2. `packages/coding-agent/src/main.ts`
3. `packages/coding-agent/src/modes/print-mode.ts`

具体读法：

1. 在 `cli.ts` 找 `main(process.argv.slice(2))`。
2. 在 `main.ts` 找 `parseArgs(args)`。
3. 在 `main.ts` 找 `resolveAppMode(...)`，确认 `-p` 为什么进入 print mode。
4. 在 `main.ts` 找 `createSessionManager(...)`，确认 session 在运行前就创建。
5. 在 `main.ts` 找 `createAgentSessionRuntime(...)`，确认 runtime 在模式启动前已经准备好。
6. 在 `main.ts` 找 `runPrintMode(...)`，跳到 `print-mode.ts`。
7. 在 `print-mode.ts` 找 `session.prompt(...)`。

你要回答：

- CLI 参数在哪里解析？
- `-p` 为什么不是 interactive？
- `session.prompt()` 是真正进入 Agent 的边界吗？
- print mode 为什么适合学习主链路？

小实验：

在 `print-mode.ts` 里只读代码，不改代码，找出它最后如何拿到 assistant 的文本输出。

你自己的项目结论：

```text
我的第一版也应该先做 print mode，不先做复杂交互 UI。
CLI 只负责参数、配置、session 创建和调用 agent.run。
```

## 5. 任务卡 3：理解 `createAgentSession()` 到底创建了什么

目标：知道应用层如何把模型、工具、session、Agent Runtime 粘起来。

追代码：

1. `packages/coding-agent/src/core/sdk.ts`
2. `packages/coding-agent/src/core/agent-session-services.ts`
3. `packages/coding-agent/src/core/agent-session.ts`

具体读法：

1. 在 `sdk.ts` 找 `createAgentSession(...)`。
2. 看它如何创建 `AuthStorage`、`ModelRegistry`、`SettingsManager`、`SessionManager`。
3. 找 `new Agent({ ... })`，看传进去的参数。
4. 特别看 `streamFn`，它是 PI 调模型的出口。
5. 看 `convertToLlmWithBlockImages`，理解为什么应用层需要把自己的消息格式转换成 LLM 消息格式。
6. 找 `new AgentSession({ ... })`，看它包住了哪些东西。

你要回答：

- `Agent` 负责什么？
- `AgentSession` 比 `Agent` 多了什么？
- 为什么不是 CLI 直接调用 `Agent.prompt()`？
- `streamFn` 为什么作为参数注入，而不是写死在 Agent 里？

关键理解：

```text
Agent 是通用状态机。
AgentSession 是 Coding Agent 应用层，负责 session、工具、模型选择、扩展、压缩、bash 等业务。
```

你自己的项目结论：

```text
我的项目里也要区分 AgentCore 和 AppSession。
AgentCore 不知道文件系统、不知道 CLI、不知道 session 文件。
AppSession 才负责把工具、配置、历史记录接进去。
```

## 6. 任务卡 4：吃透 Agent 核心循环

目标：真正理解 Coding Agent 的心脏。

追代码：

1. `packages/agent/src/agent.ts`
2. `packages/agent/src/agent-loop.ts`
3. `packages/agent/src/types.ts`

具体读法：

1. 在 `agent.ts` 找 `class Agent`。
2. 找 `prompt(...)`。
3. 跟到 `runPromptMessages(...)`。
4. 跟到 `runAgentLoop(...)`。
5. 在 `agent-loop.ts` 找 `runLoop(...)`。
6. 在 `runLoop(...)` 里分清外层循环和内层循环。
7. 找 `streamAssistantResponse(...)`，确认哪里调用 LLM。
8. 找 `executeToolCalls(...)`，确认哪里执行工具。
9. 找 `createToolResultMessage(...)`，确认工具结果如何变回消息。

把这段伪代码抄进你的笔记，然后对照源码验证：

```text
append user message
emit user message events
while true:
  call LLM with current context
  append assistant message
  if assistant has tool calls:
    execute tools
    append tool result messages
    continue
  if follow-up message exists:
    append follow-up
    continue
  stop
```

你要回答：

- 为什么工具调用后还要再请求一次 LLM？
- `toolResult` 如果不追加进上下文，会发生什么？
- `message_start`、`message_update`、`message_end` 为什么比直接返回字符串更适合 UI？
- PI 为什么支持 parallel tool execution？

小实验：

只读 `executeToolCallsParallel(...)` 和 `executeToolCallsSequential(...)`，写一句话说明两者差别。

你自己的项目结论：

```text
我的第一版 AgentLoop 可以先只做 sequential tool execution。
事件流必须保留，因为后面无论 CLI、Web UI 还是 TUI 都能复用。
```

## 7. 任务卡 5：理解消息类型

目标：知道模型看到的上下文和应用内部消息不是一回事。

追代码：

1. `packages/ai/src/types.ts`
2. `packages/agent/src/types.ts`
3. `packages/coding-agent/src/core/messages.ts`

具体读法：

1. 在 `packages/ai/src/types.ts` 找 `UserMessage`、`AssistantMessage`、`ToolResultMessage`、`Message`。
2. 在 `packages/agent/src/types.ts` 找 `AgentMessage`。
3. 在 `messages.ts` 找 `BashExecutionMessage`、`CustomMessage`、`convertToLlm(...)`。

你要确认：

- LLM 层只认识 `user`、`assistant`、`toolResult`。
- coding-agent 内部还会有 `bashExecution`、`custom`、`branchSummary`、`compactionSummary`。
- `convertToLlm(...)` 是应用内部消息进入模型上下文的过滤器。

你要回答：

- 为什么 `!!command` 的 bash 输出可以不进入上下文？
- 为什么 branch summary 和 compaction summary 最后被转换成 user message？
- 为什么不要把所有 UI 消息都发给模型？

你自己的项目结论：

```text
我的项目也要区分 InternalMessage 和 LlmMessage。
不是所有界面事件、日志、状态都应该进入模型上下文。
```

## 8. 任务卡 6：读懂一个工具，以 `read` 为例

目标：知道一个工具从 schema 到执行再到结果展示的完整结构。

追代码：

1. `packages/coding-agent/src/core/tools/index.ts`
2. `packages/coding-agent/src/core/tools/read.ts`
3. `packages/coding-agent/src/core/tools/tool-definition-wrapper.ts`

具体读法：

1. 在 `tools/index.ts` 找 `createAllToolDefinitions(...)`。
2. 跳到 `createReadToolDefinition(...)`。
3. 先看 `readSchema`，理解模型必须传什么参数。
4. 再看 `description` 和 `promptGuidelines`，理解这些文字会影响模型如何用工具。
5. 再看 `execute(...)`，只追文本文件路径，不先管图片。
6. 找 `truncateHead(...)`，理解为什么读取大文件要截断。
7. 看返回值 `{ content, details }`，确认什么会回给模型，什么只是给 UI。

你要回答：

- `path`、`offset`、`limit` 是为了解决什么问题？
- 为什么 read 工具不直接无限返回整个文件？
- `details` 和 `content` 的区别是什么？
- `promptGuidelines` 为什么也是工具设计的一部分？

小实验：

设计你自己的 `read_file` 工具接口：

```ts
type ReadFileInput = {
  path: string;
  offset?: number;
  limit?: number;
};
```

写下：

- 最大返回多少行？
- 超过限制如何提示模型继续读取？
- 路径是否允许读工作区外？

## 9. 任务卡 7：读懂危险工具，以 `bash` 和 `edit` 为例

目标：理解 Coding Agent 最容易出事故的两个能力如何被约束。

读 `bash`：

1. 打开 `packages/coding-agent/src/core/tools/bash.ts`。
2. 找 `bashSchema`。
3. 找 `createLocalBashOperations(...)`。
4. 找 `spawn(...)`。
5. 找 timeout、abort、output truncation 相关逻辑。

你要回答：

- shell 命令在哪个 cwd 执行？
- stdout/stderr 如何被收集？
- 超时如何处理？
- 输出太长如何处理？
- abort 时如何杀掉进程？

读 `edit`：

1. 打开 `packages/coding-agent/src/core/tools/edit.ts`。
2. 找 `editSchema`。
3. 看 `oldText/newText` 为什么要求精确匹配。
4. 找 `withFileMutationQueue(...)`。
5. 找 `generateUnifiedPatch(...)`。

你要回答：

- 为什么 edit 不让模型直接给整文件内容？
- 为什么多个 edit 要在原始文件上匹配，而不是依次匹配？
- 为什么写文件要排队？
- 为什么返回 diff/patch 对用户和模型都有用？

你自己的项目结论：

```text
我的第一版必须对 shell 和 edit 加确认。
edit 先做精确替换，不做“让模型重写整个文件”的接口。
shell 输出必须截断，并保存完整输出路径或给出继续查看方式。
```

## 10. 任务卡 8：读懂 session 为什么是 JSONL

目标：理解对话历史如何持久化，以及为什么 session 不只是 messages 数组。

追代码：

1. `packages/coding-agent/src/core/session-manager.ts`
2. `packages/coding-agent/src/core/messages.ts`

具体读法：

1. 在 `session-manager.ts` 找 `SessionHeader`。
2. 找 `SessionEntry` union。
3. 找 `appendMessage(...)`。
4. 找 `buildSessionContext(...)`。
5. 找 `buildContextEntries(...)`。
6. 找 `branch(...)` 和 `branchWithSummary(...)`。

你要确认：

- 文件第一行是 session header。
- 后面每一行是一个 entry。
- entry 有 `id` 和 `parentId`，所以 session 是树，不是简单数组。
- `buildSessionContext(...)` 会把树上的当前分支转换成要给模型的 messages。

你要回答：

- JSONL 相比一个大 JSON 有什么好处？
- 为什么 append-only 对 agent session 重要？
- 为什么历史记录和发给模型的上下文不是一回事？
- 分支能力为什么依赖 `parentId`？

小实验：

为自己的项目先设计简化版：

```ts
type SessionHeader = {
  type: "session";
  id: string;
  cwd: string;
  createdAt: string;
};

type SessionEntry =
  | { type: "message"; id: string; timestamp: string; message: LlmMessage }
  | { type: "tool"; id: string; timestamp: string; toolName: string; result: unknown };
```

第一版可以不做树，只做线性 JSONL。等核心 AgentLoop 稳定后再加分支。

## 11. 任务卡 9：理解 LLM Provider 抽象

目标：知道 PI 如何让不同模型 Provider 看起来像同一种接口。

追代码：

1. `packages/ai/src/types.ts`
2. `packages/ai/src/models.ts`
3. `packages/ai/src/providers/all.ts`
4. 任选一个 API 实现，例如 `packages/ai/src/api/openai-responses.ts`

具体读法：

1. 在 `types.ts` 找 `Model`。
2. 找 `Context`。
3. 找 `AssistantMessageEvent`。
4. 找 `StreamFunction`。
5. 在具体 API 文件中找 `streamSimple(...)`。

你要回答：

- PI 的统一接口要求 Provider 返回什么？
- 流式事件为什么要统一成 `AssistantMessageEvent`？
- 不同 Provider 的 tool call 格式在哪里被抹平？
- `model` 里为什么要记录 `api`、`provider`、`baseUrl`、`contextWindow`？

你自己的项目结论：

```text
我的项目第一版只实现一个 Provider，但也要定义 Provider 接口。
这样第二版增加 Anthropic/OpenAI-compatible 不会重写 AgentLoop。
```

## 12. 任务卡 10：最后再读 TUI

目标：理解 UI 是如何消费 Agent 事件的，而不是先陷进 UI 细节。

追代码：

1. `packages/coding-agent/src/modes/interactive/interactive-mode.ts`
2. `packages/coding-agent/src/modes/interactive/components/assistant-message.ts`
3. `packages/coding-agent/src/modes/interactive/components/tool-execution.ts`
4. `packages/tui/src/tui.ts`

具体读法：

1. 在 `interactive-mode.ts` 找 `session.subscribe(...)`。
2. 看它如何响应 `message_start`、`message_update`、`message_end`。
3. 看 tool execution event 如何变成 UI 组件。
4. 不要一开始读所有快捷键和 selector。

你要回答：

- 为什么 UI 不直接轮询 agent state？
- streaming 时 assistant message 为什么需要 partial update？
- tool call 为什么有开始、更新、结束三个阶段？

你自己的项目结论：

```text
第一版 CLI 只打印事件。
第二版 Web UI/TUI 可以复用同一套 AgentEvent。
```

## 13. 你的简历项目实现顺序

不要等完全读懂 PI 再动手。读一部分，实现一部分。

### 第一步：实现无工具 Agent

目标：

- CLI 收到 prompt。
- 调一个 Provider。
- 流式输出 assistant 文本。

对应 PI 学习任务：

- 任务卡 2
- 任务卡 3
- 任务卡 4 的 `streamAssistantResponse`

### 第二步：实现工具调用

目标：

- 模型能调用 `read_file`。
- 工具结果能回填。
- 模型能基于工具结果继续回答。

对应 PI 学习任务：

- 任务卡 4
- 任务卡 6

### 第三步：实现危险工具

目标：

- `edit_file` 精确替换。
- `run_shell` 支持 timeout 和确认。

对应 PI 学习任务：

- 任务卡 7

### 第四步：实现 session

目标：

- 每轮消息写入 JSONL。
- 支持 resume。

对应 PI 学习任务：

- 任务卡 8

### 第五步：整理成作品

目标：

- README 讲清楚架构。
- 录一个 demo。
- 准备面试讲解。

README 必须能回答：

- AgentLoop 怎么工作？
- 工具 schema 如何约束模型？
- session 为什么用 JSONL？
- shell/edit 有什么安全控制？
- 和 PI 相比你删掉了什么、保留了什么、为什么？

## 14. 每次读完代码后的自检

如果只是“看过文件”，不算学会。

每个任务卡结束后，你要能做到：

- 用 5 句话解释这个模块。
- 画出它和上游、下游的关系。
- 找到一个可以简化到自己项目里的设计。
- 写一个最小伪代码。
- 明确一个暂时不做的高级能力。

## 15. 不要一开始做的事

- 不要逐文件读完整仓库。
- 不要一开始做复杂 TUI。
- 不要一开始支持十几个 Provider。
- 不要一开始做 extensions/skills/package manager。
- 不要照抄 PI 的目录结构。
- 不要只包一层第三方 Agent 框架。

真正有价值的是：你能从 PI 读出 Agent Runtime 的核心机制，然后自己实现一个简化但完整、可运行、可解释的 Coding Agent。
