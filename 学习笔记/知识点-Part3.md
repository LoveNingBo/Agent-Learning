## 知识点八：Skills 与 Subagents

> **核心**：Skill 定义"做什么"（指令包，默认在主会话里执行）；Subagent 定义"谁来做"（独立上下文的工作者，只交回总结）。两者都是**描述常驻、正文按需加载**，区别在于正文加载到哪里。

### 一、定位与加载机制

| | Skill | Subagent |
|---|---|---|
| 本质 | 指令和知识 | 独立工作者 |
| 文件 | `.claude/skills/<名>/SKILL.md`（可附脚本、参考文件） | `.claude/agents/<名>.md` |
| 正文是 | 给 Claude 的指令 | 子智能体的**系统提示** |
| 常驻主会话的 | 名字 + description | 名字 + description（合计超 15,000 tokens 会警告） |
| 正文加载到 | **主会话**（调用后一直留在上下文） | **新的独立上下文**（运行时才加载） |
| 任务来自 | 你预先写好的固定文本 | Claude 当场写的委派消息 |
| 触发 | `/skill名`、按 description 自动 | 按 description 自动、自然语言点名、@ 提及、`--agent` |

两者都会被监视目录，中途修改无需重启。

### 二、Skill 要点

- **何时写**：反复粘贴同一套指令或流程；CLAUDE.md 某节已从"事实"长成"流程"。旧的 `.claude/commands/` 已合并进 skills。
- **两类内容**：参考型（规范、知识）；任务型（部署、提交，常配 `disable-model-invocation: true`）。
- **位置优先级**：企业 > 个人 `~/.claude/skills/` > 项目 `.claude/skills/`；插件以 `/插件名:skill名` 调用。云端和 Cowork 不读本机个人 skill。
- **调用控制**：
  - `disable-model-invocation: true`：只能你手动调用，适合有副作用的操作。
  - `user-invocable: false`：只能 Claude 调用，适合背景知识。
- **生命周期（难点）**：调用后正文常驻、不重读；压缩后每个 skill 只保留前 5,000 tokens（合计 25,000），**关键指令放开头**；SKILL.md 少于 500 行。
- **`allowed-tools`**：只在调用那一轮免授权，下一条消息即失效。
- **动态注入** `` !`命令` ``：发给 Claude 前先执行命令并嵌入输出；失败或被 deny 会中止整个调用。
- **其他字段**：`paths`（按文件范围触发）、`arguments` / `$ARGUMENTS`、`model` / `effort`（本轮）、`hooks`。

### 三、Subagent 要点

**内置子智能体**

| 名称 | 工具 | 特点 |
|---|---|---|
| Explore | 只读 | 代码搜索；不加载 CLAUDE.md 和 git 状态；一次性，不能续跑 |
| Plan | 只读 | plan 模式下收集信息；同上 |
| general-purpose | 全部可用工具 | 复杂多步任务，可探索可修改 |

**作用域优先级**：Managed > `--agents` 命令行 JSON > `.claude/agents/` > `~/.claude/agents/` > 插件。只有 `name` 和 `description` 必填，缺一个文件会被静默跳过。

**关键字段**

| 字段 | 作用 |
|---|---|
| `tools` / `disallowedTools` | 允许或拒绝工具；`disallowedTools: Bash(git push *)` 会去掉**整个** Bash，部分拦截要用权限 deny 规则 |
| `model` | 模型；顺序：调用时传入 > 定义字段 > `CLAUDE_CODE_SUBAGENT_MODEL` > 主会话 |
| `permissionMode` | 主会话在 `bypassPermissions` / `acceptEdits` / `auto` / `plan` 时跟随主会话，你的值被忽略 |
| `skills` | 启动时预加载 skill **完整内容** |
| `mcpServers` | 只给它的 MCP 服务器，不占主会话上下文 |
| `memory` | 自己的持久记忆：`user` / `project`（推荐）/ `local` |
| `hooks` | 只在它运行期间生效；项目子智能体需先信任文件夹 |
| `isolation: worktree` | 在临时 git worktree 里工作 |
| `maxTurns` | 轮数上限，到达后返回部分结果，可续跑 |
| `omitClaudeMd` | 启动时不加载 CLAUDE.md |

**启动时有什么（难点）**

| 会带上 | 不会带上 |
|---|---|
| 自己的系统提示 + 环境信息 | 你的对话历史 |
| Claude 写的委派消息 | 已调用的 skill、已读过的文件 |
| CLAUDE.md 各层（Explore、Plan、`omitClaudeMd` 除外） | 主会话的 auto memory、输出风格 |
| git 状态（Explore、Plan 除外） | |
| `skills` 字段预加载的内容 | |

规则必须让子智能体知道时，在委派时重新说明。

**运行方式**
- **默认后台运行**，工具集更窄；权限提示会在主会话弹出并标明来源。交互式会话里 fork 模式默认开启，Claude 不能等待子智能体，需要等待就设 `CLAUDE_CODE_FORK_SUBAGENT=0`。
- **每次调用都是新实例**，但可以**续跑**：完成后 Claude 拿到 agent ID，用 `SendMessage` 继续，保留完整历史；Explore、Plan 不能续跑。记录单独保存，主会话压缩不影响，默认保留 30 天。
- **嵌套**默认最多 3 层（`CLAUDE_CODE_MAX_SUBAGENT_SPAWN_DEPTH`）；**同时运行**最多 20 个（`CLAUDE_CODE_MAX_CONCURRENT_SUBAGENTS`）；会话总数无上限。
- 子智能体与主会话共用用量额度。

### 四、三种"隔离执行"辨析（重点）

| | 普通 Subagent | Skill + `context: fork` | Fork 对话（`/subtask`） |
|---|---|---|---|
| 是什么 | 按定义新建的工作者 | 把 skill 正文交给一个子智能体执行 | 复制当前整段对话的子智能体 |
| 对话历史 | 看不到 | 看不到 | **完整继承** |
| 任务来自 | Claude 的委派消息 | SKILL.md 正文 | 你在 `/subtask` 后写的任务 |
| 系统提示 | 定义文件正文 | 所选 `agent` 类型 | 与主会话相同 |
| 模型 | `model` 字段 | `agent` 类型或 skill 的 `model` | 与主会话相同 |
| 提示缓存 | 独立 | 独立 | 与主会话共享，更便宜 |
| 典型用途 | 固定角色、任务多变 | 固定任务、反复做、只要结论 | 需要大量背景的旁支任务、并行尝试多种方案 |

注意：`context: fork` 名字里有 fork，但**不复制对话**，本质是一次普通 subagent 调用，相当于"带固定任务模板和命令入口的 subagent"。

**`context: fork` 写法**

```yaml
---
name: pr-summary
description: Summarize changes in a pull request
context: fork
agent: Explore
allowed-tools: Bash(gh *)
---

- PR diff: !`gh pr diff`

Summarize this pull request in 3 bullet points, then list risky changes.
```

- 正文必须是完整任务，只有参考内容的 skill 加 fork 会空手返回。
- 默认后台运行，`background: false` 改为当轮等待；后台修改不进检查点，`/rewind` 恢复不了。
- 回传总结仍进主上下文，正文里写明返回格式。

### 五、两种组合方向

| | Skill + `context: fork` | Subagent + `skills` 字段 |
|---|---|---|
| 主角 | Skill（任务写在 skill 里） | Subagent（角色写在 subagent 里） |
| skill 的作用 | 就是任务本身 | 入职时预加载的参考手册 |
| 类比 | 拿任务单找某个工种来做 | 雇固定角色，发几本手册 |

为什么不全部写成 subagent：一个 agent 类型可服务多个任务 skill；任务文本固定可复现；skill 有参数、动态注入、命令入口；inline 与 fork 只差一行。

### 六、设计决策

| 内容 | 放哪里 |
|---|---|
| 每次会话都成立的事实 | CLAUDE.md |
| 有时才需要、要结合当前对话的流程或知识 | Skill（inline） |
| 任务固定、产出大量中间内容、只要结论 | Skill + `context: fork` |
| 角色固定、任务多变、需专属工具、模型或权限 | 自定义 Subagent |
| 需要大量对话背景的旁支任务 | Fork 对话（`/subtask`） |
| 需要频繁来回、多阶段共享上下文、小改动 | 主会话直接做 |
| 对话里已有内容的小问题 | `/btw` |
| 每次必须执行的规则 | Hook 或权限规则 |
| 大规模、流程固定的批处理 | Workflow |

落地要点：
- 有副作用的 skill 设 `disable-model-invocation: true`；只读的子智能体用 `tools` 排除 Edit 和 Write，必要时加 `PreToolUse` hook 兜底（如只允许 SELECT 查询）。
- description 写短、写准，能单独指向一个 skill 或子智能体；细节放正文。
- 用 `model: haiku` 等把检索类工作路由到便宜模型。
- 定期 `/skill-doctor` 清理低频 skill；`/doctor` 检查重名的子智能体文件。
- 上线前做 baseline 评估（开关对比），可用 `skill-creator` 或 `claude plugin eval`。

### 七、安全

- Skill 的 `allowed-tools` **不受工作区信任约束**，`claude -p` 下也生效，陌生仓库先审查。
- 项目子智能体的 frontmatter hooks 和内联 MCP 服务器，要信任该文件夹才生效；`-p` 不算信任。
- 插件子智能体不支持 `hooks`、`mcpServers`、`permissionMode`。
- 子智能体报告在交给主会话前会被扫描，伪装成系统标签的文本会被转义；子智能体的消息永远不算你的批准。
- 可用 `Skill(名称)`、`Agent(名称)`、`Agent(fork)` 的 deny 规则禁用特定 skill、子智能体或 fork。

### 八、排错

- **Skill 不触发**：description 是否含用户会说的关键词；`/skill名` 能否调用；YAML 是否写坏（`--debug`）。
- **Skill 用着用着不遵守**：必须执行的规则改成 hook；指导写成全程适用；压缩后重新调用。
- **子智能体没加载**：检查 `name`、`description` 是否齐全，`---` 是否在第一行；`claude plugin validate .claude/agents` 检查 YAML。
- **字段没生效**：字段名拼错会被静默忽略，`claude --debug` 后在日志里搜 `does not recognize or apply`。
- **查看子智能体用的模型**：`/tasks`。

### ⚠️ 常见误区

- ❌ `context: fork` 会复制当前对话 → ✅ 不复制，复制对话的是 `/subtask`。
- ❌ `/context fork` 是命令 → ✅ `context: fork` 是 SKILL.md 头部字段。
- ❌ 子智能体能看到主会话的历史和已读文件 → ✅ 全新上下文，只有委派消息（fork 除外）。
- ❌ 子智能体完成后就无法继续 → ✅ 可以续跑，Explore 和 Plan 除外。
- ❌ 所有子智能体都会加载 CLAUDE.md → ✅ Explore、Plan 和设了 `omitClaudeMd` 的不加载。
- ❌ `disallowedTools: Bash(git push *)` 只拦 push → ✅ 去掉整个 Bash。
- ❌ 子智能体的 `permissionMode` 总会生效 → ✅ 主会话在 bypass、acceptEdits、auto、plan 时被忽略。
- ❌ Skill 每轮都会重读 → ✅ 调用后常驻，压缩后只保留开头。
- ❌ 在 skill 或子智能体正文里写"必须做 X"就能保证 → ✅ 只是指令，强制靠 hook 或权限。
- ❌ 子智能体不占用量 → ✅ 与主会话共用额度，回传结果也占主会话上下文。

### 📌 总结

- 📦 Skill 是"做什么"，🧑‍💼 Subagent 是"谁来做"；都是描述常驻、正文按需。
- 🔀 三种隔离执行：普通 subagent（角色）、skill + `context: fork`（任务模板）、`/subtask`（带历史的分叉）。
- 🧭 任务固定 → skill（要隔离加 fork）；角色固定 → subagent；需要背景 → fork 对话；必须执行 → hook。
- 🔁 子智能体可续跑、可嵌套 3 层、并发 20 个，默认后台运行。
- 🔐 有副作用的 skill 禁止自动调用，陌生仓库先审查 `allowed-tools` 和 hooks。
