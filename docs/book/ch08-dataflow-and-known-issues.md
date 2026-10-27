# 第八章 数据流实战：一次提问的完整旅程 + 已知问题清单

> 本章目标：把前七章的零件装配成会动的机器。走读三个真实场景（启动、提问、导入+索引），看清数据在每一层的形态变化和磁盘上的最终样子；然后盘点本项目的**已知问题清单**——七个真实存在、可复现、可修的问题，它们是第 12 章练习的题库。

## 8.1 场景一：应用启动

```
npm run dev（第 9 章讲它先编译再启动）
  └─ Electron 读取 package.json 的 "main": "dist/main/main.js"，启动主进程
       ├─ main.ts:53  app.whenReady() 等待 Electron 就绪
       ├─ main.ts:39  dataDir = %APPDATA%\knowledge-base\knowledge-base-data
       ├─ main.ts:40  new PersistenceService(dataDir)
       │     └─ 构造器 persistence-service.ts:13 → ensureDirectories()
       │          └─ mkdirSync × 3：data/、documents/、index/ 就位     ← 功能 data-directory 的证据
       ├─ main.ts:41-43  组装 DocumentService / IndexingService / QaService
       ├─ main.ts:45     registerIpcHandlers：9 条通道挂到 ipcMain
       └─ main.ts:12     new BrowserWindow(...) 创建窗口
            ├─ 加载并执行 dist/preload/preload.js（第 6 章的桥）
            │     └─ window.knowledgeBase 上桌（9 个函数）
            └─ 加载 dist/renderer/index.html
                 └─ main.tsx:8  createRoot(#root).render(<App />)
                      └─ App 首次渲染：documents=[]、状态栏显示 idle / 0
```

注意启动后界面是**空列表**——App.tsx 并没有在启动时自动调用 `refreshDocuments`（没有 `useEffect(() => { refreshDocuments(); }, [])`），需要用户点一次 Refresh 或问一个问题。这解释了"打开应用文档列表是空的"——不是 bug，是缺一行；也是练习 L2 的一部分。

## 8.2 场景二：提出一个问题

用户在输入框敲入 `what is the architecture design?` 回车。逐环追踪，注意**数据形态在每一层的变化**：

| # | 位置 | 发生什么 | 数据形态 |
|---|---|---|---|
| 1 | `QuestionPanel.tsx:10-15` | 表单提交，`e.preventDefault()` 后调 `onAsk(question)` | string |
| 2 | `App.tsx:42` | `await window.knowledgeBase.qa.ask(question)` | 发起跨进程调用 |
| 3 | `preload.ts:17` | `ipcRenderer.invoke('qa:ask', question)` | Electron 序列化后经 Mojo 通道 |
| 4 | `ipc-handlers.ts:47-49` | `handle('qa:ask')` 回调被唤起，转手给 `qaService.ask(question)` | 回到普通函数调用 |
| 5 | `qa-service.ts:52` | 假装思考 100~500ms | — |
| 6 | `qa-service.ts:54` | `indexingService.getAllChunks()`：读 `index-meta.json` 登记簿，逐文档读 `chunks/<id>.json` | `Chunk[]` |
| 7 | `qa-service.ts:59-73` | 问题分词 → 每个 chunk 打分 → 过滤/排序/top-2 | `{chunk, score}[]` |
| 8 | `qa-service.ts:76-86` | 回 `documents-meta.json` 查标题，组装引用（摘录取前 200 字符） | `Citation[]` |
| 9 | `qa-service.ts:90-97` | 模板生成答案；组装 `{answer, citations, confidence, timestamp}` | `QAResponse` |
| 10 | `qa-service.ts:100` | 追加写 `qa-history.json` | 落盘 |
| 11 | 原路返回 | Promise 一路 resolve 回渲染进程 | `QAResponse`（反序列化） |
| 12 | `App.tsx:43-44` | `setLastResponse(response)` → React 重渲染 | 界面 |

其中第 6 步是全链路的**咽喉**：如果 `index-meta.json` 登记簿是空的（从没做过全量索引），`getAllChunks()` 返回空数组，问答只能得到"请先导入并索引文档"的低置信度回复。**要看到带引用的回答，必须先走一遍无参的 `indexing.start()`**（第 11 章 11.2 的操作指南就是按这个依赖顺序写的）。

## 8.3 场景三：导入一篇文档并索引

本版本的导入入口是 DevTools 控制台（`+ Import` 按钮是假的，第 4 章讲过）：

```js
// DevTools Console 里（路径用正斜杠或双反斜杠）
await window.knowledgeBase.documents.import('D:/AICoding/xxx/data/sample-documents/design-notes.md')
```

链路：`preload.ts:8` → `ipc-handlers.ts:21-23` → `document-service.ts:23-53`：

1. 读源文件、取大小、生成 uuid；
2. 复制原件进 `documents/`，正文写进 `content/<uuid>.txt`；
3. 元数据 push 进 `documents-meta.json` 全量写回。

此时文档 `status: 'imported'`，**还没有任何 chunk**。接着索引（关键：**用无参的全量路径**）：

```js
await window.knowledgeBase.indexing.start()   // 不传参数！
```

链路：`preload.ts:12`（`documentId` 为 undefined）→ `ipc-handlers.ts:34-36` → `indexing-service.ts:37-53`：遍历元数据、跳过登记簿里已有的（增量）、逐篇切块写 `chunks/<id>.json`、更新 `index-meta.json`。

再点界面 Refresh → 列表出现文档 → 提问开始有引用。

## 8.4 磁盘上的真实数据

本机 Windows 上（`main.ts:40` 决定），你的数据就在这里（这是真实运行后的样子）：

```
C:\Users\<你>\AppData\Roaming\knowledge-base\knowledge-base-data\
├── documents-meta.json      元数据总账（Document[]）
├── documents\               原件副本
│   └── 提示词.md
├── content\                 提取的正文（一篇一文件，文件名 = 文档 id）
│   └── 76cbdda6-00de-46d5-b499-f7224ba82c8f.txt
├── chunks\                  切块结果（索引后出现）
│   └── <doc-id>.json
└── index\
    └── index-meta.json      登记簿：{ "<doc-id>": ["<chunk-id>", ...] }
```

`documents-meta.json` 的真实内容（一次成功导入后）：

```json
[
  {
    "id": "76cbdda6-00de-46d5-b499-f7224ba82c8f",
    "title": "提示词",
    "filename": "提示词.md",
    "importedAt": "2026-10-24T06:07:40.400Z",
    "size": 5849,
    "status": "imported"
  }
]
```

三个观察：`status` 停在 `"imported"` 永不变成 `"indexed"`（见问题④，没有任何代码调用 `updateDocument` 改状态）；所有文件都是**人类可读的 JSON**——这是刻意的教学选择，真实产品会用数据库；整个应用没有任何网络请求，纯本地。

## 8.5 已知问题清单（第 12 章练习题库）

以下问题全部真实存在于当前代码，按"对使用体验的影响"排序。**它们不是瑕疵，是留白**——修好任何一个，你对项目的理解都上一个台阶。

### ① 【已修复·留档】preload 沙箱加载失败

现象/根因/修复见 6.7。当前 `main.ts:21` 的 `sandbox: false` 已是修复后状态。留档原因：这是全项目唯一导致"应用完全不可用"级别的问题，复盘它的价值最高。

### ② StatusBar 数据失真：类型镜像漂移

- **现象**：点 Refresh 后，状态栏 "Index:" 和 "Documents:" 后面**空白**（首次渲染显示的 idle/0 也被冲掉）。
- **根因链**：`types.d.ts:14-15` 声明 `indexing.status()` 返回 `AppStatus`（字段 `indexStatus`/`documentsLoaded`/`lastActivity`）；但主进程 `indexing-service.ts:57-71` 实际返回 `IndexStatus`（字段 `status`/`currentIndexed`/`totalDocuments`/`lastIndexed`）。`App.tsx:24` 把实返回值整个 `setAppStatus` 进去 → `StatusBar.tsx:35-37` 读 `status.indexStatus`、`status.documentsLoaded` 全是 `undefined`。TS 不报错，因为**两个编译单元互不可见**（6.6 讲过的手写镜像问题）。
- **运行时的"保险丝"**：`StatusBar.tsx:13` 的 `?? '#888'` 和 JSX 对 `undefined` 的宽容（渲染成空）让程序不崩，只是显示空——这就是"静默出错"，比崩溃更隐蔽。
- **修法**（练习 L2-2）：让 `IndexingService.getStatus()` 直接返回 `AppStatus` 形状；或 handler 里做一次映射；并把 `IndexStatus` 接口搬进 `shared/types.ts` 统一。

### ③ `+ Import` 按钮是空壳，`ImportPanel` 组件无人使用

- **现象**：点 `+ Import` 无反应（只有 Console 一行日志，`App.tsx:30-34`）；`components/ImportPanel.tsx` 写好了文件选择控件却没被任何文件 import。
- **修法**（练习 L3-1）：渲染进程拿不到本地文件**绝对路径**（`<input type="file">` 出于安全只给文件名/内容），正解是在主进程加一条 IPC，用 Electron 的 `dialog.showOpenDialog()` 打开系统文件选择框，把绝对路径返回给渲染层。这个练习会让你完整走一遍 `CLAUDE.md` "How to Add a Feature" 的六步流程。

### ④ 单文档索引不写登记簿，状态永不更新

- 7.3 已详析：`indexing-service.ts:26-35`（单文档路径）只写 `chunks/<id>.json`，不更新 `index-meta.json`；同时全项目**没有任何代码**把文档 `status` 改成 `'indexed'`（`DocumentService.updateDocument` 存在但无人调用）。
- **连锁后果**：详情页 "Index Document" 按钮（`DocumentDetail.tsx:44-59`，条件 `document.status !== 'indexed'`）永远显示；列表永远不出现 ✓；问答检索（读登记簿的 `getAllChunks`）永远搜不到从按钮索引的文档。
- **修法**（练习 L2-3 / L3-2）：抽取登记逻辑两路共用；索引完成后调用 `documentService.updateDocument(id, { status: 'indexed', chunks: n })`——这需要让 `IndexingService` 拿到 `DocumentService`（又一次依赖注入设计）。

### ⑤ 删除文档留孤儿文件

- `document-service.ts:78-88` 只删 `documents/<filename>` 和元数据条目；`content/<id>.txt`、`chunks/<id>.json`、`index-meta.json` 登记项全部残留。
- **修法**（练习 L3-3）：删除时逐一清理；注意 `PersistenceService` 需要补一个 `deleteFile(relativePath)` 方法。

### ⑥ 启动不自动加载文档列表

- 8.1 已析：缺一个 `useEffect(() => { void refreshDocuments(); }, [])`。顺手可以把 Refresh 后的"状态+列表"请求用 `Promise.all` 并行（练习 L2）。

### ⑦ 小项合集

- `writeJson` 注释称 atomic 实则不是（7.1）。
- `types.ts:64` 的 `GET_STATUS: 'app:status'` 通道无人注册无人暴露（6.4）。
- `main.ts:26-31` isDev 两分支相同（6.3）。
- `wordCount` 按空白分词，中文计数失真（7.3）。
- `package.json` 声明了 `"test": "vitest run"` 但项目**没有任何测试文件**，`npm test` 会以 "No test files found" 失败（第 9 章 9.6）。
- 仓库没有 `.gitignore`：`node_modules/`、`dist/`、`electron-*.log` 全部处于未跟踪状态污染 `git status`（练习 L4-2）。

## 8.6 追踪数据流的方法论（带走通用技能）

本章走读用的三板斧，适用于任何项目：

1. **找一个动词**（点击/输入/启动），从界面组件的事件处理函数出发；
2. **跟着数据形态走**：每跨一层（组件→桥→IPC→服务→磁盘），问三个问题——现在的类型是什么？谁转换的？转换中丢了什么？（问题②正是丢在"声明与实现的缝隙"里）
3. **在终点看真相**：界面可能骗你（静默吞错），日志可能不全，**磁盘上的文件不会说谎**。数据没落盘，就等于没发生。

## 本章小结

- 三个场景走读：启动（先服务后窗口）、提问（12 步链路，咽喉是登记簿）、导入+索引（必须无参全量索引，问答才有引用）。
- 磁盘布局五件套：元数据总账、原件副本、正文、切块、登记簿——全是可读 JSON，随时可开文件对账。
- 七个已知问题构成练习题库：类型镜像漂移、假导入按钮、登记簿缺口、孤儿文件、启动不加载、小项合集。
- 下一章：工具链——`npm run dev` 背后那台机器（tsc、Vite、dev.js）是怎么把源码变成运行中的应用的。
