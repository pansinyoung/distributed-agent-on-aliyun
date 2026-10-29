# DSH 0.2.2-alpha.2 架构实测 diff（对照 rc.8）

> 实测对象：`/Users/panxingyang/workspace/deepseek-harness`，commit `f3803c96`，2026-10-28，tag `dsh-v0.2.2-alpha.2`。
> 基线：rc.8，commit `141eb6f`，2026-08-19。
> 所有结论来自逐文件比对源码，不是读 release notes 转述。

---

## 0. 一句话结论

**这不是版本迭代，是分层重构。** 三件事从 `agent.ts` 里被拆了出来，两件事（系统提示词、流式 chunk）从"请求参数"和"逐条事件"搬进了"模型可见历史"，而 E2B 执行后端被整组删除换成了 SSH。

其中**一项直接推翻之前那份 654 行阿里云映射表的核心论断**——见 §5。

---

## 1. 规模：包数 +63%，四个模块重写

| 指标 | rc.8 | 0.2.2-alpha.2 | 变化 |
|---|---|---|---|
| 包总数 | 226 | **369** | +63% |
| Session 日志格式版本 | `v0` | **`v4`** | 4 次迁移 |
| 表面事件类型 | 3 | **5** | +2 |
| 持久事件类型 | 13 | **15** | 删 2 加 4 |
| `agent-loop/agent.ts` | 515 行 | **745 行** | +45% |
| `session/surface.ts` | 461 行 | **712 行** | +54% |
| `core/tools/` | 1 个 `index.ts` | **9 文件 3710 行** | 子系统化 |

### 文件级增删

| 文件 | rc.8 | 0.2.2 | 说明 |
|---|---|---|---|
| `agent-loop/tool-calls.ts` | 289 行 | **删除** | → `core/tools/pipeline.ts` **771 行** |
| `agent-loop/inbox.ts` | 内嵌于 `agent.ts` | **新增 232 行** | 收件箱独立 |
| `agent-loop/assistant-stream.ts` | 不存在 | **新增 140 行** | 流式装配独立 |
| `session/invariant.ts` | 存在 | **删除** | 运行时 invariant 插件一并移除 |
| `session/json.ts` | 存在 | **删除** | |
| `session/chunk-rows.ts` | 存在 | **删除** | chunk 不再是独立事件 |
| `session/fork.ts` | 不存在 | **新增 30 行** | 分叉独立 |
| `session/seq-ranges.ts` | 不存在 | **新增 73 行** | seq 区间代数 |
| `session/tool-history.ts` | 不存在 | **新增 75 行** | 工具历史投影 |

### 包分组增删（架构级）

**删除的分组**：`e2b/`、`self-modification/`、`hooks/`、`examples/`、`support/`

**新增的分组**：`attachment/` `browser-use/` `client/` `computer-use/` `deliverables/` `document/` `extensions/` `feedback/` `goal/` `host/` `jobs/` `mcp/` `ptc-runtime/` `sandbox/` `schedule/` `session-query/` `spill/` `ssh/` `storage/` `telemetry/` `workspace/`

---

## 2. 变化一：系统提示词变成了「模型可见历史」

**这是本次最有设计意味的改动。**

rc.8 里系统提示词是**请求参数**：

```ts
// rc.8 types.ts
export interface EpochHeader {
  config: LlmCallConfig
  system?: string        // ← 系统提示词挂在这里
  tools?: ToolSchema[]
}
```

0.2.2 里它变成了 **surface node 0**，一个独立事件：

```ts
// 0.2.2 types.ts:240-257
/**
 * Logged request state outside derived history: call config and tools. The
 * system prompt is derived history — surface node 0, a `system/message` event.
 */
export interface EpochHeader {
  config: LlmCallConfig
  tools?: ToolSchema[]
  /** Retired request text; system prompts belong to system/message events.
   * @persistenceReserved */
  system?: never         // ← 类型层面禁止再用
}
```

```ts
// 0.2.2 types.ts:338
'system/message': { turn: number; step: number; message: SystemMessage }
```

**为什么要这么改？** 因为一条铁律：

> **Model-visible means logged.**（模型可见 ⟺ 已记录）

rc.8 的系统提示词是**模型可见但未记录**的——它存在于 `request/header` 快照里，但不在 surface 上，所以：
- 你无法用统一的 surface 折叠逻辑重放"那次请求的 system prompt 是什么"
- 系统提示词变化不产生"历史节点"，只是一个 header 快照被覆盖
- 你没法对系统提示词做 `surfaceOp: replace`（压缩系统提示词）

0.2.2 把它拉进 surface 后，语义统一了：**所有模型能看到的东西，都是 surface 上可寻址、可替换、可审计的节点。**

代价是复杂度陡增。看 `types.ts:324-337` 那九行注释，描述了完整的状态机：

| 情形 | 行为 |
|---|---|
| 第一条 | 作为 surface node 0，在第一个 `user/message` 之前 |
| 支持 in-history 的 route | 可直接追加非空变更，延续同一 series |
| 不支持的 route | 把文本**归一化**到第一个 system 节点 |
| 渲染为空 | **清除所有** active system 节点，不留旧指令可见 |
| 后续空节点 | 休眠，投影为空 |
| 恢复非空文本 | 遵循同一 route/series 规则；空节点**永不恢复**旧文本 |

**这三个坑**：
1. 归一化要"通过已记录的 per-node replacement"完成——不是直接改日志，是**追加替换事件**。日志永远 append-only。
2. 空渲染必须清掉所有节点。如果只清最后一个，前面残留的旧 system 提示词会继续影响模型。这是**安全属性**，不是整洁属性。
3. 空的 head + 无 active 后续节点 = "没有系统提示词"，这是个**有意义的合法状态**，不是错误。

---

## 3. 变化二：`assistant/chunk` 事件被删除，流被嵌入消息

rc.8 的做法是 token 级逐条落盘：

```ts
// rc.8 agent.ts:348-352
for await (const chunk of stream) {
  chunkSeqs.push(this.session.append('assistant/chunk', { turn, step, chunk }).seq)
  assembler.push(chunk)
}
// 然后 assistant/message 用 sourceEventSeqs 引用这些 chunk seq
this.session.append('assistant/message', {...}, { surfaceOp: 'append', sourceEventSeqs: chunkSeqs })
```

0.2.2 的做法：

```ts
// 0.2.2 types.ts:349-357
'assistant/message': {
  turn: number
  step: number
  message: AssistantMessage
  /** Exact timed model stream, compacted without joining delta boundaries. */
  stream: AssistantStreamRecord[]     // ← 流直接嵌在消息里
  usage?: TokenUsage
  interrupted?: true
}
```

**为什么改？** rc.8 的做法有两个真实问题：

1. **日志条数爆炸。** 一次 2000 token 的响应可能产生几百条 chunk 事件。会话长一点，日志就是几十万条，SQLite 写入放大严重，加载要扫全量。
2. **`sourceEventSeqs` 引用链膨胀。** 一条 `assistant/message` 引用几百个 chunk seq，`assertProvenance` 每次要校验这一整串（rc.8 `surface.ts:210-243`），是 O(n) 的校验成本。

0.2.2 保留了什么？**"compacted without joining delta boundaries"** 这句是关键——压缩了，但**不合并相邻 delta 的边界**。所以：
- 你仍然知道"第 37 个 token 是在第 1200ms 到达的"（`AssistantStreamRecord` 带时间）
- 你不再有几百条日志记录，而是一条消息带一个数组

**这是一次教科书级的权衡修正**：牺牲了"流式过程中断线可续"（未结算的流不落盘），换来了日志体积和加载性能。代价写在架构文档里：

> a hard process loss before settlement leaves no durable attempt stream
> （结算前的硬进程丢失不会留下任何持久化流）

**新增 `assistant/attempt`** 就是补偿这个代价：

```ts
// 0.2.2 types.ts:358-363
/**
 * One model attempt that committed no surface message. The embedded stream
 * preserves a failed, retried, cancelled, or stream-error attempt that
 * reached settlement without fabricating model-visible history.
 */
'assistant/attempt': { turn: number; step: number; stream: AssistantStreamRecord[] }
```

注意"reached settlement"（到达结算点）——**失败的尝试也会结算并落盘**，但**不产生模型可见历史**。这解决了 rc.8 的一个真实缺陷：重试时第一次失败的内容完全消失，你事后无法诊断"为什么重试了"。

---

## 4. 变化三：工具调度从「Agent 循环」搬进「工具注册表」

rc.8 的工具调度住在 Agent 循环里：

```ts
// rc.8 agent-loop/tool-calls.ts（289 行，已删除）
export async function executeToolCalls(
  ctx: Context,
  turn: number,        // ← 工具调度知道 turn
  step: number,        // ← 也知道 step
  toolCalls: ToolCallBlock[],
  signal: AbortSignal,
  acceptContext: (context: UserMessage) => void,
): Promise<{ concluded: boolean }>
```

0.2.2 搬进了 `core/tools`：

```ts
// 0.2.2 core/tools/src/pipeline.ts:62 起，771 行
export class ToolExecutionPipeline { ... }
// pipeline.ts:771
export { executeToolBatch } from './batch.ts'
```

**为什么这是对的？** rc.8 那个签名暴露了一个分层错误：**工具调度接受 `turn` / `step` 参数，意味着工具注册表知道 Agent 的循环编号。** 但循环是 `agent-loop` 的概念，工具注册表不该知道。

后果是 rc.8 里 `tool-calls.ts` 无法独立测试——你测工具并发调度，得先构造一个假的 turn/step 上下文。

0.2.2 改成三段 waterfall，参数是 `ToolExecutionInput` 而不是 `(turn, step)`：

| 阶段 | 位置 | 职责 |
|---|---|---|
| `tools/pre-execute` | `pipeline.ts:200` | 参数校验、权限拦截、入参改写 |
| `tools/execute` | `pipeline.ts:300` | 实际执行，可替换 exec（含 signal） |
| `tools/post-execute` | `pipeline.ts:488` | 结果改写、错误归一、附加 context |

`post-execute` 有两条硬校验值得抄（`pipeline.ts:502-510`）：

```ts
throw new TypeError('tools/post-execute accept decision cannot replace both value and content')
throw new TypeError('tools/post-execute cannot replace the value of a failed result')
```

第一条防"同时改两个字段导致语义不明"，第二条防"把失败的结果改成成功"——**这是安全边界，不是类型洁癖。**

同时工具启用变成**按步骤条件化**：

```ts
// 0.2.2 core/tools/src/index.ts:251
executable(name: string, scope?: ScopeKey, toolSet?: ResolvedToolSet): ToolDefinition | undefined
```

`toolSet` 是新增参数。配合 `pipeline.ts:72` 的 `this.tools.executable(name, scope, toolSet)`，意味着**同一次 step 里，工具可能是动态启用的**（比如凭据就绪才暴露 Web 搜索——release notes 里确有这一条）。

---

## 5. 🔴 变化四：E2B 执行后端整组删除，换成 SSH

**这一条直接推翻之前那份阿里云映射表的核心结论。**

### rc.8 的事实

`packages/e2b/` 分组，三个 provider：

- `@deepseek-ai/dsh-e2b`（客户端）
- `@deepseek-ai/dsh-fs-e2b`（文件系统 provider）
- `@deepseek-ai/dsh-subprocess-e2b`（子进程 provider）

当时的论断是：阿里云 Agent Sandbox 兼容 E2B SDK/API，所以**「只改 endpoint + API key，不改代码」**。

### 0.2.2 的事实

```sh
$ ls -d packages/e2b
packages/e2b 不存在！

$ ls packages/*/ | grep -i e2b
无 e2b 目录

$ grep -rl "e2b" packages/*/*/package.json
（无输出）
```

**E2B 支持被彻底移除，连一个依赖引用都不剩。**

取代它的是全新的 `ssh/` 分组：

| 包 | 角色 |
|---|---|
| `@deepseek-ai/dsh-ssh` | SSH 连接能力 |
| `@deepseek-ai/dsh-fs-ssh` | 文件系统 provider（远程） |
| `@deepseek-ai/dsh-subprocess-ssh` | 子进程 provider（远程） |
| `@deepseek-ai/dsh-sandbox-ssh` | 沙箱 backend（远程） |
| `@deepseek-ai/dsh-ssh-helper-runtime` | 远端 helper 运行时 |

而 `sandbox/` 分组只剩**本地**实现：`sandbox`、`sandbox-local`、`sandbox-policy`、`sandbox-windows-acl`。

### 这意味着什么（诚实版）

**坏消息**：执行世界不再"白送"。原方案里最省事的一块——"DSH 已有 E2B provider，阿里云 Agent Sandbox 兼容 E2B，改 endpoint 就行"——**已经完全失效**。你要么自己写 provider，要么改架构。

**好消息，而且更大**：SSH 远程执行这条路**对阿里云更合适**：

| | Agent Sandbox（E2B） | ECS + SSH |
|---|---|---|
| DSH 原生支持 | ❌ 已删除 | ✅ `fs-ssh` / `subprocess-ssh` / `sandbox-ssh` |
| 暂停/恢复 | 需**申请白名单** | 不需要 |
| 计费 | 按沙箱时长 | 按 ECS 实例 |
| 网络策略 | `Network Config Update` 受限，必须建 Template 时定死 | 安全组随时改 |
| 语言支持 | `run_code` 仅 Python/JS/TS/Bash | 整台机器随便装 |
| 状态 | 沙箱生命周期 | 持久机器，`/home` 一直在 |
| 成熟度 | 新产品 | 十年级基础设施 |

**结论：阿里云方案应该从「Agent Sandbox」转向「ECS + SSH」。** 这不是妥协，是**顺着 DSH 现在的架构走，反而绕开了 Agent Sandbox 那五个硬限制**（白名单、Template 固化、语言限制、≤5 挂载点、暂停不可用）。

要验证的一点：DSH 的 SSH provider 是否接受标准 `~/.ssh/config` 或显式 host/key 配置。如果是，连一台阿里云 ECS 就是改配置。这个我还没核实，列入待验证。

---

## 6. 其余五处值得注意的变化

### 6.1 `agent/request` 被标记弃用，且加了运行时断言

rc.8 里这是改请求配置（含路由）的标准入口。0.2.2 保留了 waterfall，但 `agent.ts:636-639` 直接抛错：

```ts
throw new Error(`agent "${this.id}": agent/request proposed "${proposedConfig.provider}/${proposedConfig.model}"
  after step preparation fixed "${selection.provider}/${selection.model}";
  use agent.model.select() before step preparation`)
```

**这是"弃用机制"的一个好范例**：不删事件（老插件不会崩），但把违反新契约的用法变成硬错误，错误信息里直接给出替代方案。

### 6.2 新增 `developer/message` 事件

```ts
// types.ts:316-323
/** An incremental agent session change admitted at the named turn and step. */
'developer/message': {
  turn: number
  step: number
  message: DeveloperMessage
  /** Earlier request/header defining every tool addition; required exactly when additions are present. */
  headerSeq?: SessionSeq
}
```

`headerSeq` 这条约束很讲究：**如果你在这次 developer 消息里新增了工具，必须引用定义了这些工具的 `request/header` seq**。这保证"工具声明"和"工具使用"之间有可追溯的因果链，不是凭空出现的。

### 6.3 `todo/write` 不再是 session 事件

rc.8 里 todo 是 `SessionEventMap` 的正式成员。0.2.2 里已从事件表消失，只存在于 experimental 的测试 fixture 中。

**这个方向是对的**：todo 是 UI 状态，不是模型可见历史，也不该占持久日志。它迁到了投影接缝（见 §6.4）。

### 6.4 新增投影接缝 `ctx.sessionProjections`

架构文档原文：

> `dsh-session-projection` owns `ctx.sessionProjections`: registered units fold committed events; carriers read selected client views through `snapshot()`. Domain services await registration, retain private handles, and expose business methods.

包：`session-projection` + `session-projection-cache`。

**这是在开源层面承认了一个现实**：所有消费会话日志的模块都在重复做"fold 已提交事件 → 得到我要的视图"。与其各写一遍，不如给一个注册式投影接缝。

对应地，Session 相关包在 0.2.2 里膨胀到 25 个：`session-persistence-jsonl`、`session-checkpoint-policy`、`session-telemetry`、`session-telemetry-otel`、`session-title`、`session-title-llm`、`session-turn-outline`、`session-stats`、`working-directory`、`tool-working-directory`、`message-quotes`、`session-log-deepseek`……

### 6.5 Session 日志格式走到 v4，且有不可变迁移链

rc.8 是 `SESSION_FORMAT_VERSION = 0`，0.2.2 已经到 **v4**。存在完整迁移链：

```
session-format             session-format-v0-to-v1    session-format-v1-to-v2
session-format-v2-to-v3    session-format-v3-to-v4    session-format-catalog
```

物理文件命名也版本化了：v0 用 `session.jsonl[.zstd]`，v1+ 用 `session.vN.jsonl[.zstd]`。

**三条纪律值得抄**（来自架构文档）：
1. **已提交的代际文件永不重命名、替换、删除** —— 写入时在旧文件旁边发布新代际的副本
2. 每个相邻迁移包**只负责一个 `vN -> vN+1` 步骤**
3. 读取时拒绝未来版本，而不是尽力解析

这是"数据格式演进"的成熟做法。对比 rc.8 那句 "no compatibility is implied, incompatible logs are rejected"——**从"不兼容就拒绝"进化到"保留历史 + 单步迁移链"。**

---

## 7. 对既有两份文档的影响

| 文档 | 影响程度 | 具体失效内容 |
|---|---|---|
| `dsh-to-aliyun-mapping.md`（654 行） | 🔴 **§2 整节作废** | 「E2B 兼容 → 只改 endpoint」的论断失效；执行世界方案要整体转向 ECS + SSH |
| 同上 §1 决策 13 | 🔴 要改 | "执行世界：Agent Sandbox（E2B）" → 应为 "ECS + SSH provider" |
| 同上 §12 待验证 #3 #4 #8 | ⚪ 失效 | Agent Sandbox 暂停白名单、Template 挂 NAS 都不再是必答题 |
| 同上 §3 会话数据平面 | 🟡 需更新 | 事件表 schema 要改成 15 种事件、加 `stream` 字段、surface 从 3 类变 5 类 |
| 同上 §10.2 硬数字 | 🟡 部分失效 | Agent Sandbox 的限制不再是主要约束 |
| `agent-harness-from-scratch.md`（26 课，已写 3 课） | 🔴 **全部需重写** | 见下 |

### 26 课的具体失效点

| 课 | 状态 | 原因 |
|---|---|---|
| 课 1 · 事件日志 | 🟡 保留但重写 | 设计不变，但事件表从 13 → 15；`json.ts` 已删 |
| 课 2 · 表面投影 | 🔴 重写 | `surface.ts` 461→712；surface 从 3 类 → **5 类**（加 `system/message`、`developer/message`） |
| 课 3 · turn/step | 🔴 重写 | `agent.ts` 515→745；inbox 独立；turn 流程图新增 `agent/assistant-stream`、`request series` |
| 课 4 · 流式装配 | 🔴 重写 | `assistant/chunk` 删除 → `assistant-stream.ts` + 嵌入式 stream |
| 课 6 · 工具调度 | 🔴 作废重写 | 基线换成 `tools/pipeline.ts` 三段 waterfall |
| 课 9–11 · 执行世界 | 🔴 作废重写 | E2B 删除，改 SSH provider |
| 课 13 · 系统提示词 | 🔴 重写 | system prompt 变成 surface node 0 |
| 新课 · 投影接缝 | ✨ 新增 | `ctx.sessionProjections` |
| 新课 · 格式迁移 | ✨ 新增 | v0→v4 迁移链 |
| 新课 · 请求系列 | ✨ 新增 | `startsRequestSeries` / `request/header reason: 'series'` |

---

## 8. 修正后的计划

**课程大纲保持 26 课不变，但内容全部按 0.2.2 重写，并新增三课（合计 29 课）：**

1. **课 0（新增）** · 从 515 行到 745 行：一次真实的重构决策 —— 就用本文档的材料，讲"什么时候该把一个职责从大类里拆出去"
2. **课 2 扩展** · 五个表面事件类型 + 系统提示词作为 node 0 的状态机
3. **课 4 重写** · 为什么 chunk 不该逐条落盘：日志体积 vs 重放保真的真实权衡
4. **课 6 重写** · 三段 waterfall 与"工具注册表不该知道 turn/step"
5. **课 9–11 重写** · SSH provider 体系（而不是 E2B）
6. **课 13 重写** · 系统提示词的六种状态与三条安全属性
7. **课 27（新增）** · 投影接缝：所有消费者都在重复 fold，那就给个接缝
8. **课 28（新增）** · 日志格式演进：从"不兼容就拒绝"到"不可变代际 + 单步迁移链"
9. **课 29（新增）** · 弃用机制：不删事件、加断言、错误信息给替代方案

**映射表侧**：`dsh-to-aliyun-mapping.md` 的 §1 决策 13、§2 整节、§3、§10.2、§12 需要重写；其余章节（会话亲和、TTL、FNF、百炼模型接入）**不受影响**——那些是阿里云的属性，与 DSH 版本无关。

---

## 9. 待验证

| # | 待验证 | 为什么关键 |
|---|---|---|
| 1 | DSH 的 SSH provider 支持哪些认证方式（key / agent / 显式配置） | 决定能否直连阿里云 ECS |
| 2 | SSH helper runtime 需要在远端装什么 | 决定 ECS 镜像要不要定制 |
| 3 | `sandbox-ssh` 与 `sandbox-local` 的策略是否有能力差异 | 决定安全边界是否够 |
| 4 | `375` 个包中哪些是 experimental（可以不学） | 决定 26 课的范围边界 |
| 5 | `AssistantStreamRecord` 的确切结构 | 课 4 的核心数据结构 |

---

*本文档基于 commit `f3803c96` 实测。DSH 迭代极快（10 天 8 个版本），任何结论都要对着当时的源码复核。*

