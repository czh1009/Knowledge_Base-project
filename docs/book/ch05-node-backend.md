# 第五章 Node.js 与后端思想

> 本章目标：搞懂"后端"在本项目里的真实形态——一个叫 Node.js 的 JS 运行时。理解它和 CPython 的异同、模块系统的两副面孔、文件系统 API，以及"分层 + 服务 + 依赖注入"这些你将在任何后端框架里反复见到的思想。

## 5.1 Node.js 是什么：JS 界的 CPython

Python 代码需要一个解释器才能运行（CPython）；JavaScript 代码需要一个 JS 引擎 + 一套宿主 API 才能运行：

- 在浏览器里：V8 引擎 + 浏览器 API（DOM、fetch……）→ 你只能操作网页。
- 在 Node.js 里：V8 引擎 + **Node API**（文件系统、进程、网络、操作系统……）→ 你能写真正的程序：服务器、命令行工具、桌面应用主进程。

【类比】`node main.js` 之于 JavaScript，恰如 `python main.py` 之于 Python。npm 之于 Node，恰如 pip 之于 Python（5.6 节详说）。

本项目里 Node 出现在两处：
1. **主进程**（`src/main/` + `src/services/`）：真正的 Node 世界，能读写文件。
2. **工具链**（`scripts/dev.js`、Vite、TypeScript 编译器本身）：`npm run dev` 干活的都是 Node 程序。

 Electron 本质上 = "把 Node 主进程和多个 Chromium 渲染进程粘在一起"的框架（第 6 章）。

## 5.2 事件循环：和 asyncio 同一个思想

Node 是**单线程 + 事件循环（event loop）**：同一时刻只执行一段 JS，遇到耗时操作就"登记回调、继续干别的"，完成后回调排队执行。这正是 Python `asyncio` 的模型，区别是两口气：

| | Python asyncio | Node.js |
|---|---|---|
| 循环启动 | 你显式 `asyncio.run(main())` | 进程启动就自动跑，永不停止 |
| 阻塞代价 | 忘 await 只是没结果 | 一段同步代码卡住，**整个进程**冻结 |
| 普通函数能否挂起 | 不能（需 async def） | 不能（需 async/await） |

"整个进程冻结"对桌面应用意味着**窗口白屏无响应**。看本项目怎么处理这个矛盾：

```ts
// src/services/persistence-service.ts:26 —— 同步读文件（会阻塞，但本地小文件毫秒级完成）
return JSON.parse(fs.readFileSync(fullPath, 'utf-8'));

// src/services/qa-service.ts:52 —— 异步等待（不阻塞，主进程照常处理其他 IPC）
await new Promise(resolve => setTimeout(resolve, 100 + Math.random() * 400));
```

服务层大量用同步 I/O 是**刻意的偷懒**：本地 JSON 文件小到阻塞可忽略，同步代码更好写好读。而跨进程的 IPC 一律异步。工程没有教条，只有取舍——但你要知道取舍的存在。

`setTimeout(fn, ms)` 是 JS 世界最古老的"延时回调"API；包一层 `new Promise` 是为了能用 `await`（第 2 章 2.9 提过）。另外本项目 `main.ts:52` 的 `app.whenReady().then(...)` 也是同一个思想：Electron 初始化是异步的，"就绪"后回调我们的启动逻辑。

## 5.3 模块系统：源码和产物的两副面孔

你会在这个项目里同时见到两种模块语法，别慌，这是历史问题：

**源码用 ESM**（ECMAScript Module，现代标准，浏览器/打包器原生支持）：

```ts
// src/main/main.ts:1-3
import { app, BrowserWindow, ipcMain } from 'electron';
import * as path from 'path';
import { registerIpcHandlers } from './ipc-handlers';
```

**编译产物用 CommonJS**（Node 的传统模块系统，1995-2015 时代的主角）：

```js
// dist/main/main.js（tsc 编译后）
const electron_1 = require("electron");
const path = require("path");
```

谁做的转换？`tsconfig.node.json` 里一行 `"module": "commonjs"`：TypeScript 编译时把 import 翻译成 require。为什么主进程要用旧标准？因为 Node 直接运行 CommonJS 最省事，不需要额外打包步骤（对比：渲染进程交给 Vite 处理，可以用 ESM）。

`scripts/dev.js:2-3` 是手写的 CommonJS，因为它自己就是跑在 Node 里的工具脚本：

```js
const { execSync } = require('child_process');
const path = require('path');
```

体会一下对应关系：CommonJS 的 `const { a, b } = require('x')` 就是 ESM 的 `import { a, b } from 'x'`，语法不同、语义相同。

还有一个小知识点：`package.json` 没有 `"type": "module"` 字段，所以项目里所有 `.js` 文件默认按 CommonJS 解释。这是默认值，也是为什么 dev.js 用 require。

## 5.4 文件系统：`fs` 与 `path` 速成

后端最基础的能力是碰磁盘。本项目用到的全部函数一张表（都在服务层，别处不许用——这是 `AGENTS.md` 的层边界规定）：

| 函数 | 作用 | Python 对应 | 项目位置 |
|---|---|---|---|
| `fs.readFileSync(p, 'utf-8')` | 同步读文本 | `open(p).read()` | persistence:26 |
| `fs.writeFileSync(p, s)` | 同步写文本 | `open(p,'w').write(s)` | persistence:34 |
| `fs.existsSync(p)` | 文件/目录是否存在 | `os.path.exists(p)` | persistence:25 |
| `fs.mkdirSync(p, {recursive:true})` | 建目录（含多级父目录） | `os.makedirs(p, exist_ok=True)` | persistence:17 |
| `fs.copyFileSync(src, dst)` | 复制文件 | `shutil.copy` | persistence:56 |
| `fs.unlinkSync(p)` | 删除文件 | `os.remove(p)` | persistence:64 |
| `fs.readdirSync(dir)` | 列目录文件名 | `os.listdir(dir)` | persistence:72 |
| `fs.statSync(p).size` | 文件元信息（大小等） | `os.stat(p).st_size` | document-service:30 |
| `path.join(a, b)` | 跨平台拼接路径 | 无（手动拼） | 全项目 |
| `path.basename(p)` | 取文件名部分 | `os.path.basename` | document-service:28 |
| `path.dirname(p)` | 取目录部分 | `os.path.dirname` | persistence:32 |

`path.join` 必须用、不许手拼 `dir + '/' + name`：Windows 用 `\`、Linux/macOS 用 `/`，`path.join` 替你处理差异——你在本书路径里看到的 `src/main/main.ts` 是写法惯例，Windows 实际是反斜杠。

两个特殊变量：`__dirname`（当前文件所在目录的绝对路径）和 `__filename`（当前文件绝对路径）。本项目唯一用到的地方值得精读：

```ts
// src/main/main.ts:18 —— 计算 preload 脚本的磁盘位置
preload: path.join(__dirname, '..', 'preload', 'preload.js'),
```

运行时 `__dirname` = `...\solution\dist\main`（编译后 main.ts 住在 dist/main/），往上一级（`..`）再进 `preload`，得到 `dist\preload\preload.js`——正是编译产物里 preload 的家。窗口创建时 Electron 会加载并执行这个文件（第 6 章展开）。

## 5.5 `process` 与运行环境

`process` 对象携带进程的全部环境信息，本项目用到两处：

```ts
// src/main/main.ts:64-66 —— 关窗口后是否退出应用
app.on('window-all-closed', () => {
  if (process.platform !== 'darwin') app.quit();   // 非 macOS 才退出
});
```

`process.platform` 返回 `'win32'` / `'darwin'` / `'linux'`。这条 if 是 macOS 的 UI 惯例：Mac 上关掉全部窗口后应用常驻 Dock，可再点开。Windows/Linux 用户关完窗口就退出。跨平台细节见真章——桌面应用的"后端"离操作系统很近。

## 5.6 npm 生态：package.json 是项目的身份证

【类比】`package.json` ≈ `pyproject.toml` + `Makefile` 的合体；`npm` ≈ pip + venv 管理 + make 的合体。

本项目 `package.json` 的关键字段：

```json
{
  "name": "knowledge-base",        // 项目名——注意！它还决定了数据目录名（见下）
  "main": "dist/main/main.js",     // Electron 的入口文件（主进程从这里启动）
  "scripts": {
    "dev": "node scripts/dev.js",             // npm run dev = 三步流水线（第 9 章）
    "build": "tsc -p tsconfig.node.json && vite build",
    "check": "tsc --noEmit -p tsconfig.node.json && tsc --noEmit -p tsconfig.json",
    "test": "vitest run"
  },
  "dependencies": { "react": "^18.3.1", "react-dom": "^18.3.1", "uuid": "^9.0.0" },
  "devDependencies": { "typescript": "^5.7.0", "electron": "^33.2.0", "vite": "^6.0.0", ... }
}
```

- **dependencies**：应用运行必需（React、uuid）。
- **devDependencies**：只在开发/构建时需要（编译器、打包器、Electron 本身——它不进生产依赖是因为打包发布时另有安排）。
- **scripts**：命令别名。`npm run dev` 就是执行 `node scripts/dev.js`。`npm run X` ≈ `make X`。
- `^18.3.1` 表示"18.x 里不低于 18.3.1 的版本"（语义化版本）。
- `name` 有个隐藏后果：Electron 用它决定用户数据目录——Windows 上是 `C:\Users\<你>\AppData\Roaming\knowledge-base\`，本应用的所有数据都写在这下面的 `knowledge-base-data/`（第 8 章看目录内容）。

配套概念：`npm install` 按声明把包装进本地 `node_modules/`（巨大，绝不手改、不进 git）；`package-lock.json` 锁定每个包的精确版本（pip 的 requirements.txt 锁定版）；`npx X` = "临时找到 X 命令并执行"（本地没有就去下载），dev.js 里的 `npx tsc`、`npx electron .` 就是。

## 5.7 后端思想：分层与服务

"后端"不神秘，它就是一组围绕数据的组织纪律。本项目服务层 4 个类展示了两条普适原则：

**1. 单一职责 + 分层**：`PersistenceService` 只管"怎么读写文件"（唯一碰 `fs` 的类），`DocumentService`/`IndexingService`/`QaService` 只管业务规则（"导入要做什么""怎么切块""怎么检索"），`main.ts` 只管组装与窗口。改存储格式只需动一层；改业务规则不碰磁盘细节。

**2. 依赖注入（DI）**：上层把"工具对象"作为构造参数递给下层：

```ts
// src/main/main.ts:40-44 —— 组装现场
const persistence = new PersistenceService(dataDir);          // 我需要数据目录
const documentService = new DocumentService(persistence);     // 我需要持久层
const indexingService = new IndexingService(persistence);
const qaService = new QaService(persistence, indexingService); // 甚至可选注入同僚
```

对比"全局单例"（`persistence = Persistence()` 全项目随便 import）：DI 的类**不绑定任何全局环境**，换一个目录就能 new 一个新的、测试时可以塞一个假实现——这正是服务层可以独立单测的原因（第 12 章练习会真写一个测试）。

【类比 Python】你在 FastAPI 项目里见过的 `Depends()`、在 LangGraph 里把 checkpointer 传给 `MemorySaver()` 的写法，都是同一件事：依赖从外面递进来，而不是在类内部自己创建。

## 本章小结

- Node.js = V8 + 系统能力 API；本项目主进程和工具链都跑在 Node 上。
- 事件循环与 asyncio 同构；本项目服务层"同步小文件 IO + 异步 IPC"是个有意识的取舍。
- 源码写 ESM、产物被 tsc 编译成 CommonJS（`"module": "commonjs"`），两种语法都要认识。
- `fs`/`path` 函数速查表覆盖了本项目全部磁盘操作；`__dirname` 推算 preload 路径是精读案例。
- `package.json` 是身份证 + 命令别名 + 依赖清单三合一；`name` 字段还决定了用户数据目录。
- 后端思想就两条：分层（每层只管一件事）+ 依赖注入（依赖从构造函数递进来）。
- 下一章：把 renderer、Node、进程这些零件装配成 Electron——本书架构篇的高潮。
