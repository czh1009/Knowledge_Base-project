# 第十一章 运行、使用与排错实战

> 本章目标：把应用真正跑起来并完整用一遍（导入→索引→问答→看数据落盘）；建立一套"前端问题排错方法论"；附常见报错字典。本章所有命令都针对你的环境（Windows + Git Bash）实测语境。

## 11.1 十分钟跑起来

前置：已安装 Node.js（`node --version` 应 ≥ 18，你当前是 24.x）。

```bash
cd D:/AICoding/learn-harness-engineering-main/learn-harness-engineering-main/projects/project-01/solution

npm install        # 首次需要；已装过（node_modules 存在）会秒过。≈ pip install -r requirements.txt
npm run dev        # 三步流水线：编译主进程 → 打包渲染层 → 启动 Electron
```

正常的话你会看到：

```
[dev] Building main + preload...     ← tsc 输出（无消息即成功）
[dev] Building renderer...
vite v6.x building for production...
✓ built in ../dist/renderer          ← Vite 打包完成
[dev] Starting Electron...
```

然后弹出一个 1200×800 的深色窗口 "Knowledge Base"。**关掉窗口 = 结束应用**（`npm run dev` 随之返回）。

如果类型或编译报错：终端会停在 `[dev] TypeScript compilation failed`——错误信息里有 `文件名:行号:列号 - 错误码 描述`，先修最靠前的那条（后面的错误常是它的连锁反应），再重跑。

## 11.2 真实使用一遍（绕过假按钮的完整流程）

第 8 章已析：`+ Import` 按钮是占位，**DevTools 控制台才是本版本的操作正门**。

**第 1 步：打开控制台**。在应用窗口里按 `Ctrl+Shift+I`（或右键 → 检查）。出现熟悉的 Chrome DevTools。

**第 2 步：导入示例文档**。Console 里输入（一行一条；路径用正斜杠）：

```js
await window.knowledgeBase.documents.import('D:/AICoding/learn-harness-engineering-main/learn-harness-engineering-main/projects/project-01/solution/data/sample-documents/design-notes.md')
await window.knowledgeBase.documents.import('D:/AICoding/learn-harness-engineering-main/learn-harness-engineering-main/projects/project-01/solution/data/sample-documents/meeting-summary.txt')
await window.knowledgeBase.documents.import('D:/AICoding/learn-harness-engineering-main/learn-harness-engineering-main/projects/project-01/solution/data/sample-documents/retrieval-plan.md')
```

每条返回新建文档的 JSON（含 `id`、`status: "imported"`）就是成功。

**第 3 步：全量索引**（切记**不传参数**——理由见第 7 章 7.3 的两条路径）：

```js
await window.knowledgeBase.indexing.start()
// 返回 { status: 'ready', currentIndexed: 3, totalDocuments: 3, ... }
```

**第 4 步**：点界面右上角 **Refresh**——左侧列表出现三篇文档。

**第 5 步：提问**。底部输入框敲：

```
what is the architecture design?
```

回车，稍等（模拟延迟 100~500ms），右侧出现回答卡片，下方引用区列出命中的文档与 chunk 摘录——这就是第 8 章 8.2 那 12 步链路的实体演出。换个关键词试试 `meeting summary`、`retrieval plan`（对应 `MOCK_PATTERNS` 的五组关键词）。

**第 6 步：验收数据落盘**。文件管理器或终端打开：

```
C:\Users\<你>\AppData\Roaming\knowledge-base\knowledge-base-data\
```

对照第 8 章 8.4 的目录树：`documents-meta.json` 里躺着三条记录、`content/` 三个 `.txt`、`chunks/` 三个 `.json`（打开看，chunks 长什么样一目了然）、`index/index-meta.json` 登记簿、`qa-history.json` 你的提问记录。**亲手打开这几个文件看一遍，是本课程性价比最高的五分钟**。

## 11.3 排错方法论：先定位层，再用对工具

这个应用有四个可能出问题的层，每层的观察手段不同。遇到任何异常，第一件事是**判断问题在哪一层**：

| 层 | 怎么判断问题在这层 | 观察工具 |
|---|---|---|
| Renderer（React） | 界面不对/没反应，但 Console 报错 | DevTools Console（console.log/error 都在这）、Elements 面板 |
| Preload（桥） | `window.knowledgeBase` 是 undefined；任何调用报 "cannot read properties of undefined" | DevTools Console 敲 `window.knowledgeBase` 看看；启动时的 stderr 日志 |
| Main / IPC（主进程） | 调用没响应/报 IPC 错误；窗口行为异常 | **启动终端**的输出（stdio: inherit 直通）；`npm run dev 2>electron-stderr.log` 收集 |
| Services / 磁盘 | 业务结果不对（数据丢了/状态不对） | 直接开数据目录看文件——**磁盘不会说谎**（8.6 方法论） |

通用三步：

1. **读原始报错**：找到**第一条** error（后面常是连锁）；提取 `文件:行号`。
2. **最小复现**：把出问题的动作缩到一行（Console 里的一句 `await window.knowledgeBase.xxx()` 往往就是最小复现）。
3. **二分怀疑**：沿链路设检查点。比如怀疑问答：`await window.knowledgeBase.indexing.status()` 看登记簿数量 → 数量对，问题在 QaService；数量是 0，问题在索引链路。

## 11.4 案例复盘：preload 加载失败（浓缩版）

第 6 章 6.7 已完整剖析，这里只给"如果是你，从零开始怎么走"的流程骨架——因为它是本项目唯一一次"完全不可用级"事故，流程本身就是模板：

```
症状：窗口正常打开，但 Console 里 window.knowledgeBase 是 undefined
      点 Refresh → "Failed to refresh documents: TypeError: Cannot read properties of undefined"
   ↓ 为什么 App 没崩？App.tsx 的 try/catch 吞了异常只打日志（所以"看 Console"是铁律）
第 1 刀：bridge 不存在 → 要么 preload 没执行，要么执行失败
   ↓ 启动终端/stderr 里找 → "Unable to load preload script: ...preload.js"
   ↓            → "Error: module not found: ../shared/types"
第 2 刀：报错在 dist/preload/preload.js:4 的 require("../shared/types")
   ↓ 检查 dist/shared/types.js → 存在！不是文件缺失
第 3 刀：文件在却找不到 → 加载器的解析规则和完整 Node 不同
   ↓ 查证：Electron 20+ preload 默认沙箱运行，require 是受限实现
根因：TS 编译器按 Node 规则解析 import（编译通过），沙箱加载器不认（运行失败）
   ↓
修复：sandbox: false（本仓库已采用）／打包 preload／内联常量
教训：跨边界的能力假设（这里：模块解析规则）必须运行时验证——DoD 条款 2 的由来
```

## 11.5 常见报错字典

按"新手遇到概率"排序。**先读"含义"，再看"修法"**——排错能力 = 把报错翻译成人话的能力。

| 报错（样例） | 含义 | 修法 |
|---|---|---|
| `TS2307: Cannot find module 'xxx'` | import 的模块路径解析不到 | 对比路径大小写与拼写；确认相对路径前缀 `./`/`../`；`npm install` 是否装了该依赖 |
| `TS2322: Type 'A' is not assignable to type 'B'` | 类型不匹配（如把 `IndexStatus` 塞给 `AppStatus`） | 看两边的 interface 字段差异；通常要么改数据形状要么改声明（第 8 章问题②就是实战） |
| `Cannot read properties of undefined (reading 'xxx')` | 对 undefined 取属性——JS 最常见运行时错 | 顺着 `.` 链找出谁是 undefined；Console 里逐段打印；代码里用 `?.` 和 `??` 兜底 |
| `window.knowledgeBase is undefined` / `window.knowledgeBase.documents` 报错 | preload 桥没就绪或加载失败 | 11.4 流程；确认 `dist/preload/preload.js` 存在、`main.ts` 的 `sandbox: false` 在 |
| `npm ERR! code EADDRINUSE` 或端口占用 | 开发服务器端口被占 | 关掉旧的 dev 进程/窗口再启动 |
| 白屏、窗口开了什么都没有 | renderer 加载失败 | DevTools Console 看报错；确认 `dist/renderer/` 已构建（跑过 `npm run build`/`dev`）；检查 `vite.config.ts` 的 `base: './'` |
| `Error: File not found: xxx`（import 时） | 传入路径不存在 | 路径用正斜杠 `/` 或双反斜杠 `\\`；Console 里先 `await window.knowledgeBase.documents.list()` 确认 bridge 正常 |
| `No test files found`（npm test） | vitest 找不到任何测试 | 正常现象（当前零测试）；写第一个测试见练习 L4-1 |
| CSP 警告（`Insecure Content-Security-Policy`） | 开发模式的安全提示 | 良性，打包后消失，可忽略；要消除需在 index.html 配 CSP 头 |
| `SyntaxError: Unexpected token '<'` | 把 HTML 当 JS 执行了（路径/加载顺序错） | 检查 `<script>` 的 src 与 Vite 产物路径 |

补充一个 Windows 特供：路径里的 `\` 在 JS 字符串里是转义字符（`\n` 是换行！），所以控制台里传路径**永远用正斜杠**。

## 11.6 调试工具箱

**console.log 战术**（90% 的场景够用）：

```ts
// 渲染层：输出进 DevTools Console
console.log('documents:', docs);        // 打印对象（Console 里可展开）
console.error('Q&A failed:', err);      // 红色，App.tsx 里已埋好
// 主进程：输出进启动终端（dev.js 的 stdio: 'inherit'）
console.log('[main] services ready');
```

技巧：`console.log` 可以一次打多个值（自动加空格）；给日志加前缀（`[main]`/`[qa]`）方便过滤；对象永远打印变量本身而不是模板字符串拼接（`log('docs', docs)` 优于 `log('docs: ' + docs)`——后者会变成 `[object Object]`）。

**断点**：DevTools → Sources → 左侧文件树找到打包后的 JS（`assets/index-*.js`）→ 点行号设断点 → 触发操作。比 log 强在能看调用栈和局部变量、能单步。（本项目的 React 代码经压缩后可读性下降，配合 sourcemap 才舒服——新手期 log 足矣。）

**主进程排错**：主进程的输出在启动终端，不在 DevTools！两个习惯：跑 `npm run dev 2>electron-stderr.log` 留底（仓库里那两个 log 文件就是这么来的）；或在 `main.ts` 里临时加 `console.log` 再启动。

**磁盘对账**：任何"数据不对"的问题，最后都落到 8.4 那个目录开文件看。元数据、登记簿、chunks 三方对得上，业务层就是对的。

## 本章小结

- 跑起来：`npm install` + `npm run dev`；用起来：DevTools 控制台 `documents.import(...)` 三连 → **无参** `indexing.start()` → Refresh → 提问 → 去数据目录对账。
- 排错先定位层（renderer/preload/main/磁盘），每层有对口的观察工具；Console 是渲染层的真理，启动终端是主进程的真理，磁盘是业务的真理。
- preload 事故的排查流程（症状→桥→加载器→规则差异）是可复用的模板。
- 报错字典覆盖新手期最常遇到的十类；Windows 路径用正斜杠。
- 下一章：带着全部理解，动手改造这个项目——从改一个颜色到接入真 LLM 的分级练习。
