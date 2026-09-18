# goose vs Claude Cowork vs Claude Code：通用 Agent 能力全景对比（同模型假设）

> 调研日期：2026-09-18
> goose 依据：本仓库源码 v1.37.0（commit `0ab4b84`），文中文件路径均可在仓库中核对
> Claude Code 依据：code.claude.com 官方文档（本会话可直接抓取；覆盖 tools-reference、permission-modes、sandboxing、mcp、hooks、sub-agents、agent-teams、workflows、routines、claude-code-on-the-web、memory、context-window、costs、agent-sdk、feature-availability、changelog 等页面），版本参考 v2.1.275（2026-09-17）
> Cowork 依据：Anthropic 公开资料与媒体/安全研究报道（本会话网络代理禁止直接抓取 anthropic.com / support.claude.com，因此来自搜索摘要，关键结论已多源交叉）
> 重要时间点：**2026-09-16 Anthropic 宣布把 Cowork 并入统一的 Claude 体验**并推出 Claude Docs / Slides / Design。下文的 "Cowork" 指这套能力集合，而非一个独立产品名。

---

## 0. 3000 英尺结论（TL;DR）

假设三者背后是**同一个模型**，剩下的差异只来自：**harness（提示词、工具设计、上下文/记忆策略）**、**执行环境（本机 / VM / 云）**、**生态与治理（接入、权限、企业控制）**。

| 结论 | 说明 |
|---|---|
| **三者的关系** | Claude Code 与 Cowork 共用同一个 agentic 引擎（Anthropic 官方说法），Claude Code 是面向开发者的"全控制面"，Cowork 是面向知识工作者的"零配置面"。goose 是二者的开源替代品：正面对标 Claude Code（CLI / 桌面 / API / IDE），部分覆盖 Cowork（桌面、computercontroller、docx/xlsx/pdf 工具）。 |
| **harness 质量：Claude Code > Cowork ≥ goose** | Claude Code 有约 35 个内置工具（独立 `Read`/`Glob`/`Grep`、`Monitor`、`WebFetch` 隔离上下文、后台 Bash、检查点回滚）、plan mode、effort 分档、auto mode 分类器、1M 上下文与微压缩；goose 的 developer 扩展只有 5 个工具（write/edit/shell/tree/read_image），读文件靠 `cat/sed`。第三方博客引用的 SWE-bench 差距（Claude Code 72.7% vs goose ~45%，同 Sonnet）虽非严格 A/B，方向一致。 |
| **goose 的不可替代性** | 60+ provider + 本地 llama.cpp；全规格 MCP 宿主（含 sampling）；可作为 HTTP/ACP/SDK 服务被嵌入；Apache-2.0；可自定义发行版；完全可审计。Claude Code 的 Agent SDK 只允许 API key 鉴权、受商业条款约束、不得以 Claude Code 品牌出现；Cowork 完全没有 API。 |
| **Computer use / 浏览器：Cowork ≈ Claude Code > goose** | Cowork/Claude Desktop：内置浏览器、Claude in Chrome、按应用授权的 computer use（Pro/Max，mac+win，mac 可后台）。Claude Code CLI：`--chrome`、`computer-use` MCP（仅 macOS 研究预览）。goose：仅 macOS 靠 Peekaboo 像样，无内置浏览器。 |
| **文件与文档** | 代码/本地文件：Claude Code ≥ goose > Cowork（Cowork 受授权文件夹约束）。Office 文档：Cowork（Docs/Slides 编辑器 + Office 加载项 + 文档 Skills）> Claude Code（Skills + Artifacts）> goose（内置 docx/xlsx/pdf 工具）。 |
| **记忆** | Claude Code：CLAUDE.md 层级 + 自动记忆（MEMORY.md 索引 + 主题文件，默认开启，机器本地不同步）+ 子代理记忆。Cowork：账号级统一记忆（仅云端会话）。goose：显式 memory 扩展 + 历史会话全文检索 + `.goosehints/AGENTS.md` + 每轮注入的 MOIM。 |
| **效率 / 成本** | Claude Code 的成本工程最深（prompt cache 统计、按 skill/子代理/MCP 归因、OTel、effort/fast mode、子代理换小模型、MCP 工具定义延迟加载）；goose 次之且完全透明（按会话累计 token/费用、Code Mode）；三者中只有 goose 能通过换模型把单价降下来。订阅制下 Claude Code 与 Cowork 共享 5 小时 + 每周限额。 |
| **安全** | Claude Code：seatbelt/bubblewrap 沙箱（mac/linux/WSL2，原生 Windows 不支持）+ 域名白名单代理 + auto mode 分类器 + 托管策略；Cowork：本地 Linux VM 或云端 VM（2026-07 SharedRoot 逃逸后云执行成默认）；goose：进程内检查器链 + 仅 macOS 的 seatbelt 沙箱。 |
| **对 Pions 的含义** | 编码/研发：Claude Code。嵌入产品 / 私有化 / 换模型：goose（或 Claude Agent SDK，若锁定 Claude）。非技术同事的文档工作：Cowork（现在的 Claude）。三者共用 **MCP server + SKILL.md + `.claude/agents` + AGENTS.md**，这是最值得投资的公共层。 |

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

模型：三者中 Claude Code / Cowork 仅 Claude（Sonnet 5 默认、Opus 5、Fable 5.1 单独计费）；企业可指向 Bedrock / Vertex / Foundry 等兼容端点。

---

## 2. 逐维度深度对比

每个维度：**goose 实现（含源码位置）→ Claude Code 实现（官方文档）→ Cowork 实现 → 判断**。评分 1–5，是"同模型"前提下对 harness / 环境 / 生态的评估。

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

**goose**：`providers/init.rs` 注册 ~35 个代码 provider + 29 个声明式 JSON provider（Anthropic、OpenAI、Google/Vertex、Azure、Bedrock、SageMaker、Databricks、Ollama、OpenRouter、LiteLLM、HuggingFace、Snowflake、xAI、DeepSeek、Groq、Mistral、Cerebras、NVIDIA、Zhipu/Z.ai、MiniMax、Moonshot、Alibaba、Perplexity、LM Studio…）；**本地推理**（llama.cpp，CUDA/Vulkan）；`toolshim` 给无原生工具调用的小模型；**ACP providers**（`claude_acp` / `codex_acp` / `copilot_acp` / `amp_acp` / `pi_acp`）把整个 Claude Code / Codex harness 当模型用并复用订阅；主 / fast / planner / editor 多模型分工。

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

## 3. 评分矩阵（同模型前提）

| 维度 | goose | Cowork | Claude Code | 决定性因素 |
|---|:---:|:---:|:---:|---|
| Agent 循环 / 推理 harness | 3.5 | 4.5 | 5 | Claude Code 工具面最完整并为 Claude 深度调优 |
| 本地文件 / 代码操作 | 4.5 | 4 | 5 | Read/Glob/Grep/检查点 vs goose 无边界 shell vs 授权文件夹 |
| Office 文档产出 | 3 | 5 | 3.5 | Docs/Slides 编辑器 + Office 加载项 |
| Computer use / 浏览器 | 2.5 | 4.5 | 4 | Chrome 集成 + 桌面内置浏览器 + 按应用授权 |
| 分析与产出形态 | 3.5 | 4 | 4.5 | LSP / ultrareview / deep-research / Artifacts |
| 运行效率 / 成本可控 | 4 | 3 | 4.5 | 可观测性 Claude Code 最强；换模型杠杆只有 goose 有 |
| 记忆 | 3 | 3.5 | 4 | CLAUDE.md 层级 + auto memory + 子代理记忆 |
| 上下文管理 | 4 | 4 | 5 | 1M、微压缩、压缩保留清单、rewind 摘要 |
| MCP / API 接入 | 5 | 3.5 | 4.5 | goose 最开放；SDK 有商业/鉴权限制；Cowork 无 API |
| 多 Agent / 调度 | 4 | 4 | 5 | 子代理/teams/workflows/routines/channels |
| 安全与治理 | 4 | 4 | 4.5 | 沙箱 + 分类器 + 托管策略 vs VM vs 检查器链 |
| 模型选择 | 5 | 1.5 | 2 | 60+ provider + 本地模型 vs 仅 Claude |
| 平台入口 | 4.5 | 4.5 | 5 | CLI/IDE/桌面/Web/移动/Slack/CI 全覆盖 |
| 可观测 / 可审计 | 4.5 | 2.5 | 4.5 | 开源 vs OTel + 分析 API |
| 可嵌入 / 二次开发 | 5 | 1 | 4 | goosed/ACP/SDK/发行版 vs Agent SDK(API key、商业条款) vs 无 |
| 生态 | 4 | 4 | 5 | 官方+社区市场规模 |

---

## 4. 对 Pions（物理 + AI，钻井场景）的落地建议

### 4.1 按用途选型

| 用途 | 推荐 | 理由 |
|---|---|---|
| 研发 / 编码 / 代码审查 / CI 自动化 | **Claude Code** | harness 最强；routines（API/GitHub 触发）、GitHub Actions、`/ultrareview`、worktree 并行 |
| 把 agent 嵌进 Pions 产品（钻井仿真 / 实时数据 / 优化建议），可能私有化或离线，或需要换模型 | **goose 内核**（goosed API / ACP / 自定义发行版）；若锁定 Claude 且接受商业条款与 API key 计费，可选 Claude Agent SDK / Managed Agents | Cowork 无 API；Agent SDK 不得用订阅、不得以 Claude Code 品牌出现、无法接非 Claude 模型；goose 可接本地模型与任何 provider |
| 需要 Claude Code harness 但保留多 provider / recipes / 可审计 | goose + `claude_acp` provider | 用 goose 壳包 Claude Code 引擎，扩展以 MCP 透传 |
| 销售 / 市场 / 法务 / 管理的文档工作 | **Cowork（现 Claude 统一体验）** | Docs/Slides/Office 加载项、38+ connectors、零配置 |
| 无人值守定时处理 | 数据在云 SaaS/GitHub → Claude Code routines 或 Cowork 云任务；数据在内网/边缘 → goose cron + recipe `retry/checks`，或 Claude Code 自托管环境 | 云任务碰不到内网；goose 调度需进程常驻 |
| 需要审计模型每一步 | goose 或 Claude Code（OTel） | goose 可审代码；Claude Code 可导出指标与转录 |

### 4.2 一次投入、三边通吃的公共层

1. **MCP server**：把钻井仿真器、实时数据（WITSML/OPC）、地质模型封装成 MCP（Streamable HTTP + OAuth 最通用）。goose 与 Claude Code 直接接；Cowork 以"自定义远程 connector"接（需公网可达，或桌面端 stdio 本地 MCP）。
2. **SKILL.md**：钻井 SOP、压力窗口计算流程、报告模板写成 agentskills.io 规范技能，放 `.claude/skills/`——goose 与 Claude Code 都读；Cowork/Claude 用同一规范。
3. **子代理定义**：`.claude/agents/*.md`——goose 的 summon 与 Claude Code 都读。
4. **项目指令**：写 `AGENTS.md`（goose 原生读），`CLAUDE.md` 里 `@AGENTS.md` 导入（Claude Code 官方推荐做法）。
5. **会话可迁移**：goose 能导入 Claude Code 的 JSONL 转录，便于统一归档与审计。

### 4.3 需要提前认清的坑

- goose：harness 调优是自己的活（可从改 `prompts/*.md` 与 developer 指令起步）；Linux/Windows 无沙箱、computer use 弱；调度依赖进程常驻；桌面沙箱仅 macOS；工具面薄（无原生 read/grep/web）。
- Claude Code：仅 Claude；第三方供应商上功能明显缩水（无云会话、routines、Chrome、computer use、Bedrock 无 WebSearch）；Agent SDK 只能 API key 计费且不得复用订阅；agent teams / routines / channels / computer use 多为研究预览；原生 Windows 无沙箱；订阅限额与 Chat/Cowork 共享。
- Cowork：数据默认进 Anthropic 云（含 connector 调用），钻井作业者的数据主权条款要先过；订阅限额在长任务上是瓶颈；产品形态正在变（9/16 合并）；无 API、不可白标、不可换模型。

### 4.4 一个可行的混合架构

```
Pions 平台(内网/边缘)                      研发                      办公侧
┌────────────────────────────┐   ┌──────────────────────┐   ┌──────────────────────┐
│ goosed (自定义发行版)       │   │ Claude Code          │   │ Claude(原 Cowork)    │
│  ├ provider: 本地/私有模型  │   │  CLI/IDE/routines    │   │  connectors: 邮件/   │
│  │   或 Claude via Bedrock  │   │  .claude/agents      │   │  Slack/CRM/Drive     │
│  ├ MCP: 仿真器/实时数据/    │◄──┼──MCP + SKILL.md──────┼──►│  自定义 connector →  │
│  │   地质模型/报告生成       │   │  CLAUDE.md @AGENTS.md│   │  Pions 数据(只读)    │
│  ├ recipes + cron: 日报/预警 │   │  sandbox + hooks     │   │  Skills: 同一套      │
│  └ hooks: 审计/合规拦截      │   │  OTel → 统一观测     │   │  Docs/Slides 产出    │
└────────────────────────────┘   └──────────────────────┘   └──────────────────────┘
```

---

## 5. 附录

### 5.1 goose 源码索引（本次调研触达的关键文件）

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
| Provider 注册 / Anthropic 缓存 / Claude Code 与 ACP provider | `crates/goose/src/providers/init.rs`, `providers/formats/anthropic.rs`, `providers/anthropic.rs`, `providers/claude_code.rs`, `providers/claude_acp.rs`, `providers/declarative/*.json` |
| 子代理 / 编排 / 调度 | `agents/platform_extensions/{summon,orchestrator}.rs`, `agents/subagent_*.rs`, `scheduler.rs`, `agents/schedule_tool.rs` |
| Recipes / 重试 / 结构化输出 | `crates/goose/src/recipe/`, `agents/retry.rs`, `agents/final_output_tool.rs` |
| Skills / Plugins / Hooks / Review checks / `.claude` 兼容 | `crates/goose/src/skills/`, `plugins/`, `hooks/mod.rs`, `checks/mod.rs`, `sources.rs` |
| 会话存储 / 导入（含 Claude Code JSONL） | `crates/goose/src/session/session_manager.rs`, `session/import_formats/claude_code.rs` |
| 服务端 / ACP / SDK / 网关 | `crates/goose-server/src/{auth,session_event_bus}.rs`, `crates/goose/src/acp/`, `crates/goose-sdk*`, `crates/goose/src/gateway/` |
| Code Mode / Apps / MCP-UI | `agents/platform_extensions/code_execution.rs`, `goose_apps/`, `documentation/docs/guides/interactive-chat/mcp-ui.md` |
| 可观测性 | `crates/goose/src/tracing/`, `otel/`, `posthog.rs` |

### 5.2 Claude Code 关键事实（官方文档，2026-09）

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

### 5.3 Cowork 时间线（2026）

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

### 5.4 资料来源

**Claude Code（官方文档，code.claude.com/docs/en/…）**：overview、changelog、tools-reference、permissions、permission-modes、sandboxing、security、mcp、hooks、memory、context-window、agents、sub-agents、agent-teams、workflows、routines、claude-code-on-the-web、desktop、chrome、computer-use、channels、costs、agent-sdk/overview、third-party-integrations、feature-availability。

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

**第三方对比（谨慎引用）**：lowcode.agency、morphllm、theaiagentindex 的 goose vs Claude Code 文章（SWE-bench 数字非严格同条件）；softr.io / DataCamp / hatchworks 的 Cowork vs Claude Code 文章；gradually.ai / releasebot 的 Claude Code changelog 汇总。
