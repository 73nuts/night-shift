> **About this example:** This is a truncated version of a real Night Shift report (49KB original).
> The output language was set to Chinese for this particular run — Night Shift supports any language.
> Part One (three-repository research) is included in full. Part Two (SPEC methodology, ~25KB) is truncated.

---

# Harness 生态研究 + SPEC 方法论 -- 夜班报告

**日期**: 2026-04-02 | **状态**: 完成

---

## Executive Summary

1. **gstack** (61.7K stars) 是一个规模庞大的 Claude Code 配置系统（34个技能、每个技能高达113KB），其核心创新在于 template+resolver 自动注入机制和 ELI16 自适应上下文模式，而非中文博客所宣传的"6大哲学宪章"
2. **claw-code** (142K stars) 源于 Claude Code 源码泄露事件的 clean-room 重写，目前仅 30-40% 功能完成度，但揭示了 Claude Code 内部的三层记忆架构
3. **oh-my-codex** (10.4K stars) 是三者中技术上最成熟的，提供了 planning gate、completion gate、ambiguity scoring 等可直接借鉴的工作流编排模式
4. 中文博客对 gstack 的描述存在系统性夸大：7501行实为 20K+ 行，14个模块实为 34个，6大哲学中仅2个是全局性的
5. gstack 的 template+resolver 模式、REPO_MODE 上下文感知、auto-learnings JSONL 是三个最有价值的模式
6. oh-my-codex 的 planning gate（PRD + test spec 硬性前置）与 write-spec 技能高度对齐，但实现为硬约束而非建议性质
7. 生态宏观趋势：从单一 CLAUDE.md 向可组合技能系统 + hooks + worktree 隔离 + 跨模型路由演进
8. CLAUDE.md 最佳实践共识：保持 200-300 行以内，hooks 用于确定性行为，CLAUDE.md 用于建议性指导
9. **SPEC 方法论核心发现**: Martin Fowler 提出 "instruction curse" -- 规格越长，AI 对每条指令的遵守率越低。这直接反对"越详细越好"的传统假设
10. **Shape Up 的 appetite 机制**是所有方法中 kill criteria 实现最干净的：固定时间 + 可变范围，不是估算而是约束
11. **Codex CLI 的 ExecPlan** 是生态中最有结构创新性的规格模式：agent 在执行过程中更新 plan，产出决策审计轨迹
12. **当前 write-spec 的核心差距**: 缺少硬性门控机制、completion gate、ambiguity scoring、任务类型分化、和执行中的 living document 更新
13. **推荐的混合规格结构**: 融合 Shape Up (appetite/rabbit holes) + Google Design Doc (goals/non-goals) + ADR (confirmation) + BDD (Given/When/Then) + 独有元素 (hypothesis/kill criteria/foundation check)

---

## PART ONE: 三仓库深度研究

---

### 1. gstack (garrytan/gstack)

#### 1.1 是什么

**基本信息** [researched]
- Stars: 61,744 | Forks: 8,200 | Issues: 349
- 创建: 2026-03-11 | License: MIT
- 作者: Garry Tan (Y Combinator CEO)
- 定位: "23 opinionated tools that serve as CEO, Designer, Eng Manager, Release Manager, Doc Engineer, and QA"

**规模**
- 34个技能目录，每个包含 SKILL.md + SKILL.md.tmpl
- 最大技能: ship (113KB), plan-ceo-review (108KB), office-hours (93KB)
- 根级文档: SKILL.md (33KB), TODOS.md (42KB), CLAUDE.md (21KB), ARCHITECTURE.md (22KB)
- CHANGELOG: 175KB
- 总提示词内容: 约 22,000-27,000 行（远超博客声称的 7,501 行）
- 511个测试文件
- 编译后二进制: 61MB (Mach-O arm64 only)

**文件结构核心**

```
gstack/
  ETHOS.md          -- 三大哲学支柱 (7.5KB)
  CLAUDE.md         -- 开发者贡献指南 (21KB)
  ARCHITECTURE.md   -- 技术架构文档 (22KB)
  SKILL.md          -- 根级技能文档 (33KB)
  TODOS.md          -- 结构化待办 (42KB)
  AGENTS.md         -- 代理定义 (2.6KB)
  scripts/
    resolvers/
      preamble.ts   -- 注入引擎 (34.9KB) <-- 核心机制
      review.ts     -- 评审解析器 (51.6KB)
      design.ts     -- 设计解析器 (46.7KB)
      testing.ts    -- 测试解析器 (27.9KB)
  [34个技能目录]/
    SKILL.md        -- 生成的最终文档
    SKILL.md.tmpl   -- 人工维护的模板
  bin/
    gstack-global-discover  -- 编译后工具 (61MB)
  extension/        -- Chrome 扩展
```

#### 1.2 设计哲学

gstack 有三个明确声明的哲学支柱（来自 ETHOS.md），加上若干分散在不同文件中的操作原则：

**支柱 1: Boil the Lake -- 完整性原则** [researched]

> "AI-assisted coding makes the marginal cost of completeness near-zero. When the complete implementation costs minutes more than the shortcut -- do the complete thing. Every time."

> "Lake vs. ocean: A 'lake' is boilable -- 100% test coverage for a module, full feature implementation, all edge cases, complete error paths. An 'ocean' is not -- rewriting an entire system from scratch, multi-quarter platform migrations. Boil lakes. Flag oceans as out of scope."

反模式列表包括：
- "Choose B -- it covers 90% with less code." （如果 A 只多 70 行，选 A）
- "Let's defer tests to a follow-up PR." （测试是最容易 boil 的 lake）
- "This would take 2 weeks." （应该说："2 weeks human / ~1 hour AI-assisted"）

**支柱 2: Search Before Building -- 三层知识** [researched]

> "Layer 1: Tried and true. [...] Layer 2: New and popular. [...] Layer 3: First principles. Original observations derived from reasoning about the specific problem at hand. These are the most valuable of all."

**支柱 3: User Sovereignty -- 用户主权** [researched]

> "AI models recommend. Users decide. This is the one rule that overrides all others."
> "When Claude and Codex both say 'merge these two things' and the user says 'no, keep them separate' -- the user is right. Always."

**操作原则（分散在不同文件中）：**

**ELI16 -- "16岁少年语言"** (preamble.ts) [researched]

AskUserQuestion 格式要求：
> "Simplify: Explain the problem in plain English a smart 16-year-old could follow. No raw function names, no internal jargon, no implementation details."

且有自适应升级机制：当检测到 3+ 个活跃 session 时（通过 `find ~/.gstack/sessions -mmin -120 -type f | wc -l`），自动进入 ELI16 模式，增加上下文重新定位步骤。

**Error Messages for AI** (ARCHITECTURE.md) [researched]

> "Errors are for AI agents, not humans. Every error message must be actionable"

作用范围：仅限 browse daemon 错误，并非全局原则。

**Let It Crash** (ARCHITECTURE.md) [researched]

> "The server doesn't try to self-heal. If Chromium crashes, the server exits immediately. The CLI detects the dead server on the next command and auto-restarts."

作用范围：仅限 browse daemon 崩溃恢复策略，并非通用哲学。

#### 1.3 核心机制：Template + Resolver 自动注入

这是 gstack 最值得关注的工程机制：

1. 每个技能有一个 `SKILL.md.tmpl` 模板（人工编辑）
2. `scripts/gen-skill-docs.ts` 使用 TypeScript resolver 填充占位符
3. `{{PREAMBLE}}` 从 `preamble.ts` 生成，包含：session tracking, ELI16 模式触发, AskUserQuestion 格式, voice guidelines, 完整性原则
4. CI 运行 `gen:skill-docs --dry-run && git diff --exit-code` 防止生成文件过时
5. 分级注入 (T1-T4)：不是所有技能需要同等 overhead

#### 1.4 博客声明验证表

| 博客声明 | 实际发现 | 验证方法 | 判定 |
|---------|---------|---------|------|
| 7501 行提示词 | 22,000-27,000+ 行 | GitHub API 文件大小统计 | **误导** -- 实际规模是声称的 3-4 倍 |
| 14 个 AI 技能模块 | 34 个 SKILL.md 目录，README 列出 23 个 | 文件树遍历 | **不准确** -- 可能是早期版本快照 |
| "煮沸湖水"是全局哲学宪章 | 确认，ETHOS.md 一级原则 | 原文引用 | **准确** |
| 报错是给机器看的 | 仅限 browse daemon 错误 | ARCHITECTURE.md | **夸大** -- 局部设计决策被描述为全局原则 |
| Let It Crash 不自修复 | 仅限 browse server 崩溃 | ARCHITECTURE.md | **夸大** -- 同上 |
| 延迟工作必须进 TODOS.md | review/ship 技能的交叉引用 | review/SKILL.md | **准确但片面** |
| "做减法"全局哲学 | 仅在设计/视觉上下文 | office-hours/SKILL.md | **夸大** -- Dieter Rams 美学原则，非架构哲学 |
| 16岁少年语言 | 确认，全局注入的 AskUserQuestion 标准 | preamble.ts | **准确** |
| "隐藏在代码底部" | 通过 preamble 注入在顶部 | ARCHITECTURE.md | **不准确** -- 是 preamble（前置），非 postscript |
| "自动注入到每个 AI 任务" | template + resolver 机制确认 | gen-skill-docs.ts | **准确** |

---

### 2. claw-code (ultraworkers/claw-code)

#### 2.1 是什么

**基本信息** [researched]
- Stars: 142,089 | Forks: 101,523 | Issues: 1,413
- 创建: 2026-03-31（仅 2 天前） | License: 无
- 主语言: Rust
- 作者: Sigrid Jin (@instructkr)，韩裔加拿大开发者

**起源事件**: 2026 年 3 月 31 日，Anthropic 在 npm 推送 Claude Code v2.1.88 时意外包含了 59.8MB 的 source map，R2 bucket 也公开了完整源码。总暴露量：512,000 行 TypeScript，1,906 个文件。Sigrid Jin 凌晨 4 点研究暴露的代码架构后，使用 oh-my-codex 编排，在日出前完成了 clean-room Python 重写，随后完成 Rust 移植。

**泄露揭示的 Claude Code 内部架构**：
- ~40 个独立的、权限门控的工具
- **三层记忆架构**: 轻量索引始终在 context 中 / 主题文件按需加载 / 原始 transcript 仅搜索时查询
- 44 个 feature flags
- 未发布功能: KAIROS (后台自主 session), ULTRAPLAN (30分钟远程规划), BUDDY (宠物伴侣)
- `undercover.ts` 模块 -- 隐藏 Claude Code 参与 AI 生成贡献的机制
- 内部模型代号 "Capybara" (Claude 4.6 变体)

**双轨实现**:
- Python 轨道 (`src/`): ~35 个文件，初始 clean-room 移植
- Rust 轨道 (`rust/`): 8 个 crate -- api, runtime, tools, commands, plugins, claw-cli, compat-harness, lsp
- `reference_data/`: 原始 TS 系统的 JSON 快照（40K command surface, 36K tool surface），作为 parity target

**功能完成度**: ~30-40%
- Core conversation/tool loop: ~80%
- CLI command surface: ~30%
- Plugin/hook/skill ecosystem: ~5%
- Service layer: ~40%

#### 2.2 设计哲学

claw-code 的核心理念是**架构透明性和开放性**：
- harness 层（任务分解、工具调用链、context 管理、agent 行为观察）通常是私有和不透明的，claw-code 使其可检查和可扩展
- 多 provider 支持（Anthropic, OpenAI-compat, Grok）
- 无 `max_iterations` 限制
- 默认 `DangerFullAccess` -- 移除权限摩擦

#### 2.3 社区反响

- GitHub 历史上最快达到 100K stars 的仓库
- Elon Musk 公开关注此事件
- Bloomberg, CNBC, Gizmodo, BleepingComputer, Yahoo Finance 等均有报道
- HN 讨论聚焦于泄露源码中的 "Undercover Mode" -- 隐藏 AI 参与的机制引发信任危机
- 社区呼声：Claude Code 应从一开始就开源（Gemini CLI 和 Codex CLI 已经是）

---

### 3. oh-my-codex (Yeachan-Heo/oh-my-codex)

#### 3.1 是什么

**基本信息** [researched]
- Stars: 10,405 | Forks: 980 | Releases: 76
- 创建: 2026-02-02 | License: MIT
- 最新版: v0.11.12 (2026-04-02)
- 主语言: TypeScript (91.5%), Rust (4.7%)
- 姊妹项目: oh-my-claudecode (21.8K stars) -- Claude Code 版本

**定位**: Codex CLI 的工作流编排层。不替代 Codex，而是扩展它：agent teams, hooks, persistent state, structured pipelines, tmux-based parallelism。

**规模**: ~200+ 文件的 monorepo
- 50+ 技能目录 (各含 SKILL.md)
- 30+ prompt 模板
- 14 个 evaluation missions
- Rust crate: omx-runtime-core, omx-sparkshell, omx-mux, omx-explore, omx-runtime
- 11 语言 README 本地化

#### 3.2 设计哲学

**核心工作流 pipeline**: [researched]
1. `$deep-interview` -- 最多 20 轮苏格拉底式提问，ambiguity scoring，必须明确 non-goals
2. `$ralplan` -- 共识规划，架构验证，产出 PRD + test spec
3. `$team` (并行) 或 `$ralph` (顺序持久化循环) -- 执行
4. `$ultraqa` -- 最多 5 轮 QA 循环，自动修复，相同错误重现则停止

**Planning Gate（硬性前置门）**: [researched]
- PRD (`prd-*.md`) + test spec (`test-spec-*.md`) 两者都必须存在才能开始实现
- Ralph planning gate enforcement 机制会阻止提前开始
- 这不是建议，是硬约束

**Completion Gate**: [researched]
- no pending work + features working + tests passing + zero known errors + verification evidence collected
- 形式化的完成标准，而非模糊的 "done"

**Ambiguity Scoring**: [researched]
- Quick: <=0.30, Standard: <=0.20, Deep: <=0.15
- 数值阈值声明 spec "crystallized"，强制 agent 明确残余不确定性

**Agent Tier 系统**:
- LOW / STANDARD / THOROUGH -- 推理投入分级
- frontier-orchestrator / deep-worker / fast-lane -- 三种姿态
- 模型路由通过环境变量，无硬编码模型名

**状态持久化** (`.omx/` 目录):
```
.omx/
  state/       -- 模式状态 JSON
  plans/       -- PRD + test spec 产物
  notepad.md   -- session 笔记
  logs/        -- hooks-YYYY-MM-DD.jsonl
```

#### 3.3 社区反响

- oh-my-codex: 10.4K stars, 980 forks, 2 个月
- oh-my-claudecode: 21.8K stars -- Claude Code 用户群更大或更活跃
- Release cadence: 几乎每日发布
- 关键信誉加持: claw-code 的 clean-room 重写是用 oh-my-codex 编排完成的 -- "它字面上构建了 GitHub 历史上增长最快的仓库"

---

### 4. 三仓库交叉对比

| 维度 | gstack | claw-code | oh-my-codex |
|------|--------|-----------|-------------|
| Stars | 61.7K | 142K | 10.4K |
| 年龄 | 22 天 | 2 天 | 2 个月 |
| 成熟度 | 生产级（单平台） | 原型（30-40%） | 生产级 |
| 核心创新 | Template+resolver 自动注入 | 架构透明 + 多 provider | 工作流编排 + planning gate |
| 哲学 | Boil the Lake, ELI16 | 开放性 | "Weapon, not a tool" |
| 技能数 | 34 | N/A (stub) | 50+ |
| 技术栈 | TypeScript + Bun | Rust + Python | TypeScript + Rust |
| 状态持久化 | ~/.gstack/sessions + learnings | N/A | .omx/ 目录 |
| 多模型 | /codex 技能（Claude + Codex） | 多 provider (Anthropic/OpenAI/Grok) | 混合团队 (Codex + Claude + Gemini) |
| 安全机制 | /careful, /freeze hooks | DangerFullAccess 默认 | Worker isolation |
| 主要风险 | Context bloat (113KB 技能) | 法律灰区, 未成熟 | 复杂度开销 |

**最直接可行动的发现**:

| 发现 | 来源 | 行动建议 | 优先级 |
|------|------|---------|--------|
| Template + resolver 统一技能生成 | gstack | 为技能建立 shared preamble 模板 | P1 |
| ELI16 AskUserQuestion 标准 | gstack | 形式化 AskUserQuestion 格式写入 rules/ | P1 |
| Planning gate 硬约束 | oh-my-codex | write-spec 从建议升级为硬门控 | P1 |
| Completion gate 定义 | oh-my-codex | 写入 deploy-verifier agent 或 rules/ | P2 |
| REPO_MODE 上下文感知 | gstack | 检测 solo/collaborative 调整行为 | P2 |
| Auto-learnings JSONL | gstack | Session 结束自动记录发现 | P2 |
| Ambiguity scoring | oh-my-codex | 集成到 clarify-requirement 技能 | P3 |
| 三层记忆架构 | claw-code (泄露) | 审视 memory/ 是否需要 Layer 3 | P3 |
| Tiered preamble | gstack | 为技能分级减少 context bloat | P3 |

---

## PART TWO: SPEC 方法论研究

*[This section surveys 10+ specification methodologies (Google Design Doc, Amazon PR/FAQ, RFC/RFD, ADR/MADR, Shape Up, BDD, Codex ExecPlan, GitHub Spec Kit, Kiro, Cursor Rules, Windsurf) and produces a gap analysis + recommended hybrid spec template. ~25KB in the full report.]*

---

## 验证结果

| 声明 | 验证方法 | 结果 |
|------|---------|------|
| gstack 7501 行提示词 | GitHub API 文件大小统计 | 不准确 -- 实际 22K-27K+ 行 |
| gstack 14 个技能模块 | 文件树遍历 | 不准确 -- 实际 34 个 |
| gstack 6 大哲学宪章 | 原文逐条引用 | 部分准确 -- 仅 2 个全局性，4 个局部 |
| claw-code 142K stars | GitHub API | 确认 [researched] |
| claw-code 30-40% parity | PARITY.md 原文 | 确认 [researched] |
| oh-my-codex 10.4K stars | GitHub API | 确认 [researched] |
| oh-my-claudecode 21.8K stars | GitHub API | 确认 [researched] |
| Claude Code 三层记忆架构 | 泄露源码分析报道 | 确认 [researched] -- 多个独立来源交叉验证 |
| CLAUDE.md 应保持 200-300 行 | HumanLayer + Anthropic docs | 确认 [researched] |

## 来源

### gstack
- https://github.com/garrytan/gstack
- https://raw.githubusercontent.com/garrytan/gstack/main/ETHOS.md
- https://raw.githubusercontent.com/garrytan/gstack/main/ARCHITECTURE.md
- https://techcrunch.com/2026/03/17/why-garry-tans-claude-code-setup-has-gotten-so-much-love-and-hate/
- https://www.producthunt.com/products/gstack
- https://news.ycombinator.com/item?id=47355173

### claw-code
- https://github.com/ultraworkers/claw-code
- https://cybernews.com/tech/claude-code-leak-spawns-fastest-github-repo/
- https://www.bloomberg.com/news/articles/2026-04-01/anthropic-executive-blames-claude-code-leak-on-process-errors
- https://www.bleepingcomputer.com/news/artificial-intelligence/claude-code-source-code-accidentally-leaked-in-npm-package/

### oh-my-codex
- https://github.com/Yeachan-Heo/oh-my-codex
- https://github.com/yeachan-heo/oh-my-claudecode

### 生态与方法论 (30+ sources)
- https://www.humanlayer.dev/blog/writing-a-good-claude-md
- https://code.claude.com/docs/en/best-practices
- https://developers.openai.com/cookbook/articles/codex_exec_plans
- https://github.com/github/spec-kit
- https://kiro.dev/docs/specs/
- https://addyosmani.com/blog/good-spec/
- *[... and 25+ more in the full report]*

---

## Suggested First Action

Read the oh-my-codex planning gate implementation and extract the ambiguity scoring rubric. It's the single highest-leverage pattern to adopt — a 30-minute integration that prevents hours of wasted implementation on vague requirements.
