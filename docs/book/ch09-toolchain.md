# 第九章 工程工具链：从源码到运行中的应用

> 本章目标：看懂 `npm run dev` 背后的整台机器——TypeScript 编译器（tsc）处理什么、Vite 打包什么、`dev.js` 怎么串起三步、为什么有两份 tsconfig。最后补上 git 的最小必需知识，并用它解读这个仓库的当前状态。

## 9.1 总图：谁把什么变成什么

这个项目有**两堆源码、两条构建线**，因为它们运行在两个不同的世界：

```
【Node 世界】主进程 + preload + 服务层 + 共享类型          【浏览器世界】渲染层
  src/main/*.ts  src/preload/*.ts                            src/renderer/*.tsx
  src/services/*.ts  src/shared/types.ts                     （也用 src/shared 与 services 的类型）

        │ tsc -p tsconfig.node.json                              │ vite build
        │ （TypeScript 编译器，传统模式）                          │ （打包器，现代模式）
        ▼                                                        ▼
  dist/main/main.js        CommonJS，Node 直接运行          dist/renderer/
  dist/preload/preload.js                                  ├── index.html
  dist/services/*.js                                       └── assets/index-*.js
  dist/shared/types.js                                           （打包成1个JS + JSX已转译）

                    ┌──────────────────────────────┐
                    │  npx electron .               │
                    │  读 package.json 的 main 字段  │
                    │  → 运行 dist/main/main.js      │
                    │  → 创建窗口加载 renderer 页面   │
                    └──────────────────────────────┘
```

为什么要两条线？**Node 直接吃 CommonJS 的 `.js`**，tsc 老老实实翻译即可（保留目录结构，运行时 `require` 相对路径解析）。而**浏览器不认识 TSX**（JSX 语法），需要 Vite 把整个 React 应用（含 import 的模块图）打成浏览器能执行的少量文件——所谓"打包（bundle）"。

## 9.2 package.json 的 scripts：命令别名表

`npm run X` 就是执行 scripts 里 X 对应的命令串（`&&` 串联，前一步失败即停）：

| 命令 | 实际执行 | 用途 |
|---|---|---|
| `npm run dev` | `node scripts/dev.js` | 三步流水线（9.4） |
| `npm run build` | `tsc -p tsconfig.node.json && vite build` | 完整构建两条线 |
| `npm run check` | `tsc --noEmit -p tsconfig.node.json && tsc --noEmit -p tsconfig.json` | **只类型检查不产出**（`--noEmit`），两条线都查 |
| `npm test` | `vitest run` | 跑测试（当前无测试文件，见 9.6） |

`npm run check` 是日常最该敲的命令——它快（秒级）且能在写坏类型时立刻报警。`AGENTS.md` 的 Definition of Done 第一条"TypeScript compiles without errors"说的就是它。

`npm run dev` 与 `build` 的区别：dev 多一步"启动 Electron"，且未来可以在构建和启动之间插入开发服务器（现在没有）。另外可加 `npm start`（= `electron .`）单独启动已构建的应用——本项目没定义，敲 `npx electron .` 等价。

## 9.3 双 tsconfig：一份给 Node，一份给浏览器

| | `tsconfig.node.json` | `tsconfig.json` |
|---|---|---|
| 服务对象 | main / preload / services / shared | renderer / shared / services |
| `module` | `"commonjs"`（产物给 Node 运行） | `"ESNext"`（交给 Vite 进一步处理） |
| 产物 | `dist/`（`outDir`），带 `.d.ts` 和 sourcemap | `noEmit: true` **只检查不产出** |
| `jsx` | 无（没有 JSX） | `"react-jsx"`（启用 JSX 语法支持） |
| 额外严格项 | — | `noUnusedLocals` 等全开 |

为什么 services/shared 被**两边都包含**？因为它们既要被 Node 侧编译运行，又要被 renderer 侧 import 类型——同一份源码进入两个编译单元。这也正是 6.6 说过"preload 与 types.d.ts 互不可见"的根源：**编译单元是隔离的类型世界**，跨单元的一致性只能靠人（或额外工具）维护。

顺带解释两个配置：`paths` 里的 `@shared/*` 别名本意是缩短 import 路径，但本项目源码实际全用相对路径（`../shared/types`），别名是"备而未用"；`vite.config.ts` 里也注册了同样的 alias 以保持一致。

## 9.4 精读 scripts/dev.js：29 行的流水线

```js
const { execSync } = require('child_process');   // Node 子进程模块（CommonJS，第 5 章）
const path = require('path');
const root = path.resolve(__dirname, '..');       // 项目根（dev.js 在 scripts/ 里，.. 上跳）

console.log('[dev] Building main + preload...');
try {
  execSync('npx tsc -p tsconfig.node.json', { cwd: root, stdio: 'inherit' });
} catch {                                        // execSync 失败时抛异常，捕获后：
  console.error('[dev] TypeScript compilation failed');
  process.exit(1);                               // 立刻终止，不往下走
}

console.log('[dev] Building renderer...');
try {
  execSync('npx vite build', { cwd: root, stdio: 'inherit' });
} catch { process.exit(1); }

console.log('[dev] Starting Electron...');
try {
  execSync('npx electron .', { cwd: root, stdio: 'inherit' });
} catch {
  // Electron 正常关闭也返回非零？——注释说并非如此，这里留空即"吞掉非正常退出码"
}
```

三个知识点：

- `execSync(命令, {stdio: 'inherit'})`：同步执行子命令，`inherit` 让子进程输出**直接接到当前终端**——所以 `npm run dev` 能实时看到 tsc/Vite/Electron 的一切输出。等价于在 shell 里手敲三条命令，只是多了失败即停。
- 三步顺序 = 依赖顺序：主进程代码 → 界面 → 启动。第 8 章启动场景的前置。
- 想给主进程排错时，`npm run dev 2>electron-stderr.log`（把错误流重定向进文件）就是仓库根那两个 `electron-*.log` 的来历——**它们是排查 6.7 事故的关键证物**。

## 9.5 Vite：渲染层的打包器

`vite.config.ts` 只有 14 行，四个配置各有理由：

```ts
export default defineConfig({
  plugins: [react()],              // 教 Vite 认识 JSX/TSX
  root: 'src/renderer',            // 前端项目的根（index.html 在这）
  base: './',                      // 产物里的资源引用用相对路径——因为 Electron 用 file:// 协议加载本地文件，绝对路径 /assets/... 会指向磁盘根！
  build: { outDir: '../../dist/renderer', emptyOutDir: true },
  resolve: { alias: { '@shared': ..., '@services': ... } },
});
```

`base: './'` 是 Electron 项目的高频坑点，值得记住：浏览器默认假设网站部署在域名根，而桌面应用加载的是 `file:///D:/.../dist/renderer/index.html`，只有相对路径才能正确找到旁边的 JS 文件。

构建产物 `dist/renderer/assets/index-*.js`：**整个 React 应用（React 本体 + 你的组件 + 共享类型）打成一个（或少数几个）带内容哈希的 JS 文件**——hash 文件名用于缓存失效，`index.html` 里的引用被 Vite 自动改写。

## 9.6 Vitest：测试的空位

`package.json` 有 vitest 依赖和 `test` 脚本，但 `src/` 下没有任何 `*.test.ts` 文件。`npm test` 当前的真实行为是报 "No test files found" 并以失败退出——**声明的验证机制与实际不符**，在 harness 的话语体系里这是个"验证子系统缺口"（第 10 章会再提）。

好在服务层是纯逻辑 + 依赖注入（第 5、7 章），是**最容易测的代码**。第一个测试怎么写、跑起来什么样，见练习 L4-1（预告：`chunkDocument` 是私有方法，测试要么经由 `startIndexing` 触发，要么把它改成公有——这本身就是一道设计讨论题）。

## 9.7 git 最小必需课（结合本仓库实况）

git 记录的是**快照 + 变更历史**。日常五命令：

| 命令 | 作用 | Python 世界类比 |
|---|---|---|
| `git status` | 看哪些文件改了/没跟踪 | — |
| `git diff` | 看具体改了什么行 | 对比文件 |
| `git add <文件>` | 把修改放上"待提交台" | — |
| `git commit -m "说明"` | 定格一个带说明的快照 | 保存版本 |
| `git log --oneline` | 看历史 | 版本列表 |

本仓库当前实况解读（`git status` 的输出）：

```
On branch feature/my-change
Changes not staged for commit:
  modified:   scripts/dev.js          ← 三处修改都是格式性调整
  modified:   src/renderer/App.tsx      （Window 类型声明移到了 types.d.ts、
  modified:   src/services/qa-service.ts   缩进变化、多一个空行）
Untracked files:
  dist/  node_modules/  electron-stderr.log  ...   ← 构建产物和依赖，本不该进版本库
```

三个教训级观察：

1. **`node_modules/` 和 `dist/` 不该被 git 跟踪**——它们可由 `npm install`/`npm run build` 完全再生，体积巨大。需要 `.gitignore` 文件（练习 L4-2）：
   ```
   node_modules/
   dist/
   electron-*.log
   ```
2. **当前在 `feature/my-change` 分支**（main 之外的工作分支，git 的惯例用法：在分支上实验，满意再合并回 main）。
3. 有未提交修改时，"仓库的状态"与"仓库的记录"是两回事——第 10 章 harness 体系强调"仓库是唯一事实来源"，指的就是**已提交**的内容。

## 本章小结

- 两条构建线：tsc（Node 世界，产出 CommonJS）与 Vite（浏览器世界，打包 React 应用）；`main` 字段决定 Electron 入口。
- `npm run check`（只查类型）是最常用的快速体检；`npm run dev` = dev.js 的"编译→打包→启动"三步，`stdio: 'inherit'` 让输出直通终端，`2>file.log` 是拿证据的手段。
- 双 tsconfig = 双编译单元 = 双类型世界，跨世界的一致性（types.d.ts 镜像）编译器不保证——第 8 章问题②的温床。
- Vitest 已配置但零测试，验证体系有个窟窿；git 五命令 + `.gitignore` 是本仓库现状给出的直接教训。
- 下一章：回到这个项目的灵魂——harness。为什么仓库里一半文件是"给智能体看的"？
