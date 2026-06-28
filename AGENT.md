# AGENT.md · 命途推演 项目上下文

> 给 AI 编码 Agent 读的项目说明书。如果你（AI）被叫来在这个项目里写代码、修 bug、加功能，先把这份读完。

---

## 1. 一句话项目定义

**命途推演** 是一个零构建、单文件、纯前端的"人生决策推演"工具。用户在表单里描述一个决策（转专业 / 跳槽 / 婚恋 / 创业……），三阶段 AI 流水线（架构师 → Tavily 检索 → 推演师）产出一份 6,000-15,000 字的战略推演报告，可一键导出为独立 HTML 卷宗。

整个应用是**一个 `index.html`**，不需要 build、不需要后端、所有数据存浏览器 localStorage。

---

## 2. 当前文件清单

```
命途推演/
├── index.html      ~3500 行 · 148 KB · 主应用（HTML + 内联 CSS + 内联 JS）
├── README.md       面向用户的使用说明
├── prompts.md      两阶段 prompt 完整版（人类可读，便于审阅与迭代）
└── AGENT.md        本文件
```

**绝对不要**：
- 拆分 index.html 为多文件（违背"零构建"哲学）
- 引入构建工具（webpack / vite / rollup）
- 引入框架（React / Vue / Svelte）
- 引入打包好的 npm 包（直接用 CDN）

---

## 3. 技术栈

| 层 | 选择 | 理由 |
|---|---|---|
| HTML | 单文件 | 用户双击即用，零部署成本 |
| CSS | 手写内联 | 不依赖 Tailwind / 任何 CSS 框架 |
| JS | Vanilla ES2017+ | 不引入 React 等 |
| Markdown 渲染 | `marked@12.0.2` via CDN（jsdelivr → unpkg → fastly 三级 fallback）+ 内置 `simpleMarkdown()` 兜底 | CDN 挂掉也能渲染 |
| 字体 | Newsreader（衬线）+ JetBrains Mono + 系统 sans | 编辑型质感 |
| 图标 | Phosphor Icons via CDN | 禁用 emoji（视觉规范要求） |
| LLM | 任意兼容 OpenAI Chat Completions 协议的端点（用户自带 Key） | DeepSeek / Claude / GPT / 智谱 / 通义 / Kimi |
| 联网检索 | Tavily Search API（可选） | LLM-friendly、CORS 友好 |

---

## 4. 核心架构：三阶段 AI 流水线

```
表单（7 组分组 + 1 个补充框）
  ↓
[Stage 1 · Architect]
  - LLM 一次非流式调用
  - 输出结构化 JSON: sections[] + (optional) search_queries[]
  - JS 端 normalizeSections 归一化 word_budget
  ↓
[Stage 1.5 · Tavily Web Search]（仅当用户启用联网检索时）
  - 优先用架构师生成的 search_queries
  - fallback 到 buildSearchQueries(form) 硬编码模板
  - 召回去重后取 top 8 注入下游
  ↓
[Stage 2 · Strategist]
  - LLM 流式调用（SSE）
  - prompt 中注入 sections + (optional) search_evidence
  - 输出 Markdown 长文（3,500 / 7,000 / 11,000 字按深度档位）
  ↓
渲染 + 历史保存 + 导出 HTML 卷宗
```

### 为什么是这个顺序

**架构师必须先于检索**。理由：架构师才理解决策的"语义"，能避开字面歧义陷阱。例如用户标题「兰大大二从地质工程转人工智能」如果直接做 Tavily 查询，会拉到"兰州中考招生"——因为 Tavily 的中文语义索引把"兰大"和"大二"分词后匹配中学相关内容。架构师在理解决策后生成的查询是「兰州大学 人工智能学院 转专业 录取率 2026」，召回质量天差地别。

### 为什么不只用单一 prompt

不同决策需要完全不同的章节结构：
- 婚恋决策需要"兼容性矩阵"和"原生家庭对冲"
- 创业决策需要"现金流断点模拟"
- 健康决策需要"QALY 曲线"和"治疗方案对照"

单一模板套所有问题会失真。架构师 AI 先输出 `decision_category` + 定制 `sections`，再交给推演师执行。

---

## 5. index.html 内部结构（按出现顺序）

```
<head>
  Google Fonts（Newsreader + JetBrains Mono）
  Phosphor Icons CSS（regular + bold + fill）
  marked.js（含 onerror 三级 fallback）
  <style>  CSS Variables + 全部样式  ~700 行
  </style>

<body>
  <header class="topbar">           品牌名 + 历史/设置按钮
  <section class="hero">            编辑型 hero（仅装饰）
  <main class="workspace">
    <section class="pane" id="form-pane">      左栏：7 组表单
    <section class="pane" id="output-pane">    右栏：输出区（empty state / 进度 / 报告）
  <footer class="site-footer">
  <aside class="drawer drawer-settings">       设置抽屉（含 Tavily）
  <aside class="drawer drawer-history">        历史抽屉
  <div class="toast-container">

<script>
  1.  ARCHITECT_PROMPT          架构师系统 prompt（模板字符串）
  2.  STRATEGIST_PROMPT_TEMPLATE 推演师系统 prompt 模板（含变量占位）
  3.  state                      全局状态对象
  4.  DEPTH_BUDGET / MAX_TOKENS_BY_DEPTH 配置常量
  5.  loadConfig / saveConfig / clearKey         配置持久化
  6.  readForm / loadFormDraft / formatFormForPrompt  表单序列化
  7.  testConnection / callLLMStream / callLLMNonStream  LLM API 抽象
  8.  buildSearchQueries / tavilySearch / runWebSearch / formatSearchEvidence  Tavily 集成
  9.  getDefaultSections / extractJSON / validateSections / normalizeSections  架构师辅助
  10. runArchitect / runStrategist               主流水线两个核心调用
  11. runPipeline                                 整体流水线编排
  12. validateBeforeRun / renderSectionsPreview / renderQueriesPreview / renderEvidencePreview / renderReport / renderMarkdown / simpleMarkdown / parseTables  渲染层
  13. exportHTML / copyMarkdown / buildExportHTML  导出
  14. loadHistory / saveCurrentToHistory / renderHistory / loadHistoryItem / deleteHistoryItem  历史
  15. toast / escapeHtml / openDrawer / closeDrawer / setupIntersectionFade  UI helpers
  16. bindEvents / init                            初始化
```

---

## 6. State 模型

```js
state = {
  config: {
    apiBase: string,        // LLM 端点
    apiKey: string,         // LLM key
    model: string,          // 模型名
    depth: '快速'|'标准'|'极致',
    searchEnabled: boolean,
    tavilyKey: string
  },
  form: { ... },            // 表单序列化结果
  sections: {               // 架构师输出
    decision_category, structure_strategy, rationale,
    sections: [{ title, purpose, required_elements, word_budget }],
    search_queries: [...],  // 可选
    fallback: boolean       // 是否用了默认 6 段
  } | null,
  evidence: {               // Tavily 检索结果
    queries: [], results: [{ title, url, content, score, query }], errors: []
  } | null,
  report: string,           // 推演师生成的 Markdown 原文
  isGenerating: boolean,
  abortController: AbortController | null,
  history: [...]            // 最多 20 条
}
```

`localStorage` 键：
- `mt_config` → state.config
- `mt_form_draft` → 表单草稿（每次输入自动保存）
- `mt_history` → 历史推演数组

---

## 7. Prompt 工程

### 两份核心 prompt 都是 JS 模板字符串常量

- `ARCHITECT_PROMPT` ≈ 220 行：系统消息，无变量。运行时只在 user message 里注入数据。
- `STRATEGIST_PROMPT_TEMPLATE` ≈ 260 行：系统消息，含以下占位符，运行时 `.replace()` 替换：

| 占位符 | 替换为 |
|---|---|
| `${CURRENT_YEAR}` | `new Date().getFullYear()` |
| `${CURRENT_YEAR_PLUS_10}` | 当前年份 + 10 |
| `${DEPTH}` | 用户选择的深度档位 |
| `${TOTAL_BUDGET}` | 所有 sections 的 word_budget 之和 |
| `${SECTIONS_JSON}` | 架构师 sections 数组的 pretty JSON |
| `${SEARCH_EVIDENCE}` | `formatSearchEvidence(evidence)` 输出，或空串 |
| `${USER_DATA}` | `formatFormForPrompt(formData)` 输出 |

### 修改 prompt 的时候

- 这些 prompt 同时存在两个地方：`index.html` JS 内 + `prompts.md`。两边**必须保持一致**，但可读性优先于代码内（prompts.md 是给人读的，可以排版更好）。
- 改完 prompt 必须用 `node --check /tmp/check.js` 验证 JS 语法（详见 § 12 测试）。
- **绝对不要**在模板字符串内用 `` ` ``（反引号）做小代码标记 — 会提前关闭外层模板。要么用 `\``` 转义，要么用「」「角引号」代替。
- 占位符 `${VAR}` 在 ES 模板字符串内会被立即求值。`STRATEGIST_PROMPT_TEMPLATE` 里所有 `${...}` 都是用 `${'$'}{VAR}` 写法转义的（输出字面的 `${VAR}`），运行时再 replace。

### Prompt 的核心约束

**架构师**：
- 必须输出严格 JSON，sections.length ≥ 6
- 每段含 title / purpose / required_elements / word_budget
- 联网启用时必须额外输出 3-5 条 `search_queries`，要遵循"避免字面歧义"原则
- 失败时（非法 JSON / 段数不足）前端自动 fallback 到默认 6 段（`getDefaultSections`）

**推演师**：
- 总字数 = sections 的 word_budget 之和，每章 ≥ word_budget × 0.85（硬性）
- 深度档位最低字数底线：快速 3000 / 标准 6000 / 极致 9000
- 禁用空词（"多元化"、"做最好的自己"等）
- 必须用具体数字（薪资写 24k/月 而非"不错"）
- 关键拐点用 `**粗体**`
- 若有检索证据：每章至少明引 1 条，全文 ≥ ceil(N×0.6) 条，末尾必须有 `## 附录 · 引用文献索引`

---

## 8. 视觉规范

### 主应用：editorial minimalism（浅色暖白）

- 背景 `#FBFBFA`（暖白）
- 主文字 `#111111`（off-black，**不是纯黑**）
- 强调色仅用四个 pastel：`#FDEBEC / #E1F3FE / #EDF3EC / #FBF3DB`
- 衬线大字标题（Newsreader）+ 等宽 meta（JetBrains Mono）+ 系统 sans 正文
- 1px 发丝边 `#EAEAEA` 是唯一的视觉分隔（**禁止重阴影**）
- 段落分隔用 hairline 而非卡片盒
- **emoji 完全禁止**（用 Phosphor 图标替代）
- 入场动画：`translateY(12px)` + `opacity:0` → `cubic-bezier(0.16, 1, 0.3, 1)` 600ms，IntersectionObserver 触发，仅一次

### 导出 HTML：「命运卷宗」（古纸朱红）

刻意与主应用形成反差。
- 米白 `#f5f1e8` + 墨黑 + 朱红 `#8b1a1a` + 古铜 + 墨绿
- Noto Serif SC 全衬线
- 朱红方章、章首字下沉、古典双横线表格、底部「天機有數 · 決策在人」
- print 友好（含 `@media print` 规则）

这两套语言**不应互相借用**。修改时不要把朱红用到主应用，也不要把 pastel 用到导出卷宗。

---

## 9. 关键约定与"坑"

### 9.1 marked.js 兜底

如果用户网络挂了三个 CDN 全挂，`typeof marked === 'undefined'`。所有 `marked.parse(...)` 调用必须通过 `renderMarkdown(...)` 包装：

```js
function renderMarkdown(text) {
  if (typeof marked !== 'undefined' && marked?.parse) {
    try { return marked.parse(text || ''); }
    catch (e) { return simpleMarkdown(text); }
  }
  return simpleMarkdown(text);
}
```

**不要直接调用 `marked.parse`**。

### 9.2 simpleMarkdown 已知边界

`simpleMarkdown()` 是 ~80 行的兜底渲染器，覆盖：headers / paragraphs / **bold** / *italic* / `code` / 代码块 / 表格（行级解析器 `parseTables()`） / blockquote / 列表 / hr。

**不支持**：嵌套列表、链接（`[text](url)`）、图片、HTML 内嵌、行内 LaTeX。如果推演师将来要用这些，需要扩展。

如果在 simpleMarkdown 里加新规则，记得在第 1 步 HTML 转义之后处理（除非新规则需要 `<` `>`）。已经踩过的坑：
- `>` 在第 1 步被转义成 `&gt;`，blockquote 正则要匹配 `&gt;` 而不是 `>`
- 代码块占位符不能用 `§§CODE${i}§§`（会与中文混淆），改用 `CODEBLOCK_PLACEHOLDER_${i}_END`
- 段落包装要显式排除 `CODEBLOCK_PLACEHOLDER_*`，否则代码块被包进 `<p>`

### 9.3 表单字段约定

所有"明确选项"必须用点选组件（`.pill` 单选 / `.pill` + `[data-multi]` 多选 / `.segment` 数值分段 / `.depth-pill` 三档），**禁止**用自由文本输入。

新增点选字段流程：
1. HTML 加 `<div class="pill-group" data-field="myfield" [data-multi]>...<button class="pill" data-val="X">X</button>...</div>`
2. JS `readForm()` 加一行：`myfield: getSinglePill('myfield')` 或 `getMultiPills('myfield')`
3. JS `loadFormDraft()` 加一行：`setSinglePill('myfield', draft.myfield)` 或 setMultiPills
4. JS `formatFormForPrompt()` 加一行：`push('我的字段', f.myfield)`

无需改其他东西，click handler 是全局委托的。

### 9.4 max_tokens 与字数关系

中文 1 字 ≈ 1.5-2 tokens。`MAX_TOKENS_BY_DEPTH = { 快速: 6000, 标准: 12000, 极致: 16000 }`。

**大多数 LLM 单次响应输出上限是 8192 tokens**（≈ 6000 字）：DeepSeek、Claude 3.5 Sonnet、Qwen-Max、GLM-4 都是。只有 GPT-4o（16384）能撑到 11k 字。所以"极致档 11000 字"对大多数模型其实是上限了，再多就会被截。**不要把极致档目标改回 13000+**，除非你实现了多轮续写。

### 9.5 CORS 已验证清单

- ✅ DeepSeek (api.deepseek.com)
- ✅ 智谱 (open.bigmodel.cn)
- ✅ OpenAI (api.openai.com)
- ✅ 通义 (dashscope.aliyuncs.com 兼容入口)
- ✅ Tavily (api.tavily.com)
- ⚠️ Anthropic 官方 (api.anthropic.com) 不支持浏览器直连，需要 one-api / new-api 自建代理

### 9.6 SSE 流式解析

`callLLMStream()` 实现了标准 OpenAI SSE 格式解析。各厂商应该都兼容。如果遇到某家厂商行为怪异（如 chunk 不完整、字段名不同）：
- 在 `callLLMStream` 的 `for...of lines` 循环里加针对性 `JSON.parse` 容错（已有 try/catch）
- 必要时加 provider 分支

---

## 10. 常见任务速查

### 任务：添加一个新的决策类型

1. HTML 表单 ① 决策本身 → 决策类型 pill-group 里加一个 `<button class="pill" data-val="新类型">新类型</button>`
2. `ARCHITECT_PROMPT` 里【四、按决策类型的章节适配指南】加一个 ▸ 新类型 → 推荐章节
3. `ARCHITECT_PROMPT` 里【七、search_queries】按决策类型查询指南加一组示例

### 任务：调整深度档位字数

只改 `DEPTH_BUDGET` 和 `MAX_TOKENS_BY_DEPTH` 两个常量，**同时**改 hero 区显示的字数和 depth-pill 的 `.depth-meta` 文案。然后 `normalizeSections` 会自动同步。

### 任务：增加一个 LLM provider 的特殊参数

在 `callLLMStream` / `callLLMNonStream` 的 `JSON.stringify({...})` 里按 provider 加分支。可以用 `state.config.apiBase` 字符串判断（如 `apiBase.includes('moonshot')` → Kimi 特殊参数）。

### 任务：修改某段 prompt

1. 在 index.html 里改 `ARCHITECT_PROMPT` 或 `STRATEGIST_PROMPT_TEMPLATE`
2. 在 prompts.md 里同步更新（人类可读版）
3. 运行 `node --check` 验证 JS 语法（template literal 容易写挂）
4. 浏览器实测：用经典测试用例（"兰大转 AI"）跑一遍

### 任务：增加一个新的输出格式（如 PDF）

`buildExportHTML()` 是黄金参考。新加一个 `buildExportPDF()` / `buildExportDocx()` 函数，在导出工具栏（`.output-toolbar`）加按钮，绑定 click handler。注意 PDF 在浏览器里只能通过浏览器原生打印（导出 HTML 已经有 `@media print` 规则，用户可以 `Cmd+P` 出 PDF）。

### 任务：调试 Tavily 查询质量

1. 跑一次推演（确保联网开关 + Tavily Key 都对）
2. 看架构师输出的 search_queries 是否合理（UI 上「检索查询计划」卡片）
3. 看 stage-search 阶段的进度细节文案：「召回 N 条证据」
4. 看 evidence preview 卡片**按查询分组展示**的内容是否相关、是否双路径均衡
5. 若不相关或一边倒：调 `ARCHITECT_PROMPT` 里【七】小节的查询生成规则 + 反例库 + 双路径均衡硬性要求

### 任务：调整检索召回参数

- 单条 Tavily `max_results`：`tavilySearch()` 内（当前 10）
- 最终保留总数：`runWebSearch()` 调用 `aggregateSearchResults` 时的 `targetTotal`（当前 12）
- 每条查询保底数：`minPerQuery`（当前 2），增大会更强制覆盖、可能降低 score 质量
- 架构师生成查询数上限：`validateSections` 内 `.slice(0, 6)`（当前 6）
- 架构师生成查询数要求：`ARCHITECT_PROMPT` 里【七】小节首行（当前 4-6）+ `runArchitect` 的 searchHint 文案

---

## 11. 不要做的事（红线）

| ❌ | 理由 |
|---|---|
| 拆 index.html 成多文件 | 违背"零构建"哲学 |
| 引入 bundler / framework | 同上 |
| 加 emoji 到 markup | 视觉规范明确禁止 |
| 直接用 `marked.parse(...)`，不走 `renderMarkdown` 包装 | CDN 挂掉就坏 |
| 在模板字符串里裸用反引号做代码标记 | 提前关闭外层模板 |
| 让主应用与导出卷宗共用颜色 / 字体 | 两套视觉语言要保持反差 |
| 把"明确选项"字段做成文本输入 | 表单约定 |
| 给极致档目标超过 11k 字 | 大多数 LLM 单次输出上限 |
| 把搜索移回架构师之前 | 字面歧义问题（详见 § 4） |
| 在 architect prompt 输出格式里加新必填字段而不更新 `validateSections` | 校验失败会触发 fallback |
| 把 API Key 上传到任何服务器 | 隐私底线 |
| 忽略 AbortController | 用户点"停止"必须立即中断 |

---

## 12. 测试与验证

### 12.1 JS 语法快速校验

```bash
node -e "
const fs = require('fs');
const c = fs.readFileSync('/path/to/index.html','utf8');
const inline = (c.match(/<script>([\s\S]*?)<\/script>/g)||[]).find(s=>!s.includes('src='));
const js = inline.replace(/^<script>/,'').replace(/<\/script>$/,'');
require('fs').writeFileSync('/tmp/check.js', js);
"
node --check /tmp/check.js
```

任何改 JS 的提交都必须先过这一步。template literal 嵌套出错时这里会精确报行号。

### 12.2 本地启动

```bash
cd /path/to/命途推演
python3 -m http.server 8000
# 浏览器开 http://localhost:8000
```

直接双击 index.html 也行，但用 http server 跑能避免某些浏览器对 file:// 的限制（特别是 fetch 跨域到 LLM API 时）。

### 12.3 端到端测试用例

经典用例：兰大大二从地质工程转人工智能（详见 README 与之前的对话记录）。手动操作或用 DevTools 控制台粘贴 fill-in 脚本。

测什么：
1. 架构师返回的 decision_category 是否合理（应为"资源配置型 + 时间窗口型"或类似）
2. sections 是否 ≥ 6 段，word_budget 总和接近目标字数
3. 联网开启时，search_queries 是否避开了字面歧义（不应有"兰州中考"类查询）
4. 推演师输出字数是否达到 word_budget × 0.85
5. 检索证据是否在报告中被明引（搜索 "（检索证据 ["）
6. 导出 HTML 卷宗能否正确渲染表格 / 代码块 / 引用

### 12.4 marked 兜底验证

DevTools 控制台执行 `window.marked = undefined; renderReport(state.report);`，看是否走 simpleMarkdown 路径正常渲染（应看到 `[命途推演] marked.js 未加载...` warning）。

---

## 13. 当前已知限制

1. **极致档实际上限 ~11k 字**（受 LLM 单次输出上限约束），不是 prompt 说的 13k+
2. **CORS 限制**导致 Anthropic 官方 API 不能直连，必须走代理
3. **没有多轮续写**功能，输出被截断只能手动重新生成
4. **没有用户系统 / 云存储**，换设备就没历史
5. **不支持嵌套 Markdown**（如代码块内的代码块、列表内的表格）
6. **Tavily 免费额度每月 1000 次**，单次推演消耗 3-5 次
7. **simpleMarkdown 不支持链接和图片**（推演师当前 prompt 不会输出这些，所以暂时无影响）

---

## 14. 项目哲学

- **结构化输入，结构化输出** — 不是 ChatGPT 包装器
- **不替用户做决定** — 把代价摆清楚，最终决定权在用户
- **不灌鸡汤** — 推演师 prompt 明确禁用了"多元化"、"自我成长"、"做最好的自己"等空话
- **冷峻、精准、有锋芒** — 允许直言"你正在用 X 换 Y，代价是 Z 万元"
- **数据本地，永不上传** — API Key、表单、报告全部 localStorage
- **零构建、零依赖、零部署** — 一个 HTML 双击就用

任何与上述哲学相违的改动，先三思再下手。

---

## 15. 联系点（给 Agent）

如果你要做较大的改动（新增阶段、改架构、加云端功能），**不要直接动手**。先与用户确认范围与方向，然后再开始改代码。

记住：这个项目的核心价值是"决策推演的质量"和"工具的优雅"，不是"功能堆砌"。每加一个东西，问一句：「不加这个，工具坏吗？」如果不坏，那大概率不该加。
