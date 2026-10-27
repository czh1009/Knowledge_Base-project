# 第二章 TypeScript 速成：带着 Python 和 C 的直觉上路

> 本章目标：读完本章，本项目里的任何一行 TypeScript 你都能说出它在做什么。我们不讲 JS/TS 的全部，只讲**这个项目里实际出现**的语法现象——每一个例子都摘自真实源码。

## 2.1 为什么有了 JavaScript 还要 TypeScript

三个你熟悉的参照物：

- **C**：静态类型，编译期检查，类型错误编译不过——但语言表达能力弱（没有字典、没有运行时反射的日常使用）。
- **Python**：表达力强，类型注解（type hints）存在但**解释器不强制**——注解错了程序照样跑，错了才知道。
- **TypeScript**：JavaScript 的表达力 + C 式的**编译期强制类型检查**。类型错误时 `npm run check` 直接失败，代码根本不允许进入构建。

TypeScript 的本质：**它只是 JavaScript + 类型标注**。类型在编译后会被全部"擦除"——编译产物（`dist/` 里的 `.js`）不含任何类型信息。这一点后面会引出一个真实的坑（见 2.9 节）。

## 2.2 变量：`const` 和 `let`，没有 `var`

```ts
// src/main/main.ts:9
let mainWindow: BrowserWindow | null = null;   // let：值会变的变量
// src/renderer/App.tsx:10
const [documents, setDocuments] = useState<Document[]>([]);  // const：绑定不再改变
```

- `const` 声明**不可重新赋值**的绑定（类似 C 的 `const`，但对象内容仍可变）。
- `let` 是普通变量，**块级作用域**（`{}` 内有效，同 C）。
- `var` 是旧时代的遗留，有诡异的函数级作用域，本项目一处都没用，你也别用。
- 冒号后面的 `: BrowserWindow | null` 是类型标注：这个变量要么是一个 `BrowserWindow` 对象，要么是 `null`。`|` 读作"或"，这种类型叫**联合类型（union）**，2.5 节细讲。

【类比 Python】`let x: number = 1` ≈ `x: float = 1`，区别是 TS 的标注**会被编译器强制检查**，注错类型直接编译失败。

## 2.3 基本类型与类型推断

```ts
const CHUNK_SIZE = 500;              // 推断为 number（整数浮点不区分，都是 number）
const filename = 'notes.md';         // 推断为 string
let showChunks = false;              // boolean
```

TS 的推断能力很强，多数时候不用手写类型——编译器从初始值推出来。需要显式标注的场景：函数参数、空数组、跨文件契约。

字符串拼接推荐**模板字符串**（反引号 + `${}`），项目里到处都是：

```ts
// src/services/document-service.ts:25
throw new Error(`File not found: ${filePath}`);
```

【类比 Python】就是 f-string：`f"File not found: {filePath}"`。

## 2.4 对象与数组：JS 的"结构体"和"列表"

JS 的对象（object）字面量长得像 Python 字典 + C 结构体的合体：

```ts
// src/main/main.ts:12-24
mainWindow = new BrowserWindow({
  width: 1200,          // 键: 值
  height: 800,
  minWidth: 800,
  webPreferences: {     // 值本身又是一个对象——嵌套
    contextIsolation: true,
    nodeIntegration: false,
    sandbox: false,
  },
  title: 'Knowledge Base',
});
```

访问成员用点号 `app.getPath('userData')`，等价于 Python 的 `app.get_path("user_data")`。

数组就是 Python 的 list：

```ts
const docs = this.listDocuments();   // Document[] 类型（元素为 Document 的数组）
docs.push(doc);                      // 尾部追加
docs.filter(d => d.id !== id);       // 过滤，返回新数组
docs.find(d => d.id === id);         // 找第一个满足条件的元素
docs.map(c => c.id);                 // 每个元素变换，返回新数组
```

`Document[]` 这种写法读作"Document 的数组"。`===` 是严格相等（不隐式类型转换），**永远用 `===` 而不是 `==`**，这是 JS 社区的铁律。

【类比】`filter/map/find` 对应 Python 的列表推导式：`[d for d in docs if d.id != id]`。JS 里没有推导式，全靠这些数组方法。

## 2.5 `interface`：结构的契约（本项目的心脏）

`src/shared/types.ts` 全文 65 行，定义了贯穿全项目的 7 个接口。这是全书最重要的一个文件，我们逐个看：

```ts
// src/shared/types.ts:3-11
export interface Document {
  id: string;                    // 必须有：字符串类型的 id
  title: string;
  filename: string;
  importedAt: string;            // ISO 格式时间字符串
  size: number;                  // 字节数
  status: 'imported' | 'indexing' | 'indexed' | 'error';  // 字符串字面量联合
  chunks?: number;               // 问号 = 可选字段（可能是 undefined）
}
```

四个新概念：

1. **`interface` 定义的是"形状"**：任何对象只要具备这些字段、类型对得上，就算实现了这个接口。TS 是**结构化类型（structural typing）**——不看你叫什么名字，只看你长什么样。C 的 struct 声明 + Python 的 `Protocol`（鸭子类型的静态版）就是这个思路。
2. **字符串字面量联合** `status: 'imported' | 'indexing' | ...`：status 只允许是这四个字符串之一，写别的编译不过。相当于 C 的 enum，但值直接就是字符串，可读可序列化。后面 `StatusBar.tsx` 还利用它做了"以状态查颜色"的映射（2.11 节）。
3. **可选字段** `chunks?: number`：可以不存在。访问时拿到 `number | undefined`，编译器会逼你先判空。
4. **`export`**：把这个接口公开，其他文件 `import { Document } from '../shared/types'` 才能用。【类比 Python】`from shared.types import Document`。

再看两个有代表性的：

```ts
// src/shared/types.ts:13-19
export interface Chunk {
  id: string;
  documentId: string;                  // 指回所属文档——数据库外键的感觉
  content: string;                     // 正文文本
  index: number;                       // 第几块
  metadata: Record<string, string>;    // 键值都为 string 的字典
}

// src/shared/types.ts:40-44
export interface AppStatus {
  documentsLoaded: number;
  indexStatus: 'idle' | 'indexing' | 'ready' | 'error';
  lastActivity: string;
}
```

`Record<string, string>` 读作"键为 string、值为 string 的字典"，约等于 Python 的 `dict[str, str]`。`Chunk.metadata` 里实际存的是 `{ charCount: '512', wordCount: '88' }`（见 `indexing-service.ts:124-127`）——注意值是字符串，因为 `Record` 的定义如此。

最后是通道名常量：

```ts
// src/shared/types.ts:47-65（节选）
export const IPC_CHANNELS = {
  LIST_DOCUMENTS: 'documents:list',
  IMPORT_DOCUMENT: 'documents:import',
  // ...
  GET_STATUS: 'app:status',
} as const;
```

`as const` 让对象变成**深度只读**且每个值收窄为字面量类型——`IPC_CHANNELS.LIST_DOCUMENTS` 的类型不是 `string`，而是精确的 `'documents:list'`。这样 IPC 通道名在全文只有一个来源、且拼错即编译错误。这是"single source of truth"思想的类型层实现。

## 2.6 函数：箭头函数与回调

JS 的函数写法核心是**箭头函数**（arrow function）：

```ts
(x: number, y: number): number => x + y
```

读法：参数列表 `=>` 返回值表达式。【类比 Python】`lambda x, y: x + y`，但箭头函数可以写多语句 body（用 `{}` 包裹并 `return`），比 lambda 强得多，项目中所有匿名函数都用它。

函数是一等公民，可以当参数传来传去——本项目最典型的就是**回调**：

```ts
// src/renderer/components/QuestionPanel.tsx:3-5
interface Props {
  onAsk: (question: string) => void;   // 一个"吃 string、无返回值"的函数类型
}
```

【类比 C】这就是函数指针：`void (*on_ask)(char *question)`。只是 JS 里它是日常用品而非高级技巧。

## 2.7 class：和 Python 几乎是亲兄弟

服务层四个文件各是一个 class。对照着看：

```ts
// src/services/persistence-service.ts:4-20（节选）
export class PersistenceService {
  private dataDir: string;              // private：类外不可访问（Python 的 __ 前缀 / C 无此概念）
  private documentsDir: string;

  constructor(dataDir: string) {        // 构造函数，名字固定叫 constructor
    this.dataDir = dataDir;             // this 显式写出（Python 的 self），绝不省略
    this.documentsDir = path.join(dataDir, 'documents');
    this.ensureDirectories();
  }

  private ensureDirectories() { ... }   // 私有方法

  readJson<T>(relativePath: string): T | null { ... }   // 泛型方法，见 2.8
}
```

使用方：

```ts
// src/main/main.ts:40-41
const persistence = new PersistenceService(dataDir);   // new 出实例
const documentService = new DocumentService(persistence); // 把实例注入另一个类——依赖注入
```

和 Python class 的差异一览：

| 概念 | Python | TypeScript |
|---|---|---|
| 构造 | `def __init__(self, x)` | `constructor(x)` |
| 自引用 | `self.x`（每个方法必带 self） | `this.x`（方法参数里没有 this） |
| 私有 | 约定 `_x` / `__x` | `private x`（编译器强制） |
| 实例化 | `Foo()` | `new Foo()`（必须 new） |
| 继承 | `class Bar(Foo)` | `class Bar extends Foo`（本项目没用继承，只用组合） |

注意本项目一个有趣的设计：`QaService` 的构造函数第二个参数是可选的（`qa-service.ts:44-47`）——不传就自己 new 一个 `IndexingService`。这叫**带默认值的依赖注入**。

## 2.8 泛型：`readJson<T>` 是怎么工作的

```ts
// src/services/persistence-service.ts:23-27
readJson<T>(relativePath: string): T | null {
  const fullPath = path.join(this.dataDir, relativePath);
  if (!fs.existsSync(fullPath)) return null;
  return JSON.parse(fs.readFileSync(fullPath, 'utf-8'));
}

// 调用方 src/services/document-service.ts:18
const docs = this.persistence.readJson<Document[]>(DOCUMENTS_META);
```

`<T>` 是类型参数，调用时用 `readJson<Document[]>` 指定"这次读出来的 JSON 请当作 `Document[]` 看待"。于是 `docs` 的类型是 `Document[] | null`，编译器后续按这个类型检查你对 `docs` 的一切操作。

【类比】Python 的 `TypeVar`：`def read_json(path: str) -> T | None`，但 TS 版本在**调用点**完成实例化且全程编译期检查。也像 C 的宏/模板，但无运行时开销（类型被擦除）。

**重要的诚实声明**：`JSON.parse` 的返回值其实是 `any`，这里的 `<T>` 只是"信任声明"——如果磁盘上的 JSON 长得不像 `Document[]`，TS 不会在运行时救你。类型系统只活在编译期。这就是为什么 2.9 节的坑是可能的。

## 2.9 异步：Promise、async/await（和 Python asyncio 几乎同构）

JS 是**单线程**语言，任何耗时操作（读大文件、网络请求、定时器）都不能阻塞——否则整个界面卡死。解决方案和 Python 的 asyncio 同构：

```ts
// Promise ≈ Python 的 Future/awaitable
// async 函数 ≈ Python 的 async def
// await ≈ Python 的 await

// src/services/qa-service.ts:50-52
async ask(question: string): Promise<QAResponse> {
  // 人为制造 100~500ms 延迟，模拟"思考"
  await new Promise(resolve => setTimeout(resolve, 100 + Math.random() * 400));
```

`setTimeout(resolve, ms)` 读作"ms 毫秒后调用 resolve 函数"——`resolve` 是 Promise 自带的"宣告完成"回调。这行整体等价于 Python 的 `await asyncio.sleep(random.uniform(0.1, 0.5))`。

调用链上每一层都要传染 `async`：

```ts
// src/main/ipc-handlers.ts:47-49        ← main 进程的 handler 是 async
ipcMain.handle(IPC_CHANNELS.ASK_QUESTION, async (_event, question: string) => {
  return qaService.ask(question);
});
// src/renderer/App.tsx:40-47             ← renderer 的调用方也是 async
const handleAskQuestion = useCallback(async (question: string) => {
  const response = await window.knowledgeBase.qa.ask(question);
  setLastResponse(response);
}, []);
```

错误处理用 `try/catch`（Python 的 `try/except`）。`App.tsx:25-27` 捕获失败并 `console.error` 打印——这在调试时非常关键（第 11 章会用到）。

一个刻意的对比：**主进程的服务层大量使用同步 I/O**（`readFileSync`/`writeFileSync`），因为桌面本地文件很小、同步写法简单直观；而**跨进程边界（IPC）一律是异步的**。工程上"哪里必须异步、哪里可以偷懒"是有取舍的，这个项目选了最简方案。

## 2.10 模块系统：import / export

本项目源码用 ESM 语法（`import`/`export`）：

```ts
import { app, BrowserWindow, ipcMain } from 'electron';   // 从 npm 包导入
import * as path from 'path';                              // 导入整个模块并命名
import { DocumentService } from '../services/document-service';  // 相对路径导入
```

与 Python 的差异：
- 路径**必须写全**且带 `./` 或 `../` 前缀（Python 里同目录直接 import）。
- **命名导出**（named export）：`export class Foo` 导出后，导入方必须 `{ Foo }` 括号精确点名。本项目遵守 `AGENTS.md` 的约定："Use named exports (no default exports)"——default export 是另一种方式，容易导致导入名漂移，团队常禁用。
- 没有包内的隐式命名空间，一切显式。

还有一个特殊文件 `src/renderer/types.d.ts` 里的 `declare global`——声明"全局存在一个东西"，让所有 TS 文件不需要 import 就认识 `window.knowledgeBase`。第 6 章结合它讲。

## 2.11 语法糖速查表（全部来自本项目源码）

按"你可能看不懂"的概率排序：

**解构赋值**（destructuring）——把数组/对象拆开放进变量：

```ts
// src/renderer/App.tsx:10
const [documents, setDocuments] = useState<Document[]>([]);
// useState 返回 [当前值, 设置函数] 两元素数组，一次拆开

// src/main/ipc-handlers.ts:14
const { documentService, indexingService, qaService } = services;
// 等价于：const documentService = services.documentService; （×3）
```

【类比 Python】`documents, set_documents = use_state()` 与 `a, b, c = d["x"], d["y"], d["z"]`。

**展开运算符**（spread）`...`：

```ts
// src/services/indexing-service.ts:30
return { ...status, status: 'error' };
// 复制 status 对象的所有字段，再覆盖 status 字段为 'error' —— 不可变更新风格

// src/services/document-service.ts:72
docs[index] = { ...docs[index], ...updates };
// 旧对象字段 + 更新字段合并成新对象（updates 里的键覆盖前者）
```

【类比 Python】`{**old_dict, **updates}`（Python 3.5+ 的字典合并）。

**可选链** `?.` 与**空值合并** `??`：

```ts
// src/renderer/App.tsx:113
selectedId={selectedDoc?.id ?? null}
// selectedDoc 若为 null/undefined 则整个表达式为 undefined，不报错；再 ?? null 兜底

// src/renderer/components/StatusBar.tsx:13
}[status.indexStatus] ?? '#888'
// 查表查不到（键不存在）就回落到 '#888'
```

等价的 Python 大约是 `selectedDoc.id if selectedDoc else None`。这两个符号在处理"可能为空的链式访问"时无处不在，读代码时见到 `?.` 就在心里补一句"如果前面是空的，这里整体为空"。

**三元表达式**：`条件 ? 真值 : 假值`，即 Python 的 `x if cond else y`：

```ts
// src/renderer/components/DocumentList.tsx:32
background: selectedId === doc.id ? '#0f3460' : 'transparent',
```

**模板里塞 JS**：JSX 中 `{}` 内可以放任何表达式，第 4 章细讲。

## 2.12 类型即文档：一个综合例子

把 `src/renderer/components/StatusBar.tsx` 的核心三行连起来读，你会发现**读懂了类型就读懂了程序**：

```ts
const statusColor = {           // 一个"状态→颜色"的查表对象
  idle: '#888',
  indexing: '#f0ad4e',
  ready: '#5cb85c',
  error: '#d9534f',
}[status.indexStatus] ?? '#888'; // 用传入的 status 查颜色；查不到用灰色
```

`status.indexStatus` 的类型是 `'idle' | 'indexing' | 'ready' | 'error'`（`AppStatus` 里定义的），所以编译器**知道这个查表永远命中四个键之一**——`?? '#888'` 只是运行时的保险丝。这就是联合类型 + 查表替代 if-else 链的惯用法，TS 项目里随处可见。

## 本章小结

- TS = JS + 编译期强制类型；类型会被擦除，运行时不存在类型检查（`readJson<T>` 是信任不是校验）。
- 你需要的全部核心：`const/let`、对象/数组字面量、`interface`（含联合类型、可选字段、`as const`）、箭头函数、class（`this`/`private`/`constructor`）、泛型、`async/await`、命名导入导出。
- 语法糖五件套：解构、展开、`?.`、`??`、三元——认得它们，本项目代码就没有"读不出声"的行。
- 下一章往下走一层：浏览器里的世界——HTML、DOM、CSS 和事件。你将第一次看清 `index.html` 那 24 行和满屏的 `style={{...}}` 是什么。
