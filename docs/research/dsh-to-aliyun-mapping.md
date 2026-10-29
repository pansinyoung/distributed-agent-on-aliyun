# DSH 功能 → 阿里云服务 逐项映射表

> 编写时间：2026-10-28
> 目标：在阿里云上**复刻 DSH 的功能清单**（不含 Cordis 装配体系），每一处技术选择都给出依据、限制与验收标准。
> 仓库基线：本地 `commit 141eb6f`（`dsh-v0.1.0-rc.8`）
>
> **证据分级**：【A】阿里云官方文档直读或官方页面；【B】官方技术长文/开发者社区；【C】本表推断，需你实测验证
> **标记约定**：✅ 官方原生支持 ｜ ⚠️ 有条件支持/有坑 ｜ ❌ 不支持，需替代方案

---

## 0. 怎么用这张表

这张表是**施工图**，不是概览。读法：

1. **§1 是前置决策**——14 个选择必须先定，它们互相耦合（比如选了 SSE 就决定了函数类型必须是 Web 函数）。
2. **§2–§8 是逐项映射**——每个 DSH 原件对应「阿里云承接物 / 你要写的代码 / 关键限制」三列。**"你要写什么"这一列是工期来源。**
3. **§9 是放弃清单**——有四块功能逻辑上无法复刻，必须划掉，否则你会一直卡在里面。
4. **§10 是会咬人的数字**——所有硬限制集中在一处，设计时对着看，别等上线才发现。
5. **§11 是分阶段实施**——每阶段带验收标准，能跑通才进下一阶段。
6. **§12 是我没核实的地方**——诚实标注，你必须自己验证。

**一条贯穿全表的原则**：DSH 的 226 个包里有四类东西——① 能力（工具、执行、存储）② 契约（事件模型、管线语义）③ 装配（Cordis）④ 自指（改自己）。**阿里云能承接 ①的一部分，你抄 ②，放弃 ③④。** 这张表只覆盖 ① 和 ②。

---

## 1. 前置决策：14 个必须先定死的技术选择

这些决策互相耦合，顺序不能乱。**先定 1–3，再定 4–7，最后定 8–14。**

### 1.1 语言与部署形态

| # | 决策 | 选项 | **推荐** | 依据 / 理由 |
|---|---|---|---|---|
| 1 | **实现语言** | Python / Node.js / Java | **Python** | 阿里云 SDK 最全（E2B、AgentRun、FNF、OTS 全有 Python SDK）；百炼 MCP/Sandbox 示例均 Python 优先。若你坚持复用 DSH 的 TS 代码则应改用情况 B（部署原版） |
| 2 | **部署形态** | 代码包 / 容器镜像 | **容器镜像** | 容器镜像模式**只需填启动端口**，启动命令由 Dockerfile 的 `CMD`/`ENTRYPOINT` 决定【A】；代码包模式还要额外填启动命令。且 **WebSocket 仅 Custom Runtime / Custom Container 支持**【A】，代码包可能受限 |
| 3 | **函数类型** | Web 函数 / 事件函数 / 任务函数 | **Web 函数** | 🔴 **SSE 流式响应只有 Web 函数支持**；事件函数和任务函数都不支持【A】。FC 靠响应头带 `Transfer-Encoding: chunked` 判定为流式 |

### 1.2 协议与会话

| # | 决策 | 选项 | **推荐** | 依据 / 理由 |
|---|---|---|---|---|
| 4 | **对外协议** | SSE / WebSocket / 两者 | **SSE 优先**，WS 作为可选增强 | AgentRun 层已抹平网络差异，提供 HTTP / WebSocket / SSE 三种【B】。但 WS 限制更多：单包 ≤6MB、握手不支持 Body、受函数超时限制、超时后强制关闭返回 1006【A】。SSE 天然适合 token 流，且 DSH 本身是 `assistant/chunk` token 级事件 |
| 5 | **会话亲和方式** | HeaderField / Cookie / MCP SSE / MCP Streamable HTTP | **HeaderField（自定义 header）** | 三种亲和方式：`affinityHeaderFieldName`（自定义 header）、Cookie 亲和（系统生成 `x-fc-cookie-session-id`）、MCP SSE / Streamable HTTP 亲和【A/B】。选 HeaderField 因为 header 名可控、不占 Cookie、与 AgentRun 的 `X-AgentRun-Session-ID` 语义一致 |
| 6 | **亲和 header 命名** | — | **`dsh-session-id`** | 硬约束：**不能以 `x-fc-` 前缀开头**，必须字母开头，长度 5–40 字符，非首字符可含数字/中划线/下划线/字母【A】 |
| 7 | **单实例会话并发** | 1 / 20（默认）/ 其他 | **1（启用会话物理隔离）** | 🔴 这是 DSH 单写者语义的关键。启用**会话物理隔离**后，FC 会**强制将单实例 Session 并发度设为 1**，每个会话独占一个完整实例，内存/临时文件/进程空间与其他租户物理分离（基于神龙裸金属）【B】。参数 `sessionConcurrencyPerInstance` 范围 [1,200]，默认 20【A】 |
| 8 | **Session TTL** | 1–21600 秒 | **21600（6 小时，上限）** | 🔴 **这是硬上限**：`sessionTTLInSeconds` 取值范围 [1, 21600]。超过后 FC 自动销毁 Session，**不再保证亲和性**【A】。这意味着**你的任何会话都不能假设活过 6 小时** |
| 9 | **Session 空闲超时** | 0–21600 秒，默认 1800 | **1800（30 分钟）起，按业务调** | `sessionIdleTimeoutInSeconds` 范围 [0, 21600]，默认 1800【A】。设太短 → 频繁冷启动；设太长 → 空转计费 |
| 10 | **SessionID 过期后能否复用** | true / false | **false（默认）** | `disableSessionIdReuse` 默认 `false` = 允许会话过期后携带相同 SessionID 继续请求，系统视为新会话绑定新实例【A】。若设为 `true` 则不可复用（会报错）。**建议保持 false**，靠你自己的逻辑判断「是新会话还是恢复」 |

### 1.3 存储与执行

| # | 决策 | 选项 | **推荐** | 依据 / 理由 |
|---|---|---|---|---|
| 11 | **事件日志主存储** | RDS PostgreSQL / Tablestore / Lindorm | **RDS PostgreSQL 起步**，量大再迁 Tablestore | **Tablestore**：全托管 serverless 宽表，PK 驱动，支持 TTL + max versions + 本地事务（限分区键内），适合「大规模、PK 为中心、事件/时序」场景，且有 Python SDK【B】。**RDS PG**：有 SQL/join/事务，但海量事件写入扩展难、成本高【B】。**决策**：MVP 用 PG（开发效率优先、`sessionQuery` 直接写 SQL）；单表过千万行再迁 Tablestore（`PK = (session_id, seq)` + TTL） |
| 12 | **工作区持久化** | NAS / OSS / 沙箱内临时 | **NAS（挂 `/mnt/nas`）为主 + OSS 为归档** | NAS 支持 POSIX 语义，可像本地文件一样读写【A】。**硬约束**：单函数同地域**最多 5 个 NAS 挂载点 + 5 个 OSS 挂载点**，挂载的本地目录不能冲突；**只支持 NFS 协议，不支持 SMB**；本地目录建议用 `/home` `/mnt` `/tmp` `/data` 的子目录，**不要用 `/bin` `/opt` `/var` `/dev`**【A】 |
| 13 | **执行世界** | Agent Sandbox（E2B）/ 本机 / 自建 | **Agent Sandbox（E2B 兼容）** | DSH 已有 `e2b` / `fs-e2b` / `subprocess-e2b` 三个 provider，阿里云 Agent Sandbox **兼容 E2B SDK/API**，只换 endpoint + key。覆盖 `files.*` / `commands.*` / `pty` / `run_code`【A】 |
| 14 | **观测接入** | AgentLoop / ARMS 探针 / SLS / OTel 直连 | **ARMS 探针 + AgentLoop** | AgentLoop 三种接入：自研探针（零侵入）、**OpenTelemetry 标准协议**、eBPF【A/B】。**AgentRun 默认通过 CloudMonitor 2.0 接入可观测，无需装探针**【A】。ARMS 探针装法：`pip install aliyun-bootstrap && aliyun-bootstrap -a install`，环境变量 `ARMS_APP_NAME` / `ARMS_REGION_ID` / `ARMS_LICENSE_KEY` / `ARMS_WORKSPACE`，启动 `aliyun-instrument python app.py`【A】 |

### 1.4 长任务与模型

| # | 决策 | 选项 | **推荐** | 依据 / 理由 |
|---|---|---|---|---|
| 15 | **人工审批 / 长任务等待** | Serverless 工作流 FNF `waitForCallback` / 自写 pending 表轮询 / AgentCore 断点恢复 | **FNF `waitForCallback`** | 官方专门覆盖这个场景：Task 步骤设 `pattern: waitForCallback` 后提交资源并**暂停**，`taskToken` 存入 `$context.task.token` 经 `inputMappings` 传给函数，用 `ReportTaskSucceeded` / `ReportTaskFailed` + token 恢复【A】。官方原文列的适用场景包含「**需要人工干预的步骤，例如通知审批通过**」【A】 |
| 16 | **模型接入** | 百炼 OpenAI 兼容 / 直连 DeepSeek / 混合 | **百炼 OpenAI 兼容为主** | `base_url = https://dashscope.aliyuncs.com/compatible-mode/v1`（或按业务空间 `https://{WorkspaceId}.cn-beijing.maas.aliyuncs.com/compatible-mode/v1`）【A】。⚠️ **各地域 base_url 不通用**【A】，必须配置化 |
| 17 | **联网与代码工具** | 自己实现 / 百炼内置工具 | **百炼 Responses API 内置工具优先** | 🔴 重要发现：百炼 Responses API 原生支持 `web_search`、`web_extractor`、`code_interpreter` 三个工具【A】，直接对应 DSH 的 `web_search` / `web_fetch` / 代码执行 |
| 18 | **AgentCore 协作面** | 开 / 不开 | **不开** | ¥3/小时（实例/小时），常开约 **¥2,160/月**，是算力（约 ¥360/月）的 6 倍，且**不缩容到 0**。除非你要 AgentCore 的 Team 能力，否则不要创建 |

> **⚠️ 决策 7 与决策 5 的一个陷阱**：开启 **Cookie 亲和**后，系统会**自动将单实例并发度调整为 200（系统默认值，不可手动调整）**【A】。而 **Session 隔离**走的是另一条路径（强制 1）。**如果你要单写者保证，必须走「会话物理隔离」，不要走 Cookie 亲和。**

---

## 2. 执行世界映射（DSH 的 `fs` / `shell` / `subprocess` / `terminal` / `code-runtime` / `lsp` / `sandbox`）

这是**最容易复刻的一块**，因为阿里云 Agent Sandbox 的 E2B 兼容矩阵几乎正好覆盖 DSH 的执行层。**DSH 自己的 `e2b` provider 就是照着这个写的，所以你等于在用同一套协议。**

### 2.1 E2B 兼容矩阵 → DSH 能力对照

| DSH 原件 | 包路径 | Agent Sandbox 承接 | 兼容性 | 你要写的代码 |
|---|---|---|---|---|
| `fs` 工具组（读/写/编辑/列目录/glob/grep） | `packages/fs/*` | `sandbox.files.read()` / `write()` / `list()` / `exists()` / `getInfo()` / `makeDir()` / `remove()` / `rename()` / `watchDir()` | ⚠️ **部分兼容** | 5 个工具函数 + **glob/grep 无原生实现**，走 `commands.run('rg ...')` 或 `find`。`write()` 支持文本/二进制/流式内容【A】 |
| `bash` / `shellEnv` | `packages/shell/*` | `sandbox.commands.run()` | ✅ 兼容 | 环境变量注入 + cwd 管理 + 输出截断。⚠️ **必须设命令级超时**，官方建议处理 stdout/stderr 流而不是只等最终结果【A】 |
| `subprocess`（进程树） | `packages/subprocess/*` | `commands.run()` / `connect()` / `sendStdin()` / `kill()` / `list()` | ✅ 兼容 | 后台进程句柄表 + 生命周期清理 |
| `terminal`（持久 PTY 会话） | `packages/terminal/*` | **`sandbox.pty`（独立 API）** | ✅ 兼容 | 官方明确：普通批处理用 `commands.run()`；**只有交互式 CLI、彩色输出、进度条或依赖 TTY 判断的才用 `sandbox.pty`**【A】。照这个判据分流 |
| `code-runtime`（PTC 代码模式） | `packages/*/code-runtime` | `sandbox.run_code()` + `createCodeContext()` / `listCodeContexts()` / `restartCodeContext()` / `removeCodeContext()` | ✅ 兼容 | 🔴 **语言只支持 Python / JavaScript / TypeScript / Bash；Java 和 R 不支持**【A】。✅ **变量与上下文状态跨多次执行保留**【A]——这正好对应 DSH 的 codeRuntime 语义 |
| `lsp` | `packages/lsp/*` | 沙箱内装 LSP server + `commands.run` 拉起，走 JSON-RPC over stdio | ⚠️ 需自建 | **完整的 LSP client 要自己写**（initialize / textDocument/* / diagnostics 订阅）。**建议放到 V2**，MVP 先不做 |
| `sandbox` / `sandboxPolicy` | `packages/*/sandbox*` | **Agent Sandbox Template**（镜像 + 资源规格 + 文件挂载 + 网络策略 + 环境变量 + 生命周期） | ⚠️ 有硬限制 | 策略 → Template 配置映射。🔴 **`Network Config Update` 受限**（API 可调但返回结果/实际效果有限）→ **网络策略必须在建 Template 时定死，不能运行时改** |
| `e2b` / `fs-e2b` / `subprocess-e2b` | `packages/e2b/*` | **Agent Sandbox 原生 E2B** | ✅ 兼容 | **只改 endpoint + API key，不改代码** |

### 2.2 Agent Sandbox 的硬限制（必须设计进去）

| 限制 | 官方表述 | 对你的影响 |
|---|---|---|
| **暂停/恢复需白名单** | *"Sandbox Pause and resume is currently available only to allowlisted accounts."*【A】 | 🔴 你要靠「暂停保状态」省钱，**必须先申请白名单**。否则只能靠 idle timeout 自动回收，状态丢失 |
| **Volume 不兼容** | *"Snapshots / Volume / Team / Access Token — Not currently compatible"*【A】 | workspace 持久化**不能靠挂卷**，必须走 NAS 或沙箱内 `files.*` 同步 |
| **无自定义文件 metadata** | 传入 `metadata` 会抛 `TemplateException: File metadata requires envd 0.6.2 or later`【A】 | 不要依赖文件属性存业务元数据，另建表 |
| **图表只能 file-based** | *"FC Agent Sandbox currently supports the file-based fallback path only: generate image files inside the sandbox and retrieve them through Filesystem or download URLs."*【A】 | 图片/图表要在沙箱内落文件再取回，不能直接从返回体拿 |
| **沙箱最长存活 24 小时** | `SessionIdleTimeout` 后自动销毁；沙箱最长可存活 24h【B】 | 超长任务必须分段 + 外部 checkpoint |
| **Logs / Metrics 受限** | Metrics 支持 CPU/内存，但 **Disk 和 page-cache 返回占位值不可用**；Logs 受限【A】 | 别指望从沙箱拉日志做审计，自己记 |
| **不要拼 shell** | 官方明确：*"不要把未经校验的用户输入直接拼接进 shell 命令"*【A】 | 参数校验层必须做，这是安全底线 |

### 2.3 执行世界的一个额外选择：百炼内置 `code_interpreter`

有两条路跑代码，**要选一条，不要两条都用**：

| 路径 | 优点 | 缺点 | 何时选 |
|---|---|---|---|
| **Agent Sandbox** `run_code` | 可控（自定义镜像、NAS 挂载、网络策略） | 你要管沙箱生命周期 | 需要文件系统、需要长会话、需要自定义依赖 |
| **百炼 Responses API** `code_interpreter` 工具 | 零运维，模型直接调 | 你不能控制环境，拿不到文件系统 | 纯计算、数据分析，不需要工作区 |

---

## 3. 会话数据平面映射（DSH 最核心、也最必须自己写的一块）

**这一块是所有映射里「你要写的代码最多」的部分。** 因为阿里云没有「Agent 会话事件日志」这个产品——它就是一张表 + 一个 fold 函数。

### 3.1 逐项映射

| DSH 原件 | 阿里云承接 | 你要写什么 | 关键限制 / 设计要点 |
|---|---|---|---|
| `SessionEvent` append-only 日志 | **RDS PostgreSQL** 单表 | 完整 schema：`(session_id, seq, type, payload jsonb, created_at)`；`PRIMARY KEY (session_id, seq)`；按 `created_at` 分区 | 🔴 **这是全系统的权威数据源**。DSH 的原则是「Model-visible ⟺ logged」——**模型看到的必须都被记录**，包括 system prompt、工具 schema、模型参数 |
| `deriveMessages()` fold 投影 | 无 | **约 100–150 行纯函数**：按 seq 顺序 fold 事件流 → messages 列表 | 🔴 必须保留 DSH 的语义：**只有 3 种事件进模型可见面**（`user/message`、`assistant/message`、`tool/result`），其余事件只影响系统状态不进 context。压缩走 `surfaceOp: replace` 遮蔽而非删除 |
| `sessionPersistence` flush checkpoint | 无 | 写入策略：逐事件同步写 or 批量（如每 step 结束批量提交） | ⚠️ **配合 FC 的 `prestop` 钩子做最终 flush**——官方明确 FC「当实例要销毁前会调用 prestop 钩子方法，用户可以在这个方法中做数据善后处理」【B】。这是你唯一能拿到「优雅关闭」的地方 |
| `projection` + cache | Tair（Redis 兼容） | 缓存层：把最近 N 个事件 fold 结果缓存 | 可选。**MVP 直接每次重 fold**，量小的时候完全够 |
| `sessionTitle` | 百炼一次模型调用 | 提示词 + 异步更新 | 别阻塞主链路，异步做 |
| `sessionTelemetry` | **AgentLoop / ARMS / SLS** | 埋点（OTel span 或 ARMS 探针） | AgentRun 默认接 CloudMonitor 2.0，无需装探针【A】 |
| `messageFeedback` | RDS 一张表 | 简单增删 | — |
| `workspace` | **NAS 挂 `/mnt/nas`** | 目录约定：`/mnt/nas/{session_id}/` | ⚠️ NAS 只支持 NFS；挂载会让冷启动 **+2–5 秒**【B]。若要跨函数共享，**必须配置相同的用户/用户组**【A】 |
| `storage` / `storageDomain` | **OSS** | provider 适配（put/get/delete/sign_url） | ⚠️ **OSS 挂载写入延迟远高于本地 SSD，小文件写吞吐可能只有本地 1/10**【B】。策略：**先写 `/tmp` 再批量上传**，不要直接往挂载点写小文件 |
| `attachments` | OSS | 上传 + 引用 | — |
| 会话恢复（resume） | 无 | **恢复逻辑必须自己写** | 🔴 官方给的最佳实践：**在请求进入 FC 之前需要一个管控服务**，它按 Session ID 查 OSS/数据表，有数据就走恢复（下载 tar 解压恢复目录），查不到就当新会话【B】。**这就是你 DSH `resumeSessionId` 的实现位置** |

### 3.2 事件表 schema 建议（含 DSH 关键字段）

DSH 有一个容易被忽略但极重要的设计：`request/header` 事件保存**每一次真实发给模型的请求的完整 Header 快照**——使用的模型与参数、渲染后的 System Prompt、本轮看到的工具 Schema。目的是「可以重建每一次真实发给模型的请求」。

**这个字段必须复刻**，否则：

- 出问题时无法解释模型当时看到了什么
- resume 依赖「当前进程碰巧加载了什么」
- Prompt Cache 前缀不稳定
- 工具无法审计「谁改了模型输入」

建议 schema 至少包含事件类型：

```
turn/start         turn/end
step/start         step/end
user/message
request/header     ← 完整请求快照（模型、参数、system prompt 渲染结果、tool schema）
assistant/chunk    ← token 级
assistant/message
tool/call          tool/result
compaction/summary ← 带 surfaceOp: replace + 被遮蔽的事件范围
```

🔴 **`assistant/chunk` 是 token 级的**——这是 DSH 对分布式最友好的设计（前面调研已确认）。但注意：**它也是你表里最大的数据量来源**，必须做 TTL 或冷热分离（Tablestore 的 TTL / PG 的分区 drop）。

### 3.3 存储选型的量化依据

| 方案 | 写入 | 查询 | 成本 | 何时用 |
|---|---|---|---|---|
| **RDS PostgreSQL** | 单表过千万行后写入变慢 | SQL 任意查，`sessionQuery` 直接写 | 中高（约为 NoSQL 的 5–10 倍存储成本） | **MVP 首选**（0–1000 万事件） |
| **Tablestore** | 海量 PK 写入快，支持 TTL + max versions | 无 join，二级索引有限 | 低，pay-as-you-go 但**索引存储和写放大会显著推高成本**【B】 | 事件量级上来后迁移。`PK = (session_id, seq)` |
| **Lindorm** | 宽表引擎，百万级/秒写入，冷热分层 | 宽表 SQL（**不支持事务**）+ 内置全文检索 | 冷热分层后最低（官方称降 60%） | 需要全文检索 + 超大规模 |
| **Tair（Redis）** | 微秒级 | KV | 高（内存） | 只做缓存，不做 system of record |

> **一份参考架构**（Tablestore 官方给的零售事件平台范例，结构与你的需求高度相似）：
> API 服务写入 Tablestore，`PK = (tenant_id, user_id, reverse_ts)`，**TTL = 90 天，max versions = 1**，导出管线到 OSS/MaxCompute 做长期分析，SLS 集中日志，CloudMonitor 做指标【B】。
> **这套正好可以照搬**：PK 换成 `(session_id, reverse_seq)`，TTL 按你的合规要求定。

---

## 4. 上下文治理映射

### 4.1 逐项映射

| DSH 原件 | 阿里云承接 | 你要写什么 | 关键限制 / 决策 |
|---|---|---|---|
| `compaction`（两级压缩） | **没有一个能直接接的** | 🔴 **完整自写**：① 微压缩（把旧的 tool result 换成摘要，保留最近 N 个完整）② 全量摘要（追加 `compaction/summary` 事件 + `surfaceOp: replace` 遮蔽旧范围） | 🔴 **不要把单会话窗口管理委托给 Agent Context / Context Engine**。它们管的是**跨会话长期记忆**，你管的是**单会话窗口**。两层都做压缩会互相打架（平台压完你再压，信息二次损失）。这个边界必须划清 |
| `tokenMeter` | 百炼 `response.usage`（含 `input_tokens` / `output_tokens` / `cached_tokens` / `reasoning_tokens`）【A】 | 计数器 + 阈值触发 + 落到事件表 | ⚠️ **不同模型的 tokenizer 不同，同一段文本 token 数会有差异**【A】。不要硬编码换算比例，一律以 `usage` 返回值为准 |
| `spill`（大输出溢出到文件） | OSS + NAS | 阈值策略：单 tool result > N KB → 写文件，回填引用摘要 | 必须做阈值，否则大输出会把 context 和事件表同时撑爆 |
| `skill`（provider registry + catalog 工具 + 按需注入） | **无对应产品** | 🔴 **完整自写**：markdown 加载 + 目录索引 + `skill` 工具 + 按需注入 | ⚠️ 阿里云有「技能门户」（60+ 云产品转 MCP skills），但那是**云产品能力的 MCP 封装**，不是 DSH 那种「本地 markdown 知识注入」，两者不是一回事 |
| `mcp-client` | **AgentRun 可作 MCP 网关**（支持 MCP SSE / MCP Streamable HTTP 会话亲和）【B】 | MCP client（官方 Python SDK）+ 工具注册到你的 tool registry | ✅ 双向可用：① 你作为 client 连别人的 MCP server ② AgentRun 作网关对外暴露你的工具 |
| `sessionQuery` | **AgentLoop 的 Dataset**（带 CRUD + 灵活 schema + 向量检索 + 多维分析）【A】 或 RDS SQL | 查询接口 | MVP 直接 RDS SQL；需要语义检索时再接 AgentLoop Dataset |
| `context` 插件族（请求上下文注入） | 无 | 自写：在 fold 出的 messages 里按规则插入运行时上下文 | — |
| 长期记忆 | **AgentLoop 记忆库 / 经验库**（记忆库已可用，经验库内测中）【A】 或百炼知识库 / Lindorm 向量 | 接入 | 这是唯一**建议委托给平台**的一块，别自己写向量库 |

### 4.2 压缩策略的一个具体建议

DSH 的设计是「压缩不修改历史」——追加一条摘要事件，用 `surfaceOp: replace` 让摘要**在模型视图中遮蔽**旧消息，但**旧事件不删除**。这样系统同时保留两种答案：事件日志回答「真实发生过什么」，当前 Surface 回答「模型接下来需要看到什么」。

**这个设计必须保留。** 不要图省事直接 DELETE 旧消息——那样审计历史就没了，而且 resume 会丢信息。

---

## 5. 模型与工具映射

### 5.1 模型接入

| DSH 原件 | 阿里云承接 | 你要写什么 | 关键限制 |
|---|---|---|---|
| `llm` 适配器 seam（Service Definition + Consumer + DeepSeek provider） | **百炼 OpenAI 兼容** | 薄适配器：把 DSH 的 `Message` / streaming 词汇表映射到 OpenAI 格式 | ⚠️ **各地域 base_url 不通用**【A】。北京 `https://{WorkspaceId}.cn-beijing.maas.aliyuncs.com/compatible-mode/v1`；新加坡 `...ap-southeast-1...`；德国 `...eu-central-1...`；日本 `...ap-northeast-1...`；美国 `https://dashscope-us.aliyuncs.com/compatible-mode/v1`。**必须全部配置化** |
| 模型选择 | 百炼模型列表 | 配置文件 | qwen-max / qwen-plus / qwen-turbo / qwen-long / qwen-vl-max / qwen-vl-plus / qwen-vl-ocr / deepseek-r1 / deepseek-v3 / qwen3.8-max / qwen3.5-plus（1M 上下文）【A】 |
| 推理模式（thinking） | `enable_thinking` | 参数映射 | 百炼原生支持【A】 |
| 联网 | `enable_search` | 参数映射 | 百炼原生支持【A】 |

### 5.2 工具注册（DSH 的 `defineTool` 契约）

DSH 的 `defineTool` 有六个字段，每一个都是「模型接口契约」，**建议原样照抄**：

| 字段 | 作用 | 你的实现要点 |
|---|---|---|
| `name` | 模型可见的工具名 | ≤64 字符，只允许 `[A-Za-z0-9_-]` |
| `description` | 模型用来判断是否调用的描述 | 这是准确率的关键，别敷衍 |
| `parameters` | JSON Schema | **自动推导 args 类型 + 运行时校验**。用 Pydantic / jsonschema |
| `output.schema` | 声明输出 schema | 让模型知道会拿到什么结构 |
| `output.render` | 规范输出 → 模型可见的 ContentBlock | 控制「怎么展示给模型」 |
| `execute` | 实际执行，接收校验后的 args | 这里只写业务，不写校验 |

### 5.3 27 类工具的映射总表

| DSH 工具类别 | 承接方 | 工作量 |
|---|---|---|
| 文件读写编辑 / 目录 / glob / grep | Agent Sandbox `files.*` + `commands.run`（rg/find） | 小 |
| 命令终端 / PTY | Agent Sandbox `commands.run` / `pty` | 小 |
| 代码执行 | Agent Sandbox `run_code` 或百炼 `code_interpreter` | 小 |
| 网络搜索 | **百炼 `web_search`（Responses API）** ✅ | 极小 |
| 网络抓取 | **百炼 `web_extractor`（Responses API）** ✅ | 极小 |
| 任务 / 目标 / 计划（`todo_write` / plan） | 自写：RDS 表 + 状态机 | 中 |
| Agent 协作（subagent 派生） | 见 §6 | 大 |
| 上下文记忆（压缩 / 检索） | 见 §4 | 大 |
| skill（加载 + 目录） | 自写 | 中 |
| MCP 工具调用 | MCP Python SDK | 中 |
| 自我修改（`cordis_*`） | ❌ **放弃**（见 §9） | — |

🔴 **重要**：`web_search` / `web_extractor` / `code_interpreter` 在百炼走的是 **Responses API（`client.responses.create`）**，不是 `chat.completions`，工具声明形如 `tools=[{"type": "web_search"}, {"type": "web_extractor"}]`【A】。**你的适配层要同时支持两套 API 形态**，否则流式和工具调用会打架。

---

## 6. Agent 协作映射

| DSH 原件 | 阿里云承接 | 你要写什么 | 关键限制 |
|---|---|---|---|
| `subagent` 基础派生 | 同进程 `asyncio` 任务 | 子任务生命周期 + 结果回传 | 最容易，先做这个 |
| `subagent` 的 **continuable**（`startContinuable` / `followup` / `interrupt` / `reportFrom`） | 无 | 🔴 **完整自写**：stable child id 预占 + detached `ContinuableCreateSpec` + continuation manager + inbox 状态机（running 入队 / waiting 唤醒 / 无 Activation 冷启动）+ `interrupt` 保留 inbox + 子到父回报凭证 | 这是 DSH 最精巧的设计之一。**MVP 只做 basic spawn，continuable 放 V2** |
| `agent teams`（leader-worker 治理） | **AgentCore Team**（¥3/小时协作面，约 ¥2,160/月）或自写 | 自写更划算 | AgentTeams 已下线并入 AgentCore（2026-09-20 迁移截止） |
| `jobs`（后台任务） | **MNS / RocketMQ** 或 RDS 任务表 + 轮询 | 任务表 + 消费循环 | 用消息队列更省；简单场景 RDS 表就够 |
| `workflow`（worker-thread provider） | **Serverless 工作流 FNF** | 状态机定义（YAML/JSON） | 见 §7 |
| `goal` / `schedule`（定时） | **FNF 定时触发 / EventBridge** | 调度配置 | — |
| `ralph` 类长循环 | 无 | 自写 + FNF 编排 | 注意迭代预算，别无限跑 |

> **判据**：DSH 的子 Agent 有「跨多次对话甚至跨冷启动存在」的语义。**如果你的场景不需要这个，就不要实现 continuable**——它会让工期从 3 天变成 3 周。

---

## 7. 人机协作与长任务映射

### 7.1 核心方案：Serverless 工作流 FNF 的 `waitForCallback`

| DSH 原件 | 阿里云承接 | 兼容性 | 说明 |
|---|---|---|---|
| `approval`（fail-closed 权限确认） | **FNF `pattern: waitForCallback`** | ✅ 官方明确覆盖 | Task 步骤设 `pattern: waitForCallback` 后，提交资源并**进入暂停状态**，`taskToken` 存入 `$context.task.token`，通过 `inputMappings` 传给函数；直到 FNF 收到回调或超时才继续【A】 |
| `userQuestions`（向用户提问） | 同上 | ✅ | 同一个机制，换个语义 |
| `planMode`（log-only，只记录不拦截） | 无 | ✅ 无需承接 | 纯事件记录，最容易实现 |
| `commands`（斜杠命令） | 无 | ⚠️ 自建 | 简单的命令路由表 |

**FNF 回调的完整流程**（官方示例结构）【A】：

```yaml
version: v1
type: flow
steps:
  - type: task
    name: approval-wait
    resourceArn: acs:fc:::services/{service}/functions/{function}
    pattern: waitForCallback          # 提交后暂停，等回调
    inputMappings:
      - target: taskToken
        source: $context.task.token   # 把 token 作为入参传给函数
    outputMappings:
      - target: decision
        source: $local.decision       # 从 ReportTaskSucceeded 的输出映射回来
```

恢复方式：任意有权限的环境调 `ReportTaskSucceeded` / `ReportTaskFailed` + `taskToken`（Python SDK：`aliyun-python-sdk-fnf`）【A】。

**为什么这个机制比自写 pending 表好**：
- 官方原文列的适用场景包含「**需要人工干预的步骤，例如通知审批通过**」【A】
- 消除轮询延迟；消除高并发轮询带来的资源浪费【A】
- 可以把非 FC 任务（自建机房、ECS 进程）一起编排进来【A】

### 7.2 Checkpoint 该放在哪里（阿里云官方的工程建议）

阿里云开发者社区有一篇专门讲 Agent 状态保存的文章，给出了检查点位置的推荐清单，**与 DSH 的 checkpoint 设计高度一致，可直接采用**【B】：

| 该保存检查点的时机 |
|---|
| 一个耗时较长的子任务完成后 |
| 文件、数据库或外部系统发生重要变化后 |
| **准备等待用户确认之前** |
| 即将执行高风险操作之前 |
| 一轮验证通过、可以进入下一阶段时 |

**状态字段的追加/覆盖规则**【B】：

| 类型 | 字段 | 策略 |
|---|---|---|
| 持续追加 | 消息、日志 | append |
| 被新值覆盖 | `current_step`、`status`、`last_error` | overwrite |
| 按集合维护 | 已完成任务 | set/list |
| 按路径为键 | 文件变更 | key = file path |

🔴 **两条必须写进设计的硬约束**：
1. **状态 Schema 必须带版本号 + 迁移规则**。Agent 功能增加后字段会变（旧任务可能没有新字段），明确版本号和迁移规则可以避免系统升级后读不出旧状态【B】。
2. **恢复内部状态 ≠ 回滚外部世界**。模型回复、任务进度、工作文件可恢复，但**已发送的邮件、已完成的付款、已删除的远程数据无法撤销**。涉及不可逆操作时必须配合人工确认、幂等设计或补偿流程【B】。

### 7.3 长任务的组合方案

```
用户请求
  → AgentRun 容器（会话亲和 = 该 session 独占实例）
      → Agent step 循环
          → 遇到需要审批 / 长等待
              → 写 checkpoint 到 RDS + workspace 快照到 NAS/OSS（prestop 兜底）
              → 调 FNF StartExecution（带 waitForCallback 的流程）
              → 立即返回「等待中」给用户
          → FNF 收到回调
              → 重新唤起 AgentRun（同一 session ID）
              → 从 checkpoint 恢复，继续执行
```

🔴 **注意 Session TTL 上限 6 小时**：如果你的审批等待可能超过 6 小时，**FC 会话已经销毁**，恢复必须走「查 RDS/NAS → 重建实例」的路径，而不是指望会话亲和。这就是 §3.1 里「管控服务判断 Session 是否存在」那条官方最佳实践的必要性。

---

## 8. 接入协议映射

| DSH 原件 | 阿里云承接 | 兼容性 | 说明 |
|---|---|---|---|
| `host/webserver`（HTTP + `registerUpgrade` WS） | **AgentRun 容器 + Web 函数** | ⚠️ 有限制 | 见下方细则 |
| `sdk/server`（JSON-RPC over **stdio**） | **不可用** | ❌ | Serverless 没有 stdio 对端。必须改成 HTTP |
| `acp/acp`（JSON-RPC over **stdio**） | **不可用** | ❌ | 同上 |
| Typert RPC gateway（`/api` 路由） | 无 | ⚠️ 自写 | 普通 HTTP API |

### 8.1 SSE（推荐主通道）

| 要点 | 内容 |
|---|---|
| 前提 | 必须是 **Web 函数**；事件函数/任务函数**不支持**流式【A】 |
| 判定方式 | FC 根据响应头是否带 `Transfer-Encoding: chunked` 判断是否流式【A】 |
| 实现 | 你的 HTTP 框架必须真的分块输出（不要缓冲完再发） |
| 适用 | token 级流式、Agent trace 推送 |

### 8.2 WebSocket（可选增强）

| 要点 | 内容 | 影响 |
|---|---|---|
| Runtime 支持 | **仅 Custom Runtime 和 Custom Container** 支持 WS 协议【A】 | 容器镜像模式 ✅ 满足 |
| 触发方式 | GET 方式触发函数，适用长连接、实时消息【A】 | — |
| 超时 | **受函数配置的超时时间限制**；超时后 FC 强制关闭连接，客户端收到 **1006**【A】 | 🔴 长会话要考虑重连 |
| 单包大小 | 建立之后，**一次发送或接收的数据包最大 6 MB**【A】 | 🔴 大 tool result 不能走 WS |
| 握手 Request | **不支持以 `x-fc-` 开头的自定义字段**；Headers 总和 ≤8KB；Path（含 Query）≤8KB；**不支持 Body**（发了也被忽略）【A】 | 🔴 认证 token 不能放握手 Body，要放 header 或 path |
| 握手 Response | Headers 总和 ≤8KB【A】 | — |
| 无状态警告 | *"For functions with concurrency greater than 1, there is no guarantee that multiple connections from the same client go to the same instance."*【A】 | 🔴 **必须启用会话亲和 + 会话隔离=1**，否则同一客户端多连接会散到不同实例 |

### 8.3 一个务实的建议

**MVP 只做 SSE + 普通 HTTP POST。** WebSocket 的收益在 Serverless 上被限制得很厉害（6MB / 超时 / 握手限制），而 DSH 的事件流本来就是 token 级增量，SSE 完美匹配。

需要双向实时（比如用户中途打断）时，用「HTTP POST 发送控制指令 + SSE 接流」的组合，比 WS 简单得多。

---

## 9. 必须放弃的清单（这四块逻辑上无法复刻）

| # | 放弃什么 | DSH 里的位置 | 为什么不能复刻 | 替代方案 |
|---|---|---|---|---|
| 1 | **`cordis_*` 自我修改工具** | `packages/self-modification/`（`cordisInspect` / `dynamicCordisRunner` / `tool-cordis`） | 这是**关于插件体系的元能力**——「让 Agent 检查/挂载/卸载自己的插件」。没有进程内插件运行时，这个功能**没有复刻对象** | ❌ 无替代。这是能力性缺失，接受它 |
| 2 | **插件热插拔 / 热重载** | Cordis Fiber 生命周期 + `ctx.effect` | 依赖「卸载」这个事件，Serverless 只有「进程被杀」 | ✅ **换镜像重启**。FC 有优雅升级：`UpdateFunction` 后，**存量 Session 关联的请求仍路由到旧实例，新建 Session 路由到新实例**【B】——这已经够用 |
| 3 | **`profile` / `bundle` 四层 patch 叠加 + `cordis.yml` 组合语义** | `packages/bundle/*`、`packages/preset/*` | 依赖 cordis.yml 的覆盖/insert/disabled 语义 | ✅ **多镜像方案**：每个 preset 打一个镜像，差异用环境变量注入。牺牲灵活性换确定性 |
| 4 | **`agentPresets`（每会话挂载一份 preset）** | `packages/preset/*` | 依赖 `ctx.isolate` realm（进程内 symbol 表，不可序列化） | ✅ 环境变量 + 配置表 + 按 session 读配置 |
| 5 | **`creator` 运行模式** | 四种模式之一 | 字面意义「在内存里试 Cordis 插件」 | ❌ 无替代 |
| 6 | **ACP / SDK 的 stdio 协议** | `packages/acp/acp`、`packages/sdk/server` | Serverless 无 stdio 对端 | ✅ 改 HTTP JSON-RPC |

### 9.1 关于「四种运行模式」的取舍

| DSH 模式 | 能否复刻 | 说明 |
|---|---|---|
| `standard`（完整工具集） | ✅ 完全可复刻 | 你的默认模式 |
| `PTC`（Programmatic Tool Calling，模型生成代码组合多轮工具调用） | ✅ 可复刻 | 用 Agent Sandbox `run_code` + code context（跨执行保留变量） |
| `minimal`（只有 shell + 文件编辑） | ✅ 完全可复刻 | 最容易，正好用来做冒烟测试 |
| `creator`（检查运行时、试 Cordis 插件） | ❌ 放弃 | 见上表 #5 |

**建议**：先做 `minimal` 作为最小验证闭环，再做 `standard`，`PTC` 放第三，`creator` 不做。

---

## 10. 会咬人的数字（设计时对着看）

### 10.1 会话与协议

| 限制项 | 数字 | 出处 | 后果 |
|---|---|---|---|
| **FC Session TTL** | **最大 21600 秒 = 6 小时** | 【A】`sessionTTLInSeconds` 范围 [1, 21600] | 🔴 任何会话都不能假设活过 6 小时。超时后**不再保证亲和性** |
| Session 空闲超时 | [0, 21600] 秒，默认 **1800** | 【A】 | 30 分钟无请求即销毁，决定冷启动频率 |
| 单实例会话并发 | [1, 200]，默认 **20** | 【A】 | **设 1 才有排他性**；会话隔离会强制 1 |
| 开 Cookie 亲和后单实例并发度 | **强制 200，不可手动调整** | 【A】 | 🔴 与单写者目标冲突，**不要用 Cookie 亲和做单写者** |
| 亲和 header 名 | 不能 `x-fc-` 前缀，字母开头，**5–40 字符** | 【A】 | 建议 `dsh-session-id` |
| WebSocket 单包 | **≤ 6 MB** | 【A】 | 大 tool result 不能走 WS |
| WebSocket 超时 | 受函数超时限制，超时返回 **1006** | 【A】 | 必须实现重连 |
| WS 握手 Request Headers | 总和 **≤ 8 KB** | 【A】 | — |
| WS 握手 Path（含 Query） | **≤ 8 KB** | 【A】 | — |
| WS 握手 Body | **不支持**（发了被忽略） | 【A】 | 认证信息不能放 Body |
| WS 握手 Response Headers | 总和 **≤ 8 KB**，超出返回 502 | 【A】 | — |
| HTTP 响应 Headers | 总和 **≤ 8 KB**，超出返回 502 `BadResponse` | 【A】 | — |

### 10.2 Agent Sandbox

| 限制项 | 数字 / 状态 | 出处 | 后果 |
|---|---|---|---|
| 沙箱最长存活 | **24 小时** | 【B】 | 超长任务必须分段 |
| **pause / resume** | **需白名单**（allowlisted accounts） | 【A】 | 🔴 想靠暂停省钱必须先申请 |
| `run_code` 语言 | **Python / JS / TS / Bash**；**Java、R 不支持** | 【A】 | 只规划这四种 |
| 图表输出 | **只能 file-based**（沙箱内生成图片文件再取回） | 【A】 | 不能从返回体直接拿图 |
| `Network Config Update` | **受限**（可调但效果有限） | 【A】 | 网络策略建 Template 时定死 |
| Volume / Snapshot / Team / Access Token | **不兼容** | 【A】 | workspace 走 NAS |
| 文件自定义 metadata | **不支持**（需 envd 0.6.2+） | 【A】 | 元数据另建表 |
| Metrics 的 Disk / page-cache | **返回占位值，不可用** | 【A】 | 别靠它做容量规划 |

### 10.3 存储

| 限制项 | 数字 | 出处 | 后果 |
|---|---|---|---|
| NAS 挂载点 | **≤ 5 个/函数/地域** | 【A】 | — |
| OSS 挂载点 | **≤ 5 个/函数/地域** | 【A】 | NAS 与 OSS 挂载的本地目录**不能冲突** |
| NAS 协议 | **仅 NFS，不支持 SMB** | 【A】 | — |
| NAS 本地目录 | 建议 `/home` `/mnt` `/tmp` `/data` 的子目录；**禁用 `/bin` `/opt` `/var` `/dev`** | 【A】 | 用错目录挂载会失败 |
| 挂载后冷启动 | **+2–5 秒** | 【B】 | 🔴 直接影响 TTFT，对交互式场景是硬伤 |
| OSS 挂载小文件写吞吐 | 约为本地 SSD 的 **1/10** | 【B】 | 先写 `/tmp` 再批量上传 |
| NAS VPC 前提 | **只能挂私有 VPC 内的挂载点** | 【A】 | 必须先配 VPC 网络 |

### 10.4 计费

| 项 | 单价（上海） | 出处 | 备注 |
|---|---|---|---|
| AgentCore 协作专享管控面 | **¥3/小时 ≈ ¥2,160/月** | 【A】 | **不缩容到 0**，不删就一直烧 |
| 百炼 token | 按模型 | 【A】 | 不同模型 tokenizer 不同 |
| AgentRun CU | 约 1 元 ≈ 1000 CU，每月 100 万 CU 免费额度 | 【B】 | 个人测试基本不花钱 |

---

## 11. 分阶段实施与验收标准

**总计约 20 个工作日（4 周）**，不含 V2 项（LSP、continuable subagent）。

### Phase 0 · 地基（2 天）

| 任务 | 产出 |
|---|---|
| Dockerfile | `python:3.11-slim`，`--platform=linux/amd64`，含 `/bin/bash` 和 `/bin/sh` |
| FastAPI 骨架 | `/healthz`（返回 200）、`/readyz`（依赖检查）、`POST /v1/chat/completions`（OpenAI 兼容） |
| SSE 输出 | 真分块，响应头带 `Transfer-Encoding: chunked` |
| 事件表 | RDS PostgreSQL 建表 + 索引 + 分区 |
| AgentRun Agent | 容器镜像模式，填端口 + 健康检查，**会话隔离开、并发 1、TTL 21600、idle 1800** |

**验收**：带 `dsh-session-id: test-1` 打两次 `/v1/chat/completions`，返回 SSE 流；两次请求落到同一实例（看实例日志的 request id）；事件表里有落盘记录。

### Phase 1 · 会话与执行世界（4 天）

| 任务 | 产出 |
|---|---|
| `SessionEvent` 模型 | 事件类型枚举 + payload schema（含 `request/header` 快照） |
| `derive_messages()` | fold 函数，只认 3 种 surface 事件 + `surfaceOp` |
| Agent Sandbox 适配层 | `files.*` / `commands.run` / `pty` / `run_code` 四个薄封装 |
| workspace 挂载 | NAS 挂 `/mnt/nas`，目录约定 `/mnt/nas/{session_id}/` |
| `prestop` 兜底 | 实例销毁前 flush 未落盘事件 |

**验收**：跑通「模型说『帮我看看 /mnt/nas 下有什么文件』→ 调 fs 工具 → 拿到真实目录列表 → 模型总结」；杀掉实例后重新请求，能重建 messages。

### Phase 2 · 工具与 Agent 循环（4 天）

| 任务 | 产出 |
|---|---|
| `define_tool` | 六字段契约（name/description/parameters/output.schema/output.render/execute）+ JSON Schema 校验 |
| 工具五段管线 | **校验 → 权限（fail-closed）→ 执行 → 截断 → 记录** |
| 并发池 + barrier | 只读工具并行；有副作用工具阻塞等待（抄 DSH `tool-calls.ts` 的滚动池语义） |
| turn / step 模型 | 1 step = 1 次模型请求 + 它的 tool calls；1 turn = 0..N steps |
| 基础工具集 | fs 5 个 + bash + web_search（百炼）+ web_fetch（百炼） |
| 模型适配器 | 百炼 OpenAI 兼容，base_url 配置化 |

**验收**：模型在一个 turn 内并行调 3 个只读工具 + 顺序调 2 个写工具，顺序正确；权限被拒时**默认拒绝**且事件留痕。

### Phase 3 · 上下文治理（4 天）

| 任务 | 产出 |
|---|---|
| `token_meter` | 读 `response.usage`，累计 + 阈值触发 |
| 微压缩 | 旧 tool result → 摘要 |
| 全量摘要 | 追加 `compaction/summary` 事件 + `surfaceOp: replace` 遮蔽范围 |
| `spill` | 单 tool result > 阈值 → 写 OSS，回填引用 |
| `skill` | markdown 加载 + catalog 工具 + 按需注入 |

**验收**：造一个超长会话（>20 轮大 tool result），压缩触发后**模型仍能正确回答早期问题**（说明摘要没丢关键信息），且**旧事件仍在表里**（可审计）。

### Phase 4 · 人机协作与长任务（3 天）

| 任务 | 产出 |
|---|---|
| FNF 流程定义 | `waitForCallback` 的审批流程 |
| 审批回传 | `ReportTaskSucceeded` / `ReportTaskFailed` + taskToken |
| checkpoint | 按 §7.2 的五个时机打点 + schema 版本号 |
| 恢复路径 | 管控逻辑：查 RDS/NAS → 有则恢复，无则新建会话 |

**验收**：发起一个需要审批的工具调用 → 流程暂停 → 通过 API 回调批准 → Agent 恢复并完成。**中途手动杀掉实例**，恢复后不重复已完成的步骤。

### Phase 5 · 观测与协议收尾（3 天）

| 任务 | 产出 |
|---|---|
| ARMS 探针 | `aliyun-bootstrap -a install` + `aliyun-instrument python app.py` |
| AgentLoop 接入 | 建 AgentSpace，绑 CMS 2.0 |
| MCP client | 官方 Python SDK + 工具注册 |
| MCP 网关（可选） | 用 AgentRun 的 MCP SSE / Streamable HTTP 亲和对外暴露 |
| 文档 | 「怎么加一个工具」「怎么加一个 skill」两篇 |

**验收**：AgentLoop 能看完整 Trace（每一步输入输出、工具调用、耗时、token）；接一个官方 MCP server 的工具能用。

### 11.1 最小可运行骨架（Phase 0 的代码形状）

```python
# app/main.py
import json
from fastapi import FastAPI, Request
from fastapi.responses import StreamingResponse

app = FastAPI()

@app.get("/healthz")
def healthz():
    return {"ok": True}

@app.get("/readyz")
def readyz():
    return {"ok": True}          # 加：DB 连通性、NAS 可写

@app.post("/v1/chat/completions")
async def chat(request: Request):
    body = await request.json()
    session_id = request.headers.get("dsh-session-id", "default")

    async def gen():
        # 1. 从 RDS 载入该 session 的事件流
        # 2. derive_messages() fold 出 messages
        # 3. 调百炼，逐 chunk yield
        # 4. 每个 chunk 落成 assistant/chunk 事件
        yield f"data: {json.dumps({'delta': 'hi'})}\n\n"
        yield "data: [DONE]\n\n"

    # 关键：必须真的分块，不能缓冲完再发
    return StreamingResponse(gen(), media_type="text/event-stream")
```

```dockerfile
# Dockerfile
FROM --platform=linux/amd64 python:3.11-slim
RUN apt-get update && apt-get install -y --no-install-recommends bash && rm -rf /var/lib/apt/lists/*
WORKDIR /app
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt
COPY . .
EXPOSE 8000
CMD ["uvicorn", "app.main:app", "--host", "0.0.0.0", "--port", "8000"]
```

🔴 **三个必检项**：① `--platform=linux/amd64`（Apple Silicon 上不加会构建出 ARM 镜像）② 基础镜像必须有 `/bin/bash`（`slim` 有，`alpine` 没有）③ `--host 0.0.0.0`（`127.0.0.1` 收不到平台转发）

---

## 12. 我没核实的地方（你必须自己验证）

**这一节比前面任何一节都重要——因为以下每一条如果与我的理解不符，都可能推翻对应的设计。**

| # | 待验证 | 为什么关键 | 怎么验 |
|---|---|---|---|
| 1 | **AgentRun 容器镜像模式能否 SSE 流式** | 官方说 AgentRun「提供标准化 HTTP/WebSocket 接口，并支持 SSE」【B】，但 FC 层面「SSE 只支持 Web 函数」【A】。**AgentRun 是否已把容器镜像包装成 Web 函数形态，我没查到明确说明** | 部署一个最小 SSE 服务，`curl -N` 看是否逐块到达 |
| 2 | **AgentRun 是否暴露 `sessionTTLInSeconds` / `affinityHeaderFieldName` 配置** | 我的 Phase 0 假设你能自己设。如果被 AgentRun 包装掉（只能用 `X-AgentRun-Session-ID`），那 §1.2 的决策 6/8 要改 | 控制台逐个翻高级配置项 |
| 3 | **Agent Sandbox Template 能否挂 NAS** | 如果能，workspace 持久化最干净；如果不能，只能靠 `files.*` 往 NAS 同步 | 建 Template 时看有没有挂载配置 |
| 4 | **Agent Sandbox `pause` 白名单怎么申请** | 直接决定成本模型（能否靠暂停省钱） | 提工单问 |
| 5 | **AgentRun 能否作为 FNF 的 `resourceArn`** | Phase 4 的审批方案依赖它。如果不行，要用 FC 函数作 mediator 转调 AgentRun | 建一个 `waitForCallback` 流程试 |
| 6 | **AgentLoop 探针对自研 FastAPI 应用的兼容度** | 官方说支持「各类自研 Agent 系统」+ OTel 标准协议【A】，但探针主要针对已知框架 | 装上探针，看 Trace 里有没有你的自定义 span |
| 7 | **百炼 Responses API 的 `web_extractor` 行为细节** | 它和 `web_search` 是配套的，但抽取质量、超时、反爬处理未知 | 拿几个真实站点试 |
| 8 | **NAS 挂载对 AgentRun 容器镜像模式是否同样生效** | 我是按 FC 的 NAS 文档推的，AgentRun 可能封装不同 | 部署后 `df -h` 看 |
| 9 | **Tablestore Python SDK 的具体 API 形态** | 我只确认了有 Python SDK，没核具体写法 | 读 SDK 文档 |
| 10 | **AgentGateway / AI 网关是否需要前置** | 生产环境通常要在 AgentRun 前面加 API 网关做鉴权和限流。我这张表没包含这一层 | 按你的安全要求决定 |

> **建议的验证顺序**：先验 #1 和 #2（1 天，决定地基能不能立），再验 #3 #8（半天，决定存储形态），其余在对应 Phase 开始前验。

---

## 13. 一页速查：DSH 能力 → 承接方

| DSH 能力块 | 承接方 | 工作量 |
|---|---|---|
| 文件 / 命令 / PTY / 代码执行 | **Agent Sandbox（E2B）** | 🟢 小 |
| 网络搜索 / 抓取 / 代码解释 | **百炼 Responses API 内置工具** | 🟢 极小 |
| 模型接入 | **百炼 OpenAI 兼容** | 🟢 小 |
| 会话事件日志 + fold 投影 | **RDS PostgreSQL + 自写** | 🟡 中 |
| workspace | **NAS** | 🟢 小 |
| 大文件 / transcript / 附件 | **OSS** | 🟢 小 |
| 观测 / 评估 / 审计 | **AgentLoop + ARMS 探针** | 🟡 中 |
| 人工审批 / 长任务 | **FNF `waitForCallback`** | 🟡 中 |
| 压缩 / token 计量 / spill | **自写** | 🔴 大 |
| skill | **自写** | 🟡 中 |
| subagent（基础） | **自写（asyncio）** | 🟡 中 |
| subagent（continuable） | **自写** | 🔴 大（建议 V2） |
| MCP client | **MCP Python SDK** | 🟡 中 |
| LSP | **自建** | 🔴 大（建议 V2） |
| 多 Agent 治理 | AgentCore Team（付费）或自写 | 🔴 大 |
| 插件 / 自我修改 / 4 层配置 | ❌ **放弃** | — |

---

## 14. 三条最容易踩的坑（写在最后，因为最容易忘）

1. **不要把「会话亲和」当成「状态持久化」。** 会话亲和的 TTL 上限是 6 小时，超时后 FC 直接销毁会话并且不再保证亲和。**状态的权威必须在 RDS/NAS 里，内存只是缓存。** DSH 的教训正好相反——它把内存当权威，这在单机成立，在 Serverless 不成立。

2. **不要在 AgentRun 上既开 Cookie 亲和又想要单写者。** Cookie 亲和会强制单实例并发 200。要单写者，走「会话物理隔离」（强制并发 1）。

3. **不要用 Agent Context / Context Engine 做单会话压缩。** 它管跨会话长期记忆，你管单会话窗口。两层都压 = 信息二次损失。边界必须写进设计文档。

---

## 附录：来源

| 内容 | 来源 | 级别 |
|---|---|---|
| FC 会话亲和参数（TTL/IdleTimeout/Concurrency/header 命名约束/disableSessionIdReuse） | `help.aliyun.com/zh/functioncompute/api-fc-2023-03-30-struct-headerfieldsessionaffinityconfig` | 【A】 |
| Cookie 亲和、强制并发 200、Cookie 名与 Max-Age | `help.aliyun.com/zh/functioncompute/fc/cookie-affinity` | 【A】 |
| 会话物理隔离（强制并发 1）、三种亲和方式、prestop 钩子、恢复需管控服务 | 阿里云开发者社区《AI Agent 运行时相比传统应用有什么不同》+《函数计算进化之路：AI Sandbox 新基座》 | 【B】 |
| WS/SSE/gRPC 触发器限制（6MB、8KB、无 Body、1006、仅 Web 函数支持 SSE、仅 Custom Runtime/Container 支持 WS） | `helpcdn.aliyun.com/document_detail/71229.htm` + `help.aliyun.com/zh/functioncompute/fc/does-function-compute-support-sse-streaming-response` | 【A】 |
| E2B 兼容矩阵（pause 白名单、files 部分兼容、run_code 语言、图表 file-based、Network Config Update 受限、Volume 不兼容、metadata 限制） | `help.aliyun.com:443/en/functioncompute/e2b-compatibility-explanation` + `help.aliyun.com/zh/agent-sandbox/e2b-sdk-compatible-api-list` | 【A】 |
| NAS 挂载（≤5 个、NFS only、目录约束、VPC 前提、自动配置） | `help.aliyun.com/document_detail/2513529.html` | 【A】 |
| 挂载冷启动 +2–5s、OSS 小文件吞吐 1/10 | 阿里云开发者社区《FC 代码包超 500MB 怎么传》 | 【B】 |
| AgentRun 会话亲和、OpenAI 端点、X-AgentRun-Session-ID、代码包/镜像模式 | `help.aliyun.com/zh/functioncompute/fc/agent-integration` + `alibabacloud.com/help/zh/doc-detail/3049249.html` | 【A】 |
| AgentRun 24h 沙箱、会话即沙箱、MCP SSE/Streamable HTTP 亲和、三模板 | 阿里云开发者社区《突破 Serverless 无状态限制》 | 【B】 |
| FNF `waitForCallback`、taskToken、ReportTaskSucceeded、人工审批场景 | `help.aliyun.com/knowledge_detail/130496.html` + `intl.aliyun.com/help/zh/doc-detail/357640.htm` | 【A】 |
| Agent 状态保存工程建议（checkpoint 时机、schema 版本化、不可回滚外部世界） | 阿里云开发者社区《长任务不重来：Agent 状态保存的工程设计》 | 【B】 |
| 百炼 OpenAI 兼容 base_url、地域差异、模型列表 | `helpcdn.aliyun.com/document_detail/2579562.html` + `help.aliyun.com/document_detail/2938331.html` | 【A】 |
| 百炼 Responses API 内置 `web_search` / `web_extractor` / `code_interpreter` | `docs.bailian.console.aliyun.com/zh/model-studio/qwen-code-interpreter` | 【A】 |
| AgentLoop 三种接入方式、ARMS 探针命令与环境变量、AgentRun 默认接 CloudMonitor 2.0 | `help.aliyun.com/zh/document_detail/3033823.html` + `alibabacloud.com/zh/product/agentloop` | 【A】 |
| Tablestore / Lindorm / RDS / Tair 选型与成本对比、TTL 与 max versions | `devopsschool.com/tutorials/alibaba-cloud-tablestore-tutorial-*` + `itechguides.com/compare/alibaba-cloud-lindorm-vs-alibaba-cloud-tablestore/` | 【B】 |
| DSH 侧全部源码证据（226 包、事件模型、tool 五段管线、`defineTool` 六字段、subagent continuable、四种模式、stdio-only） | 本机仓库 `/Users/panxingyang/workspace/deepseek-harness` 实读核实 | 本地核实 |

