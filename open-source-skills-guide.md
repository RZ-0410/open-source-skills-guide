
# 开源 Skills 系统详解 / Open-Source Skills System In-Depth

> 一份面向开发者的可插拔 AI Agent 技能框架深度指南
> A deep-dive guide into the pluggable AI Agent skill framework for developers.

---

## 目录 / Table of Contents

1. [什么是 Skills / What Are Skills](#1-什么是-skills--what-are-skills)
2. [核心设计理念 / Core Design Philosophy](#2-核心设计理念--core-design-philosophy)
3. [架构概述 / Architecture Overview](#3-架构概述--architecture-overview)
4. [Skill 的结构规范 / Skill Structure Specification](#4-skill-的结构规范--skill-structure-specification)
5. [Skill 生命周期 / Skill Lifecycle](#5-skill-生命周期--skill-lifecycle)
6. [内置 Skills 一览 / Built-in Skills Overview](#6-内置-skills-一览--built-in-skills-overview)
7. [如何编写一个 Skill / How to Write a Skill](#7-如何编写一个-skill--how-to-write-a-skill)
8. [Skill 注册与分发 / Skill Registration & Distribution](#8-skill-注册与分发--skill-registration--distribution)
9. [安全与审核机制 / Security & Vetting Mechanism](#9-安全与审核机制--security--vetting-mechanism)
10. [实战案例 / Practical Examples](#10-实战案例--practical-examples)
11. [FAQ](#11-faq)

---

## 1. 什么是 Skills / What Are Skills

Skills 是 AI Agent 生态中的**可插拔专业指令模块**。每个 Skill 封装了特定领域的专家知识、操作流程和约束规则，Agent 在运行时按需加载，从而在不修改模型权重的前提下获得专业能力。

Skills are **pluggable expert instruction modules** in the AI Agent ecosystem. Each Skill encapsulates domain-specific expert knowledge, operational workflows, and constraint rules. The Agent loads them on demand at runtime, acquiring specialized capabilities without modifying model weights.

**一句话概括 / In one sentence**：Skills 之于 AI Agent，如同插件之于 IDE、扩展之于浏览器。
Skills are to AI Agents what plugins are to IDEs, and extensions are to browsers.

### 为什么需要 Skills / Why Skills Matter

| 痛点 / Pain Point | Skills 解决的方案 / How Skills Solve It |
|------|-------------------|
| 通用模型缺乏领域深度 / General models lack domain depth | 每个 Skill 由领域专家编写，嵌入最佳实践 / Each Skill is authored by domain experts with embedded best practices |
| 提示词膨胀，上下文超载 / Prompt bloat & context overload | 按需加载，不占用默认上下文窗口 / Loaded on demand, never consuming default context window |
| 任务执行质量不可控 / Uncontrollable execution quality | Skill 内置校验、安全检查、回退策略 / Built-in validation, security checks, and fallback strategies |
| 能力扩展依赖模型更新 / Capability expansion tied to model updates | 即插即用，无需等待新模型发布 / Plug-and-play, no model update required |
| 团队知识难以沉淀 / Team knowledge hard to institutionalize | Skill 即文档，可版本管理、可审查、可复用 / Skills are documentation — versionable, reviewable, reusable |

---

## 2. 核心设计理念 / Core Design Philosophy

### 2.1 专业分工 / Division of Labor

Skill 不追求大而全，每个 Skill 只做一件事并做到极致。例如 `frontend-slides` 只负责生成 HTML 演示文稿，不会涉足视频编辑。

Skills don't aim to be everything. Each Skill does one thing and does it extremely well. For instance, `frontend-slides` only generates HTML presentations — it won't touch video editing.

### 2.2 安全优先 / Security-First

所有 Skill 在执行前必须通过安全审查机制（`skill-vetter`）。Skill 声明所需的权限范围，Agent 在沙箱中执行，禁止越权访问系统敏感资源。

All Skills must pass the security vetting mechanism (`skill-vetter`) before execution. Each Skill declares its required permission scope; the Agent executes within a sandbox, prohibiting unauthorized access to sensitive system resources.

### 2.3 声明式指令 / Declarative Instructions

Skill 使用结构化自然语言描述，而非代码。这让非开发者（设计师、产品经理、领域专家）也能参与 Skill 编写，大幅降低贡献门槛。

Skills are written in structured natural language, not code. This allows non-developers — designers, product managers, domain experts — to contribute Skills, dramatically lowering the barrier to entry.

### 2.4 惰性加载 / Lazy Loading

Agent 仅在用户需求匹配时才加载对应 Skill。未使用的 Skill 不消耗任何上下文窗口，确保基础对话性能不受影响。

The Agent loads a Skill only when a user request matches it. Unused Skills consume zero context window, ensuring baseline conversation performance is never compromised.

### 2.5 可组合性 / Composability

多个 Skill 可以在同一任务中协同工作。例如"生成 PPT"的需求可能先调用 `brainstorming` 构思结构，再调用 `frontend-slides` 生成页面。

Multiple Skills can collaborate within a single task. For example, a "make a presentation" request might first invoke `brainstorming` to structure ideas, then `frontend-slides` to generate slides.

---

## 3. 架构概述 / Architecture Overview

```
┌──────────────────────────────────────────────────────────┐
│                      User Request                         │
└─────────────────────┬────────────────────────────────────┘
                      ▼
┌──────────────────────────────────────────────────────────┐
│                    Main Agent                             │
│   ┌──────────┐   Intent Parsing & Routing                 │
│   │ Router   │── Matches Skill vs Sub-Agent vs Tool       │
│   └──────────┘                                            │
│        │                                                  │
│        ▼                                                  │
│   use_skill(skill_name, task)                             │
│        │                                                  │
│        ▼                                                  │
│   ┌──────────────────────────────────────────┐            │
│   │           Skill Registry                  │            │
│   │  ┌─────────┐  ┌──────────┐  ┌─────────┐  │            │
│   │  │Skill A  │  │ Skill B  │  │ Skill C │  │            │
│   │  └─────────┘  └──────────┘  └─────────┘  │            │
│   └──────────────────────────────────────────┘            │
│        │                                                  │
│        ▼                                                  │
│   ┌──────────────────────────────────────────┐            │
│   │      Skill Execution Context              │            │
│   │  Load → Inject Context → Execute → Verify │            │
│   └──────────────────────────────────────────┘            │
│        │                                                  │
│        ▼                                                  │
│   ┌──────────────────────────────────────────┐            │
│   │            Tools & Sub-Agents              │            │
│   │  shell_executor / python_executor / ...   │            │
│   └──────────────────────────────────────────┘            │
└──────────────────────────────────────────────────────────┘
```

### 调度优先级 / Dispatch Priority

Agent 内部的任务调度遵循严格的优先级链：

The Agent's internal task dispatch follows a strict priority chain:

```
Sub-Agents  >  Skills  >  Tools  >  Generated Code Execution
```

Skills 位于第二层——当没有专用 Sub-Agent 能闭环处理任务时，Agent 会尝试匹配 Skill；如果 Skill 也无法胜任，才会降级到通用工具或手写代码。

Skills sit at tier two — when no dedicated Sub-Agent can handle a task end-to-end, the Agent attempts to match a Skill. Only if no Skill fits does it degrade to generic tools or hand-written code.

---

## 4. Skill 的结构规范 / Skill Structure Specification

一个标准 Skill 包含以下部分 / A standard Skill consists of the following parts:

### 4.1 元数据 / Metadata

```yaml
name: "frontend-slides"
version: "1.2.0"
author: "community"
description: "Create stunning HTML presentations from scratch"
tags: ["presentation", "html", "slides", "frontend"]
permissions:
  - file_write
  - shell_executor
```

| 字段 / Field | 说明 / Description |
|------|------|
| `name` | 唯一标识符，调用时使用 / Unique identifier, used during invocation |
| `version` | 语义化版本号 / Semantic version number |
| `author` | 作者或组织 / Author or organization |
| `description` | 一句话描述 Skill 的用途 / One-line description of the Skill's purpose |
| `tags` | 用于自动路由匹配的关键词 / Keywords for automatic routing and matching |
| `permissions` | 声明的权限范围 / Declared permission scope |

### 4.2 触发条件 / Trigger Rules

```markdown
## 触发条件 / Trigger Conditions
- 用户提到"生成 PPT"、"制作演示文稿"、"convert PPTX to HTML"
- 用户上传 .pptx 文件并请求转换为网页 / User uploads .pptx and requests web conversion
- 用户描述需要一个演讲用的幻灯片 / User describes needing slides for a talk
```

### 4.3 指令体 / Instructions

指令体是 Skill 的核心，使用 Markdown 格式的结构化自然语言：

The instruction body is the heart of a Skill, written in structured natural language using Markdown:

```markdown
## 执行流程 / Execution Flow

### Phase 1: 分析输入 / Analyze Input
1. 判断输入是新建还是转换 / Determine if input is new creation or conversion
2. 提取内容主题、目标受众、风格偏好 / Extract topic, target audience, style preference
3. 确定幻灯片数量（默认 8-12 页）/ Determine slide count (default 8-12)

### Phase 2: 构思结构 / Structure Outline
- 标题页：包含主标题、副标题、演讲者 / Title slide: main title, subtitle, speaker
- 目录页：3-5 个核心要点 / Agenda slide: 3-5 key points
- 内容页：每页一个核心论点 + 支撑数据 / Content slides: one core argument + supporting data each
- 总结页：行动号召或关键结论 / Summary slide: call to action or key takeaways

### Phase 3: 生成 HTML / Generate HTML
使用 reveal.js 或 impress.js 框架... / Use reveal.js or impress.js frameworks...
（详细模板和代码规范）/ (Detailed templates and coding standards)

### Phase 4: 验证 / Verification
- 检查所有资源链接可访问 / Verify all resource links are accessible
- 确认响应式布局正常 / Confirm responsive layout works correctly
- 验证过渡动画流畅 / Validate transition animations are smooth

## 安全约束 / Security Constraints
- 禁止生成包含恶意脚本的 HTML / Forbid HTML containing malicious scripts
- 所有外部资源必须使用 HTTPS / All external resources must use HTTPS
- 图片使用占位符或明确标注来源 / Use placeholders or clearly attribute image sources
```

### 4.4 输出规范 / Output Contract

每个 Skill 必须声明其输出格式，确保 Agent 能正确处理结果：

Every Skill must declare its output format so the Agent can handle results correctly:

```markdown
## 输出规范 / Output Specification
- 生成单个 .html 文件 / Generate a single .html file
- 文件命名：{主题}_slides_{日期}.html / File naming: {topic}_slides_{date}.html
- 写入结果产物目录 / Write to the output artifact directory
- 返回产物路径卡片 / Return artifact path card
```

---

## 5. Skill 生命周期 / Skill Lifecycle

```
  [Register] → [Vet] → [Ready] → [Invoke] → [Execute] → [Complete]
      │          │                 │          │           │
      │          │                 │          │           └─ Return result
      │          │                 │          └─ Error → [Fallback]
      │          │                 └─ No match → [Degrade to Tool]
      │          └─ Reject → [Dismiss]
      └──────────────────────────────────────────→ [Archive/Delete]
```

### 各阶段说明 / Stage Details

| 阶段 / Stage | 动作 / Action | 负责方 / Responsible |
|------|------|--------|
| 注册 / Register | Skill 提交到 Registry / Skill submitted to Registry | 开发者 / Developer |
| 审核 / Vet | `skill-vetter` 执行安全检查 / `skill-vetter` performs security checks | 系统自动 / System (auto) |
| 就绪 / Ready | 加入可用 Skill 列表 / Added to available Skill list | Agent Router |
| 调用 / Invoke | `use_skill(skill_name, task)` | Main Agent |
| 执行 / Execute | Skill 指令注入 LLM 上下文 / Skill instructions injected into LLM context | Agent Runtime |
| 完成 / Complete | 产出物交付，释放上下文 / Artifact delivered, context released | Agent Runtime |

---

## 6. 内置 Skills 一览 / Built-in Skills Overview

### 6.1 brainstorming

**创意构思助手 / Creative Brainstorming Assistant**。在开始任何创造性工作前必须调用。帮助探索用户意图、需求和设计方案。

Must be invoked before any creative work. Helps explore user intent, requirements, and design approaches.

**触发场景 / Trigger Scenarios**：创建功能、构建组件、添加功能、修改行为等创意任务 / Creating features, building components, adding functionality, modifying behavior.

### 6.2 frontend-slides

**前端演示文稿生成器 / Frontend Slides Generator**。从零生成动画丰富的 HTML 演示文稿，或从 PowerPoint 文件转换。

Generates animation-rich HTML presentations from scratch or converts PowerPoint files.

**技术栈 / Tech Stack**：reveal.js / impress.js，支持 3D 过渡、代码高亮、演讲者备注 / Supports 3D transitions, code highlighting, speaker notes.

### 6.3 huashu-nuwa (女娲造人 / Nuwa Creates Life)

**人物思维模型蒸馏器 / Persona Thinking Model Distiller**。输入人名或主题，自动执行深度调研 → 思维框架提炼 → 生成可运行的 Skill。

Input a name or topic; automatically performs deep research → thinking framework distillation → generates a runnable Skill.

**核心理念 / Core Concept**：将大师的思维方式封装为可复用的 AI Skill / Encapsulate masters' thinking patterns into reusable AI Skills.

### 6.4 elon-musk-perspective

**马斯克思维操作系统 / Musk Thinking OS**。基于传记、播客、推文、法庭证词等深度调研，提炼出 / Distilled from biographies, podcasts, tweets, court testimonies, and more:

- 5 个核心心智模型 / 5 core mental models
- 8 条决策启发式 / 8 decision heuristics
- 完整的表达 DNA / Complete expression DNA

**关键方法论 / Key Methodologies**：第一性原理 / First Principles、白痴指数 / Idiot Index、五步算法 / Five-Step Algorithm、垂直整合 / Vertical Integration.

### 6.5 humanizer

**AI 写作痕迹去除器 / AI Writing Trace Remover**。基于 Wikipedia 的"AI 写作特征"指南，检测并修复 / Based on Wikipedia's "Signs of AI Writing" guide, detects and fixes:

- 膨胀的象征主义 / Inflated symbolism
- 促销式语言 / Promotional language
- 肤浅的 -ing 分析 / Superficial -ing analyses
- 模糊归因 / Vague attributions
- 破折号滥用 / Em dash overuse
- 三连法则 / Rule of three

### 6.6 skill-vetter

**Skill 安全审查器 / Skill Security Vetter**。在安装任何来自社区 Hub、GitHub 或其他来源的 Skill 前使用。检查 / Use before installing any Skill from community Hubs, GitHub, or other sources. Checks for:

- 危险行为标记 / Red flags for dangerous behavior
- 权限范围合理性 / Permission scope reasonableness
- 可疑代码模式 / Suspicious code patterns

### 6.7 systems-paper-writing

**系统论文写作指南 / Systems Paper Writing Guide**。面向 OSDI、SOSP、ASPLOS、NSDI、EuroSys 顶会。提供 / Targeting top-tier systems conferences. Provides:

- 段落级结构蓝图 / Paragraph-level structural blueprints
- 写作模式与模板 / Writing patterns and templates
- 各会议检查清单 / Venue-specific checklists
- Reviewer 指南 / Reviewer guidelines
- LaTeX 模板 / LaTeX templates

---

## 7. 如何编写一个 Skill / How to Write a Skill

### Step 1: 明确边界 / Define Boundaries

问自己三个问题 / Ask yourself three questions:

1. 这个 Skill 解决什么具体问题？/ What specific problem does this Skill solve?
2. 输入是什么，输出是什么？/ What is the input and output?
3. 什么**不**属于它的职责范围？/ What is **not** within its scope?

### Step 2: 编写触发规则 / Write Trigger Rules

```markdown
## 触发条件 / Trigger Conditions
用户提到以下任意关键词或场景时触发 / Triggers when user mentions any of these keywords or scenarios:
- "图片批量压缩" / "batch image compress"
- "把照片缩小到 xx MB" / "shrink photos to xx MB"
- "优化图片大小" / "optimize image size"
```

触发规则应**宽松但不模糊**，避免漏匹配或误匹配。

Trigger rules should be **broad but not vague** — avoiding both missed matches and false positives.

### Step 3: 设计执行流程 / Design Execution Flow

用 Phase 分阶段描述，每个阶段有明确的输入、处理、输出。

Describe in phases; each phase has clear input, processing, and output.

```markdown
### Phase 1: 解析参数 / Parse Parameters
- 目标目录（必填）/ Target directory (required)
- 目标大小（默认 1MB）/ Target size (default 1MB)
- 输出格式（默认保持原格式）/ Output format (default: keep original)

### Phase 2: 遍历图片 / Scan Images
- 扫描目标目录下所有 .jpg/.png/.webp / Scan all .jpg/.png/.webp in target directory
- 按文件大小降序排列 / Sort by file size descending

### Phase 3: 压缩执行 / Execute Compression
- 使用 Pillow 或 ffmpeg 进行压缩 / Compress using Pillow or ffmpeg
- 超过目标大小的图片逐步降低质量参数 / Gradually reduce quality for oversized images
- 保持 EXIF 元数据（可选）/ Preserve EXIF metadata (optional)
```

### Step 4: 添加安全约束 / Add Security Constraints

```markdown
## 安全约束 / Security Constraints
- 🟡 中风险：会覆盖原文件，执行前需用户确认 / Medium risk: overwrites originals, user confirmation required
- 禁止删除原始文件（除非用户明确要求）/ Forbid deleting originals (unless user explicitly requests)
- 限制单次处理文件数 ≤ 500 / Limit batch size to ≤ 500 files
- 禁止修改系统目录下的图片 / Forbid modifying images in system directories
```

### Step 5: 声明输出 / Declare Output

```markdown
## 输出规范 / Output Specification
- 生成压缩报告（包含每个文件的原大小、新大小、压缩比）/ Generate compression report (original size, new size, ratio per file)
- 报告写入 `output/compression_report_{timestamp}.md` / Write report to `output/compression_report_{timestamp}.md`
```

### Step 6: 编写边缘案例处理 / Write Edge Case Handling

```markdown
## 边缘案例 / Edge Cases
- **超大文件（>100MB）/ Oversized files (>100MB)**：分块处理，显示进度 / Chunked processing, show progress
- **非图片文件 / Non-image files**：跳过并警告 / Skip and warn
- **已压缩文件 / Already compressed**：检测压缩率，<5% 增益则跳过 / Check compression ratio, skip if gain <5%
- **磁盘空间不足 / Insufficient disk space**：预检查可用空间，不足时中止 / Pre-check available space, abort if insufficient
```

### 完整模板 / Complete Template

```markdown
# Skill: {Name}

## 元数据 / Metadata
- name: "xxx"
- version: "1.0.0"
- author: "xxx"
- permissions: [xxx]

## 触发条件 / Trigger Conditions
{List of keywords and scenarios}

## 执行流程 / Execution Flow
### Phase 1: {Phase Name}
{Detailed instructions}

## 安全约束 / Security Constraints
{Risk levels and constraints}

## 边缘案例 / Edge Cases
{Exception handling}

## 输出规范 / Output Specification
{Artifact format and path}
```

---

## 8. Skill 注册与分发 / Skill Registration & Distribution

### 分发渠道 / Distribution Channels

| 渠道 / Channel | 适用场景 / Use Case | 审核要求 / Review Requirement |
|------|----------|----------|
| 官方内置 / Official Built-in | 通用高频场景 / Common high-frequency scenarios | 内部严格审核 / Strict internal review |
| Community Hub | 社区贡献 / Community contributions | skill-vetter 自动审查 + 社区评分 / Auto vetting + community rating |
| GitHub 仓库 / GitHub Repo | 个人/团队私有 / Personal/team private | 自行管理 / Self-managed |

### 注册流程 / Registration Flow

```
开发者 / Developer → 编写 Skill / Write Skill → 提交到 Registry / Submit to Registry
                                                   │
                                                   ▼
                                              skill-vetter 审查 / Review
                                               ├─ 通过 / Pass → 发布 / Publish
                                               └─ 拒绝 / Reject → 返回修改建议 / Return suggestions
                                                   │
                                                   ▼
                                              Agent Router 索引更新 / Index Update
```

### 版本管理 / Version Management

Skill 遵循 Semantic Versioning / Skills follow Semantic Versioning:

- **MAJOR**：不兼容的指令变更 / Incompatible instruction changes
- **MINOR**：向后兼容的功能新增 / Backward-compatible feature additions
- **PATCH**：修复、优化、安全补丁 / Fixes, optimizations, security patches

---

## 9. 安全与审核机制 / Security & Vetting Mechanism

### 三级风险定级 / Three-Tier Risk Classification

| 级别 / Level | 定义 / Definition | 示例 / Example |
|------|------|------|
| 🔴 高风险 / High | 不可逆的破坏性操作 / Irreversible destructive operations | `rm -rf`, `format`, registry modification |
| 🟡 中风险 / Medium | 可逆但有副作用的变更 / Reversible but side-effect changes | File overwrite, config modification, process termination |
| 🟢 低风险 / Low | 只读或无副作用 / Read-only or no side effects | Query, search, format conversion |

### skill-vetter 审查项 / Vetter Check Items

```yaml
checks:
  - name: "危险命令检测 / Dangerous Command Detection"
    pattern: ["rm -rf", "del /f", "format", "> /dev/sda"]
  - name: "权限越界检测 / Permission Overreach Detection"
    rule: "声明的权限不得超过所需范围 / Declared permissions must not exceed required scope"
  - name: "网络访问审查 / Network Access Review"
    rule: "所有外部请求必须声明目的和域名 / All external requests must declare purpose and domain"
  - name: "路径遍历防护 / Path Traversal Protection"
    pattern: ["../", "~/", "$HOME", "%APPDATA%"]
  - name: "代码注入检测 / Code Injection Detection"
    pattern: ["eval", "exec(", "subprocess", "os.system"]
```

### 执行沙箱 / Execution Sandbox

Skill 在受限的运行时环境中执行 / Skills execute within a restricted runtime environment:

- 文件操作仅限声明的工作目录 / File operations limited to declared working directory
- 网络访问需白名单审批 / Network access requires whitelist approval
- 系统命令需逐条审核 / System commands reviewed individually
- 超时限制：默认 30 分钟 / Timeout: 30 minutes default

---

## 10. 实战案例 / Practical Examples

### 案例一 / Example 1：批量图片压缩 Skill / Batch Image Compression Skill

```markdown
# Skill: image-batch-compress

## 元数据 / Metadata
- name: "image-batch-compress"
- version: "1.0.0"
- permissions: [file_read, file_write, python_executor]
- tags: ["image", "compress", "batch", "optimize"]

## 触发条件 / Trigger Conditions
- "压缩图片" / "compress images", "批量压缩" / "batch compress"
- "缩小图片体积" / "reduce image size", "优化图片" / "optimize images"

## 执行流程 / Execution Flow

### Phase 1: 收集参数 / Collect Parameters
确认目标目录、目标大小上限、是否保留原图。
Confirm target directory, size cap, whether to keep originals.

### Phase 2: 扫描 / Scan
使用 Python 的 pathlib 扫描目录，过滤出 .jpg/.png/.webp 文件。
Use Python pathlib to scan directory, filter .jpg/.png/.webp files.

### Phase 3: 压缩 / Compress
- JPEG: PIL 的 Image.save(quality=xx)，从 85 递减至达标 / PIL Image.save(quality=xx), decrement from 85 until target met
- PNG: 使用 pngquant 或 Pillow 的 optimize=True / Use pngquant or Pillow optimize=True
- WebP: PIL 的 quality 参数 / PIL quality parameter

### Phase 4: 报告 / Report
生成 Markdown 格式的压缩对比报告。
Generate Markdown compression comparison report.

## 边缘案例 / Edge Cases
- 单张图 > 500MB：跳过并警告 / Single image >500MB: skip and warn
- 压缩后反而变大：保留原图 / Compressed larger than original: keep original
```

### 案例二 / Example 2：Git 提交信息生成 Skill / Git Commit Message Generator

```markdown
# Skill: git-commit-gen

## 触发条件 / Trigger Conditions
- "写 commit message" / "write commit message"
- "生成提交信息" / "generate commit message"
- "Git commit"

## 执行流程 / Execution Flow
1. 执行 `git diff --staged` 获取变更内容 / Run `git diff --staged` to get changes
2. 分析 diff，提取：变更类型、影响范围、变更摘要 / Analyze diff: change type, scope, summary
3. 按 Conventional Commits 格式生成 / Generate in Conventional Commits format:
   - `feat:` 新功能 / new feature
   - `fix:` 修复 / bug fix
   - `refactor:` 重构 / refactoring
   - `docs:` 文档 / documentation
   - `chore:` 杂项 / miscellaneous
4. 提供 3 个候选，用户选择后执行 `git commit -m "xxx"` / Offer 3 candidates, execute after user selection

## 安全约束 / Security Constraints
- 🟡 中风险：会执行 git commit / Medium risk: will execute git commit
- 仅提交已 staged 的变更 / Only commit staged changes
- 不修改 git config / Do not modify git config
```

---

## 11. FAQ

### Q: Skill 和 Sub-Agent 的区别是什么？/ What's the difference between Skill and Sub-Agent?

| 维度 / Dimension | Skill | Sub-Agent |
|------|-------|-----------|
| 形态 / Form | 纯文本指令 / Plain-text instructions | 独立 Agent 实例 / Independent Agent instance |
| 上下文 / Context | 注入到当前 Agent / Injected into current Agent | 拥有独立上下文 / Has its own context |
| 工具访问 / Tool Access | 继承主 Agent 工具 / Inherits main Agent tools | 有专属工具集 / Has dedicated tool set |
| 适用场景 / Use Case | 专业性指导、流程规范 / Expert guidance, process standards | 跨域任务、长时间运行 / Cross-domain tasks, long-running |
| 开销 / Overhead | 低（仅文本注入）/ Low (text injection only) | 高（独立实例）/ High (separate instance) |

### Q: 一个 Skill 能调用另一个 Skill 吗？/ Can one Skill invoke another Skill?

可以，通过 Agent 中间层实现。Skill A 可以在指令中声明"完成后调用 Skill B 处理产物"，Agent 会自动编排。

Yes, via the Agent orchestration layer. Skill A can declare "after completion, invoke Skill B to process the artifact" in its instructions, and the Agent auto-orchestrates.

### Q: Skill 支持多语言吗？/ Does Skill support multiple languages?

Skill 指令体支持任意语言。推荐使用英语编写（方便国际化协作），触发条件可添加多语言关键词。

Skill instruction bodies support any language. English is recommended for global collaboration; trigger conditions can include multilingual keywords.

### Q: 如何调试 Skill？/ How to debug a Skill?

1. 使用 `skill-vetter` 先做静态检查 / Use `skill-vetter` for static analysis first
2. 在小范围场景下手动触发测试 / Manually trigger tests in limited scenarios
3. 观察 Agent 的执行日志 / Observe Agent execution logs
4. 使用 Git 管理版本，方便回退 / Version with Git for easy rollback

### Q: Skill 的执行有超时限制吗？/ Is there a timeout for Skill execution?

有。默认 30 分钟超时。如果 Skill 需要更长时间（如大规模数据处理），可以在元数据中声明 `timeout: 120m`。

Yes. Default timeout is 30 minutes. If a Skill requires more time (e.g., large-scale data processing), declare `timeout: 120m` in metadata.

---

## 贡献指南 / Contribution Guide

我们欢迎社区贡献 Skill！请遵循以下流程：

We welcome community Skill contributions! Please follow this process:

1. **Fork** 本仓库 / Fork this repository
2. 在 `skills/` 目录下创建你的 Skill 文件 / Create your Skill file under the `skills/` directory
3. 使用上述模板编写，确保包含完整的元数据和指令 / Use the template above; ensure complete metadata and instructions
4. 本地运行 `skill-vetter` 进行自查 / Run `skill-vetter` locally for self-review
5. 提交 **Pull Request** 并附上使用场景说明 / Submit a **Pull Request** with usage scenario description
6. 等待社区 Review / Wait for community review

### 编写规范 / Writing Guidelines

- 指令使用 Markdown 格式 / Instructions in Markdown format
- 明确标注风险等级（🔴/🟡/🟢）/ Clearly label risk levels (🔴/🟡/🟢)
- 包含至少 3 个边缘案例处理 / Include at least 3 edge case handlers
- 提供完整的输入/输出示例 / Provide complete input/output examples
- 注明所需的最低 Agent 版本 / Note the minimum required Agent version

---

## 许可证 / License

本项目采用 MIT 许可证。Skills 内容版权归原作者所有。

This project is licensed under the MIT License. Skill content copyright belongs to the original authors.

---

*文档版本 / Doc Version: 1.0.0 | 最后更新 / Last Updated: 2025-06-15 | 维护者 / Maintainer: Skills Community*
*（内容由AI生成，仅供参考）*
