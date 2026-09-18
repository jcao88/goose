# goose vs Claude Cowork vs Claude Code vs DeepSeek Harness：通用 Agent 能力全景对比（同模型假设）

> 调研日期：2026-09-18
> goose 依据：本仓库源码 v1.37.0（commit `0ab4b84`），文中文件路径均可在仓库中核对
> Claude Code 依据：code.claude.com 官方文档（本会话可直接抓取），版本参考 v2.1.275（2026-09-17）
> Cowork 依据：Anthropic 公开资料与媒体/安全研究报道（本会话网络代理禁止直接抓取 anthropic.com / support.claude.com，因此来自搜索摘要，关键结论已多源交叉）
> DeepSeek Harness 依据：`deepseek-ai/deepseek-harness` 源码克隆（v0.1.6-alpha.2，commit `ddefc45`，2026-09-17），含 `docs/`、`packages/*/README.md`、生成的工具目录与配置目录；官方文档站与媒体报道被代理拦截，基准数字来自第三方转述并如此标注
> 重要时间点：**2026-09-16 Anthropic 宣布把 Cowork 并入统一的 Claude 体验**并推出 Claude Docs / Slides / Design；**2026-08-13 DeepSeek 开源 DeepSeek Harness（`dsh`，MIT，开发者预览）**。

---

## 0. 3000 英尺结论（TL;DR）

假设四者背后是**同一个模型**，剩下的差异只来自：**harness（提示词、工具设计、上下文/记忆策略）**、**执行环境（本机 / VM / 云）**、**生态与治理（接入、权限、企业控制）**。

| 结论 | 说明 |
|---|---|
| **四者的关系** | Claude Code 与 Cowork 共用同一个 agentic 引擎（Anthropic 官方说法），Claude Code 是面向开发者的"全控制面"，Cowork 是面向知识工作者的"零配置面"。goose 是二者的开源替代品：正面对标 Claude Code（CLI / 桌面 / API / IDE），部分覆盖 Cowork。DeepSeek Harness（dsh）是 DeepSeek 为自家模型共训练出来的开源 harness，"一切皆插件"，定位是可组装的 agent 运行时而非最终用户产品。 |
| **harness 质量：Claude Code ≈ dsh（各自模型上）> Cowork ≥ goose** | Claude Code 有约 35 个内置工具、plan mode、effort 分档、auto mode 分类器、1M 上下文与微压缩；dsh 有 read/write/edit/glob/grep（内置 ripgrep）/持久 PTY/后台 jobs/LSP/spill/token meter/PTC，且 DeepSeek V4 的公开基准就是在 dsh"极简模式"上跑的；goose 的 developer 扩展只有 5 个工具，读文件靠 `cat/sed`。 |
| **goose 的短板能不能靠 dsh 补？能，但分两层** | **工具面 / 上下文 / 沙箱**这一层可以移植（MIT→Apache 兼容，1–2 个季度），goose 还能凭出口代理与多 provider 反超；**模型共训练**这一层只能"借力"：把 dsh 当 goose 的 ACP provider（照 `claude_acp.rs` 模式，1–2 周），或按模型族对齐工具画像（DeepSeek 用 dsh 的工具 schema，Claude 用 Claude Code 的）。细节见第 5 节。 |
| **goose 的不可替代性** | 60+ provider + 本地 llama.cpp；全规格 MCP 宿主（含 sampling）；可作为 HTTP/ACP/SDK 服务被嵌入；Apache-2.0；可自定义发行版；完全可审计。Claude Code 的 Agent SDK 只允许 API key、受商业条款约束；Cowork 没有 API；dsh 可嵌入但仍是预览期、Node 生态、DeepSeek 优先。 |
| **Computer use / 浏览器：Cowork ≈ Claude Code > dsh > goose** | Cowork/Claude Desktop 内置浏览器、Claude in Chrome、按应用授权的 computer use；Claude Code CLI `--chrome` + `computer-use` MCP（仅 macOS 预览）；dsh 走实验性 provider（Playwright MCP / Chrome DevTools MCP / Stagehand；Cua Driver computer use）；goose 仅 macOS 靠 Peekaboo 像样，无内置浏览器。 |
| **记忆** | Claude Code：CLAUDE.md 层级 + 自动记忆（机器本地）+ 子代理记忆；Cowork：账号级统一记忆（仅云端会话）；goose：显式 memory 扩展 + 历史会话全文检索 + `.goosehints/AGENTS.md` + MOIM；dsh：**无内置记忆**，只有 AGENTS.md 加载 + MCP 记忆服务器 overlay + 会话全文检索。 |
| **效率 / 成本** | Claude Code 成本工程最深（缓存统计、归因、OTel）；dsh 的 KV-cache 纪律最严（append-only 日志、系统提示作为历史节点、`in-history` 更新、工具目录跨模式稳定）且 DeepSeek 单价低；goose 透明可控且唯一能换本地模型；Cowork 受订阅限额约束。 |
| **安全 / 数据出境** | Claude Code：seatbelt/bubblewrap 沙箱 + 域名白名单代理 + 分类器 + 托管策略；Cowork：VM/云隔离（7 月有逃逸先例）；goose：检查器链 + 仅 macOS 沙箱 + 出口代理；dsh：**跨平台文件沙箱（含 Windows restricted token）但不管网络**，SAFETY.md 自述"未经安全审计"，且**默认把会话日志增量随每次请求上传到 DeepSeek 端点**（`dsh_session_log`，可关）。 |
| **对 Pions 的含义** | 编码/研发：Claude Code。嵌入产品 / 私有化 / 换模型：goose；若主力模型是 DeepSeek，用 goose + `dsh` ACP provider（或直接 dsh SDK），并**关闭 dsh 的会话日志上传**。非技术同事的文档工作：Cowork（现在的 Claude）。四者共用 **MCP server + SKILL.md + AGENTS.md** 这一公共层。 |

---

## 1. 定位与架构

### 1.1 goose（AAIF / Linux Foundation，原 Block；Apache-2.0）

```
┌────────────── 交互面 ──────────────┐
│ Desktop(Electron, mac/win/linux)   │
│ CLI / TUI(Ink over ACP) / Telegram │
│ VS Code / Zed / JetBrains (ACP)    │
│ iOS(隧道) / 任意 HTTP 客户端       │
└──────────────┬─────────────────────┘
               │ HTTP+SSE (goosed, axum, OpenAPI) / ACP (stdio, ws) / SDK(uniffi)
┌──────────────▼─────────────────────┐
│ crates/goose  (Rust 内核)          │
│  Agent loop ─ ToolInspection 链    │
│  ExtensionManager (MCP 客户端)     │
│  Platform 扩展(进程内): developer, │
│   analyze, todo, apps, summon,     │
│   skills, chatrecall, code_mode,   │
│   orchestrator, tom, ext_manager   │
│  Builtin MCP: computercontroller,  │
│   memory, autovisualiser, tutorial │
│  Providers(60+), Scheduler(cron),  │
│  Sessions(SQLite), Security, Hooks │
└──────────────┬─────────────────────┘
               │ MCP (stdio / streamable HTTP+OAuth / UDS / inline python)
        70+ 外部扩展目录
```

一切都在用户自己的机器/服务器上运行；模型通过 provider 远程或本地（llama.cpp）调用。

### 1.2 Claude Code（Anthropic；闭源，v2.1.x）

```
┌──────────────────── 交互面 ────────────────────┐
│ CLI(mac/linux/WSL/原生 win) · VS Code · JetBrains │
│ Desktop(mac/win, linux beta) · Web claude.ai/code │
│ 移动端 Code tab · Remote Control · Dispatch       │
│ Slack @Claude · GitHub Actions / GitLab CI        │
└──────────────────────┬─────────────────────────┘
                       │ 同一引擎（Agent SDK = 同一 loop，Python/TS）
┌──────────────────────▼─────────────────────────┐
│ 约 35 个内置工具 + MCP(stdio/HTTP/SSE/WS, OAuth)  │
│ 权限模式 default/acceptEdits/plan/auto/dontAsk/  │
│  bypass；auto mode 由分类器模型审查               │
│ 沙箱：seatbelt(mac) / bubblewrap(linux, WSL2)    │
│ 子代理 · agent view · agent teams · workflows    │
│ CLAUDE.md 层级 + auto memory · 33 种 hook 事件    │
│ 执行位置：本机 / 云端 VM(cloud sessions, routines)│
│  / 自托管环境 / SSH                              │
└──────────────────────┬─────────────────────────┘
                       │ Anthropic API / Bedrock / Vertex(Agent Platform)
                       │ / Foundry / Claude Platform on AWS / LLM 网关
```

### 1.3 Claude Cowork（Anthropic Labs，2026-01-12 发布，2026-09-16 并入 Claude）

```
┌───────── 交互面 ─────────┐
│ Claude Desktop(mac/win)  │
│ claude.ai Web / iOS / Android (2026-07-07 beta) │
│ Dispatch(手机遥控桌面会话, 2026-03-17) │
│ Office 加载项(Excel/PPT/Word GA 2026-05-07, Outlook beta) │
└────────────┬─────────────┘
             │ "与 Claude Code 相同的 agentic 架构"，无终端
┌────────────▼──────────────────────────────┐
│ 执行位置：云端(2026-07-07 起默认) 或本地 Linux VM │
│ 文件：授权文件夹(本地 VirtioFS 挂载 / 云会话需桌面 │
│  App 保持打开才能读写本地)                 │
│ Skills(docx/xlsx/pptx/pdf 等), Plugins(11 官方)，│
│ Connectors(38+ 远程 MCP, 自 Anthropic 云发起)， │
│ 本地 stdio MCP(桌面端, 可策略禁用)，        │
│ Sub-agents, 云端 Scheduled tasks，          │
│ Computer use(beta) + 内置浏览器/Chrome，     │
│ Claude Docs / Slides / Design(2026-09-16)   │
└───────────────────────────────────────────┘
```

### 1.4 DeepSeek Harness（`dsh`；DeepSeek AI，MIT，2026-08-13 开发者预览）

```
┌──────────────── 交互面 ────────────────┐
│ Web UI(127.0.0.1:3080) · Electron Desktop │
│  (mac arm64/x64, win x64；Linux 非发布目标) │
│ headless 一次性 CLI · TS/Python SDK(JSON-RPC) │
│ ACP server(automation-only) · 无 TUI    │
└────────────────┬───────────────────────┘
                 │ profile = bundle 分层 + cordis.patch.yml 覆盖
┌────────────────▼───────────────────────┐
│ Cordis 内核（"时空可组合性"元框架，源自 Koishi）│
│  一切皆插件：模型适配器、工具注册表、        │
│  会话日志、agent loop 本身都可替换          │
│ 291 个 @deepseek-ai/dsh-* 包            │
│ Agent presets(工作模式)：标准 / PTC / 极简 / 创造 │
│ append-only 会话日志 = 模型上下文唯一真源  │
│ 沙箱：Linux bwrap→Landlock / macOS Seatbelt / │
│  Windows ACL restricted token（仅文件效应）│
│ 子代理：in-process spawn/fork · ACP ·     │
│  Codex(app-server) · Claude Code(Agent SDK) │
└────────────────┬───────────────────────┘
                 │ deepseek-official(Messages/Chat, 1M ctx)
                 │ + pi-ai 目录(anthropic/openai/moonshot/zai/自定义网关)
```

模型：Claude Code / Cowork 仅 Claude；dsh 以 DeepSeek 为一等公民并可接 pi-ai 目录与自定义 OpenAI/Anthropic 协议网关，无本地推理；goose 60+ provider 含本地推理。

---

## 2. 逐维度深度对比（goose / Claude Code / Cowork）

每个维度：**goose 实现（含源码位置）→ Claude Code 实现（官方文档）→ Cowork 实现 → 判断**。评分 1–5，是"同模型"前提下对 harness / 环境 / 生态的评估。dsh 的逐维度事实集中在第 5 节，四方评分见第 3 节。

### 2.1 Agent 循环、规划与推理 harness

**goose**
- 主循环 `crates/goose/src/agents/agent.rs::reply_internal`（约 L1616–2300）：注入 MOIM → 流式调 provider → 工具请求过 `ToolInspectionManager`（permission / repetition / security / adversary / egress 五个检查器）→ 通过的工具**并发执行**（`stream::select_all`，L1967）→ 回灌。默认最多 1000 轮无人干预（`GOOSE_MAX_TURNS`）。
- 自驱：`/goal` 自评、grind"没干完继续干"（L2254–2293）；recipe `retry`+`checks`+`on_failure`（`agents/retry.rs`）；结构化输出 `recipe__final_output` 按 JSON schema 校验。
- 规划：独立 planner 模型（`GOOSE_PLANNER_MODEL`，`prompts/plan.md`）；Todo 每轮可见。
- 思考：`CLAUDE_THINKING_TYPE=adaptive|enabled|disabled` + budget。
- 工具面：developer 扩展仅 write / edit（唯一精确匹配）/ shell（2000 行截断）/ tree / read_image；**没有独立 read、glob、grep、web 工具**，靠 shell 与 MCP 补。

**Claude Code**
- 内置工具约 35 个（tools-reference）：`Read`（带行号，图片/PDF ≤20 页/notebook）、`Edit`（读后改，唯一匹配或 `replace_all`）、`Write`、`Glob`、`Grep`（ripgrep）、`Bash`（默认 2 分钟超时、30KB 内联输出上限、`run_in_background`）、`PowerShell`、`Monitor`（监听日志/文件/WebSocket 并响应）、`Agent`、`WebFetch`（**隔离上下文**防注入）、`WebSearch`、`AskUserQuestion`、`Task*`、`Skill`、`ToolSearch`（MCP 工具定义延迟加载）、`Workflow`、`EnterPlanMode/ExitPlanMode`、`EnterWorktree`、`Artifact`、`CronCreate`、`SendMessage/ListAgents`（跨会话）等。
- 规划：plan mode（只读探索后出计划，退出需批准）；effort 档位 low/medium/high/xhigh/max + `ultracode`（xhigh + 自动写 workflow）；extended thinking 默认开；`/goal`；Advisor（Anthropic API 专有）；checkpoints `/rewind` 可同时回滚代码与对话。
- 权限判断：auto mode 用**独立分类器模型**审查每个动作（Pro/Max/Team 默认起始模式）。

**Cowork**
- 同一引擎，无终端；任务模式先出计划、进度跟踪、并行 workstream；effort 可调；Skill 可指定模型；不能改系统提示/循环参数。

**判断**：Claude Code 是三者中 harness 最完整、最"贴模型"的；Cowork 是其简化面；goose 的循环完全可编程但工具面薄。

评分：goose 3.5 / Cowork 4.5 / Claude Code 5

### 2.2 文件处理

**goose**：任意路径（工作目录只是默认 cwd 与 MCP roots）；`write` 自动建目录；`edit` 可接 `GOOSE_EDITOR_MODEL`（Morph 等 fast-apply）；工具结果 >200k 字符落盘（`large_response_handler.rs`）；computercontroller 内置 `xlsx_tool`/`docx_tool`/`pdf_tool`；`analyze` 扩展 tree-sitter 调用图（10 种语言）；桌面 `@` 模糊文件搜索；macOS 可开 seatbelt 写保护 `~/.ssh`、shell rc、goose 配置。没有检查点/回滚。

**Claude Code**：Manual 模式下写操作限工作目录及子目录，读父目录外路径要问；`Read` 原生看图/PDF/notebook；`/rewind` 检查点回滚；worktree 隔离；`.claude/rules/` 按路径加载规则；Office 文档靠官方 docx/xlsx/pptx/pdf skills（插件）与 Artifacts（Pro+）；MCP 工具输出 >25k tokens 自动落盘。

**Cowork**：边界是**授权文件夹**（本地 VirtioFS 挂进 VM；云会话需桌面 App 打开且会话在桌面发起才能读写本地）；Claude Docs / Slides / Design 原生可编辑产物；Excel/PPT/Word/Outlook 加载项共享上下文。

**判断**：代码与本地文件 Claude Code ≥ goose > Cowork；Office 文档 Cowork > Claude Code > goose。

评分：本地文件/代码 goose 4.5 / Cowork 4 / Claude Code 5；Office 文档 goose 3 / Cowork 5 / Claude Code 3.5

### 2.3 Computer use 与浏览器

**goose**：`computer_control` 按 OS 分流（`goose-mcp/src/computercontroller/mod.rs:353–500`）——macOS 自动 `brew install steipete/tap/peekaboo`，提供 `see --annotate` 元素级点击/输入/窗口/菜单/剪贴板；Windows 退化为 PowerShell + 截图；Linux 为 xdotool/wmctrl；**无内置浏览器**（Playwright / Chrome DevTools / Browserbase MCP 补）；`web_scrape` 只是 HTTP GET。

**Claude Code**：`claude --chrome` 接 Claude in Chrome 扩展（共享登录态、控制台/DOM、表单、文件上传 ≤10MB、GIF 录制；Chromium 系浏览器；直连 Anthropic 计划，Bedrock 等不可用）；桌面 App 内置浏览器面板（2026-07，干净 profile，写操作有安全分类器审查）；`computer-use` 内置 MCP（**仅 macOS CLI 研究预览，Pro/Max，不含 Team/Enterprise**）：按应用逐会话授权、分级（浏览器只看、终端/IDE 只点、其他全控）、终端窗口排除在截图外、Esc 全局中止、同一时间只能一个会话；Desktop 版 mac + win，mac 可后台运行。

**Cowork**：内置浏览器或 Chrome；computer use beta（2026-03-23 起，Pro/Max）按应用弹窗 Deny / Always allow / Allow once；2026-09 支持后台运行。

**判断**：Cowork 与 Claude Desktop 共享同一 computer use 引擎，成熟度相当；Claude Code CLI 的 Chrome 集成对开发者最实用；goose 落后一代。

评分：goose 2.5 / Cowork 4.5 / Claude Code 4

### 2.4 分析、推理与产出形式

**goose**：`autovisualiser`（图表/地图/图谱）、Apps 扩展（HTML 小应用在沙箱窗口）、MCP Apps / MCP-UI 渲染、`summarize` 扩展、主/fast/planner/editor 多模型分工、结构化 JSON 产物、`goose review`（`.agents/checks/*.md` 子代理审查器）。

**Claude Code**：LSP 代码智能插件（跳转/引用/类型错误）、`/ultrareview` 多代理云端审查、`/security-review`、`/code-review`（Team/Enterprise）、`/deep-research` 内置 workflow（多角度检索 + 交叉验证）、`/insights` 使用模式报告、Artifacts 网页产物、WebSearch/WebFetch 原生。

**Cowork**：Docs / Slides / Design 产物、Office 文件、官方领域插件（Data/Finance/Legal 等）自带 skills 与 sub-agents。

评分：goose 3.5 / Cowork 4 / Claude Code 4.5

### 2.5 运行效率与成本

**goose**：Rust 单二进制、SSE 流式、工具并发；Anthropic prompt caching 打在 system / 最后 tool / 最后两条 user（`providers/formats/anthropic.rs:281–349`）+ `token-efficient-tools` beta；工具输出后台配对摘要（`context_mgmt/mod.rs`）；Code Mode（pctx/Deno，LLM 写 JS 批量调工具，只暴露 3 个元工具）；sessions.db 记录每会话 token/费用；`GOOSE_CLI_SHOW_COST`；按 API token 付费或经 ACP provider 复用订阅。**唯一能靠换模型（含本地模型）降单价的方案。**

**Claude Code**：prompt caching 默认开（订阅 1h TTL，API/用量信用 5m）；微压缩（编辑缓存条目而非重发）；MCP 工具 schema 默认延迟加载；`/usage` 给出缓存命中率、缓存失效原因、按 skill/子代理/插件/MCP 服务器归因、行为标记；`/cost`、`modelPricing` 合同价、状态栏成本；OTel 每用户 token/成本；effort 分档、fast mode、子代理指定 haiku、hooks 预处理工具输出；企业平均约 $13/开发者/活跃日、$150–250/月（官方 costs 页）；订阅制 5 小时 + 每周窗口与 Chat/Cowork 共享，routines 有每日运行上限，可开用量信用超额。

**Cowork**：云执行与本机解耦；同一订阅限额；用户普遍反映"启动就读整个文件夹""历史重复读取"导致 token 飙升，桌面 App 大上下文卡顿。

**判断**：成本**可观测性**Claude Code 最强；成本**可控杠杆**goose 最多；Cowork 最弱。

评分：goose 4 / Cowork 3 / Claude Code 4.5

### 2.6 记忆

**goose**（显式、本地、可审计）：`memory` 扩展（分类 + 标签 markdown，`.goose/memory/` 或 `~/.config/goose/memory/`，保存前需确认）；`chatrecall` 对 SQLite `sessions.db` 全文检索历史会话；`.goosehints` + `AGENTS.md`（全局/项目/随工具触达的子目录动态加载，`hints/load_hints.rs`）；MOIM 每轮注入时间、cwd、todo、`GOOSE_MOIM_MESSAGE_TEXT/FILE`（≤64KB）；会话 resume/fork/export/import（**可导入 Claude Code / Codex / Pi 的 JSONL**，`session/import_formats/`）。无隐式自动记忆。

**Claude Code**：CLAUDE.md 四级（托管策略 / `~/.claude/CLAUDE.md` / 项目 `CLAUDE.md` 或 `.claude/CLAUDE.md` / `CLAUDE.local.md`），`@path` 导入（4 跳），`.claude/rules/*.md` 可按 `paths` 前置元数据按文件类型加载，`claudeMdExcludes`；**auto memory 默认开**：`~/.claude/projects/<project>/memory/MEMORY.md` 索引（前 200 行或 25KB 每次加载）+ 主题文件，记录 user / feedback / project / reference 四类，跳过可从代码推导的内容，`/memory` 可查看编辑；**机器本地，不跨设备/云同步**；子代理可有独立记忆（user/project/local 三种作用域）；压缩后 CLAUDE.md 与 auto memory 从磁盘重注入；读 `AGENTS.md` 需 `@AGENTS.md` 导入或软链。

**Cowork**：2026-08-25 起与 Claude 聊天共用账号级记忆（按主题分类条目，可查看/编辑/删除），**仅云端会话可用**，本地 VM 会话不可用。

**判断**：Claude Code 的记忆体系最完整（人写 + 自动 + 子代理 + 路径规则）；Cowork 体验最省心但仅云端；goose 最适合 git 化与合规审计。三者都无向量语义长期记忆。

评分：goose 3 / Cowork 3.5 / Claude Code 4

### 2.7 上下文管理

**goose**：自动压缩阈值 80%（`GOOSE_AUTO_COMPACT_THRESHOLD`），压缩提示词可改（`prompts/compaction.md`）；工具配对摘要；超限兜底 summarize / truncate / clear / prompt 与逐步移除工具响应；`/summarize`；子代理独立上下文；provider 自管上下文时跳过。

**Claude Code**：自动压缩（按模型有默认阈值，可 `/autocompact 500k` 提前）、微压缩、`/compact <focus>`、`/rewind` 选段摘要、`/clear`；**压缩后保留清单**：系统提示与输出风格、根 CLAUDE.md 与未限定路径的 rules、auto memory、plan、最近修改的 5 个文件重读（>5k tokens 只留引用）、已调用 skill 正文（每个 ≤5k、总 ≤25k）、后台命令与子代理继续运行、`SessionStart(compact)` hook 可再注入；**1M 上下文**：Sonnet 5 默认 1M，Fable / Opus 4.6+ / Sonnet 4.6 选 `[1m]` 变体；长时间离开后可"从摘要恢复"。

**Cowork**：同一引擎的 compaction；用户报告 context rot 仍存在。

评分：goose 4 / Cowork 4 / Claude Code 5

### 2.8 API / MCP 接入能力

**goose（作为 MCP 宿主）**：stdio、Streamable HTTP（OAuth 授权码流 + 凭证刷新 + 自定义 header 环境变量替换 + HTTP over Unix socket）、inline python（uvx）、frontend 工具；客户端能力 roots、**sampling**（`create_message`）、elicitation、progress/logging、resources、prompts（`mcp_client.rs:246–419`）；MCP Apps / MCP-UI；模型可自行搜索/启停扩展；扩展超时 300s；OSV 恶意包检查；`GOOSE_ALLOWLIST`；>5 扩展或 >50 工具提醒收缩。
**goose（作为可被接入的服务）**：`goosed` REST + SSE（OpenAPI、TLS、`X-Secret-Key`，可远程部署）；ACP server（stdio / HTTP / WebSocket）供 Zed / JetBrains / VS Code；`goose-sdk` uniffi（Python/Kotlin 脚手架）；Telegram 网关；`goose run` 无头 + `--output-format json|stream-json`。

**Claude Code（作为 MCP 宿主）**：stdio / HTTP（OAuth 2.0 动态注册或预置凭证、`headersHelper` 动态鉴权、RFC 9728/8414 发现）/ SSE（弃用）/ WebSocket；作用域 local / project（`.mcp.json`）/ user / managed；工具搜索延迟加载默认开（v2.1.232+）；输出上限默认 25k tokens 超出落盘；长调用 2 分钟后自动转后台；v2 运行时（协议 2026-07-28，`list_changed` 动态更新）；Channels（MCP server 向会话推送事件）；elicitation、resources、prompts；企业 `allowedMcpServers/deniedMcpServers`、连接器工具级 ask/block；claude.ai connectors 同步到终端。**未见 sampling 支持文档。**
**Claude Code（作为可被接入的服务）**：Agent SDK（Python/TS，同一 loop 与工具；**仅 API key，不得为第三方产品提供 claude.ai 登录/限额；受商业条款；不得以 Claude Code 品牌**）；`claude -p --output-format json|stream-json` 子进程；Managed Agents（2026-04，托管 REST + 沙箱）；routines API 触发端点（bearer token）；GitHub Actions / GitLab CI；Slack；ACP 通过社区适配器 `claude-agent-acp`（goose、Zed 均用它）。

**Cowork**：38+ connectors（远程 MCP，由 Anthropic 云发起，服务器需公网可达）；自定义远程 MCP；桌面端本地 stdio MCP（MDM `isLocalDevMcpEnabled` 可禁）；**没有驱动 Cowork 本身的 API**。

**判断**：MCP 客户端完整度 goose ≈ Claude Code（goose 多 sampling 与 MCP Apps，Claude Code 多工具搜索、输出落盘、Channels、企业策略）；"被嵌入"goose 最开放，Claude Code 有 SDK 但带商业与鉴权限制，Cowork 为零。

评分：goose 5 / Cowork 3.5 / Claude Code 4.5

### 2.9 多 Agent、编排与自动化

**goose**：`summon` 的 `load`/`delegate`（可指定 provider/model/extensions/max_turns，`async` 后台并行，默认最多 5 个、25 轮）；`.md` agents（读 `.agents/agents`、`.claude/agents`）；隐藏 `orchestrator` 扩展（list/start/send/interrupt 其他 agent 会话）；Recipes（YAML：instructions/prompt/extensions/参数/settings/sub_recipes/response schema/retry）+ deeplink + GitHub 仓库；本地 cron 调度（`tokio-cron-scheduler`，仅进程存活时）；`goose review`；Hooks 13 个事件（PreToolUse 可拒、Stop 可阻断），open-plugins / Gemini 插件格式，24h 自动更新。

**Claude Code**（官方 agents 页把并行方式分为五种）：
- **Subagents**：`.claude/agents/*.md`（`tools/disallowedTools/model/permissionMode/skills/memory/mcpServers/hooks/maxTurns/isolation: worktree/effort/omitClaudeMd`），内置 Explore / Plan / general-purpose，交互会话默认后台运行，默认 20 个并发、嵌套深度 3，可 resume，转录独立存储；fork 子代理继承全部上下文。
- **Agent view**（`claude agents`）：后台会话统一调度/监控，自动进 worktree。
- **Agent teams**（实验，默认关，仅 CLI 交互）：lead + teammates、共享任务列表（文件锁）、mailbox 消息、`TeammateIdle/TaskCreated/TaskCompleted` hook 质量门；约 7× token。
- **Dynamic workflows**：Claude 写 JS 脚本（`agent/parallel/pipeline/phase`），默认 16 并发、单次 ≤1000 agent、可暂停/恢复/保存为命令/随插件分发；`/deep-research` 内置；`ultracode` 关键字或 `/effort ultracode`。
- **Projects**：claude.ai/code 上一个长期对话协调多个云端线程。
- 辅助：worktrees、跨会话消息（本机/他机/云）、`/batch`（5–30 个 worktree 子代理各开 PR）。
- 调度：**Routines**（云端，schedule / API / GitHub 三种触发，最小 1 小时，每日运行上限，自带 GitHub 身份与 connectors）、桌面本地 scheduled tasks、`/loop`、Channels（Telegram/Discord/iMessage/webhook 推入运行中会话）、PR auto-fix。
- Hooks：**33 个事件**（PreToolUse/PermissionRequest/PostToolBatch/PreCompact/PostCompact/WorktreeCreate/PreModelSwitch/Elicitation/…），类型 command / http / mcp_tool / prompt / agent，可放 settings、插件、skill 或子代理前置元数据。

**Cowork**：sub-agents 并行 workstream；plugins 打包 skills + slash commands + connectors + sub-agents；云端 scheduled tasks（不需开机，享 connectors/skills/plugins）；Dispatch。

**判断**：Claude Code 的编排原语最丰富也最成熟；goose 的 recipes（可版本化 + 成功校验 + 重试）是独有优势；Cowork 面向非技术用户封装。

评分：goose 4 / Cowork 4 / Claude Code 5

### 2.10 自主性、权限与安全

**goose**：4 种模式（`auto` 默认 / `approve` / `smart_approve` LLM 判定只读放行 / `chat`）+ 逐工具 Always/Ask/Never + `readOnlyHint` 注解；提示注入扫描（正则 + 可选 HF 分类器）；adversary 模式（`adversary.md` 规则由 LLM 审 shell，失败放行）；egress 检查器；OSV 恶意包检查；扩展白名单；危险环境变量黑名单；macOS seatbelt 沙箱 + 本地 CONNECT 出口代理（域名黑名单、禁裸 IP、SSH 仅 git 主机、封 nc/socat）——**仅 macOS**；MCP 客户端可在 Docker 容器启动；开源可审计。

**Claude Code**：权限模式 `default`（Manual，只读免问）/ `acceptEdits` / `plan` / `auto`（分类器审查，Pro/Max/Team 默认起始）/ `dontAsk`（CI 用，会问的一律拒）/ `bypassPermissions`（仅隔离环境）；规则语法 `Bash(npm run *)`、`Read(~/secrets/**)`、`WebFetch(domain:…)`、`mcp__server__tool`；受保护路径（`.git`、`.claude`）除 bypass 外永不自动批准；命令注入检测、`curl/wget` 默认不自动批准、WebFetch 隔离上下文、首次进目录与新 MCP 需信任确认；**沙箱**：seatbelt（mac）/ bubblewrap + socat（linux、WSL2，原生 Windows 不支持），默认只写 cwd 与附加目录、读全盘但可 `denyRead`，网络经沙箱外代理走域名白名单（默认不解 TLS，官方注明域名前置风险；实验性 `tlsTerminate` + 凭证掩码可让命令永远看不到真实密钥）；`strictAllowlist`、`allowManagedDomainsOnly` 托管锁定；托管策略（MDM、server-managed settings、`permissions.deny`、`disableBypassPermissionsMode`）；云会话隔离 VM + 作用域凭证代理 + 审计日志；Enterprise 可 Zero Data Retention；computer use 跑在真实桌面，靠按应用授权。闭源，HackerOne 漏洞奖励。

**Cowork**：本地 Linux VM（VirtioFS 共享授权文件夹）或云端执行；2026-07-27 SharedRoot（Linux 内核 CVE-2026-46331 提权 → 宿主机整个 `/` 读写挂载）约 50 万 macOS 本地会话用户暴露，已修复，7 月 7 日起云执行默认；computer use 按应用授权；企业 MDM 策略、SSO/SCIM、审计；Enterprise 2026-09-10 起默认开启。

**判断**：Claude Code 的"策略 + 沙箱 + 分类器 + 托管"组合最完整；Cowork 靠 VM/云硬隔离但有逃逸先例；goose 防线多而软、Linux/Windows 无沙箱，但唯一可完整审计代码。

评分：goose 4 / Cowork 4 / Claude Code 4.5

### 2.11 平台与交互入口

| | goose | Claude Code | Cowork |
|---|---|---|---|
| 桌面 | macOS / Windows / Linux（zip/deb/rpm/flatpak） | Claude Desktop Code 标签：macOS / Windows，Linux beta（apt） | Claude Desktop：macOS / Windows |
| 终端 | 完整 CLI + TUI（Ink over ACP）+ `goose term` | 完整 CLI（mac/linux/WSL/原生 Windows） | 无 |
| IDE | Zed / JetBrains / VS Code（ACP） | VS Code / Cursor 扩展、JetBrains 插件 | 无 |
| Web / 移动 | 远程 goosed、iOS 隧道（实验）、Telegram 网关 | claude.ai/code、移动端 Code tab、Remote Control、Dispatch、`--cloud`/`--teleport` | claude.ai Web、iOS/Android beta、Dispatch |
| 聊天/CI | Telegram | Slack @Claude、Channels（Telegram/Discord/iMessage）、GitHub Actions、GitLab CI | — |
| Office | 无 | Artifacts | Excel/PPT/Word GA、Outlook beta、Docs/Slides/Design |
| 远程执行 | 自建 goosed | Anthropic 云 VM、自托管环境、SSH | Anthropic 云 VM |
| 语音 | 听写：OpenAI/Groq/ElevenLabs/本地 whisper | `/voice`（需 claude.ai） | 移动端语音 |

评分：goose 4.5 / Cowork 4.5 / Claude Code 5

### 2.12 模型与供应商

**goose**：`providers/init.rs` 注册 ~35 个代码 provider + 29 个声明式 JSON provider（Anthropic、OpenAI、Google/Vertex、Azure、Bedrock、SageMaker、Databricks、Ollama、OpenRouter、LiteLLM、HuggingFace、Snowflake、xAI、DeepSeek、Groq、Mistral、Cerebras、NVIDIA、Zhipu/Z.ai、MiniMax、Moonshot、Alibaba、Perplexity、LM Studio…）；**本地推理**（llama.cpp，CUDA/Vulkan）；`toolshim` 给无原生工具调用的小模型；**ACP providers**（`claude_acp` / `codex_acp` / `copilot_acp` / `amp_acp` / `pi_acp`）把整个 Claude Code / Codex harness 当模型用并复用订阅；主 / fast / planner / editor 多模型分工。注意：goose 的 DeepSeek 声明式 provider（`providers/declarative/deepseek.json`）目录仍是 `deepseek-chat` / `deepseek-reasoner` @128k、走 OpenAI 兼容 Chat Completions，落后于 dsh 的 Messages 协议 + 1M 目录。

**Claude Code**：仅 Claude；接入 Anthropic API、Bedrock、Claude Platform on AWS、Google Cloud Agent Platform（原 Vertex）、Microsoft Foundry、LLM 网关（如 LiteLLM）；**功能随供应商递减**（feature-availability 页）：Bedrock 无 WebSearch / fast mode / Advisor / Channels；所有第三方供应商无云会话、routines、Desktop（除 3P 版）、Chrome、computer use、Remote Control；auto mode 在第三方仅限 Sonnet 5 / Opus 4.7+ / Fable 且默认起始为 Manual。子代理/skill/workflow 各阶段可指定模型；effort、fast mode。

**Cowork**：仅 Claude；企业可指向兼容端点。

评分：goose 5 / Cowork 1.5 / Claude Code 2

### 2.13 可观测性与可审计

goose：Langfuse 层、OTel OTLP、请求日志、会话导出/导入、诊断 zip、PostHog 遥测默认关；源码可审。
Claude Code：OTel 指标/日志（每用户 token/成本/工具活动，任何供应商可用）、`/usage`（缓存与归因）、`/insights`、Team/Enterprise 分析仪表盘、Enterprise Analytics API、Compliance API、`InstructionsLoaded`/`ConfigChange` hook 审计、会话 JSONL 转录（`~/.claude/projects/`，goose 可导入）；引擎闭源。
Cowork：会话记录、企业审计日志；轨迹不可导出到自有平台。

评分：goose 4.5 / Cowork 2.5 / Claude Code 4.5

### 2.14 生态与可扩展性

goose：Skills（agentskills.io 规范，扫描 `~/.agents/skills`、`.agents/skills`、插件目录，**兼容 `.claude/skills` 与 `.claude/agents`**）、Plugins（open-plugins / Gemini 格式）、Hooks、Recipes、MCP Apps、自定义发行版（`CUSTOM_DISTROS.md`）、Apache-2.0、AAIF 治理。
Claude Code：Plugins 打包 skills / agents / hooks / MCP / commands / LSP / workflows，官方市场（签名审核）+ 社区市场（数百插件、数千 skills）、`claude plugin eval`；Skills（同一规范）；`/import` 从其他 agent 迁移 AGENTS.md / MCP / 子代理；CLAUDE.md；Agent SDK 生态。
Cowork：插件市场（官方 11 + 组织私有）、Skills、Connectors 目录。

评分：goose 4 / Cowork 4 / Claude Code 5

---

## 3. 评分矩阵（同模型前提，四方）

| 维度 | goose | Cowork | Claude Code | dsh | 决定性因素 |
|---|:---:|:---:|:---:|:---:|---|
| Agent 循环 / 推理 harness | 3.5 | 4.5 | 5 | 4.5 | Claude Code 工具面最完整；dsh 与 DeepSeek 模型共训练、KV-cache 纪律最严，但仍是预览 |
| 本地文件 / 代码操作 | 4.5 | 4 | 5 | 4.5 | dsh：read/write/edit/glob/grep/持久 PTY/LSP/后台 jobs |
| Office 文档产出 | 3 | 5 | 3.5 | 3.5 | dsh 有 office skills + Office→PDF + `present` 交付 |
| Computer use / 浏览器 | 2.5 | 4.5 | 4 | 3 | dsh 走实验性 Playwright/Chrome DevTools/Stagehand 与 Cua Driver |
| 分析与产出形态 | 3.5 | 4 | 4.5 | 3.5 | dsh：LSP、web search（exa/perplexity/deepseek）、workflows、deliverables |
| 运行效率 / 成本可控 | 4 | 3 | 4.5 | 4 | dsh：token meter 路由计价、spill、剪枝、PTC；DeepSeek 单价低 |
| 记忆 | 3 | 3.5 | 4 | 2 | dsh 无内置记忆，只有 AGENTS.md + MCP 记忆 overlay + 会话检索 |
| 上下文管理 | 4 | 4 | 5 | 4 | dsh：0.8 阈值 / 保留 16% / 先剪枝后摘要 / 图像卸载 / 1M 目录 |
| MCP / API 接入 | 5 | 3.5 | 4.5 | 4 | dsh：MCP tools+resources（无 prompts、无 OAuth 文档）、ACP server（无 modes/fork/elicitation）、TS/Python SDK、webhook |
| 多 Agent / 调度 | 4 | 4 | 5 | 4.5 | dsh：spawn/fork/ACP/Codex/Claude Code 子代理、workflows、ralph、实验 teams、会话内 schedule、webhook |
| 安全与治理 | 4 | 4 | 4.5 | 3 | dsh：跨平台文件沙箱含 Windows，但不管网络；未经审计；会话日志默认上传 |
| 模型选择 | 5 | 1.5 | 2 | 3.5 | dsh：DeepSeek 一等 + pi-ai 目录 + 自定义网关；无本地推理、无 OAuth 类 provider |
| 平台入口 | 4.5 | 4.5 | 5 | 3 | dsh：Web UI、桌面 mac/win、headless、SDK；无 TUI、无 Linux 桌面、无移动端 |
| 可观测 / 可审计 | 4.5 | 2.5 | 4.5 | 4.5 | dsh：append-only 日志、"模型可见即已记录"、OTel、会话检索；开源 |
| 可嵌入 / 二次开发 | 5 | 1 | 4 | 5 | dsh：一切皆插件、profile/bundle、TS/Python SDK、ACP、MIT |
| 生态 | 4 | 4 | 5 | 3 | dsh：发布一个月，`dsh-plugin` 话题起步，社区以中文为主 |

---

## 4. 对 Pions（物理 + AI，钻井场景）的落地建议

### 4.1 按用途选型

| 用途 | 推荐 | 理由 |
|---|---|---|
| 研发 / 编码 / 代码审查 / CI | **Claude Code** | harness 最强；routines（API/GitHub 触发）、GitHub Actions、`/ultrareview`、worktree 并行 |
| 嵌进 Pions 产品（钻井仿真 / 实时数据 / 优化建议），可能私有化或离线，或需要换模型 | **goose 内核**（goosed API / ACP / 自定义发行版）；若锁定 Claude 且接受商业条款与 API key 计费，可选 Claude Agent SDK / Managed Agents | Cowork 无 API；Agent SDK 不得用订阅、不得以 Claude Code 品牌出现、无法接非 Claude 模型；goose 可接本地模型与任何 provider |
| 主力模型是 DeepSeek 的场景 | goose + `dsh` ACP provider（第 5.3 节路径 1），或直接 dsh TS/Python SDK | 拿到与 DeepSeek 共训练的 harness；务必关闭 `dsh_session_log` 上传 |
| 要 Claude Code harness 但保留多 provider / recipes / 可审计 | goose + `claude_acp` provider | 用 goose 壳包 Claude Code 引擎，扩展以 MCP 透传 |
| 销售 / 市场 / 法务文档 | **Cowork（现 Claude 统一体验）** | Docs/Slides/Office 加载项、38+ connectors、零配置 |
| 无人值守定时处理 | 数据在云 SaaS/GitHub → Claude Code routines 或 Cowork 云任务；数据在内网/边缘 → goose cron + recipe `retry/checks`，或 Claude Code 自托管环境 | 云任务碰不到内网；goose 调度需进程常驻 |
| 需要审计模型每一步 | goose 或 Claude Code（OTel）或 dsh（append-only 日志） | goose/dsh 可审代码；Claude Code 可导出指标与转录 |

### 4.2 一次投入、四边通吃的公共层

1. **MCP server**：把钻井仿真器、实时数据（WITSML/OPC）、地质模型封装成 MCP（Streamable HTTP 最通用；dsh 的 MCP client 无 OAuth 流程文档，内网部署用 header token）。goose / Claude Code / dsh 直接接；Cowork 以"自定义远程 connector"接。
2. **SKILL.md**：钻井 SOP、压力窗口计算流程、报告模板写成 agentskills.io 规范技能。goose 与 Claude Code 读 `.claude/skills/`；dsh 读 `.agents/skills/`（goose 也读）——放 `.agents/skills/` 三者通吃。
3. **子代理定义**：`.claude/agents/*.md`——goose 的 summon 与 Claude Code 都读。
4. **项目指令**：写 `AGENTS.md`（goose、dsh 原生读），`CLAUDE.md` 里 `@AGENTS.md` 导入（Claude Code 官方推荐做法；dsh 也读 CLAUDE.md）。
5. **会话可迁移**：goose 能导入 Claude Code 的 JSONL 转录，便于统一归档与审计。

### 4.3 需要提前认清的坑

- goose：harness 调优是自己的活（可从改 `prompts/*.md` 与 developer 指令起步）；Linux/Windows 无沙箱、computer use 弱；调度依赖进程常驻；桌面沙箱仅 macOS；工具面薄（无原生 read/grep/web）。
- Claude Code：仅 Claude；第三方供应商上功能明显缩水；Agent SDK 只能 API key 计费且不得复用订阅；agent teams / routines / channels / computer use 多为研究预览；原生 Windows 无沙箱；订阅限额与 Chat/Cowork 共享。
- Cowork：数据默认进 Anthropic 云（含 connector 调用）；订阅限额在长任务上是瓶颈；产品形态正在变（9/16 合并）；无 API、不可白标、不可换模型。
- dsh：开发者预览、明示会有破坏性变更；Node ≥22.19、pnpm 11、291 包与 Cordis 学习曲线；**会话日志与插件清单默认随请求上传 DeepSeek**；沙箱不管网络；无内置记忆；无 TUI 与 Linux 桌面；SAFETY.md 自述未审计。

### 4.4 一个可行的混合架构

```
Pions 平台(内网/边缘)                      研发                      办公侧
┌────────────────────────────┐   ┌──────────────────────┐   ┌──────────────────────┐
│ goosed (自定义发行版)       │   │ Claude Code          │   │ Claude(原 Cowork)    │
│  ├ provider: 本地/私有模型  │   │  CLI/IDE/routines    │   │  connectors: 邮件/   │
│  │   / Claude via Bedrock   │   │  .claude/agents      │   │  Slack/CRM/Drive     │
│  │   / DeepSeek via dsh-acp │◄──┼──MCP + SKILL.md──────┼──►│  自定义 connector →  │
│  ├ MCP: 仿真器/实时数据/    │   │  CLAUDE.md @AGENTS.md│   │  Pions 数据(只读)    │
│  │   地质模型/报告生成       │   │  sandbox + hooks     │   │  Skills: 同一套      │
│  ├ recipes + cron: 日报/预警 │   │  OTel → 统一观测     │   │  Docs/Slides 产出    │
│  └ hooks: 审计/合规拦截      │   └──────────────────────┘   └──────────────────────┘
└────────────────────────────┘
```

---

## 5. DeepSeek Harness（dsh）深度对比与 goose 补短板分析

### 5.1 dsh 是什么（源码事实）

| 项目 | 事实 |
|---|---|
| 仓库 / 许可 / 状态 | `deepseek-ai/deepseek-harness`，MIT；2026-08-13 开发者预览，README 明示"会有破坏兼容性的变更"；本次克隆 v0.1.6-alpha.2（2026-09-17）；GitHub org 页显示约 22.8 万 star |
| 技术栈与规模 | TypeScript / Node ≥22.19、pnpm 11；291 个 `@deepseek-ai/dsh-*` 包（`packages/<group>/<pkg>`），约 76 万行 TS（含测试与生成物）；2206 份 `.agents/notes` 决策记录；CI 要求 `packages/*/*/src` 每文件 100% 覆盖；`CLAUDE.md → AGENTS.md` 软链 + `.claude/skills`——**用 agent 开发 agent harness** |
| 内核 | Cordis（"时空可组合性"元框架，源自 Koishi 聊天机器人框架，vendored）：插件贡献服务、类型化事件与可回滚副作用；模型适配器、工具注册表、会话日志、agent loop 都是可替换插件（`docs/architecture.md`） |
| 组装模型 | profile（`web` / `headless` / `sdk` / `sdk-minimal` / `acp` / 桌面专属 `desktop`）= 有序 bundle 层 + 用户 `cordis.patch.yml` + `--patch` 覆盖；`dsh-base` 是共享底层；HMR 热重载 |
| 交互面 | Web UI（`npx @deepseek-ai/dsh web` → 127.0.0.1:3080）、Electron Desktop（mac arm64/x64、win x64，**Linux 非发布目标**）、headless 一次性 CLI、TS/Python SDK（JSON-RPC stdio）、ACP server（automation-only）；**无交互式 TUI** |
| 模型 | `deepseek-official`：Messages（默认）与 Chat Completions 双协议，默认目录 `deepseek-flash`（图像）/`deepseek-v4-pro`（文本）均 1M 上下文，思考 effort `off/low/high/max`，Files API 传图，reasoning passback；`llm-pi-ai`：anthropic / openai / moonshotai / zai 等目录 + 自定义 `openai-completions` / `openai-responses` / `anthropic-messages` 网关；OAuth 类 provider（如 Codex）暂不支持；无本地推理 |
| 工作模式（agent presets） | **标准**：bash/pwsh、read/write/edit、glob/grep（内置 `@vscode/ripgrep`）、后台 jobs、skills、goal、plan mode、compaction + 工具结果剪枝、subagent（spawn/fork/Codex/Claude Code）、workflow + `ralph`、ask_user、todo、web_search/web_fetch、present、plugin_manager；**PTC**：同上但其他工具通过 `run_code` 生成的 TypeScript SDK 呈现（类似 goose Code Mode）；**极简**：固定 persona、**仅一个持久 bash 工具**、无压缩、无运行时上下文——DeepSeek 公开基准所用；**创造**：标准 + Cordis 运行时检查 + 持久插件管理，用于编写新 preset |
| 循环 | turn/step 事件模型；`agent/pre-step`、`agent/request`、`llm/stream`、`tools/pre-execute → execute → post-execute` 瀑布；并行工具默认 10（unary parallel-safe 分类，exclusive 为屏障）；`run_code` 子调用并发 10；`llm-retry` 在步边界重试 |
| 会话日志 | append-only `SessionEvent` 日志是模型上下文的唯一真源，`deriveMessages()` 投影历史；**"模型可见 ⟺ 已记录"是运行时不变量**；JSONL(+zstd) 持久化、版本化迁移、fork/resume/telemetry 全部由日志派生 |
| 上下文 | `compaction-basic`：阈值 0.8、保留最近 16%（`retainRatio`）、先 `tool-result-pruner` 剪枝再摘要、溢出后重试 1 次；`spill` 策略把超限工具结果落盘并留头尾预览；`token-meter` 按路由计价（含 DeepSeek 视觉 token 网格）；`image/offload` |
| KV-cache 纪律 | 系统提示作为历史 surface node，模型目录声明 `systemPromptUpdate: in-history` 时提示变化**追加在缓存历史之后**；工具目录跨模式稳定（`exit_plan_mode` 常驻以免 schema 变动）；每个包 README 都有"Model Experience / Token effect / KV Cache effect"三段 |
| 记忆 | **无内置记忆**；`agent-instructions` 加载 AGENTS.md / CLAUDE.md（用户全局 + 项目向上遍历，编辑后刷新）；记忆靠 MCP overlay（文档给出 Memorix / MCP reference memory / Engram，默认关闭）；`session-query` 提供 SQLite 全文检索历史会话（5 个只读工具） |
| 多 Agent | subagent seam 多 provider 并存：in-process spawn / fork（继承历史）/ ACP / **Codex（app-server 协议）/ Claude Code（Agent SDK，捆绑平台 CLI 载荷）**/ dsh-sdk；continuable children + `send_message`/`interrupt_agent`/`list_agents`；通用后台 jobs；workflow（模型写 JS，`agent()` 编排，元数据词汇与 Claude Code dynamic workflows 一致）+ `ralph` 固定循环；实验 Agent Teams（roster / 任务 DAG / mailbox）；会话内 schedule（after / at / every ≥5 分钟）；GitHub webhook 触发 fire-and-forget 会话 |
| 权限与沙箱 | permission preset = sandbox mode（`read-only` / `workspace-write` / `danger-full-access`）+ approval policy（`ask` / `never`），默认 `workspace-write + ask`；实验性 Auto review（用当前模型审每次调用）；沙箱后端 Linux bwrap→Landlock（静态 `landlock-run`）、macOS Seatbelt、**Windows ACL restricted token**、SSH 远程；**只管文件效应，不管网络、进程、设备**；guard：重复调用提醒 + 超时；SAFETY.md："未经安全审计，不得视为生产可用" |
| 接入 | MCP client：stdio / Streamable HTTP + headers、重连退避、Tools + Resources（**Prompts 未消费**，无 OAuth 流程文档）；ACP server：v1 `session/new|list|resume|close`、`set_config_option`（model / reasoning_effort）、`prompt`、`request_permission`，**不支持 modes、fork、elicitation、client fs**；Claude Code / Codex `hooks.json` 桥（仅 7/30 事件且多为部分支持）；skills 扫描 `.dsh/skills`、`.agents/skills`、`~/.dsh/skills`、`~/.agents/skills`（不读 `.claude/skills`，除非 `customSkillDirs`）；LSP 工具（4 种操作）；web search provider exa / perplexity / deepseek；office skills（docx/pptx/xlsx，Python 脚本）+ Office→PDF（LibreOffice kit） |
| 数据出境（重要） | `dsh_session_log` **默认开启**：每次请求把会话日志增量（cwd、系统提示、用户内容、工具参数/结果、压缩摘要、插件事件…）作为请求体附加字段上传到 DeepSeek 官方端点或配置的 `baseURL` 网关，`enabled: false` 关闭；`dsh_plugin_packages` 上报插件清单；`x-deepseek-harness-user-id` 匿名 ID；OTel 会话遥测默认 `FEEDBACK_ONLY`（仅用户主动反馈时释放前缀） |
| 基准（第三方转述，未核实） | DeepSeek V4 Flash 0731 在 dsh 极简模式：Terminal-Bench 2.1 82.7（4 月预览 61.8）、DeepSWE 54.4（vs 7.3）；findharness.com 称 DSH Minimal 在 DeepSWE v1.1 得 72.6 vs Claude Code 69.8 / Codex 65.6 / OpenCode 65.5。来源与模型版本不一致，只作方向性参考 |

**关键洞察**：极简模式的持久 bash 工具描述（"does NOT need to be XML-escaped"、"State is persistent across command calls"…）与可选的 `str_replace_editor`（`view/create/str_replace/insert`，与 Anthropic text_editor 同款接口）是模型在后训练中见过的工具分布。Terminal-Bench 从 61.8 到 82.7 的跃升来自**后训练对齐同一 harness**，而不是 harness 本身多聪明——这与 Claude Code 之于 Claude 是同一件事。

### 5.2 dsh vs goose 逐维度（harness 层）

| 维度 | goose（源码） | dsh（源码） | 差距 |
|---|---|---|---|
| 工具面 | write / edit / shell / tree / read_image；读靠 `cat/sed`，搜靠 `rg` | read（窗口）/ write / edit / glob / grep（内置 ripgrep）/ bash + 持久 PTY（6 个 terminal 工具）/ run_in_background + `job_*` / lsp / web_search / web_fetch / ask_user / todo / present / skill / subagent 族 / workflow / ralph / goal / schedule | dsh 明显更完整，且工具 schema 与模型共训练 |
| 工具执行管线 | 五检查器 + 并发 `select_all` + 最大轮数 | 瀑布 pre/execute/post + unary parallel-safe 分类 + 屏障 + 重复提醒 + 超时 + spill | 等价，dsh 的并行安全分类更细 |
| 上下文 | 80% 压缩 + 工具配对摘要 + >200k 字符落盘（无预览） | 0.8 / 保留 16% + 剪枝先于摘要 + spill 头尾预览 + 路由计价 token meter + 图像卸载 | dsh 更精细；goose 有 CLI 多策略兜底 |
| KV-cache | Anthropic cache_control 三处；MOIM 每轮以新时间戳注入到尾部附近（`moim.rs` 在最后一条 assistant 前插入），尾部数条无法命中缓存；动态启停扩展改变工具列表 | 系统提示作为历史节点 + `in-history` 追加；工具目录跨模式稳定；每包文档化缓存效应 | dsh 更严谨（DeepSeek 磁盘缓存计费驱动） |
| 记忆 | memory 扩展 + chatrecall + hints + MOIM | 无内置；AGENTS.md + MCP overlay + session-query | goose 更好 |
| 沙箱 | 仅 macOS seatbelt + 出口代理（域名黑名单） | Linux bwrap/Landlock + macOS Seatbelt + Windows ACL；无网络控制 | 各半：dsh 跨平台文件沙箱，goose 有网络出口 |
| 权限 | 4 模式 + 逐工具 + LLM 只读判定 + adversary/egress/注入扫描 | sandbox mode × approval policy 预设 + 实验 Auto review | goose 检查器更多；dsh 结构更清晰 |
| 子代理 | summon delegate（可指定 provider/model，async ≤5）；`.claude/agents` | spawn/fork/ACP/Codex/Claude Code/dsh-sdk 多后端 + continuable + teams（实验） | dsh 后端更多；goose 的 `delegate(provider=claude-acp)` 已可等价 |
| 编排/自动化 | recipes（重试/成功校验/schema）+ cron + hooks 13 事件 | workflow 脚本 + ralph + 会话内 schedule + webhook + Claude Code/Codex hooks 桥（7 事件） | 各有独有物：goose recipes vs dsh workflows |
| 模型 | 60+ provider + 本地推理 + toolshim | DeepSeek 一等 + pi-ai 目录 + 自定义网关 | goose 更广；dsh 对 DeepSeek 更深（Messages、1M、Files、effort） |
| 可嵌入 | goosed HTTP/SSE + ACP server + uniffi 脚手架 | TS/Python SDK + ACP server（automation-only） | 相当 |
| 界面 | Desktop 三平台 + CLI + TUI + Telegram | Web UI + Desktop mac/win + headless | goose 更广 |
| 治理 | 遥测默认关；开源 | 会话日志默认上传；未审计；开源 | goose 更稳妥 |

### 5.3 goose 能否用 dsh 补 harness 短板：三条路径

**路径 1（借力，1–2 周）：把 dsh 当 goose 的 ACP provider**

- 实现：新增 `crates/goose/src/providers/dsh_acp.rs`，照抄 `claude_acp.rs` / `codex_acp.rs` 的 `ProviderDef` 模式：`command = dsh`（npm 全局 `@deepseek-ai/dsh`），`args = ["--profile", "acp"]`；goose 扩展经既有 `extension_configs_to_mcp_servers()` 透传（dsh ACP `session/new` 接受 stdio / HTTP MCP 并在发布前校验）；模型与 effort 经既有 `send_set_config_option()`（dsh 暴露 `model` / `reasoning_effort` 选项）；权限经既有 `handle_permission_confirmation()` 路由（dsh 用 `session/request_permission` 一次性允许/拒绝）；`session_mode_id = None`（dsh ACP 不支持 modes，goose 的 auto/approve 用 dsh 侧 `cordis.patch.yml` 的 permission preset 表达，如 `danger-full-access + never` 对应 goose `auto`）。
- 收益：goose 立即获得 dsh 完整工具面与 DeepSeek 共训练的 harness，同时保留 goose 的 UI / recipes / 调度 / 多 provider / 会话库；与现有 `claude_acp` 并列，Pions 可按模型族切换 harness。
- 限制：与其它 ACP provider 相同——goose 侧无 resume/fork；goose 检查器只能看到 ACP 工具事件（dsh 内部执行）；Node 运行时依赖；dsh 预览期 API 变动；**必须在 dsh 侧 patch `session-log-deepseek: enabled: false`** 并按需关闭 `plugin-package-inventory-deepseek`。

**路径 2（内化，1–2 个季度）：把 dsh 的设计要素移植进 goose Rust 内核**（MIT → Apache-2.0 兼容，可直接借鉴甚至翻译代码与文档）

| 要素 | goose 现状 | 移植建议 | 杠杆 |
|---|---|---|---|
| 工具面 | 5 个 developer 工具 | 增加 `read`（行号/窗口）、`glob`/`grep`（内置 ripgrep：goose 已依赖 `ignore` crate，可加 `grep-*` crates）、持久 PTY terminal（`portable-pty`）、`run_in_background` + `job_*`、`str_replace_editor` 兼容接口、`present` 交付 | 高 |
| **按模型族切换工具画像** | `toolshim`、developer 指令按 OS 分流 | 新增"tool profile"：DeepSeek 路由用 dsh 极简/标准模式的工具名与描述（含 `str_replace_editor`），Claude 路由用 Read/Edit/Bash/Grep/Glob 命名，其它模型用通用集；这是"对齐共训练分布"，改动小、收益大 | **最高** |
| 上下文 | >200k 字符落盘无预览；配对摘要 | spill 头尾预览 + 每工具上限；剪枝档位先于摘要；`retainRatio` 保留尾部；按路由计价的 token meter | 中 |
| KV-cache 纪律 | MOIM 时间戳每轮变动；扩展启停改工具列表 | 稳定工具目录（模式切换只改提示段）；系统提示变化追加到历史尾部（对支持的 API）；MOIM 降低时间戳粒度或移到系统提示尾部 | 中 |
| 沙箱 | 仅 macOS seatbelt | Linux Landlock（Rust `landlock` crate）+ bwrap、Windows restricted token，复用 dsh 的 `sandbox-local` 设计与 `landlock-run` 契约；再叠加 goose 已有的出口代理即超越 dsh | 高 |
| 会话日志 | sessions.db 存消息，MOIM/hints 注入不落库 | 采纳"模型可见 ⟺ 已记录"：把每轮注入写入会话事件，回放/审计/训练数据一致 | 中 |
| DeepSeek provider | `deepseek.json` 停留在 deepseek-chat/reasoner @128k、OpenAI 兼容 | 更新目录（deepseek-flash / v4-pro @1M）、Messages 协议、effort、Files 传图、reasoning passback | 高（若主力 DeepSeek） |
| 子代理后端 | `delegate` 已支持 `provider` 参数 | 直接用 `claude-acp` / `codex-acp` / 未来 `dsh-acp` 作子代理 provider——无需新做 | 已具备 |

**路径 3（战略）：训练侧对齐**（为什么、怎么做、数据从哪来：见 5.4）

goose 作为开源 harness 的最大机会不是再造工具，而是成为开源模型的"训练 harness"：与模型方（DeepSeek / Qwen / Kimi / GLM）共建 RL 环境，让模型在 goose 的工具分布上后训练。短期可行动：用 recipes + `goose-self-test.yaml` + `evals/open-model-gym` 搭 Terminal-Bench 类评测流水线，量化路径 2 中"工具画像"改动的收益；这也是 Pions 评估任何 harness 改动的唯一可靠尺子。

**结论**：goose 的 harness 短板**可以补，但分两层**——工具面 / 上下文 / 沙箱这一层靠移植 dsh 设计 1–2 个季度可补齐并有望反超（goose 有网络出口代理、多 provider、记忆、TUI）；"模型共训练"这一层 goose 自身代码消除不了，只能通过路径 1 借力或路径 2 的工具画像对齐来逼近。对 Pions：主力模型是 DeepSeek → goose + dsh-acp（或直接 dsh SDK）；主力是 Claude → goose + claude-acp；两条路都要在 dsh/Claude 侧处理数据出境。

### 5.4 训练侧对齐：为什么要训练、怎么训、数据从哪来

#### 5.4.1 为什么要训练（以及什么时候不必训练）

**机制**：模型的工具使用不是"读懂 JSON schema"的通用能力，而是后训练（SFT + RL）学出来的策略 π(action | 系统提示, 工具目录, 观测历史)。harness 定义了这个策略的**动作空间**（工具名、参数格式、能否并行、何时结束）和**观测空间**（`cat -n` 的行号格式、`str_replace` 的报错措辞、截断/spill 的形状、压缩摘要的样子）。换 harness 就是换环境分布：权重一字未动，但模型处在分布外——表现为参数格式错、并行调用少、错误恢复差、重复调用、过早放弃。这些正是"同模型、不同 harness、成绩差 5–20 分"的来源，也是 SWE-bench / Terminal-Bench 排行榜都按 **agent × model 组合**列名的原因。

**三条证据**：

| 证据 | 说明 |
|---|---|
| 同模型换 harness 掉分 | DeepSeek V4 Flash 在 dsh 极简 / Claude Code / Codex / OpenCode 上 DeepSWE 得分 72.6 / 69.8 / 65.6 / 65.5（findharness，第三方）；Anthropic 报告 SWE-bench 时也强调自家最小 scaffold，并注明 scaffold 影响结果 |
| 同 harness 换训练涨分 | DeepSeek V4 Flash 4 月预览 → 0731，同一个 dsh 极简模式，Terminal-Bench 61.8 → 82.7（第三方转述）：harness 没变，是模型对齐了 harness |
| 厂商都在这么做 | Anthropic：Claude Code 与 Claude 共同开发，2025-09-28 起消费者计划（含 Claude Code）会话默认用于训练、可退出；OpenAI：codex-1 / GPT-5-Codex 在 Codex harness 的真实任务上做 RL；Kimi K2：大规模 agentic 数据合成 + 联合 RL；Qwen3-Coder：2 万并行环境的长程 agentic RL；GLM-4.5：slime RL 基础设施；DeepSeek：`dsh_session_log` 默认上传会话日志——**harness 就是数据采集器** |

**训练解决不了什么**：训练不会凭空长出 harness 没有的工具；它只对齐**一个**工具分布，工具 schema 一改就要重训（这也是 dsh 坚持"工具目录跨模式稳定"的另一层原因）。所以工具画像（路径 2）永远是零成本的第一步，训练是最后一步。

**决策阶梯（何时不必训练）**

| 层级 | 做法 | 成本 | 适用 |
|---|---|---|---|
| L0 匹配分布 | 工具名 / 描述 / 观测格式照抄目标模型共训练的 harness（DeepSeek → dsh，Claude → Claude Code） | 天 | 所有情况的第一步 |
| L1 借力 | 把共训练 harness 作为 goose 的 ACP provider（`claude-acp` / `dsh-acp`） | 周 | 主力模型是闭源或有官方 harness |
| L2 SFT / 蒸馏 | 成功轨迹 LoRA 微调开源模型到 goose 工具画像 + 领域任务 | 周–月 | 自托管 / 私有化，L0/L1 不够 |
| L3 RL | 在带验证器的环境里做策略优化 | 季度 | 需要厂商没训练过的领域行为 |

对 Pions，真正的训练理由不是通用 harness（交给厂商 + L0/L1），而是 **L3 的领域行为**：钻井仿真器配置与调参、井身结构 / 钻具组合约束、测井与日报解析、异常工况诊断——没有厂商会为这些做后训练，而 Pions 手里恰好有可验证奖励的来源（见 5.4.3 第 5 条）。

#### 5.4.2 怎么做训练

对 agentic 训练，"数据集"不是一堆文本，而是 **(任务, 环境, 验证器)** 三元组；轨迹是在线 rollout 出来的。流水线五步：

| 步骤 | 内容 | goose 里已有的零件 | 差什么 |
|---|---|---|---|
| 1. 环境套件 | 数百–数千个带验证器的任务；验证器 = 单测 / 命令退出码 / 文件断言 / 仿真器一致性 | `evals/open-model-gym`：models × runners × scenarios 矩阵，scenario YAML = `setup` 文件 + 多轮 `prompt` + `validate`（`file_contains` / `file_matches` / `file_not_matches` / `file_exists` / `command_succeeds` / `tool_called` / `str`），3 次取最差；recipes 的 `retry.checks`（shell 成功校验）也可当验证器 | 目前只有 4 个场景；缺容器化隔离与并行调度 |
| 2. 轨迹采集 | 教师模型（开源强模型 / `dsh-acp`）在 goose 工具画像上跑 → 验证器过滤 → 成功轨迹；或目标模型自采样 + 拒绝采样（expert iteration） | 会话即轨迹：`sessions.db` 的 `messages` 表、`goose session export`、13 个 hook 事件旁路记录工具调用 / 结果 | 缺"训练格式"导出（含工具 schema、系统提示、奖励标签） |
| 3. SFT / 蒸馏（LoRA） | 成功轨迹（连同错误恢复片段）监督微调；混入通用数据防遗忘 | — | 公开锚点：SWE-Gym 2.4k 真实任务 → Qwen2.5-Coder-32B 32.0% SWE-bench Verified；SWE-smith 5 万合成任务 → SWE-agent-LM-32B 40.2% |
| 4. RL（RLVR / agentic RL） | 目标模型在环境中 rollout，验证器给 0/1 或分档奖励，GRPO / DAPO 类算法更新；瓶颈是**环境吞吐**（每条 rollout 数分钟沙箱），不是 GPU | — | 公开锚点：DeepSWE（Qwen3-32B，R2E-Gym 4.5k 任务，64×H100 × 6 天，纯 RL 42.2% → TTS 59%）；框架 veRL / slime / SkyRL / prime-rl / AReaL |
| 5. 评测与部署 | held-out 任务 + 通用能力回归；部署为 goose provider（vLLM / SGLang / llama.cpp），**工具画像与训练时逐字一致** | 60+ provider、本地推理、toolshim | 工具画像功能本身（路径 2） |

**算力与途径（小团队视角）**

| 途径 | 能做什么 | 量级 |
|---|---|---|
| 自建 1 节点 8×H100 | 30B-A3B 级（Qwen3-Coder-30B-A3B、gpt-oss-20b）LoRA SFT；百级并行环境的小规模 GRPO | SFT 天级、千美元级；RL 周级、万美元级 |
| 托管训练 API | Thinking Machines Tinker（开源权重 LoRA，`forward_backward` / `sample` 原语可跑 RL）；OpenAI RFT（仅其模型，grader 定义奖励）；Predibase / Together / Fireworks 微调 | 免运维，按 token 计费 |
| 复现 DeepSWE 规模 | 64×H100 × 1 周 | 数万美元 |
| 不要碰 | 600B+ MoE 全参 RL | 厂商的事 |

候选开源底座（许可宽松）：Qwen3-Coder-30B-A3B（Apache-2.0）、GLM-4.5-Air（MIT）、gpt-oss-120b / 20b（Apache-2.0）、MiniMax-M2、DeepSeek 开源权重（MIT）。

**要踩的坑**：奖励作弊（LLM 评判会被钻空子，优先用可执行验证器）；灾难性遗忘（混通用数据、LoRA 而非全参）；过拟合 harness 版本（工具 schema 冻结 + 版本化，改 schema 即回归评测）；仿真到实钻的 sim-to-real 差距（奖励里加实测井偏差项）；数据治理（会话含代码与井数据，训练前脱敏）。

#### 5.4.3 数据从哪里来

| 来源 | 例子 | 给什么 | 注意 |
|---|---|---|---|
| 1. 公开环境套件（真实仓库 + 测试） | SWE-Gym（2.4k）、SWE-smith（5 万+ 合成 bug）、R2E-Gym（8k+）、SWE-rebench（持续更新）、Multi-SWE-bench / Multi-SWE-RL（多语言）、Terminal-Bench 2.0 / Harbor 任务 | 通用编码与终端能力的任务 + 验证器 | 大多 Python 为主；可直接接到 open-model-gym 的 scenario 格式 |
| 2. 工具调用合成数据 | APIGen-MT（模拟人–agent 多轮）、ToolACE、Toucan（从约 500 个真实 MCP 服务器合成 150 万条）、Kimi K2 的"模拟工具 + 模拟用户 + rubric 评判"管线 | 工具调用格式、多轮、MCP 工具分布 | 合成数据的验证器弱，只适合 SFT 不适合 RL 主奖励 |
| 3. 自家 harness 遥测 | goose 会话（`sessions.db`、hooks）+ 结果标注（验证器 / 用户反馈 / 是否被回滚） | 真实任务分布、真实错误恢复 | 这正是 dsh 默认上传会话日志、Claude Code 消费者数据默认训练的原因；Pions 内部会话可合规使用，对外产品须明示并可退出 |
| 4. 教师蒸馏 | 强模型在目标工具画像上跑 → 验证器过滤 → SFT | 高质量轨迹最快的来源 | Anthropic / OpenAI 条款禁止用输出训练竞争模型；教师用 DeepSeek（MIT）、Qwen（Apache-2.0）、GLM（MIT）、Kimi K2（modified MIT） |
| 5. **领域数据（Pions 护城河）** | 物理仿真器（扭矩摩阻、水力、ROP、井眼轨迹）= **可验证奖励生成器**：任务 = 给定井况让 agent 配置 / 运行 / 解释仿真，奖励 = 物理一致性 + 约束满足 + 与历史井实测的偏差；历史井（日报、测井、BHA）→ 真实任务；参数扫描 → 无限合成井况 | 厂商永远不会训的那层 | 先冻结"仿真器工具"的 schema，再采集；奖励要有实测项防止只学会讨好仿真器 |
| 6. 人类示范与偏好 | 工程师用 goose 的真实会话、对结果的采纳 / 修改 | 偏好数据（DPO）、rubric 校准 | 量小但决定"像不像我们的工程师" |

**分阶段建议（Pions）**

| 阶段 | 做什么 | 训练量 |
|---|---|---|
| Q0（现在） | 实现工具画像（L0）与 `dsh-acp`（L1）；把 `open-model-gym` 扩到 100+ 研发 / 钻井场景并容器化；用它量化 L0/L1 收益 | 0 |
| Q1 | 用开源教师 + 拒绝采样在 goose 工具画像上采 5k–20k 条成功轨迹；LoRA SFT 一个 30B 级底座；部署为 goose provider | SFT |
| Q2+ | 只对领域 agent 做仿真器奖励的 GRPO；通用 harness 对齐继续交给厂商 + L0/L1 | RL |

**goose 自身的战略含义**：dsh 极简模式（单个持久 bash 工具）不是 UX 选择而是训练选择——最小动作空间最便宜地扩环境、最容易泛化到标准模式。goose 若发布一个**冻结的最小工具画像 + 开放环境套件**（open-model-gym 的方向），让 Qwen / GLM / Kimi / MiniMax 等在其上做后训练，就能成为中立（AAIF / Linux Foundation）的"开源模型训练 harness"——这是路径 3 的完整含义。

---

## 6. 附录

### 6.1 goose 源码索引（本次调研触达的关键文件）

| 主题 | 路径 |
|---|---|
| 主循环、并发工具执行、goal/grind | `crates/goose/src/agents/agent.rs` |
| 检查器链 / 权限 / 重复熔断 | `crates/goose/src/tool_inspection.rs`, `permission/*.rs`, `tool_monitor.rs`, `config/goose_mode.rs` |
| 安全 | `crates/goose/src/security/*.rs`, `agents/extension_malware_check.rs`, `documentation/docs/guides/sandbox.md` |
| 开发者工具 | `crates/goose/src/agents/platform_extensions/developer/{mod,shell,edit,tree,image}.rs` |
| 平台扩展注册 | `crates/goose/src/agents/platform_extensions/mod.rs` |
| Computer control / Office 工具 / Peekaboo | `crates/goose-mcp/src/computercontroller/`, `crates/goose-mcp/src/peekaboo/mod.rs` |
| 记忆 / 历史检索 / 提示文件 / MOIM | `crates/goose-mcp/src/memory/mod.rs`, `agents/platform_extensions/chatrecall.rs`, `hints/load_hints.rs`, `agents/moim.rs` |
| 上下文压缩 / 大输出 | `crates/goose/src/context_mgmt/mod.rs`, `agents/large_response_handler.rs`, `prompts/compaction.md` |
| MCP 宿主 | `crates/goose/src/agents/extension_manager.rs`, `agents/mcp_client.rs`, `agents/extension.rs` |
| Provider 注册 / Anthropic 缓存 / Claude Code 与 ACP provider / DeepSeek 声明式 provider | `crates/goose/src/providers/init.rs`, `providers/formats/anthropic.rs`, `providers/anthropic.rs`, `providers/claude_code.rs`, `providers/claude_acp.rs`, `providers/codex_acp.rs`, `acp/provider.rs`, `providers/declarative/deepseek.json` |
| 子代理 / 编排 / 调度 | `agents/platform_extensions/{summon,orchestrator}.rs`, `agents/subagent_*.rs`, `scheduler.rs`, `agents/schedule_tool.rs` |
| Recipes / 重试 / 结构化输出 | `crates/goose/src/recipe/`, `agents/retry.rs`, `agents/final_output_tool.rs` |
| Skills / Plugins / Hooks / Review checks / `.claude` 兼容 | `crates/goose/src/skills/`, `plugins/`, `hooks/mod.rs`, `checks/mod.rs`, `sources.rs` |
| 会话存储 / 导入（含 Claude Code JSONL） | `crates/goose/src/session/session_manager.rs`, `session/import_formats/claude_code.rs` |
| 服务端 / ACP / SDK / 网关 | `crates/goose-server/src/{auth,session_event_bus}.rs`, `crates/goose/src/acp/`, `crates/goose-sdk*`, `crates/goose/src/gateway/` |
| Code Mode / Apps / MCP-UI | `agents/platform_extensions/code_execution.rs`, `goose_apps/`, `documentation/docs/guides/interactive-chat/mcp-ui.md` |
| 可观测性 | `crates/goose/src/tracing/`, `otel/`, `posthog.rs` |

### 6.2 Claude Code 关键事实（官方文档，2026-09）

| 主题 | 事实 |
|---|---|
| 版本 | v2.1.275（2026-09-17，第三方 changelog 汇总）；agent teams 研究预览始于 v2.1.32（2026-02-05） |
| 内置工具 | 约 35 个：Read/Edit/Write/Glob/Grep/Bash/PowerShell/Monitor/Agent/WebFetch/WebSearch/AskUserQuestion/Task*/Skill/ToolSearch/Workflow/Artifact/Cron*/SendMessage/… |
| 权限模式 | `default`(Manual)/`acceptEdits`/`plan`/`auto`(分类器；Pro/Max/Team 默认)/`dontAsk`/`bypassPermissions` |
| 沙箱 | seatbelt（mac）/ bubblewrap + socat（linux、WSL2）；写 cwd + 附加目录；读全盘可 deny；域名白名单代理；凭证掩码；原生 Windows 不支持 |
| 子代理 | 默认 20 并发、深度 3、后台默认；worktree 隔离；独立记忆；可 resume |
| Workflows | JS 脚本，默认 16 并发、≤1000 agent/次、≤4096 项/`parallel`；`/deep-research` 内置 |
| Routines | 云端；schedule / API / GitHub 触发；最小 1 小时；每日运行上限；2026-04 起研究预览 |
| Hooks | 33 个事件；command / http / mcp_tool / prompt / agent 五种类型 |
| 记忆 | CLAUDE.md 四级 + `.claude/rules/` 路径规则；auto memory `MEMORY.md` 前 200 行/25KB 加载，机器本地 |
| 上下文 | Sonnet 5 默认 1M；Fable/Opus 4.6+/Sonnet 4.6 `[1m]`；压缩后重注入 CLAUDE.md、记忆、plan、最近 5 文件、skill（≤5k/25k） |
| MCP | stdio/HTTP/SSE/WS；OAuth；local/project/user/managed 作用域；工具搜索延迟加载；输出 >25k tokens 落盘；2 分钟自动后台；Channels |
| 云会话 | Pro/Max/Team、Enterprise 高级席位；隔离 VM；无额外算力费；ZDR 组织不可用；`--cloud` / `--teleport` |
| Computer use | CLI 仅 macOS 研究预览（Pro/Max）；Desktop mac+win；按应用授权分级；Team/Enterprise 不可用 |
| 成本 | 企业平均约 $13/开发者/活跃日、$150–250/月（API）；订阅 5 小时 + 每周窗口共享 |
| Agent SDK | Python/TS；仅 API key；不得为第三方产品提供 claude.ai 登录；商业条款；品牌限制 |
| 第三方供应商 | Bedrock/Agent Platform/Foundry/Claude Platform on AWS：无云会话、routines、Desktop（除 3P）、Chrome、computer use；Bedrock 无 WebSearch |

### 6.3 Cowork 时间线（2026）

| 日期 | 事件 |
|---|---|
| 01-12 | 研究预览发布（macOS，Max 先行），"Claude Code for the rest of your work" |
| 01-30 | Plugins（skills + slash commands + connectors + sub-agents 打包） |
| 02 | Windows 全功能；Claude for PowerPoint；02-20 Enterprise 自助开通（SSO/SCIM） |
| 03-17 | Dispatch（手机遥控桌面会话） |
| 03-23 | Computer use 研究预览（后转 beta，Pro/Max，按应用授权） |
| 04-22 | 支持指向任意 `/v1/messages` 兼容端点并保留管理控制 |
| 05-07 | Excel/Word/PowerPoint 加载项 GA，Outlook beta |
| 07-07 | Web + iOS/Android beta；云执行成为默认；云端 scheduled tasks |
| 07-27 | SharedRoot VM 逃逸披露（CVE-2026-46331 链），约 50 万 macOS 本地会话用户，已修复 |
| 08-25 | 聊天与 Cowork 统一记忆（仅云会话） |
| 09-10 | Enterprise 默认开启 |
| 09-16 | Cowork 并入统一 Claude；Claude Docs / Slides / Design 发布 |

### 6.4 DeepSeek Harness 源码索引（克隆于会话 scratchpad，commit `ddefc45`）

| 主题 | 路径（仓库内） |
|---|---|
| 架构 / 循环 / 事件 | `docs/architecture.md`, `docs/agent-lifecycle.md`, `docs/event-producer-consumer.md`, `packages/core/agent-loop/README.md` |
| 工具目录（生成） | `docs/tool-catalog.md`；工具注册表 `packages/core/tools/README.md` |
| 工作模式 | `packages/preset/agent-presets/presets/{standard,ptc,minimal,cordis}/agent.cordis.yml` |
| 上下文 | `docs/subsystems/compaction.md`, `packages/compaction/compaction-basic/README.md`, `docs/subsystems/spill.md`, `docs/subsystems/token-meter.md` |
| 模型适配 / 缓存效应 | `packages/llm/llm-deepseek/README.md`, `packages/llm/llm-pi-ai/README.md`, `docs/user/guide/providers.md` |
| DeepSeek 请求扩展（会话日志上传） | `docs/deepseek-llm-api-wire-extensions.md`, `packages/session/session-log-deepseek/README.md`, `packages/session/session-telemetry-otel/README.md` |
| 沙箱 / 权限 | `docs/subsystems/{sandbox,approval,permission-presets}.md`, `packages/sandbox/README.md`, `native/system/README.md`, `packages/experimental/auto-review/README.md` |
| 子代理 / 编排 | `docs/subsystems/{subagent,agent-team,jobs,workflow,ptc-runtime,schedule,webhook,goal,plan}.md`, `packages/subagent/README.md`, `packages/subagent/subagent-claude-code/README.md` |
| 接入 | `packages/acp/acp/README.md`, `packages/sdk/README.md`, `python/README.md`, `packages/mcp/mcp-client/README.md`, `docs/subsystems/mcp.md`, `packages/hooks/hooks-claude-code/README.md`, `docs/subsystems/skills.md` |
| 界面 | `apps/cli/README.md`, `packages/bundle/web-app/README.md`, `apps/desktop/README.md` |
| 安全声明 | `SAFETY.md` |

### 6.5 资料来源

**Claude Code（官方文档，code.claude.com/docs/en/…）**：overview、changelog、tools-reference、permissions、permission-modes、sandboxing、security、mcp、hooks、memory、context-window、agents、sub-agents、agent-teams、workflows、routines、claude-code-on-the-web、desktop、chrome、computer-use、channels、costs、agent-sdk/overview、third-party-integrations、feature-availability。

**DeepSeek Harness**：github.com/deepseek-ai/deepseek-harness 源码（上表）；The New Stack "DeepSeek open sources an agent harness where everything is a plugin"；InfoQ "The Open-Sourcing of DeepSeek Harness…"；Pandaily "DeepSeek Harness Hands-On: Four Work Modes…"；CometAPI "DeepSeek V4 Flash 0731 & DeepSeek Harness"；findharness.com "Which Harness Runs DeepSeek V4 Best?"；cordiverse/cordis 与论文 "A Programming Paradigm for Spatiotemporal Composability"（arXiv 2608.25512）。基准数字均为第三方转述。

**Cowork**：
- VentureBeat：Anthropic launches Cowork（2026-01）；Anthropic is killing off Cowork and folding it into Claude chat, launching Claude Docs and Claude Slides（2026-09-16）
- TechRepublic / The Next Web / Engadget / BusinessToday：Cowork 并入 Claude 与 Docs/Slides 报道（2026-09-17）
- Claude Help Center：Get started with Claude Cowork；Use Claude Cowork on web, desktop, and mobile；Schedule recurring tasks；Let Claude use your computer in Cowork；Claude Cowork architecture overview；Use plugins in Claude；Get started with custom connectors using remote MCP；Use Claude Cowork on Team and Enterprise plans
- claude.com/docs/cowork/changelog；claude.com/blog/cowork-plugins；claude.com/connectors
- CSA Research Note / The Hacker News / 9to5Mac / AppleInsider / SOCRadar：SharedRoot sandbox escape（2026-07-27）
- Engadget / TechCrunch / The New Stack：Claude memory now works across chats and Cowork（2026-08-25）
- Forbes / DataCamp：Claude Dispatch（2026-03）
- VentureBeat / The Decoder / The New Stack：Claude for Excel/PowerPoint/Word/Outlook（2026-02 ~ 05）
- Simon Willison：First impressions of Claude Cowork（2026-01）

**goose**：本仓库源码；github.com/aaif-goose/goose v1.37.0 发布说明；aaif.io/projects/goose；goose-docs.ai。

**训练侧（5.4 引用的论文与发布）**：SWE-Gym（Pan et al., 2024）；SWE-smith（Yang et al., 2025）；R2E-Gym（Jain et al., 2025）；DeepSWE（Agentica × Together, 2025）；SWE-rebench（Nebius）；Multi-SWE-bench / Multi-SWE-RL（ByteDance）；Terminal-Bench 2.0 / Harbor；APIGen-MT、Toucan（Salesforce 等）；ToolACE；Kimi K2 技术报告（agentic 数据合成与联合 RL）；Qwen3-Coder 发布说明（2 万并行环境 agentic RL）；GLM-4.5 / slime；OpenAI codex-1 与 GPT-5-Codex 发布说明；Anthropic 消费者条款更新（2025-08-28 公告，09-28 生效）；Thinking Machines Tinker；OpenAI Reinforcement Fine-Tuning；veRL / SkyRL / prime-rl / AReaL。数字均取自各自论文或发布，未在本会话复现。

**第三方对比（谨慎引用）**：lowcode.agency、morphllm、theaiagentindex 的 goose vs Claude Code 文章（SWE-bench 数字非严格同条件）；softr.io / DataCamp / hatchworks 的 Cowork vs Claude Code 文章；gradually.ai / releasebot 的 Claude Code changelog 汇总。
