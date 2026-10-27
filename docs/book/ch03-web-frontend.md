# 第三章 前端第一课：浏览器里的世界

> 本章目标：理解"网页"由什么构成——HTML（结构）、CSS（外观）、DOM（浏览器眼中的树）、事件（交互的起点）。本项目没有单独的 `.css` 文件、几乎没写原生 HTML，但渲染层的一切都建立在这四个概念上。学完你就能看懂 `index.html` 的每一行和 React 里 `style={{...}}` 的含义。

## 3.1 前端到底指什么

"前端"= 运行在**浏览器**里的那部分程序：负责把数据画成界面、响应用户操作。"后端"= 提供数据与计算的那部分。你写 Python 时如果用过 tkinter 做界面，已经有正确的直觉：

| 概念        | tkinter                  | 浏览器          |
| --------- | ------------------------ | ------------ |
| 结构（有哪些控件） | `Label(text=...)` 等代码搭建  | HTML 标签文本描述  |
| 外观（长什么样）  | `bg=`, `fg=`, `font=` 参数 | CSS 规则       |
| 行为（点了怎么办） | `command=` 回调函数          | 事件监听器（JS 函数） |

区别在于：浏览器读的是**声明式的文本文件**（HTML/CSS），而不是构造控件的代码。

Electron 应用的"渲染进程"（`src/renderer/`）就是一个装在窗口里的 Chrome 浏览器页面——所以它遵循浏览器的一切规则。这就是为什么这个项目要先讲浏览器。

## 3.2 HTML：用标签描述结构

HTML 是一种嵌套的标签（tag）语言。看本项目唯一的页面 `src/renderer/index.html`，全文只有 24 行：

```html
<!DOCTYPE html>                                <!-- 声明"我是 HTML5 文档" -->
<html lang="en">                               <!-- 根元素 -->
<head>                                         <!-- 头部：给浏览器看的元信息，不显示 -->
  <meta charset="UTF-8" />                     <!-- 字符编码（没有它中文会乱码） -->
  <meta name="viewport" content="width=device-width, initial-scale=1.0" />
  <title>Knowledge Base</title>                <!-- 窗口/标签页标题 -->
  <style>                                      <!-- 内联 CSS，见 3.4 -->
    *, *::before, *::after { box-sizing: border-box; margin: 0; padding: 0; }
    body {
      font-family: -apple-system, BlinkMacSystemFont, 'Segoe UI', Roboto, sans-serif;
      background: #1a1a2e;
      color: #e0e0e0;
      height: 100vh;
      overflow: hidden;
    }
    #root { height: 100vh; display: flex; flex-direction: column; }
  </style>
</head>
<body>                                         <!-- 身体：用户看到的内容 -->
  <div id="root"></div>                        <!-- 一个空的容器，id 为 root -->
  <script type="module" src="./main.tsx"></script>  <!-- 加载 React 入口 -->
</body>
</html>
```

三个要点：

1. **标签成对出现** `<div>...</div>`，也可以自闭合 `<meta ... />`。标签可以层层嵌套，形成一棵树。
2. `id="root"` 是全局唯一的名字。整个 React 应用建成后会被"塞进"这个空 div——`src/renderer/main.tsx:5` 的 `document.getElementById('root')` 找的就是它。
3. `<script src="./main.tsx">` 引用了 TypeScript 文件——浏览器本不认识 TSX，这是 Vite 构建时替换/打包的（开发构建后它指向打包产物）。`type="module"` 表示按现代 ES 模块方式加载。

【类比 C】HTML 像 `.ini`/`.xml` 配置文件：本身不含逻辑，只是被宿主（浏览器）解析后生成内部结构。

## 3.3 DOM：浏览器眼中的那棵树

浏览器读入 HTML 后，会在内存里建一棵**DOM 树**（Document Object Model）：每个标签变成一个节点（node）对象，嵌套关系变成父子关系。上面的 HTML 大致生成：

```
document
 └── html
      ├── head ── meta, meta, title, style
      └── body
           ├── div#root          ← React 把整个应用挂在这
           └── script
```

JS 可以随时增删改这棵树，页面随之变化——这套 API 叫 DOM API。本项目只用了一次最原始的 DOM 调用（`getElementById`），因为 React 替我们操作 DOM（第 4 章的主题）。但你必须知道：**React 做的一切，本质都是在帮你生成 DOM 更新**。界面 = 树；改界面 = 改树。

在 DevTools（第 11 章）的 Elements 面板里，你能亲眼看到这棵树——运行中的应用，树里 `div#root` 下面已经不再是空的，而是满满当当的 React 生成的内容。

## 3.4 CSS：外观规则

CSS 的基本形态是"**选择器 + 一组属性**"：

```css
选择器 { 属性: 值; 属性: 值; }
```

`index.html` 里的 `#root { height: 100vh; display: flex; flex-direction: column; }` 读作：选中 id 为 root 的元素，高度占满视口（100vh = 视口高度的 100%），使用 flex 布局并纵向排列子元素。

**选择器**常见三种：`div`（按标签名）、`#root`（按 id）、`.title`（按 class）。本项目没用 class 选择器——因为整个应用的样式几乎都写在组件里（内联样式，马上讲）。

本项目用到的 CSS 属性速查（读代码时回来查表即可）：

| 属性                           | 作用           | 项目取值示例                  |
| ---------------------------- | ------------ | ----------------------- |
| `display: flex`              | 弹性布局（一维排布神器） | 几乎所有容器                  |
| `flex-direction: row/column` | 子元素横排/竖排     | `column` 为主             |
| `flex: 1`                    | 占满剩余空间       | 左栏定宽 280px，右栏 `flex: 1` |
| `padding` / `margin`         | 内边距 / 外边距    | `'12px 20px'`（上下 左右）    |
| `background`                 | 背景色          | `'#16213e'`             |
| `color`                      | 文字颜色         | `'#e0e0e0'`             |
| `border` / `borderBottom`    | 边框           | `'1px solid #0f3460'`   |
| `borderRadius`               | 圆角           | `'4px'`                 |
| `fontSize` / `fontWeight`    | 字号 / 字重      | `'13px'` / `600`        |
| `cursor: pointer`            | 鼠标悬停变手形      | 可点击的行                   |
| `overflow: auto`             | 内容溢出时出滚动条    | 列表容器                    |

**暗色主题色板**：本项目颜色是成体系的，认识这 5 个色号你就读懂了全部"设计"：

| 色号                 | 角色                          |
| ------------------ | --------------------------- |
| `#1a1a2e`          | 页面最底层背景（近黑的深蓝）              |
| `#16213e`          | 面板背景（头部、侧栏、输入区）             |
| `#0f3460`          | 边框线 / 选中态                   |
| `#533483`          | 强调色（Import/Ask/Index 按钮，紫色） |
| `#e0e0e0` / `#888` | 主文字 / 次要文字                  |

### 内联样式：React 项目的主流写法

按传统，CSS 应写在 `.css` 文件里。但 React 允许把样式作为 **JS 对象**直接挂在元素上，本项目全部采用这种方式（`src/renderer/App.tsx:51-58`）：

```tsx
<header style={{
  padding: '12px 20px',
  background: '#16213e',
  borderBottom: '1px solid #0f3460',
}}>
```

注意与 CSS 文本的两点差异：属性名用**小驼峰**（`borderBottom` 而非 `border-bottom`，因为它是 JS 对象的键，不能有连字符），值几乎都要**引号**（`'12px'`）。`style={{...}}` 的双花括号 = JSX 的"进入 JS 世界"外层 `{}` + 对象字面量内层 `{}`（第 4 章展开）。

TSX 文件 = TypeScript 语法 + JSX 标签语法；React 库 = 让这些标签真正跑起来的引擎。

## 3.5 事件：交互的起点

用户每次点击、输入、提交，浏览器都会产生一个**事件（event）对象**，沿 DOM 树传播。JS 通过"给元素挂监听函数"来响应。常见事件：`click`、`change`（输入框内容变化）、`submit`（表单提交）、`keydown`。

原生 JS 写法是 `el.addEventListener('click', fn)`；React 里简化成标签属性 `onClick={fn}`。你会反复见到这三处：

```tsx
<button onClick={refreshDocuments}>Refresh</button>          {/* App.tsx:61 */}
<input onChange={e => setQuestion(e.target.value)} />        {/* QuestionPanel.tsx:30 */}
<form onSubmit={handleSubmit}> ... </form>                   {/* QuestionPanel.tsx:18 */}
```

特别说明 `QuestionPanel.tsx:11` 的 `e.preventDefault()`：HTML 表单的**默认行为**是提交后刷新整个页面（浏览器时代的遗物）。单页应用绝不能整页刷新，所以表单处理函数第一行几乎总是"阻止默认行为"——相当于告诉浏览器："这次提交归我管，你别动。"

【类比 tkinter】`command=on_click` ↔ `onClick={on_click}`；不同之处是 React 事件处理函数会收到一个事件对象 `e`，从它身上取 `e.target.value`（触发事件的输入框的当前值）。

## 3.6 浏览器的"控制台"：你的命令行界面

按 `Ctrl+Shift+I`（Electron 窗口内同样有效）打开 DevTools：

- **Console**：既是日志输出区（`console.log`/`console.error` 都来这里），也是**交互式 REPL**——可以直接输入 JS 表达式执行，比如 `window.knowledgeBase.documents.list()`。本项目"导入文档"的正门就是这扇（第 11 章手把手教）。
- **Elements**：实时查看/编辑 DOM 树和样式。
- **Sources**：断点调试 JS（第 11 章）。

【类比】Console 之于前端，如同 `print` + `python -i` 之于 Python：观察 + 试探，两件事。

## 本章小结

- 网页 = HTML 结构 + CSS 外观 + JS 行为；浏览器把 HTML 解析成 DOM 树，界面的一切都是"改树"。
- 本项目只有一个几乎空的 `index.html`：样式内联在 `<style>` 里打底，React 应用挂载到 `div#root`。
- 样式全部采用 React 内联对象写法：属性小驼峰、值加引号；五个色号构成暗色主题。
- 交互从事件开始：`onClick`/`onChange`/`onSubmit`，表单先 `e.preventDefault()`。
- DevTools 的 Console 是前端的第一调试工具。
- 下一章：React——为什么本项目敢让 HTML 几乎为空？因为界面是用 JS 函数"算"出来的。

