# scientific-agent-skills 源码学习笔记

> 仓库地址：[K-Dense-AI/scientific-agent-skills](https://github.com/K-Dense-AI/scientific-agent-skills)
> 学习日期：2026-05-24

---

> **以下为 AI 源码分析**
>
> ### 一句话概括
>
> 由 K-Dense 维护的「Agent Skills 标准」科研技能库——把 138 个跨学科科研工作流（生信、药化、临床、医学影像、量子化学、天文物理、地理空间、写作出版等）沉淀为可被 Cursor / Claude Code / Codex 等通用 Agent 直接消费的 `SKILL.md` 资产，并配套两个轻量 Python CLI（`scan_skills.py` / `scan_pr_skills.py`）以及三个 GitHub Actions 工作流，把「LLM 安全扫描 + PR sticky comment + 自动发版」串成可持续运营的开放治理闭环。
>
> ### 要点速览
>
> | 模块 | 职责 | 关键文件 / 目录 |
> |------|------|-----------------|
> | `scientific-skills/` | Skill 资产仓库，每个子目录是一个独立可分发的 Skill | 139 个子目录，每个内含 `SKILL.md` + 可选 `scripts/` `references/` `assets/` `templates/` `tests/` |
> | `scan_skills.py` | 全量 Skill 安全扫描器，生成仓库级 `SECURITY.md` | `build_scanner` / `scan_with_progress` / `generate_report` |
> | `scan_pr_skills.py` | PR 级增量扫描，输出 sticky PR comment 并按阈值阻断合入 | `scan_skill_dirs` / `format_comment` / `_should_block` |
> | `.github/workflows/` | CI/CD：PR 安全扫描、每周全量扫描、按版本号自动发 release | `pr-skill-scan.yml` / `security-scan.yml` / `release.yml` |
> | `autoskill` (代表性 skill) | 通过 screenpipe 观察用户屏幕、聚类工作流、生成新 skill 草案——是 skill 创作元能力的样板 | `scientific-skills/autoskill/` 下的 11 个脚本 + tests |
> | `database-lookup` (代表性 skill) | 把 78 个公共科研数据库的 REST API 用「单一 Skill + 78 份 reference 文档」的方式统一暴露 | `scientific-skills/database-lookup/SKILL.md` + `references/*.md` |
> | `docs/` & `README.md` | 用户向文档：安装、使用、示例工作流、贡献指南 | `docs/scientific-skills.md` 138 个 skill 的目录索引 |
> | `pyproject.toml` + `uv.lock` | 仅声明扫描器自身依赖（`cisco-ai-skill-scanner` / `firecrawl-py` / `python-dotenv`），跑 skill 用各自 `SKILL.md` 内的 uv 命令 | Python ≥ 3.13 |

---

## 项目简介

scientific-agent-skills 不是一个会运行的应用，而是一个 **「Agent 能力分发仓库 + 自动化治理工具链」**。它面向「希望让 AI 编程 Agent 变成 AI 科研助理」的研究人员，把 138 个跨学科的科研工作流沉淀为遵循 [Agent Skills](https://agentskills.io/) 开放标准的 Skill 资产；每个 Skill 是一份带 YAML 前置元数据的 `SKILL.md`（描述「什么时候触发」「怎么用」「依赖什么 Python 包 / API / 凭证」），以及围绕它的 `scripts/`（可执行 Python 脚本）、`references/`（密集知识参考文档）、`assets/`（模板与代码示例）、可选的 `templates/` 和 `tests/`。

仓库自身的代码量很小，核心创新是 **治理闭环**：

1. 每个 Skill 都会被 Cisco AI Defense 的 `skill-scanner` 用 LLM 行为分析、触发条件分析、跨 Skill 重叠检测三类分析器扫描，输出一份带严重等级的 `SECURITY.md` 报告。
2. PR 改动到任意 Skill 时，CI 通过 `git diff --name-only` 解析出受影响的 Skill 目录，只增量扫描这些目录并以 sticky comment 形式回写到 PR；扫到 HIGH 以上严重等级直接阻断合入。
3. 每周一 09:00 UTC 重新跑全量扫描并提交 `SECURITY.md`；每次 `pyproject.toml` 版本号变化触发自动 release。

这样既保证了「众包式 Skill 贡献」的开放性，又把「Skill 可以执行任意代码」这种风险锁在可观测、可回滚、可度量的治理框架里。

## 技术栈

| 类别 | 技术 |
|------|------|
| 语言 | Python 3.13（仓库自身脚本）；各 Skill 内部使用 Python 为主，少量 LaTeX / Bash |
| 框架 | 无服务端框架；Skill 标准遵循 [agentskills.io](https://agentskills.io/) 公约（YAML frontmatter + `SKILL.md`） |
| 构建工具 | [uv](https://docs.astral.sh/uv/)（仓库 CI 与各 Skill 推荐的依赖安装器） |
| 依赖管理 | `pyproject.toml` + `uv.lock`（仓库根级仅 3 个直接依赖：`cisco-ai-skill-scanner`、`firecrawl-py`、`python-dotenv`） |
| 测试框架 | `pytest`（部分 skill 自带 `tests/`，例如 `autoskill/tests/` 含 13 个测试模块） |
| 关键外部服务 | 安装通道 `npx skills add` / `gh skill install`；扫描器 LLM 默认 `claude-sonnet-4-6`；CI 用 GitHub Actions + `marocchino/sticky-pull-request-comment@v2` + `softprops/action-gh-release@v2` |

## 目录结构

```
scientific-agent-skills/
├── README.md                       # 用户视角主入口（770 行，安装 / 示例 / 治理说明）
├── SECURITY.md                     # 自动生成的全量安全扫描结果（每周一刷新）
├── pyproject.toml                  # Python ≥3.13；仓库根脚本的依赖
├── uv.lock                         # uv 锁文件
├── scan_skills.py                  # 【核心】全量 Skill 安全扫描器
├── scan_pr_skills.py               # 【核心】PR 增量扫描，复用 scan_skills.build_scanner
├── docs/
│   ├── scientific-skills.md        # 138 个 skill 的分类目录与一句话简介
│   ├── examples.md                 # 跨 skill 组合的端到端工作流示例（3043 行）
│   └── open-source-sponsors.md     # 上游开源项目致谢
├── .github/workflows/
│   ├── pr-skill-scan.yml           # PR 触发：检测改动的 skill → uv sync → scan_pr_skills.py → sticky 评论
│   ├── security-scan.yml           # 周一 09:00 UTC + 手动：scan_skills.py → 提交 SECURITY.md
│   └── release.yml                 # pyproject.toml 改动时按版本号打 tag、生成 release notes
└── scientific-skills/              # 【主资产】139 个 skill 目录
    ├── autoskill/                  # 元能力 skill：观察屏幕→聚类→生成新 skill 草案
    │   ├── SKILL.md
    │   ├── config.yaml             # backend: local|claude|foundry，clustering 参数，redaction 开关
    │   ├── scripts/                # 11 个 Python 模块（fetch_window/cluster/redact/synthesize/promote/...）
    │   ├── references/             # screenpipe 配置示例 + HTTPS 代理指南
    │   └── tests/                  # 13 个 pytest 模块
    ├── database-lookup/            # 单 skill 暴露 78 个公共数据库
    │   ├── SKILL.md                # 长形 prompt + 数据库选择表
    │   └── references/             # 78 份 *.md，每份描述一个数据库的端点 / 查询格式 / 示例
    ├── scanpy/                     # 经典「Python 包封装」型 skill 模板
    │   ├── SKILL.md
    │   ├── assets/analysis_template.py
    │   ├── references/{api_reference,plotting_guide,standard_workflow}.md
    │   └── scripts/qc_analysis.py
    ├── literature-review/          # 综合型工作流 skill（链式调用其他 skill）
    ├── markdown-mermaid-writing/   # 含 templates/ 的写作类 skill
    ├── citation-management/ ...    # 其他 134 个 skill，结构类似
```

> 各 Skill 的内部布局并不强制统一，但绝大多数遵循 `SKILL.md` + `references/` + `scripts/` + `assets/` 的四件套约定。`autoskill` 因其「写新 skill」的元能力，是少数携带 `tests/` 与 `config.yaml` 的 skill。

## 架构设计

### 整体架构

仓库本质上是 **「内容资产层 + 治理工具链 + 分发通道」** 的三段式组合，没有运行时服务，所有「执行」都发生在用户侧 Agent（Cursor / Claude Code / Codex …）加载 Skill 之后。

```mermaid
flowchart TD
    subgraph Contributor["贡献者侧"]
        DEV[研究员/开发者]
        PR[Pull Request]
    end

    subgraph Repo["scientific-agent-skills 仓库"]
        SKILLS[scientific-skills/<br/>139 个 SKILL.md 资产]
        SCAN_FULL[scan_skills.py<br/>全量扫描]
        SCAN_PR[scan_pr_skills.py<br/>增量扫描]
        WF1[".github/workflows/<br/>pr-skill-scan.yml"]
        WF2[".github/workflows/<br/>security-scan.yml"]
        WF3[".github/workflows/<br/>release.yml"]
        SECURITY[SECURITY.md]
    end

    subgraph Distribution["分发通道"]
        NPX["npx skills add"]
        GH["gh skill install"]
    end

    subgraph User["最终用户侧 Agent 运行时"]
        AGENT[Cursor/Claude Code/Codex]
        DISCOVER[Skill 自动发现]
        EXEC[Skill 执行<br/>SKILL.md+scripts/+references/]
    end

    DEV --> PR --> SKILLS
    PR -.触发.-> WF1 --> SCAN_PR
    SCAN_PR -.sticky comment.-> PR
    WF2 -.周一 cron / 手动.-> SCAN_FULL --> SECURITY
    SCAN_FULL --> SKILLS
    WF3 -.pyproject 变更.-> Repo

    SKILLS --> NPX & GH
    NPX & GH --> AGENT --> DISCOVER --> EXEC
```

设计意图：

1. **资产与运行时解耦**：仓库不维护任何 server/agent 代码，所有 Skill 都是「文档 + 脚本」的纯文本资产，由用户侧的 Agent 在自己的进程里加载执行。这使仓库可被任意符合 Agent Skills 标准的 Agent 复用，避免和某一个 Agent 实现绑死。
2. **CI 即治理**：仓库不需要服务器持续运行，只在 GitHub Actions 里以 cron + PR 事件触发扫描；扫描结果直接写回仓库（`SECURITY.md` 周更）或写回 PR（sticky 评论），把治理动作沉淀为公开可追溯的版本历史。
3. **Skill 内部结构社区化**：仓库不强制 Skill 必须包含哪些文件，仅约定 `SKILL.md` 与 frontmatter 必填字段。这降低了贡献门槛，但代价是要靠 `skill-scanner` 的行为分析与触发条件分析在 PR 阶段把控质量。

### 核心模块

#### 1. `scientific-skills/` — Skill 资产层

- **职责**：承载 138 个领域 Skill + 1 个 `autoskill` 元 skill。每个 Skill 自闭环，声明触发条件、外部依赖、关键脚本。
- **典型结构**（以 `scanpy/` 为代表）：
  - `SKILL.md`：YAML frontmatter（`name` / `description` / 可选 `allowed-tools` / 可选 `license` / `metadata.skill-author`）+ Markdown 正文（Overview / When to Use / Quick Start / Workflow / API 速览）。
  - `references/*.md`：稠密知识参考，避免把所有信息塞进一个超长 `SKILL.md`，由 Agent 在需要细节时按文件名再读取。
  - `scripts/*.py`：可执行脚本（如 `qc_analysis.py`），由 Agent 调起执行。
  - `assets/`：可复用模板（如 `analysis_template.py`）。
- **特殊形态**：
  - `database-lookup/` — 单 Skill 内置 78 份 `references/<db>.md`，把「78 个数据库」浓缩为「一个 Skill + 一张选择表」，形成超大 prompt-as-router 模式。
  - `autoskill/` — 携带 `config.yaml`、`tests/`、可执行 CLI（`scripts/autoskill.py` 暴露 `run` / `doctor` / `promote` 三子命令），是「写 Skill 的 Skill」。

#### 2. `scan_skills.py` — 全量安全扫描器

- **职责**：扫描整个 `scientific-skills/` 目录，落地 `SECURITY.md`。
- **关键函数**：
  - `build_scanner()`（L26-43）：构造 `SkillScanner`，配置 `BehavioralAnalyzer`（行为模式）、`TriggerAnalyzer`（描述/触发分析）、`LLMAnalyzer`（LLM 风险研判，默认 `anthropic/claude-sonnet-4-6`），并通过 `ScanPolicy.from_preset("balanced")` 设置分析预算（`max_*_chars`）。
  - `scan_with_progress()`（L125-202）：自定义遍历 `base.rglob("SKILL.md")` 的父目录列表，逐个调用 `scanner.loader.load_skill` + `scanner._scan_single_skill`，按宽度对齐打印类似 `[ 12/138] scanpy   🟢 SAFE   (3.2s)` 的进度行，最后调用 `CrossSkillScanner` 跑跨 Skill 描述重叠检测。
  - `generate_report()`（L58-122）：把 `Report` 渲染成两段 Markdown——表格汇总 + 仅列出有 finding 的详细块，使用 emoji 严重等级（🔴/🟠/🟡/🔵/⚪/🟢）做视觉锚点。
- **运行依赖**：`SKILL_SCANNER_LLM_API_KEY`（GitHub secret）+ `SKILL_SCANNER_LLM_MODEL` env（默认 `claude-sonnet-4-6`）。

#### 3. `scan_pr_skills.py` — PR 增量扫描

- **职责**：在 PR 里只扫改到的 Skill 目录、生成 sticky comment、按严重等级阈值决定是否阻断合入。
- **关键函数**：
  - 直接 `from scan_skills import build_scanner, severity_badge`，复用全量扫描器的构造逻辑。
  - `scan_skill_dirs()`（L42-87）：对显式传入的 `skill_dirs` 列表逐个扫描；同时若 ≥2 个被扫还会跑跨 Skill 描述重叠分析。
  - `format_comment()`（L99-168）：渲染前置 `<!-- skill-security-scan -->` 标记（让 sticky-comment action 能定位到旧评论复用），用 `<details>` 折叠每个 Skill 的 finding 详情。
  - `_should_block()`（L171-179）：按 `--fail-on` 等级在 `SEVERITY_ORDER` 里截断，命中则非零退出。

#### 4. `.github/workflows/` — CI/CD 编排

- `pr-skill-scan.yml`：
  - `pull_request` + paths filter（仅 `scientific-skills/**` 等触发）。
  - 用 `git diff --name-only --diff-filter=ACMR base..head -- 'scientific-skills/**'` 提取改动文件，再 `awk -F/ '... print $1 "/" $2'` 取一级 skill 目录，最后 `[ -f "$d/SKILL.md" ]` 过滤已删 / 非 skill 路径。
  - `astral-sh/setup-uv@v8.0.0` + `uv sync --python 3.13` 安装依赖；`uv run python scan_pr_skills.py --fail-on HIGH ...` 跑扫描。
  - `marocchino/sticky-pull-request-comment@v2` 用 `header: skill-security-scan` 复用同一条 PR 评论。
- `security-scan.yml`：周一 09:00 UTC cron + `workflow_dispatch`；`uv run python scan_skills.py`；`git diff --quiet SECURITY.md && exit 0` 判断空变更避免空提交，否则用 `github-actions[bot]` 身份提交。
- `release.yml`：仅在 `pyproject.toml` 变更时跑；用 `grep '^version' pyproject.toml | sed 's/.*"\(.*\)".*/\1/'` 抽版本号，与 `git rev-parse v$VERSION` 比对存在性，未存在则用 `git log $PREVIOUS_TAG..HEAD --pretty=format:"* %s (%h)" --no-merges` 生成 release note，最后 `softprops/action-gh-release@v2` 打 tag。

#### 5. `autoskill/` — 元能力 Skill（推荐重点学习）

- **职责**：让 Agent 反过来「自己创造 Skill」——用户运行 `autoskill run --start <ts> --end <ts>`，它通过 [screenpipe](https://github.com/screenpipe/screenpipe) 本地 daemon 拉取屏幕 OCR + 窗口标题，做 session 切分 + 应用维度聚类 + 与现有 skills 做 embedding 相似度匹配，最终让 LLM 决定每个聚类是 `reuse` / `compose` / `novel` 并起草 SKILL.md。
- **核心脚本与函数**：
  - `scripts/fetch_window.py:fetch_window`：分页拉 `screenpipe` 的 `/search` 接口，bounded-loop（`_MAX_PAGES = 10_000`）防失控，把每条事件归一化为 `{ts, app, window_title, text, content_type}`。
  - `scripts/redact.py:redact`：在数据离开本机前先做正则脱敏——私钥块、`AWS_*/ANTHROPIC_API_KEY/...` 等已知 env、Bearer token、JWT、各家 API key 前缀（`xox*` / `hf_*` / `sk-*` / `ghp_*` / `AKIA*` / `AIza*`）、邮箱、电话、SSN，按从长到短的顺序匹配。
  - `scripts/cluster.py`：`segment_sessions` 用「`idle_gap_seconds` 切段 + `min_session_seconds` 过滤」分会话；`cluster_sessions` 按 `tuple(s["apps"])` 哈希分桶聚类，要求 `min_cluster_size ≥ 2`。
  - `scripts/match_skills.py`：自己写 `_parse_frontmatter` / `_cosine`，避免引入 `pyyaml` / `numpy`，靠 `embedder` 函数 + `top_k_matches` 跟现有 skill 描述做余弦相似度。
  - `scripts/synthesize.py:synthesize`：构造一个固定 prompt 给 LLM——「在 reuse / compose / novel 中选其一，用单个 JSON 对象返回」，再用 `_extract_json` 兼容 fenced 与无 fence 两种返回格式。
  - `scripts/backends.py`：`ClaudeBackend`（POST `https://api.anthropic.com/v1/messages`）+ `LocalBackend`（OpenAI 兼容的 `/chat/completions`，用于 LM Studio）+ `FoundryBackend`（企业网关）。
  - `scripts/run.py:run`：编排上面所有步骤，把结果写到 `~/.autoskill/proposed/<ts>/`，包括 `report.md` 与 `new-skills/<name>/SKILL.md` / `composition-recipes/<name>/SKILL.md` 草案。
  - `scripts/promote.py`：人工审阅后用 `autoskill promote --name <skill>` 把草案 `shutil.move` 到 `scientific-skills/`，target 已存在则拒绝以防覆盖。
  - `scripts/doctor.py`：`autoskill doctor` 一键体检 `screenpipe` `/health` + LLM 后端可达性 + skills 目录存在性 + config 合法性。
- **设计哲学**：检测/聚类/embedding **永远在本地**；只有「已经被聚合并脱敏的 cluster summary」才进 LLM；LLM 后端默认 `local`（LM Studio + Gemma-4-31B-it），云后端是 opt-in。

### 模块依赖关系

```mermaid
graph LR
    SKILLS["scientific-skills/<br/>139 SKILL.md 资产"]
    SCAN[scan_skills.py]
    PR[scan_pr_skills.py]
    SCANNER["cisco-ai-skill-scanner<br/>(外部包)"]
    LLM["LLM API<br/>claude-sonnet-4-6 默认"]
    WF1[pr-skill-scan.yml]
    WF2[security-scan.yml]
    WF3[release.yml]
    SECURITY[SECURITY.md]
    PROJECT[pyproject.toml]

    SCAN -->|build_scanner| SCANNER
    PR -->|import build_scanner| SCAN
    PR -->|import severity_badge| SCAN
    SCANNER -->|LLMAnalyzer| LLM
    SCAN -->|遍历| SKILLS
    PR -->|增量传参| SKILLS
    WF1 -->|uv run| PR
    WF2 -->|uv run| SCAN
    WF2 -->|git commit| SECURITY
    SCAN --> SECURITY
    WF3 -->|按版本号| PROJECT
    PROJECT -->|声明依赖| SCANNER
```

`autoskill` skill 的内部依赖是单独的微型 DAG（仓库主代码不调用它）：

```mermaid
graph TD
    CLI[autoskill.py CLI dispatcher]
    RUN[run.py]
    DOC[doctor.py]
    PROM[promote.py]
    FETCH[fetch_window.py]
    RED[redact.py]
    CLU[cluster.py]
    MATCH[match_skills.py]
    SYN[synthesize.py]
    BK[backends.py]
    SP[(screenpipe<br/>localhost:3030)]
    LLM2[(LLM backend<br/>local/claude/foundry)]

    CLI --> RUN & DOC & PROM
    RUN --> FETCH --> SP
    RUN --> RED
    RUN --> CLU
    RUN --> MATCH
    RUN --> SYN --> BK --> LLM2
    DOC --> SP
    DOC --> BK
    PROM --> SKILLS2[(scientific-skills/)]
```

## 核心流程

### 流程一：PR 改动了某个 Skill 后的增量安全扫描

下面这条调用链每天在每个 PR 上跑数次，是仓库治理体系最核心的「热路径」。

```mermaid
sequenceDiagram
    autonumber
    participant DEV as 贡献者
    participant GH as GitHub
    participant CI as Actions Runner
    participant SCAN as scan_pr_skills.py
    participant SCAN2 as scan_skills.build_scanner
    participant SK as cisco-ai-skill-scanner
    participant LLM as Claude Sonnet 4.6

    DEV->>GH: push commits to PR (修改 scientific-skills/foo/SKILL.md)
    GH->>CI: 触发 pr-skill-scan.yml (paths 命中)
    CI->>CI: git diff --name-only base..head -- 'scientific-skills/**'
    CI->>CI: awk 解析顶层目录 → 过滤掉无 SKILL.md 的项
    CI->>CI: uv sync --python 3.13
    CI->>SCAN: uv run python scan_pr_skills.py --fail-on HIGH foo
    SCAN->>SCAN2: build_scanner() 构造 BehavioralAnalyzer + TriggerAnalyzer + LLMAnalyzer
    SCAN->>SK: scanner.loader.load_skill(foo)
    SK->>SCAN: Skill 对象（解析 frontmatter / 收集脚本 / 提示文本）
    SCAN->>SK: scanner._scan_single_skill(skill, foo)
    SK->>LLM: 把 Skill 文本 + 启发式特征拼成 prompt
    LLM-->>SK: findings (rule_id / severity / remediation)
    SK->>SCAN: ScanResult
    SCAN->>SK: CrossSkillScanner.analyze_skill_set([...]) (>=2 个 skill 时)
    SCAN->>CI: 写 pr_scan_comment.md
    SCAN->>SCAN: _should_block(report, "HIGH")
    alt 命中 HIGH/CRITICAL
        SCAN-->>CI: exit 1
        CI-->>GH: PR check 红色，阻断合入
    else 仅 LOW/INFO/SAFE
        SCAN-->>CI: exit 0
    end
    CI->>GH: marocchino/sticky-pull-request-comment@v2 (header=skill-security-scan)
    GH-->>DEV: PR 评论自动覆盖更新
```

关键点说明：

- **「只扫改动」的实现**：CI 第 3 步用 `git diff` + `awk -F/ 'NF>=2 && $1=="scientific-skills"' | sort -u` 把 `scientific-skills/foo/scripts/x.py` 这种深路径折叠回 `scientific-skills/foo`，再确认 `SKILL.md` 仍在（避免扫已删 skill 报错）。
- **PR 评论复用机制**：`format_comment` 第一行硬写 `<!-- skill-security-scan -->`；`marocchino/sticky-pull-request-comment@v2` 通过 `header: skill-security-scan` 找到旧评论并 in-place 更新，避免每次 push 灌评论。
- **跨 skill 重叠分析的触发条件**：仅当一次扫描里 `len(loaded_skills) > 1` 时才跑（见 `scan_pr_skills.py:68`），单 skill PR 跳过省时间。
- **失败语义**：`--fail-on HIGH` 是默认 PR 阈值，比全量扫描更严格；周更扫描 `scan_skills.py` 不阻断、只汇总报告。

### 流程二：autoskill 把用户屏幕活动转成 Skill 草案

这是仓库里逻辑最长、最值得学习的单条业务流，演示了「Local-first 隐私保护 + Embedding 路由 + LLM 结构化输出」三件套。

```mermaid
sequenceDiagram
    autonumber
    participant U as 用户
    participant CLI as autoskill CLI
    participant RUN as run.run()
    participant SP as screenpipe daemon
    participant RED as redact.py
    participant CLU as cluster.py
    participant MAT as match_skills.py
    participant SYN as synthesize.py
    participant BK as backends.py
    participant LLM as 本地/云 LLM
    participant FS as proposed/<ts>/

    U->>CLI: autoskill run --start ... --end ... [--dry-run]
    CLI->>RUN: 加载 config.yaml + 构造 httpx.Client / SentenceTransformer
    RUN->>SP: GET /search (Bearer SCREENPIPE_TOKEN, 分页 limit=50)
    SP-->>RUN: 屏幕事件列表 [{ts, app, window_title, text, ...}]
    RUN->>RED: 对每条 event 的 text / window_title 做正则脱敏
    RED-->>RUN: 仅含占位符的清洗后事件
    RUN->>CLU: segment_sessions(events, idle_gap=10min, min_session=5min)
    CLU-->>RUN: sessions (按时间间隔切段)
    RUN->>CLU: cluster_sessions(sessions, min_cluster_size=2)
    CLU-->>RUN: clusters (按 apps 元组分桶 + 重复出现 ≥2 次)
    alt --dry-run
        RUN->>FS: 写 plan.md，结束
    else 正常
        RUN->>MAT: load_skill_descriptions(scientific-skills/)
        MAT-->>RUN: [{name, description}, ...] (139 项)
        loop 每个 cluster
            RUN->>MAT: top_k_matches(cluster_query, skills, embedder, k=5)
            MAT-->>RUN: top_5 候选 skill (含余弦分)
            RUN->>SYN: synthesize(cluster, top_k, backend)
            SYN->>BK: backend(prompt: 选 reuse|compose|novel + 给出 JSON)
            BK->>LLM: HTTP POST /v1/messages 或 /chat/completions
            LLM-->>BK: 响应文本
            BK-->>SYN: text
            SYN->>SYN: _extract_json (兼容 ```json``` 与无 fence)
            SYN-->>RUN: {verdict, target?, name?, skill_body?}
            alt verdict=reuse
                RUN->>FS: 仅记录 target
            else verdict=compose / novel
                RUN->>FS: 写 proposed/<ts>/{new-skills|composition-recipes}/<name>/SKILL.md
            end
        end
        RUN->>FS: 写汇总 report.md
    end
    U->>CLI: autoskill promote --name <name>
    CLI->>FS: shutil.move 草案 → scientific-skills/<name>/
```

设计要点：

- **Bounded loop 防失控**：`fetch_window.py` 用 `for _page in range(_MAX_PAGES = 10_000)` 而不是 `while True`，即便 screenpipe 分页元数据错乱也不会无限循环（注释里直接说 "bounded exit: hard ceiling so the loop cannot spin forever"）。
- **脱敏顺序很关键**：`redact.py:_PATTERNS` 注释明确写「Order matters: multi-line and prefixed patterns run before narrower ones」，先匹配 `-----BEGIN PRIVATE KEY-----` 这类大段、再 `Bearer xxx` / JWT、最后窄正则（邮箱、电话、SSN）。这种「自外而内」的优先级避免局部正则把整段密钥切碎成可重组的片段。
- **Embedding 在本地、LLM 用云端是 opt-in**：`match_skills.py` 的 cosine 计算自己手写避免引 numpy，embedding 模型用 `sentence-transformers/all-MiniLM-L6-v2`（~80MB，本地跑）。LLM 调用走 `backends.py`，默认 `LocalBackend` → LM Studio (`http://localhost:1234/v1`)，让用户的「数据 + 模型」都不离开本机。
- **结构化输出兜底**：`synthesize._extract_json` 同时兼容 `\`\`\`json {...} \`\`\`` 与裸 `{...}` 两种返回，找不到 JSON 就抛 `SynthesisError`——比直接 `json.loads(text)` 更稳，但并不放纵 LLM 给非 JSON 输出。
- **Promote 是人工守门**：`promote.py` 不会自动把草案合入，要求用户显式跑 `autoskill promote --name <skill>`，且 `target.exists()` 时直接 `PromoteError`，禁止覆盖现有 skill。

## 关键设计亮点

### 1. 把「Skill 仓库」做成了纯文档资产仓 + CI 治理，而不是 server

- **解决了什么问题**：Agent Skills 标准要求 Skill 跨 Agent 复用；如果维护者把 Skill 绑到自己 host 的服务上，立刻退化为「私有 SDK」，就再也复用不动。
- **具体实现**：仓库根目录 **没有任何 server 入口**，所有运行时全在用户侧 Agent；治理动作（扫描 / 发版 / 评论）100% 由 GitHub Actions 跑（见 `.github/workflows/*.yml`）。`pyproject.toml` 仅 3 个直接依赖，是供 Actions 装扫描器用的，不是给最终用户的运行时。
- **为什么这样设计**：让 Skill 资产具备最强的可移植性——用户可以 `git clone` 拷一份本地用、可以 `npx skills add` / `gh skill install` 远程拉、企业可以 fork 后接自己的 LLM 后端（看 `autoskill/config.yaml` 里 `backend: local|claude|foundry` 三选一），仓库自身不增加任何运行依赖。

### 2. 把「质量治理」也变成 LLM-driven，并用 sticky comment + 阈值阻断闭环

- **解决了什么问题**：138 个 Skill 来自不同贡献者，纯人工 review 顶不住，但 Skill 又能让 Agent 跑任意代码，安全风险极高。
- **具体实现**：`scan_skills.py:build_scanner` 用 `BehavioralAnalyzer + TriggerAnalyzer + LLMAnalyzer` 三个分析器叠加，prompt 预算用 `ScanPolicy.from_preset("balanced")` 里的 `max_*_chars` 限上限（避免单个超大 skill 把 prompt 撑爆）；`scan_pr_skills.py:_should_block` + `pr-skill-scan.yml` 的 `--fail-on HIGH` 把「评分」直接绑定到「PR 是否合入」上；sticky comment 配合 `<!-- skill-security-scan -->` 标记保证一个 PR 只有一条评论持续刷新，不灌噪声。
- **为什么这样设计**：把 LLM 用在最适合它的场景——「自然语言描述 + 行为模式 + 跨 skill 语义重叠」这种规则难以穷举的判断；同时把可观测性（`SECURITY.md` 周更）、可阻断性（`exit 1`）、可追溯性（PR 评论历史 + git 提交）做齐，治理压力可量化、可分摊。

### 3. `database-lookup` 用「单 Skill + 78 份 reference」实现 prompt-as-router

- **解决了什么问题**：78 个数据库分别开 Skill 会污染 Agent 的 Skill 选择空间（每次都要在 78+ 选项里挑一个），还会让 SKILL.md 注册成本爆炸。
- **具体实现**：`scientific-skills/database-lookup/SKILL.md` 在 frontmatter 里把 78 个数据库**一并塞进 description**（数百字的关键字 cloud），让 Agent 在「触发判断」时即可命中；正文用大表 `用户问什么 → 主推数据库 → 备选数据库` 把语义路由展开；细节都丢到 `references/<db>.md` 让 Agent 按需再读，不主动塞进上下文。
- **为什么这样设计**：Agent Skills 标准里 `description` 字段是触发匹配的关键，77+ 个独立 skill 互相挤压描述空间会出现「`uniprot` 和 `chembl` 选哪个」式的混乱。集中后只剩一个一级路由器 + 78 个二级文档，路由判断准确率上去了，文档维护也变成「加一行表格 + 一份新 md」的小手术。

### 4. autoskill 的「检测在本地、合成在云端、兜底在人工」分层信任模型

- **解决了什么问题**：「让 Agent 看用户屏幕」是隐私敏感度 10 分满的操作；如果整个流程都丢给云 LLM，没人敢用。
- **具体实现**（见 `autoskill/scripts/`）：
  - 抓取（`fetch_window.py`）和聚类（`cluster.py`）100% 本地，原始 OCR 永远不出机；
  - `redact.py:_PATTERNS` 在数据离开本机前再扫一遍 secret，防御纵深；
  - `match_skills.py` 用本地 sentence-transformer 做 embedding 路由，LLM 只看「`apps + 重复次数 + 几条已脱敏的 window title + top-5 候选 skill 名`」这种几百字的 cluster summary；
  - `promote.py` 强制人工审批后才把草案合入仓库，且禁止覆盖。
- **为什么这样设计**：把成本和信任度反着配——计算便宜的本地能干的（OCR、聚类、embedding）就不丢给昂贵又敏感的云 LLM；最终落地动作（合入仓库）则反过来交给最贵但最可信的人工，实现「云只看摘要、本地藏全量、写入靠人手」的三层风险隔离。

### 5. CI 里的 robust 实现细节，值得 copy 到自己的项目

- **`pr-skill-scan.yml` 用 path filter + concurrency cancel-in-progress**：仅当 `scientific-skills/**` 等关键路径变动时才跑扫描；同 PR 多次 push 自动取消旧 run（`group: pr-skill-scan-${{ github.event.pull_request.number }}` + `cancel-in-progress: true`），避免 LLM 调用浪费。
- **`security-scan.yml` 的空提交防御**：`git diff --quiet SECURITY.md && exit 0` 在 commit 前判断有无变化，没变化直接 0 退出，不会污染 git history。
- **`release.yml` 的版本号幂等**：`git rev-parse "v${VERSION}" >/dev/null 2>&1` 检查 tag 是否已存在，存在就 skip 而不是失败重试。
- **`scan_skills.py` 的进度条用 ANSI `\r` 回写**：先打印 `[12/138] foo ...`，再用 `\r  [12/138] foo (padded)  🟢 SAFE  3.2s` 同行覆盖；既能在 GitHub Actions 终端里看实时进度，又不会刷屏。
- **`scan_pr_skills.py` 的「nothing to scan 也写一条占位评论」**：`if not scan_targets: format_comment(Report(), [])` —— 即便 PR 没改 skill 仍写一条「nothing to scan」comment，配合 sticky 机制让旧的扫描结果被透明覆盖，避免「上次有红条这次没人更新」的悬挂状态。
