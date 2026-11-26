# 实践课 5 报告：Pi Agent 工具调用与会话恢复

## 基本信息

| 项目 | 内容 |
| --- | --- |
| 姓名拼音 | maoenqi |
| GitHub 登录名 | Mao123415 |
| 个人作业仓库 | [Mao123415/-5-maoenqi](https://github.com/Mao123415/-5-maoenqi) |
| 小组代码仓库 | [MangDui/pi-hunnu-2026](https://github.com/MangDui/pi-hunnu-2026) |
| 实验环境 | Windows + PowerShell，本地构建 Pi v1.5.2，保留实践 04 的 `[P4]` 日志 |
| 实验模型 | `deepseek / deepseek-flash`，思考模式 off |
| 实验日期 | 2026 年 11 月 26 日 |

本次实验先验证一次输入如何触发工具执行和后续模型请求，再比较恢复会话与新建会话发送的聊天内容。结果以本人实际会话消息为依据，两张图使用本人更新的 draw.io 导出图片。

报告导读：[A 阶段运行证据](task-a-evidence.json) · [B 阶段运行证据](task-b-evidence.json) · [图一源文件](img/figure-1-tool-call.drawio) · [图二源文件](img/figure-2-session-compare.drawio)

## A. 单轮交互与工具调用

### A-1 准备交接单与启动环境

- 实验日期：2026 年 11 月 26 日，Asia/Shanghai。
- 源码与运行目录：`D:\02-pi-agent-fork\pi-hunnu-2026`。
- 源码提交：`c3cc91632d0a1950fb122605b190233f2aa0006e`。调用路径按本地工作区源码核对；工作区已有实践 04 的 `[P4]` 日志语句。
- 使用该目录已经构建的 `packages/coding-agent/dist/cli.js`，版本为 `1.5.2`。
- 在新交互会话输入 `/model`，选择 `deepseek / deepseek-flash`（选择器显示 DeepSeek V4.1 Flash），思考模式为 off。

启动命令如下。其中 `--offline` 禁止启动时的联网更新，模型推理仍会联网；`--tools read` 将可用工具限定为 read，便于观察本轮行为。

```powershell
cd D:\02-pi-agent-fork\pi-hunnu-2026
node .\packages\coding-agent\dist\cli.js --offline --no-extensions --no-mcp --no-skills --no-prompt-templates --tools read --approve --name practice-05-A-retry
```

在运行目录中准备两行 `handoff.txt`，以随机生成的标识替换文档示例中的 `river-731`：

```text
待办：补充 README 的启动说明
启动命令：npm run demo -- --profile group-a9b32a
```

这里的启动命令是供模型读取的交接内容，本实验没有执行这条命令。提交的问题不包含命令或随机标识：

> 请用 read 工具读取当前目录的 handoff.txt，原样告诉我待办和启动命令。

### A-2 实验过程与观察

这一次用户输入触发了 **2 次模型调用、1 次 read 工具执行**。

| 顺序 | 会话记录 | 观察结果 |
| --- | --- | --- |
| 1 | user | 提交上述读取问题，没有透露启动命令。 |
| 2 | assistant，`stopReason=toolUse` | 返回 `I'll read the file.`，以及一个 read 工具调用。 |
| 3 | toolResult，`toolName=read` | 返回文件的两行内容，`isError=false`，工具耗时记录为 4 ms。 |
| 4 | assistant，`stopReason=stop` | 根据工具结果回答待办和完整启动命令，没有继续调用工具。 |

终端显示 `read D:/02-pi-agent-fork/pi-hunnu-2026/handoff.txt`，随后出现文件内容和模型最终回答。已有 `[P4]` 日志反映的调用顺序是：

```text
[P4] Agent 即将请求模型
    → 模型返回 read 工具调用
    → read 返回文件内容
[P4] 工具结果已加入上下文
[P4] Agent 即将请求模型
    → 模型返回最终回答
```

上面整理的是本轮顺序，不是终端截图或原样日志转储。调用次数也可以由消息记录独立核对：两个 assistant 消息分别以 `toolUse` 和 `stop` 结束，中间只有一条成功的 read 工具结果。

| 模型请求 | 未命中缓存的输入 `input` | 缓存读取 `cacheRead` | 输入合计 | 输出 `output` | 结果 |
| --- | ---: | ---: | ---: | ---: | --- |
| 第 1 次 | 4155 | 0 | 4155 | 92 | 返回 read 调用 |
| 第 2 次 | 192 | 4096 | 4288 | 41 | 根据文件内容回答 |

第二次请求的输入合计增加了 133 token；其上下文新增了助手的工具调用和工具结果。本轮 toolResult 的 `durationMs=4`，这是工具执行耗时，不能当作整轮交互耗时。

模型返回的原始工具参数如下。工具调用 ID 为 `call_00_Z2zYdWB32URwpooEPPm64773`，工具结果记录中的 `toolCallId` 与其一致。

```json
{
  "path": "D:/02-pi-agent-fork/pi-hunnu-2026/handoff.txt",
  "offset": null,
  "limit": null
}
```

模型的最终回答原文：

```markdown
handoff.txt 内容如下：

- **待办**：补充 README 的启动说明
- **启动命令**：`npm run demo -- --profile group-a9b32a`
```

待办与命令的文字内容一致；模型额外添加了引导语、列表和 Markdown 格式，没有完全保持文件的两行排版，按实际结果记录。

成功会话 ID 为 `01a216a6-9626-75fd-8445-b93065317661`。原始会话保存在：

```text
C:\Users\Mao\.pi\agent\sessions\--D--02-pi-agent-fork-pi-hunnu-2026--\2026-11-26T06-59-05-894Z_01a216a6-9626-75fd-8445-b93065317661.jsonl
```

本报告附有从该会话抽取的[运行证据](task-a-evidence.json)，保留会话头与本轮 user、assistant、toolResult 消息，未包含认证信息或系统上下文。

### 图 1：单轮工具调用的代码级时序图

![图 1：单轮工具调用的代码级时序图](img/figure-1-tool-call.drawio.png)

[编辑 draw.io 版本](img/figure-1-tool-call.drawio)。图中五条生命线按职责归并，模型适配器与远端模型之间的网络边界、本地文件读取边界分别以注释标出。

### 对应源码与调用路径

以下路径均相对于 `D:\02-pi-agent-fork\pi-hunnu-2026`。会话记录证明调用次数、参数和结果；内部函数链、网络与文件边界根据源码映射。

| 职责 | 源码位置 | 本轮对应行为 |
| --- | --- | --- |
| 接收用户输入 | `packages/coding-agent/src/modes/interactive/interactive-mode.ts:3325` | 编辑器的 `onSubmit` 通过 `onInputCallback(text)` 或 `pendingUserInputs` 交出输入；`getUserInput()` 在 4430 行接收。 |
| 交互主循环 | `packages/coding-agent/src/modes/interactive/interactive-mode.ts:1292` | 等待 `getUserInput()`，随后 `await this.session.prompt(userInput)`。普通空闲输入通过这里进入会话。 |
| 会话到 Agent | `packages/coding-agent/src/core/agent-session.ts:2175` | `AgentSession.prompt()` → `_preparePrompt()` → `_startRun()`；1982 行调用 `this.agent.prompt(messages)`。 |
| 启动执行循环 | `packages/agent/src/agent.ts:382` | `Agent.prompt()` → `runPromptMessages()` → `runAgentLoop()`；后者进入 `agent-loop.ts` 中的 `runLoop()`。 |
| 请求模型 | `packages/agent/src/agent-loop.ts:391` | `streamAssistantResponse()` 执行上下文转换与 `convertToLlm()`，再调用 `streamFunction(model, llmContext, …)`。 |
| 模型适配与网络边界 | `packages/coding-agent/src/core/sdk.ts:412`；`packages/ai/src/models.ts:958`；`packages/ai/src/api/openai-completions.ts:385` | SDK 的 `streamFn` → `modelRuntime.streamSimple()` → provider / API 的 `streamSimple()` → `stream()` → `client.chat.completions.create(params, requestOptions).withResponse()`。DeepSeek 使用 OpenAI 兼容协议；HTTPS 请求跨到远端模型，SSE 响应返回文本和工具调用。 |
| 工具派发 | `packages/agent/src/agent-loop.ts:520`、`:710`、`:826` | `executeToolCalls()` 按默认 parallel 模式派发。本轮只有一个工具；`prepareToolCall()` 校验参数，`executePreparedToolCall()` 调用 `prepared.tool.execute()`。 |
| read 执行 | `packages/coding-agent/src/core/tools/read.ts:245` | 经 `wrapToolDefinition()` 进入 read 的 `execute()`，解析路径、检查可读性、探测文件类型，然后读取文本。 |
| 工具结果回到上下文 | `packages/agent/src/agent-loop.ts:284` | 将 `ToolResultMessage` 加入 `currentContext.messages`，继续内层循环，再次请求模型；第二次请求包含本轮提问、工具调用与文件内容。 |
| 回复显示 | `packages/coding-agent/src/modes/interactive/interactive-mode.ts:3525`、`:3649`、`:3685` | `session.subscribe()` → `handleEvent()`；`message_update` 调用 `streamingComponent.updateContent(message, true)`，`message_end` 调用 `updateContent(message, false)`；通过 `ui.requestRender()` 显示与完成回复。 |

事件由 `Agent.processEvents()` 更新状态并交给订阅者，`AgentSession._handleAgentEvent()` 再转发给交互界面。因此工具结果不是直接变成最终回答，而是先进入上下文，让模型再生成一次回复。

### 本次文本读取为什么走流式路径

`read.ts` 的默认 `ReadOperations` 将 `createReadStream` 指向 `readLocalFileChunks()`。虽然会话中的原始工具参数带有 `offset=null`、`limit=null`，`prepareToolCall()` 会先调用 `validateToolArguments()`。`packages/ai/src/utils/validation.ts:54` 使用带类型转换的 schema 校验器，将这两个可选的 null 字段移除。本地对同一构建的校验函数验证后，实际参数为只有 path 的对象。

因此 `isStreamableRange(undefined, undefined)` 为 true，文本正文进入 `selectLinesFromStream(ops.createReadStream(…))`。`readLocalFileChunks()` 使用 `fsOpen(path, "r")`，随后 `handle.stat()`、`handle.createReadStream({ end, autoClose: false })`，最终在 `finally` 中关闭文件。`selectLinesFromStream()` 使用 `for await` 和 `TextDecoder` 解码、选行，再应用输出截断规则。本文件很小，完整返回，没有触发截断。

文件类型探测是另一次文件操作：`packages/coding-agent/src/utils/mime.ts` 中的 `detectSupportedImageMimeTypeFromFile()` 使用 `open()`、`handle.read()` 读取文件头的少量字节，再关闭文件。应当区分这次缓冲区探测与后续正文的流式读取。正文不会在本次条件下进入 `selectLinesFromBuffer(await ops.readFile(…))`。

### 首次失败与重试说明

首次用原默认模型 `platform / ep-20261109191201-kspgk` 请求时，服务返回 `403 AccountOverdueError`，该次没有执行工具。随后通过 `/model` 切换到 `deepseek-flash`，在新会话中完成了上面的成功实验。图 1 以成功读取 `handoff.txt` 的这一轮为依据，没有复用此前实践 04 的记录，也没有把失败请求计入成功这一轮的模型调用次数。

任务 A 的 `handoff.txt` 与成功会话均已保留；下文使用该会话进行任务 B 对照。


## B. 恢复上次会话，继续写 README

### B-1 恢复原会话（-c）

退出 A 阶段的 Pi 后，在同一目录用原来的启动命令加 `-c` 重新启动：

```powershell
cd D:\02-pi-agent-fork\pi-hunnu-2026
node .\packages\coding-agent\dist\cli.js --offline --no-extensions --no-mcp --no-skills --no-prompt-templates --tools read --approve -c
```

确认终端重新显示了 A 阶段的提问、read 调用、待办和完整启动命令。恢复的是 A 会话 `01a216a6-9626-75fd-8445-b93065317661`，模型仍为 `deepseek-flash`。随后提交下列提问：

> 接着上次的待办，给 README 起草一条启动说明，写出交接单里的完整启动命令。只用这次会话已有的信息。不要调用工具。如果缺少命令，请直接说明。

### B-2 新建会话（/new）

待回答结束后输入 `/new`。终端显示 `New session started`，旧聊天记录消失。此版本新建会话后回到了配置默认的 platform 模型，因此先输入 `/model deepseek-flash` 切回与恢复分支相同的模型，再提交**逐字相同**的上述提问。没有把命令或随机标识加入提问，也没有禁用 read 工具；是否调用工具按实际记录统计。

两次请求均在同一目录、使用 `deepseek / deepseek-flash`，思考模式 off。只在聊天里起草说明，没有执行交接单中的命令，也没有修改源码仓库的 README。

### B-3 两次回答与对照记录

| 对照项 | 恢复 A 会话 | `/new` 新会话 |
| --- | --- | --- |
| 用户提问 | 相同 | 相同 |
| 本轮模型调用 | 1 次 | 1 次 |
| 本轮工具调用 | 0 次 | 0 次 |
| 完整命令 | 正确返回 `npm run demo -- --profile group-a9b32a` | 明确表示没有交接单内容，缺少完整命令 |
| 界面是否回放 A 阶段聊天 | 是，显示原提问、read 记录和完整命令 | 否，显示 `New session started` |
| 未命中缓存的输入 `input` | 150 token | 206 token |
| 缓存读取 `cacheRead` | 4224 token | 3968 token |
| 输入合计 `input + cacheRead` | 4374 token | 4174 token |
| 输出 `output` | 150 token | 130 token |
| `stopReason` | `stop` | `stop` |
| 会话 ID | `01a216a6-9626-75fd-8445-b93065317661` | `01a216c9-2dc1-7462-83cf-99155cf1d8c3` |

**统计口径说明：** 本地 OpenAI 兼容适配器将缓存命中部分从 `usage.input` 中扣除，另记为 `usage.cacheRead`。因此不能只比较 150 与 206 就判断哪一边上下文更多；本次应比较输入合计 4374 与 4174，差值为 200 token。输入还包含系统指令、工具定义等内容，所以新会话的输入不是只有这句话所占的 token。这里的 token 数据用于辅助观察，完整命令是否出现、有没有新工具调用及源码中的消息恢复路径构成主要证据。

恢复会话的回答原文：

````text
README 启动说明草稿（仅基于 handoff.txt 的信息）：

```markdown
## 启动

运行演示：

    npm run demo -- --profile group-a9b32a
```

说明：命令 `npm run demo -- --profile group-a9b32a` 来自交接单，是本次会话中唯一可用的信息。除该命令外，没有其他可用信息，例如前置安装步骤、Node 版本要求、`--profile` 参数含义、输出说明或预期结果；这些内容我都没有读取 README 或任何文件来确认，因此未写入草稿。如果需要完整、可验证的启动说明，需要读取仓库 README 和 `package.json` 进行核对。
````

新会话的回答原文：

````text
这次会话里没有“上次的待办”或“交接单”的内容，也没有 README 文件被读取过，所以我无法起草一条包含完整启动命令的启动说明。

缺少的信息：
- 交接单里的完整启动命令，我从未看到过
- README 的路径与现有内容
- 启动命令的各个部分（如依赖安装、构建、运行的顺序和参数）

按你的要求，这次我不调用工具，所以也不去仓库里查找 README 或相关脚本。请把交接单中的启动命令贴给我，或者明确允许我读取 README 和相关配置文件，我再起草。
````

完整随机标识只在 A 的工具结果和回答里出现。恢复分支没有重新读取文件却能准确给出标识，而新会话明确缺少命令；这组结果支持恢复分支使用了从会话文件重建的历史消息。结论针对本次运行，不能仅凭一次回答推广为模型始终遵守工具限制。

两轮的消息及统计见 [任务 B 运行证据](task-b-evidence.json)。其中恢复分支保留 A 与 B 的关联消息；新分支仅有本次提问和回答，二者均没有本轮 toolCall 或 toolResult。A 阶段的历史 read 不计作 B 本轮的工具调用。

### 图 2：恢复旧会话和新建会话的对照图

![图 2：恢复旧会话和新建会话的对照图](img/figure-2-session-compare.drawio.png)

[图二 draw.io 可编辑文件](img/figure-2-session-compare.drawio)

### 对应源码

以下路径相对于源码目录 `D:\02-pi-agent-fork\pi-hunnu-2026`：

| 环节 | 源码位置 | 作用 |
| --- | --- | --- |
| 识别 `-c` | `packages/coding-agent/src/main.ts:479` | `parsed.continue` 分支调用 `SessionManager.continueRecent(cwd, sessionDir)`。 |
| 找回会话文件 | `packages/coding-agent/src/core/session-manager.ts:1811` | `continueRecent()` 用当前目录定位最近会话，构造读取该文件的 SessionManager。 |
| 读取与重建上下文 | `packages/coding-agent/src/core/session-manager.ts:628`、`:544`、`:577` | `loadEntriesFromFile()` 读取 JSONL；`buildSessionProjection()` 根据当前分支等信息投影消息；`buildSessionContext()` 返回模型可用的 messages 与模型设置。本实验没有发生压缩。 |
| 恢复到 Agent | `packages/coding-agent/src/core/sdk.ts:210`、`:409` | 调用 `sessionManager.buildSessionContext()`，把 `existingSession.messages` 赋给新 Agent 的 `initialState.messages`。 |
| `/new` 入口 | `packages/coding-agent/src/modes/interactive/interactive-mode.ts:3429`、`:7116` | 编辑器识别 `/new`，执行 `handleClearCommand()`，随后调用 `runtimeHost.newSession()`。 |
| 新运行时 | `packages/coding-agent/src/core/agent-session-runtime.ts:226` | `newSession()` 创建新的 SessionManager，拆除当前运行时，用新 manager 创建并应用新的运行时。 |
| 空的新会话 | `packages/coding-agent/src/core/session-manager.ts:1065` | `newSession()` 生成新会话 ID / 文件路径，将 `fileEntries` 重置为新 header，清空索引，将 `leafId` 置空；不复制 A 的聊天消息。 |
| 发送本次请求 | `packages/agent/src/agent-loop.ts:391`；`packages/coding-agent/src/core/sdk.ts:412` | `streamAssistantResponse()` 将当前 messages 经 `convertToLlm()` 转换，再通过 `streamFunction()` / `modelRuntime.streamSimple()` 请求模型。 |

因此恢复分支的请求带有 A 的 4 条聊天消息与 B 的本次提问，新会话的聊天部分只有 B 的本次提问。这里的“空历史”指没有旧的用户、助手和工具结果消息；新会话仍会构造系统指令、工具定义、项目上下文等信息，并不表示请求完全没有其他上下文。会话 JSONL 用于保存、重建历史，模型请求时发送的是转换后的上下文，并非直接把整个 JSONL 文件发给模型。

## 图表编辑

`img/*.drawio` 为原生 XML 图元文件，文字、矩形、生命线和连接线均可分别编辑，没有嵌入整张 PNG。用 draw.io / diagrams.net 的“文件 → 打开”载入即可。图一与图二分别提供源文件；报告引用本人更新后的 `.drawio.png` 图片，图片中也内嵌了可编辑的 draw.io 数据。

## 适合修改的位置

| 想改变的行为 | 优先查看的位置 | 原因 |
| --- | --- | --- |
| 输入如何提交、回复如何显示 | `interactive-mode.ts` 的 `onSubmit`、`handleEvent()` | 分别连接用户输入和界面渲染。 |
| 工具执行后是否继续请求模型 | `agent-loop.ts` 的 `runLoop()` | 工具结果在这里进入上下文，决定是否进入下一次模型请求。 |
| read 如何读取和截断文件 | `core/tools/read.ts` | 文件路径处理、正文读取与输出截断集中在工具实现中。 |
| 恢复哪些历史消息、新建会话清理哪些内容 | `session-manager.ts`、`agent-session-runtime.ts`、`sdk.ts` | 分别负责会话文件和历史投影、运行时替换，以及 Agent 初始消息的装配。 |

## 实验体会

这次实验让我把“一轮交互”和“一次模型调用”区分开了。用户只提交一次读取请求，模型先给出工具调用，read 的结果进入上下文后，模型再生成最终回答；工具结果参与了第二次请求，而不是直接成为最终回复。

会话恢复也能沿源码解释：`-c` 找到已保存的会话，Pi 将历史消息重建到 Agent 的上下文，再加入新提问。完整随机命令已经保存在工具结果和旧回答里，所以恢复分支无需重新读文件。`/new` 创建新的聊天历史，即使还在同一目录，也不会自动拥有文件内容；本次模型因缺少命令而明确说明。

另外，模型服务的缓存统计与聊天历史是两个不同层面。新会话仍可能命中相同系统提示的缓存，缓存命中不等于恢复了旧聊天。本次必须把未缓存输入与缓存读取相加，才能比较两次请求的输入规模。

## 参考与证据说明

- 报告的分节组织参考了[同学的实践课 5 报告](https://github.com/sakgratama-collab/pi-agent-practice/blob/main/practice5/practive-05.md)。模型、随机标识、回答、调用次数和 token 统计均使用本人记录。
- [任务 A 原始消息节选](task-a-evidence.json)与[任务 B 原始消息节选](task-b-evidence.json)保留了工具参数、结果、最终回答和 usage 字段，可核对图示与结论。
- 本报告的两张图片为 draw.io 绘制的源码分析图，不是终端截图；本次修订未新增截图。
