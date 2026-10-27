# 第一章 全景：你拿到的是一个什么项目

> 本章目标：10 分钟后，你能回答三个问题——这个项目是干什么的？它由哪些部分组成？课程的真正主题为什么不是"写这个应用"，而是"给 AI 编程智能体搭环境"？

## 1.1 你手里的是什么：一个套娃结构

打开这个仓库，你实际上同时拿着**三层东西**，由外到内：

```
┌─────────────────────────────────────────────────────────────┐
│ 第 1 层：一门课程                                             │
│   learn-harness-engineering（学习"harness 工程"的课程）        │
│   ┌─────────────────────────────────────────────────────┐   │
│   │ 第 2 层：一个对照实验                                  │   │
│   │   starter/（只有一句模糊提示） vs solution/（你现在 here）│   │
│   │   ┌─────────────────────────────────────────────┐   │   │
│   │   │ 第 3 层：一个真实的应用                        │   │   │
│   │   │   Electron 桌面知识库：导入文档→分块索引→问答    │   │   │
│   │   │   TypeScript + React + Node.js，共 1121 行    │   │   │
│   │   └─────────────────────────────────────────────┘   │   │
│   └─────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────┘
```

- **应用层**（第 3 层）是你的"第一个项目"本体：一个能跑、能点、能把你的 `.md` 文件切成小块再回答问题的桌面程序。本书第 2～9 章逐行讲解它。
- **实验层**（第 2 层）是课程设计：同样的任务给 AI 智能体做两遍——一遍只给一句话提示（`starter/task-prompt.md`，中文原文只有一行："用 Electron 做一个知识库应用，窗口左边是文档列表区域，右边是问答面板区域……"），另一遍给完整的规则文件（`AGENTS.md`、`feature_list.json`、`init.sh`）。对比两次的结果差异。第 10 章详细讲。
- **课程层**（第 1 层）的主题叫 **harness engineering（挽具工程 / 环境工程）**：模型能力已经很强，但在真实仓库里长期可靠地干活，需要在外围搭一套"环境"——指令、状态、验证、范围、生命周期。这个 `solution/` 目录本身就是一套示范环境。

【类比】如果你用过 LangChain/LangGraph：你早就体会过"光有强模型不够"——需要给它配 tools 白名单、memory、checkpointer、interrupt 机制。harness engineering 讲的就是同一件事，只不过对象从"对话里的 agent"扩大到"在真实代码仓库里干活的 coding agent"。

## 1.2 这个应用做什么

用一个用户视角的故事描述：

1. 你启动应用，看到一个 1200×800 的深色窗口：左边是文档列表，右边是详情和问答区，底部是状态栏。
2. 你导入一篇 `.txt` 或 `.md` 文档（本版本通过开发者控制台导入，第 11 章教你具体操作）。
3. 你点击"建立索引"：程序把文档按**段落边界切成约 500 字符的块（chunk）**存到磁盘。
4. 你在底部输入框提问，比如 `what is the architecture design?`。
5. 程序用关键词匹配从所有 chunk 里检索最相关的 2 块，生成一段回答，并附上**引用（citation）**——告诉你答案出自哪篇文档的第几块。有引用时置信度 0.85，没有则 0.30。

界面布局（来自 `docs/PRODUCT.md`）：

```
+------------------+----------------------------------------+
| Header: Knowledge Base                        [Refresh] |
+------------------+----------------------------------------+
| Document List    |  Document Detail / 欢迎语               |
| (左侧栏)         |  Q&A 回答 + 引用列表                     |
| [+ Import]       |                                        |
+------------------+----------------------------------------+
| Question Input                                    [Ask]  |
+-----------------------------------------------------------+
| Status: idle | Documents: 0                               |
+-----------------------------------------------------------+
```

注意：问答部分是**假的 LLM**——`QaService` 里写死了 5 组"关键词→标准答案"的模板（`src/services/qa-service.ts:8-38`），检索部分才是真的（基于关键词重合度打分）。这是课程刻意为之：先把"RAG 的骨架"搭对，第 12 章的练习里你可以把假 LLM 换成真 LLM。

【类比】整个应用 = 一个**本地迷你 RAG 系统**：

| 本项目 | LangChain 里的对应物 |
|---|---|
| `IndexingService.chunkDocument()` | `TextSplitter`（按段落切 500 字符） |
| `QaService.ask()` 里的打分排序 | `Retriever`（这里是关键词重合，不是向量相似度） |
| `MOCK_PATTERNS` + `generateAnswer()` | prompt + LLM（这里是写死的模板） |
| `QAResponse{answer, citations, confidence}` | 带引用的结构化输出（grounded output） |

## 1.3 仓库地图：每个文件是干什么的

```
solution/
├── AGENTS.md                  ★ harness：智能体的"操作手册"（启动规则/分层边界/完成定义）
├── CLAUDE.md                  ★ harness：给 Claude Code 的快捷参考，第一行 @AGENTS.md 引入上文
├── feature_list.json          ★ harness：功能清单+状态+证据（机器可读的进度账本）
├── claude-progress.md         ★ harness：会话日志（上个会话做了什么、下个会话做什么）
├── init.sh                    ★ harness：开工体检脚本（安装→类型检查→构建）
│
├── docs/
│   ├── ARCHITECTURE.md          架构文档（四层结构图）
│   ├── PRODUCT.md               产品需求文档
│   └── book/                    ← 你正在读的这本书
│
├── src/                        —— 应用源码，共 1121 行 ——
│   ├── main/                    【主进程】Node 世界，掌管窗口和文件系统
│   │   ├── main.ts              入口：创建窗口、组装服务（68 行）
│   │   └── ipc-handlers.ts      把 IPC 通道名映射到服务方法（55 行）
│   ├── preload/                 【预加载】两个世界之间唯一的桥
│   │   └── preload.ts           contextBridge 暴露 window.knowledgeBase API（23 行）
│   ├── renderer/                【渲染进程】React 世界，只管界面
│   │   ├── index.html           唯一的 HTML 页面（暗色主题内联样式）
│   │   ├── main.tsx             React 挂载入口（13 行）
│   │   ├── App.tsx              根组件：布局+状态+事件（157 行）
│   │   ├── types.d.ts           window.knowledgeBase 的类型声明
│   │   └── components/          5 个子组件
│   │       ├── DocumentList.tsx     左侧文档列表（45 行）
│   │       ├── DocumentDetail.tsx   右侧详情+分块展示+索引按钮（87 行）
│   │       ├── ImportPanel.tsx      导入面板（写了但没被使用！见第 8 章）
│   │       ├── QuestionPanel.tsx    底部提问输入框（61 行）
│   │       └── StatusBar.tsx        底部状态栏（43 行）
│   ├── services/                【服务层】纯业务逻辑，跑在主进程里
│   │   ├── persistence-service.ts  文件读写（唯一碰磁盘的类，94 行）
│   │   ├── document-service.ts     文档增删改查（89 行）
│   │   ├── indexing-service.ts     切块与索引（130 行）
│   │   └── qa-service.ts           问答（135 行）
│   └── shared/
│       └── types.ts             跨层共享的类型 + IPC 通道名常量（65 行）
│
├── scripts/
│   └── dev.js                  npm run dev 背后的三步脚本（编译→打包→启动）
├── package.json                项目说明书：依赖+脚本命令
├── tsconfig.json               TypeScript 配置（渲染进程用）
├── tsconfig.node.json          TypeScript 配置（主进程/preload/服务用）
├── vite.config.ts              Vite 打包配置（只打包渲染进程）
├── data/sample-documents/      3 篇示例文档，供你导入试验
├── init.sh                     开工体检：npm install → check → build
├── dist/                       构建产物（编译后的 JS，可随时删除重建）
└── node_modules/               第三方依赖（npm install 生成，永不手改）
```

★ 标记的 5 个文件就是课程所说的 **harness 文件**——它们不参与应用运行，而是给"在仓库里干活的 AI（或人）"定规矩。注意一个关键事实：**应用源码只占仓库文件数的一半左右，另一半是给人和智能体看的过程设施**。这是这门课最想让你建立的直觉。

## 1.4 四层架构鸟瞰

应用源码被切分进四个严格的层，数据自上而下请求、自下而上返回：

```
+-----------------------------------------------------------+
|  渲染进程 Renderer（React，跑在 Chromium 里）                |
|  App.tsx → DocumentList / DocumentDetail /                 |
|            QuestionPanel / StatusBar                       |
+-----------------------------------------------------------+
        │  调用 window.knowledgeBase.* （类型化的 IPC 桥）
+-----------------------------------------------------------+
|  Preload（桥接层，先于页面加载）                              |
|  contextBridge.exposeInMainWorld → documents/indexing/qa   |
+-----------------------------------------------------------+
        │  ipcRenderer.invoke('qa:ask', question) 等等
+-----------------------------------------------------------+
|  主进程 Main（Node.js，拥有全部权限）                        |
|  main.ts → createWindow() + initializeServices()           |
|  ipc-handlers.ts → registerIpcHandlers()                   |
+-----------------------------------------------------------+
        │  普通方法调用（同进程内）
+-----------------------------------------------------------+
|  服务层 Services（纯业务逻辑）                               |
|  DocumentService │ IndexingService │ QaService             |
|  PersistenceService（唯一做文件 I/O 的类）                   |
+-----------------------------------------------------------+
```

为什么搞这么麻烦？一句话：**安全**。渲染进程显示的内容可能被攻击者控制（比如你导入了一篇恶意文档），所以它必须被关在"浏览器沙箱"里——碰不到文件系统、碰不到操作系统。所有特权操作都收口在主进程，渲染进程只能通过一条窄窄的、显式声明的通道（preload 桥）发请求。第 6 章会逐文件剖析这套机制，包括它在本仓库里真实翻过的一次车。

【类比】这套"隔离 + 窄接口"的思路你在 LangChain 里见过：给 agent 的不是 shell 权限，而是一个个声明好的 tool——它能调用的只有白名单里的函数。preload 桥就是渲染进程的"tool 白名单"。

## 1.5 课程的真正主题：harness 五个子系统

先混个脸熟，第 10 章展开。课程认为一个可靠的 coding agent 环境由五个子系统组成，本仓库恰好每样都放了一个文件做最小示范：

| 子系统 | 作用 | 本仓库的对应物 |
|---|---|---|
| Instructions（指令） | 告诉智能体先读什么、按什么顺序做 | `AGENTS.md`（启动规则）、`CLAUDE.md` |
| State（状态） | 记录做到哪了，跨会话不丢 | `claude-progress.md`、`feature_list.json`、git 历史 |
| Verification（验证） | 只有可运行的证据才算完成 | `npm run check`、`npm run build`、`init.sh` |
| Scope（范围） | 一次只做一个功能，不许越界 | `feature_list.json` 的功能粒度、"Definition of Done" |
| Lifecycle（生命周期） | 开工体检、收尾清理 | `init.sh`（开工）、DoD 检查（收尾） |

`starter/` 目录里这五样**一样都没有**，只有一句模糊的任务描述——这就是实验的自变量。

## 1.6 这本书怎么配合代码读

- 全书代码引用格式为 `路径:行号`，如 `src/main/main.ts:17`，行号以你本地文件为准（若你改过代码可能有少量偏移）。
- 第 2～5 章是"语言与平台速成"，解决"读不懂语法"的问题；第 6～8 章是"本项目精读"，解决"为什么这样设计"的问题；第 9 章讲工具链；第 10 章点题 harness；第 11 章教你跑起来并排错；第 12 章给你分级练习。
- **强烈建议边读边跑**：先照第 11 章第 1 节把应用跑起来，再回来读第 2 章。看到任何一行代码有疑惑，就打开对应文件对照。

## 本章小结

- 你拿到的是"课程 → 对照实验 → 真实应用"的三层套娃；应用本体是一个 1121 行的 Electron 知识库程序，本质是本地迷你 RAG。
- 源码分四个严格分层：renderer（界面）→ preload（桥）→ main（窗口+IPC）→ services（业务+文件）。
- 仓库里一半文件不是给应用用的，而是给"干活的智能体"用的——这是本课程的核心：模型决定写什么代码，harness 决定它何时、何地、以什么标准写。
- 下一章：从 Python 和 C 的直觉出发，两小时掌握读懂本项目所需的全部 TypeScript。
