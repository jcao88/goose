# goose vs Claude Cowork：通用 Agent 能力全景对比（同模型假设）

> 调研日期：2026-09-18
> goose 依据：本仓库源码 v1.37.0（commit `0ab4b84`），文中给出的文件路径均可在仓库中核对
> Cowork 依据：Anthropic 公开资料与媒体/安全研究报道（截至 2026-09-18；本会话网络代理禁止直接抓取 anthropic.com / support.claude.com，因此 Cowork 侧信息来自搜索摘要，关键结论已多源交叉）
> 重要时间点：**2026-09-16 Anthropic 宣布把 Cowork 并入统一的 Claude 体验**（Claude 自行决定"聊天"还是"执行任务"），并推出 Claude Docs / Slides / Design。下文的 "Cowork" 指这套能力集合，而非一个独立产品名。

---

## 0. 3000 英尺结论（TL;DR）

假设两者背后是**同一个模型**，剩下的差异就是三件事：**harness（提示词、工具设计、上下文/记忆策略）**、**执行环境（本机/VM/云）**、**生态与治理（接入、权限、企业控制）**。

| 结论 | 说明 |
|---|---|
| **goose 是"开放的 agent 平台"，Cowork 是"封装好的 agent 产品"** | goose：Apache-2.0、Rust 内核、60+ provider、MCP 全规格客户端、可作为 HTTP/ACP/SDK 被嵌入。Cowork：闭源、仅 Claude、无对外 API、只能作为最终用户产品使用。 |
| **同模型下，Cowork 的 harness 质量大概率仍优于 goose** | Cowork 复用 Claude Code 的 agentic 架构（Anthropic 官方说法），针对 Claude 深度调优；goose 的 prompt/工具是通用型设计（例如没有独立 `read` 工具，靠 `cat`/`sed`）。第三方博客引用的 SWE-bench 差距（Claude Code 72.7% vs goose ~45%，同 Sonnet）虽非严格 A/B，但方向一致。 |
| **Computer use / 浏览器：Cowork 明显领先** | Cowork 内置浏览器、Claude in Chrome、按应用授权的 computer use（beta，2026-03 起，9 月新增后台运行）。goose 在 macOS 上靠 Peekaboo CLI 做 GUI 自动化，Windows/Linux 退化为 PowerShell/xdotool 脚本，浏览器完全依赖 MCP（Playwright/Chrome DevTools）。 |
| **文件与 Office 文档：各有强项** | goose 拥有无限制 shell + 精确 edit + 内置 docx/xlsx/pdf 工具；Cowork 以"授权文件夹"为边界，但 Office 文档生成/编辑（Skills、Docs/Slides 原生编辑器、Excel/PPT/Word 加载项）远强于 goose。 |
| **记忆：Cowork 8 月起有统一的自动记忆，但仅云端会话可用；goose 是显式、可审计的文件/DB 记忆** | goose：memory 扩展（分类 markdown）、chatrecall（SQLite 全文检索历史会话）、`.goosehints`/`AGENTS.md`、每轮注入的 MOIM。 |
| **效率/成本：goose 可控可见，Cowork 受订阅限额约束** | goose 有 prompt caching、工具输出后台摘要、Code Mode（用代码批量调工具省 token）、按会话累计 token/费用；Cowork 是 5 小时 + 每周限额（与 Claude Code/聊天共享），用户普遍反映 token 消耗快。 |
| **安全：思路不同** | goose 是"进程内检查器链 + 可选 macOS seatbelt 沙箱 + 出口代理"，Linux/Windows 无沙箱；Cowork 是"本地 Linux VM 或云端执行"，2026-07 曝出 SharedRoot VM 逃逸（约 50 万 macOS 用户受影响，已修复，且 7 月 7 日起云执行为默认）。 |
| **对 Pions 的含义** | 要**嵌进产品、跑在钻井现场/私有化、接本地模型或物理仿真器**：只有 goose（或 Claude Agent SDK）能做。要**给非技术同事做销售/市场/法务文档工作**：Cowork（现在的 Claude）开箱即用更强。两者的 **MCP server 和 SKILL.md 可以共用**，这是最值得投资的公共层。 |

---

## 1. 两者定位与架构

### 1.1 goose（AAIF / Linux Foundation，原 Block）

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

- 核心事实：`crates/goose/src/agents/agent.rs`（3.4k 行）是主循环；`extension_manager.rs`（2.9k 行）是 MCP 宿主；`goose-server`（`goosed`）以 `X-Secret-Key` 鉴权、SSE 流式回复（带重放缓冲，`session_event_bus.rs`）。
- 一切都在**用户自己的机器/服务器上**运行，模型通过 provider 远程或本地（llama.cpp）调用。

### 1.2 Claude Cowork（Anthropic Labs，2026-01-12 发布）

```
┌───────── 交互面 ─────────┐
│ Claude Desktop(mac/win)  │
│ claude.ai Web / iOS / Android (2026-07-07 beta) │
│ Dispatch(手机遥控桌面会话, 2026-03-17) │
│ Office 加载项(Excel/PPT/Word GA 2026-05-07, Outlook beta) │
└────────────┬─────────────┘
             │
┌────────────▼──────────────────────────────┐
│ "与 Claude Code 相同的 agentic 架构"       │
│  执行位置：                                │
│   · 云端(2026-07-07 起默认)：agent loop +  │
│     代码执行在 Anthropic 服务器            │
│   · 本地：桌面端内置 Linux VM              │
│  文件：授权文件夹(本地 VirtioFS 挂载 /     │
│        云会话需桌面 App 保持打开才能读写)  │
│  Skills(docx/xlsx/pptx/pdf 等), Plugins,   │
│  Connectors(38+ 远程 MCP, 自 Anthropic 云  │
│  发起), 本地 stdio MCP(桌面端, 可策略禁用) │
│  Sub-agents, Scheduled tasks(云端),        │
│  Computer use(beta), 内置浏览器/Chrome     │
└───────────────────────────────────────────┘
```

- 2026-09-16 起 Cowork 与聊天合并为一个 Claude；Claude Docs / Slides / Design 成为原生产物编辑器；Pro/Max 先行，Team/Free 随后。
- 模型：仅 Claude（Sonnet 5 默认、Opus 5、Fable 5.1 单独计费）；企业版可指向 Bedrock/Vertex/Azure 等兼容 `/v1/messages` 的端点（2026-04-22 文档）。

---

## 2. 逐维度深度对比

每个维度：**goose 的实际实现（含源码位置）→ Cowork 的实现 → 差距判断**。评分 1–5（5 最强），是在"同模型"前提下对 harness/环境/生态的评估，不评模型本身。

### 2.1 Agent 循环、规划与推理 harness

**goose**
- 主循环 `agent.rs::reply_internal`（约 L1616–2300）：每轮 = 注入 MOIM → 调 provider 流式生成 → 工具请求进入 `ToolInspectionManager`（`tool_inspection.rs`）→ 通过的工具**并发执行**（`stream::select_all`，`agent.rs:1967`）→ 结果回灌。默认最大 1000 轮无人干预（`GOOSE_MAX_TURNS`，L1692）。
- 检查器链：permission（权限模式/逐工具规则/LLM 判定只读）、repetition（重复调用熔断，`tool_monitor.rs`）、security（提示注入）、adversary（自定义规则的 LLM 审计）、egress（命令出口目标提取）。
- 自我驱动：`/goal`（完成后自评是否达标）和 grind（"未完成就继续干"循环，`agent.rs:2254–2293`）；recipe 的 `retry`+`checks`+`on_failure`（`agents/retry.rs`，shell 成功检查、最多 N 次）。
- 规划：独立 planner 模型（`GOOSE_PLANNER_MODEL`，`prompts/plan.md` 一次性输出计划或澄清问题）；Todo 平台扩展在 MOIM 中每轮可见。
- 思考控制：`CLAUDE_THINKING_TYPE=adaptive|enabled|disabled`、`CLAUDE_THINKING_BUDGET`（Anthropic/Databricks provider）；Gemini 3 thinking level。
- 提示词：`prompts/system.md` 是通用的"你是 goose"+扩展说明清单；developer 扩展指令让模型自己用 `rg/cat/sed` 读文件（没有独立 read 工具，见 `developer/mod.rs:101–175`：write/edit/shell/tree/read_image）。
- 结构化输出：recipe `response.json_schema` → `recipe__final_output` 工具强制校验（`final_output_tool.rs`）。

**Cowork**
- 直接复用 Claude Code 的 harness：读/写/编辑/搜索工具、Task/子代理、计划模式、compaction、hooks、skills，全部围绕 Claude 训练分布设计（Anthropic 内部用 Claude 自身反复调 harness）。
- 任务模式：先出计划、进度跟踪、并行 workstream；effort 档位可调；Skill 可指定使用的模型。
- 无法自定义系统提示/循环参数（只能通过 skills/plugins/记忆/项目指令影响）。

**判断**：同模型下，Cowork/Claude Code 的 harness 更"贴模型"，这是 goose 最大的软肋，也是它 `claude_acp` provider 存在的原因（见 2.12）。goose 的优势是循环**完全可编程**：max turns、检查器、goal/grind、retry、结构化输出、可替换 prompt 模板（`prompt-templates` 文档）。

评分：goose 3.5 / Cowork 4.5

### 2.2 文件处理

**goose**
- 工具：`write`（自动建目录）、`edit`（before 文本必须唯一精确匹配；可选 `GOOSE_EDITOR_MODEL` 走 Morph 等"fast-apply"模型）、`shell`（每流最多 2000 行，超出落临时文件，`shell.rs:150`；`timeout_secs`；`GOOSE_SHELL` 可换 shell）、`tree`（尊重 .gitignore，带行数）、`read_image`（png/jpeg/gif/webp，本地或 URL，供视觉模型看图）。
- 大结果保护：单次工具结果 >200k 字符自动写文件并只回传路径（`large_response_handler.rs`）。
- 办公文档：computercontroller 内置 `xlsx_tool`（读/写/公式/多 sheet）、`docx_tool`、`pdf_tool`（`goose-mcp/src/computercontroller/*_tool.rs`）；`web_scrape` 抓网页/API 存缓存。
- 访问范围：**整机**（工作目录只是默认 cwd 和 MCP roots），没有"文件夹授权"概念；macOS 可开 seatbelt 沙箱写保护 `~/.ssh`、shell rc、goose 配置。
- 桌面端 `@` 模糊文件搜索（5 层目录）；文件读取还是靠 shell。
- 代码理解：`analyze` 扩展用 tree-sitter 做目录概览/文件符号/调用图，支持 rust/python/js/ts/tsx/go/java/kotlin/swift/ruby 10 种语言（`platform_extensions/analyze/languages.rs`）。

**Cowork**
- 边界是**用户授权的文件夹**；本地模式下文件夹通过 VirtioFS 挂进 Linux VM；云模式下"只有桌面 App 打开且会话在桌面发起"才能读写本地文件夹，否则会话继续跑但摸不到本地文件。
- Office 能力：Anthropic 官方 Skills（docx/xlsx/pptx/pdf）、9 月起 Claude Docs/Slides/Design 原生可编辑产物（可导出 Google Docs / Word）、Excel/PPT/Word/Outlook 加载项共享上下文。
- 桌面文件读取无 shell 暴露给用户，但 VM 内有完整 bash 供 Claude 使用。

**判断**：作为"开发者/工程师的文件操作"goose 更自由（任意路径、任意脚本、精确编辑、超大输出保护）；作为"知识工作者的文档产出"Cowork 完胜（Docs/Slides 编辑器 + Office 深度集成 + 更成熟的文档 Skills）。

评分：本地文件/代码 goose 4.5 vs Cowork 4；Office 文档 goose 3 vs Cowork 5

### 2.3 Computer use 与浏览器

**goose**
- `computer_control` 工具按 OS 分流（`computercontroller/mod.rs:353–500`）：
  - macOS：自动 `brew install steipete/tap/peekaboo`（`goose-mcp/src/peekaboo/mod.rs`），提供 `see --annotate`（带元素 ID 的标注截图）、click/type/press/hotkey/paste/scroll/drag、app/window/menu/menubar/dock/dialog/clipboard/space 等——这是相当完整的 macOS UI 自动化。
  - Windows：PowerShell/WMI/注册表脚本 + 截图；无元素级定位。
  - Linux：xdotool/wmctrl/D-Bus 脚本；无显示则禁用。
- `automation_script`：Shell/Ruby/AppleScript（mac）、PowerShell/Batch（win）、Shell（linux）。
- 浏览器：**没有内置浏览器**，靠 MCP 目录里的 Playwright / Chrome DevTools / Browserbase / Fetch / Firecrawl 等扩展；`web_scrape` 只是简单 HTTP GET。
- 无"按应用授权"的细粒度 UI 权限，靠权限模式（approve/smart_approve）和逐工具规则。

**Cowork**
- 内置浏览器（桌面 App）或通过 Claude in Chrome 使用用户自己的 Chrome；没有 connector 的工具直接在浏览器里操作。
- Computer use：2026-03-23 研究预览 → beta（Pro/Max）；对每个应用首次访问弹窗 Deny / Always allow / Allow once；用户可随时看到并中止；2026-09 支持后台运行。要求机器唤醒、App 打开。
- Windows/macOS 同等支持。

**判断**：Cowork 领先一个代际，尤其是浏览器和跨平台一致性。goose 在 macOS 上因 Peekaboo 不差，但 Windows/Linux 基本是"写脚本 + 截图看"的水平。

评分：goose 2.5 / Cowork 4.5

### 2.4 分析、推理与产出形式

**goose**
- 数据/可视化：`autovisualiser` 扩展（图表、地图、图谱 UI）、Apps 扩展（把 HTML 小应用当 MCP App 资源在沙箱窗口运行）、MCP Apps / MCP-UI 渲染。
- 代码分析：tree-sitter 调用图；`summarize` 扩展一次调用把文件/目录喂给 LLM 摘要。
- 多模型分工：主模型 + `GOOSE_FAST_MODEL`（摘要、会话命名、工具调用标题、权限判定）+ planner + editor 模型，可以"贵模型推理、便宜模型打杂"。
- 输出：Markdown/JSON/stream-json（CLI `goose run`），结构化 JSON schema 产物。

**Cowork**
- 产物：Claude Docs/Slides/Design、Office 文件、artifacts；Data/Finance/Legal 等官方 plugins 自带领域 skills 与 sub-agents。
- 同一会话可混用 skill 指定的模型；effort 控制。

**判断**：推理本身同模型无差；差别在"产出形态"——Cowork 面向文档/幻灯片/设计稿，goose 面向代码、数据、可交互 HTML 应用和结构化 JSON。

评分：goose 3.5 / Cowork 4（面向知识工作）

### 2.5 运行效率与成本

**goose**
- Rust 单二进制，本地执行零网络往返（除模型调用）；SSE 流式；工具并发执行。
- Anthropic 专项优化：prompt caching 打在 system、最后一个 tool 定义、最后两条 user 消息（`providers/formats/anthropic.rs:281–349`）；`token-efficient-tools` 与 `output-128k` beta 头（`anthropic.rs:184–187`）。
- 上下文瘦身：工具输出后台"配对摘要"（超过 cutoff 后每 10 个一批，`context_mgmt/mod.rs:22,434`）；Code Mode（`code_execution.rs`，pctx/Deno 运行时，LLM 写 JS 批量调工具，只暴露 3 个元工具，适合 5+ 扩展）；shell 2000 行截断；>200k 字符落盘。
- 成本可见：sessions.db 记录每会话 input/output/累计 token 与费用（`session_manager.rs` Session 字段），CLI `GOOSE_CLI_SHOW_COST`；tiktoken 本地估算兜底（`token_counter.rs`, `usage_estimator.rs`）。
- 成本模型：按 API token 付费，或通过 ACP provider 复用 Claude/ChatGPT 订阅（`acp-providers.md`）。

**Cowork**
- 云执行可与本机解耦、并行 sub-agents 由 Anthropic 基础设施承载。
- 成本模型：订阅限额（5 小时窗口 + 每周，Chat/Code/Cowork 共享；Team 高级席位 6.25× Pro；Enterprise $20/席 + 按 API 费率计量）。
- 用户反馈集中在：会话启动就读整个文件夹、历史重复读取导致 token 飙升；桌面 App 在大上下文下卡顿。

**判断**：goose 的效率工程更透明、可调，成本可精确归因；Cowork 的效率取决于 Anthropic 的 harness 与限额策略，用户无杠杆。

评分：goose 4 / Cowork 3

### 2.6 记忆

**goose**（全部显式、本地、可审计）
- `memory` 扩展：`remember_memory/retrieve_memories/remove_*`，分类 + 标签，存 `.goose/memory/`（项目）或 `~/.config/goose/memory/`（全局）的 markdown（`goose-mcp/src/memory/mod.rs:336–430`）；指令要求保存前先确认。
- `chatrecall`：对 SQLite `sessions.db` 的历史会话做关键词检索/按 session 加载首尾消息（`platform_extensions/chatrecall.rs`, `session/chat_history_search.rs`）。
- 持久指令：`.goosehints` + `AGENTS.md`（全局/项目/随工具触达的子目录动态加载，`hints/load_hints.rs`）；MOIM 每轮注入时间、cwd、todo 及 `GOOSE_MOIM_MESSAGE_TEXT/FILE`（≤64KB）。
- 会话：resume/fork/export/import，能导入 Claude Code / Codex / Pi 的 JSONL 记录（`session/import_formats/`）；`projects.json` 记录目录与最近会话；nostr 分享会话（feature）。
- 没有"自动跨会话学习用户"的隐式记忆。

**Cowork**
- 2026-08-25 起与 Claude 聊天共用一套账号级自动记忆：按主题分类的条目（角色、偏好、项目、纠正过的做法），Settings > Memory 可查看/编辑/删除。
- **限制**：仅云端会话可用，本地 VM 会话不可用；项目（Projects）提供组织与指令；无法像 goose 那样把记忆当文件放进 git。

**判断**：Cowork 的"零操作"记忆体验更好；goose 的记忆更适合团队共享、版本控制与合规审计。两者都没有向量/语义检索的长期记忆（goose 靠外部 MCP 如 Cognee/Knowledge Graph 补）。

评分：goose 3 / Cowork 3.5

### 2.7 上下文管理

**goose**
- 自动压缩阈值 80%（`GOOSE_AUTO_COMPACT_THRESHOLD`），压缩提示词可改（`prompts/compaction.md`，要求保留用户意图/文件/错误/待办/下一步）；手动 `/summarize` 或桌面"Compact now"。
- 超限兜底：summarize / truncate / clear / prompt 四策略（CLI），并有"逐步移除工具响应"的重试（`context_mgmt` 测试 `progressive_removal_on_context_exceeded`）。
- provider 若自管上下文（ACP providers）则跳过（`manages_own_context`）。
- 子代理各自独立上下文，只回传结果。

**Cowork**
- Claude Code 的 compaction（阈值可配）+ Anthropic 平台侧 compaction API；长任务被明确宣传为"数小时不丢上下文"；sub-agents 隔离上下文。
- 用户报告"context rot"仍存在，属模型/所有 harness 通病。

**判断**：机制等价，工程细节 goose 更可调；Cowork 在 1M 上下文与服务端压缩上有平台优势。

评分：goose 4 / Cowork 4

### 2.8 API / MCP 接入能力

**goose（作为 MCP 宿主）**
- 传输：stdio、Streamable HTTP（OAuth 授权码流 + 凭证存储 + 主动刷新、自定义 header 带环境变量替换、HTTP over Unix socket）、内置、进程内 platform、frontend（工具由 UI 执行）、inline python（uvx）；SSE 已弃用（`agents/extension.rs` `ExtensionConfig` 枚举）。
- 客户端能力：roots、sampling（`create_message`，即 MCP server 反向调模型）、elicitation、progress/logging 通知、resources、prompts（`mcp_client.rs:246–419`）；MCP Apps / MCP-UI 交互界面。
- 运行时管理：`extensionmanager` 平台扩展让模型自己搜索/启用/禁用扩展；扩展超时默认 300s；npm/PyPI 扩展先查 OSV 恶意包库（`extension_malware_check.rs`）；`GOOSE_ALLOWLIST` 企业白名单；危险环境变量黑名单（如 `LD_PRELOAD`）。
- 提示：超过 5 个扩展或 50 个工具会在系统提示中提醒收缩（`prompt_manager.rs:19–20`）；文档建议 <25 个工具。

**goose（作为可被接入的 API/Agent）**
- `goosed` REST + SSE（OpenAPI 生成 TS SDK，`ui/desktop/openapi.json`），可远程部署（TLS + 密钥）。
- ACP server（stdio / HTTP / WebSocket）——Zed、JetBrains、VS Code 直接把 goose 当 agent；ACP 自定义方法覆盖扩展管理、偏好、onboarding 导入等（`goose-sdk-types/src/custom_requests.rs`）。
- `goose-sdk` uniffi 绑定（Python/Kotlin，目前是脚手架）。
- Telegram 网关（文本 + 语音）、iOS 隧道。
- `goose run -t/-i/--recipe` 无头执行，`--output-format json|stream-json`，适合 CI/编排。

**Cowork**
- 38+ 官方 connectors（Google Workspace、Slack、M365、Salesforce、HubSpot、Jira、Notion、Confluence、Box、Dropbox、GitHub、Asana、Stripe、Figma、Canva、Snowflake…），本质是**远程 MCP，由 Anthropic 云发起连接**（服务器必须公网可达，VPN/内网不行）。
- 自定义远程 MCP connector（所有付费档可用）；桌面端支持本地 stdio MCP（MDM `isLocalDevMcpEnabled` 可禁用）。
- **没有驱动 Cowork 本身的 API**；程序化需求要去 Claude Agent SDK / Managed Agents / Claude Code。

**判断**：这是 goose 最强的维度——既是全规格 MCP 宿主，又能被当作服务/agent 嵌入。Cowork 的 connector 生态更大且免配置，但接入方向是单向的（工具进 Cowork，Cowork 不出去）。

评分：goose 5 / Cowork 3.5

### 2.9 多 Agent、编排与自动化

**goose**
- `summon` 扩展：`load`（列出/加载 recipes、subrecipes、`.md` agents、等待后台任务）+ `delegate`（ad-hoc 指令或按 source，可指定 provider/model/extensions/max_turns，`async: true` 后台并行，`GOOSE_MAX_BACKGROUND_TASKS` 默认 5，子代理默认 25 轮，`summon.rs`, `subagent_task_config.rs`）。
- `orchestrator`（隐藏扩展）：list/view/start/send_message/interrupt 其他 agent 会话——agent 操控 agent 的雏形。
- Recipes：YAML 打包 instructions/prompt/extensions/参数/settings/sub_recipes/response schema/retry；deeplink 分发；GitHub 仓库目录；`recipe-scanner` 安全扫描。
- 调度：内置 cron（`tokio-cron-scheduler`，5/6 字段，本地时区，`schedule.json`），支持 run_now/pause/kill/按调度看会话；模型可通过 schedule 工具自己建任务——**但只在本机 goosed/CLI 进程存活时运行**。
- `goose review`：`.agents/checks/*.md` 定义的子代理审查器 + `REVIEW.md`。
- Hooks：13 个生命周期事件（PreToolUse 可拒绝、Stop 可阻断继续、Subagent 起止、文件读写/命令前后等，`hooks/mod.rs`），遵循 Open Plugins 规范；插件支持 Gemini 与 open-plugins 两种格式并 24h 自动更新。

**Cowork**
- Sub-agents 并行 workstream；plugins 打包 skills + slash commands + connectors + sub-agents（11 个官方：Marketing/Sales/Data/Legal/Finance/HR/Ops/PM/Design/Dev/Office 等）；组织私有插件市场。
- Scheduled tasks：**云端运行，不需要电脑开机**，享有 connectors/skills/plugins 全部能力（本地文件除外）。
- Dispatch：手机下达指令给桌面会话。
- 无 recipe 式"可版本化的任务定义 + 成功校验 + 重试"。

**判断**：goose 的编排原语更丰富、可 git 化；Cowork 的调度因为在云端而更"可靠地无人值守"。

评分：goose 4 / Cowork 4

### 2.10 自主性、权限与安全

**goose**
- 权限模式：`auto`（默认，代码 `GooseMode::default`）/ `approve` / `smart_approve`（LLM 用 `platform__tool_by_tool_permission` 判定只读操作自动放行，`permission_judge.rs`）/ `chat`；逐工具 Always/Ask/Never；MCP `readOnlyHint` 注解直接采信。
- 检测：提示注入扫描（正则模式 + 可选 HuggingFace 文本分类端点，`security/scanner.rs`, `classification_client.rs`）；adversary 模式（`~/.config/goose/adversary.md` 写规则，LLM 审 shell 命令，失败时放行）；egress 检查器提取命令中的外联目标；重复调用熔断。
- 隔离：macOS seatbelt 沙箱 + 本地 CONNECT 出口代理（域名黑名单、禁裸 IP、SSH 仅限 git 主机、封 nc/socat）——**仅 macOS**；MCP 客户端可在 Docker 容器中启动（`connect_with_container`）；Linux/Windows 无系统级沙箱。
- 供应链：OSV 恶意包检查、扩展白名单、recipe 扫描；密钥存系统 keyring。
- 可审计：开源；Langfuse / OTel 追踪；诊断包导出。

**Cowork**
- 隔离：本地 Linux VM（VirtioFS 共享授权文件夹）或云端执行；2026-07-27 披露 SharedRoot（Linux 内核 CVE-2026-46331 提权 → 以 root 身份看到宿主机整个 `/` 的读写挂载），约 50 万 macOS 本地会话用户暴露，已修复；7 月 7 日起云执行默认，规避该类风险。
- 权限：文件夹授权、computer use 按应用授权、动作可视可中止；企业 MDM 策略（禁本地 MCP、禁桌面扩展）、SSO/SCIM、审计；Enterprise 2026-09-10 起默认开启。
- 数据流向：云执行 + connectors 由 Anthropic 云发起——数据驻留需按企业合同/区域配置。

**判断**：goose 的防线多但"软"（进程内检查 + 仅 mac 沙箱）；Cowork 的防线"硬"（VM/云）但闭源且有过逃逸先例。企业治理成熟度 Cowork 更高。

评分：goose 4 / Cowork 4（企业治理 goose 3.5 / Cowork 4.5）

### 2.11 平台与交互入口

| | goose | Cowork |
|---|---|---|
| 桌面 | macOS / Windows / Linux（zip/deb/rpm/flatpak） | macOS / Windows |
| 终端 | 完整 CLI + 新 TUI（Ink over ACP）+ `goose term` 每终端一会话 | 无（Claude Code 是另一产品） |
| Web / 移动 | 远程 goosed + Desktop、iOS 隧道（实验）、Telegram 网关 | claude.ai Web、iOS/Android（beta）、Dispatch |
| IDE | Zed / JetBrains / VS Code（ACP） | 无（Claude Code 负责） |
| Office | 无 | Excel/PPT/Word GA、Outlook beta |
| 语音 | 听写：OpenAI/Groq/ElevenLabs/本地 whisper | 移动端语音 |
| 国际化 | 多语言桌面（含简体中文） | 多语言 |

评分：goose 4.5 / Cowork 4.5（面向不同人群）

### 2.12 模型与供应商

**goose**：`providers/init.rs` 注册 ~35 个代码 provider + 29 个声明式 JSON provider：Anthropic、OpenAI、Google/Vertex、Azure、Bedrock、SageMaker、Databricks、Ollama、OpenRouter、LiteLLM、HuggingFace、Snowflake、xAI、Tetrate、NanoGPT、Kimi、GitHub Copilot、DeepSeek、Groq、Mistral、Cerebras、NVIDIA、Zhipu/Z.ai、MiniMax、Moonshot、Alibaba、Perplexity、LM Studio、Venice 等；**本地推理**（llama.cpp，CUDA/Vulkan，feature `local-inference`）；`toolshim` 给不支持原生工具调用的小模型用；**ACP providers**（`claude_acp`、`codex_acp`、`copilot_acp`、`amp_acp`、`pi_acp`）——把 Claude Code / Codex 整个 harness 当 provider 用，goose 的扩展以 MCP server 形式透传，且能复用订阅额度。

**Cowork**：仅 Claude 家族；企业可自带兼容端点（Bedrock/Vertex/Azure）。

评分：goose 5 / Cowork 1.5

### 2.13 可观测性与可审计

goose：Langfuse 层、OTel OTLP 导出、请求日志、会话导出/导入、诊断 zip、PostHog 遥测默认关闭（`GOOSE_TELEMETRY_ENABLED=false`）。Cowork：会话记录、企业审计日志；内部推理与工具轨迹不可导出到自有平台。

评分：goose 4.5 / Cowork 2.5

### 2.14 生态与可扩展性

goose：Skills（agentskills.io 规范，扫描 `~/.agents/skills`、`.agents/skills`、插件目录，并向后兼容 `.claude/skills`）、Plugins（open-plugins / Gemini 格式）、Hooks、Recipes、MCP Apps、自定义发行版（`CUSTOM_DISTROS.md`：预置 provider/扩展/品牌）、Apache-2.0、AAIF 治理。
Cowork：插件市场（官方 + 组织私有）、Skills、Connectors 目录、Claude Code 生态共享。

评分：goose 4 / Cowork 4

---

## 3. 评分矩阵（同模型前提）

| 维度 | goose | Cowork | 决定性因素 |
|---|:---:|:---:|---|
| Agent 循环 / 推理 harness | 3.5 | 4.5 | Cowork = Claude Code harness，为 Claude 深度调优 |
| 本地文件 / 代码操作 | 4.5 | 4 | goose 无边界 shell + 精确 edit + 大输出落盘 |
| Office 文档产出 | 3 | 5 | Docs/Slides/Design、Office 加载项、文档 Skills |
| Computer use / 浏览器 | 2.5 | 4.5 | goose 仅 mac(Peekaboo) 像样，无内置浏览器 |
| 分析与产出形态 | 3.5 | 4 | goose 偏代码/数据/HTML 应用，Cowork 偏文档 |
| 运行效率 / 成本可控 | 4 | 3 | caching、Code Mode、按会话计费 vs 订阅限额 |
| 记忆 | 3 | 3.5 | 显式可审计 vs 自动但仅云端 |
| 上下文管理 | 4 | 4 | 机制等价 |
| MCP / API 接入 | 5 | 3.5 | 全规格宿主 + 可被嵌入 vs connector 单向 |
| 多 Agent / 调度 | 4 | 4 | recipes/hook/cron vs 云端定时任务 |
| 安全与治理 | 4 | 4 | 软防线多 vs 硬隔离但闭源 |
| 模型选择 | 5 | 1.5 | 60+ provider + 本地模型 vs 仅 Claude |
| 平台入口 | 4.5 | 4.5 | 开发者面 vs 知识工作者面 |
| 可观测 / 可审计 | 4.5 | 2.5 | 开源 + Langfuse/OTel |
| 可嵌入 / 二次开发 | 5 | 1 | HTTP/ACP/SDK/自定义发行版 vs 无 API |
| 生态 | 4 | 4 | 双方 Skills/MCP 互通 |

---

## 4. 对 Pions（物理 + AI，钻井场景）的落地建议

### 4.1 按用途选型

| 用途 | 推荐 | 理由 |
|---|---|---|
| 把 agent 嵌进 Pions 产品（钻井仿真/实时数据/优化建议），交付给作业者，可能私有化或离线 | **goose 内核**（`goosed` API 或 ACP），或 Claude Agent SDK（若锁定 Claude） | Cowork 无 API、不可白标、不可离线；goose 可自定义发行版、接本地模型、接 MCP 化的仿真器 |
| 研发/编码 | Claude Code（不是 Cowork）；或 goose + `claude_acp` provider | 得到 Claude Code harness，同时保留 goose 的 recipes/hooks/多 provider |
| 销售/市场/法务/管理的文档工作（方案书、PPT、合同审阅、周报） | Cowork（现 Claude 统一体验） | Docs/Slides/Office 加载项、38+ connectors、零配置 |
| 无人值守的定时数据处理（例如每日钻井日报汇总） | 若数据在云 SaaS：Cowork 云端 scheduled task；若数据在内网/边缘：goose 调度 + recipe + `retry/checks` | Cowork 云任务碰不到内网；goose 调度需进程常驻（可用 systemd/launchd） |
| 需要审计模型每一步（合规、事故复盘） | goose | Langfuse/OTel/会话导出 |

### 4.2 值得一次投入、两边通吃的公共层

1. **MCP server**：把钻井仿真器、实时数据（WITSML/OPC）、地质模型封装成 MCP（Streamable HTTP + OAuth 最通用）。goose 直接接；Cowork 以"自定义远程 connector"接（需公网可达，或桌面端 stdio 本地 MCP）。
2. **SKILL.md**：钻井工程 SOP、压力窗口计算流程、报告模板等写成 agentskills.io 规范的技能；goose 读 `.agents/skills` 与 `.claude/skills`，Cowork/Claude 读 Skills——一份维护。
3. **AGENTS.md / 项目指令**：goose 每次会话自动加载；Claude Code 侧对应 CLAUDE.md。

### 4.3 需要提前认清的坑

- goose：harness 调优是自己的活（可从改 `prompts/*.md` 和 developer 指令开始）；Linux 上无沙箱且 computer use 弱；调度依赖进程常驻；桌面沙箱仅 macOS。
- Cowork：数据默认进 Anthropic 云（含 connector 调用），钻井作业者的数据主权条款要先过；订阅限额在长任务上会成为瓶颈；产品形态正在变（9 月 16 日合并），API/行为不保证稳定；无法把模型换成成本更低或私有部署的开源模型。

### 4.4 一个可行的混合架构

```
Pions 平台(内网/边缘)                       办公侧
┌────────────────────────────┐            ┌──────────────────────┐
│ goosed (自定义发行版)       │            │ Claude(原 Cowork)    │
│  ├ provider: 本地/私有模型  │            │  connectors: 邮件/   │
│  │   或 Claude via Bedrock  │            │  Slack/CRM/Drive     │
│  ├ MCP: 仿真器/实时数据/    │◄──MCP──────│  自定义 connector →  │
│  │   地质模型/报告生成       │  (公网/   │  Pions 数据(只读)    │
│  ├ recipes + cron: 日报/预警 │   零信任)  │  Skills: 同一套      │
│  └ hooks: 审计/合规拦截      │            │  SKILL.md            │
└────────────────────────────┘            └──────────────────────┘
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
| Provider 注册 / Anthropic 缓存 | `crates/goose/src/providers/init.rs`, `providers/formats/anthropic.rs`, `providers/anthropic.rs`, `providers/declarative/*.json` |
| 子代理 / 编排 / 调度 | `agents/platform_extensions/{summon,orchestrator}.rs`, `agents/subagent_*.rs`, `scheduler.rs`, `agents/schedule_tool.rs` |
| Recipes / 重试 / 结构化输出 | `crates/goose/src/recipe/`, `agents/retry.rs`, `agents/final_output_tool.rs` |
| Skills / Plugins / Hooks / Review checks | `crates/goose/src/skills/`, `plugins/`, `hooks/mod.rs`, `checks/mod.rs` |
| 会话存储 / 导入 | `crates/goose/src/session/session_manager.rs`, `session/import_formats/` |
| 服务端 / ACP / SDK / 网关 | `crates/goose-server/src/{auth,session_event_bus}.rs`, `crates/goose/src/acp/`, `crates/goose-sdk*`, `crates/goose/src/gateway/` |
| Code Mode / Apps / MCP-UI | `agents/platform_extensions/code_execution.rs`, `goose_apps/`, `documentation/docs/guides/interactive-chat/mcp-ui.md` |
| 可观测性 | `crates/goose/src/tracing/`, `otel/`, `posthog.rs` |

### 5.2 Cowork 时间线（2026）

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

### 5.3 资料来源（Cowork 侧）

- VentureBeat：Anthropic launches Cowork（2026-01）；Anthropic is killing off Cowork and folding it into Claude chat, launching Claude Docs and Claude Slides（2026-09-16）
- TechRepublic / The Next Web / Engadget / BusinessToday：Cowork 并入 Claude 与 Docs/Slides 报道（2026-09-17）
- Claude Help Center：Get started with Claude Cowork；Use Claude Cowork on web, desktop, and mobile；Schedule recurring tasks；Let Claude use your computer in Cowork；Claude Cowork architecture overview；Use plugins in Claude；Get started with custom connectors using remote MCP；Use Claude Cowork on Team and Enterprise plans
- claude.com/docs/cowork/changelog；claude.com/blog/cowork-plugins；claude.com/connectors
- CSA Research Note / The Hacker News / 9to5Mac / AppleInsider / SOCRadar：SharedRoot sandbox escape（2026-07-27）
- Engadget / TechCrunch / The New Stack：Claude memory now works across chats and Cowork（2026-08-25）
- Forbes / DataCamp：Claude Dispatch（2026-03）
- VentureBeat / The Decoder / The New Stack：Claude for Excel/PowerPoint/Word/Outlook（2026-02 ~ 05）
- Simon Willison：First impressions of Claude Cowork（2026-01）
- 第三方对比（谨慎引用）：lowcode.agency、morphllm、theaiagentindex 的 goose vs Claude Code 文章（SWE-bench 数字非严格同条件）
- goose：github.com/aaif-goose/goose v1.37.0 发布说明；aaif.io/projects/goose；goose-docs.ai
