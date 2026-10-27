# 读懂你的第一个项目：Project-01 全解

**一本写给"只懂 Python / C / LangChain，第一次面对前后端项目"的人的书。**

讲解对象：[learn-harness-engineering](https://github.com/walkinglabs/learn-harness-engineering) 课程的 Project 01——一个 Electron + TypeScript + React 的桌面知识库应用（导入文档 → 分块索引 → 带引用的问答），同时也是"harness 工程"课程的最小示范仓库。

## 这本书讲什么

不是简化科普，是把 1121 行源码**逐层逐文件讲透**，并把路上需要的前置知识（TS 语法、HTML/DOM/CSS、React、Node、Electron、npm、git）按需补齐——每个知识点都用本项目**真实源码**做例子，每个类比都从你已经会的 Python / C / LangChain 出发。

全书 12 章，分四个部分：

| 部分 | 章 | 内容 | 建议节奏 |
|---|---|---|---|
| **全景** | [ch01](ch01-big-picture.md) | 三层套娃（课程/实验/应用）、仓库地图、四层架构 | 20 分钟 |
| **语言与平台** | [ch02](ch02-typescript.md) TypeScript · [ch03](ch03-web-frontend.md) 浏览器 · [ch04](ch04-react.md) React · [ch05](ch05-node-backend.md) Node 与后端思想 | 读懂每一行所需的一切 | 每章 1~2 小时 |
| **本项目精读** | [ch06](ch06-electron.md) Electron 三层 · [ch07](ch07-services.md) 四个服务 · [ch08](ch08-dataflow-and-known-issues.md) 数据流与问题清单 · [ch09](ch09-toolchain.md) 工具链 | 架构与实现细节 | 每章 1~2 小时 |
| **主题与实战** | [ch10](ch10-harness.md) Harness 工程 · [ch11](ch11-run-and-debug.md) 运行与排错 · [ch12](ch12-exercises.md) 分级练习与路线 | 课程灵魂 + 动手 | 各 1 小时起 |

## 三条阅读路线

- **顺序路线（推荐）**：ch01 → 先照 ch11 §11.1 把应用跑起来玩一遍 → 回来 ch02 按序读到 ch12。遇到代码疑惑就跳到对应精读章节查。
- **赶时间路线**：ch01 → ch11（跑起来+使用+排错）→ ch06（架构）→ 按兴趣跳读。
- **只关心课程主题**：ch01 → ch10 → 读 `AGENTS.md`、`feature_list.json`、`init.sh`、`claude-progress.md` 原文 → ch12 的 L5-3。

## 开始前你需要

- 已装 Node.js（≥18，`node --version` 检查）与 Git——你已具备（Node 24.x）。
- 一个能跑 `npm` 命令的终端（Git Bash 即可）。
- 5 分钟跑起来的命令（详见 ch11）：

```bash
cd <本目录即 solution/>
npm install
npm run dev
```

## 三个先知道（避免误以为是自己坏了）

1. **界面左上 `+ Import` 按钮是占位假实现**（点击只在控制台打日志）。导入文档的正门是 DevTools 控制台，ch11 §11.2 有逐行命令。
2. **仓库曾发生过一次 preload 加载失败**（`sandbox` 沙箱与模块解析的冲突），当前 `src/main/main.ts` 中的 `sandbox: false` 就是修复。完整复盘在 ch06 §6.7——它是全书最有价值的一课。
3. **应用数据不在项目目录里**，而在 `C:\Users\<你>\AppData\Roaming\knowledge-base\knowledge-base-data\`。ch08 §8.4 带你看每个文件。

## 约定

- 代码引用格式 `src/main/main.ts:17` = 文件路径:行号（相对 `solution/` 根）；你若改过代码，行号可能有小偏移。
- 【类比】标记 = 从 Python / C / LangChain 出发的对照。
- 全书辨认的五个主题色、全部 fs 函数表、报错字典等速查表，可直接当工作手册用。
- 已知问题清单（ch08 §8.5，共 7 组）与练习（ch12，共 12 道）一一对应——修 bug 就是最好的练习。

## 从哪开始

现在就打开 [第一章](ch01-big-picture.md)。
