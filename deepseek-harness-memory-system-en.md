# DeepSeek Harness Memory System

This document systematically organizes the Memory mechanism in the DeepSeek Harness project — including core data structures, storage methods, retrieval strategies, lifecycle management, and its relationship with the Session system.

---

## 1. Core Design Philosophy

### 1.1 No Standalone "memory" Package

DeepSeek Harness **does not have a standalone core package named "memory"**. Its "memory" is a **distributed, multi-level system** composed of multiple cooperating subsystems.

The core memory carrier is the **Session event log**, around which multiple layers of mechanisms are built: compaction, projection, persistence, spill handling, etc.

### 1.2 Event Sourcing

The entire memory architecture is based on the event sourcing pattern:
- **The single source of truth** is the append-only event log
- All derived state (message history, surface view, projections, statistics) is folded from the log
- Replay = re-fold, results are deterministic

### 1.3 Structural Memory First, Semantic Memory as Extension

Unlike traditional "vector database + semantic retrieval" memory systems, DeepSeek Harness adopts:
- **Built-in memory**: Focused entirely on structural management of conversation history (storage, compression, persistence)
- **Semantic memory**: Handled by specialized external systems through the MCP protocol (model actively calls tools)

This is an **intentional design choice** — the framework does not provide built-in long-term semantic memory, but instead integrates third-party systems through standardized interfaces.

---

## 2. Core Data Structures

### 2.1 SessionEvent — The Atomic Unit of Memory

The most basic unit of memory is `SessionEvent`. It is an append-only log entry for event sourcing:

```typescript
type SessionEvent<T extends SessionEventType = SessionEventType> = {
  type: K                    // event type
  seq: number                // monotonically increasing sequence number
  time: number               // Unix timestamp (ms)
  data: SessionEventMap[K]   // event payload
  ignorable?: true           // safe to skip if type unknown
  // surface events only
  sourceEventSeqs?: number[] // source event sequence numbers (for provenance)
  surfaceOp?: SurfaceOp      // surface operation: 'append' or replace
}
```

`SessionEventMap` is a **declaration-merging extensible** interface — each domain package registers its own event types into it.

Core event types include:
- `turn/start`, `turn/end` — turn boundaries
- `step/start`, `step/end` — step boundaries (one model call + tool execution)
- `user/message` — user message (surface event)
- `assistant/chunk` — streaming output chunks (raw token-level replay)
- `assistant/message` — assistant message (surface event)
- `tool/call`, `tool/result` — tool calls and results (latter is surface event)
- `todo/write` — todo list snapshot
- `request/header`, `request/context` — request header snapshots
- `session/end-seed` — seed history boundary marker

### 2.2 Surface — The Model-Visible Memory Surface

Not all events are visible to the model. `SurfaceEventType` defines three types of events that enter the model's context:
- `user/message`
- `assistant/message`
- `tool/result`

`SurfaceOp` supports two operations:
- `'append'` — append to the tail (normal path)
- `{ op: 'replace', start, end }` — replace a range (used by compaction)

The `SessionSurface` interface maintains the current surface view:
```typescript
interface SessionSurface {
  readonly nodes: readonly number[]   // current surface event seq list
  readonly replaceGeneration: number  // number of replacements that have occurred
}
```

### 2.3 SessionHeader — Session Metadata

```typescript
interface SessionHeader {
  readonly version: number           // log format version
  readonly id: SessionId             // session ID (branded type)
  readonly createdAt: number         // creation time
  readonly cwd?: string              // working directory
  readonly parentSession?: SessionId // parent session (fork/subagent)
  readonly seedLength?: number       // seed history length
  readonly origin?: 'subagent'       // origin type
  readonly delegationDepth?: number  // delegation depth
  readonly agentPreset?: string      // agent preset
}
```

### 2.4 Compaction-Related Types

Compaction is the core mechanism for long-term memory, extending SessionEventMap via declaration merging:
- `compaction/start` — start compaction (acquire lock)
- `compaction/summary` — summary result (range, token count, model info)
- `compaction/end` — end compaction (release lock)
- `compaction/prune` — tool result pruning

### 2.5 Projection — Derived Memory Views

`SessionProjectionMap` is a declaration-merging type table where each domain package registers its own projection units. Each projection unit is a pure functional fold unit:

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

For example:
- The `goal` package registers `goal: GoalProjection | null`
- The `subagent` package registers `subagent` and `subagentTiming`

---

## 3. Memory Storage Methods

### 3.1 In-Memory Storage

`SessionStore` (`ctx.sessions`) is the in-memory session store. All active sessions reside in memory. The event log is an append-only array, and the surface view is maintained via incremental folding.

### 3.2 Persistent Storage

Persistence is implemented through the `session-persistence` capability seam, with two backends:

#### JSONL Backend (`session-persistence-jsonl`)
- **Format**: One file per session, first line is header JSON, each subsequent line is one `SessionEvent` JSON
- **Compression**: Default uses Zstandard frame compression (with checksum)
- **Packing**: Consecutive `assistant/chunk` events are packed into `text-chunks`/`reasoning-chunks`/`tool-call-chunks` lines (lossless, ~60% reduction)
- **Directory structure**: `root/<project-dir>/<session-id>/session.jsonl.zst`

#### SQLite Backend (`session-persistence-sqlite`)
- **Two core tables**:
  - `sessions` — session metadata (id, version, created_at, cwd, parent_session, etc.)
  - `events` — event table (session_id, seq, type, time, data(JSON), etc.)
- **Foreign key cascade delete**: deleting a session cascades to delete events
- **Journal mode**: WAL by default
- **Application ID**: `0x44534850` ("DSHP"), prevents accidentally operating on other databases

### 3.3 Persistence Coordinator

Both backends share the same write coordinator, responsible for:
- **Batch writes**: `writeBatchMaxDelayMs` coalescing window, events enter buffer first then batch write
- **Crash recovery**: on cold load, detect torn tail, truncate and synthesize `turn/end {interrupted}` to close interrupted turns
- **Lazy materialization**: don't write file immediately on session creation, materialize only on first append
- **LRU cache**: preparation result cache for cold sessions

### 3.4 Projection Cache (`session-projection-cache`)

Projection unit states are also persistently cached:
- Each row: `(sessionId, key) → { ver, seq, val }`
- `ver` is the unit's `stateVersion`, if mismatched the cache is discarded and recomputed
- Write timing: `turn/end`, session dispose, and configurable throttling

### 3.5 Spill Storage

Oversized tool results are spilled to files via the `spill` system:
- **Backend**: `spill-local` uses session-scoped private files
- **Result**: returns `SpillRef { locator, bytes, retrievalHint }`
- Model sees locator + retrieval hint, needs to actively read full content

---

## 4. Memory Retrieval Mechanisms

### 4.1 Surface Derivation (`deriveMessages`)

The message history the model sees is derived from the event log via `deriveEventMessage()` + `foldSurface()`. This is a **pure function**: given the event log, the message list is deterministically obtained.

Key rules:
- Only surface events (user/message, assistant/message, tool/result) produce messages
- Replace operations (compaction) shadow old messages, replacing them with summaries
- Empty-content assistant/message are skipped (only carry usage statistics)

### 4.2 Compaction Retrieval — Summary-Based Memory

Compaction is the most core "memory retrieval" mechanism. When history gets too long, old conversations are **summarized into a checkpoint**, and subsequent requests only see the summary + recent history.

Retrieval strategy:
- **Token-pressure-based**: `ctx.tokenMeter` measures the estimated token count of the current request
- **Threshold trigger**: triggers when exceeding `thresholdRatio × contextWindow` (default 0.8)
- **Retain tail**: retains the most recent `retainRatio × contextWindow` (default 0.16) as recent context
- **Balanced boundaries**: compaction boundaries must fall on complete tool call-result pairs, cannot cut unanswered tool calls

### 4.3 No Semantic Retrieval / Vector Search

**DeepSeek Harness itself does not provide semantic similarity search or vector retrieval**. Memory retrieval is purely structural:
- Chronological message history
- Summary checkpoint replacement
- Tool result pruning (head + tail)

Semantic memory (fact memory, preference memory) is integrated via **third-party MCP memory servers**.

### 4.4 Projection Retrieval

The `session-projection` system provides structured derived view retrieval:
- `ctx.sessionProjections.snapshot(session)` — synchronously get current values of all registered units
- `ctx.sessionProjections.onChanged(listener)` — subscribe to changes
- Each unit is a pure function fold, event-driven updates

---

## 5. Memory Write/Update Mechanisms

### 5.1 Event Append (`Session.append`)

All memory writes enter the log through `Session.append()`. This is the only write path, guaranteeing:
- Sequential sequence numbers
- Deep frozen events (immutable)
- Surface metadata validation
- JSON serializability validation

### 5.2 When Memory Is Created

| Timing | Event Type | Description |
|---|---|---|
| User input | `user/message` | User messages, injected context, goal continuation |
| Model output | `assistant/chunk`, `assistant/message` | Streaming chunks + complete messages |
| Tool calls | `tool/call`, `tool/result` | Tool call requests and results |
| Turn/step | `turn/start`, `turn/end`, `step/start`, `step/end` | Boundary markers |
| Todo updates | `todo/write` | Complete list snapshot (last-write-wins) |
| Request headers | `request/header`, `request/context` | Configuration snapshots |
| Compaction | `compaction/start/summary/end/prune` | Compaction transaction records + replacement messages |
| Goals | `goal/change` | Goal state snapshots |
| Title | `session/title` | Session title revisions |
| Subagent | `subagent/descriptor` | Subagent descriptors |
| Seed boundary | `session/end-seed` | Seed/live history boundary |

### 5.3 Compaction Write Flow

Compaction is the most complex write path:

1. **Append `compaction/start`** — synchronously acquire the log record lock
2. **Call LLM to generate summary** — replay the compressed range of conversation + compaction instructions
3. **Append `compaction/summary`** — record summary, range, token count, model info
4. **Append replacement `user/message`** — carries `surfaceOp: { op: 'replace', start, end }`, this is the only surface mutation
5. **Append `compaction/end`** — release the lock

The entire process completes under lock protection. If a crash occurs mid-process, unpaired `compaction/start` events are detected as "orphan locks".

### 5.4 Persistence Writes

- **Write-behind pattern**: memory log is appended first, then persistence backend is notified via `session/event` events
- **Batch coalescing**: events within the `writeBatchMaxDelayMs` window are merged into a single write
- **Explicit flush**: `session/flush` is a parallel-waiting persistence barrier
- **Checkpoint policy**: `session-checkpoint-policy` forces flush before model requests, before tool execution, and at step boundaries

---

## 6. Memory Usage in the Agent Loop

### 6.1 Request Assembly

At the start of each step, agent-loop assembles the model request:
1. System prompt
2. Tool schemas
3. **Message history derived from session** (`session.deriveMessages()`)
4. Current step's user message

Memory (history messages) is **directly injected into the prompt**, not as a tool.

### 6.2 Compaction Trigger Points in the Loop

Compaction hooks into agent-loop through two points:

1. **`agent/pre-step` waterfall** — check token pressure before each step, auto-compact if threshold exceeded
2. **`agent/request-error` waterfall** — when context overflow (`CONTEXT_WINDOW_EXCEEDED`) occurs, force compaction then retry

### 6.3 Impact of Memory on KV Cache

- **Append-only history**: when prefix is unchanged, provider's KV cache can be reused
- **Compaction replacement**: invalidates from the first shadowed token onward
- **System prompt / tool schema changes**: also cause cache invalidation

### 6.4 Injected Memory

Beyond conversation history, various "injected memories" enter context through `agent.inject()`:
- File change notifications
- Subdirectory AGENTS.md content
- Skill content (skill loading)
- Scheduled notifications
- Goal continuation prompts

These all enter the log and surface as `user/message` events (with `source` distinguishing origin).

---

## 7. Memory Type Classification

### 7.1 By Persistence

| Type | Carrier | Lifecycle |
|---|---|---|
| **Transient memory** | In-memory Session log | Within process, while session is active |
| **Short-term memory** | Surface message history (uncompressed portion) | Same session, until compacted |
| **Long-term memory** | Compaction summary checkpoints | Same session, persists across restarts |
| **Persistent memory** | Persisted complete event log | Cross-session, cross-process, cross-restart |

### 7.2 By Content Type

| Type | Carrier | Description |
|---|---|---|
| **Conversation memory** | Session event log | Complete interaction history |
| **Summary memory** | Compaction checkpoints | LLM-generated summaries of old conversations |
| **Goal memory** | goal/change events | Current goal state |
| **Todo memory** | todo/write events | Todo item list |
| **Skill memory** | Skill registry + loaded content | Callable instruction sets |
| **Preference memory** | Settings system | User settings and preferences |
| **Workspace memory** | Workspace system | Session grouping and working directory归属 |
| **Spill memory** | Spill files | Offline storage of oversized tool results |
| **Statistics memory** | session-stats projection | Session statistical data |
| **Title memory** | session/title events | Session title |
| **Delegation memory** | subagent/descriptor + parentSession | Parent-child session relationships and delegation depth |

### 7.3 By Visibility

| Type | Model-visible? | Description |
|---|---|---|
| **Surface memory** | Yes | user/assistant/tool messages + compaction summaries |
| **Log-only memory** | No | Boundary markers, summary raw text, goal state, titles, etc. |
| **Projection memory** | No (unless exposed via tools) | Derived views like goal, stats, title, etc. |
| **Spill memory** | Indirect | Model sees locator + retrieval hint, needs active reading |

---

## 8. Memory Lifecycle Management

### 8.1 Compaction

Compaction is the primary lifecycle management mechanism:

- **Trigger conditions**: token pressure exceeds threshold, or context overflow
- **Strategy parameters**:
  - `thresholdRatio` (default 0.8) — proportion of window at which to trigger
  - `retainRatio` (default 0.16) — proportion of recent context to retain
  - `compactionRetries` (default 1) — number of retries if still over threshold after compaction
  - `maxOverflowRetries` (default 1) — maximum retries after context overflow
- **Summary generation**: uses configured summarization model, replays conversation prefix to reuse KV cache
- **Summary structure**: fixed 8-section Markdown structure (main request, key concepts, files & code, errors & fixes, todos, current work, next steps, key context)
- **Convergence guarantee**: summary must be smaller than original, otherwise rejected

### 8.2 Tool Result Pruning

`compaction-tool-result-pruner` provides model-agnostic pruning:
- Tool results exceeding `thresholdChars` (default 8192) are pruned
- Retains `headChars` (default 4096) head + `tailChars` (default 1024) tail
- Middle marked with `[... tool result middle pruned ...]`
- Executes before compaction trigger, may avoid unnecessary LLM calls

### 8.3 Spill

Oversized outputs are moved out of context via the spill system:
- Model only sees locator and retrieval hint
- Full content stored in session-scoped files
- Model can retrieve on-demand via read tools

### 8.4 Mechanisms That Do NOT Exist

- **No TTL expiration**: event log is permanently append-only
- **No forgetting policy**: old memories are not automatically deleted
- **No archiving mechanism**: sessions are not automatically archived
- **No cross-session memory merging**: each session's memory is independent

---

## 9. Relationship Between Memory System and Session System

**Session is the core carrier of the memory system**. The entire memory architecture is built around Session.

### 9.1 Layered Architecture

```
Model-visible surface (Surface)
    ↑ derivation
Event log (SessionEvent log) — single source of truth
    ↑ append
Various event producers (loop, tools, compaction, goal...)
    ↓ subscription
Persistence (persistence), projection (projection), stats (stats), UI
```

### 9.2 Inter-Session Isolation

- Each Session has independent memory
- Parent-child sessions establish lineage through `parentSession` + `seedLength`
- Forked child sessions inherit parent's seed history, but are independent afterward
- Subagents can choose to inherit context (fork) or start from scratch (spawn)

### 9.3 Cold Start and Recovery

- Persisted sessions can be resumed via `ctx.agents.resume()`
- On recovery, all derived state is re-folded from the persisted log
- Crashed sessions are repaired: synthesize interrupted tool results + step end + turn end
- Projection cache accelerates cold start (folding starts from cache row instead of zero)

---

## 10. Related Plugins and Service Definitions

### 10.1 Core Services

| Service | ctx key | Package | Role |
|---|---|---|---|
| `SessionStore` | `ctx.sessions` | `dsh-session` | In-memory session store |
| `SessionPersistence` | `ctx.sessionPersistence` | `dsh-session-persistence` | Persistence service definition |
| `CompactionEngine` | `ctx.compaction` | `dsh-compaction` | Compaction service definition |
| `BasicCompactionEngine` | `ctx.compaction` | `dsh-compaction-basic` | Compaction service provider |
| `ToolResultPruner` | `ctx.toolResultPruner` | `dsh-compaction-tool-result-pruner` | Tool result pruning |
| `SpillStore` | `ctx.spillStore` | `dsh-spill` | Spill storage service definition |
| `SessionProjectionRegistry` | `ctx.sessionProjections` | `dsh-session-projection` | Projection registry |
| `SessionTitleService` | `ctx.sessionTitle` | `dsh-session-title` | Session title service |
| `GoalService` | `ctx.goals` | `dsh-goal` | Goal state service |
| `SkillRegistry` | `ctx.skills` | `dsh-skill` | Skill registry |
| `SettingsService` | `ctx.settings` | `dsh-settings` | User settings service |
| `WorkspaceRegistry` | `ctx.workspaceRegistry` | `dsh-workspace` | Workspace service |
| `SubagentRuntime` | `ctx.subagents` | `dsh-subagent` | Subagent runtime |

### 10.2 Third-Party Memory System Integration (MCP)

DeepSeek Harness bridges third-party memory systems through the MCP client. Example configurations are in the `examples/mcp-memory/` directory, supporting three reference implementations:

| System | Transport | Features |
|---|---|---|
| **Memorix** | stdio | Local heuristic mode, no LLM/embedding needed |
| **MCP Reference Memory** | stdio | Knowledge graph (entities/relations/observations), substring match search, JSONL storage |
| **Engram** | stdio | Go implementation, Git project-aware |

Integration is very simple — through the `@deepseek-ai/dsh-mcp-client` plugin, memory server tools are registered to `ctx.tools` in the form `mcp__<serverName>__<tool>`, and the model can call memory read/write/search functions just like native tools.

Key design decisions:
- **DSH does not provide built-in long-term semantic memory** — this is an intentional design choice
- **Memory as tools**: the model actively decides when to write/read memory, rather than automatic injection
- **Fully decoupled**: DSH does not manage memory server installation, database, or model selection

### 10.3 Long-Term Memory in the Ralph Workflow

The `tool-ralph` package implements a special multi-turn workflow where **workspace is explicitly used as long-term memory**:
- Each round starts a fresh subagent (no conversation seed)
- State is passed between subagents through structured reports
- Shared workspace filesystem serves as the "source of truth" and long-term memory
- This is a "working memory" pattern, not semantic memory

---

## 11. Summary

DeepSeek Harness's memory system is a **multi-level, extensible architecture with event-sourced Session at its core**:

```
┌─────────────────────────────────────────┐
│  Extension layer: MCP bridge to 3rd-party │  ← model actively calls tools
│  semantic memory systems                  │
├─────────────────────────────────────────┤
│  Injection layer: AGENTS.md / skills /    │  ← dynamic context injection
│  goals                                    │
├─────────────────────────────────────────┤
│  Surface layer: Surface (model-visible    │  ← derived from event log
│  messages)                                │
├─────────────────────────────────────────┤
│  Compaction layer: Compaction (summary    │  ← primary form of long-term memory
│  checkpoints)                             │
├─────────────────────────────────────────┤
│  Core layer: Session event log            │  ← single source of truth
├─────────────────────────────────────────┤
│  Derived layer: Projection (goal/stats/   │  ← structured derived views
│  title)                                   │
├─────────────────────────────────────────┤
│  Persistence layer: JSONL / SQLite        │  ← cross-restart persistence
└─────────────────────────────────────────┘
```

**Core design philosophy**:
1. **Event sourcing**: all memory is events, everything is derived from events
2. **Structural memory first**: built-in mechanisms focus on conversation history management
3. **Semantic memory externalized**: integrate specialized third-party memory systems via MCP
4. **Model-active invocation**: memory as tools, the model decides when to read/write, not automatic injection
5. **Extensible**: declaration-merging event types and projection units, each domain package extends on its own
