# 第十章 真正的主角：Harness 工程

> 本章目标：理解这门课的灵魂——为什么"模型很强"和"模型可靠地完成工程任务"是两回事；五个 harness 文件各自解决什么问题；starter/solution 对照实验怎么做；以及这套思想如何与你熟悉的 LangChain/LangGraph 世界对接。

## 10.1 核心命题：能力 ≠ 可靠交付

课程 README 引用了两个业界实验（Anthropic 与 OpenAI 各自的结论），大意相同：**同一个模型，在没有环境约束时产出不可用，在完整环境（规划+生成+评估）下产出可交付**。模型没变，变的是模型周围的结构。

不用实验你也一定有体感——AI 写代码的三大经典失败：

1. **提前宣告胜利**："已完成"——但没编译过、没跑过、或只跑了它自己写的"测试"。
2. **跨会话失忆**：昨天做到一半，今天重开会话，一切从零开始，甚至推翻重来。
3. **越界与半途**：一次改五个功能，每个改一半；顺手重写你没让它动的文件。

harness engineering 的主张：这些问题**不靠更聪明的模型解决，靠环境设计解决**。模型决定写什么代码；harness 决定它何时、何地、按什么标准、做到什么程度才算完成。

【类比 LangChain】你在搭 agent 时早就实践过这件事：不给 shell 而给白名单 tools（控制能力边界）、用 checkpointer 存状态（跨步续跑）、在 graph 里放 interrupt 等人审核（完成的门槛）。harness engineering 是把同样的纪律推广到"agent 在真实代码仓库里长期干活"的场景。

## 10.2 五个子系统 × 本仓库的文件

课程把 harness 拆成五个子系统。本仓库（solution/）每样放了一个最小实现：

### ① Instructions（指令）——`AGENTS.md` + `CLAUDE.md`

`AGENTS.md` 是智能体的**操作手册**，结构值得逐段看：

- **Startup Rules（启动规则）**：五步固定开场——读我 → 读架构文档 → 读产品文档 → 跑 `init.sh` 验证 → 读功能清单。作用：**消灭"从哪里开始"的自由发挥**。智能体（或新人）最容易犯的错不是"不会写"而是"没读就写"。
- **层边界（Electron Layer Boundaries）**：四层各一段"住在哪里、能做什么、绝不做什么"。没有这段，智能体会"顺手"在渲染层 import fs 来"快速实现功能"——它能跑，但架构烂了。
- **Conventions（约定）**：strict 模式、命名导出、IPC 通道名单一来源、异步规范——把代码风格从"每次现场决定"变成"照章办事"。
- **Definition of Done**：五条完成标准（见 10.4）。

`CLAUDE.md` 第一行是 `@AGENTS.md`——引用语法，意为"全文并入"。CLAUDE.md 只补充 Claude Code 特有的快捷信息（关键文件表、加功能六步流程）。**为什么要两个文件而不是一个**：不同的 coding agent 读不同的默认文件名（AGENTS.md 是跨工具约定，CLAUDE.md 是 Claude Code 的默认），用引用避免两份内容漂移。这也是课程 L04 讲的"渐进披露"思想的反面教材规避法：**任何内容只写一处**。

### ② State（状态）——`feature_list.json` + `claude-progress.md` + git 历史

跨会话记忆三件套，各管一段：

**`feature_list.json`**：机器可读的进度账本。看它的一条真实记录：

```json
{
  "id": "window-launch",
  "name": "Window Launch",
  "description": "Electron app opens a BrowserWindow with correct dimensions and preload script",
  "status": "pass",
  "evidence": "npm run dev launches window at 1200x800 with contextIsolation=true and nodeIntegration=false",
  "testedAt": "2026-03-30T10:00:00Z"
}
```

精妙之处在 **evidence（证据）字段**：不是"做完了"三个字，而是"怎么验证的"。"pass + 证据"与"我觉得行了"之间隔着整门课的距离——这正是 L09（为什么智能体会提前宣告胜利）要解决的问题。规则同时写明：不许删除条目、被阻塞就标 fail 并写原因——**账本只许补记，不许篡改**。

**`claude-progress.md`**：会话日志，人类可读。看它现有的一条记录结构：目标 / 做了什么 / 决策 / 问题 / **下个会话做什么**。最后一项是接力棒：新会话读它，从断点继续，而不是重新规划。

**git 历史**：最硬的状态。代码层面的真相永远以已提交的快照为准（第 9 章 9.7 讲过未提交修改的微妙之处）。

### ③ Verification（验证）——`init.sh` + 各 npm scripts

`init.sh` 全文 23 行，三步：`npm install`（环境就绪）→ `npm run check`（类型检查）→ `npm run build`（完整构建）。任何一步失败，`set -euo pipefail` 让脚本立刻非零退出。

它的角色是**开工体检**：智能体（或人）在动手前先确认"基线是绿的"。没有它，坏基线 + 新修改 = 算不清谁的锅；有了它，任何失败都指向本次修改。课程的 L06 专门讲为什么"初始化值得单独一个阶段"。

验证的完整梯队（本项目用到的）：类型检查（check）→ 构建（build）→ 启动冒烟（`npm run dev` 看窗口）→ 单元测试（vitest，**当前是空位**，第 9 章 9.6）→ 端到端验证（feature_list 的 evidence 记录）。注意梯队有洞：测试为零、evidence 靠人写——所以 P05/P06 课程项目就是去补这些洞。

### ④ Scope（范围）——功能粒度 + Definition of Done

`feature_list.json` 把工作切成**一次一个功能**的颗粒。配套规则（AGENTS.md 的 Working with the Feature List 一节）：做一个、验一个、记一个。对比"一次改五个"：颗粒化让每步都有明确的完成判定，也让回滚单位变得清晰。

【类比 LangChain】把大任务拆成 graph 的节点、每个节点有明确的输入输出和成功判据——同一思想，粒度即护栏。

### ⑤ Session Lifecycle（会话生命周期）——完整的闭环

把四个文件串成一个会话的剧本（课程的标准流程）：

```
开工：读 AGENTS.md → 跑 init.sh（体检）→ 读 progress.md（上次到哪了）
      → 读 feature_list.json（挑恰好一个未完成功能）
干活：实现 → 验证（check/build/运行）→ 失败则修复重验 → 通过则记证据
收尾：更新 progress.md（本次做了什么/下会话做什么）→ 更新 feature_list.json
      → 提交（只在"可安全续跑"的状态提交）→ 留干净的重启路径
```

注意"只在干净状态提交"：半成品要么收完要么明确标注未完成——绝不留一个"看起来完成"的烂摊子给下个会话。

## 10.3 对照实验：starter vs solution 怎么做

project-01 的**任务本体**不是写应用，而是做这个实验（`projects/project-01/README.md` 的 Exact Task Contract）：

1. **弱 harness 组**：进 `starter/`，把 `task-prompt.md` 的那一句话（"用 Electron 做一个知识库应用，左边文档列表右边问答面板，要建本地数据目录"）直接交给一个 coding agent，不给任何其他文件。观察它做什么。
2. **强 harness 组**：进 `solution/`，让 agent 先读 AGENTS.md / init.sh / feature_list.json / claude-progress.md 再干活。
3. **对比记录**（README 列的观察指标）：任务完成了吗？重试了几次？agent 是否过早宣布"完成"？四个验收点（窗口启动 / 文档列表面板 / 问答面板 / 数据目录）各自的真实状态。

预期（也是课程核心论点）：弱组大概率产出"能看但不能跑"或"跑了但四缺三"的结果，且倾向口头宣布完成；强组的行为被文件约束：先体检、一次一个功能、拿证据说话。

**你不需要 coding agent 也能从这个实验里学到东西**：把两组"开工指令"拿来对比自问——如果我是那个收到指令的新工程师（或新 AI），哪组指令能让我第一次就做对？答案就是 harness 的价值：**它是给"没有上下文的执行者"设计的接口**。你的同事、三个月后的你、和 AI，都是这种执行者。

## 10.4 Definition of Done 逐条解剖

`AGENTS.md` 的完成定义五条，每条都对应可执行命令/动作——这是"完成的门槛"从口号变成闸门的关键：

| 条款 | 验证方式 | 挡住什么 |
|---|---|---|
| 1. TypeScript 编译无错误 | `npm run check` 退出码 0 | 语法/类型层面的烂尾 |
| 2. 应用启动且窗口可见 | `npm run dev` 亲眼看到窗口 | "编译过但跑不起来"（6.7 事故正是这类！） |
| 3. feature_list 有 pass + 证据 | 查看该条目的 status/evidence | 口头完成 |
| 4. 遵守层边界 | 代码评审：import 关系对不对 | 架构腐蚀 |
| 5. 正常操作无控制台报错 | DevTools Console 干净 | 静默错误（第 8 章问题②那种"显示空白"） |

特别看第 2 条——它存在的理由就是 6.7 复盘的教训：**类型检查通过不能证明运行正常**。条款 2 强制一次真实的启动。这就是 L09 的主题：验证必须对准"真实失败模式"，而不是"容易检查的东西"。

## 10.5 与你的 LangChain/LangGraph 经验对接

一张翻译表，把 harness 五子系统映射到你熟悉的 agent 工程概念：

| Harness 子系统 | LangChain/LangGraph 对应 | 共同原理 |
|---|---|---|
| Instructions（AGENTS.md） | system prompt + tool 使用说明 | 把"怎么干活"从对话内记忆固化为外部文档 |
| State（feature_list/progress） | checkpointer（MemorySaver/SqliteSaver）+ 状态图 | 状态外置持久化，跨执行周期续跑 |
| Verification（init.sh/DoD） | graph 里的校验节点 / interrupt 审核门 | 完成判据可执行、自动化，不靠自觉 |
| Scope（功能粒度） | 节点边界 + 每节点的输入输出 schema | 颗粒化控制爆炸半径 |
| Lifecycle（会话闭环） | graph 的 entry/END + 断点恢复 | 每次运行从干净起点到干净终点 |

一个反直觉的共性：两边都不追求"让模型更聪明"，只追求**让失败尽早暴露、让进度可累积、让边界不可逾越**。你在 LangGraph 里不会指望换一个更强的模型就消除状态丢失——同样，coding agent 的可靠性来自仓库结构，不来自模型版本号。

## 10.6 马上可迁移的一步

课程提供 `harness-creator` 技能（仓库 `skills/` 目录）可以给你的项目生成一套 harness 文件。但手写一遍更能理解，给你自己的 LangGraph 项目最小起步三件套：

1. `AGENTS.md`：写清目录结构、启动命令、层边界（如"node 层不许 import ui 层"）。
2. `feature_list.json`：把计划中的 graph 能力列成条目，做一条标一条，附验证命令（如 `python -m pytest tests/test_retrieval.py`）。
3. `init.sh` 或 `init.ps1`：`pip install -r requirements.txt && pytest`，开工先跑绿。

下个会话（无论对话者是 AI 还是你自己）的世界会完全不同。

## 本章小结

- 命题：模型能力 ≠ 可靠交付；可靠性来自环境设计（harness），不来自更聪明的模型。
- 五子系统 × 五文件：AGENTS.md（指令）、feature_list.json + progress.md + git（状态）、init.sh + scripts（验证）、功能粒度 + DoD（范围）、会话闭环（生命周期）。
- 实验设计：同一任务、两种开工指令（一句话 vs 一套文件），对比完成率/重试/提前宣布——这个对比本身就是课程内容。
- DoD 五条的可贵之处：每条都能落到一条可执行命令；条款 2（窗口必须真的开出来）正是 6.7 事故的制度化免疫。
- 与 LangGraph 的对应：checkpointer ↔ 状态文件、interrupt ↔ DoD 闸门、节点边界 ↔ 功能粒度。可靠性工程思想跨领域通用。
- 下一章（也是动手章）：把应用跑起来、真实地用一遍、学会排错的基本功。
