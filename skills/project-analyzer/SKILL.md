---
name: project-analyzer
description: Analyze a software project and produce a structured markdown report whose outline adapts to the project's type (interface library, application, framework, protocol implementation, research code, monorepo, or a generic fallback) rather than forcing every project through the same template. Universal elements include a concepts/terminology glossary, README-vs-reality verification, code archaeology checks, license/distribution review, and an observed-facts/subjective-assessment split. Accepts a GitHub repo name (searches and clones), a local directory path, an uploaded archive (.zip/.tar.gz/.tar), or a submodule form `<repo-or-path>::<subpath>` for drilling into one subdirectory. Use this skill whenever the user wants to understand, explore, study, or summarize an existing project, codebase, or repository — even without the word "analyze." Strong trigger phrases include "了解这个项目", "看一下这个仓库", "整理一份项目报告", "分析 X repo", "study this repo", "understand this codebase", or simply naming a project and asking for a writeup. Reports are written in 简体中文 to `./reports/<name>_analysis_<depth>.md` with ASCII-only filenames; cross-platform (Windows PowerShell + *nix bash). Supports three depth levels (quick / medium / deep) and two writing styles (standard / concise) via parameter.
---

# Project Analyzer

Produces a structured markdown analysis report of a software project. **The outline is chosen based on what kind of project this is**, not pre-decided by template. An interface library and a CLI application demand fundamentally different framings; this skill picks one of seven presets (or a fallback) so the report's structure serves the project, not the other way around.

## Inputs to collect

1. **Source** (required — must ask if missing) — one of:
   - GitHub repo name (full `owner/repo` or partial like `langchain`)
   - Local directory path (already on disk)
   - Path to an uploaded archive (`.zip`, `.tar.gz`, `.tar`)
   - **Submodule form**: `<repo-or-path>::<subpath>` — analyzes a single subdirectory in the context of its parent repo. Examples: `./.cache/goclaw::internal/pipeline`, `nextlevelbuilder/goclaw::internal/pipeline`. See "Submodule mode" below for behavior differences.
2. **Depth** (optional, default `medium`): `quick` | `medium` | `deep`
3. **Style** (optional, default `standard`): `standard` | `concise`

For Depth / Style: if the user didn't specify, **use the defaults and state them explicitly** before starting ("I'll do a medium-depth standard-style analysis — say so if you want different."). Don't block on confirmation for defaults. Only Source must be confirmed.

## Workflow

### Step 1 — Resolve source to a local directory

All clone / extract targets go into **`./.cache/<name>/`** under the current working directory. Create `./.cache/` if it doesn't exist. This keeps the workspace clean across platforms (no `/home/claude`, no `%TEMP%`).

**GitHub repo name:**
- Use `gh search repos <query>` if the `gh` CLI is available, otherwise use `web_search` with a query like `site:github.com <query>`
- If multiple plausible matches exist, list the **top 3 by star count** with a one-line description each, and ask the user to pick. Do NOT pick silently.
- Once confirmed, clone with shallow depth (same command works on bash and PowerShell):
  ```bash
  git clone --depth 1 https://github.com/<owner>/<repo>.git ./.cache/<repo>
  ```
- For private repos: ask the user to clone manually and provide the local path.

**Local directory path:** use as-is.

**Submodule form (`<repo>::<subpath>`):**
- Resolve the `<repo>` part first via the rules above (clone / use local path / extract archive).
- Verify `<subpath>` exists under the resolved repo. If not, ask the user to correct.
- Carry both the resolved repo root and the subpath forward — the analysis runs against the subpath, but you are allowed to read up to 1–2 levels above for helper / type context (don't go further unless necessary).

**Archive:** extract to `./.cache/<archive-basename>/`.

bash:
```bash
mkdir -p ./.cache/<basename>
# zip
unzip -q <archive> -d ./.cache/<basename>
# tar.gz / tar
tar -xzf <archive> -C ./.cache/<basename>
```

PowerShell:
```powershell
New-Item -ItemType Directory -Force -Path ./.cache/<basename> | Out-Null
# zip
Expand-Archive -Path <archive> -DestinationPath ./.cache/<basename>
# tar.gz / tar (Win10+ bundles tar)
tar -xzf <archive> -C ./.cache/<basename>
```

### Step 1.5 — Size check & gate

Run the LoC count against the **analysis target** — for submodule mode that's the subpath, not the whole repo. Submodule LoC usually stays well under threshold and the gate rarely triggers.

Use whichever shell is available:

bash:
```bash
find <dir> -name '*.go' -o -name '*.py' -o -name '*.js' -o -name '*.ts' \
       -o -name '*.rs' -o -name '*.java' -o -name '*.cpp' -o -name '*.c' \
       -o -name '*.rb' -o -name '*.php' \
  | grep -v -E '/(node_modules|vendor|\.git|target|build|dist)/' \
  | xargs wc -l 2>/dev/null | tail -1
```

PowerShell:
```powershell
Get-ChildItem -Path <dir> -Recurse -File `
    -Include *.go,*.py,*.js,*.ts,*.rs,*.java,*.cpp,*.c,*.rb,*.php `
  | Where-Object { $_.FullName -notmatch '[\\/](node_modules|vendor|\.git|target|build|dist)[\\/]' } `
  | Get-Content | Measure-Object -Line
```

**If total LoC > 100,000**, pause and warn the user:

> "项目体量较大（约 X 万行），按 `<depth>` 档分析会有以下影响：
> - **token 消耗显著增加**（可能挤占后续对话的上下文）
> - **分析时间变长**（需要读更多文件）
> - **可能漏掉细节**（即便 deep 档也无法完整覆盖）
>
> 建议：
> - 改用 `quick` 档（更省 token，覆盖关键面）
> - 或限定到子目录 / 子模块进行分析
> - 或继续当前 `<depth>` 档，接受上述 trade-off
>
> 你想怎么做？"

Wait for user confirmation before proceeding. Honor the user's choice.

If LoC ≤ 100,000, proceed silently.

### Step 2 — Identify project type

Before the main analysis, do a fast type pass: read README + top-level structure, then assign the project to one of seven types. **Different types get different outlines and different focus lists** — this is where the redesign earns its keep over a fixed template.

**Detection signals** (any 2+ → match; if multiple types match, use the priority order at the bottom):

| Type | Signals |
|---|---|
| **interface-library** (接口库 / 系统库) | 主要产物是 `.so` / `.a` / crate / wheel / gem，**无 main 二进制**或仅有 demo CLI；README 描述方式是"提供 X / Y / Z 能力"或"封装 syscall X"；公共 API 可独立列出（headers / `pub fn` / `__all__`）；直接依赖某底层机制（syscall、内核子系统、协议帧、设备节点）；测试以 API 调用为主 |
| **application** (应用 / 工具) | 顶层有 `main` 入口或显式二进制目标；CLI / daemon / TUI 行为；README 讲使用场景或子命令；issue 多讨论 e2e 行为 |
| **framework** (框架 / SDK) | 库形态但通过 trait / interface / abstract class 强加架构约束；明确暴露扩展点（middleware / plugin / hook）；用户代码需要"塞进"框架的生命周期 |
| **protocol** (协议实现) | 引用 RFC / draft 号；存在 wire format / conformance test；命名里含 protocol / spec / codec / wire / frame |
| **research** (研究代码) | 引用论文 arXiv id / paper.bib；有 `experiments/` / `checkpoints/` / `configs/` 目录；README 主要讲方法和实验结果 |
| **monorepo** | `packages/` / `apps/` / `crates/` 下有 ≥ 3 个独立可用的子项目；根 README 是聚合性的 |
| **fallback** (兜底通用) | 以上信号都不够强，或项目混合多种特征难以归类 |

**Priority when multiple match**: interface-library > protocol > framework > research > monorepo > application > fallback.

**Hybrid handling**: if a project is genuinely A+B (e.g. CLI tool that's also a library), pick the primary outline and add 1–2 chapters from the secondary preset. State this explicitly in the report header.

**State the inferred type to the user** before continuing (don't pause):

> "看起来这是一个 **<type>** 项目（依据：<一句话理由>），按对应的 outline 来写。如果判定错了请打断我。"

In submodule mode, type detection runs on the **subpath**, not the parent repo — a subdirectory may have a different character than the whole.

### Step 3 — Analyze (according to type + depth)

Run two things in parallel:

1. **Universal analysis methods** — applied on every project, all depths. See "Universal analysis methods" section.
2. **Type-specific focus list** — see the relevant preset under "Project type presets".

The depth tier sets intensity, not which dimensions to cover:

- `quick` — touch each dimension lightly (read README, top-level structure, list public surface or main entry points, skim 1–2 source files)
- `medium` — go into the type's primary chapters in real depth; verify README claims against code; surface archaeology signals
- `deep` — expand every chapter the preset names; trace multiple end-to-end paths or cover every interface group; include ABI / versioning / cross-platform notes where relevant

**No TOP-N scoring formula.** Pick which items to expand (which interfaces, modules, paths, extension points) based on what's interesting in *this* project. If you do narrow the focus, name what you skipped so readers don't infer "skipped = unimportant".

### Step 4 — Generate the report

**Output path:**

- Repo analysis: `./reports/<project-name>_analysis_<depth>.md`
- Submodule analysis: `./reports/<repo>__<subpath-flattened>_analysis_<depth>.md`
  - `<subpath-flattened>`: replace `/` and `\` with `-` (e.g., `internal/pipeline` → `internal-pipeline`)
  - Example: `./reports/goclaw__internal-pipeline_analysis_deep.md`

Create `./reports/` if it doesn't exist.

**Filename rules (ASCII only):**
- Use ASCII characters only — no CJK / accented / emoji. Non-ASCII paths can break tooling on Windows and in cross-platform pipelines.
- Sanitize the project name: strip `/`, `\`, `:`, `*`, `?`, `"`, `<`, `>`, `|`, and any whitespace (replace with `-`).
- The report **content** is still written in 简体中文 per the style guide; only the filename is ASCII.

#### Universal sections (always present, in this order)

1. **元数据头** — see "Header" below
2. **问题域** — what problem does this solve? who's the target user? **what would users have to do without this project?** (反事实陈述是这一节的灵魂)
3. **概念与术语** — definition-list glossary; see "Universal section: 概念与术语" below
4. **[type-specific chapters]** — chapters dictated by the preset chosen in Step 2
5. **协议与分发** — license, commercial use restrictions, distribution model. Usually a short section near the end. Promote to higher position **only** when there's a conflict worth flagging (e.g., "production-ready" claim + non-commercial license)
6. **特点与不足** — split into:
   - **客观事实** — observed, verifiable items. README-vs-reality mismatches and code archaeology signals that didn't fit naturally inside other chapters land here.
   - **主观评价** — Claude's own assessment grounded in those facts. No fluff; tie each opinion to a specific fact above.

**Submodule mode dimension overrides:**
- **协议与分发** collapses to one line: *"继承自母仓库 `<repo>` 的 `<license>`"*. No standalone section unless the submodule has its own LICENSE/NOTICE.
- **概念与术语** still present, scoped to the subpath.
- Type detection is run on the subpath, not the parent.
- Other dimensions stay, scoped to the submodule.

#### Style guide

**`standard` (default)**
- Complete sentences, prose-heavy
- Use markdown headings, tables, fenced code blocks
- Suitable for sharing with teammates
- Example: *"The project uses a Channel trait to abstract over transport mechanisms, allowing transports like Unix sockets and TCP to plug in through a uniform interface."*

**`concise` (速记式)**
- Telegraphic, note-style — drop articles and connecting words
- Heavy use of `→`, `:`, `/`, bullet fragments
- Information density over readability for outsiders
- Example: *"Channel trait → 抽象传输层；transports（unix sock / tcp）统一接入"*
- Sections still use markdown headers, but bullets dominate over paragraphs

#### Header

Always start the report with a metadata header:

```markdown
# <项目名> 分析报告

- **来源**: <github URL / local path / archive>
- **项目类型**: interface-library | application | framework | protocol | research | monorepo | fallback  <!-- 若为 hybrid，写 "A (主) + B (次)" -->
- **分析深度**: quick | medium | deep
- **写作风格**: standard | concise
- **生成日期**: YYYY-MM-DD
- **commit (if cloned)**: <short SHA>
```

### Step 5 — Present the report

After writing the file:

1. Print the **absolute path** to the report so the user can open it directly. Resolve the path with whatever the environment provides — `realpath ./reports/<file>` on bash, `Resolve-Path ./reports/<file>` on PowerShell.
2. In chat, give a 2–3 sentence summary of the headline findings — do **not** restate the whole report.
3. Tell the user something like: *"已写入 `<absolute path>`，可在编辑器中打开。"*

Do not assume any sandbox-only presentation helper exists.

## Project type presets

Each preset specifies the outline chapter order, what each chapter must answer, and the per-depth focus. **Treat the outline as a starting point, not a contract** — drop a chapter if the project genuinely has nothing to say there, or add one if you find something the preset didn't anticipate.

### Preset: interface-library (接口库 / 系统库)

**Chapter outline** (after universal 元数据头 / 问题域 / 概念与术语):

- **底层基础** — 依赖的 syscall / 内核机制 / 协议；项目把哪些底层细节抽象掉了；和同层级竞品（如有）的关系
- **接口分类** — 表格 + 简述，按"上下文构造 / 配置 / 状态查询 / 序列化 / 销毁"或项目自身的自然分组列出全部公共 API
- **接口设计要点** — 命名约定、错误模型、生命周期、线程安全、不透明指针 vs 暴露结构体、可扩展性。每条配 1–2 行代码引用
- **使用方式** — 一段最小 working 示例 + 真实项目中的典型集成位置（哪些知名项目用它、用在哪一步）
- 协议与分发 / 特点与不足（universal）

**Per-depth focus**:
- `quick`: 罗列公共头文件 / pub 接口，分组到一级（如"上下文 / 规则 / 应用"），不深入
- `medium`: 接口分类填到二级（每个 API 一句话功能，从 source 验证）；选 1–2 个典型 API 读实现，提炼设计取舍；从 examples/ 或 tests/ 摘最小示例
- `deep`: 每个接口分组成段（覆盖：API 列表 + 典型调用顺序 + 内部如何翻译到底层调用）；错误模型 / 线程安全 / 生命周期完整成段；跨平台 / 多架构差异；ABI / symbol versioning

### Preset: application (应用 / 工具)

**Chapter outline**:

- **整体架构** — 高层组件图（文字或 mermaid 描述）、进程模型、主要状态机
- **关键数据流 / 控制流** — 至少一条 e2e 路径（用户输入 → 输出 / 副作用），把途径的组件按顺序串起来；指出哪些是同步、哪些异步、哪些跨进程
- **实现要点** — 选 2–4 个非平凡的实现细节（关键数据结构、算法、并发模型、IPC / RPC 形式、持久化、调度）
- **部署与运维** — 怎么部署、配置、监控、升级；纯 CLI 工具时缩为"使用方式"小节
- 协议与分发 / 特点与不足

**Per-depth focus**:
- `quick`: 入口 → 顶层架构图 → 一条主路径名字层面贯通
- `medium`: 一条 e2e 路径详写（含关键文件:行号）；2 个实现要点深入
- `deep`: 多条 e2e 路径；错误处理 / 重试 / 退路；完整部署故事；性能关键路径与 benchmark（如有）

### Preset: framework (框架 / SDK)

**Chapter outline**:

- **设计哲学** — 核心抽象（最重要的 1–3 个 trait / interface / 概念）；与同类框架（如 axum vs actix、Express vs Fastify）的差异点
- **扩展点矩阵** — 表格：扩展点 / 形式（trait / hook / middleware / decorator）/ 触发时机 / 典型用途
- **内部机制** — 框架自己怎么把扩展点串起来、生命周期管理、错误传播、类型推断或反射的代价
- **典型用法** — minimal "hello world" + 一个非平凡场景（中间件链、自定义提取器之类）
- 协议与分发 / 特点与不足

**Per-depth focus**:
- `quick`: 列出核心抽象 + 扩展点矩阵（一级）
- `medium`: 设计哲学成段 + 跟踪一个扩展点从声明到调用的完整路径
- `deep`: 内部机制完整、与同类对比成段、覆盖错误传播 / 生命周期边界 case

### Preset: protocol (协议实现)

**Chapter outline**:

- **协议定位** — 在协议栈中的位置、上下游、与同类协议（如有）的关系；引用的 RFC / draft
- **wire format** — 报文 / 帧 / 包结构、字段含义、版本编码；尽量给一张 ASCII 字段图或字节布局表
- **实现策略** — 状态机、解析器、序列化策略；手写 parser 还是 codegen；零拷贝 / 内存安全策略
- **互操作 / 一致性** — 与官方测试套件的覆盖、已知互操作问题
- 协议与分发 / 特点与不足

**Per-depth focus**:
- `quick`: 协议定位 + wire format 一级骨架
- `medium`: 完整 wire format + 解析器主路径
- `deep`: 全部状态转换、错误恢复、互操作矩阵、与参考实现的差异

### Preset: research (研究代码)

**Chapter outline**:

- **方法** — 论文核心 idea 与代码的映射；指出哪段代码对应论文中哪个公式 / 算法 / 模块
- **实验配置** — 数据集、超参、硬件假设
- **复现路径** — 如何从 clone 到跑通；缺失了什么（数据、checkpoint、license）；预估资源
- **局限** — 已知不工作的场景；论文未声明但代码里能看到的限制
- 协议与分发 / 特点与不足

**Per-depth focus**:
- `quick`: 方法概要 + 顶层目录映射到论文章节
- `medium`: 公式 ↔ 代码映射详写；至少一条数据集 → 输出的复现命令
- `deep`: 实验 reproducibility 全部覆盖；评测脚本逐项核对；与论文报告数字的可信度评估

### Preset: monorepo

**Chapter outline**:

- **子项目矩阵** — 表格：子项目 / 一句话职责 / LoC / 主要语言 / 对外接口 / 状态（active / archived / experimental）
- **关键流转** — 子项目之间的数据 / 依赖流（哪个产出被哪个消费）；如果是 trunk + plugins 模式，说清 trunk 是哪个
- **共享基础设施** — build / lint / CI / 共享库 / 发布流程
- **下钻建议** — 这个 monorepo 哪些子项目最适合用 `<repo>::<sub>` 单独分析
- 协议与分发 / 特点与不足

**Per-depth focus**:
- `quick`: 子项目矩阵 + 一句话流转关系
- `medium`: 流转成段 + 共享基础设施实地核对
- `deep`: 每个子项目一段简评 + 给出 2–3 条具体的 submodule 下钻路径

### Preset: fallback (兜底)

Use when type detection isn't confident, or the project legitimately doesn't fit elsewhere. This is the old single-template approach, kept as a safety net.

**Chapter outline**:

- **设计思路** — 核心抽象、设计哲学、关键架构决策
- **代码组织** — 模块清单表（name / 一句话职责 / LoC / 关键入口文件），全量列出。**不做 TOP-N 评分**；如果模块多，选 3–5 个用粗体段落详写，并显式列出"未详写模块"。
- 协议与分发 / 特点与不足

## Universal section: 概念与术语

读后续章节前的词汇表。两类内容都收：
- **外部概念**: 项目依赖但外部存在的技术（syscall、BPF、OCI spec、cgroup、SPI 等）
- **本项目引入的抽象**: 项目自己造的词或赋予特定含义的类型（filter context、extractor、channel、layer 等）

**收录规则**:
- 3–8 条，硬上限 10。多了就成词典了
- 只收"报告读者可能不知道 / 在本项目中有特定含义"的项
- **不收**通用 CS 概念（function / module / branch / trait / class 等）
- 每条 2–3 行：是什么 + 为什么这么设计 / 非显然之处，再加一行"在本项目中"指明它出现的位置和角色

**格式（定义列表式，不是表格——密度高的内容塞不进表格单元格）**:

```markdown
## 概念与术语

**<术语>** *(外部 / 本项目引入)*
2–3 行：是什么、为什么这么设计、非显然之处。
→ 本项目：在哪里出现、扮演什么角色。

**<下一个术语>** *(外部 / 本项目引入)*
...
```

**示例（节选自 libseccomp）**:

```markdown
**seccomp-bpf** *(外部)*
Linux 内核 ≥3.5 提供的 syscall 过滤机制：进程一次性向内核安装一段 BPF 程序，之后每次发起 syscall 都先过该程序判定（allow/deny/kill/trap/log）。安装是单向的、不可撤销——这是它的安全模型基石。
→ 本项目：所有公共 API 最终都落在"生成并安装一段 BPF"。

**filter context (`scmp_filter_ctx`)** *(本项目引入)*
不透明句柄类型（`typedef void *`），调用方不能直接访问字段。一个 context 持有：规则集、默认动作、目标架构集合、属性表。所有写操作作用在 context 上，最后由 `seccomp_load` 统一编译成 BPF 并安装。
→ 几乎所有公共 API 的第一参数；生命周期：`init → arch_add* → rule_add* → load → release`。
```

## Universal analysis methods

These run on every project regardless of type, at every depth (depth controls intensity, not whether to do them).

### README-vs-reality check

For each verifiable claim in README / 顶层 docs，**到代码里核对**：
- 数字型 claim（"8-stage pipeline"、"20+ providers"、"5-layer security"）— 数一数代码里实际有几个
- 列表型 claim（功能 / 平台 / 兼容版本）— 一项一项找代码证据
- 性能型 claim（"10x faster"、"zero allocation"）— 找 benchmark 或 micro-optimization；找不到就标"未独立验证"
- 状态型 claim（"production-ready"、"battle-tested"）— 看测试覆盖、CI 配置、issue 历史

发现不符时，把它写进对应的正文章节（接口章节里发现接口数量不符 → 写在接口分类里），而不是单独堆到"特点与不足"。**只有**找不到合适章节安放的，才放进"特点与不足 → 客观事实"。

为什么这重要：很多用户从 README 了解项目，从不读代码；surfacing 这种 gap 是本 skill 最高价值的产出之一。

### Code archaeology signals

观察 meta-signals — 它们往往比代码本身更能说明项目的开发背景、质量和风险。

| Signal | 看什么 | 可能含义 |
|---|---|---|
| **Translation comments** | `// Adapted from <other-project>`, `// Matching TS <thing>`, `// Port of <X>` | 从另一种语言移植；idiom 可能不本土；质量取决于移植者水平 |
| **Idiom mismatch** | Go 里 `MemoryMB int` 而不是 `time.Duration`；语言有 `Option` 却手写 nullable 模式 | AI 辅助移植或非母语作者；fork 后改造成本 |
| **Aggressive version pins** | 语言 / 运行时 / DB 锁在最新或预发布版本 | 优先尝鲜，部署阻力；企业落地难 |
| **Scale vs attribution mismatch** | 200k+ LoC 但只有单作者或"X Contributors"泛指 | 重度 AI 辅助 / 影子团队 / 历史合并；质量在不同模块间差距大 |
| **Bundled third-party assets without attribution** | vendored 的 skills / data / configs 来自具名项目但 LICENSE / NOTICE 无致谢 | 若 redistribute 有合规风险 |
| **README hyperbole vs code reality** | "8-stage" 但代码 7 个；"20+ providers" 但实际 12；"production-tested" 无可观测性配置 | 营销驱动的 docs；信代码不信 README |
| **Localization theater** | 30 种语言 README 但核心 docs 只 1 种 | 为 star 数优化，不是为 maintenance |
| **Test/code organization** | 分层测试（contracts / integration / invariants / scenarios）vs 一个 `tests/` 堆 | 结构化测试 → 工程成熟度信号 |
| **Commit cadence and authorship** | 长时间空档、单作者爆发、AI 风格 commit message | 可持续性信号 |

发现这些信号时，按以下原则落地：
- 信号本身（"BPF 编译器是手写 C"） → "客观事实"
- 它意味着什么（"维护成本高，但 zero 第三方依赖"） → "主观评价"

### 概念抽取

读 README 和顶层结构时同步收集候选术语，按"概念与术语"的收录规则筛选。优先收：项目反复出现的大写类型名、README 中加粗或代码 fence 起来的名词、依赖列表中需要解释才能让读者理解为什么需要它的库。

## 输出示例

下面是一份 `medium` / `standard` 报告的片段（interface-library 类型，节选自 libseccomp），用来锚定风格——**不是模板**，章节顺序和措辞应根据具体项目调整。

```markdown
# libseccomp 分析报告

- **来源**: https://github.com/seccomp/libseccomp
- **项目类型**: interface-library
- **分析深度**: medium
- **写作风格**: standard
- **生成日期**: 2026-05-14
- **commit**: abc1234

## 问题域

直接用 `prctl(PR_SET_SECCOMP, SECCOMP_MODE_FILTER)` + 手写 BPF 字节码可以完成 syscall 过滤，
但调用方要自己承担三件不平凡的事：
(1) 组装 BPF 指令集（指令小、人写易出错）；
(2) 处理不同架构上同名 syscall 编号不同（x86_64 / arm64 / i386 / x32 各异）；
(3) 安装失败就是进程崩溃，调试只能靠 strace + audit log。
libseccomp 把这三件事抽象成"规则 + 编译"，调用方写一组高层规则，库负责生成正确架构的 BPF 并加载。
目标用户：容器运行时（runc/crun/podman）、systemd 单元、桌面沙箱（Flatpak/Bubblewrap）等需要 hardening 的程序。

## 概念与术语

**seccomp-bpf** *(外部)*
Linux 内核 ≥3.5 提供的 syscall 过滤机制：进程一次性向内核安装一段 BPF 程序，之后每次发起 syscall 都先过该程序判定（allow/deny/kill/trap/log）。安装是单向的、不可撤销——这是它的安全模型基石。
→ 本项目：所有公共 API 最终都落在"生成并安装一段 BPF"。

**BPF** *(外部)*
Berkeley Packet Filter，最早为 tcpdump 设计的内核态过滤虚拟机；指令集只十余条、无任意循环，便于内核做加载前静态验证。seccomp 直接复用这套指令集来表达 syscall 规则——这也是为什么 libseccomp 内部要带一个编译器。
→ 本项目：规则的目标语言；编译器位于 `src/gen_bpf.c`。

**syscall number** *(外部)*
syscall 在内核 syscall 表中的整数索引。同一名字（`openat`、`read`）在不同架构编号各异，BPF 程序里只能拿数字比较——所以多架构过滤必须先把名字翻成正确编号。
→ 本项目：`src/arch-*.c` 维护各架构映射；`SCMP_SYS(name)` 宏在编译期翻译。

**filter context (`scmp_filter_ctx`)** *(本项目引入)*
不透明句柄类型（`typedef void *`），调用方不能直接访问字段。一个 context 持有：规则集、默认动作、目标架构集合、属性表。所有写操作作用在 context 上，最后由 `seccomp_load` 统一编译成 BPF 并安装。
→ 几乎所有公共 API 的第一参数；生命周期：`init → arch_add* → rule_add* → load → release`。

**default action** *(本项目引入)*
context 的核心属性，未命中任何规则时的兜底。取值：`SCMP_ACT_KILL` / `TRAP`（发 SIGSYS）/ `ERRNO`（返回指定 errno）/ `LOG` / `ALLOW`。这是 seccomp 从 "allow-list-or-die" 推广到细粒度策略的关键扩展。
→ `seccomp_init()` 调用时第一个必须确定的参数。

## 底层基础

libseccomp 完全构建在 seccomp-bpf 之上，自身不引入任何额外的内核 patch。三件抽象掉的事：
1. **BPF 编译**：`src/gen_bpf.c` 实现了一个针对 syscall 过滤场景的优化编译器，把规则树编译成对应架构的 BPF 程序。优化点包括 syscall 编号排序后的二分跳转、相同动作的规则合并。
2. **多架构映射**：`src/arch-*.c`（每架构一个文件）维护 syscall 名 → 编号映射；`SCMP_SYS(name)` 在编译期完成翻译。新增架构主要工作是补这张表。
3. **错误传播**：所有公共 API 用 `int` 返回 `-errno`，避免 errno 全局污染；这点与 POSIX 风格不一致但更适合库使用。

无 vendored 第三方代码、无 codegen 工具链——`gen_bpf.c` 是手写 C。

## 接口分类

| 分组 | API | 一句话功能 | 典型调用时机 |
|---|---|---|---|
| 上下文 | `seccomp_init` / `seccomp_release` | 创建 / 释放 filter context | 启动期最早 / 最晚 |
| 架构 | `seccomp_arch_add` / `_remove` / `_exist` | 配置目标架构集合 | 启动期，rule_add 之前 |
| 规则 | `seccomp_rule_add` / `_exact` / `_array` | 加入 allow/deny 规则 | 启动期，load 之前 |
| 属性 | `seccomp_attr_set` / `_get` | 设置 default action、NoNewPrivs 等 | 启动期，可与 rule_add 交叉 |
| 应用 | `seccomp_load` | 把 context 编译并安装到内核 | 启动期最后一步（不可逆） |
| 序列化 | `seccomp_export_pfc` / `_bpf` | 导出规则用于调试 / 离线编译 | 调试 |

## 接口设计要点

- **不透明句柄**: `scmp_filter_ctx` 被 typedef 到 `void *`，调用方拿到也无法访问字段——版本演进时内部结构可任意改。代价是 debugger 里看不到内容，需要 `seccomp_export_pfc` 转储。
- **错误模型**: 公共 API 返回 `int`，0 成功，负值是 `-errno`。没有全局 errno 污染；但用户需要记得取反才能传给 `strerror()`。
- **生命周期**: 单写多读语义——`seccomp_load` 后 context 进入只读阶段，再调写操作返回 `-EBUSY`。
- **线程安全**: 单个 context 不是线程安全的（写操作需要外部加锁）；但加载后的 filter 自然作用于整个进程的所有线程。
- **跨架构**: 同一份规则可以同时编译到 x86_64 + arm64 + i386，`seccomp_arch_add` 增加目标架构后规则会被多次实例化。

## 使用方式

最小示例（拒绝 `getuid`、其余允许）：

\`\`\`c
#include <seccomp.h>
scmp_filter_ctx ctx = seccomp_init(SCMP_ACT_ALLOW);  // 默认允许
seccomp_rule_add(ctx, SCMP_ACT_ERRNO(EPERM), SCMP_SYS(getuid), 0);
seccomp_load(ctx);
seccomp_release(ctx);
\`\`\`

典型集成位置：
- **runc/crun**: 在 init 阶段（容器命名空间已建立、exec 用户进程之前）从 `config.json` 的 seccomp profile 加载规则
- **systemd**: 在 unit 启动时通过 `SystemCallFilter=` 配置项，systemd 内部调用 libseccomp 构建规则
- **Bubblewrap / Flatpak**: 在 sandbox 启动器中加载预设策略

## 协议与分发

LGPL-2.1（不是 GPL）。允许链接到闭源程序（动态或静态链接都满足 LGPL 条款），无商业限制。
分发为系统包（apt/dnf/...）和源码 release；上游不提供官方 binary release。

## 特点与不足

### 客观事实
- BPF 编译器 `src/gen_bpf.c` 是手写 C，无 codegen 依赖
- 多架构支持表（`src/arch-*.c`）覆盖 x86 / arm / mips / s390 / ppc / riscv / loongarch 等
- 测试 (`tests/`) 同时包含 API 行为测试和 BPF 输出快照测试（按架构）
- README 声称支持的架构在 `src/arch.c` 的 `arch_def` 表中可逐一对应——无水分
- 错误返回用 `-errno` 而非 POSIX 风格的 `errno + -1`，对 C 库来说是少见选择

### 主观评价
项目质量高、scope 小而专注——这是它能成为容器生态默认依赖的根本原因。
不透明句柄 + `-errno` 风格让 ABI 演进自由度大，从 2.x 到 2.5+ 保持了二进制兼容。
对新使用者最大的认知障碍不是 API 本身（很简单），而是理解"安装是单向且作用于整个进程"这一前置事实——
这一点 README 提及但不显眼，可能是新用户的踩坑高发区。
```

注意示例中：
- **问题域**用反事实陈述（"没有它你要自己做……"）打开，比"这是一个 syscall 过滤库"信息量大得多
- **概念与术语**紧跟问题域，让后续章节里出现的 BPF / context / default action 不需要再解释
- **底层基础**和**接口分类 / 设计要点**是 interface-library 类型独有的，替代了通用模板里的"设计思路 / modules"
- **协议与分发**位置靠后（只占一段），因为这是个 LGPL 库，没有需要前置警告的冲突
- **客观事实 / 主观评价**严格分开；评价的每一句都能追溯到上面的一条事实

---

- If a project is too large to fully analyze at the requested depth, write the report anyway and note in the header which areas were sampled vs fully covered.
- If the source resolves to multiple top-level directories (and not detected as monorepo), ask the user whether to analyze the whole repo or a specific subproject.
- If the user provides a name that returns no GitHub matches, ask whether they meant something else or want to provide a URL/path directly.
- Don't fabricate. If you couldn't determine something (e.g., performance characteristics without benchmarks), say so explicitly in the report.
