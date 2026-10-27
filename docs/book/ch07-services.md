# 第七章 服务层精读：业务逻辑的心脏

> 本章目标：逐行读通 `src/services/` 下的四个类——持久化、文档、索引、问答。重点两个：`IndexingService` 的切块算法（RAG 的"R"和"G"都从这里开始）和 `QaService` 的检索打分（一个手写的迷你 RAG）。你会看到它与你熟悉的 LangChain 概念一一对应。

回顾组织方式（第 5 章讲过的 DI 链，本章的主角就是链上的四个类）：

```
PersistenceService（唯一碰磁盘）
   ├─→ DocumentService（文档增删改查）
   ├─→ IndexingService（切块与索引）
   └─→ QaService（问答，还可以选用 IndexingService）
```

## 7.1 PersistenceService：文件系统的唯一守门人

`src/services/persistence-service.ts`，94 行。设计意图：**全项目只有这个类 import `fs`**——所有磁盘操作收口于此，将来换成 SQLite 或云存储，只改这一个文件。

**构造即初始化**（9-20 行）：

```ts
constructor(dataDir: string) {
  this.dataDir = dataDir;
  this.documentsDir = path.join(dataDir, 'documents');
  this.indexDir = path.join(dataDir, 'index');
  this.ensureDirectories();    // 构造时就把目录结构建好——"这个对象存在，目录就存在"
}
private ensureDirectories() {
  fs.mkdirSync(this.dataDir, { recursive: true });      // recursive：父目录不存在也一并建
  fs.mkdirSync(this.documentsDir, { recursive: true });
  fs.mkdirSync(this.indexDir, { recursive: true });
}
```

这就是 `feature_list.json` 里 `data-directory` 功能的实现证据：应用一启动、`new PersistenceService(dataDir)` 一执行，`%APPDATA%\knowledge-base\knowledge-base-data\` 及其子目录就位。

**核心读写**（23-50 行）——两个泛型方法承担全部 JSON 存取：

```ts
readJson<T>(relativePath: string): T | null {
  const fullPath = path.join(this.dataDir, relativePath);   // 相对路径 → 绝对路径
  if (!fs.existsSync(fullPath)) return null;                 // 不存在返回 null（而非抛错）
  return JSON.parse(fs.readFileSync(fullPath, 'utf-8'));     // 读文本 → 解析为对象
}

writeJson<T>(relativePath: string, data: T): void {
  const fullPath = path.join(this.dataDir, relativePath);
  fs.mkdirSync(path.dirname(fullPath), { recursive: true }); // 写前确保父目录在
  fs.writeFileSync(fullPath, JSON.stringify(data, null, 2), 'utf-8');  // 2 = 缩进两格（人类可读）
}
```

`JSON.stringify(x, null, 2)` 是 Python `json.dumps(x, indent=2)` 的对应物；`JSON.parse` 对应 `json.loads`。注意"文件不存在返回 null 而不是抛异常"的约定——调用方用 `?? []` 兜底（如 `document-service.ts:19`：`const docs = this.persistence.readJson<Document[]>(DOCUMENTS_META); return docs ?? [];`），首次启动没有元数据文件时优雅降级为空数组。

**一处值得批判性阅读的注释**——`writeJson` 上方写着 `/** Write a JSON file atomically. */`（"原子地写"），但实现就是一次普通的 `writeFileSync`：**没有**"先写临时文件再 rename"的原子化套路。如果写入中途断电，文件会残缺。注释描述的是愿望而非事实——读代码时永远以实现为准，这种"注释与代码的落差"本身就是信号（要么补实现，要么改注释；第 12 章有对应练习）。

其余方法（`readText`/`writeText` 读写纯文本、`copyFileToDocuments` 复制原件、`deleteFromDocuments` 删除、`listFiles` 列目录、`exists` 探测、三个 getter）都是薄封装，对照第 5 章的 fs 速查表即可读懂。

## 7.2 DocumentService：文档的增删改查

`src/services/document-service.ts`，89 行。元数据存于单个 JSON 文件（`documents-meta.json`），正文单独存文本文件：

```
knowledge-base-data/
├── documents-meta.json        ← Document[] 数组（元数据总账）
├── documents/                 ← 原始文件副本（按原文件名）
│   └── 提示词.md
└── content/
    └── 76cbdda6-....txt       ← 每篇文档提取的正文，文件名 = 文档 id
```

**importDocument（23-53 行）**——最完整的业务方法，走读一遍：

```ts
importDocument(filePath: string): Document {
  if (!fs.existsSync(filePath)) throw new Error(`File not found: ${filePath}`);
  // 防御：路径不存在直接抛异常（IPC 会把异常序列化传回渲染进程）

  const filename = path.basename(filePath);            // 'C:/a/b/notes.md' → 'notes.md'
  const content = fs.readFileSync(filePath, 'utf-8');  // 读正文
  const stats = fs.statSync(filePath);                 // 取文件大小

  const doc: Document = {
    id: uuidv4(),                       // 生成全局唯一 id（v4 = 随机 UUID）
    title: filename.replace(/\.[^.]+$/, ''),  // 'notes.md' → 'notes'（正则去掉扩展名）
    filename,
    importedAt: new Date().toISOString(),     // '2026-10-24T06:07:40.400Z'（ISO 8601）
    size: stats.size,
    status: 'imported',                       // 生命周期起点
  };

  this.persistence.copyFileToDocuments(filePath, filename);   // 存原件
  this.persistence.writeText(`content/${doc.id}.txt`, content); // 存正文
  const docs = this.listDocuments();   // 读旧账
  docs.push(doc);                      // 追加
  this.persistence.writeJson(DOCUMENTS_META, docs);   // 写回新账（读-改-写全量覆写）
  return doc;                          // 新文档（含 id）返回给界面
}
```

值得吸收的三个细节：`uuid` 是 npm 包（`import { v4 as uuidv4 } from 'uuid'`），随机 128 位 id 当主键；正则 `/\.[^.]+$/` 读作"最后一个点及其后的所有字符"；`new Date().toISOString()` 是时间存储的标准姿势（带时区、可排序、可 `new Date(s)` 还原）。

**updateDocument（67-75 行）**——不可变更新的服务层版本：

```ts
const index = docs.findIndex(d => d.id === id);   // 找下标
if (index === -1) return null;
docs[index] = { ...docs[index], ...updates };     // 旧字段 + 更新字段 → 新对象
this.persistence.writeJson(DOCUMENTS_META, docs);
```

`Partial<Document>` 参数类型（`updates`）读作"Document 的字段每个都变成可选"——调用方可以只传要改的字段。【类比 Python】`TypedDict` 的 Partial 版本或 `**updates` 字典合并。

**deleteDocument（78-88 行）**：找到文档 → 删 `documents/` 里的原件 → `filter` 掉该条 → 全量写回。**注意它删了什么、没删什么**：`content/<id>.txt` 和（若已索引的）`chunks/<id>.json`、`index-meta.json` 里的登记都**没删**——孤儿文件留在磁盘上。这是一个真实的不完整实现，第 8 章问题清单编号 ⑤，第 12 章练习 L3-3。

## 7.3 IndexingService：切块与索引（RAG 的地基）

`src/services/indexing-service.ts`，130 行。两件事：把正文切成 chunk；维护"哪些文档已索引"的登记簿。

### 切块算法 chunkDocument（92-116 行）——逐行精读

目标：把长文切成 **约 500 字符** 的块，且**尽量不切断段落**（段落完整性 > 尺寸精度，这是 RAG 切块的基本功：检索命中后引用的摘要要读得通）。

```ts
private chunkDocument(documentId: string, content: string): Chunk[] {
  const CHUNK_SIZE = 500;
  const chunks: Chunk[] = [];

  // 按"段落"切分：正则 /\n\s*\n/ = 换行 + 任意空白 + 换行（即空行分隔的段落）
  const paragraphs = content.split(/\n\s*\n/).filter(p => p.trim().length > 0);

  let buffer = '';          // 累积桶
  let chunkIndex = 0;

  for (const para of paragraphs) {
    // 桶非空 且 再装就要超限 → 先把桶倒出去成一个 chunk，当前段落成为新桶底
    if (buffer.length + para.length > CHUNK_SIZE && buffer.length > 0) {
      chunks.push(this.createChunk(documentId, chunkIndex++, buffer.trim()));
      buffer = para;
    } else {
      buffer += (buffer ? '\n\n' : '') + para;   // 还装得下 → 续进桶（段落间补回空行）
    }
  }

  if (buffer.trim()) {   // 别忘了最后一桶
    chunks.push(this.createChunk(documentId, chunkIndex, buffer.trim()));
  }
  return chunks;
}
```

**手工模拟一遍**（务必自己拿纸笔走一遍，比看十遍代码有用）。设 `CHUNK_SIZE=50`，输入三段：A=30 字符、B=40、C=20：

| 步骤 | buffer 内容 | 长度判断 | 动作 | chunks |
|---|---|---|---|---|
| 装 A | '' → A（30） | 30 ≤ 50，直接装 | buffer=A | [] |
| 装 B | 30+40=70 > 50 且 buffer 非空 | **先倒出 A**，buffer=B | chunks=[chunk0(A)] |
| 装 C | 40+20=60 > 50 且 buffer 非空 | **先倒出 B**，buffer=C | chunks=[A, B] |
| 收尾 | buffer=C 非空 | 倒出 C | chunks=[A, B, C] |

结果三个 chunk，每块 ≤ 50，任何段落都没被腰斩。两个隐含行为：单个段落本身超 500 字符时**不会**被硬切（它独自成一个超长 chunk——算法优先保段落完整）；空行 `/\n\s*\n/` 是段落边界，Markdown 的 `## 标题` 行与正文若之间无空行会被并进同一段。

`createChunk`（118-129 行）给每块发 uuid、记录序号、算元数据：`charCount`/`wordCount` 都转成字符串存（因为 `Chunk.metadata` 的类型是 `Record<string, string>`，`types.ts:18`）。`wordCount` 用 `content.split(/\s+/).length` 估——按空白切，对中文会严重低估（中文无词间空格），这是个小坑，练习 L1 可修。

### 两条索引路径 startIndexing（23-54 行）——注意不对称！

```ts
async startIndexing(documentId?: string): Promise<IndexStatus> {
  const status = this.getStatus();

  if (documentId) {
    // 路径一：只索引这一篇（详情页 "Index Document" 按钮走这里）
    const content = this.persistence.readText(`content/${documentId}.txt`);
    if (!content) return { ...status, status: 'error' };   // 展开覆盖状态为 error（第 2 章的 spread）
    const chunks = this.chunkDocument(documentId, content);
    this.persistence.writeJson(`chunks/${documentId}.json`, chunks);
    return this.getStatus();
    // ⚠ 只写了 chunks/<id>.json，没有更新登记簿 index-meta.json！
  }

  // 路径二：全量索引（IPC 无参调用走这里）
  const docsMeta = this.persistence.readJson<Document[]>('documents-meta.json') ?? [];
  const chunksMeta = this.persistence.readJson<Record<string, string[]>>(INDEX_META) ?? {};

  for (const doc of docsMeta) {
    if (chunksMeta[doc.id]) continue;              // 已登记的跳过（增量索引）
    const content = this.persistence.readText(`content/${doc.id}.txt`);
    if (!content) continue;
    const chunks = this.chunkDocument(doc.id, content);
    this.persistence.writeJson(`chunks/${doc.id}.json`, chunks);
    chunksMeta[doc.id] = chunks.map(c => c.id);    // ⭕ 这里才登记
  }

  this.persistence.writeJson(INDEX_META, chunksMeta);   // 登记簿落盘
  return this.getStatus();
}
```

`documentId?`（可选参数）让一个方法承担两种语义。**但两条路径不对称**：单文档路径写了 chunks 文件却**不更新登记簿**。后果链条（第 8 章问题清单 ④）：

1. `getStatus()` 数的是登记簿长度 → 单文档索引后状态不变；
2. `getAllChunks()`（问答检索的数据源，79-89 行）**遍历的是登记簿** → 单文档索引的块**问答永远搜不到**；
3. 界面上"Index Document"按钮调的就是单文档路径——所以从界面索引的文档，问答引用里永远不会出现。

这是"一个方法两条路径，一条忘了同步副作用"的经典 bug 形态。修复不难（把登记逻辑抽出来两路共用），难的是**发现**——发现它靠的是追踪"数据从哪来"（第 8 章的方法论）。

### getStatus（57-71 行）——状态的推导

```ts
const currentIndexed = Object.keys(chunksMeta).length;   // 登记簿里几个文档
const totalDocuments = docs.length;
const isReady = currentIndexed === totalDocuments && totalDocuments > 0;

return {
  status: isReady ? 'ready' : currentIndexed > 0 ? 'indexing' : 'idle',
  currentIndexed, totalDocuments,
  lastIndexed: new Date().toISOString(),
};
```

状态不是存储的，是**每次现算**的（无状态推导，免同步）。三个状态的语义：一篇没索引 = `idle`；索引了一部分（或总数对不上）= `indexing`（严格说是"进行中/未完成"）；全部索引完 = `ready`。注意返回类型 `IndexStatus` 与渲染层期待的 `AppStatus` 字段名不一致——6.6 已预告，8.4 展开。

## 7.4 QaService：手写的迷你 RAG

`src/services/qa-service.ts`，135 行。构造（44-47 行）：

```ts
constructor(persistence: PersistenceService, indexingService?: IndexingService) {
  this.persistence = persistence;
  this.indexingService = indexingService ?? new IndexingService(persistence);  // 可选注入，缺省自建
}
```

第二个参数可选（`?`）——`main.ts:44` 实际传了共享实例，但单独 new `QaService` 也能活。这是"便利性与显式性"的折中。

### ask()（50-103 行）——四步流水线

```ts
async ask(question: string): Promise<QAResponse> {
  // ① 模拟网络延迟（LLM 的仪式感）
  await new Promise(resolve => setTimeout(resolve, 100 + Math.random() * 400));

  const chunks = this.indexingService.getAllChunks();   // ② 取回全部已登记的 chunk
  const citations: Citation[] = [];

  if (chunks.length > 0) {
    // ③ 检索：关键词重合度打分
    const questionWords = question.toLowerCase().split(/\s+/).filter(w => w.length > 2);
    // 问题拆词（去停用短词："is/a/of" 这类 ≤2 字符的丢掉）

    const scored = chunks.map(chunk => {
      const contentLower = chunk.content.toLowerCase();
      const score = questionWords.reduce(        // reduce 累加：每个命中词 +1
        (acc, word) => acc + (contentLower.includes(word) ? 1 : 0),
        0
      );
      return { chunk, score };
    });

    const relevant = scored
      .filter(s => s.score > 0)        // 只留有命中的
      .sort((a, b) => b.score - a.score)  // 按分数降序（比较函数：负数 a 前 b 后）
      .slice(0, 2);                     // top-2

    // 给引用补文档标题（chunk 只存 docId，要回元数据里查 title）
    const docs = this.persistence.readJson<...>('documents-meta.json') ?? [];
    for (const { chunk } of relevant) {
      const doc = docs.find(d => d.id === chunk.documentId);
      citations.push({
        documentId: chunk.documentId,
        documentTitle: doc?.title ?? 'Unknown Document',
        chunkIndex: chunk.index,
        excerpt: chunk.content.substring(0, 200),   // 前 200 字符做摘录
      });
    }
  }

  // ④ 生成答案（假 LLM）+ 组装响应
  const answer = this.generateAnswer(question, citations);
  const response: QAResponse = {
    answer, citations,
    confidence: citations.length > 0 ? 0.85 : 0.3,   // 有据 0.85，无据 0.30
    timestamp: new Date().toISOString(),
  };
  this.saveToHistory(question, response);   // 每问必记录
  return response;
}
```

`reduce((acc, x) => ..., 0)` 是"带初值折叠数组"：【类比 Python】`functools.reduce(lambda acc, w: acc + (1 if w in text else 0), words, 0)`。`for (const { chunk } of relevant)` 是循环 + 解构二合一：每轮从 `{chunk, score}` 对象里直接拆出 `chunk`。

### generateAnswer（110-128 行）——模板即"假 LLM"

先按问题关键词匹配 `MOCK_PATTERNS`（8-38 行的五组 `keywords/answer/excerpt`），命中则返回模板答案（有引用时拼接文档标题和摘录）；不命中但有引用，返回"据已有文档……"的保守回答；什么都没有，返回"请先导入并索引文档"的引导语。**置信度与答案内容解耦**（0.85 只代表"有引用"）——这是简化，真实系统里 confidence 应该反映生成质量。

### 与 LangChain 的完整对照

把 `ask()` 拆成管道，你一眼就能看出它是什么：

```
question ──→ [分词+打分检索] ──→ [top-2 截断] ──→ [模板生成] ──→ [结构化输出+落盘]
                Retriever          类似 k=2          LLM 的替身     OutputParser + Memory
```

| 本项目 | LangChain 概念 | 差距与升级方向（练习 L5） |
|---|---|---|
| `chunkDocument`（按段落 500 字符） | `TextSplitter`（如 RecursiveCharacterTextSplitter） | 无重叠（overlap）策略 |
| 关键词 includes 打分 | VectorStore 相似度检索（embedding） | 换成真向量检索 |
| `MOCK_PATTERNS` + 模板拼接 | prompt + LLM 调用 | 换成 `ChatOpenAI` 等 |
| `QAResponse{answer,citations,confidence}` | 带 schema 的结构化输出 | 已同构 |
| `saveToHistory` 每问必存 | 消息历史 / memory | 已同构 |

换句话说：**这个项目是去掉向量和 LLM 的 RAG 骨架**。把检索和生成两个函数换成真实现，它就是一个本地知识库问答应用——这正是第 12 章最高阶练习的设计意图。

## 本章小结

- `PersistenceService`：唯一碰 `fs` 的类；`readJson<T>` 的 `null`-兜底约定；"atomic"注释与实现不符（批判性阅读）。
- `DocumentService`：读-改-写全量覆写的元数据管理；uuid 主键；`Partial` + 展开实现局部更新；删除留孤儿文件（问题⑤）。
- `IndexingService`：段落优先的 500 字符切块（手工模拟过一遍）；**单文档/全量两条路径不对称**，单文档不写登记簿，导致问答搜不到（问题④）——本章最重要的 bug 案例。
- `QaService`：延迟→检索→top-2→模板生成→落盘，四步流水线即迷你 RAG；与 LangChain 的组件一一对应，升级路径清晰。
- 下一章：把四层架构和服务层串成完整数据流，配行号走读三个典型场景，并盘点全项目已知问题清单。
