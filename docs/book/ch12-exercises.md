# 第十二章 动手练习与进阶路线

> 本章目标：把读到的东西变成做出来的东西。12 道分级练习（L1 热身 → L5 进阶），每道标注目标、涉及文件、提示和**验收标准**——验收标准全部按第 10 章 Definition of Done 的风格写，做完请真的逐条执行。最后给一条后续学习路线。

## 12.0 通用纪律（做完任何一题都执行）

1. `npm run check` 通过；
2. `npm run dev` 启动无报错（DevTools Console 干净）；
3. 在 `feature_list.json` 里**新增**一条（不动别人的条目），写 id、描述、status、evidence；
4. `git add -A && git commit -m "练习 L?-?：一句话说明"`。

这四步就是 harness 的小型演练——从第一次修改开始养成。

## 12.1 L1 热身（每题 10~30 分钟，只碰渲染层）

### L1-1 改窗口与品牌

- **目标**：窗口标题改为"我的知识库"，初始尺寸改为 1000×700。
- **文件**：`src/main/main.ts:12-23`。
- **验收**：重启后窗口标题栏与尺寸正确；`feature_list` 记录证据。

### L1-2 换一套主题色

- **目标**：把暗色蓝紫主题（第 3 章的五色板）换成暖色或绿色系，全项目颜色成套变化。
- **文件**：`index.html` 的 `<style>` + 各组件的 `style` 对象（全局搜 `#16213e` 等色号）。
- **提示**：先画一张旧→新色号映射表再动手，改完对比截图。
- **验收**：无残留旧色号（全局搜索确认）；界面和谐。

### L1-3 状态栏多显示一个数字

- **目标**：状态栏在 Documents 旁边再显示 `Indexed: N/total`。
- **文件**：`StatusBar.tsx`；数据来自 `indexing.status()`（注意字段名，见 L2-2 的前置坑——这道题先允许显示不完美）。
- **验收**：导入+索引后显示 `Indexed: 3/3`。

## 12.2 L2 修真 bug（每题 30~60 分钟，跨层追数据）

### L2-1 启动自动加载文档列表

- **目标**：应用一打开就显示已有文档与状态，不需要手点 Refresh。
- **文件**：`src/renderer/App.tsx`。
- **提示**：`useEffect(() => { void refreshDocuments(); }, [refreshDocuments])`；注意依赖数组为什么放 `refreshDocuments`（它是 useCallback 的产物）。
- **验收**：重启应用（此前已导入过文档），列表自动出现。

### L2-2 修类型镜像漂移（第 8 章问题②）

- **目标**：Refresh 后状态栏正确显示 `ready` 和文档数，而不是空白。
- **文件**：`src/services/indexing-service.ts`（`getStatus` 的返回形状）、`src/shared/types.ts`、`src/renderer/types.d.ts`。
- **提示**：推荐做法——把 `IndexStatus` 接口搬进 `shared/types.ts`，让 `getStatus()` 返回与 `AppStatus` **完全一致**的字段名（或定义一个映射函数），再让 types.d.ts 的声明引用真实类型。
- **验收**：Console 无报错；状态栏文本正确；`npm run check` 通过。
- **思考题**：为什么 TS 没有提前发现这个错？（答案在第 6 章 6.6，能向别人讲明白才算过关。）

### L2-3 让 ✓ 出现：索引后更新文档状态

- **目标**：全量索引完成后，每篇文档 `status` 变为 `'indexed'` 且 `chunks` 数量写入；列表出现 ✓，详情页 Index 按钮消失。
- **文件**：`indexing-service.ts`（全量路径的循环里）、`document-service.ts`（现成的 `updateDocument` 没人用——用起来）。
- **提示**：`IndexingService` 目前只有 `persistence`，没有 `DocumentService`。改 `main.ts:42-44` 的组装代码，把 `documentService` 也注入进来（构造函数加参）。
- **验收**：`documents-meta.json` 里 status 全部 `"indexed"` 且带 chunks 数；界面 ✓ 出现。

## 12.3 L3 补全功能（每题 1~2 小时，完整走六步流程）

> 这一级每题都请严格按 `CLAUDE.md` 的 "How to Add a Feature" 六步走：定义通道 → handler → preload 暴露 → types.d.ts 声明 → 组件 → feature_list。

### L3-1 接通真导入按钮（问题③）

- **目标**：点 `+ Import`（或启用闲置的 `ImportPanel`）弹出系统文件选择框，选 `.txt/.md` 完成导入。
- **新知识**：主进程用 `dialog.showOpenDialog({ filters: [{ name: 'Documents', extensions: ['txt','md'] }] })`（从 `electron` 导入 `dialog`）。渲染进程**拿不到**绝对路径，所以必须走 IPC。
- **提示**：新通道名建议 `documents:pick-and-import`；main 侧 handler 里 `dialog` 返回 `filePaths[0]` 后直接调 `documentService.importDocument`；界面侧调完记得刷新列表。
- **验收**：不碰控制台、纯鼠标完成导入；文件进 `documents/` 与 `content/`；六步每步的产物能对上号。

### L3-2 修登记簿缺口（问题④）

- **目标**：详情页 "Index Document"（单文档索引）也能让问答搜到。
- **文件**：`indexing-service.ts` 单文档路径。
- **提示**：把"写 chunks + 更新 index-meta"抽成一个私有方法，两条路径共用（顺带完成 L2-3 的状态更新）。
- **验收**：只从按钮索引一篇文档后提问，引用能命中该文档；`index-meta.json` 有登记。

### L3-3 删除按钮 + 清理孤儿（问题⑤）

- **目标**：详情页加 Delete 按钮，删除文档时**全部**关联数据清理干净。
- **提示**：`PersistenceService` 需补一个 `deleteFile(relativePath)`；要清理：元数据条目、`documents/<filename>`、`content/<id>.txt`、`chunks/<id>.json`、`index-meta.json` 的键。删除后清空选中态（`setSelectedDoc(null)`）并刷新。
- **验收**：删除后数据目录里搜不到该 id 的任何痕迹；问答检索不再返回它。

## 12.4 L4 工程化（每题 1 小时左右）

### L4-1 第一个单元测试

- **目标**：用 vitest 给切块逻辑写测试并跑绿。
- **提示**：新建 `src/services/indexing-service.spec.ts`（vitest 默认找 `*.spec.ts`/`*.test.ts`）。`chunkDocument` 是 private——两条路：经 `startIndexing('某id')` 触发后用 `getChunksForDocument` 读结果；或把它改成 public（顺便讨论哪种更好）。构造被测对象：`new IndexingService(new PersistenceService(临时目录))`——依赖注入在这里兑现价值。
- **用例建议**：空字符串→0 块；三个短段落的经典案例（对照第 7 章手工模拟表）；超长单段落→1 块且不切断。
- **验收**：`npm test` 绿；提交。

### L4-2 仓库卫生

- **目标**：补 `.gitignore`（`node_modules/`、`dist/`、`electron-*.log`），让 `git status` 只剩真实源码变更。
- **验收**：`git status` 干净可读；`dist/` 未跟踪状态消失。

### L4-3 问答历史面板

- **目标**：界面某处展示 `qa:history`（通道已通、数据已存，只差 UI）。
- **提示**：`window.knowledgeBase.qa.history()` 已可用；放在右栏问答卡片下方，或做成可折叠区。
- **验收**：重启应用后历史仍在（它一直在 `qa-history.json` 里，现在终于看得见）。

## 12.5 L5 进阶（每题半天+，做完你就有真产品雏形）

### L5-1 接入真 LLM

- **目标**：把 `QaService.generateAnswer` 的模板替换为真 LLM 调用，构成真·本地 RAG。
- **路线**：Node 侧直接 `fetch` 一个 OpenAI 兼容接口（主进程有完整网络能力），prompt = 问题 + top-2 chunk 原文，要求"仅依据资料回答并标注出处"；把 `MOCK_PATTERNS` 留作离线回退。密钥放环境变量 `process.env`，绝不硬编码。
- **验收**：断网时回退到模板；联网时答案随导入的文档内容变化。

### L5-2 关键词检索 → 向量检索

- **目标**：用 embedding 相似度替换 includes 计分。
- **提示**：最省事——继续用 API 算 embedding，chunk 向量存进 `chunks/<id>.json` 的新字段，问答时余弦相似度排序（自己写也就十行）。中文效果提升立竿见影。
- **验收**：同一问题下，引用命中质量明显优于关键词版。

### L5-3 给你的 LangGraph 项目建 harness

- **目标**：回到第 10 章 10.6 的三件套清单，给自己的项目写 `AGENTS.md` + `feature_list.json` + `init` 脚本。
- **验收**：让一个 AI 助手只读这三样文件就正确跑起你的项目并完成一个指定小改动——harness 的最终考试。

## 12.6 后续学习路线

按"对这个项目最有用"排序，官方文档优先：

| 顺序 | 学什么 | 资源 | 与本项目的衔接 |
|---|---|---|---|
| 1 | JavaScript 本体（闭包/原型/数组方法） | MDN《JavaScript 指南》 | 第 2 章每个语法点的展开 |
| 2 | React 官方教程 | react.dev（新版文档，中文可读） | 重写本项目界面状态管理（如用 Context） |
| 3 | TypeScript 深入 |《TypeScript Handbook》(typescriptlang.org) | 泛型/类型收窄/模块解析 |
| 4 | Node.js API 面 | nodejs.org/docs | 把同步 IO 改异步流式 |
| 5 | Electron 安全与进程模型 | electronjs.org/docs（Security 章必读） | 第 6 章三把锁的完整版 |
| 6 | 课程后续 P02~P08 | 本仓库 docs/ | 这个应用的下一站（导入/持久化/可观测性/完整 harness） |

两个建议：学 JS/TS 期间**不换项目**，就在这个 1121 行的仓库里改——上下文熟悉，反馈快；每完成一个练习按 12.0 提交，三个月后 `git log` 就是你自己的进度曲线。

## 12.7 结语

你已经读完了人生第一个完整项目：四层架构、四个服务、五个 React 组件、一条 IPC 桥、两套构建线、一套 harness——合计 1121 行源码 + 13 个章节。更重要的是沿途攒下的可迁移资产：类型即文档的读法、UI = f(state) 的思想、进程边界的代价、"编译通过≠运行正确"的警觉、先定位层再排错的方法、以及"harness 让能力变可靠"的世界观。

第 12 章练习做完之前，这本书不算读完。去吧。

## 本章小结

- 12 道练习从改颜色一路铺到接真 LLM，每道都有 DoD 式验收标准；做题纪律 = check + dev + feature_list + commit 四连。
- 修复类练习（L2/L3）全部对应第 8 章问题清单，等于把项目的已知债还清的同时完成学习。
- 学习路线官方文档优先、不换项目、以 git log 记录成长。
