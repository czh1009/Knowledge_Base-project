# 第四章 React：界面是状态的函数

> 本章目标：理解 React 的核心思想（声明式 UI），掌握组件、props、state、三个 hook，然后**逐文件精读**渲染层的全部 8 个文件。读完本章，`src/renderer/` 里没有任何一行会让你困惑。

## 4.1 核心思想：从"命令式"到"声明式"

用 tkinter（命令式）写一个计数器，你要亲手管理每个细节：创建 Label → 按钮回调里算新值 → **手动调用 label.config(text=新值)** 更新界面。界面怎么变，是你的代码一步步"命令"出来的。

React（声明式）把界面定义成一个**函数**：

```
UI = f(state)
```

你只负责两件事：定义 `f`（"state 长这样时，界面应该长那样"），以及改变 `state`。**界面如何从旧样子变成新样子，是 React 的事**（它高效地计算出 DOM 的最小改动）。

本项目的真实例子：`App.tsx` 里存着 `documents`（文档数组）。点击 Refresh → `setDocuments(新数组)` → React 自动重新渲染左侧列表。代码里**找不到任何一行"把列表项加到界面上"的操作**。

【类比 LangGraph】这个思想你其实见过：你声明 graph 的节点和边（结构）+ 更新 shared state，LangGraph 负责调度执行。React 同理：声明组件树（结构）+ 更新 state，React 负责同步到 DOM。

## 4.2 组件与 JSX

**组件（component）就是返回界面的函数**。JSX 是它返回的东西的写法——在 JS 里直接写"类 HTML"：

```tsx
// src/renderer/components/StatusBar.tsx:7-15（节选）
export function StatusBar({ status }: Props) {     // 一个普通函数，参数是 props
  const statusColor = { idle: '#888', /* ... */ }[status.indexStatus] ?? '#888';
  return (                                          // 返回 JSX
    <div style={{ padding: '4px 20px', background: '#0f1729' }}>
      <span>Index: {status.indexStatus}</span>     {/* {} 里嵌入 JS 表达式 */}
      <span>Documents: {status.documentsLoaded}</span>
    </div>
  );
}
```

JSX 语法规则五条：

1. 返回**必须有且只有一个最外层标签**（上面是 `<div>`）。
2. `{}` 进入 JS 世界：`{status.indexStatus}` 输出变量值；`{items.map(...)}` 用 JS 生成一批标签。
3. 标签属性小驼峰：`onClick`、`className`（class 是 JS 保留字，故改名）。
4. 样式是对象：`style={{ padding: '4px' }}`（外 `{}` 进 JS，内 `{}` 是对象）。
5. 注释写 `{/* ... */}`；所有标签必须闭合（`<input />`、`<br />`）。

**JSX 不是字符串**，是编译期语法糖（Vite 把它转成函数调用），所以拼写错误在构建时就会暴露。

## 4.3 Props：父传子的只读通道

组件的输入叫 **props**。本项目惯例是每个组件文件顶部用 `interface Props` 声明：

```tsx
// src/renderer/components/DocumentList.tsx:3-7
interface Props {
  documents: Document[];                  // 数据：要显示的文档
  onSelect: (doc: Document) => void;      // 回调：用户点了某行时通知父组件
  selectedId: string | null;              // 数据：当前选中谁（用于高亮）
}
```

使用方（`App.tsx:110-114`）像写属性一样传值：

```tsx
<DocumentList
  documents={documents}
  onSelect={handleSelectDocument}
  selectedId={selectedDoc?.id ?? null}
/>
```

props 是**只读**的：子组件绝不能改 `documents`，要用新数据只能调用 `onSelect` 通知父组件、由父组件改 state 再传下来。数据单向向下流动（"单向数据流"），这是 React 架构的第一铁律——它让"谁改了数据"永远有唯一答案。

【类比 C】props 像函数入参里的 `const` 结构体 + 函数指针回调；【类比 tkinter】`command=` 回调，但方向被严格规范化。

## 4.4 State：组件自己的可变记忆

**state** 是组件私有、会随时间变化、且变化会触发重新渲染的数据。用 `useState` 创建：

```tsx
// src/renderer/App.tsx:10-17 —— App 的全部四个 state
const [documents, setDocuments] = useState<Document[]>([]);        // 文档列表
const [selectedDoc, setSelectedDoc] = useState<Document | null>(null); // 当前选中
const [appStatus, setAppStatus] = useState<AppStatus>({            // 索引状态
  documentsLoaded: 0, indexStatus: 'idle', lastActivity: '',
});
const [lastResponse, setLastResponse] = useState<QAResponse | null>(null); // 最近回答
```

`useState(初始值)` 返回 `[当前值, 设置函数]`（解构赋值拆开）。两条纪律：

1. **只能通过 set 函数改 state**。直接 `documents.push(x)` 不会触发重新渲染——界面纹丝不动，这是新手第一大坑。
2. **改 state 用新对象/新数组替换，而不是原地修改**（不可变更新）：

```tsx
// src/renderer/App.tsx:21-22（正确：set 整个数组）
const docs = await window.knowledgeBase.documents.list();
setDocuments(docs);            // 用全新的数组替换 → React 察觉变化 → 重渲染
```

为什么必须替换而不是原地改？React 用**引用比较**判断变化：同一个数组对象 = "没变"。所以你会在服务层反复见到 `docs.filter(...)`、`{ ...old, ...updates }` 这种"造新的"写法——第 2 章的展开运算符在这里派上用场。

## 4.5 三个 Hook

Hook 是 React 提供的"特殊函数"，名字都以 `use` 开头，**只能在组件函数体顶层调用**（不能放进 if/循环里）。

### useState —— 记忆 + 变更通知

见 4.4。

### useCallback —— 缓存函数引用

```tsx
// src/renderer/App.tsx:19-28
const refreshDocuments = useCallback(async () => {
  try {
    const docs = await window.knowledgeBase.documents.list();
    setDocuments(docs);
    const status = await window.knowledgeBase.indexing.status();
    setAppStatus(status);
  } catch (err) {
    console.error('Failed to refresh documents:', err);
  }
}, []);   // 依赖数组为空：这个函数引用永远不变
```

`useCallback(fn, [])` 返回一个**引用稳定**的函数版本。为什么需要：JS 每次执行函数都会创建新的函数对象，如果 `App` 每次渲染都把"新函数"传给子组件，子组件会误以为 props 变了而跟着重渲染。`useCallback` + 空依赖数组 = "这个函数一辈子只有一个版本"。第二参数（依赖数组）声明"这些值变了才允许换新版本"。

### useEffect —— 渲染之外的副作用

```tsx
// src/renderer/components/DocumentDetail.tsx:9-14
const [chunks, setChunks] = useState<Chunk[]>([]);
const [showChunks, setShowChunks] = useState(false);

useEffect(() => {
  window.knowledgeBase.indexing.chunks(document.id).then(setChunks);
}, [document.id]);   // 依赖：document.id 变化时重新执行
```

`useEffect(fn, deps)` 读作："**渲染完成后**，如果 deps 里的值变了，就执行 fn"。它是组件与外部世界（IPC、定时器、订阅）打交道的出口——拉取数据是典型场景：进入详情页 → 渲染出骨架 → effect 发起 IPC 请求 → 数据回来 setChunks → 带着数据再渲染一次。

两个细节：`.then(setChunks)` 是 `await` 的等价写法（Promise 直接 `.then(回调)`）；开发模式下 React 的 `<StrictMode>`（`main.tsx:9`）会**故意把每个 effect 执行两遍**帮你暴露问题——所以你在 Console 里看到同一次请求发了两次，不是 bug。

## 4.6 条件渲染与列表渲染

**条件渲染**三件套，全部出现在 `App.tsx`：

```tsx
// 1. 三元：二选一（App.tsx:120-126）
{selectedDoc ? <DocumentDetail document={selectedDoc} /> : <div>请选择文档…</div>}

// 2. && 短路：有才显示（App.tsx:127）
{lastResponse && (<div>...回答卡片...</div>)}
// lastResponse 为 null 时整个表达式为 null，React 什么都不画

// 3. 提前 return（DocumentList.tsx:10-19）：空列表时直接返回另一个界面
if (documents.length === 0) {
  return <div>No documents imported yet. ...</div>;
}
```

**列表渲染**用 `map` 把数组变成标签数组：

```tsx
// src/renderer/components/DocumentList.tsx:24-42（节选）
{documents.map(doc => (
  <div key={doc.id} onClick={() => onSelect(doc)}
       style={{ background: selectedId === doc.id ? '#0f3460' : 'transparent' }}>
    <div>{doc.title}</div>
    <div>{doc.status === 'indexed' ? '✓ ' : ''}{(doc.size / 1024).toFixed(1)} KB</div>
  </div>
))}
```

`key={doc.id}` 是 React 的硬要求：列表变化时靠 key 对上"同一行"，进行高效的增删移。用稳定唯一 id（本项目是 uuid），不要用数组下标。

## 4.7 受控组件：输入框的正确打开方式

`QuestionPanel` 是教科书级的**受控组件（controlled component）**——输入框的值由 React state 驱动，而不是 DOM 自己记：

```tsx
// src/renderer/components/QuestionPanel.tsx:8-15
const [question, setQuestion] = useState('');

const handleSubmit = (e: React.FormEvent) => {
  e.preventDefault();               // 别刷新页面（第 3 章）
  if (!question.trim()) return;     // 空白不提交
  onAsk(question.trim());           // 通知父组件
  setQuestion('');                  // 清空输入框
};

// JSX 部分（27-31 行）：
<input
  type="text"
  value={question}                          // 值 = state（单向：界面显示 state）
  onChange={e => setQuestion(e.target.value)} // 输入 → 改 state（反向：用户输入进 state）
  placeholder="Ask a question about your documents..."
/>
```

`value` + `onChange` 成对出现，state 成为输入框内容的**唯一真相**。数据流闭环：用户敲键 → onChange 改 state → 重渲染 → 输入框显示新值。

## 4.8 状态提升：整个应用只有一个"总账"

看 `App.tsx` 的结构你会发现：**所有 state 都住在顶层 App，子组件全是无 state（或只有局部小 state）的纯展示件**。这叫"状态提升（lifting state up）"。

```
                    ┌────────────── App（唯一的总账）──────────────┐
                    │ documents / selectedDoc / appStatus /        │
                    │ lastResponse + 4 个事件处理函数               │
                    └───────┬───────────┬──────────┬───────────────┘
                     props↓↑回调   props↓↑回调  props↓
                 DocumentList  DocumentDetail  QuestionPanel  StatusBar
```

子组件想改总账？只能调用父组件传下来的回调（`onSelect`/`onAsk`）。事件向上、数据向下，两条单行道。这样设计的好处：任何一个 state 变化影响了谁，一眼可查；任何一个组件都可以独立拿去复用。

## 4.9 渲染层逐文件精读

### `src/renderer/main.tsx`（13 行）—— 挂载入口

```tsx
const container = document.getElementById('root');   // 找到 index.html 里的空 div
if (!container) throw new Error('Root element not found');
createRoot(container).render(<StrictMode><App /></StrictMode>);
```

`createRoot(...).render(...)` = "把 `<App />` 组件树画进这个容器"。从此 React 接管 `#root` 内的一切。`<StrictMode>` 是开发期的"严格教练"（双执行 effect 等）。

### `src/renderer/App.tsx`（157 行）—— 根组件

结构：**上半部分是 4 个 state + 4 个处理函数（4.4 已讲），下半部分是一大段 JSX 布局**。布局分五块，对着界面读：

| JSX 位置 | 界面区域 |
|---|---|
| 51-74 | 顶部 header：标题 + Refresh 按钮 |
| 76-115 | 左栏（280px 定宽）：`Documents (n)` 计数 + `+ Import` 按钮 + `<DocumentList>` |
| 117-151 | 右栏（flex:1 占满）：`<DocumentDetail>` 或欢迎语 + 问答结果卡片 + `<QuestionPanel>` |
| 154 | 底部 `<StatusBar>` |

两个必须点破的"诚实细节"：

1. **`handleImport` 是假的**（`App.tsx:30-34`）：点击 `+ Import` 只在控制台打一行日志，注释明说真实实现需要文件对话框。所以界面上这个按钮点了没反应——不是你的环境坏了。
2. **`ImportPanel` 组件从未被使用**：`components/ImportPanel.tsx` 写好了一个像模像样的导入面板（含 `<input type="file">`），但没有任何文件 import 它。它是"半成品功能"的实物展品，第 12 章练习会把它接上。

问答结果卡片（127-147 行）值得读一遍：`lastResponse && (...)` 控制显隐；`lastResponse.citations.map(...)` 逐条画引用（文档标题 + chunk 序号 + 摘录前 100 字符）。

### `src/renderer/components/DocumentList.tsx`（45 行）

两个分支：空列表 → 友好空状态提示（10-19 行）；非空 → `map` 出可点击的行，选中行换背景色（32 行三元），标题下显示状态符号与 KB 大小。

### `src/renderer/components/DocumentDetail.tsx`（87 行）

进入时用 `useEffect` 拉取该文档的 chunks（4.5 已讲）。元信息区展示文件名/导入时间/大小/状态；按钮区有两个按钮：`Show/Hide Chunks` 切换分块列表显隐（`useState(false)`），`Index Document` 在文档未索引时出现，点击直接调 `window.knowledgeBase.indexing.start(document.id)`——注意它没经过 App 的总账，属于"局部即发即忘"调用，索引后状态栏要等下次刷新才更新（第 8 章会盘点这个行为缝隙）。

### `src/renderer/components/QuestionPanel.tsx`（61 行）

4.7 的受控组件就是它，不重复。注意表单回车和点击 Ask 按钮都会触发 `onSubmit`——浏览器原生行为，白赚的快捷键。

### `src/renderer/components/StatusBar.tsx`（43 行）

一个纯函数展示件：查表得状态颜色，画一个 8×8 圆点 + 两三段文字。它是"把状态可视化"的最小实现。

### `src/renderer/types.d.ts`（27 行）

不是组件，是给 `window.knowledgeBase` 补的类型声明（第 6 章详细讲它的机制和隐患）。

## 4.10 一次点击的完整旅程（预习）

把本章知识串起来，用户点击 Refresh 之后发生的事：

```
1. onClick 触发 refreshDocuments（useCallback 缓存的那个函数）
2. await window.knowledgeBase.documents.list()  → 经 preload → IPC → 主进程 → 服务层 → 磁盘
3. 拿到 Document[]，setDocuments(docs) 改 state
4. App 函数重新执行，JSX 用新 documents 渲染
5. React 计算差异，只更新 DOM 里变化的部分（列表行）
6. 用户看到新列表
```

第 2 步内部穿过四层架构——那是第 6、8 章的故事。

## 本章小结

- React 一个公式：`UI = f(state)`。你改数据（state），React 改界面（DOM）。
- 组件 = 返回 JSX 的函数；props 父传子只读、回调子传父；state 私有可变、必须 set、必须不可变更新。
- 三个 hook：`useState`（记忆）、`useCallback`（稳定函数引用）、`useEffect`（渲染后副作用，拉数据用）。
- 列表渲染 `map + key`；条件渲染 三元/`&&`/提前 return；输入框用 `value+onChange` 受控。
- 本项目把所有全局 state 提升到 App，子组件全是展示件；`handleImport` 是占位假实现、`ImportPanel` 未接线——两个最好的练习素材。
- 下一章：往下走到 Node.js 世界——渲染层脚下那台"JavaScript 服务器"。
