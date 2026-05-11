---
name: project-analyzer
description: Analyze a software project and produce a structured markdown report covering its problem domain, license/distribution, design philosophy, modules breakdown, and strengths/weaknesses. Accepts a GitHub repo name (searches and clones), a local directory path, an uploaded archive (.zip/.tar.gz/.tar), or a submodule form `<repo-or-path>::<subpath>` for drilling into one subdirectory. Use this skill whenever the user wants to understand, explore, study, or summarize an existing project, codebase, or repository — even without the word "analyze." Strong trigger phrases include "了解这个项目", "看一下这个仓库", "整理一份项目报告", "分析 X repo", "study this repo", "understand this codebase", or simply naming a project and asking for a writeup. Goes beyond surface summary by checking README claims against actual code, flagging code archaeology signals (AI-port artifacts, idiom mismatches, license/positioning conflicts), and gating large codebases (over 100k LoC) for user confirmation. Reports are written in 简体中文 to `./reports/<name>_analysis_<depth>.md` with ASCII-only filenames; cross-platform (Windows PowerShell + *nix bash). Supports three depth levels (quick / medium / deep) and two writing styles (standard / concise) via parameter.
---

# Project Analyzer

Produces a structured markdown analysis report of a software project, tailored to a user-selected depth and style.

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

### Step 2 — Analyze according to depth

For each tier, do everything in the lower tiers first, then add.

#### `quick` — surface scan
- Read `README*`, top-level docs (`CONTRIBUTING`, `ARCHITECTURE`, `docs/`)
- Get directory tree 2–3 levels deep (`tree -L 3` or `find . -maxdepth 3`)
- Identify entry points (`main.*`, `index.*`, `__main__.py`, binary targets in `Cargo.toml`, `bin` field in `package.json`)
- Read package metadata (`Cargo.toml`, `package.json`, `pyproject.toml`, `go.mod`, `pom.xml`, etc.)
- Skim 1–2 representative source files
- **Build the module list** (see "Module identification" below) — just the list, no per-module dive. **Skip entirely in submodule mode** — a submodule report has no modules section.

#### `medium` — selective deep dive (default)
- Fully read entry points and main modules
- Identify and read core data structures / type definitions / key traits/interfaces
- Trace 1–2 typical end-to-end call chains
- Note key dependencies and their purpose
- Look at the test suite to learn intended behaviors
- **Compare README claims to code reality** — for each major feature claim in the README (especially numbered claims like "8-stage pipeline", "5-layer security", "20+ providers"), verify against actual code. Note discrepancies in the report. Many users learn projects from READMEs and never read code; surfacing these gaps is one of the highest-value things this skill provides.
- **Build the module list, score it, deep-dive the TOP-N (default 3–5)** — see "Module identification" below. **Skip entirely in submodule mode.**
- **Watch for code archaeology signals** (see `Code archaeology signals` section below)

#### `deep` — comprehensive analysis
- Map the full module graph and inter-module dependencies
- Read non-trivial algorithms / clever logic
- Identify performance-critical paths and any benchmarks
- Note security / safety considerations (unsafe blocks, eval, sandboxing, etc.)
- Examine error handling patterns
- Check build / CI / release configuration
- Look at issue tracker or CHANGELOG for recurring pain points (if accessible)
- **Expand every module** in the module list to the medium per-module depth. If the module count is > 15, print the full list to chat and **continue expanding all** — append: *"模块数较多（X 个），按 deep 档展开全部。如希望只展开部分，请打断我并指定。"* Don't pause execution. **Skip entirely in submodule mode.**
- **All the medium-tier items above also apply** — including README-vs-reality check and archaeology signals — done more thoroughly

### Step 3 — Generate the report

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

Examples: `./reports/langchain_analysis_medium.md`, `./reports/axum_analysis_deep.md`

**Required core dimensions** (always present, in this order):

1. **需求 / 问题域** — what problem does this solve? who's the target user?
2. **协议与分发** — license (MIT / Apache / GPL / CC / proprietary / dual-license / etc.), commercial use restrictions, distribution model (single binary / Docker / SaaS / library). This dimension comes early because it determines whether the target users from #1 can actually use the project. Flag any conflicts (e.g., "production-ready multi-tenant platform" + "non-commercial-only license" is a red flag worth surfacing).
3. **设计思路** — core abstractions, design philosophy, key architectural decisions
4. **modules** — module-level breakdown of the codebase. Replaces the older free-form "具体实现" / "code organization" prose. Always starts with a **module list table** (name, one-line responsibility, LoC, key entry files). Per-depth expansion rules are in the "Module identification" section below.
5. **特点与不足** — split into two subsections:
   - **客观事实** — observed facts, no judgment (e.g., "no test coverage on the parser module", "uses unsafe in 12 places")
   - **主观评价** — Claude's own assessment grounded in the facts above

**Submodule mode dimension overrides:**
- **协议与分发** collapses to one line: *"继承自母仓库 `<repo>` 的 `<license>`"*. No standalone section unless the submodule has its own LICENSE/NOTICE.
- **modules** section is **omitted** — a submodule report is itself a deep view of one module. **All module-list / scoring / TOP-N steps are skipped at every depth tier in submodule mode** — this override beats the depth-tier instructions in Step 2.
- Other dimensions stay, scoped to the submodule.

**Flexible chapters** — beyond the four required, add chapters that fit *this* project. Don't force every chapter on every project. Examples by project type:

| Project type | Likely extra chapters |
|---|---|
| Framework | Extension points, plugin model, escape hatches |
| Library | API design, ergonomics, common pitfalls |
| Application | Deployment model, configuration, ops story |
| Research code | Experimental scope, reproducibility |
| Protocol/spec | Wire format, conformance, interop |

### Style guide

#### `standard` (default)
- Complete sentences, prose-heavy
- Use markdown headings, tables, fenced code blocks
- Suitable for sharing with teammates
- Example: *"The project uses a Channel trait to abstract over transport mechanisms, allowing transports like Unix sockets and TCP to plug in through a uniform interface."*

#### `concise` (速记式)
- Telegraphic, note-style — drop articles and connecting words
- Heavy use of `→`, `:`, `/`, bullet fragments
- Information density over readability for outsiders
- Example: *"Channel trait → 抽象传输层；transports（unix sock / tcp）统一接入"*
- Sections still use markdown headers, but bullets dominate over paragraphs

### Header

Always start the report with a metadata header:

```markdown
# <项目名> 分析报告

- **来源**: <github URL / local path / archive>
- **分析深度**: quick | medium | deep
- **写作风格**: standard | concise
- **生成日期**: YYYY-MM-DD
- **commit (if cloned)**: <short SHA>
```

### Step 4 — Present the report

After writing the file:

1. Print the **absolute path** to the report so the user can open it directly. Resolve the path with whatever the environment provides — `realpath ./reports/<file>` on bash, `Resolve-Path ./reports/<file>` on PowerShell.
2. In chat, give a 2–3 sentence summary of the headline findings — do **not** restate the whole report.
3. Tell the user something like: *"已写入 `<absolute path>`，可在编辑器中打开。"*

Do not assume any sandbox-only presentation helper exists.

## Module identification

The **modules** dimension is one of the highest-value parts of the report. It gives readers a navigable map of the codebase and tells them where to dig next.

### Candidate discovery (structural signals first)

| Language / layout | Module candidates |
|---|---|
| Go | each top-level subdir of `cmd/`, `internal/`, `pkg/` |
| Rust | each crate in a Cargo workspace |
| Node monorepo | each entry in `packages/*`, `apps/*` (read `pnpm-workspace.yaml` / `package.json#workspaces`) |
| Python | packages declared in `pyproject.toml`, or top-level dirs under `src/` |
| Java / Maven / Gradle | top-level Maven `<modules>` entries from `pom.xml`; otherwise each second-level package under `src/main/java/<group>/<artifact>/`; Gradle multi-project reads `settings.gradle(.kts)` |
| Other / flat | each top-level non-vendor / non-build / non-test directory |

### Aggregation rules (when candidate count > 12)

Plain candidate lists explode on large repos. Aggregate before showing:

- **Infrastructure bucket**: merge utility-ish dirs (name matches `testutil`, `safego`, `internal/util*`, `i18n`, `version`, `errors`, etc.) into one "基础设施" entry
- **Semantic clusters**: merge dirs whose names indicate a shared concern (e.g., `memory` + `consolidation` + `knowledgegraph` + `vault` → "记忆与知识"). Use LLM judgment, but err toward keeping things separate when in doubt
- **Independent-seat exception**: directories explicitly named in README / CLAUDE.md / ARCHITECTURE / docs are **never aggregated** — those are the modules the project itself considers core

### User visibility (non-blocking)

- `quick`: produce the list silently
- `medium` / `deep`: print the aggregated list to chat (one line per module: name + one-line responsibility + LoC), then **continue the analysis using that list as-is**. Append: *"如需合并、拆开或忽略某些模块，请打断我；否则按此清单继续。"* Do not pause execution waiting for a reply — the user can always interrupt.

### TOP-N scoring (medium depth)

Score each module to decide which ones get a deep paragraph in `medium`:

```
score = 0.4 * LoC_normalized
      + 0.4 * incoming_imports_normalized
      + 0.2 * readme_mentions_normalized
```

All three components are normalized **relative to the maximum value across modules in this analysis** so they're directly comparable (top module on each axis = 1.0):

- `LoC_normalized` = module LoC ÷ **max module LoC**
- `incoming_imports_normalized` = this module's incoming-import count ÷ **max incoming-imports across modules**. Count = number of other modules that reference this module's package path, excluding the module's own directory.
  - **Prefer the harness's Grep tool** (cross-platform, no shell quoting issues). Pattern: the module's import path or display name. Filter out matches whose file path starts with the module's own directory.
  - bash fallback (Go example): `grep -rln "<module-pkg-path>" --include='*.go' . | grep -v "^./<module-dir>/" | wc -l`
  - PowerShell fallback (Go example): `(Get-ChildItem -Recurse -Filter *.go | Select-String -Pattern '<module-pkg-path>' -List | Where-Object { $_.Path -notmatch '[\\/]<module-dir>[\\/]' }).Count`
  - Other languages: search for the module's outward identifier — Rust: crate name in `use` statements; Node: package name in `import`/`require`; Python: dotted import path.
- `readme_mentions_normalized` = this module's mention count ÷ **max mention count across modules**. Count = occurrences of the module name (or its display name) in README / CLAUDE.md / top-level docs (`ARCHITECTURE.md`, `docs/*.md`).

Default `N = 3–5` depending on total module count. **Always print the score table for selected modules** in the report so the reader can disagree with the ranking.

### Per-depth section behavior

| Depth | modules section contents |
|---|---|
| `quick` | **Module list table only** (name, one-line responsibility, LoC, entry files). Do NOT read inside modules. Append: *"如需展开某模块，请用 `<repo>::<module>` 再次触发本 skill。"* |
| `medium` | Module list table + score table for TOP-N + **per-TOP-N deep paragraph** (~150–250 字 each, covering: core abstractions, key files, visible idiom / archaeology signals). Explicitly call out which modules were **not** detailed, so the reader doesn't infer "not detailed = not important". |
| `deep` | Per-module paragraph for every module (medium per-module depth). Table still appears first as a TOC. If module count > 15, list them and ask the user to pick which to expand vs. summarize — deep ≠ blind. **Optional**: append a "模块依赖" subsection with ≤ 15 prominent edges (A → B), filtered by call density. |

## Code archaeology signals

Beyond just describing what's in the code, look for *meta-signals* that tell you about the project's development context. These often reveal more about quality and risk than the code itself:

| Signal | What to look for | What it might mean |
|---|---|---|
| **Translation comments** | `// Adapted from <other-project>`, `// Matching TS <thing>`, `// Port of <X>` | Code is a port from another language; idioms may not be native; quality depends on translator skill |
| **Idiom mismatch** | Field naming like `MemoryMB int` instead of `time.Duration` in Go; manual nullable patterns where a language has `Option`; etc. | AI-assisted port or non-native author; refactor cost if you fork |
| **Aggressive version pins** | Language / runtime / DB pinned to the latest-or-newer release (a version ahead of current stable, or a DB version that just GA'd) | Author prioritizes new features over deployability; will block enterprise adoption |
| **Scale vs attribution mismatch** | 200k+ LoC with single author or "X Contributors" generic credit | Either heavy AI assistance, ghost team, or historical aggregation; quality varies sharply across modules |
| **Bundled third-party assets without attribution** | Vendored skills/data/configs from named projects with no credit in LICENSE/NOTICE | Compliance risk if you redistribute |
| **README hyperbole vs code reality** | "8-stage pipeline" but code has 7; "20+ providers" but only 12 actively wired; "production-tested" with no observability config | Marketing-driven docs; trust code over README |
| **Localization theater** | 30 README languages but core docs only in one | Optimizing for stars, not maintenance |
| **Test/code organization** | Layered test directories (contracts/integration/invariants/scenarios) vs single `tests/` dump | Structured testing suggests engineering maturity |
| **Commit cadence and authorship** (if accessible) | Long gaps, single-author bursts, AI-style commit messages | Sustainability signal |

When you find these signals, include them in the report — usually under "特点与不足 → 客观事实" (the signal itself) and "主观评价" (what it implies).

## 输出示例（standard，节选）

下面是一份 `medium` / `standard` 报告的片段，用来锚定风格——**不是模板**，章节顺序和措辞应根据具体项目调整。

```markdown
# axum 分析报告

- **来源**: https://github.com/tokio-rs/axum
- **分析深度**: medium
- **写作风格**: standard
- **生成日期**: 2026-05-11
- **commit**: a1b2c3d

## 协议与分发

axum 采用 MIT license，无附加商业限制，作为 crate 通过 crates.io 分发。它是一个库（library），不提供独立运行的二进制，使用者需要在自己的 Rust 工程中引入。由于无 GPL 类传染条款，可自由用于商业闭源项目。

## modules

| 模块 | 一句话职责 | LoC | 关键入口 |
|---|---|---|---|
| routing | URL → handler 匹配树 | 2.8k | `src/routing/mod.rs` |
| extract | request → handler 参数提取 | 3.2k | `src/extract/mod.rs` |
| middleware | tower 层栈适配 | 1.4k | `src/middleware.rs` |
| response | 类型 → HTTP response 转换 | 0.9k | `src/response/mod.rs` |
| handler | Handler trait 与实现 | 0.7k | `src/handler/mod.rs` |

**TOP-N 评分**（`score = 0.4·LoC + 0.4·imports + 0.2·readme`，组内最大值归一化）：

| 模块 | LoC | imports | readme | score |
|---|---|---|---|---|
| extract | 1.00 | 0.84 | 0.62 | **0.86** |
| routing | 0.88 | 1.00 | 1.00 | **0.95** |
| response | 0.28 | 0.71 | 0.40 | 0.48 |

**routing（详写）**：核心是 `Router::new()` 构建的 prefix-tree matcher。路由通过 builder API（`.route("/x", get(handler))`）注册，无过程宏；这是 README "macro-free" 主张的代码对应。匹配在 hot path 上做了零分配优化，但需要在编译期推断 handler 签名，是 trait bound 报错冗长的主要来源……（继续到 150–250 字）

**extract（详写）**：通过 `FromRequest` / `FromRequestParts` trait 把 request 拆成 handler 参数。提取器可组合（`Json<T>` + `Query<U>` + `State<S>` 同时存在），失败路径返回 `Rejection` 类型……（继续到 150–250 字）

**未详写模块**：middleware（功能聚焦于一层 tower 适配，无独立架构话题）；response、handler（实现直白，可在 routing/extract 的语境中顺带理解）。

## 特点与不足

### 客观事实
- 核心 trait `Handler` 通过 GAT 实现零分配的中间件链
- 测试套件覆盖路由匹配、提取器、中间件三个层次，但缺少端到端的 HTTP/2 测试
- 依赖 `tower` 生态，意味着 axum 用户事实上需要先理解 `tower::Service`

### 主观评价
学习曲线在 trait bound 推断错误时陡峭——错误信息常常长达数十行，这是 axum 借力 tower 生态付出的成本。对于已经熟悉 Rust async 与 tower 的团队，axum 是当前最人体工学的选择；对于刚接触 Rust 后端的团队，`actix-web` 的报错更友好。

注：README 宣称的 "macro-free routing" 在源码中确实成立（路由通过 builder 而非过程宏构建），与代码一致。
```

注意示例中：
- 协议与分发先于设计/实现出场，符合"能不能用得上"的优先级
- modules 章节先给清单表（覆盖全部模块），再给 TOP-N 评分表（让读者看到打分依据），最后用粗体模块名分段详写；**未详写模块必须显式列出**，避免读者误认为它们"不重要"
- 客观事实只列可验证项，主观评价基于这些事实展开
- README 主张被显式比对（"macro-free routing" 那一句）

（上面省略了"## 设计思路"章节，按要求顺序它应位于"协议与分发"与"modules"之间。）

- If a project is too large to fully analyze at the requested depth, write the report anyway and note in the header which areas were sampled vs fully covered.
- If the source resolves to multiple top-level directories (a monorepo), ask the user whether to analyze the whole repo or a specific subproject.
- If the user provides a name that returns no GitHub matches, ask whether they meant something else or want to provide a URL/path directly.
- Don't fabricate. If you couldn't determine something (e.g., performance characteristics without benchmarks), say so explicitly in the report.
