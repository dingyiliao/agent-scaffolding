---
name: project-analyzer
description: 分析一个软件项目并生成一份结构化的 markdown 报告。当用户希望理解、探索、研究或总结一个已有项目、代码库或仓库时调用——即便用户没明确说"分析"。强触发短语包括"了解这个项目"、"看一下这个仓库"、"整理一份项目报告"、"分析 X repo"、"study this repo"、"understand this codebase"，或者直接给出项目名让你写一份说明。
---

# Project Analyzer

生成一份结构化的项目分析报告。**outline 由项目本身的类型决定**，不是预先定死的模板。一个接口库和一个 CLI 应用所需要的叙述方式根本不同；这个 skill 在七种 preset 中选一种（或走兜底），让报告结构服务于项目，而不是反过来。

## 需要收集的输入

1. **Source** (必填——缺失必须问) ——以下之一：
   - GitHub 仓库名（完整的 `owner/repo` 或片段如 `langchain`）
   - 本地目录路径（已经在磁盘上）
   - 上传压缩包路径（`.zip` / `.tar.gz` / `.tar`）
   - **子模块形式**：`<repo-or-path>::<subpath>` ——在母仓库的上下文里分析单个子目录。例：`./.cache/goclaw::internal/pipeline`、`nextlevelbuilder/goclaw::internal/pipeline`。行为差异见下文"子模块模式"。
2. **Depth** (可选，默认 `medium`)：`quick` | `medium` | `deep`
3. **Style** (可选，默认 `standard`)：`standard` | `concise`

Depth / Style：用户没指定时，**用默认值并显式说明**再开始（"我会按 medium 深度 + standard 风格分析——想换档请打断我。"）。默认值不要阻塞确认。**只有 Source 必须确认**。

## 工作流程

### Step 1 — 把 source 解析成本地目录

所有 clone / 解压都放到当前工作目录下的 **`./.cache/<name>/`**。不存在就建。这样跨平台不污染 workspace（不用 `/home/claude`，不用 `%TEMP%`）。

**GitHub 仓库名：**
- 优先用 `gh search repos <query>`（如可用），否则用 `web_search`，查询如 `site:github.com <query>`
- 多个候选时，列出 **star 数前 3** 加一句话描述，让用户选。**不要默选**。
- 确认后浅 clone（bash 和 PowerShell 通用）：
  ```bash
  git clone --depth 1 https://github.com/<owner>/<repo>.git ./.cache/<repo>
  ```
- 私有库：让用户手动 clone 后提供本地路径。

**本地目录路径：** 直接用。

**子模块形式 (`<repo>::<subpath>`)：**
- 先按上述规则解析 `<repo>` 部分（clone / 本地路径 / 解压）
- 校验 `<subpath>` 在解析后的仓库下存在；不存在就让用户改正
- 同时带住"仓库根"和"子路径"——分析在子路径上跑，但允许向上读 1–2 级查 helper / 类型上下文（不必要不再往上）

**压缩包：** 解压到 `./.cache/<archive-basename>/`。

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
# tar.gz / tar (Win10+ 自带 tar)
tar -xzf <archive> -C ./.cache/<basename>
```

### Step 1.5 — 体量检查与阈值

对**分析目标**统计 LoC——子模块模式下是子路径，不是整个仓库。子模块通常远低于阈值，这个 gate 很少触发。

按可用 shell 选一种：

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

**LoC 超过 100,000** 时暂停并警告：

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

等用户确认。尊重用户选择。

LoC ≤ 100,000 时静默通过。

### Step 2 — 判定项目类型

主分析之前先做一次快速类型识别：读 README + 顶层结构，把项目归到七种类型之一。**不同类型走不同 outline、不同关注点列表**——这是这套设计相对固定模板的核心价值。

**判定信号**（命中 ≥ 2 项视作匹配；多类同时命中按文末优先级）：

| 类型 | 信号 |
|---|---|
| **interface-library** (接口库 / 系统库) | 主产物是 `.so` / `.a` / crate / wheel / gem，**无 main 二进制**或仅有 demo CLI；README 描述方式是"提供 X / Y / Z 能力"或"封装 syscall X"；公共 API 可独立列出（headers / `pub fn` / `__all__`）；直接依赖某底层机制（syscall、内核子系统、协议帧、设备节点）；测试以 API 调用为主 |
| **application** (应用 / 工具) | 顶层有 `main` 入口或显式二进制目标；CLI / daemon / TUI 行为；README 讲使用场景或子命令；issue 多讨论 e2e 行为 |
| **framework** (框架 / SDK) | 库形态但通过 trait / interface / abstract class 强加架构约束；明确暴露扩展点（middleware / plugin / hook）；用户代码需要"塞进"框架的生命周期 |
| **protocol** (协议实现) | 引用 RFC / draft 号；存在 wire format / conformance test；命名含 protocol / spec / codec / wire / frame |
| **research** (研究代码) | 引用论文 arXiv id / paper.bib；有 `experiments/` / `checkpoints/` / `configs/` 目录；README 主要讲方法和实验结果 |
| **monorepo** | `packages/` / `apps/` / `crates/` 下有 ≥ 3 个独立可用的子项目；根 README 是聚合性的 |
| **fallback** (兜底通用) | 以上信号都不够强，或项目混合多种特征难以归类 |

**多类同时命中时的优先级**：interface-library > protocol > framework > research > monorepo > application > fallback。

**混合形态处理**：如果项目确实是 A+B（例如 CLI 工具同时也是库），选主类型 outline，加 1–2 个次类型的章节。报告头里显式说明。

**把推断出的类型告知用户**再继续（不要阻塞）：

> "看起来这是一个 **<type>** 项目（依据：<一句话理由>），按对应的 outline 来写。如果判定错了请打断我。"

子模块模式下，类型判定跑在**子路径**上，不是母仓库——一个子目录的性格可能跟整体不同。

**判定完成后**，读取 `presets/<type>.md`（路径相对本 SKILL.md 所在目录）获取该类型的章节大纲与各档关注点。同一个项目只需读一次。混合形态时先读主类型 preset，需要的次类型章节再去读对应 preset。文件清单见下方"项目类型 preset"小节。

### Step 3 — 按类型 + 深度分析

并行做两件事：

1. **通用分析方法**——所有项目、所有深度都做。见"通用分析方法"小节。
2. **类型专属关注点**——见下方对应 preset。

深度档决定强度，不决定覆盖哪些维度：

- `quick` ——每个维度浅触（读 README、顶层结构、列公共表面或 main 入口，扫 1–2 个源文件）
- `medium` ——按类型主要章节真正深入；核对 README claim；提取 archaeology signals
- `deep` ——把 preset 中每个章节都展开；跟踪多条端到端路径或覆盖每个接口分组；涉及 ABI / 版本 / 跨平台的也写

**不做 TOP-N 评分**。挑哪些项展开（接口 / 模块 / 路径 / 扩展点）基于*这个*项目本身有意思的点，不是公式打分。如果做了取舍，**显式列出"未详写的部分"**，避免读者推断"未详写 = 不重要"。

### Step 4 — 生成报告

**输出路径：**

所有报告按仓库分文件夹存放：`./reports/<repo>/` 下集中存放该仓库的所有报告（不同深度、不同子模块的分析都落在同一个仓库目录里）。

- 仓库分析：`./reports/<repo>/analysis_<depth>.md`
- 子模块分析：`./reports/<repo>/<subpath-flattened>_analysis_<depth>.md`
  - `<subpath-flattened>`：把 `/` 和 `\` 替换为 `-`（例：`internal/pipeline` → `internal-pipeline`）
  - 示例：`./reports/goclaw/internal-pipeline_analysis_deep.md`

不存在就建 `./reports/<repo>/`。

**文件名规则（仅 ASCII）：**
- 仅用 ASCII 字符——不带 CJK / 重音 / emoji。非 ASCII 路径会在 Windows 和跨平台流水线里出问题。
- 净化项目名：去掉 `/`、`\`、`:`、`*`、`?`、`"`、`<`、`>`、`|` 和空白（替换成 `-`）。
- 报告**内容**仍按风格指南写简体中文；只有文件名是 ASCII。

#### 通用章节（始终存在，顺序如下）

1. **元数据头** ——见下方"Header"
2. **问题域** ——这个项目解决什么问题？目标用户是谁？**没有它，用户得自己做什么？** （反事实陈述是这一节的灵魂）
3. **概念与术语** ——定义列表式词汇表；见"通用小节：概念与术语"
4. **[类型专属章节]** ——按 Step 2 选定的 preset 决定
5. **协议与分发** ——许可证、商业使用限制、分发模式。一般是靠后的短小一段。**仅当**存在值得前置警告的冲突（例如"production-ready"主张 + 非商用许可）才提到前面
6. **特点与不足** ——分两小节：
   - **客观事实** ——可观测、可验证的项。无处安放的 README-vs-reality 不符和 archaeology signals 也落在这里
   - **主观评价** ——Claude 自己的判断，每条都要能追溯到上面具体的事实。不空话

**子模块模式下的维度覆盖：**
- **协议与分发**收成一行：*"继承自母仓库 `<repo>` 的 `<license>`"*。除非子模块有自己的 LICENSE / NOTICE，否则不独立成节。
- **概念与术语**仍保留，scope 限于子路径。
- 类型判定跑在子路径上，不是母仓库。
- 其他维度都在，scope 收到子模块。

#### 风格指南

**`standard` (默认)**
- 完整句子、以叙述为主
- 用 markdown 标题、表格、代码块
- 适合分享给同事
- 例：*"项目通过 Channel trait 抽象传输层，使 Unix socket 和 TCP 等传输能以统一接口接入。"*

**`concise` (速记式)**
- 电报式、便签式——删冠词和连接词
- 大量使用 `→`、`:`、`/`、bullet fragment
- 信息密度优先于对外可读性
- 例：*"Channel trait → 抽象传输层；transports（unix sock / tcp）统一接入"*
- 仍用 markdown 标题，但 bullet 占主导

#### Header

报告始终以元数据头开头：

```markdown
# <项目名> 分析报告

- **来源**: <github URL / local path / archive>
- **项目类型**: interface-library | application | framework | protocol | research | monorepo | fallback  <!-- 混合形态写 "A (主) + B (次)" -->
- **分析深度**: quick | medium | deep
- **写作风格**: standard | concise
- **生成日期**: YYYY-MM-DD
- **commit (if cloned)**: <short SHA>
```

### Step 5 — 呈现报告

写完文件后：

1. 打印**绝对路径**让用户能直接打开。用环境提供的方式解析路径——bash 上 `realpath ./reports/<file>`，PowerShell 上 `Resolve-Path ./reports/<file>`。
2. 在聊天里给 2–3 句关键发现的摘要——**不要**复述整份报告。
3. 告知用户：*"已写入 `<absolute path>`，可在编辑器中打开。"*

不要假设存在沙盒专属的呈现 helper。

## 项目类型 preset

七种类型各自的章节大纲与各档关注点拆到 `presets/` 目录下按需读取，避免每次调用都装载全部内容：

| 类型 | preset 文件 |
|---|---|
| interface-library | `presets/interface-library.md` |
| application | `presets/application.md` |
| framework | `presets/framework.md` |
| protocol | `presets/protocol.md` |
| research | `presets/research.md` |
| monorepo | `presets/monorepo.md` |
| fallback | `presets/fallback.md` |

每个 preset 包含两块：**章节大纲**（接在通用章节之后的类型专属章节）和**各档关注点**（quick / medium / deep 三档分别强调什么）。**把 outline 视作起点而非契约**——某章节项目里实在没什么可写就删掉，发现 preset 未预料到的东西就加章节。

## 通用小节：概念与术语

读后续章节前的词汇表。两类都收：
- **外部概念**：项目依赖但外部存在的技术（syscall、BPF、OCI spec、cgroup、SPI 等）
- **本项目引入的抽象**：项目自己造的词或赋予特定含义的类型（filter context、extractor、channel、layer 等）

**收录规则**：
- 3–8 条，硬上限 10。多了就成词典了
- 只收"报告读者可能不知道 / 在本项目中有特定含义"的项
- **不收**通用 CS 概念（function / module / branch / trait / class 等）
- 每条 2–3 行：是什么 + 为什么这么设计 / 非显然之处，再加一行"在本项目中"指明位置和角色

**格式（定义列表式，不是表格——密度高的内容塞不进表格单元格）**：

```markdown
## 概念与术语

**<术语>** *(外部 / 本项目引入)*
2–3 行：是什么、为什么这么设计、非显然之处。
→ 本项目：在哪里出现、扮演什么角色。

**<下一个术语>** *(外部 / 本项目引入)*
...
```

**示例（节选自 libseccomp）**：

```markdown
**seccomp-bpf** *(外部)*
Linux 内核 ≥3.5 提供的 syscall 过滤机制：进程一次性向内核安装一段 BPF 程序，之后每次发起 syscall 都先过该程序判定（allow/deny/kill/trap/log）。安装是单向的、不可撤销——这是它的安全模型基石。
→ 本项目：所有公共 API 最终都落在"生成并安装一段 BPF"。

**filter context (`scmp_filter_ctx`)** *(本项目引入)*
不透明句柄类型（`typedef void *`），调用方不能直接访问字段。一个 context 持有：规则集、默认动作、目标架构集合、属性表。所有写操作作用在 context 上，最后由 `seccomp_load` 统一编译成 BPF 并安装。
→ 几乎所有公共 API 的第一参数；生命周期：`init → arch_add* → rule_add* → load → release`。
```

## 通用分析方法

不论项目类型和深度档都做（深度档控制强度，不控制做不做）。

### README 与代码事实核对

对 README / 顶层 docs 里每个可验证的 claim，**到代码里核对**：
- 数字型 claim（"8-stage pipeline"、"20+ providers"、"5-layer security"）——数代码里实际有几个
- 列表型 claim（功能 / 平台 / 兼容版本）——一项一项找代码证据
- 性能型 claim（"10x faster"、"zero allocation"）——找 benchmark 或 micro-optimization；找不到就标"未独立验证"
- 状态型 claim（"production-ready"、"battle-tested"）——看测试覆盖、CI 配置、issue 历史

发现不符时，把它写进对应的正文章节（接口数量不符 → 写在接口分类里），而不是单独堆到"特点与不足"。**只有**找不到合适章节安放的，才放进"特点与不足 → 客观事实"。

为什么重要：很多用户从 README 了解项目、从不读代码；揭示这种 gap 是本 skill 最高价值的产出之一。

### Code archaeology signals

观察 meta-signals——它们往往比代码本身更能说明项目的开发背景、质量和风险。

| Signal | 看什么 | 可能含义 |
|---|---|---|
| **Translation comments** | `// Adapted from <other-project>`、`// Matching TS <thing>`、`// Port of <X>` | 从另一语言移植；idiom 可能不本土；质量取决于移植者水平 |
| **Idiom mismatch** | Go 里 `MemoryMB int` 而不是 `time.Duration`；语言有 `Option` 却手写 nullable 模式 | AI 辅助移植或非母语作者；fork 后改造成本 |
| **Aggressive version pins** | 语言 / 运行时 / DB 锁在最新或预发布版本 | 优先尝鲜，部署阻力；企业落地难 |
| **Scale vs attribution mismatch** | 200k+ LoC 但只有单作者或"X Contributors"泛指 | 重度 AI 辅助 / 影子团队 / 历史合并；质量在不同模块间差距大 |
| **Bundled third-party assets without attribution** | vendored 的 skills / data / configs 来自具名项目但 LICENSE / NOTICE 无致谢 | 若 redistribute 有合规风险 |
| **README hyperbole vs code reality** | "8-stage" 但代码 7 个；"20+ providers" 但实际 12；"production-tested" 无可观测性配置 | 营销驱动的 docs；信代码不信 README |
| **Localization theater** | 30 种语言 README 但核心 docs 只 1 种 | 为 star 数优化，不是为 maintenance |
| **Test/code organization** | 分层测试（contracts / integration / invariants / scenarios）vs 一个 `tests/` 堆 | 结构化测试 → 工程成熟度信号 |
| **Commit cadence and authorship** | 长时间空档、单作者爆发、AI 风格 commit message | 可持续性信号 |

发现这些信号时按以下原则落地：
- 信号本身（"BPF 编译器是手写 C"）→ "客观事实"
- 它意味着什么（"维护成本高，但 zero 第三方依赖"）→ "主观评价"

### 概念抽取

读 README 和顶层结构时同步收集候选术语，按"概念与术语"收录规则筛选。优先收：项目反复出现的大写类型名、README 中加粗或代码 fence 起来的名词、依赖列表中需要解释才能让读者理解为什么需要它的库。

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

- **不透明句柄**：`scmp_filter_ctx` 被 typedef 到 `void *`，调用方拿到也无法访问字段——版本演进时内部结构可任意改。代价是 debugger 里看不到内容，需要 `seccomp_export_pfc` 转储。
- **错误模型**：公共 API 返回 `int`，0 成功，负值是 `-errno`。没有全局 errno 污染；但用户需要记得取反才能传给 `strerror()`。
- **生命周期**：单写多读语义——`seccomp_load` 后 context 进入只读阶段，再调写操作返回 `-EBUSY`。
- **线程安全**：单个 context 不是线程安全的（写操作需要外部加锁）；但加载后的 filter 自然作用于整个进程的所有线程。
- **跨架构**：同一份规则可以同时编译到 x86_64 + arm64 + i386，`seccomp_arch_add` 增加目标架构后规则会被多次实例化。

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
- **runc/crun**：在 init 阶段（容器命名空间已建立、exec 用户进程之前）从 `config.json` 的 seccomp profile 加载规则
- **systemd**：在 unit 启动时通过 `SystemCallFilter=` 配置项，systemd 内部调用 libseccomp 构建规则
- **Bubblewrap / Flatpak**：在 sandbox 启动器中加载预设策略

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

- 项目过大无法在请求深度下完整分析时，仍然出报告，但在元数据头里注明哪些区域是抽样、哪些是完整覆盖。
- source 解析到多个顶层目录但未被判定为 monorepo 时，问用户是分析整个 repo 还是某个子项目。
- 用户给的名字在 GitHub 搜索无结果时，问是否拼写有误，或是否直接提供 URL / 本地路径。
- 不编。无法判断的就在报告里明说（例如"无 benchmark，性能特性不做断言"）。
