# claude-progress.md -- Session Log

## Project 01: Baseline vs Minimal Harness

### Session 1 -- 2026-03-30

**Duration**: ~45 minutes
**Goal**: Establish baseline Electron app with proper harness

**What was done**:

- Verified Electron window launches at 1200x800 with correct webPreferences
- Confirmed document list panel renders with empty state message
- Confirmed question panel accepts input and submits via IPC
- Verified PersistenceService creates data directories under userData
- Updated feature_list.json with all 4 features at status "pass"
- Wrote AGENTS.md with startup rules and layer boundaries
- Wrote docs/ARCHITECTURE.md describing Electron layer structure
- Wrote docs/PRODUCT.md describing knowledge base requirements

**Decisions**:

- Used constructor injection for PersistenceService to keep services testable
- Kept all IPC channel names in a single const object in types.ts
- Window title set to "Knowledge Base" for consistency

**Issues**: None

**Next session**: Proceed to Project 02 to add import, detail view, and persistence features.

### Session 2 -- 2026-10-25

**Duration**: 跨多天（2026-10-21 至 2026-10-25)
**Goal**: 让项目真正达到 Definition of Done，并修正失真的状态文件

**What was done**:

- 修复两个类型错误：删除 App.tsx 第 8~28 行与 types.d.ts 冲突的过期 `declare global` 块；删除 qa-service.ts 未使用的 `Chunk` import
- 修复断桥问题：main.ts webPreferences 增加 `sandbox: false`(preload 沙箱的 require 白名单不允许加载 ../shared/types，导致 window.knowledgeBase 从未挂载）
- 绕过环境故障：Node v24 与 extract-zip@2.0.1 不兼容导致 Electron 二进制静默不解压（退出码仍为 0)；手动用系统 tar 解压缓存 zip 到 node_modules/electron/dist，并用 node 写 path.txt(cmd 的 echo 会带 CRLF)
- 用真实证据重新验证全部 4 条 feature，更新 feature_list.json 的 evidence 与 testedAt
- 新增 feature `import-button-ui`,status: not-started(Import 按钮目前只是空壳 console.log)

**Decisions**:

- 修桥选择 `sandbox: false` 而非把 IPC_CHANNELS 复制进 preload：同一事实只能有一个来源
- evidence 一律写成"可复现的命令 + 可观察的结果"，禁止凭印象描述

**Issues**:

- Session 1 的 4 条 "pass" 不可信：桥当时是断的，question-panel 的旧 evidence 不可能为真；"Issues: None" 属漏报
- feature_list 旧 evidence 声称子目录为 data/documents/index，实际为 content/documents/index
- 数据真实位置是 %APPDATA%\knowledge-base\knowledge-base-data(dev.js 未做 .userdata 重定向，早期文档描述有误)
- 项目 harness 缺冒烟测试：DoD 只查"窗口可见"，未查桥是否存活；建议后续在 init.sh 或文档中补 typeof window.knowledgeBase 检查

**Next session**: 实现 import-button-ui(main 进程加 dialog:showOpenDialog IPC,App.tsx 接上，走完 check/build/冒烟验证后标 pass)。

### Session 3 -- 2026-10-25

**Duration**: ~1 小时
**Goal**: 把 `import-button-ui` 从 not-started 做到 pass——Import 按钮弹系统文件框并完成导入

**What was done**:

- `src/shared/types.ts`：`IPC_CHANNELS` 新增 `DIALOG_OPEN_FILE: 'dialog:openFile'`（频道名的唯一来源）
- `src/main/ipc-handlers.ts`：注册 `ipcMain.handle`，调用 Electron `dialog.showOpenDialog`（过滤 md/txt/markdown），用户取消或未选中时返回 `null`
- `src/preload/preload.ts`：在桥上暴露 `dialog.openFile()`
- `src/renderer/types.d.ts`：同步声明 `dialog.openFile: () => Promise<string | null>`
- `src/renderer/App.tsx`：`handleImport` 由 `console.log` 空壳改为「取路径 → `documents.import` → `refreshDocuments`」
- 验证链：`npm run check` 退出码 0；`npm run build` 退出码 0；`npm run dev` 后 DevTools Console 输出 `typeof window.knowledgeBase` = `'object'`、`typeof window.knowledgeBase.dialog.openFile` = `'function'`；真实点击 **+ Import** 选择 `retrieval-plan.md`，左侧列表显示 `Documents (2)`，条目为 `提示词 5.7 KB` 与 `retrieval-plan 1.6 KB`，Console 无红色报错
- `dir` 该目录可见 retrieval-plan.md，大小 1.6 KB 与 UI 一致

**Decisions**:

- 弹框逻辑放 main 进程：renderer 运行时没有 Node 环境（`nodeIntegration: false`），够不着 `dialog`
- 取消文件框返回 `null`，renderer 直接 return 且不报错：取消是正常操作，不是失败
- 导入成功后立即 `refreshDocuments`：否则磁盘写成功但界面不变，会被误判成导入失败

**Issues**:

- 沙箱 EPERM（环境问题，非代码缺陷）：首次由 AI 在受限终端里 `npm run dev`，点击 Import 后 `documents:import` 报 `EPERM: operation not permitted, copyfile`，写 `%APPDATA%\knowledge-base\knowledge-base-data\documents` 被系统拒绝；同一实例的启动日志另有 `Unable to move the cache: 拒绝访问。(0x5)`。改由用户在自己的 cmd 中启动后，同样的操作一次成功
- 前三关全绿仍不代表功能可用：`check`、`build`、两个 `typeof` 冒烟都过，真实点击仍可能失败（本次 EPERM 就是只有真实操作才暴露）。DoD 里的"真实操作"不能省
- 观察未深究：某次启动时 UI 显示 `Documents (0)`、列表空，但 `documents` 目录下已存在 `提示词.md`，说明磁盘文件与元数据清单（`documents-meta.json`）不同步；本次未定位原因，留待后续排查

**Next session**: 把 `typeof window.knowledgeBase` 冒烟检查补进 `init.sh` 或 DoD 文档；`feature_list.json` 中 `import-button-ui` 的 status/evidence/testedAt 由学习者补写。
