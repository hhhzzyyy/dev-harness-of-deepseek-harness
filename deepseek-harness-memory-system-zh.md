# DeepSeek Harness 记忆机制

本文档系统整理 DeepSeek Harness 项目中的记忆（Memory）机制——包括核心数据结构、存储方式、检索策略、生命周期管理，以及它与 Session 系统的关系。

---

## 一、核心设计理念

### 1.1 没有独立的 "memory" 包

DeepSeek Harness **没有一个名为 "memory" 的独立核心包**。它的"记忆"是一个**分布式、多层级的系统**，由多个子系统协同构成。

核心的记忆载体是 **Session 事件日志**，围绕它构建了压缩、投影、持久化、溢出处理等多层机制。

### 1.2 事件溯源（Event Sourcing）

整个记忆架构基于事件溯源模式：
- **唯一真相来源**是 append-only 的事件日志
- 所有派生状态（消息历史、表面视图、投影、统计）都从日志折叠而来
- 重放 = 重新折叠，结果确定

### 1.3 结构性记忆为主，语义记忆为扩展

与传统的"向量数据库 + 语义检索"记忆系统不同，DeepSeek Harness 采用的是：
- **内置记忆**：完全围绕对话历史的结构性管理（存储、压缩、持久化）
- **语义记忆**：通过 MCP 协议交由专业的外部系统处理（模型主动调用工具）

这是一个**有意的设计选择**——框架不提供内置的长期语义记忆，而是通过标准化接口接入第三方系统。

---

## 二、核心数据结构

### 2.1 SessionEvent — 记忆的原子单元

记忆的最基本单元是 `SessionEvent`。它是事件溯源的 append-only 日志条目：

```typescript
type SessionEvent<T extends SessionEventType = SessionEventType> = {
  type: K                    // 事件类型
  seq: number                // 单调递增的序列号
  time: number               // Unix 时间戳（毫秒）
  data: SessionEventMap[K]   // 事件负载
  ignorable?: true           // 未知类型时是否可安全跳过
  // 仅 surface 事件有以下字段
  sourceEventSeqs?: number[] // 来源事件序列号（溯源用）
  surfaceOp?: SurfaceOp      // 表面操作：'append' 或 replace
}
```

`SessionEventMap` 是一个**声明合并可扩展**的接口，各领域包向其中注册自己的事件类型。

核心事件类型包括：
- `turn/start`, `turn/end` — 回合边界
- `step/start`, `step/end` — 步骤边界（一次模型调用 + 工具执行）
- `user/message` — 用户消息（surface 事件）
- `assistant/chunk` — 流式输出块（原始 token 级回放）
- `assistant/message` — 助手消息（surface 事件）
- `tool/call`, `tool/result` — 工具调用与结果（后者是 surface 事件）
- `todo/write` — 待办列表快照
- `request/header`, `request/context` — 请求头快照
- `session/end-seed` — 种子历史边界标记

### 2.2 Surface — 模型可见的记忆表面

不是所有事件都对模型可见。`SurfaceEventType` 定义了三种会进入模型上下文的事件：
- `user/message`
- `assistant/message`
- `tool/result`

`SurfaceOp` 支持两种操作：
- `'append'` — 追加到尾部（正常路径）
- `{ op: 'replace', start, end }` — 替换一个范围（压缩机制使用）

`SessionSurface` 接口维护当前的表面视图：
```typescript
interface SessionSurface {
  readonly nodes: readonly number[]   // 当前表面事件 seq 列表
  readonly replaceGeneration: number  // 已发生的替换次数
}
```

### 2.3 SessionHeader — 会话元数据

```typescript
interface SessionHeader {
  readonly version: number           // 日志格式版本
  readonly id: SessionId             // 会话 ID（branded 类型）
  readonly createdAt: number         // 创建时间
  readonly cwd?: string              // 工作目录
  readonly parentSession?: SessionId // 父会话（fork/子代理）
  readonly seedLength?: number       // 种子历史长度
  readonly origin?: 'subagent'       // 来源类型
  readonly delegationDepth?: number  // 委托深度
  readonly agentPreset?: string      // 代理预设
}
```

### 2.4 Compaction 相关类型

压缩是长期记忆的核心机制，通过声明合并扩展 SessionEventMap：
- `compaction/start` — 开始压缩（获取锁）
- `compaction/summary` — 摘要结果（范围、token 数、模型信息）
- `compaction/end` — 结束压缩（释放锁）
- `compaction/prune` — 工具结果裁剪

### 2.5 Projection — 派生记忆视图

`SessionProjectionMap` 是一个声明合并的类型表，各领域包向其中注册自己的投影单元。每个投影单元是一个纯函数式的折叠单元：

```typescript
{
  key: string,
  schema: Schema,
  init: () => State,
  apply: (state: State, event: SessionEvent) => State,
  view: (state: State) => View,
  stateVersion: number
}
```

例如：
- `goal` 包注册 `goal: GoalProjection | null`
- `subagent` 包注册 `subagent` 和 `subagentTiming`

---

## 三、记忆的存储方式

### 3.1 内存存储

`SessionStore`（`ctx.sessions`）是内存中的会话存储，所有活跃会话都驻留在内存中。事件日志是 append-only 的数组，表面视图通过增量折叠维护。

### 3.2 持久化存储

持久化通过 `session-persistence` 能力缝实现，有两个后端：

#### JSONL 后端 (`session-persistence-jsonl`)
- **格式**：每个会话一个文件，第一行是 header JSON，后续每行一个 `SessionEvent` JSON
- **压缩**：默认使用 Zstandard 帧压缩（带校验和）
- **打包**：连续的 `assistant/chunk` 事件被打包成 `text-chunks`/`reasoning-chunks`/`tool-call-chunks` 行（无损，约减小 60%）
- **目录结构**：`root/<project-dir>/<session-id>/session.jsonl.zst`

#### SQLite 后端 (`session-persistence-sqlite`)
- **两张核心表**：
  - `sessions` — 会话元数据（id, version, created_at, cwd, parent_session 等）
  - `events` — 事件表（session_id, seq, type, time, data(JSON) 等）
- **外键级联删除**：删除会话时级联删除事件
- **Journal 模式**：默认 WAL
- **应用 ID**：`0x44534850`（"DSHP"），防止误操作其他数据库

### 3.3 写入协调器（PersistenceCoordinator）

两个后端共享同一个写入协调器，负责：
- **批量写入**：`writeBatchMaxDelayMs` 合并窗口，事件先进入缓冲区再批量写入
- **崩溃修复**：冷加载时检测撕裂尾部，截断后合成 `turn/end {interrupted}` 关闭中断的回合
- **延迟物化**：创建会话时不立即写文件，第一次 append 时才物化
- **LRU 缓存**：冷会话的 preparation 结果缓存

### 3.4 投影缓存（session-projection-cache）

投影单元的状态也被持久化缓存：
- 每行：`(sessionId, key) → { ver, seq, val }`
- `ver` 是单元的 `stateVersion`，不匹配则丢弃重算
- 写入时机：`turn/end`、会话 dispose、以及可配置的节流

### 3.5 Spill 存储

超大工具结果通过 `spill` 系统溢出到文件：
- **后端**：`spill-local` 使用会话作用域的私有文件
- **结果**：返回 `SpillRef { locator, bytes, retrievalHint }`
- 模型看到的是定位符 + 检索提示，需主动读取完整内容

---

## 四、记忆的检索机制

### 4.1 表面推导（deriveMessages）

模型看到的消息历史通过 `deriveEventMessage()` + `foldSurface()` 从事件日志推导而来。这是一个**纯函数**：给定事件日志，确定性地得到消息列表。

关键规则：
- 只有 surface 事件（user/message, assistant/message, tool/result）产生消息
- 替换操作（压缩）会遮蔽旧消息，用摘要替换
- 空内容的 assistant/message 被跳过（仅承载 usage 统计）

### 4.2 压缩检索 — 摘要式记忆

压缩是最核心的"记忆检索"机制。当历史过长时，旧的对话被**摘要成一个检查点**，后续请求只看到摘要 + 近期历史。

检索策略：
- **基于 token 压力**：`ctx.tokenMeter` 测量当前请求的预估 token 数
- **阈值触发**：超过 `thresholdRatio × contextWindow`（默认 0.8）时触发
- **保留尾部**：保留最近的 `retainRatio × contextWindow`（默认 0.16）作为近期上下文
- **平衡边界**：压缩边界必须在完整的工具调用-结果对上，不能切开未回答的工具调用

### 4.3 没有语义检索 / 向量搜索

**DeepSeek Harness 本身不提供语义相似度搜索或向量检索**。记忆检索是纯结构性的：
- 时间顺序的消息历史
- 摘要检查点替换
- 工具结果裁剪（head + tail）

语义记忆（事实记忆、偏好记忆）通过**第三方 MCP 记忆服务器**接入。

### 4.4 投影检索

`session-projection` 系统提供结构化的派生视图检索：
- `ctx.sessionProjections.snapshot(session)` — 同步获取所有注册单元的当前值
- `ctx.sessionProjections.onChanged(listener)` — 订阅变化
- 每个单元是纯函数折叠，事件驱动更新

---

## 五、记忆的写入/更新机制

### 5.1 事件追加（Session.append）

所有记忆写入都通过 `Session.append()` 进入日志。这是唯一的写入路径，保证：
- 序列号连续
- 事件深度冻结（不可变）
- surface 元数据验证
- JSON 可序列化验证

### 5.2 何时产生记忆

| 时机 | 事件类型 | 说明 |
|---|---|---|
| 用户输入 | `user/message` | 用户消息、注入上下文、目标延续 |
| 模型输出 | `assistant/chunk`, `assistant/message` | 流式块 + 完整消息 |
| 工具调用 | `tool/call`, `tool/result` | 工具调用请求与结果 |
| 回合/步骤 | `turn/start`, `turn/end`, `step/start`, `step/end` | 边界标记 |
| 待办更新 | `todo/write` | 完整列表快照（last-write-wins） |
| 请求头 | `request/header`, `request/context` | 配置快照 |
| 压缩 | `compaction/start/summary/end/prune` | 压缩事务记录 + 替换消息 |
| 目标 | `goal/change` | 目标状态快照 |
| 标题 | `session/title` | 会话标题修订 |
| 子代理 | `subagent/descriptor` | 子代理描述符 |
| 种子边界 | `session/end-seed` | 种子/实时历史分界 |

### 5.3 压缩写入流程

压缩是最复杂的写入路径：

1. **追加 `compaction/start`** — 同步获取日志记录的锁
2. **调用 LLM 生成摘要** — 重放被压缩范围的对话 + 压缩指令
3. **追加 `compaction/summary`** — 记录摘要、范围、token 数、模型信息
4. **追加替换用的 `user/message`** — 携带 `surfaceOp: { op: 'replace', start, end }`，这是唯一的表面突变
5. **追加 `compaction/end`** — 释放锁

整个过程在锁的保护下完成。如果崩溃在中间，未配对的 `compaction/start` 会被检测为"孤儿锁"。

### 5.4 持久化写入

- **写后模式**：内存日志先追加，然后通过 `session/event` 事件通知持久化后端
- **批量合并**：`writeBatchMaxDelayMs` 窗口内的事件合并为一次写入
- **显式刷新**：`session/flush` 是并行等待的持久化屏障
- **检查点策略**：`session-checkpoint-policy` 在模型请求前、工具执行前、步骤边界强制刷新

---

## 六、记忆在 Agent 循环中的使用

### 6.1 请求组装

在每个 step 开始时，agent-loop 组装模型请求：
1. 系统提示（system prompt）
2. 工具 schemas
3. **从 session 推导的消息历史**（`session.deriveMessages()`）
4. 当前 step 的用户消息

记忆（历史消息）是**直接注入到 prompt 中的**，不是作为工具。

### 6.2 压缩在循环中的触发点

压缩通过两个钩子接入 agent-loop：

1. **`agent/pre-step` 瀑布** — 每步开始前检查 token 压力，超过阈值则自动压缩
2. **`agent/request-error` 瀑布** — 发生上下文溢出（`CONTEXT_WINDOW_EXCEEDED`）时，强制压缩后重试

### 6.3 记忆对 KV 缓存的影响

- **Append-only 历史**：前缀不变时，provider 的 KV 缓存可复用
- **压缩替换**：从第一个被遮蔽的 token 开始失效
- **系统提示/工具 schema 变化**：也会导致缓存失效

### 6.4 注入式记忆

除了对话历史，还有多种"注入式记忆"通过 `agent.inject()` 进入上下文：
- 文件变更通知
- 子目录 AGENTS.md 内容
- 技能内容（skill loading）
- 定时通知
- 目标延续提示

这些都作为 `user/message` 事件（`source` 区分来源）进入日志和表面。

---

## 七、记忆类型分类

### 7.1 按持久性

| 类型 | 载体 | 生命周期 |
|---|---|---|
| **瞬时记忆** | 内存中的 Session 日志 | 进程内，会话活跃时 |
| **短期记忆** | 表面消息历史（未压缩部分） | 同会话，直到被压缩 |
| **长期记忆** | 压缩摘要检查点 | 同会话，持久化后跨重启 |
| **持久记忆** | 持久化的完整事件日志 | 跨会话、跨进程、跨重启 |

### 7.2 按内容类型

| 类型 | 载体 | 说明 |
|---|---|---|
| **会话记忆** | Session 事件日志 | 完整的交互历史 |
| **摘要记忆** | compaction 检查点 | 旧对话的 LLM 生成摘要 |
| **目标记忆** | goal/change 事件 | 当前目标状态 |
| **待办记忆** | todo/write 事件 | 待办事项列表 |
| **技能记忆** | skill 注册表 + 已加载内容 | 可调用的指令集 |
| **偏好记忆** | settings 系统 | 用户设置和偏好 |
| **工作区记忆** | workspace 系统 | 会话分组与工作目录归属 |
| **溢出记忆** | spill 文件 | 超大工具结果的离线存储 |
| **统计记忆** | session-stats 投影 | 会话统计数据 |
| **标题记忆** | session/title 事件 | 会话标题 |
| **委托记忆** | subagent/descriptor + parentSession | 父子会话关系与委托深度 |

### 7.3 按可见性

| 类型 | 模型可见？ | 说明 |
|---|---|---|
| **表面记忆** | 是 | user/assistant/tool 消息 + 压缩摘要 |
| **日志-only 记忆** | 否 | 边界标记、摘要原文、目标状态、标题等 |
| **投影记忆** | 否（除非通过工具暴露） | goal、stats、title 等派生视图 |
| **溢出记忆** | 间接 | 模型看到定位符 + 检索提示，需主动读取 |

---

## 八、记忆生命周期管理

### 8.1 压缩（Compaction）

压缩是最主要的生命周期管理机制：

- **触发条件**：token 压力超过阈值，或上下文溢出
- **策略参数**：
  - `thresholdRatio`（默认 0.8）— 达到窗口的多少比例时触发
  - `retainRatio`（默认 0.16）— 保留多少近期上下文
  - `compactionRetries`（默认 1）— 压缩后仍超阈值的重试次数
  - `maxOverflowRetries`（默认 1）— 上下文溢出后的最大重试次数
- **摘要生成**：使用配置的 summarization model，重放对话前缀以复用 KV 缓存
- **摘要结构**：固定的 8 节 Markdown 结构（主要请求、关键概念、文件与代码、错误与修复、待办、当前工作、下一步、关键上下文）
- **收敛保证**：摘要必须比原文小，否则拒绝

### 8.2 工具结果裁剪（Tool Result Pruning）

`compaction-tool-result-pruner` 提供模型无关的裁剪：
- 超过 `thresholdChars`（默认 8192）的工具结果被裁剪
- 保留 `headChars`（默认 4096）开头 + `tailChars`（默认 1024）结尾
- 中间用 `[... tool result middle pruned ...]` 标记
- 在压缩触发前执行，可能避免不必要的 LLM 调用

### 8.3 溢出（Spill）

超大输出通过 spill 系统移出上下文：
- 模型只看到定位符和检索提示
- 完整内容存储在会话作用域的文件中
- 模型可通过读取工具按需取回

### 8.4 没有的机制

- **没有 TTL 过期**：事件日志是永久 append-only 的
- **没有遗忘策略**：不会自动删除旧记忆
- **没有归档机制**：会话不会自动归档
- **没有跨会话记忆合并**：每个会话的记忆是独立的

---

## 九、记忆系统与 Session 系统的关系

**Session 是记忆系统的核心载体**。整个记忆架构围绕 Session 构建。

### 9.1 分层架构

```
模型可见表面 (Surface)
    ↑ 推导
事件日志 (SessionEvent log) — 唯一真相来源
    ↑ 追加
各种事件生产者 (loop, tools, compaction, goal...)
    ↓ 订阅
持久化 (persistence)、投影 (projection)、统计 (stats)、UI
```

### 9.2 会话间隔离

- 每个 Session 有独立的记忆
- 父子会话通过 `parentSession` + `seedLength` 建立谱系关系
- Fork 的子会话继承父会话的种子历史，但之后独立
- 子代理（subagent）可以选择继承上下文（fork）或从零开始（spawn）

### 9.3 冷启动与恢复

- 持久化的会话可以通过 `ctx.agents.resume()` 恢复
- 恢复时重新从持久化日志折叠所有派生状态
- 崩溃的会话会被修复：合成中断的工具结果 + 步骤结束 + 回合结束
- 投影缓存加速冷启动（从缓存行开始折叠，而非从零开始）

---

## 十、相关插件与服务定义

### 10.1 核心服务

| 服务 | ctx 键 | 包 | 角色 |
|---|---|---|---|
| `SessionStore` | `ctx.sessions` | `dsh-session` | 内存会话存储 |
| `SessionPersistence` | `ctx.sessionPersistence` | `dsh-session-persistence` | 持久化服务定义 |
| `CompactionEngine` | `ctx.compaction` | `dsh-compaction` | 压缩服务定义 |
| `BasicCompactionEngine` | `ctx.compaction` | `dsh-compaction-basic` | 压缩服务提供者 |
| `ToolResultPruner` | `ctx.toolResultPruner` | `dsh-compaction-tool-result-pruner` | 工具结果裁剪 |
| `SpillStore` | `ctx.spillStore` | `dsh-spill` | 溢出存储服务定义 |
| `SessionProjectionRegistry` | `ctx.sessionProjections` | `dsh-session-projection` | 投影注册表 |
| `SessionTitleService` | `ctx.sessionTitle` | `dsh-session-title` | 会话标题服务 |
| `GoalService` | `ctx.goals` | `dsh-goal` | 目标状态服务 |
| `SkillRegistry` | `ctx.skills` | `dsh-skill` | 技能注册表 |
| `SettingsService` | `ctx.settings` | `dsh-settings` | 用户设置服务 |
| `WorkspaceRegistry` | `ctx.workspaceRegistry` | `dsh-workspace` | 工作区服务 |
| `SubagentRuntime` | `ctx.subagents` | `dsh-subagent` | 子代理运行时 |

### 10.2 第三方记忆系统接入（MCP）

DeepSeek Harness 通过 MCP 客户端桥接第三方记忆系统。示例配置在 `examples/mcp-memory/` 目录中，支持三个参考实现：

| 系统 | 传输方式 | 特点 |
|---|---|---|
| **Memorix** | stdio | 本地启发式模式，无需 LLM/embedding |
| **MCP Reference Memory** | stdio | 知识图谱（实体/关系/观察），子串匹配搜索，JSONL 存储 |
| **Engram** | stdio | Go 实现，Git 项目感知 |

接入方式非常简单——通过 `@deepseek-ai/dsh-mcp-client` 插件，记忆服务器的工具会以 `mcp__<serverName>__<tool>` 的形式注册到 `ctx.tools`，模型可以像调用原生工具一样调用记忆的读写搜索功能。

关键设计决策：
- **DSH 不提供内置的长期语义记忆**——这是有意的设计选择
- **记忆作为工具**：模型主动决定何时写入/读取记忆，而非自动注入
- **完全解耦**：DSH 不管理记忆服务器的安装、数据库、模型选择

### 10.3 Ralph 工作流中的长期记忆

`tool-ralph` 包实现了一个特殊的多轮工作流，其中 **workspace 被明确作为长期记忆**：
- 每轮启动一个全新的子代理（无对话种子）
- 子代理之间通过结构化报告传递状态
- 共享工作区文件系统作为"真实来源"和长期记忆
- 这是一种"工作记忆"模式，而非语义记忆

---

## 十一、总结

DeepSeek Harness 的记忆系统是一个**以事件溯源 Session 为核心、多层级、可扩展**的架构：

```
┌─────────────────────────────────────────┐
│  扩展层：MCP 桥接第三方语义记忆            │  ← 模型主动调用工具
├─────────────────────────────────────────┤
│  注入层：AGENTS.md / skills / goals      │  ← 动态注入上下文
├─────────────────────────────────────────┤
│  表面层：Surface（模型可见的消息）         │  ← 推导自事件日志
├─────────────────────────────────────────┤
│  压缩层：Compaction（摘要检查点）          │  ← 长期记忆的主要形式
├─────────────────────────────────────────┤
│  核心层：Session 事件日志                 │  ← 唯一真相来源
├─────────────────────────────────────────┤
│  派生层：Projection（goal/stats/title）   │  ← 结构化派生视图
├─────────────────────────────────────────┤
│  持久层：JSONL / SQLite                  │  ← 跨重启持久化
└─────────────────────────────────────────┘
```

**核心设计哲学**：
1. **事件溯源**：所有记忆都是事件，从事件推导一切
2. **结构性记忆为主**：内置机制专注于对话历史的管理
3. **语义记忆外置**：通过 MCP 接入专业的第三方记忆系统
4. **模型主动调用**：记忆作为工具，由模型决定何时读写，而非自动注入
5. **可扩展**：声明合并的事件类型和投影单元，各领域包自行扩展
