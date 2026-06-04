# InkOS 核心业务需求说明书

> 从现有 InkOS v1.1.0 代码库反向提取，供技术栈重构时参考。本文档描述「要做什么」而非「怎么做」。既有实现细节仅作佐证，核心契约不可变，具体技术方案可替换。

---

## 目录

1. [系统概述](#1-系统概述)
2. [交互模式](#2-交互模式)
3. [领域模型](#3-领域模型)
4. [建书流程](#4-建书流程)
5. [完整管线 write next](#5-完整管线-write-next)
6. [原子命令](#6-原子命令)
7. [Agent 体系](#7-agent-体系)
8. [审计体系](#8-审计体系)
9. [输入治理](#9-输入治理)
10. [真相文件系统](#10-真相文件系统)
11. [伏笔治理](#11-伏笔治理)
12. [记忆系统](#12-记忆系统)
13. [字数治理](#13-字数治理)
14. [同人/续写/导入](#14-同人续写导入)
15. [文风系统](#15-文风系统)
16. [AIGC 检测](#16-aigc-检测)
17. [导出](#17-导出)
18. [守护进程与调度器](#18-守护进程与调度器)
19. [通知系统](#19-通知系统)
20. [审核工作流](#20-审核工作流)
21. [配置系统](#21-配置系统)
22. [LLM 集成](#22-llm-集成)
23. [Studio Web UI](#23-studio-web-ui)
24. [题材管理](#24-题材管理)
25. [数据分析](#25-数据分析)
26. [诊断与运维](#26-诊断与运维)
27. [文件目录结构](#27-文件目录结构)
28. [非功能性需求](#28-非功能性需求)
29. [附录：关键接口契约](#附录关键接口契约)

---

## 1. 系统概述

### 1.1 定位
InkOS 是一个**自主小说写作 AI Agent 平台**，通过多 Agent 协作管线实现小说的全自动创作、审计与修订。支持中英文双语，覆盖玄幻、仙侠、都市、科幻等题材，支持续写、番外、同人、仿写。

### 1.2 核心设计原则
| 原则 | 说明 |
|------|------|
| **控制/执行分离** | 先编译创作意图（plan/compose），后执行写作（draft） |
| **审计后回写** | 每章须经审计，不通过则修订，关键问题清零才放行 |
| **真相文件唯一事实来源** | 7 大 truth files 作为跨章连续性的唯一权威 |
| **结构化状态** | v0.6 起以 JSON delta 驱动，Zod schema 校验，markdown 为人类可读投影 |
| **人工门控** | 关键问题暂停等人工，非关键问题自动处理 |
| **降级兼容** | SQLite 记忆需 Node 22+，旧版自动回退 Markdown 方案 |

---

## 2. 交互模式

系统提供三种交互模式，底层共享同一组原子操作：

### 2.1 完整管线（一键式）
```bash
inkos write next 书名          # 写 → 审 → 改，一步到位
inkos write next 书名 --count 5 # 连写 5 章
```
执行 Plant → Compose → Draft → Audit → Revise 完整链路。

### 2.2 原子命令（可组合）
```bash
inkos plan chapter 书名 --context "指导语" --json
inkos compose chapter 书名 --json
inkos draft 书名 --context "指导语" --json
inkos audit 书名 31 --json
inkos revise 书名 31 --json
```
每个命令独立执行单一操作，`--json` 输出结构化数据，供外部 Agent 或脚本编排。

### 2.3 自然语言 Agent 模式
```bash
inkos agent "帮我写一本都市修仙，主角是个程序员"
```
内置 18 个工具，LLM 通过 tool-use 自主决策调用顺序。推荐工作流：先调整控制面 → `plan`/`compose` → 决定写草稿还是跑完整管线。

#### 2.3.1 Agent 工具清单

| 工具名 | 功能 | 核心参数 |
|--------|------|---------|
| `write_draft` | 写下一章草稿，生成正文+更新真相文件 | bookId, guidance? |
| `write_full_pipeline` | 完整管线：写→审→改，一键完成 | bookId, count? |
| `plan_chapter` | 生成下一章 intent.md | bookId, guidance? |
| `compose_chapter` | 生成 context.json / rule-stack.yaml / trace.json | bookId, guidance? |
| `audit_chapter` | 审计指定章节 | bookId, chapterNumber? |
| `revise_chapter` | 修订章节（5 种模式） | bookId, chapterNumber?, mode? |
| `create_book` | 创建新书，含基础设定 | title, genre, platform, brief? |
| `list_books` | 列出所有书籍 | — |
| `get_book_status` | 书籍状态概览 | bookId |
| `read_truth_files` | 读取长期记忆文件 | bookId |
| `update_author_intent` | 更新 long-term 意图 | bookId, content |
| `update_current_focus` | 更新近期关注点 | bookId, content |
| `scan_market` | 扫描平台市场趋势 | — |
| `import_style` | 导入文风指纹 | bookId, referenceText |
| `import_chapters` | 导入已有章节，重建真相文件 | bookId, text, split? |
| `import_canon` | 从正传导入正典到番外书 | targetBookId, parentBookId |
| `web_fetch` | 抓取指定 URL | url, maxChars? |
| `write_truth_file` | 直接覆写某个真相文件 | bookId, section, path |

### 2.4 守护进程模式
```bash
inkos up     # 启动后台循环自动写章
inkos down   # 停止守护进程
```
按 cron 调度自动执行：写章周期（默认每 15 分钟）和雷达扫描周期（默认每 6 小时）。

---

## 3. 领域模型

### 3.1 Book（书籍）
```
BookConfig {
  id: string              # 唯一标识，建书时从 title 生成（如 "吞天魔帝" → "tuntianmodi"）
  title: string           # 书名
  platform: "tomato" | "feilu" | "qidian" | "other"
  genre: string           # 题材 ID，关联 genres/ 目录下的 YAML
  status: "incubating" | "outlining" | "active" | "paused" | "completed" | "dropped"
  targetChapters: int     # 目标总章数，默认 200
  chapterWordCount: int   # 每章目标字数，默认 3000
  language?: "zh" | "en"  # 不填则从题材配置继承
  createdAt: ISO8601
  updatedAt: ISO8601
  parentBookId?: string   # 系列/番外关联父书
  fanficMode?: "canon" | "au" | "ooc" | "cp"
}
```

### 3.2 Chapter（章节）
```
ChapterMeta {
  number: int             # 从 1 开始
  title: string
  status: ChapterStatus
  wordCount: int
  createdAt: ISO8601
  updatedAt: ISO8601
  auditIssues: string[]   # "[severity] description" 格式
  lengthWarnings: string[]
  reviewNote?: string
  detectionScore?: float  # 0-1
  detectionProvider?: string
  detectedAt?: ISO8601
  lengthTelemetry?: LengthTelemetry
  tokenUsage?: { promptTokens, completionTokens, totalTokens }
}
```

**Chapter 状态机（含过渡条件）：**

```
card-generated → drafting → drafted → auditing
                    ↑           ↓         ↓
                    |    revising ← audit-failed
                    |         ↓         ↓
                    |    state-degraded  ready-for-review
                    |                    ↓         ↓
                    |              approved    rejected
                    |                    ↓
                    └── write rewrite ──→ (回滚到快照后重新 drafting)

特殊：imported（从外部导入，无正常创作流程）
```

**状态说明**：
- `card-generated`：章节卡片已生成（仅占位）
- `drafting`：正在起草
- `drafted`：草稿完成，待审计
- `auditing`：正在审计
- `audit-passed`：审计通过
- `audit-failed`：审计未通过，需修订
- `state-degraded`：真相文件结算失败，降级保存（阻塞后续章节）
- `revising`：正在修订
- `ready-for-review`：待人工审阅
- `approved`/`rejected`：人工审阅结果
- `published`：已发布
- `imported`：从外部文本导入

### 3.3 GenreProfile（题材配置）
每题材一个 YAML 文件，含：
```yaml
name: string
id: string
language: "zh" | "en"
chapterTypes: string[]    # 该题材支持的章节类型
fatigueWords: string[]    # 高频疲劳词表
numericalSystem: bool     # 是否有数值体系
powerScaling: bool        # 是否有战力体系
eraResearch: bool         # 是否需要年代考据
pacingRule: string        # 节奏规则文本
satisfactionTypes: string[]  # 爽点类型列表
auditDimensions: int[]    # 该题材启用的审计维度 ID
```
YAML 后接自由文本（题材写作规则正文），供 Writer 和 Auditor 的 system prompt 使用。

### 3.4 BookRules（书籍级规则）
`book_rules.md`，YAML frontmatter + 自由文本：
```yaml
version: "1.0"
protagonist:
  name: string
  personalityLock: string[]
  behavioralConstraints: string[]
genreLock:
  primary: string
  forbidden: string[]
numericalSystemOverrides:
  hardCap: number | string
  resourceTypes: string[]
eraConstraints:
  enabled: bool
  period: string
  region: string
prohibitions: string[]            # 额外硬禁令
chapterTypesOverride: string[]    # 覆盖默认章节类型
fatigueWordsOverride: string[]    # 覆盖默认疲劳词
additionalAuditDimensions: int[]  # 额外审计维度
enableFullCastTracking: bool      # 全配角追踪
fanficMode: "canon" | "au" | "ooc" | "cp"
allowedDeviations: string[]
```

---

## 4. 建书流程

### 4.1 原子目录创建（Staging）
```
生成基础设定 → 写入 staging 目录 → FoundationReviewer 评审 → 通过后 rename 到正式目录
```
使用临时目录 `.tmp-book-create-<id>-<timestamp>-<rand>` 作为 staging，防止部分写入损坏书籍。失败时自动清理 staging 目录。

### 4.2 FoundationReviewer（基础设定审核）
- **5 维度百分制评审**：
  - 原创模式：核心冲突、开篇节奏、世界一致性、角色区分度、节奏可行性
  - 同人/系列模式：原作 DNA 保留、新叙事空间、核心冲突、开篇节奏、节奏可行性
- **通过阈值**：总分 ≥ 80，每个维度 ≥ 60
- **驳回自动重试**：最多 2 次，将具体审核意见反馈给 Architect 重生成
- **最终兜底**：达到最大重试次数后无论是否通过都接受

### 4.3 建书完整步骤
```
1. ArchitectAgent.generateFoundation → story_bible.md + volume_outline.md + book_rules.md + current_state.md + pending_hooks.md
2. FoundationReviewerAgent.review → 5 维度打分 → 不过则重试
3. 写入 book.json（BookConfig）
4. 写入 5 大基础文件（architect.writeFoundationFiles）
5. 初始化控制文档（author_intent.md + current_focus.md）
6. 初始化空章节索引 []
7. 创建第 0 章快照
8. 从 staging 目录 rename 到正式目录
```

### 4.4 创作简报
`--brief <file.md>` 传入脑洞/世界观/人设，Architect 基于简报生成设定而非凭空创作。简报同时落盘到 `author_intent.md`。

---

## 5. 完整管线 write next

### 5.1 主流程

```
0. 锁书（process-level file lock: .write.lock）
1. ensureControlDocuments（author_intent.md + current_focus.md 不存在则创建默认）
2. assertNoPendingChapterReview（有未审阅章节则拒绝写下一章）
3. prepareWriteInput（plan + compose，v2 模式）
4. buildLengthSpec（推导字数区间）
5. WriterAgent.writeChapter
   ├─ Phase 1: 创意写作（temp=0.7）
   ├─ Phase 2a: Observer 提取事实（temp=0.5）
   ├─ Phase 2b: Settler 回写真相文件（temp=0.3）
   └─ Post-write validation（零 LLM 成本：规则匹配+AI味检测）
6. runChapterReviewCycle
   ├─ 字数归一化（out of soft range）
   ├─ 审计（ContinuityAuditor）
   ├─ 若 audit 失败 → 修订 → 归一化 → 再审计（循环至关键问题清零或达到最大重试）
   └─ 内部决策：none | local-fix | rewrite
7. 标题去重（与已有章节标题比较，自动调整重复标题）
8. 长跨度疲劳检测（情绪单调/节奏单调/标题聚集/开头同构/结尾同构）
9. 真相文件校验（StateValidator：对比新旧 state/hooks）
   ├─ 若校验失败 → 仅重试 Settler 层（最多 1 次）
   ├─ 若仍失败 → 标记 state-degraded，阻塞后续章节
   └─ 若通过 → 正常状态
10. 段落形态检测（段落等长/段落长度漂移）
11. persistChapterArtifacts（落盘）
    ├─ saveChapter（章节 .md 文件）
    ├─ saveTruthFiles（state.md + hooks.md + ledger.md 等）
    ├─ saveChapterIndex（更新 book.json）
    ├─ markBookActiveIfNeeded（状态改为 active）
    ├─ persistAuditDriftGuidance（审计漂移指导）
    ├─ snapshotState（创建本章快照）
    └─ syncCurrentStateFactHistory（同步 SQLite 记忆索引）
12. 通知推送（Telegram/飞书/企业微信/Webhook）
13. Webhook 事件发射（pipeline-complete）
14. 解锁
```

### 5.2 审计修订循环（Chapter Review Cycle）

```
审计 → 若 repairDecision ≠ none 则：
  ├─ "local-fix": 按 issues 局部修复本章
  │   └─ 若修复后内容与原文相同且非 rewrite → 跳出循环
  ├─ "rewrite": 整章改写
  │   └─ 若改写后内容与原文相同 → 跳出循环
  ├─ 修复后归一化字数
  ├─ 再次审计（temp=0）
  ├─ 若 AI-tell 增加 → 回退到上一轮内容，跳出循环
  └─ 若问题减少 → 接受修订，继续循环
```

**关键约束**：
- 修订后内容为空 → 抛出异常（`assertChapterContentNotEmpty`）
- 修订后 blockingCount 未减少且 criticalCount 未减少 → 拒绝应用修订
- AI-tell 数量增加 → 回退原文

### 5.3 状态校验与降级（State Settlement Recovery）
当 `StateValidator` 检测到 Settler 输出与正文不一致时：
1. 触发 `retrySettlementAfterValidationFailure`
2. 仅重试 Settler 层（不重写正文），将 `validationFeedback` 注入 Settler prompt
3. 重试后再次 StateValidator 验证
4. 若仍失败 → 章节状态标记为 `state-degraded`，保存 review note 含降级原因
5. state-degraded 章节：不更新真相文件、不更新记忆索引、阻塞后续写章
6. `write repair-state` 手动修复降级章节

---

## 6. 原子命令

| 命令 | 功能 | 需锁 | 需 LLM | 核心行为 |
|------|------|------|--------|---------|
| `plan chapter` | 生成 intent.md | 否 | 是 | PlannerAgent.planChapter：读取控制面 + 记忆检索 → 产出 intent |
| `compose chapter` | 生成运行时产物 | 否 | 否（纯本地编译） | ComposerAgent.composeChapter：按相关性选择上下文 → 编译产物 |
| `draft` | 写草稿 | 是 | 是 | 同 write next 步骤 2-5，不含 audit/revise 循环 |
| `audit` | 审计章节 | 否 | 是 | 读取章节 → ContinuityAuditor → 更新索引 |
| `revise` | 修订章节 | 是 | 是 | 重新审计 → 修复 → 归一化 → 再审计 → 条件落盘 |
| `write rewrite` | 回滚重写 | 是 | 是 | 恢复指定章快照 → 重新执行完整管线 |
| `write repair-state` | 修复降级章节 | 是 | 是 | 仅重跑 Settler，不重写正文 |
| `consolidate` | 压缩章摘要 | 否 | 是 | ConsolidatorAgent：将已完成卷的章摘要压缩为卷级叙事摘要 |

---

## 7. Agent 体系

### 7.1 Agent 清单

| Agent | 类名 | 职责 | 默认 temp | 输入 | 输出 |
|-------|------|------|-----------|------|------|
| **Radar** | `RadarAgent` | 抓取平台排行榜，分析市场趋势 | 0.6 | 实时排行数据 | recommendations + marketSummary |
| **Planner** | `PlannerAgent` | 读控制面+记忆，生成本章意图 | 0.5 | author_intent + current_focus + memory | ChapterIntent |
| **Composer** | `ComposerAgent` | 从真相文件选择上下文，编译规则栈 | 0.3 | 全量真相文件 + ChapterIntent | ContextPackage + RuleStack + Trace |
| **Architect** | `ArchitectAgent` | 建书时生成基础设定 | 0.7 | 外部指导 + review feedback | 5 大基础文件 |
| **FoundationReviewer** | `FoundationReviewerAgent` | 独立评审基础设定 | 0.3 | Architect 输出 | FoundationReviewResult |
| **Writer** | `WriterAgent` | 两阶段：创作正文 + Observer+Settler | 0.7/0.3 | 真相文件(过滤后) + 控制面 | WriteChapterOutput |
| **LengthNormalizer** | `LengthNormalizerAgent` | 字数压缩/扩展 | 0.3 | 正文 + LengthSpec | 归一化正文 |
| **ContinuityAuditor** | `ContinuityAuditor` | 多维度审计 + AI 味检测 | 0.3 | 章节正文 + 真相文件 + 控制面 | AuditResult |
| **StateValidator** | `StateValidatorAgent` | 校验 Settler 输出一致性 | 0.3 | old/new state + 正文 | ValidationResult |
| **ChapterAnalyzer** | `ChapterAnalyzerAgent` | 导入章节时逆向工程真相文件 | 0.3 | 已有章节正文 + 标题 | 分析后的真相文件 |
| **Consolidator** | `ConsolidatorAgent` | 压缩章摘要为卷摘要 | 0.3 | chapter_summaries + volume_outline | volume_summaries |
| **FanficCanonImporter** | `FanficCanonImporterAgent` | 从原作素材提取结构化正典 | 0.3 | 原作文本 + 模式 | worldRules + characterProfiles + keyEvents + 等 |

### 7.2 Agent 公共契约
```
AgentContext { client, model, projectRoot, bookId?, logger?, onStreamProgress? }
BaseAgent {
  chat(messages, options?): LLMResponse      // 标准 LLM 调用
  chatWithSearch(messages, options?): LLMResponse  // 带搜索的 LLM 调用
}
```
- `chatWithSearch`：OpenAI 用原生 `web_search_options`；其他 Provider 用 Tavily API（`TAVILY_API_KEY`）搜索 + 注入 prompt
- Token 用量通过 `LLMResponse.usage` 统一追踪

### 7.3 多模型路由
- 全局默认模型：所有 Agent 共用
- Agent 覆盖：`inkos config set-model <agent> <model>`，可独立指定 provider / baseUrl / apiKeyEnv / stream
- 未配置的 Agent 自动回退全局模型
- 不同 Agent 走不同 Provider 时，自动创建独立 client（按 provider+baseUrl+apiFormat 组合缓存）

---

## 8. 审计体系

### 8.1 审计维度清单（37 个）

**通用维度（ID 1-27）**：

| ID | 名称 | 说明 |
|----|------|------|
| 1 | OOC检查 | 角色行为是否符合性格底色 |
| 2 | 时间线检查 | 事件顺序是否合理 |
| 3 | 设定冲突 | 世界设定是否自洽 |
| 4 | 战力崩坏 | 战力体系是否崩坏 |
| 5 | 数值检查 | 数值是否前后一致 |
| 6 | 伏笔检查 | 伏笔推进/回收是否合理 |
| 7 | 节奏检查 | 章节节奏是否符合题材 |
| 8 | 文风检查 | 是否符合注入的文风指纹 |
| 9 | 信息越界 | POV 角色是否知道不该知道的事 |
| 10 | 词汇疲劳 | 高频词/AI 标记词密度 |
| 11 | 利益链断裂 | 角色动机是否合理 |
| 12 | 年代考据 | 是否符合设定年代 |
| 13 | 配角降智 | 配角是否被不合理降智 |
| 14 | 配角工具人化 | 配角是否纯工具人 |
| 15 | 爽点虚化 | 爽点是否被稀释 |
| 16 | 台词失真 | 对话是否符合角色 |
| 17 | 流水账 | 是否变成流水账 |
| 18 | 知识库污染 | 是否引入不该有的现代知识 |
| 19 | 视角一致性 | 视角切换是否有过渡 |
| 20 | 段落等长 | 段落长度是否过于均匀 |
| 21 | 套话密度 | 套话/公式化表达密度 |
| 22 | 公式化转折 | 转折是否过于公式化 |
| 23 | 列表式结构 | 是否出现列表式叙述 |
| 24 | 支线停滞 | 支线是否长期无推进 |
| 25 | 弧线平坦 | 情感弧线是否平坦 |
| 26 | 节奏单调 | 近期章节类型是否过于单一 |
| 27 | 敏感词检查 | 是否含敏感内容 |

**衍生维度（ID 28-37，同人/番外/系列专属）**：

| ID | 名称 | 说明 |
|----|------|------|
| 28 | 正传事件冲突 | 番外是否冲突正传 |
| 29 | 未来信息泄露 | 是否泄露分歧点后信息 |
| 30 | 世界规则跨书一致性 | 系列中世界规则是否一致 |
| 31 | 番外伏笔隔离 | 番外是否越权回收正传伏笔 |
| 32 | 读者期待管理 | 是否管理读者期待 |
| 33 | 大纲偏离检测 | 是否偏离卷纲 |
| 34 | 角色还原度 | 同人角色还原度 |
| 35 | 世界规则遵守 | 同人世界规则遵守 |
| 36 | 关系动态 | 关系动态是否合理 |
| 37 | 正典事件一致性 | 同人正典事件一致性 |

### 8.2 题材维度选择
每个题材在 YAML 中声明 `auditDimensions: [1,2,3,6,7,...]`，Auditor 仅执行本题材启用的维度。额外维度通过 `bookRules.additionalAuditDimensions` 追加。

### 8.3 审计后 AI 味检测
`analyzeAITells()` —— 零 LLM 成本的规则检测：
- 段落长度均匀度（变异系数 < 0.15）
- 模糊词密度（似乎/可能/某种程度上）
- 转折词密度（然而/不过/与此同时）
- 同一前缀句密度（列表式结构）

### 8.4 审计结果
```
AuditResult {
  passed: boolean
  issues: AuditIssue[]     // severity: critical | warning | info
  summary: string
  tokenUsage?: { promptTokens, completionTokens, totalTokens }
}
AuditIssue { severity, category, description, suggestion }
```

### 8.5 后写校验（Post-write Validation）
在 Writer 完成后、Auditor 启动前，运行零 LLM 成本的确定性检查：
- 硬性禁令检测：禁止句式（"不是…而是…"）、报告术语（"核心动机"/"信息边界"）、说教词（"显然"/"毋庸置疑"）
- 惊喜标记词密度（仿佛/忽然/竟然/猛地/不禁/宛如）——每 3000 字超 1 次即 warning
- 元叙事模式（"到这里，算是…"/"接下来，就是…"）
- 全场震惊类集体反应（"全场一片寂静"/"众人纷纷震惊"）
- 跨章重复检测：与最近 5 章内容的段落级重复
- 段落长度漂移：当前章段落长度与近期偏差过大

### 8.6 审计漂移指导（Audit Drift Guidance）
每次审计后，critical + warning 级别的问题写入 `story/runtime/audit-drift-guidance.md`，供后续章 Auditor 参考——若同类问题在连续多章中出现，升级严重性。

---

## 9. 输入治理

### 9.1 四层控制文档
```
story/author_intent.md                      # L1: 长期作者意图（人可编辑）
story/current_focus.md                      # L2: 近期 1-3 章关注点（人可编辑）
story/runtime/chapter-XXXX.intent.md        # L3: Planner 生成的本章意图
story/runtime/chapter-XXXX.context.json     # L4: Composer 编译的上下文包
story/runtime/chapter-XXXX.rule-stack.yaml  # L4: Composer 编译的规则栈
story/runtime/chapter-XXXX.trace.json       # L4: Composer 编译的输入追踪
```

### 9.2 治理模式
- **v2（默认）**：走 Planner → Composer 完整链路
- **legacy**：仅传 externalContext 字符串，跳过 plan/compose（向后兼容）

### 9.3 Planner 职责
- 读取 author_intent + current_focus + 记忆检索结果（SQLite 或 markdown）
- 生成 `ChapterIntent`：
  - `goal`：本章目标
  - `mustKeep`：必须保留的事项
  - `mustAvoid`：必须避免的事项
  - `styleEmphasis`：风格强调项
  - `conflicts`：冲突及解决方式
  - `hookAgenda`：伏笔推进排班

### 9.4 Composer 职责
- 从全量真相文件按相关性选择上下文：
  - **hard sources**（必须包含）：story_bible、current_state、book_rules
  - **soft sources**（按需选择）：pending_hooks、chapter_summaries、subplots、emotional_arcs、character_matrix
- 编译 `RuleStack`：
  - `layers`：优先级层（global → book → arc → local）
  - `sections`：hard/soft/diagnostic 三类规则
  - `overrideEdges`：规则覆盖边
  - `activeOverrides`：当前章节启用的覆盖
- 编译 `ContextPackage`：selectedContext（含 source/reason/excerpt）
- 输出 `ChapterTrace`：记录 plannerInputs、composerInputs、selectedSources 用于调试

---

## 10. 真相文件系统

### 10.1 7 大真相文件

| 文件 | 用途 | 更新方式 |
|------|------|---------|
| `current_state.md` | 世界状态：角色位置、关系、信息、情感 | Settler JSON delta → markdown 投影 |
| `particle_ledger.md` | 资源账本：物品、金钱、物资 | Settler 更新 |
| `pending_hooks.md` | 未闭合伏笔：铺垫、承诺、冲突 | Settler JSON delta → markdown 投影 |
| `chapter_summaries.md` | 各章摘要表格：人物/事件/状态/伏笔/情绪 | Settler 追加行 |
| `subplot_board.md` | 支线进度：A/B/C 线状态 | Settler 更新 |
| `emotional_arcs.md` | 情感弧线：按角色追踪 | Settler 更新 |
| `character_matrix.md` | 角色交互矩阵：相遇记录、信息边界 | Settler 更新 |

### 10.2 双重存储（v0.6+）
- **权威来源**：`story/state/*.json`（结构化 JSON，Zod schema 校验）
  - `manifest.json`、`current_state.json`、`hooks.json`、`chapter_summaries.json`、`subplots.json`、`emotional_arcs.json`、`character_matrix.json`
- **人类可读投影**：`story/*.md`（markdown 表格）
- **迁移**：旧书首次运行时 `bootstrapStructuredStateFromMarkdown` 自动迁移

### 10.3 结构化状态 Schema

```
StateManifest { schemaVersion: 2, language, lastAppliedChapter, projectionVersion }

CurrentStateState {
  chapter: int,
  facts: CurrentStateFact[]
}
CurrentStateFact { subject, predicate, object, validFromChapter, validUntilChapter, sourceChapter }

HooksState {
  hooks: HookRecord[]
}
HookRecord { hookId, startChapter, type, status, lastAdvancedChapter, expectedPayoff, payoffTiming, notes }
HookStatus: "open" | "progressing" | "deferred" | "resolved"
HookPayoffTiming: "immediate" | "near-term" | "mid-arc" | "slow-burn" | "endgame"

ChapterSummariesState {
  rows: ChapterSummaryRow[]
}
ChapterSummaryRow { chapter, title, characters, events, stateChanges, hookActivity, mood, chapterType }

RuntimeStateDelta {
  chapter: int,
  currentStatePatch?: { currentLocation, protagonistState, currentGoal, currentConstraint, currentAlliances, currentConflict }
  hookOps: { upsert[], mention[], resolve[], defer[] }
  newHookCandidates: { type, expectedPayoff, payoffTiming, notes }[]
  chapterSummary?: ChapterSummaryRow
  subplotOps[], emotionalArcOps[], characterMatrixOps[]  // loose op records
  notes[]
}
```

### 10.4 StateManager 核心操作
```
loadBookConfig(id)        → BookConfig
saveBookConfig(id, cfg)
listBooks()               → string[] (仅含 valid book.json 的目录)
getNextChapterNumber(id)  → int (索引长度 + 1)
loadChapterIndex(id)      → ChapterMeta[]
saveChapterIndex(id, idx)
snapshotState(id, chNum)  → 复制 story/ 到 story/snapshots/chapter-XXXX/
acquireBookLock(id)       → 返回 release 函数，基于文件锁 (.write.lock)
ensureControlDocuments(id) → 创建 author_intent.md + current_focus.md（若不存在）
```

---

## 11. 伏笔治理

### 11.1 HookAgenda（Planner 生成）
```
HookAgenda {
  pressureMap: HookPressure[]   # 各伏笔压力评估
  mustAdvance: string[]         # 必须推进的伏笔 ID
  eligibleResolve: string[]     # 可以回收的伏笔 ID
  staleDebt: string[]           # 陈年债务（推进不足的伏笔）
  avoidNewHookFamilies: string[] # 避免新增伏笔的族类
}
```

**HookPressure 字段**：
- `movement`：quiet-hold | refresh | advance | partial-payoff | full-payoff
- `pressure`：low | medium | high | critical
- `phase`：opening | middle | late
- `reason`：fresh-promise | building-debt | stale-promise | ripe-payoff | overdue-payoff | long-arc-hold
- `blockSiblingHooks`：是否阻塞同族其他伏笔

### 11.2 HookOps（Settler 输出）
```
HookOps {
  upsert: HookRecord[]   # 新增/更新伏笔
  mention: string[]      # 仅提及（不算推进，防假推进）
  resolve: string[]      # 回收伏笔
  defer: string[]        # 推迟伏笔
}
```

### 11.3 伏笔健康分析
`analyzeHookHealth()` 在每章后检查：
- 回收率（resolved / total）
- 陈年债务（stale + 未推进的伏笔）
- 重复伏笔（同类型已存在 open/progressing 的伏笔）

---

## 12. 记忆系统

### 12.1 SQLite 时序记忆库
- 路径：`story/memory.db`
- 运行时：Node 22+ 原生 `node:sqlite`，自动启用 WAL 模式
- 低于 Node 22 自动回退到 markdown 方案（仍可正常工作）

#### 表结构
```
facts (
  id, subject, predicate, object,
  valid_from_chapter, valid_until_chapter, source_chapter, created_at
)
chapter_summaries (
  chapter, title, characters, events, state_changes,
  hook_activity, mood, chapter_type
)
hooks (
  hook_id, start_chapter, type, status,
  last_advanced_chapter, expected_payoff, payoff_timing, notes
)
```

#### 核心操作
- `addFact/ invalidateFact / getCurrentFacts`：时序事实管理
- `upsertChapterSummary / getSummariesForChapters`：摘要索引
- `upsertHook / getActiveHooks / getHookHistory`：伏笔索引
- 每次写完整章后重建叙事记忆索引

### 12.2 上下文过滤
Writer 读取真相文件时不全量注入，而是：
- **filterHooks**：仅注入活跃/近期相关的伏笔
- **filterSummaries**：最近 N 章 + 与本章相关的摘要
- **filterSubplots**：活跃支线
- **filterEmotionalArcs**：主要角色近期弧线
- **filterCharacterMatrix**：本章出场角色的关系
- **POV 过滤**：若卷纲指定本章 POV 角色，矩阵和钩子进一步过滤到该角色所知范围

---

## 13. 字数治理

### 13.1 LengthSpec
```
LengthSpec {
  target: int           # 目标字数（用户设定的 chapterWordCount 或 --words）
  softMin: int          # 软下限（≈ target × 0.85）
  softMax: int          # 软上限（≈ target × 1.15）
  hardMin: int          # 硬下限（≈ target × 0.5）
  hardMax: int          # 硬上限（≈ target × 1.5）
  countingMode: "zh_chars" | "en_words"
  normalizeMode: "expand" | "compress" | "none"
}
```

### 13.2 治理规则
- `--words` 指定的是目标字数，系统推导允许区间，不承诺逐字精确命中
- 正文越 soft range → 归一化器单 pass 修正
- **安全网**：归一化后字数 < 原字数 25% → 拒绝归一化，保留原文（防止毁章）
- 1 次纠偏后仍越 hard range → 照常保存，标记 length warning + telemetry
- 用户 `INKOS_LLM_MAX_TOKENS` 作为全局幻觉上限生效

### 13.3 LengthTelemetry（每章记录）
```
{
  target, softMin, softMax, hardMin, hardMax, countingMode,
  writerCount,          // 写手原始输出字数
  postWriterNormalizeCount,  // 审计前归一化后字数
  postReviseCount,      // 修订后字数
  finalCount,           // 最终保存字数
  normalizeApplied,     // 是否执行了归一化
  lengthWarning         // 是否有长度警告
}
```

---

## 14. 同人/续写/导入

### 14.1 同人创作（Fanfic）

**创建命令**：`fanfic init --from source.txt --mode <canon|au|ooc|cp>`

**四种模式**：
| 模式 | 含义 | 角色约束 | 世界规则 | 审计要求 |
|------|------|---------|---------|---------|
| canon | 正典延续 | 严格遵守性格底色 | 不可改 | 角色还原度+正典一致性 |
| au | 架空世界 | 保留性格 | 可改 | 新设定内洽性 |
| ooc | 性格重塑 | 允许偏离 | 可改 | 仅记录偏离，不判失败 |
| cp | CP 向 | 聚焦配对关系 | 可改 | 关系动态检查 |

**核心约束**：
- 强制要求**新时空设定**：必须设计原创分岔点和独立核心冲突，不允许复述原作剧情
- 自动生成 `fanfic_canon.md`（FanficCanonImporter 提取）：
  - world_rules、character_profiles（含语癖/口头禅/信息边界）
  - key_events（含对同人写作的约束程度）
  - power_system、writing_style
- 启用衍生维度审计（ID 28-37）
- 信息边界管控：防止泄露分歧点后的信息

### 14.2 续写/系列导入

**命令**：`import chapters --from path.txt [--split <pattern>] [--resume-from <n>]`

**两种导入模式**：
- `continuation`（默认）：接续原文，不新建时空
- `series`：同一世界观写独立新故事，需新建时空 → FoundationReviewer 评审（series 维度）

**导入流程**：
1. 首次运行（resumeFrom = 1）：
   - Architect.generateFoundationFromImport：从所有章节生成基础设定
   - FoundationReviewer 评审（series 模式）
   - 导出 `style_profile.json` + `style_guide.md`
   - 初始化空索引 + 第 0 章快照
2. 顺序回放（从 resumeFrom 开始）：
   - ChapterAnalyzerAgent.analyzeChapter：分析已有章节 → 提取真相文件更新
   - Writer.saveChapter/ saveNewTruthFiles：落盘章节 + 真相文件
   - 创建章节索引条目（status: "imported"）
3. 导入后无缝接续 `write next`

**章节分割**：
- 默认：`/第\d+章\s/`（中文）、`/Chapter\s+\d+/i`（英文）
- 自定义：`--split <regex>`
- 断点续导：`--resume-from <chapter>`

### 14.3 正传正典导入
**命令**：`import canon <targetBookId> --from <parentBookId>`
- 从父书的 7 大真相文件提取正典约束
- 生成 `story/parent_canon.md`
- 番外书启用完整正典审计

---

## 15. 文风系统

### 15.1 文风分析（Style Analysis）
`style analyze <file>` → 纯文本分析 + LLM 定性分析

**统计指纹**（零 LLM 成本，`analyzeStyle()`）：
- 句长分布（均值 + 标准差）
- 段落长度特征（均值/最小/最大）
- 词汇多样性（字符级 TTR）
- 高频句首模式（top 5）
- 修辞特征密度（比喻/排比/反问/夸张/拟人/短句节奏）

**LLM 定性分析**（`StyleAnalyzerAgent.generateStyleGuide()`）：
- 生成 `style_profile.json`（结构化指纹）
- 生成 `style_guide.md`（自然语言风格指南）

### 15.2 文风注入
`style import <file> [bookId]`：
- 将 `style_profile.json` + `style_guide.md` 写入指定书籍
- 后续所有章节 Writer 自动采用该风格
- Auditor 用风格标准做审计（维度 8）

### 15.3 对话指纹
- 从近期 5 章提取各角色的对话特征（语气、用词习惯、句式）
- 注入 Writer 上下文，维持对话一致性

---

## 16. AIGC 检测

### 16.1 检测配置
```
DetectionConfig {
  provider: "gptzero" | "originality" | "custom"
  apiUrl, apiKeyEnv, threshold: 0.5
  enabled: false
  autoRewrite: false
  maxRetries: 3
}
```

### 16.2 检测流程
- 调用外部 Detection API（GPTZero/Originality/Custom）
- 分数 0-1，越高越像 AI
- `detect [id] [n]`：检测指定章节
- `--all`：检测全部章节
- `--stats`：统计报告

### 16.3 自动重写循环
当 `autoRewrite: true` 且分数 > threshold：
```
检测 → revise(anti-detect mode) → 再检测 → ... → maxRetries 次或分数 < threshold
```
记录历史（`story/detection-history.json`），含每次尝试的分数。

---

## 17. 导出

### 17.1 格式
| 格式 | 描述 | 依赖 |
|------|------|------|
| `txt` | 纯文本，章节间空行分隔 | 无 |
| `md` | Markdown，书名 H1 | 无 |
| `epub` | EPUB 电子书 | marked + epub-gen-memory |

### 17.2 选项
- `--approved-only`：仅导出已批准章节
- `--output <path>`：指定输出路径
- `--json`：输出 JSON 结果
- 默认路径：`{projectRoot}/{bookId}_export.{format}`

---

## 18. 守护进程与调度器

### 18.1 调度配置
```
{
  schedule:
    radarCron: "0 */6 * * *"      # 雷达扫描周期
    writeCron: "*/15 * * * *"     # 写作周期
  maxConcurrentBooks: 3           # 每周期最多处理几本书（并行）
  chaptersPerCycle: 1             # 每周期每书写几章
  retryDelayMs: 30000
  cooldownAfterChapterMs: 10000
  maxChaptersPerDay: 50           # 每日上限（跨书合计）
  qualityGates: {
    maxAuditRetries: 2
    pauseAfterConsecutiveFailures: 3
    retryTemperatureStep: 0.1
  }
}
```

### 18.2 质量门控
- 连续失败计数器（per book）
- 失败维度聚类（per book → per dimension）
- 连续失败 ≥ pauseAfterConsecutiveFailures → 暂停该书
- 需人工 `resume`

### 18.3 调度策略
- 写章周期：筛选 status=active/outlining 的书，并行处理（最多 maxConcurrentBooks 本）
- 重叠保护：当前周期未完成时跳过新周期
- 雷达周期：独立于写章周期

---

## 19. 通知系统

### 19.1 渠道
| 渠道 | 所需配置 | 说明 |
|------|---------|------|
| Telegram | botToken + chatId | 发送纯文本消息 |
| 飞书 | webhookUrl | 富文本卡片 |
| 企业微信 | webhookUrl | Markdown 消息 |
| Webhook | url + secret? + events[] | HMAC-SHA256 签名，事件过滤 |

### 19.2 事件类型
- `pipeline-complete`：管线完成
- `pipeline-error`：管线错误
- `audit-passed` / `audit-failed`：审计结果
- `chapter-complete`：草稿完成（仅 draft）

### 19.3 通知内容（管线完成时）
```
标题：{✅/⚠️/🧯} {书名} 第{n}章
正文：
  **{章节标题}** | {字数}
  {📝 已自动修正}
  审稿: {通过/需人工审核}
  - [{severity}] {description}
  ...
```

### 19.4 失败策略
- 通知失败不阻塞管线
- 错误写入 stderr，不抛出异常

---

## 20. 审核工作流

### 20.1 人工审阅命令
| 命令 | 功能 |
|------|------|
| `review list [bookId]` | 列出待审阅章节 |
| `review approve-all [bookId]` | 批量通过所有待审阅章节 |
| `review reject [bookId] <n> [--reason]` | 拒绝指定章节 |

### 20.2 拒绝回滚
`review reject` 触发回滚：
- 恢复被拒章节之前的快照（StateManager.snapshotState 备份）
- 丢弃下游章节和记忆索引
- 回滚至快照状态

---

## 21. 配置系统

### 21.1 三层配置（优先级从高到低）
1. **Agent 模型覆盖**：`inkos config set-model <agent> <model>`
2. **项目级 .env**：`{projectRoot}/.env` → 覆盖 `inkos.json` 中的 LLM 设置
3. **全局配置**：`~/.inkos/.env`（`inkos config set-global` 写入）

### 21.2 环境变量
```
INKOS_LLM_PROVIDER=openai|anthropic|custom
INKOS_LLM_BASE_URL=
INKOS_LLM_API_KEY=
INKOS_LLM_MODEL=
INKOS_LLM_TEMPERATURE=0.7
INKOS_LLM_MAX_TOKENS=8192
INKOS_LLM_THINKING_BUDGET=0
INKOS_LLM_API_FORMAT=chat|responses
INKOS_LLM_HEADERS={"X-Custom":"value"}  # JSON 或单键值对
INKOS_LLM_EXTRA_<param>=<value>         # 额外 LLM 参数
INKOS_DEFAULT_LANGUAGE=zh|en            # 默认语言
TAVILY_API_KEY=                         # Tavily 搜索（非 OpenAI 时）
```

### 21.3 API Key 可选场景
- 当 baseUrl 指向 localhost / 127.0.0.1 / .local 域名 / 私有 IP 时，API Key 非必填

### 21.4 项目级配置 `inkos.json`
```json
{
  "name": "string",
  "version": "0.1.0",
  "language": "zh",
  "llm": { /* LLMConfig */ },
  "notify": [ /* NotifyChannel[] */ ],
  "detection": { /* DetectionConfig */ },
  "modelOverrides": { "writer": "claude-3-7" },
  "inputGovernanceMode": "v2",
  "daemon": { /* SchedulerConfig */ }
}
```

---

## 22. LLM 集成

### 22.1 支持的 Provider
| Provider | SDK | 特性 |
|----------|-----|------|
| OpenAI | openai SDK | 原生 web search、responses API、SSE stream |
| Anthropic | @anthropic-ai/sdk | thinking budget、SSE stream |
| Custom | openai SDK（复用） | 任何 OpenAI 兼容接口、自定义 headers |

### 22.2 流式降级
- 默认启用 SSE stream
- 中转站不支持 SSE 时自动降级为 sync 调用
- `PartialResponseError`：流中断但已生成 ≥ 500 字符 → 保留已生成内容

### 22.3 保留键保护
`max_tokens`、`temperature`、`model`、`messages`、`stream` 由 Provider 层管理，`llm.extra` 中的同名键自动过滤，防止意外覆盖。

### 22.4 Anthropic 特殊处理
- SDK 内部自动追加 `/v1/`，配置的 baseUrl 若含 `/v1/` 会自动剥离
- 支持 `thinkingBudget`（扩展思考预算）
- 支持 `apiFormat: "responses"`（Anthropic Responses API）

---

## 23. Studio Web UI

### 23.1 API 端点

**Books**
- `GET /api/books`：列出所有书籍摘要（含审阅状态统计）
- `GET /api/books/:id`：获取书籍详情 + 章节列表
- `POST /api/books/create`：异步创建新书，SSE 通知进度
- `GET /api/books/:id/create-status`：查询创建状态

**Chapters**
- `GET /api/books/:id/chapters`：章节列表
- `GET /api/books/:id/chapters/:number`：章节详情（含正文）
- `PUT /api/books/:id/chapters/:number`：保存章节编辑

**Truth Files**
- `GET /api/books/:id/truth-files`：真相文件列表
- `GET /api/books/:id/truth-files/:name`：真相文件内容
- `PUT /api/books/:id/truth-files/:name`：保存编辑

**Runs（管线执行）**
- `POST /api/books/:id/runs`：触发管线动作（draft/audit/revise/write-next）
- `GET /api/books/:id/runs/:runId`：查询状态
- `GET /api/books/:id/runs/:runId/stream`：SSE 实时流

**Review（人工审阅）**
- `POST /api/books/:id/chapters/:number/review`：审阅操作（approve/reject）

**Config**
- `GET /api/config`：项目配置
- `PUT /api/config`：更新配置

**其他**
- `GET /api/health`：健康检查（含 API 连通性）
- `GET /api/genres`：题材列表

### 23.2 SSE 事件
- `log`：管线日志（level/tag/message）
- `llm:progress`：LLM 流式进度（elapsed/totalChars/chineseChars）
- `book:creating` / `book:created` / `book:error`：建书进度
- `run:snapshot` / `run:status` / `run:stage`：运行状态推送

### 23.3 安全性
- Book ID 校验：`isSafeBookId` 防止路径穿越
- 统一错误响应格式：`{ error: { code, message } }`
- CORS 全开（`cors()`）

---

## 24. 题材管理

### 24.1 题材目录
```
project-root/genres/
  xuanhuan.yaml
  xianxia.yaml
  urban.yaml
  horror.yaml
  scifi.yaml
  other.yaml
  ...
```
每个 YAML 文件定义题材名称、ID、语言、章节类型、疲劳词、审计维度等。

### 24.2 题材命令
| 命令 | 功能 |
|------|------|
| `genre list` | 列出所有可用题材 |
| `genre show <id>` | 查看指定题材配置 |
| `genre copy <id> --to <new-id>` | 复制题材 |
| `genre create <id>` | 创建新题材模板 |

---

## 25. 数据分析

### 25.1 Analytics 指标（`computeAnalytics`）
- 审计通过率（per book / overall）
- 高频问题维度排行
- 章节质量排行（审计 issue 数量）
- Token 用量统计（prompt/completion/total）
- 字数偏差统计（target vs actual 偏差分布）
- 伏笔回收率（resolved / total）
- 平均修订次数

---

## 26. 诊断与运维

### 26.1 Doctor 命令
`inkos doctor`：诊断环境配置
- 检查 inkos.json 是否存在
- 检查 .env 加载状态
- API 连通性测试（发送小请求）
- Node.js 版本检查（SQLite 记忆可用性）
- Provider 兼容性提示

### 26.2 自我更新
`inkos update`：检查 npm registry 最新版本，下载并替换

### 26.3 评估命令
`inkos eval [id] [n]`：对已有章节质量做综合评估（AIGC 检测 + AI 味检测 + 文风一致性）

### 26.4 日志
- 写入 `inkos.log`（JSON Lines）
- 支持 `-q` 静默模式

---

## 27. 文件目录结构

```
project-root/
  inkos.json                    # 项目级配置
  .env                          # 项目级环境变量
  inkos.log                     # 运行日志（JSON Lines）
  books/
    <book-id>/
      .write.lock               # 进程级文件锁（运行时）
      book.json                 # BookConfig + ChapterMeta[]
      chapters/
        0001_标题.md
        0002_标题.md
        ...
      story/
        # 控制面
        author_intent.md         # 长期作者意图
        current_focus.md         # 近期关注点
        volume_outline.md        # 卷纲
        book_rules.md            # 书籍级规则（YAML frontmatter）
        story_bible.md           # 世界观设定
        # 真相文件（markdown 投影）
        current_state.md
        particle_ledger.md
        pending_hooks.md
        chapter_summaries.md
        subplot_board.md
        emotional_arcs.md
        character_matrix.md
        # 结构化状态（v0.6+ 权威来源）
        state/
          manifest.json
          current_state.json
          hooks.json
          chapter_summaries.json
          subplots.json
          emotional_arcs.json
          character_matrix.json
        # 衍生文件
        style_guide.md           # 文风指南
        style_profile.json       # 文风指纹
        parent_canon.md          # 系列/番外正典
        fanfic_canon.md          # 同人正典
        volume_summaries.md      # 卷级摘要（Consolidator）
        # 运行时产物
        runtime/
          chapter-0001.intent.md
          chapter-0001.context.json
          chapter-0001.rule-stack.yaml
          chapter-0001.trace.json
          audit-drift-guidance.md  # 审计漂移指导
      # 快照
      snapshots/
        chapter-0000/
          current_state.md
          pending_hooks.md
          particle_ledger.md
          chapter_summaries.md
          subplot_board.md
          emotional_arcs.md
          character_matrix.md
        chapter-0001/...
      memory.db                 # SQLite 时序记忆（Node 22+）
      memory.db-shm
      memory.db-wal
      detection-history.json    # AIGC 检测历史
  genres/
    xuanhuan.yaml
    xianxia.yaml
    ...
```

---

## 28. 非功能性需求

### 28.1 可靠性
- 每章自动快照 → `write rewrite` 可回滚
- 文件锁防止并发写入同一书
- 建书 staging 目录 + atomic rename 防部分写入
- JSON delta immutable apply + Zod 校验 → 坏数据直接拒绝
- Stream 中断时 PartialResponseError → 保留已生成内容

### 28.2 可扩展性
- Agent 模型路由：按 Agent 粒度独立配置模型/Provider/APIKey
- 题材插件化：新增题材只需添加 YAML + 正文
- 审计维度可扩展：题材级 + 书籍级额外维度

### 28.3 性能
- 上下文过滤：写手只注入相关片段，非全量真相文件
- SQLite 记忆索引：按 chapter 范围检索，替代 markdown 全文扫描
- Consolidator：长书自动压缩章摘要为卷摘要
- Agent client 缓存：按 provider+baseUrl+apiFormat 复用连接

### 28.4 兼容性
- Node.js ≥ 20
- SQLite 记忆需 Node 22+（自动降级 markdown 方案）
- 流式降级：不支持 SSE 时自动 sync
- 旧书自动迁移：markdown → JSON 结构化状态

### 28.5 国际化
- 所有用户可见字符串支持 `zh` / `en`
- CLI 输出、日志、提示词、真相文件全部双语
- 中英文写作支持各自的规范（AI 味检测词表、字数计数模式、段落特征）

---

## 附录：关键接口契约

### A. PipelineRunner 公共方法

```typescript
class PipelineRunner {
  // 完整管线
  writeNextChapter(bookId, wordCount?, tempOverride?): ChapterPipelineResult
  writeDraft(bookId, context?, wordCount?): DraftResult

  // 原子命令
  planChapter(bookId, context?): PlanChapterResult
  composeChapter(bookId, context?): ComposeChapterResult
  auditDraft(bookId, chapterNumber?): AuditResult & { chapterNumber }
  reviseDraft(bookId, chapterNumber?, mode?): ReviseResult
  repairChapterState(bookId, chapterNumber?): ChapterPipelineResult

  // 建书
  initBook(bookConfig): void

  // 导入
  importChapters(input): ImportChaptersResult
  importCanon(targetBookId, parentBookId): string
  importFanficCanon(bookId, sourceText, sourceName, fanficMode): string
  createFanficBook(bookConfig, sourceText, sourceName, fanficMode): BookConfig

  // 查询
  getBookStatus(bookId): BookStatusInfo
  readTruthFiles(bookId): TruthFiles
  listBooks(): string[]
  analyzeStyle(text, sourceName?): { profile, guide }

  // 雷达
  runRadar(): RadarResult
}
```

### B. 核心返回值契约

```typescript
ChapterPipelineResult {
  chapterNumber, title, wordCount,
  auditResult: AuditResult,
  revised: boolean,
  status: "ready-for-review" | "audit-failed" | "state-degraded",
  lengthWarnings?, lengthTelemetry?, tokenUsage?
}

DraftResult {
  chapterNumber, title, wordCount,
  filePath: string,
  lengthWarnings?, lengthTelemetry?, tokenUsage?
}

PlanChapterResult {
  bookId, chapterNumber, intentPath, goal, conflicts[]
}

ComposeChapterResult extends PlanChapterResult {
  contextPath, ruleStackPath, tracePath
}

ReviseResult {
  chapterNumber, wordCount, fixedIssues[],
  applied: boolean,
  status: "unchanged" | "ready-for-review" | "audit-failed",
  skippedReason?,
}
```

### C. Revise 命令的模式参数

| 模式 | 说明 |
|------|------|
| `local-fix` | 局部修复，最小改动（默认） |
| `rewrite` | 整章改写 |
| `polish` | 润色（不改剧情） |
| `rework` | 重做（可调整剧情） |
| `anti-detect` | 去 AI 味改写 |

---

*文档版本：v2.0 — 基于 InkOS v1.1.0 代码库全面反向提取并交叉验证*
