# 第六章 Electron 架构：一个应用，三个世界

> 本章目标：理解 Electron 的进程模型与安全模型，逐行精读 `main.ts`、`ipc-handlers.ts`、`preload.ts`、`types.d.ts` 四个文件。本章末尾将复盘本仓库**真实发生过的一次事故**——preload 加载失败——并把"编译通过 ≠ 运行正确"刻进你的直觉。

## 6.1 Electron 是什么

一句话：**Electron = 一个 Node.js 主进程 + 若干 Chromium 渲染进程 + 一套进程间通信机制**，打包成桌面应用。

为什么造桌面应用要这样拆？因为两种能力各自住在一个世界里：

- 画界面，浏览器（Chromium）是地表最强引擎——React、CSS、动画全是现成的。
- 碰系统（读写文件、开对话框、托盘图标），Node 有全套 API。

但这两个世界**绝不能合并**：界面显示的内容是不可信的（用户导入的文档、加载的网页都可能含恶意代码），如果画界面的进程同时握着文件系统权限，一次 XSS 就能偷走你整块硬盘。所以 Electron 把两者分进**不同进程**，中间架一座受控的桥。

```
        ┌──────────────────────────────────────────────┐
        │            主进程 Main（Node 世界）            │
        │  全部权限：窗口、文件系统、操作系统              │
        │  src/main/main.ts + ipc-handlers.ts           │
        │  src/services/*（业务逻辑也住这里）             │
        └───────────────┬──────────────────────────────┘
                        │ IPC（进程间通信，请求-响应模式）
        ┌───────────────┴──────────────────────────────┐
        │            Preload（桥，两边都沾一点）          │
        │  src/preload/preload.ts                       │
        │  把"允许暴露的能力"包装成 window.knowledgeBase  │
        └───────────────┬──────────────────────────────┘
                        │ window.knowledgeBase.* 函数调用
        ┌───────────────┴──────────────────────────────┐
        │         渲染进程 Renderer（浏览器世界）         │
        │  零系统权限，只有 DOM/JS/网络                  │
        │  src/renderer/*（React 应用）                  │
        └──────────────────────────────────────────────┘
```

【类比】这就是**客户端-服务器架构**的桌面版：渲染进程像浏览器里的前端，主进程像后端服务器，IPC 像 HTTP 请求，preload 桥像前端项目里的 API 客户端层（`api.ts`）。你在第 4 章见过的 `await window.knowledgeBase.qa.ask(question)`，语义上就是 `await fetch('/api/qa/ask')`。

## 6.2 安全模型：三把锁

`main.ts:17-22` 的 `webPreferences` 配置了前两把锁，第三把锁是 Electron 20 起的默认值：

```ts
webPreferences: {
  preload: path.join(__dirname, '..', 'preload', 'preload.js'),
  contextIsolation: true,     // 锁 1：上下文隔离
  nodeIntegration: false,     // 锁 2：渲染进程禁用 Node
  sandbox: false,             // 锁 3（本仓库当前值）：preload 沙箱开关
},
```

| 锁 | 关住谁 | 效果 |
|---|---|---|
| `nodeIntegration: false` | 渲染进程 | 页面 JS 里**没有** `require`/`fs`——就算被注入恶意脚本，也摸不到系统 |
| `contextIsolation: true` | 渲染进程 | 页面 JS 与 preload 运行在**隔离的 JS 上下文**里，页面看不到 preload 的内部变量，只有显式暴露的 API 可见 |
| `sandbox` | preload 脚本 | `true`（Electron 20+ 默认）时 preload 在沙箱中运行，只有受限的 `require`；`false` 则 preload 拥有完整 Node 能力 |

三层递进的哲学：**渲染层零权限，桥接层最小权限，特权全部收口在主进程**。渲染进程想干活，只有一条路——调用桥上显式声明的函数。

【类比 LangChain】给 agent 的不是 shell，而是一个个声明好的 tool；工具签名就是能力边界。`contextBridge.exposeInMainWorld` 暴露的 9 个函数就是这个应用的"tool 白名单"。

## 6.3 精读 `src/main/main.ts`（69 行）—— 主进程入口

按执行顺序走读：

**① 模块导入（1-7 行）**：从 `electron` 拿 `app`（应用生命周期）、`BrowserWindow`（窗口类）、`ipcMain`（IPC 服务端）；`path` 拼路径；再导入 IPC 注册器和三个服务。注意它**没有导入任何 React 代码**——`AGENTS.md` 的铁律"main never imports renderer code"。

**② 窗口引用（9 行）**：`let mainWindow: BrowserWindow | null = null;`。模块级变量保存当前窗口。

**③ createWindow（11-37 行）**：

```ts
mainWindow = new BrowserWindow({
  width: 1200, height: 800,        // 初始尺寸（feature_list 里 window-launch 的验收标准）
  minWidth: 800, minHeight: 600,   // 用户最小能拖多小
  webPreferences: { ... },          // 6.2 讲过的三把锁 + preload 路径
  title: 'Knowledge Base',
});

mainWindow.loadFile(path.join(__dirname, '..', 'renderer', 'index.html'));
// 加载界面：dist/renderer/index.html（Vite 构建产物，见第 9 章）

mainWindow.on('closed', () => { mainWindow = null; });
// 事件：窗口销毁时把引用置空，防止用到"僵尸窗口"对象
```

一个教学彩蛋：26-31 行有个 `isDev` 分支，但 if 和 else 两侧**加载的是同一个文件**——这是"预留开发模式钩子（比如将来接 Vite 开发服务器热更新）但暂未启用"的痕迹。读别人的代码时会常见这种"看似废话实则留座"的结构。

**④ initializeServices（39-51 行）**——主进程的另一半职责，组装后端：

```ts
const dataDir = path.join(app.getPath('userData'), 'knowledge-base-data');
// userData：操作系统给应用划的用户数据目录
//   Windows: C:\Users\<你>\AppData\Roaming\knowledge-base\
const persistence = new PersistenceService(dataDir);   // ← 第 5 章讲过的 DI 链
const documentService = new DocumentService(persistence);
const indexingService = new IndexingService(persistence);
const qaService = new QaService(persistence);

registerIpcHandlers(ipcMain, { documentService, indexingService, qaService });
// 把服务挂到 IPC 上，渲染进程的请求才有地方落地
```

注意时序：`initializeServices()` 在 `createWindow()` **之前**执行（53-55 行）。先开后端、再开窗口，保证界面一出现就能调 API——和"先启动服务器再让客户端连"同一个道理。

**⑤ 生命周期（53-68 行）**：`app.whenReady().then(...)`（Electron 就绪后启动，相当于 `asyncio` 的"等循环跑起来再干活"）；`activate` 事件处理 macOS 点 Dock 重开窗口；`window-all-closed` 在非 macOS 平台退出应用（5.5 节讲过）。

## 6.4 精读 `src/main/ipc-handlers.ts`（55 行）—— IPC 路由表

这个文件的角色：**后端的路由注册表**。每个 `ipcMain.handle(通道名, 处理函数)` 声明"这条通道我接，处理逻辑如下"：

```ts
// src/main/ipc-handlers.ts:7-14
export interface Services {
  documentService: DocumentService;   // 三个服务以接口形式声明——依赖注入的接收端
  indexingService: IndexingService;
  qaService: QaService;
}

export function registerIpcHandlers(ipcMain: IpcMain, services: Services) {
  const { documentService, indexingService, qaService } = services;  // 解构（第 2 章）

  ipcMain.handle(IPC_CHANNELS.LIST_DOCUMENTS, async () => {
    return documentService.listDocuments();       // 处理函数：直接委托给服务
  });

  ipcMain.handle(IPC_CHANNELS.IMPORT_DOCUMENT, async (_event, filePath: string) => {
    return documentService.importDocument(filePath);
  });
  // ……共 9 个 handler，结构完全同构
}
```

三个观察：

1. **模式极其规整**：`handle(通道, async (事件, 参数) => 服务方法(参数))`。handler 不写业务，只做"IPC 世界 → 函数调用世界"的翻译。就像 FastAPI 里 `@app.post("/qa/ask")` 下面一行 `return qa_service.ask(q)`。
2. **参数 `_event`**：每个 handler 第一个参数都是 Electron 塞进来的事件对象（含发送方信息等）。本项目用不到，前缀下划线 + tsconfig 的 `noUnusedParameters` 是 TS 社区"我knowingly不用它"的惯用写法。
3. **一处不对称**：`shared/types.ts:64` 定义了通道 `GET_STATUS: 'app:status'`，但这个文件**没有注册**它，preload 也没暴露它。这是一个"死通道"——定义了却没人用。无害，但读代码时发现这种线索要习惯性追问一句"为什么"（很可能是设计时预留、后来用了 `indexing:status` 代替）。

## 6.5 精读 `src/preload/preload.ts`（23 行）—— 唯一的桥

全文值得整段贴出：

```ts
import { contextBridge, ipcRenderer } from 'electron';
import { IPC_CHANNELS } from '../shared/types';

const api = {
  documents: {
    list: () => ipcRenderer.invoke(IPC_CHANNELS.LIST_DOCUMENTS),
    import: (filePath: string) => ipcRenderer.invoke(IPC_CHANNELS.IMPORT_DOCUMENT, filePath),
    get: (id: string) => ipcRenderer.invoke(IPC_CHANNELS.GET_DOCUMENT, id),
    delete: (id: string) => ipcRenderer.invoke(IPC_CHANNELS.DELETE_DOCUMENT, id),
  },
  indexing: {
    start: (documentId?: string) => ipcRenderer.invoke(IPC_CHANNELS.START_INDEXING, documentId),
    status: () => ipcRenderer.invoke(IPC_CHANNELS.GET_INDEXING_STATUS),
    chunks: (documentId: string) => ipcRenderer.invoke(IPC_CHANNELS.GET_CHUNKS, documentId),
  },
  qa: {
    ask: (question: string) => ipcRenderer.invoke(IPC_CHANNELS.ASK_QUESTION, question),
    history: () => ipcRenderer.invoke(IPC_CHANNELS.GET_HISTORY),
  },
};

contextBridge.exposeInMainWorld('knowledgeBase', api);
```

设计要点：

- **每一项都是 `() => ipcRenderer.invoke(通道, 参数)` 的薄包装**。`invoke` 发起一次请求-响应式 IPC（返回 Promise），对端就是 6.4 的某个 `handle`。通道名从 `IPC_CHANNELS` 引入——**通道名跨三个文件共享同一份常量**，拼错即编译错误。
- **为什么不直接暴露 `ipcRenderer` 本身？** 那等于把整条高速公路交给界面：任何页面代码都能向**任意**通道发**任意**消息。包装成 `api` 对象后，暴露面被冻结为 9 个函数、每个函数的参数形状固定——最小暴露原则。（`import(filePath)` 的参数依然是"任意路径"，安全性并不完美，但比裸 ipcRenderer 好得多。安全是一段光谱。）
- **分组命名空间**：`documents`/`indexing``/qa` 三组，调用侧读起来像 `window.knowledgeBase.qa.ask(...)`——REST 风格的资源分组搬进了 JS 对象。
- **执行时机**：Electron 在加载页面前先执行 preload，所以页面脚本开跑时 `window.knowledgeBase` 一定已就绪。（"一定已就绪"的前提是 preload 成功加载——见 6.7 的事故。）

## 6.6 精读 `src/renderer/types.d.ts` —— 类型世界的"补丁"

渲染进程要调用 `window.knowledgeBase`，但 `window` 是浏览器环境自带的类型，TS 并不知道上面多了个 `knowledgeBase`。这个文件就是补丁：

```ts
declare global {
  interface Window {
    knowledgeBase: {
      documents: {
        list: () => Promise<import('../shared/types').Document[]>;
        // ...每个方法的精确签名
      };
      // ...
    };
  }
}
export {};   // 让本文件成为"模块"，declare global 才生效（TS 的机制性要求）
```

两个语法点：`declare global` 声明"全局存在此物"，全项目无需 import 直接可用；`import('../shared/types').Document` 是**类型内联导入**——在类型位置现场引入另一个模块的类型，不产生运行时代码。

一个需要严肃对待的设计弱点：这份声明是**手写的镜像**，编译器**不会**核对它和 preload 的真实实现是否一致——`types.d.ts` 属于 renderer 的编译单元（`tsconfig.json`），preload 属于另一个（`tsconfig.node.json`），两者从不"见面"。手写镜像就会漂移，而漂移在本项目里真实发生了：声明说 `indexing.status()` 返回 `AppStatus`（含 `indexStatus`/`documentsLoaded` 字段），主进程实际返回的却是 `IndexStatus`（字段名为 `status`/`currentIndexed`/`totalDocuments`，见 `indexing-service.ts:57-71`）。运行时后果与修复方案在第 8 章 8.4 节盘点（它是个很好的练习题）。

## 6.7 事故复盘：本仓库真实的 preload 加载失败

`electron-stderr.log`（运行 Electron 时 stderr 重定向的产物）记录过这样一段：

```
Unable to load preload script: ...\dist\preload\preload.js
Error: module not found: ../shared/types
```

**症状**：应用窗口照常打开，但 `window.knowledgeBase` 是 undefined——点 Refresh 报错、文档列表永远空白。界面层一切"看起来正常"，功能全瘫。

**排查链**（第 11 章方法论的一次实战预演）：

1. 报错指向 `dist/preload/preload.js` 第 4 行：`require("../shared/types")`。
2. 检查磁盘：`dist/shared/types.js` **存在**（tsc 编译 shared 目录的产物）——不是文件缺失。
3. 那为什么找不到？回想 6.2 的第三把锁：**Electron 20 起 preload 默认在沙箱中运行，沙箱版的 `require` 是受限实现，无法解析这种不带扩展名的相对路径模块引用**（社区资料见 electron-vite 文档与 electron issue #36437 的同类讨论）。
4. TypeScript 编译器对此**一无所知**：`import '../shared/types'` 完全合法（它按 TS/Node 规则解析），但运行 preload 的沙箱加载器不按完整 Node 规则解析。**两层规则的错位，编译期无法发现**。

**修复选项与权衡**：

| 方案 | 做法 | 代价 |
|---|---|---|
| A. 关沙箱 | `webPreferences` 加 `sandbox: false` | preload 恢复完整 Node require，一行修复。安全性略降（本项目 preload 极薄、可接受）。**本仓库采用的正是它**（`main.ts:21`，修复后导入功能实测恢复） |
| B. 显式扩展名 | 源码改成 `import '../shared/types.js'` | TS 能把 `.js` 映射回 `.ts` 源文件编译，产物 require 带扩展名后部分环境可解析；但对沙箱加载器的兼容性仍取决于 Electron 版本，属于"碰运气" |
| C. 打包 preload | 用 Vite/webpack 把 preload 连依赖打成单文件（electron-vite 的默认做法） | 引入构建复杂度，工业界的标准解 |
| D. 零依赖 | 把 `IPC_CHANNELS` 的字符串直接内联进 preload.ts，不 import 任何本地模块 | 牺牲"单一来源"原则，但 preload 天然零依赖、绝对稳 |

**教训**（本章最重要的一段）：类型检查和编译都通过了，程序照样可以在运行时死掉——因为**运行环境（沙箱加载器）的规则不在编译器的知识范围内**。"编译通过"只覆盖编译器看得见的世界。这也是 `AGENTS.md` 把"app launches and the window is visible"写进 Definition of Done 的原因：光编译过不算完，必须真的跑起来看。

## 6.8 IPC 一来一回全景图

把本章与第 4 章串起来，"渲染进程问一个问题"的完整链路：

```
[Renderer] QuestionPanel 用户回车
    → onAsk(question) → App.handleAskQuestion
    → window.knowledgeBase.qa.ask(question)          [types.d.ts 提供类型]
[Preload]  ipcRenderer.invoke('qa:ask', question)     [preload.ts:17]
    ═══════ 跨进程边界（Chromium 的 Mojo 通道） ═══════
[Main]     ipcMain.handle('qa:ask') 的回调            [ipc-handlers.ts:47]
    → qaService.ask(question)                        [qa-service.ts:50]
    → 读磁盘 chunks → 检索打分 → 生成回答
    → return QAResponse（Electron 自动序列化/反序列化）
    ═══════ 原路返回（Promise resolve） ═══════
[Renderer] await 拿到 response → setLastResponse(response)
    → React 重渲染 → 用户看到回答与引用
```

第 8 章会把这条链路的每一环配上代码行号再走一遍，并看清数据在磁盘上的模样。

## 本章小结

- Electron = Node 主进程（全权限）+ Chromium 渲染进程（零权限）+ preload 桥（最小暴露），IPC 采用 `invoke`/`handle` 请求-响应模式。
- 三把安全锁：`nodeIntegration: false`、`contextIsolation: true`、`sandbox`（默认 true，本项目为修 preload 显式关成 false）。
- `main.ts` = 窗口 + 服务组装（先服务后窗口）；`ipc-handlers.ts` = 路由表（handler 只翻译不做业务）；`preload.ts` = 9 个函数的受控白名单；`types.d.ts` = 手写类型镜像（会漂移，本项目真的漂了）。
- 事故教训：编译通过 ≠ 运行正确；沙箱加载器的限制编译器看不见。凡跨边界（进程/沙箱/网络），一定要真跑一次验证。
- 下一章进入主进程腹地：四个服务类的逐行精读，包括那个 500 字符切块算法。
