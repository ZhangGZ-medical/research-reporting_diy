---
name: research-reporting_diy
description: |
  通用结构化调研报告生成技能，适用于所有调研任务。用户说出调研需求后，按六阶段工作流执行：
  内容澄清 → 技能分析 → 提示词生成 → 用户确认 → 执行调研 → 报告撰写与DOCX交付。
  触发关键词：先给出良好的提示词、给出提示词包括技能调用、学术引用格式、深度调研、
  科技文献引用格式、多维度分析、引用追溯、生成调研报告、帮我调研、分析任务、
  做个调研、调研一下、出份报告。
  触发方式：用户说「调研 [主题]」或「帮我调研 [主题]」即可自动进入本技能工作流。
agent_created: true
---

# research-reporting_diy — 通用结构化调研报告生成技能

## 核心原则（首次执行必读）

| 原则 | 规则 |
|------|------|
| 交付格式 | **固定**：Markdown + DOCX（纵向A4），不纳入澄清范围 |
| 引用格式 | **固定**：国自然基金引文格式（正文方括号编号，文末顺序排列），不纳入澄清范围 |
| 澄清聚焦 | **只问内容**，不问格式 |
| 澄清上限 | 最多 50 次问答回合，超限则总结理解并执行 |
| 用户确认 | 提示词生成后**必须展示并等待用户确认**，用户说「确认执行」才启动 |
| 技能就绪 | 进入提示词生成前，确认所有必要技能已安装或已记录替代方案 |

---

## 六阶段工作流

### 阶段一 → 内容澄清

**目的**：明确任务范围、关键问题、预期产出。

**触发条件**：以下任一情形即启动澄清，否则直接进入阶段二：
- 任务含多个子方向，不知聚焦哪个
- 术语/产品名称/技术代号不精确
- 分析维度未定（技术/市场/政策/竞争）
- 未说要回答什么核心问题
- 时间/地理范围模糊
- 不知要概览还是详析（5页 vs 50页）

**执行方式**：
- 用 `AskUserQuestion` 工具，一次 ≤3 个问题，每题 ≤4 个选项
- 默认选项标 `(Recommended)`
- 动态追问，直到信息足够
- 如果用户说「直接输出无解释、仅交付最终版本」，跳过澄清

**典型首轮问题模板**：

```
Q1: 本次调研聚焦哪些维度？（多选）
   A. 技术/产品分析 (Recommended)
   B. 市场竞争格局
   C. 政策/监管环境
   D. 商业模式/投资价值

Q2: 调研的地理范围？
   A. 全球 (Recommended)
   B. 中国为主
   C. 特定国家/地区

Q3: 时间范围？
   A. 近3年 (Recommended)
   B. 近5年
   C. 不限
```

---

### 阶段二 → 技能分析

**目的**：识别所需技能，缺失则安装，确保工具箱就绪。

**步骤**：

**2.1 识别** — 对照下表判断需要哪些技能：

| 任务类型 | 可能需要的能力 | 候选技能 |
|---------|-------------|---------|
| 医学/生物医学 | 临床试验、文献检索 | `medical-research-toolkit` |
| 金融/市场数据 | 行情、财务分析 | `westock-data`、`neodata-financial-search` |
| PDF处理 | 读取/合并/转换 | `pdf`、`pdfkit-py` |
| 网页抓取 | 渲染页面、反爬 | `web-scraper`、`playwright-scraper-skill` |
| Excel/数据 | 分析、图表 | `xlsx`、`minimax-xlsx` |
| PPT生成 | 演示文稿 | `pptx-generator`、`pptx` |
| DOCX | 文档读写 | `docx` |
| 医学翻译 | 中英互译 | `cn2en_med_diy`、`en2cn_diy` |
| 深度调研 | 并行搜索 | `deep-research:research` 系列 |
| 新闻/政策 | 实时资讯 | `multi-search-engine`、`tencent-news` |

**2.2 检查** — 列出已安装技能：

```bash
ls ~/.workbuddy/skills/             # 用户级技能
ls .workbuddy/skills/ 2>/dev/null    # 项目级技能
```

同时对照 `available_skills` 列表交叉验证。

**2.3 安装** — 按优先级查找缺失技能：

```bash
# 优先级 1：本地技能市场（最快）
ls ~/.workbuddy/skills-marketplace/skills
cp -r ~/.workbuddy/skills-marketplace/skills/<name> ~/.workbuddy/skills/<name>

# 优先级 2：SkillHub 注册表
curl -s "https://lightmake.site/api/v1/search?q=<query>&limit=10"     # 搜索
curl -L -o /tmp/skill.zip "https://lightmake.site/api/v1/download?slug=<slug>"  # 下载
mkdir -p ~/.workbuddy/skills/<slug>
unzip -o /tmp/skill.zip -d ~/.workbuddy/skills/<slug>

# 优先级 3：备用注册表
npx skills find <query>                                              # Vercel Skills
npx clawhub search <query>                                           # ClawHub
```

**2.4 兜底** — 三个优先级都未找到 → 记录为「未找到，用替代方案」，不阻塞流程。

**2.5 输出** — 技能就绪状态报告：

```markdown
## 技能分析结果

**任务类型**：[类型]
**已安装**：skill-a ✅、skill-b ✅
**需安装**：skill-c ⬇️（已从本地市场复制）| skill-d ⚠️（未找到，用 WebSearch 替代）
**状态**：全部就绪 / N就绪 M缺失
```

---

### 阶段三 → 提示词生成

**目的**：输出一份可直接执行的完整提示词，含八要素。

#### 提示词模板

````markdown
## 调研任务提示词

### 1. 角色设定
你是 [根据任务领域动态设定，如：技术尽职调查专家 / 市场研究分析师 / 政策合规顾问]

### 2. 任务背景
[简短说明：为什么做、给谁看、要解决什么问题]

### 3. 事实数据基础
先读取以下文件获取已有数据：

| # | 文件路径 | 内容说明 |
|---|---------|---------|
| 1 | `[绝对路径]/xxx.md` | [说明] |
| 2 | `[绝对路径]/xxx.json` | [说明] |

### 4. 输出结构
```
## 执行摘要
## 1. [一级标题]
### 1.1 [二级标题]
### 1.2 [二级标题]
## 2. [一级标题]
...
## 参考文献
```

### 5. 引用格式（国自然基金）
- 正文：`[1]` 单篇 / `[1,3,5]` 多篇 / `[1-3]` 连续编号
- 文末按正文出现顺序排列
- 期刊论文：`[编号] 作者. 题目. 期刊, 年, 卷(期): 页码.`
- 网址：`[编号] 作者/机构. 标题. 来源, 日期. 访问日期. URL.`
- 内部文件：`[文件: xxx.json]`
- Agent结果：`[来源XX]`

### 6. 分析框架
[定义核心分析维度]

### 7. 格式要求
- 执行摘要前置（≤1页）
- 结论前置，详析后置
- 对比用表格，避免大段文字
- 不确定性标注：`[未公开]` / `[推断]` / `[待确认]`

### 8. 技能调用链
1. 读取工作空间相关文件（Glob 搜索 .md/.json/.csv/.xlsx/.pdf）
2. 使用 [技能] 查询 [数据]
3. WebSearch: [关键词A]、[关键词B]
4. WebFetch: [页面X]、[页面Y]
5. 启动 Agent 并行调研（3-5个话题，run_in_background: true）
6. md2docx_diy 转换交付
````

---

### 阶段四 → 用户确认

生成提示词后 **必须** 展示给用户：

```markdown
---
## 提示词已生成

[嵌入阶段三的完整提示词]

---

**请确认**：
1. 任务范围是否准确？
2. 输出结构是否符合预期？
3. 需要调整的地方？

回复 **"确认执行"** 或 **"开始执行"** 启动调研。
如需修改，告诉我具体调整点。
```

**⚠️ 等待用户明确确认后才进入阶段五，禁止自动执行。**

---

### 阶段五 → 执行调研

#### 5.1 创建任务

用 `TaskCreate` 创建 5 个任务：

```
Task 1: 读取工作空间相关文件
Task 2: 技能调用与数据获取
Task 3: Web搜索 + Agent 并行补充调研
Task 4: 撰写 Markdown 报告（国自然基金引文格式）
Task 5: md2docx_diy 转换 + 交付
```

#### 5.2 读取文件

```bash
# 搜索工作空间相关文件
find . -name "*.md" -o -name "*.json" -o -name "*.csv" -o -name "*.xlsx" 2>/dev/null | head -20
```

逐个读取搜索到的文件，优先读取与任务关键词匹配度最高的 5-10 个。

**JSON 容错**：
- 支持扁平 `{"key": "val"}` 和嵌套 `{"category": {"key": "val"}}` 两种结构
- 遇到 JSONDecodeError：中文文本中的 ASCII `"` → `「」`，`\'` → `'`

#### 5.3 Agent 并行调研

每个 Agent 聚焦一个信息维度，`run_in_background: true` 并行运行，`reasoning` 模型用于复杂分析。

通用配置模板：

| Agent | 维度 | 模型 | 搜索来源 |
|-------|------|------|---------|
| Agent 1 | 政策/法规 | reasoning | 官方网站、政府公告 |
| Agent 2 | 技术/标准 | reasoning | 标准组织、技术文档 |
| Agent 3 | 市场/竞争 | reasoning | 行业报告、新闻、数据库 |
| Agent 4 | 学术/研究 | reasoning | 学术数据库、会议论文 |
| Agent 5 | 案例/实践 | reasoning | 行业案例、白皮书 |

#### 5.4 直接搜索

Agent 运行期间同步进行：`WebSearch` 搜关键词 → `WebFetch` 抓权威页面。

JavaScript 渲染页面兜底：用存档页、PDF 版或第三方聚合平台。

---

### 阶段六 → 报告撰写与交付

#### 6.1 撰写 Markdown

- **执行摘要**前置（≤1页，结论速览）
- **结论前置**：每节开头放结论，详析后置
- **表格优先**：对比数据用表格
- **国自然基金引文**：正文 `[编号]`，文末按出现顺序排列
- **不确定性**：`[未公开]` / `[推断]` / `[待确认]`

参考文献格式速查：

```
期刊：  [1] 作者. 题目. 期刊, 年, 卷(期): 页码.
网址：  [2] 机构. 标题. 来源, 日期. 访问日期. URL.
书籍：  [3] 作者. 书名. 版次. 出版地: 出版社, 年.
内件：  [文件: xxx.json] 说明. WorkBuddy. 日期.
Agent： [来源XX] Agent X: 主题. 来源1/来源2. WorkBuddy. 日期.
```

#### 6.2 转换 DOCX

```python
from md2docx_diy import md_to_docx
md_to_docx('/path/to/report.md', '/path/to/report.docx', orientation='portrait')
```

#### 6.3 交付

同时调用 `deliver_attachments`（DOCX 在前，MD 在后）和 `open_result_view`（DOCX）。

#### 6.4 记忆写入

在 `.workbuddy/memory/YYYY-MM-DD.md` 追加记录（报告名称、核心发现 3-5 条、数据源数量、关键结论）。

---

## 技能依赖

| 类型 | 技能 | 说明 |
|------|------|------|
| **固定** | `md2docx_diy` | Markdown → DOCX 转换 |
| 动态 | `deep-research:research` | 大规模并行调研 |
| 动态 | `medical-research-toolkit` | 医学临床试验 |
| 动态 | `westock-data` / `neodata-financial-search` | 金融数据 |
| 动态 | `pdf` / `pdfkit-py` | PDF 处理 |
| 动态 | `web-scraper` / `playwright-scraper-skill` | 网页抓取 |
| 动态 | `xlsx` / `minimax-xlsx` | 表格处理 |
| 动态 | `pptx-generator` / `pptx` | PPT 生成 |
| 动态 | `cn2en_med_diy` / `en2cn_diy` | 医学翻译 |
| 动态 | `multi-search-engine` / `tencent-news` | 新闻搜索 |

动态依赖在阶段二中按需识别和安装。

---

## 路径约定

| 路径 | 用途 |
|------|------|
| `{WORKSPACE}/` | 根目录：报告输出、文件搜索起点 |
| `{WORKSPACE}/results/` | 调研数据（JSON/MD）存储 |
| `~/.workbuddy/skills/` | 已安装技能目录 |
| `~/.workbuddy/skills-marketplace/skills/` | 本地技能市场 |
