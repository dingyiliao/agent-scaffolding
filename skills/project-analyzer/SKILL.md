---
name: project-analyzer
description: Analyze a software project and produce a structured markdown report covering its problem domain, license/distribution, design philosophy, code organization, and strengths/weaknesses. Accepts a GitHub repo name (searches and clones), a local directory path, or an uploaded archive (.zip/.tar.gz/.tar). Use this skill whenever the user wants to understand, explore, study, or summarize an existing project, codebase, or repository — even without the word "analyze." Strong trigger phrases include "了解这个项目", "看一下这个仓库", "整理一份项目报告", "分析 X repo", "study this repo", "understand this codebase", or simply naming a project and asking for a writeup. Goes beyond surface summary by checking README claims against actual code, flagging code archaeology signals (AI-port artifacts, idiom mismatches, license/positioning conflicts), and gating large codebases (over 100k LoC) for user confirmation. Supports three depth levels (quick / medium / deep) and two writing styles (standard / concise) via parameter.
---

# Project Analyzer

Produces a structured markdown analysis report of a software project, tailored to a user-selected depth and style.

## Inputs to collect

Before analysis, confirm these from the user:

1. **Source** — one of:
   - GitHub repo name (full `owner/repo` or partial like `langchain`)
   - Local directory path (already on disk)
   - Path to an uploaded archive (`.zip`, `.tar.gz`, `.tar`)
2. **Depth** (default `medium`): `quick` | `medium` | `deep`
3. **Style** (default `standard`): `standard` | `concise`

If the user provided a request without specifying depth or style, pick the defaults and **state them explicitly** ("I'll do a medium-depth standard-style analysis — say so if you want different."). Don't block on confirmation for defaults.

## Workflow

### Step 1 — Resolve source to a local directory

**GitHub repo name:**
- Use `gh search repos <query>` if the `gh` CLI is available, otherwise use `web_search` with a query like `site:github.com <query>`
- If multiple plausible matches exist, list the **top 3 by star count** with a one-line description each, and ask the user to pick. Do NOT pick silently.
- Once confirmed, clone with shallow depth:
  ```bash
  git clone --depth 1 https://github.com/<owner>/<repo>.git /home/claude/<repo>
  ```
- For private repos: ask the user to clone manually and provide the local path.

**Local directory path:** use as-is.

**Archive:** extract to `/home/claude/<archive-basename>/`:
```bash
# zip
unzip -q <archive> -d /home/claude/<basename>
# tar.gz
mkdir -p /home/claude/<basename> && tar -xzf <archive> -C /home/claude/<basename>
```

### Step 1.5 — Size check & gate

Before diving into analysis, run a quick LoC count on the resolved directory:

```bash
find <dir> -name '*.go' -o -name '*.py' -o -name '*.js' -o -name '*.ts' \
       -o -name '*.rs' -o -name '*.java' -o -name '*.cpp' -o -name '*.c' \
       -o -name '*.rb' -o -name '*.php' \
  | grep -v -E '/(node_modules|vendor|\.git|target|build|dist)/' \
  | xargs wc -l 2>/dev/null | tail -1
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

#### `medium` — selective deep dive (default)
- Fully read entry points and main modules
- Identify and read core data structures / type definitions / key traits/interfaces
- Trace 1–2 typical end-to-end call chains
- Note key dependencies and their purpose
- Look at the test suite to learn intended behaviors
- **Compare README claims to code reality** — for each major feature claim in the README (especially numbered claims like "8-stage pipeline", "5-layer security", "20+ providers"), verify against actual code. Note discrepancies in the report. Many users learn projects from READMEs and never read code; surfacing these gaps is one of the highest-value things this skill provides.
- **Watch for code archaeology signals** (see `Code archaeology signals` section below)

#### `deep` — comprehensive analysis
- Map the full module graph and inter-module dependencies
- Read non-trivial algorithms / clever logic
- Identify performance-critical paths and any benchmarks
- Note security / safety considerations (unsafe blocks, eval, sandboxing, etc.)
- Examine error handling patterns
- Check build / CI / release configuration
- Look at issue tracker or CHANGELOG for recurring pain points (if accessible)
- **All the medium-tier items above also apply** — including README-vs-reality check and archaeology signals — done more thoroughly

### Step 3 — Generate the report

**Output path:** `/mnt/user-data/outputs/<project-name>_分析_<depth>.md`

Examples: `langchain_分析_medium.md`, `axum_分析_deep.md`

**Required core dimensions** (always present, in this order):

1. **需求 / 问题域** — what problem does this solve? who's the target user?
2. **协议与分发** — license (MIT / Apache / GPL / CC / proprietary / dual-license / etc.), commercial use restrictions, distribution model (single binary / Docker / SaaS / library). This dimension comes early because it determines whether the target users from #1 can actually use the project. Flag any conflicts (e.g., "production-ready multi-tenant platform" + "non-commercial-only license" is a red flag worth surfacing).
3. **设计思路** — core abstractions, design philosophy, key architectural decisions
4. **具体实现** — code organization, module structure, technologies used
5. **特点与不足** — split into two subsections:
   - **客观事实** — observed facts, no judgment (e.g., "no test coverage on the parser module", "uses unsafe in 12 places")
   - **主观评价** — Claude's own assessment grounded in the facts above

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

After writing, call `present_files` with the report path so the user can download it. Give a brief 2–3 sentence summary in chat — do **not** restate the whole report.

## Code archaeology signals

Beyond just describing what's in the code, look for *meta-signals* that tell you about the project's development context. These often reveal more about quality and risk than the code itself:

| Signal | What to look for | What it might mean |
|---|---|---|
| **Translation comments** | `// Adapted from <other-project>`, `// Matching TS <thing>`, `// Port of <X>` | Code is a port from another language; idioms may not be native; quality depends on translator skill |
| **Idiom mismatch** | Field naming like `MemoryMB int` instead of `time.Duration` in Go; manual nullable patterns where a language has `Option`; etc. | AI-assisted port or non-native author; refactor cost if you fork |
| **Aggressive version pins** | Latest-and-greatest language version (Go 1.26 when 1.24 is current), brand-new DB versions (PG 18 fresh GA) | Author prioritizes new features over deployability; will block enterprise adoption |
| **Scale vs attribution mismatch** | 200k+ LoC with single author or "X Contributors" generic credit | Either heavy AI assistance, ghost team, or historical aggregation; quality varies sharply across modules |
| **Bundled third-party assets without attribution** | Vendored skills/data/configs from named projects with no credit in LICENSE/NOTICE | Compliance risk if you redistribute |
| **README hyperbole vs code reality** | "8-stage pipeline" but code has 7; "20+ providers" but only 12 actively wired; "production-tested" with no observability config | Marketing-driven docs; trust code over README |
| **Localization theater** | 30 README languages but core docs only in one | Optimizing for stars, not maintenance |
| **Test/code organization** | Layered test directories (contracts/integration/invariants/scenarios) vs single `tests/` dump | Structured testing suggests engineering maturity |
| **Commit cadence and authorship** (if accessible) | Long gaps, single-author bursts, AI-style commit messages | Sustainability signal |

When you find these signals, include them in the report — usually under "特点与不足 → 客观事实" (the signal itself) and "主观评价" (what it implies).



- If a project is too large to fully analyze at the requested depth, write the report anyway and note in the header which areas were sampled vs fully covered.
- If the source resolves to multiple top-level directories (a monorepo), ask the user whether to analyze the whole repo or a specific subproject.
- If the user provides a name that returns no GitHub matches, ask whether they meant something else or want to provide a URL/path directly.
- Don't fabricate. If you couldn't determine something (e.g., performance characteristics without benchmarks), say so explicitly in the report.
