# Claude 多智能体与 Claude Code 学习笔记（知识点一～七）

> **来源标记**：未标注 = 官方文档；🔶 = 第三方来源；❓ = 推断或未查证，使用前请核对。

## 总览

| # | 知识点 | 一句话 |
|---|---|---|
| 1 | 何时使用多智能体 | 默认单智能体，只有遇到单智能体克服不了的约束才拆，且按上下文边界拆 |
| 2 | 三种协作方式 | Subagents、Workflow、Advisor 的区别在于谁掌握计划、谁合并结果 |
| 3 | 多智能体的构建方式 | Managed Agents 用 `multiagent` 块配置；Claude Code 用 md 定义子智能体、用脚本跑 workflow |
| 4 | Claude Code 运行机制 | 默认本地执行，模型在云端；云端 VM 只在主动选择时出现 |
| 5 | 记忆机制 | CLAUDE.md（你写）+ auto memory（Claude 写），都是上下文，不是强制配置 |
| 6 | 设置与 Hooks | Settings 由客户端强制执行；Hook 在固定时点确定性运行 |
| 7 | 权限 | 权限是 settings.json 的一部分，按 deny → ask → allow 求值 |

**贯穿主线：建议 vs 强制**

| 层级 | 机制 | 性质 |
|---|---|---|
| 建议 | CLAUDE.md、`.claude/rules/`、auto memory | 上下文，Claude 尽量遵守，不保证 |
| 强制 | Settings、权限规则 | 客户端执行，按工具和命令文本匹配 |
| 强制 + 自定义 | Hooks | 在固定时点运行你的逻辑，`PreToolUse` 的阻止连 `bypassPermissions` 也绕不过 |
| 系统级 | 沙盒（Sandboxing） | 操作系统级限制 shell 命令的文件和网络访问，最硬 |

---

## 知识点一：何时使用多智能体系统

> **核心**：多智能体是多个拥有独立上下文的 LLM 实例，由代码协调。默认先用单智能体，只有遇到单智能体克服不了的约束时才拆。
>
> 来源：[Building multi-agent systems: When and how to use them](https://claude.com/resources/articles/building-multi-agent-systems-when-and-how-to-use-them)（Anthropic，2026-01）。文章聚焦"编排者-子智能体"模式，另有 swarm、基于能力、消息总线等模式未展开。

### 一、为什么默认单智能体

- 每多一个智能体，就多一个故障点和一套要维护的提示词。
- 同等任务下，多智能体的 token 消耗通常是单智能体的 **3–10 倍**，来自上下文重复、协调消息和交接摘要。
- 不少团队花数月搭建多智能体，最后发现优化单智能体的提示词就能达到同样效果。

### 二、三种值得拆的场景

1. **上下文保护**：子任务产生大量与主任务无关的内容，会污染主上下文、降低推理质量。子智能体在自己的干净上下文里处理，只回传精简结果（文中例子：2000+ tokens 的订单历史，回传 50–100 tokens 的摘要）。适用于子任务产出多（>1000 tokens）但大部分与主任务无关、提取标准明确、需要先过滤的检索类操作。
2. **并行**：多个智能体同时探索更大的搜索空间，适合研究和检索。**主要收益是覆盖面，不是速度**：总计算量变大，整体耗时往往比单智能体更长。
3. **专业化**：
   - 工具：数量过多（常见 20+）导致选错；工具跨多个不相关领域；新增工具让原有任务变差。
   - 系统提示词：行为要求相互冲突（共情耐心 vs 严谨挑剔）。
   - 领域知识：需要大量领域背景，会压垮通用智能体。
   - 代价：编排者要正确路由，路由错误结果就差；提示词维护成本增加。

### 三、拆之前先试的替代方案

- 上下文压力大 → 先试上下文压缩（compaction）。
- 工具太多（15–20+）→ 先试 Tool Search Tool，按需加载工具定义。

### 四、怎么拆：按上下文边界，不按工种

- **反模式**：按工作类型拆（规划 → 实现 → 测试 → 评审，或大纲 → 写作 → 润色）。每次交接丢失上下文，形成"传话游戏"；文中实验里，子智能体花在协调上的 token 超过了实际工作。
- **好的边界**：独立的研究路径（亚洲市场 vs 欧洲市场；A 公司财报 vs B 公司财报）、接口清晰的独立组件（有 API 契约的前后端）、黑盒验证。
- **差的边界**：同一项工作的先后阶段、紧耦合组件、需要频繁同步的共享状态。
- 原则：负责某个功能的智能体也应该写它的测试，因为它已经有上下文。

### 五、验证子智能体（最稳定的模式）

- 专职测试或验证主智能体的产物，只需要产物、明确的成功标准和工具，不需要构建历史。
- 适用：质量保证、合规检查、输出校验、事实核查。
- **主要失败模式是"过早宣布胜利"**：跑一两个测试就判通过。对策：给具体标准、覆盖多种场景和边界、做负向测试、明确写"必须跑完整测试套件才能判通过"。
- 能力更强的编排模型越来越能直接评估子智能体的产出；在编排模型较弱、验证需要专门工具、或要强制检查点时，验证子智能体仍有价值。

### 六、决策清单

1. 存在多智能体能解决的真实约束（上下文、并行、专业化）。
2. 按所需上下文拆，而不是按问题类型拆。
3. 子智能体有不依赖全局上下文的清晰验证点。

从最简单可行的方案开始，有证据再增加复杂度。

### ⚠️ 常见误区

- ❌ 多智能体主要是为了加速 → ✅ 主要换来覆盖面，总耗时常更长，token 是 3–10 倍。
- ❌ 按工种拆最像真实团队 → ✅ 交接损耗大，应按上下文边界拆。
- ❌ 工具多就该拆成多个智能体 → ✅ 先试 Tool Search Tool。
- ❌ 验证子智能体说通过就可信 → ✅ 警惕过早宣布胜利，要求完整测试。

### 📌 总结

- 🧱 默认单智能体，多智能体只用于上下文保护、并行覆盖、专业化三种场景。
- ✂️ 按上下文边界拆，不按工种拆。
- ✅ 验证子智能体是最稳的模式，但要防止过早宣布胜利。

---

## 知识点二：三种协作方式——Subagents、Dynamic Workflows、Advisor

> **核心**：三者的本质区别在于**谁掌握计划、谁合并结果**。Claude 逐轮决定 → Subagents；脚本决定 → Workflow；Advisor 不派活，只给指导。
>
> 来源：Managed Agents 文档「多智能体编排」、Claude Code 文档「Dynamic workflows」。

### 一、三种方式对比

| | Subagents | Dynamic Workflows | Advisor |
|---|---|---|---|
| 做什么 | 主智能体委派任务，读取每份报告 | 主智能体写一个程序，后台跑多个智能体并合并结果 | 主智能体自己干活，在关键节点咨询更强的模型 |
| 谁决定下一步 | 主智能体，逐轮 | 脚本 | 主智能体 |
| 中间结果在哪 | 主会话上下文 | 脚本变量，主会话只拿最终结果 | 无 |
| 能否中途追问 | Managed Agents 中子线程持久，可发后续消息 | 运行中不能，结束后线程归档 | 不适用 |
| 规模 | 每轮几个委派任务 | 每次几十到几百个智能体 | 不适用 |

### 二、Workflow 的优势与代价

- **确定性**：流程由代码控制，执行路径更稳定。
- **规模**：中间结果不进主会话。Claude Code 中单次运行最多 1,000 个智能体、同时最多 16 个（CPU 核数少时更少）。
- **后台运行**：主会话保持可交互，可以继续和用户沟通、查看进度。
- **可复用**：脚本可保存为命令反复运行。
- **可恢复**：在同一会话内暂停后可恢复，已完成的智能体返回缓存结果。
- **可内置质量模式**：如让独立智能体对抗式互审结论、从多个角度起草方案再比较。
- **代价**：运行中不能接收用户输入；合并逻辑写死，中间结果出乎意料时不会临场调整；token 消耗大，它省的是**主会话上下文**，不是总花费。

### 三、Advisor 的定位

- 独立的第三条线，挂在主智能体上，可单独用，也可与 Subagents、Workflow 任意组合。
- Managed Agents 中的规则：只有主线程能咨询；按顾问模型的费率计费；智能体自身模型的能力不能高于顾问。
- 典型用法：用较便宜的模型执行，在规划、遇到障碍、最终审查时问强模型（文档示例：Sonnet 执行 + Opus 顾问）。
- ❓ 在写 workflow 之前先问 advisor 尤其值得，因为 workflow 启动后难以调整（推断，原文未说）。
- ❓ Claude Code 本地是否有同名的 advisor 设置，本次资料未覆盖。

### 四、Claude Code 中的四种多步机制

| | Subagents | Skills | Agent teams | Workflows |
|---|---|---|---|---|
| 是什么 | Claude 派生的工作者 | Claude 遵循的指令 | 主智能体监督多个对等会话 | 运行时执行的脚本 |
| 谁决定下一步 | Claude，逐轮 | Claude，按提示 | 主智能体，逐轮 | 脚本 |
| 中间结果 | Claude 的上下文 | Claude 的上下文 | 共享任务列表 | 脚本变量 |
| 可复用的是 | 工作者定义 | 指令 | 团队定义 | 编排本身 |
| 被中断时 | 重启该轮 | 重启该轮 | 队友继续运行 | 同一会话内可恢复 |

### 五、如何选择

判断标准：**能不能提前把"怎么拆、怎么合"写成代码？**

- 不能，要看了中间结果才知道下一步（探索型任务）→ **Subagents**。
- 能，流程已知、规模大、要求稳定可复现（审计、迁移、交叉核对）→ **Workflow**。
- 一个智能体就能做完，只需关键节点的高质量判断 → **Advisor**。

### ⚠️ 常见误区

- ❌ Workflow 更便宜 → ✅ 省的是主会话上下文，总 token 往往更多。
- ❌ Advisor 属于 subagents 或 workflow → ✅ 独立设置，挂在主智能体上。
- ❌ Workflow 只有确定性这一个优势 → ✅ 还有规模、后台运行、可复用、可恢复、内置质量模式。
- ❌ Workflow 和 Subagents 二选一 → ✅ Workflow 内部编排的就是子智能体，两者是组合关系。

### 📌 总结

- 🧭 Subagents：Claude 当项目经理，灵活，上下文负担重。
- ⚙️ Workflow：脚本当项目经理，可扩展、可复用，运行中不能插话。
- 🧑‍🏫 Advisor：向上问路，不向下派活，与前两者正交。

---

## 知识点三：多智能体的构建方式

> **核心**：智能体的定义只有一种（md 文件或 API），Subagents 与 Workflow 只是"怎么被调用"不同。先分清 Managed Agents（API 平台）和 Claude Code（本地工具）两套体系。

### 一、先分清两套体系

- **Managed Agents（API 平台）**：智能体托管在 Anthropic 云端运行，通过 API 使用，面向把智能体嵌进自己产品的开发者。用 `ant apply xxx.md`（YAML 头部 + 系统提示）或 API 创建，用 `multiagent` 块配置。
- **Claude Code（本地工具）**：子智能体是 md 文件，workflow 是 Claude 生成的 JavaScript 脚本。
- **Codex 的 `Agents.md` / `xx.toml` / `config.toml`**：另一个产品，不要混用。（注意：Claude Code 可以读取仓库里的 `AGENTS.md` 作为项目指令，见知识点五，但它不是子智能体定义。）
- 只用本地 Claude Code，可以不学 Managed Agents 的配置；概念是通用的。

### 二、Managed Agents 的配置

- `multiagent.type` 设为 `multiagent_20261001` 时，`subagents` 和 `workflows` **默认都开启**，`advisor` 默认关闭；想只用一种，需显式把另一种设为 `{"type": "disabled"}`。
- `subagents.predefined_agents` 与 `workflows.predefined_agents` 是两个独立列表，各最多 20 个，互不继承；`{"type": "self"}` 可列出自身副本。
- 两边都默认允许**内联智能体**（运行时临时定义，使用主智能体的模型）；关闭后列表里至少要有一个智能体。workflow 中想让部分步骤用别的模型，需预先创建并列入 `workflows.predefined_agents`。
- 委派只有一层；一个会话同时最多 25 个子线程（advisor 线程和 workflow 线程不计入）。
- 在系统提示里告诉智能体何时启动 workflow（例如"超过几份合同就开 run"）；建议设置会话预算。

```json
{
  "multiagent": {
    "type": "multiagent_20261001",
    "workflows": { "type": "enabled" },
    "subagents": {
      "type": "enabled",
      "predefined_agents": [{ "type": "agent", "id": "agent_xxx" }]
    },
    "advisor": { "type": "enabled", "model": "claude-opus-5-5" }
  }
}
```

### 三、Claude Code 的子智能体

- 用 md 文件定义，放在 `.claude/agents/`（项目，可共享）或 `~/.claude/agents/`（个人，所有项目）。
- 可在权限里用 `Agent(名称)` 的 deny 规则禁用某个子智能体（见知识点七）。

### 四、Claude Code 的 Workflow

- **如何触发**：
  - 在提示里直接说"use a workflow"，或带上关键词 `ultracode`。
  - `/effort ultracode`：本会话内由 Claude 自行判断何时用 workflow（更耗 token）。
  - 内置 workflow：`/deep-research <问题>`。
- **启动审批**：Default 和 Accept edits 模式下每次询问（可选"不再询问该 workflow"）；Auto 模式只在首次询问；Bypass、`claude -p`、Agent SDK 直接运行。
- **保存与复用**：`/workflows` 里选中一次运行按 `s`，保存到 `.claude/workflows/`（项目共享）或 `~/.claude/workflows/`（个人），之后用 `/<名称>` 运行；可通过 `args` 传入参数。
- **手改脚本**：每次运行的脚本都写在 `~/.claude/projects/` 的会话目录下，可阅读、对比、编辑后让 Claude 从修改版重新启动。
- **转换现有编排**：可以让 Claude 把已有的子智能体提示词或 skill 改写成等价的 workflow。
- **限制**：运行中不能接收用户输入（需要人工签核时，把每个阶段拆成独立的 workflow）；脚本本身不能直接读写文件或运行 shell，由智能体去做；workflow 拉起的子智能体以 acceptEdits 模式运行并继承你的工具允许列表。
- **开关**：`/config` 中的 Dynamic workflows、`"disableWorkflows": true`、`CLAUDE_CODE_DISABLE_WORKFLOWS=1`。需要 v2.1.154 或更高版本。

### 五、与 Google ADK 的对比

- **ADK YAML**：你事先声明编排（顺序、并行、循环），固定、人工设计。
- **Claude 动态工作流**：Claude 针对任务在运行时生成 JS 脚本，灵活，每次可能不同。
- 想要固定流程：Claude 没有声明式 YAML，但可以把满意的运行**保存为 workflow 命令**，或手改脚本后复用，这是最接近 ADK 的做法。

### 六、实例：CLAUDE.md 里的流水线 DAG

- 写在 CLAUDE.md 里的 DAG 只是给 Claude 看的说明，没有强制力；交给 skill 执行时，仍是 Claude 逐轮决定。
- 想变成真正的编排：让 Claude 把 skill 转成 workflow 并保存。人工审批门（GATE）无法放在脚本中间，应拆成"S1–S5 一个 workflow → 人工批准 → S6–S7 一个 workflow"。
- 硬性约束（如部署前必须确认）应配合权限规则或 hook。

### ⚠️ 常见误区

- ❌ Workflow 要用 SDK 手写 → ✅ Claude 写脚本，你负责开启、引导、保存。
- ❌ Subagents 和 Workflow 需要不同格式的智能体文件 → ✅ 智能体定义相同，区别在调用方式。
- ❌ Managed Agents 里没开 workflow 就用 subagents → ✅ 两者默认同时开启，由智能体按任务选择。
- ❌ Claude Code 的 workflow 无法复用或修改 → ✅ 可保存为命令，也可编辑脚本后重跑。
- ❌ Codex 的 toml 配置也适用于 Claude → ✅ 两套体系，不要混用。

### 📌 总结

- 🧩 智能体定义只有一种；Subagents 与 Workflow 只是调用方式不同。
- ☁️ Managed Agents 用 `multiagent` 块；💻 Claude Code 用 `.claude/agents/` + workflow 脚本。
- 💾 没有 ADK 式 YAML，但 workflow 可保存为命令复用。

---

## 知识点四：Claude Code 运行机制

> **核心**：Claude Code 是包在模型外的 harness（提供工具、管理上下文）。默认在本机运行，模型在云端推理；云端 VM 只在你主动选择云端模式时出现。

### 一、智能体循环

- 收到任务后反复经历**收集上下文 → 采取行动 → 验证结果**，用户可随时打断（`Esc`）或直接输入修正。
- 模型负责推理，工具负责行动；内置工具分五类：文件操作、搜索、执行（shell、git）、网页、代码智能。
- 扩展机制：skills、MCP、hooks、subagents，叠在核心循环之上。

### 二、三个位置

- **本机**：Claude Code、会话记录、工具执行都在这里。
- **云端模型**：负责推理。
- **云端 VM**：Anthropic 托管或组织自托管的执行环境，只在选择云端模式时使用。

### 三、本地模式

- 默认模式，代码直接运行在你的机器上，完整访问项目文件、终端和 git 状态，直接使用本机的 CPU、内存和磁盘，没有"切分资源"的机制。
- 会话以 JSONL 保存在本机 `~/.claude/projects/`，支持恢复（`--continue` / `--resume`）和分叉（`--fork-session`）；每个新会话从空白上下文开始。
- 安全机制：
  - **检查点**：改文件前自动快照，按两次 `Esc` 回退；只覆盖文件改动，数据库、API、部署等远程操作不可回退。
  - **权限模式**：见知识点七。
  - **沙盒（可选）**：操作系统级限制 Bash、PowerShell、Monitor 命令及其子进程的文件和网络访问，需要开启；它是访问限制，不是资源切分。
- **并行开发**：会话与目录绑定，同一项目开多个会话应使用 git worktree，一个目录一个会话，避免互相覆盖文件。

### 四、上下文窗口

- 窗口里有对话历史、文件内容、命令输出、CLAUDE.md、auto memory、已加载的 skills 和系统指令。
- 快满时自动压缩，早期的详细指令可能丢失：持久规则写进 CLAUDE.md；skills 按需加载；子智能体在自己的上下文里工作，只回传摘要。
- `/context` 查看上下文占用。

### 五、什么时候在云端 VM 运行

必须主动选择：

- **在 claude.ai/code 发起任务**：从头到尾在云端，VM 克隆你的 GitHub 仓库；关闭浏览器后会话仍保留，可用手机 App 查看。
- **从本地终端把任务送到云端**：🔶 第三方文章写的参数是 `--remote`，日文官方文档摘要写的是 `--cloud`（拉回本机用 `--teleport`），两者不一致，以官方最新文档为准。
- **组织自托管的云端环境**。

注意事项：
- 研究预览阶段，面向 Pro、Max、Team，以及有高级席位或 Chat + Claude Code 席位的 Enterprise 用户；启用 Zero Data Retention 的组织不能使用。
- 云端会话从仓库的全新克隆开始，只有提交到仓库的内容可用，本机未推送的改动看不到。
- 云端只读取仓库里的 `.claude/settings.json` 和组织的 server-managed 设置，不读你的 `~/.claude/settings.json` 和 `settings.local.json`；auto memory 也不同步到云端。
- 🔶 VM 规格 4 vCPU / 16 GB / 30 GB，未在官方页面确认。

### 六、界面不等于执行位置

| 使用方式 | 代码在哪执行 |
|---|---|
| 本地终端、桌面应用、IDE 插件（默认） | 本机 |
| Remote Control（浏览器操作） | 本机 |
| claude.ai/code 发起任务 | 云端 VM |
| 从本地终端送到云端 | 云端 VM |
| 组织自托管环境 | 组织运营的环境 |

Slack、CI/CD 在官方文档中只列为使用界面，是否分配云端沙箱没有说明。

### 七、MCP 与 Tool call

- 官方文档明确：MCP 是让 Claude 连接外部服务（GitHub、数据库等）的扩展机制，不是客户端与模型的通信通道。MCP 工具命名为 `mcp__<server>__<tool>`，在权限规则和 hook 中与其他工具一样作为工具调用处理。
- ❓ 未查证：模型与 harness 之间通过 HTTPS API 通信；模型返回"请调用某工具"的请求，由 harness 执行后送回结果，这一层是 tool call；内置工具由 harness 直接执行，不经过 MCP。

### ⚠️ 常见误区

- ❌ 客户端通过 MCP 把会话历史发给云端服务器 → ✅ MCP 用于连接外部服务，会话历史存在本机。
- ❌ 本机有一个固定资源的沙盒，只做简单任务 → ✅ 本地模式直接运行在你的机器上；可选的沙盒只限制 shell 命令的文件和网络访问。
- ❌ 有客户端就在本地，没客户端就在云端 → ✅ 取决于运行模式，claude.ai/code 也是客户端但在云端执行。
- ❌ 云端能看到本机所有改动 → ✅ 只有提交到仓库的内容可用。
- ❌ 同一目录开多个会话并行改文件 → ✅ 用 git worktree 隔离。

### 📌 总结

- 🔄 核心是"收集上下文 → 行动 → 验证"的循环，Claude Code 就是 harness。
- 🖥️ 默认本机执行，模型在云端；☁️ 云端 VM 只在主动选择时出现。
- 🔌 MCP 是连接外部服务的扩展，不是与模型通信的通道。

---

## 知识点五：记忆机制——CLAUDE.md、AGENTS.md 与 Auto Memory

> **核心**：每个会话从空白上下文开始，跨会话靠 CLAUDE.md（你写）和 auto memory（Claude 写）。两者都是**上下文，不是强制配置**；要强制，用权限规则或 hook。

### 一、两种机制

| | CLAUDE.md | Auto memory |
|---|---|---|
| 谁写 | 你 | Claude |
| 内容 | 指令和规则 | 偏好、纠正、项目背景 |
| 作用范围 | 组织、用户、项目、本地 | 每个仓库一份，同仓库的 worktree 共享 |
| 加载 | 每个会话 | 每个会话（`MEMORY.md` 前 200 行或 25KB） |

- CLAUDE.md 作为系统提示之后的一条用户消息传入，不保证严格遵守；需要系统提示级别的指令用 `--append-system-prompt`。

### 二、CLAUDE.md 的位置（加载顺序从广到窄）

- **托管策略**：组织统一部署，不能用 `claudeMdExcludes` 排除。
- **用户级**：`~/.claude/CLAUDE.md`，所有项目通用。
- **项目级**：`./CLAUDE.md` 或 `./.claude/CLAUDE.md`，随版本控制共享。
- **本地级**：`./CLAUDE.local.md`，个人偏好，加入 `.gitignore`。

### 三、加载规则

- **向上**：工作目录及其上面每一级目录的 CLAUDE.md，启动时一次性全部加载，没有层数限制。
- **拼接而非覆盖**：从根目录往下排，离工作目录越近越靠后；同目录内 `CLAUDE.local.md` 排在 `CLAUDE.md` 之后。冲突时 Claude 可能任选其一，应避免冲突。
- **向下**：子目录的 CLAUDE.md 按需加载，Claude 读写该子目录文件时才加载。
- ❓ **兄弟目录**：不在上层链也不在下层，按规则推断不会加载，可用 `/context all` 验证。
- **细节**：
  - `@路径` 导入最多递归 4 层，但不减少上下文占用；工作目录外的导入首次需批准。
  - 块级 HTML 注释 `<!-- -->` 注入前会被去掉，可给人类留备注。
  - 项目根 CLAUDE.md 在 `/compact` 后会从磁盘重新读入；只在对话里说过的指令才会丢。
  - 大型 monorepo 中无关的上层文件用 `claudeMdExcludes` 跳过。

### 四、怎么写

- **何时添加**：Claude 第二次犯同样的错；每次会话都在重复同一句纠正；新同事也需要这条背景。
- **具体可验证**：写"缩进用 2 个空格"，不写"代码格式要好看"。
- **短小**：每个文件少于 200 行；按标题和列表分组；定期清理过时和矛盾的内容。
- **分流**：多步骤流程放 skill；只涉及部分代码的规则放 `.claude/rules/` 并加 `paths`。
- **命令**：`/init` 生成起始文件；`/doctor prompt-audit` 检查过时或矛盾的指令；`/memory` 浏览和编辑。

### 五、`.claude/rules/`

- 按主题拆成多个 md 文件，支持子目录和软链接共享。
- 头部 `paths` 字段（glob）限定范围，只在处理匹配文件时加载；没写 `paths` 的启动时加载。
- 个人规则放 `~/.claude/rules/`，对所有项目生效，在项目规则之前加载。

### 六、AGENTS.md

- Claude Code（v2.1.277+）可直接读 `AGENTS.md`，但默认**只在工作目录及上层没有任何 CLAUDE.md（含 `CLAUDE.local.md`）时**才读。`~/.claude/CLAUDE.md` 和托管 CLAUDE.md 不算在这个检查里。
- 想同时读：`/config` 把 Project instructions 设为 `claude-md-and-agents-md`。
- 想共用一份：在 CLAUDE.md 里写 `@AGENTS.md`，再追加 Claude 专属内容。
- 迁移：`/init` 吸收 Cursor、Copilot 等规则；`/import` 导入其他编码智能体的配置。

### 七、Auto memory

- 本地会话默认开启，Claude 自行判断什么值得记。
- 四类笔记：`user`（角色、偏好）、`feedback`（纠正）、`project`（进行中的工作、决策）、`reference`（外部资料位置）；能从代码推断的和 CLAUDE.md 已写的不记。
- 存储在 `~/.claude/projects/<项目>/memory/`：`MEMORY.md` 是索引，每次会话加载；主题文件按需读取。
- 本机、按仓库；不跨机器、不跨云端环境；不会写进 `~/.claude/CLAUDE.md`。
- 说"记住用 pnpm"会存入 auto memory；要写进 CLAUDE.md 需明确说"加到 CLAUDE.md"。
- 主会话的 auto memory 不加载给子智能体（fork 除外），子智能体可有自己的记忆。

### 八、排错

1. `/context all` 查看 Memory files 列表（子目录文件按需加载，不在此列）。
2. 检查位置、具体程度、是否冲突。
3. 文件超过 4 MiB 会被整个跳过；子目录文件和按需规则有单文件字符上限，超出只收到开头。
4. 用 `InstructionsLoaded` hook 记录哪些文件何时加载。
5. 必须在固定时点执行的事写成 hook。

### 九、Claude Code 记忆 vs Claude 客户端（chat）记忆

- **Claude Code**：本机文件、按项目隔离。
- **Chat**：后台从对话自动生成，也可明确要求"记住"或"忘掉"；有开关；Projects 可设为与账号级记忆连通或隔离；"搜索历史对话"是另一个功能。
- 🔶 两者不互通（第三方文章和 GitHub 用户反馈，非官方声明）。想让 Claude Code 跨项目记住偏好，写进 `~/.claude/CLAUDE.md` 或 `~/.claude/rules/`。

### ⚠️ 常见误区

- ❌ CLAUDE.md 是强制配置 → ✅ 只是上下文。
- ❌ 加载是一级一级逐步进行 → ✅ 上层链启动时一次性拼接加载，子目录才按需。
- ❌ 下层 CLAUDE.md 覆盖上层 → ✅ 拼接，冲突时任选其一。
- ❌ 有 AGENTS.md 就一定会读 → ✅ 默认只在没有任何 CLAUDE.md 时读。
- ❌ `@` 导入能省上下文 → ✅ 只便于组织。
- ❌ auto memory 跨机器同步、写进用户级 CLAUDE.md → ✅ 本机、按仓库。

### 📌 总结

- 🧠 CLAUDE.md 你写、auto memory Claude 写，都是上下文。
- 📂 上层链启动时拼接加载，子目录按需。
- ✂️ 具体、简短、不矛盾；流程放 skill，局部规则放 rules。

---

## 知识点六：设置（Settings）与 Hooks

> **核心**：Settings 是由客户端强制执行的 JSON 配置，多处并存时按优先级覆盖、列表合并。Hook 是在生命周期固定时点**必定执行**的用户命令，是"确定性自动化"的核心手段。

### 一、Settings vs CLAUDE.md

- **CLAUDE.md**：文字指令，Claude 尽量遵守；多处并存时拼接。
- **Settings**：JSON 键值（模型、权限、hooks、插件、环境变量等），客户端强制执行；多处并存时按优先级覆盖，列表合并。

### 二、四类设置文件

| 范围 | 文件 | 影响谁 | 用途 |
|---|---|---|---|
| User | `~/.claude/settings.json` | 你，本机所有项目 | 个人偏好、默认模型 |
| 共享项目 | `.claude/settings.json` | 项目所有人（提交到 git） | 团队权限、hooks、插件 |
| 项目本地 | `.claude/settings.local.json` | 你，仅该项目 | 个人覆盖、试验 |
| Managed | `managed-settings.json`、MDM、claude.ai 控制台 | 组织内所有人 | 安全策略、合规 |

- 本地文件由 Claude Code 首次写入时会自动加进全局 git 排除；手动创建的需自己加 `.gitignore`。在 git 仓库中它位于仓库根目录，作用于整个仓库。
- `~/.claude.json` 由 Claude Code 自己维护（账号、MCP 配置、项目信任决定等），不需手动编辑。

### 三、修改与确认

- `/config`：仅终端可用。
- 直接编辑：严格 JSON，不允许注释和末尾逗号；建议加 `$schema` 获得补全。多数改动无需重启；`model`、`effortLevel` 等只在启动时读取，中途用 `/model`、`/effort`。
- 只改本次会话：`--settings '{...}'`、专用参数（`--model`）、环境变量（`ANTHROPIC_MODEL`）。
- `/status` 看加载了哪些文件（不显示每个键的来源）；`claude doctor` 看被拒绝的条目。

### 四、优先级

**Managed > 命令行 `--settings` > 项目本地 > 共享项目 > User**

- 环境变量不在层级内，按"键与变量一对一"决定。
- 列表合并（如 `permissions.allow`）；`fallbackModel`、`modelPicker`、`availableModels`、`modelSettings` 有各自规则。
- 少数安全键例外：低层写的更严格值会被采用，即使 managed 更宽松。

### 五、设置没生效怎么查

1. 更高层设了同一个键。
2. 该文件不能设这个值（如 `defaultMode` 的 `auto`、`bypassPermissions` 不从项目或本地文件生效）。
3. 文件写坏。
4. 提交的键没到队友：部分键不从仓库文件生效；`allow` 规则要等队友信任文件夹，`deny`、`ask` 立即生效。

### 六、Hooks：为什么重要

- CLAUDE.md 靠 Claude 自觉，hook 由客户端在固定时点执行，**必定发生**。文档中凡是"必须每次发生"的需求，都指向 hook。
- 典型用途：编辑后自动格式化、阻止修改受保护文件、等待输入时发通知、压缩后重新注入关键上下文、审计配置变更、自动批准特定提示。
- `PreToolUse` 在所有权限模式下都会触发，返回 deny 即使在 `bypassPermissions` 下也能拦截。

### 七、配置结构：事件 → matcher → handler

```json
{
  "hooks": {
    "PostToolUse": [
      {
        "matcher": "Edit|Write",
        "hooks": [
          { "type": "command", "command": "jq -r '.tool_input.file_path' | xargs npx prettier --write" }
        ]
      }
    ]
  }
}
```

上例：Claude 每次用 Edit 或 Write 改完文件，自动用 Prettier 格式化该文件。成功时对话里不显示任何内容。

- 位置决定范围：User、共享项目、本地、managed，以及插件 `hooks/hooks.json`、skill 或 subagent 的 frontmatter。
- `/hooks` 浏览；`"disableAllHooks": true` 关闭（managed 里的 hook 仍运行，除非 managed 也设置）。

### 八、常用事件

- `SessionStart`：会话开始或恢复（matcher：`startup`、`resume`、`clear`、`compact`、`fork`）。
- `UserPromptSubmit`：提交提示词后、Claude 处理前，可注入上下文。
- `PreToolUse`：工具执行前，**可阻止**。
- `PermissionRequest`：即将询问权限时，可代为批准。
- `PostToolUse`：工具成功后，不能撤销。
- `Notification`：Claude 等待输入或权限。
- `Stop`：Claude 每次回复结束（不只是任务完成，用户中断时不触发）。
- 其他：`SubagentStart/Stop`、`PreCompact/PostCompact`、`InstructionsLoaded`、`ConfigChange`、`FileChanged`、`SessionEnd` 等，完整列表见官方 Hooks reference。

### 九、过滤

- **matcher**：工具类事件按工具名（`Bash`、`Edit|Write`、`mcp__github__.*`），区分大小写；`UserPromptSubmit`、`Stop` 等不支持 matcher。
- **`if`**：按工具名 + 参数过滤（`Bash(git *)`），仅限工具类事件；是尽力而为，硬性允许或拒绝用权限系统。
- Bash 命令也能改文件，只匹配 `Edit|Write` 不够，可加 `Stop` 或 `FileChanged`。

### 十、输入与输出

- **输入**：事件 JSON 通过 stdin 传入。
- **退出码 0**：无异议；对 `PreToolUse` **不等于批准**，仍走正常权限流程。
- **退出码 2**：阻止，stderr 作为原因（`SessionStart` 等事件不可阻止）。
- **其他退出码**：多数事件视为非阻塞错误，动作继续。
- **JSON 输出**：exit 0 加 JSON 可精细控制（如 `permissionDecision`）；每个 hook 只选一种方式；字段须放在 `hookSpecificOutput` 内，放顶层会被静默忽略。

### 十一、类型

- `command`：shell 命令，最常用，确定性。
- `http`：POST 到 URL；失败默认不阻止，可设 `onFailure: "block"`。
- `mcp_tool`：调用 MCP 工具。
- `prompt`：单轮 LLM 判断，适合需要判断力的场景。
- `agent`：多轮验证，可读文件和跑命令，**实验性**。

### 十二、Hook 与权限规则的关系

- hook 返回 `allow` 可以跳过交互提示，但**绕不过 deny 和 ask 规则**；设置和插件中的 hook 只能收紧、不能放宽权限规则。
- hook 的阻止（exit 2）**优先于 allow 规则**：可以"整体允许 Bash + hook 精确拦截少数命令"。
- 同一事件的多个 hook 并行运行，权限决定取最严格（`deny` > `defer` > `ask` > `allow`）；一个 deny 不会阻止其他 hook 的副作用。

### 十三、安全与排错

- **陌生仓库的风险**：仓库 settings 里的 hooks 和 `env`，在只信任了上层文件夹或 `claude -p`、SDK 模式下**仍会运行**。对不是你写的仓库运行 `claude -p` 前，考虑 `--setting-sources user`、`--bare`，或 `--settings '{"disableAllHooks": true}'`（只写在用户设置里不够，项目设置优先级更高）。
- `Stop` hook 连续阻止 8 次会被强制放行，脚本应检查 `stop_hook_active`。
- 排错：`/hooks` 确认注册；手动喂样例 JSON 测试；`chmod +x`、装好 `jq`、用绝对路径或 `${CLAUDE_PROJECT_DIR}`；shell 配置文件里无条件的 `echo` 会污染 stdout 导致 JSON 失效；`Ctrl+O` 看 transcript，`claude --debug` 看日志。

### ⚠️ 常见误区

- ❌ 规则写进 CLAUDE.md 就能强制 → ✅ 强制靠权限规则或 hook。
- ❌ 环境变量是设置优先级的一层 → ✅ 按键与变量一对一决定。
- ❌ `PreToolUse` exit 0 就是批准 → ✅ 只是无异议。
- ❌ hook 返回 allow 就能绕过 deny → ✅ 绕不过 deny 和 ask。
- ❌ `Stop` hook 只在任务完成时触发 → ✅ 每次回复结束都触发。
- ❌ 云端会话读我的用户级设置 → ✅ 只读仓库共享设置和组织 server-managed 设置。

### 📌 总结

- ⚙️ Settings 客户端强制执行，managed > 命令行 > 本地 > 共享项目 > User，列表合并。
- 🪝 Hook = 确定性自动化：事件（何时）+ matcher / `if`（对谁）+ handler（做什么）。
- 🔒 强制拦截：`PreToolUse` + exit 2 或 `permissionDecision: deny`。

---

## 知识点七：权限（Permissions）

> **核心**：权限就是 settings.json 里的 `permissions` 键，共享 Settings 的文件位置和优先级；区别在于合并后按 **deny → ask → allow** 求值，任一层的 deny 都无法被 allow 放开。权限由客户端执行，CLAUDE.md 只影响 Claude 想做什么。

### 一、和 Settings 的关系

| | 普通设置 | 权限规则 |
|---|---|---|
| 存放位置 | settings.json 各键 | settings.json 的 `permissions` 键 |
| 多文件冲突 | 高层覆盖低层 | 列表合并后按 deny → ask → allow 求值 |
| 高层能否放开低层限制 | 能 | 不能，任一层 deny 都一票否决 |
| 信任要求 | 部分键需信任 | 项目的 `allow`、`additionalDirectories` 需信任，`deny`、`ask` 不需要 |

```json
{
  "permissions": {
    "allow": ["Bash(npm run *)"],
    "ask":   ["Bash(git push *)"],
    "deny":  ["Read(./.env)"],
    "defaultMode": "acceptEdits"
  }
}
```

### 二、默认审批（Manual 模式）

| 工具类型 | 是否需要批准 | "Yes, and don't ask again" |
|---|---|---|
| 只读（读文件、Grep） | 工作目录内不需要 | 无 |
| Bash | 需要，内置只读命令除外 | 永久，按仓库和命令 |
| 改文件 | 需要 | 仅到会话结束 |
| WebFetch | 需要，预批准文档域名除外 | 永久，按仓库和域名 |
| WebSearch | 需要 | 永久，按仓库 |

永久批准保存在仓库根目录的 `.claude/settings.local.json`；回答提示时可按 `Tab` 附一条评论给 Claude。

### 三、规则与求值

- 顺序固定 deny → ask → allow，第一个匹配决定结果，具体程度不影响顺序。
- allow 不能给 deny 开例外：`Bash(aws *)` 的 deny 也挡住 `Bash(aws s3 ls)` 的 allow。
- ask 压过 allow：点了"Yes, and don't ask again"仍被问，通常是另有 ask 规则。
- 裸工具名 deny（`Bash`）会把工具从 Claude 的上下文中移除；带范围的 deny（`Bash(rm *)`）只拦匹配调用。
- 管理入口：`/permissions`（会话中修改，下次工具调用即生效）；命令行 `--allowedTools`、`--disallowedTools`。

### 四、六种模式（`defaultMode`）

- `default`（Manual）：首次使用每个工具时询问。
- `acceptEdits`：自动接受工作目录内的编辑和 `mkdir`、`mv` 等常见文件命令。
- `plan`：读文件、跑只读命令来探索，不改源文件。
- `auto`：后台分类器审查 shell 命令、网络请求等，拦截风险操作。
- `dontAsk`：凡需询问的一律拒绝，只运行已预批准的。
- `bypassPermissions`：跳过提示（含对 `.git`、`.claude` 的写入），只在容器或虚拟机等隔离环境使用。
- `auto`、`bypassPermissions` 不能从项目或本地文件设置；可用 `permissions.disableBypassPermissionsMode` / `disableAutoMode` 设为 `"disable"` 禁用。

### 五、Bash 规则

- **`*` 放在子命令之后**：`Bash(git log *)` 只放行 `git log`；`Bash(git *)` 放行所有 git 命令。
- **结尾 ` *` 也匹配裸命令，空格有意义**：`Bash(ls *)` 匹配 `ls`，不匹配 `lsof`；`Bash(ls*)` 两者都匹配。`:*` 是结尾通配符的等价写法。
- **复合命令**：识别 `&&`、`||`、`;`、`|` 等，每个子命令都被允许才免询问；deny、ask 只要任一子命令（含子 shell、命令替换）匹配就生效。
- **包装器**：`timeout`、`time`、`nice`、`nohup`、`stdbuf`、`command`、`builtin`、不带参数的 `xargs` 会被剥掉再匹配；`npx`、`docker exec`、`devbox run` 等环境运行器不剥，`Bash(devbox run *)` 会放行其后任何命令，应写成具体规则。
- **内置只读命令**（`ls`、`cat`、`grep`、`find`、只读 `git` 等）在所有模式下免询问，不可配置；想询问就加 ask 或 deny。
- **不是安全边界**：匹配的是命令文本，`/usr/bin/curl`、`sh -c '...'`、`git -C . push` 都能绕过；约束参数的规则（如只允许 curl 访问某域名）很脆弱。硬性限制用沙盒，自定义检查用 `PreToolUse` hook。

### 六、Read / Edit 路径规则

- gitignore 语法，四种前缀：

| 写法 | 含义 |
|---|---|
| `//path` | 文件系统绝对路径 |
| `~/path` | 家目录 |
| `/path` | 相对于设置文件来源（项目设置 → 项目根；用户设置 → `~/.claude/`） |
| `path` 或 `./path` | 相对于当前目录 |

- `/Users/alice/file` **不是**绝对路径，要写 `//Users/alice/file`。
- `Write`、`Grep`、`Glob` 的路径规则不生效，改写为 `Edit(...)`、`Read(...)`。
- `Read` 的 deny 同时阻止对该路径的 Edit 和 Write，也作用于 `@` 提及和粘贴的图片路径。
- 只覆盖内置文件工具、可识别的 Bash 文件命令和重定向目标；不覆盖 Python、Node 等脚本间接读写，彻底阻止用沙盒。
- 符号链接：allow 要求请求路径与真实路径都匹配；deny 任一匹配即拦。

### 七、其他工具

- **WebFetch**：`WebFetch(domain:example.com)`，子域名用 `domain:*.example.com`；只靠 WebFetch 规则挡不住 Bash 里的 curl。
- **MCP**：`mcp__<server>__<tool>`，`mcp__<server>__*` 匹配该服务器全部工具；allow 规则不要带括号参数。
- **Agent**：`Agent(Explore)`、`Agent(自定义名)` 放进 deny 可禁用特定子智能体。
- **Cd**：控制 `/cd` 可去的目录，Claude 自己不能调用。

### 八、工作目录

- 默认只能访问启动目录；用 `--add-dir`、`/add-dir`、`additionalDirectories` 扩展。
- 添加目录**只给文件访问权，不加载配置**：CLAUDE.md、rules 默认不加载（需 `CLAUDE_CODE_ADDITIONAL_DIRECTORIES_CLAUDE_MD=1`）；仅 skills、commands、subagents 等少数例外从 `--add-dir` 目录加载。
- `permissions.blockReadsOutsideWorkingDirectories` / `blockWritesOutsideWorkingDirectories` 设为 `true`，所有模式下拒绝工作目录外的读或写（写入限制不覆盖 shell 命令）。

### 九、与 Hook、沙盒、Managed 的分工

- **Hook**：`PreToolUse` 在权限提示前运行；hook 的 allow 绕不过 deny 和 ask；hook 的阻止优先于 allow。
- **沙盒**：操作系统级限制 shell 命令的文件和网络访问，提示词注入也绕不过；建议与权限并用。开启沙盒且 `autoAllowBashIfSandboxed` 为默认 `true` 时，沙盒内的 Bash 命令可免于整工具级的 ask 提示，但 `Bash(git push *)` 这类带内容的 ask 仍会询问。
- **Managed**：组织的 deny 任何层都无法放开；`allowManagedPermissionRulesOnly` 可让 managed 成为唯一的权限规则来源。

### ⚠️ 常见误区

- ❌ 权限是独立于 settings 的配置 → ✅ 就是 settings.json 的 `permissions`。
- ❌ 高层 allow 能覆盖低层 deny → ✅ deny 一票否决。
- ❌ 更具体的 allow 能给宽泛的 deny 开例外 → ✅ 顺序固定 deny → ask → allow。
- ❌ `Bash(curl *)` 能彻底禁止 curl → ✅ 匹配文本可被绕过，要用沙盒。
- ❌ `/path` 是绝对路径 → ✅ 绝对路径写 `//path`。
- ❌ CLAUDE.md 写"别读 .env"就安全 → ✅ 要用 `Read(./.env)` deny。
- ❌ `--add-dir` 会加载那个目录的 CLAUDE.md 和设置 → ✅ 默认只给文件访问权。

### 📌 总结

- 🧩 权限 = settings.json 的 `permissions`，deny → ask → allow，deny 一票否决。
- ⚠️ Bash 规则匹配命令文本，不是安全边界；硬性隔离靠沙盒。
- 🪝 hook 的阻止优先于 allow，hook 的 allow 绕不过 deny 和 ask。
