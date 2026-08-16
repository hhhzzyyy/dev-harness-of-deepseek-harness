# AGENTS.md Hierarchy System and Loading Mechanism

This document systematically organizes the complete AGENTS.md system in the DeepSeek Harness project — including its hierarchical structure, dynamic loading mechanism, content at each level, design principles, and how it guides AI to work within the project.

---

## 1. Core Concepts

### 1.1 What is AGENTS.md

`AGENTS.md` is the **standing orders** for AI Agents (and human developers). It is not ordinary project documentation, but the "work code" that AI needs to read and follow every time it enters the project.

> The DeepSeek Harness project is itself an AI Agent framework — it builds itself using the very concepts it constructs. The AGENTS.md system is the embodiment of this idea.

### 1.3 Dual Identity: Both Product Feature and Development Convention

The AGENTS.md system has a **dual identity**, which is key to understanding it:

**As a product feature**: `@deepseek-ai/dsh-agent-instructions` is an officially published npm package, included in the base bundle, and is a standard plugin of the DeepSeek Harness framework. **Any AI Agent built with Harness comes with AGENTS.md auto-loading capability by default**. Alongside session, tools, and agent-loop, it is one of the framework's core capabilities.

**As a development convention**: The AGENTS.md files at various levels in the DeepSeek Harness repository (root, packages/, docs/, examples/, etc.) are the practice of **developing the product with the product itself**. The AI Agent that develops this framework loads these AGENTS.md files while working, to guide itself in writing code, writing documentation, and running tests.

```
┌─────────────────────────────────────────────┐
│  DeepSeek Harness Framework (Product)        │
│  Includes: AGENTS.md dynamic loading         │
└──────────────┬──────────────────────────────┘
               │ built using this framework
               ▼
┌─────────────────────────────────────────────┐
│  DeepSeek Harness Project (Code Repository)  │
│  Contains: AGENTS.md at all levels           │
│            (guides the developing AI)         │
└─────────────────────────────────────────────┘
```

This **reflexivity** — the framework building itself using the concepts it constructs — is one of the most core characteristics of the DeepSeek Harness project.

### 1.2 What It Is Not

- ❌ **Not one independent Agent per folder** — always the same Agent, dynamically loading different levels of instructions
- ❌ **Not E2E-driven development** — instructions are rule constraints, not test cases
- ❌ **Not a single file** — a hierarchical network of instructions

---

## 2. Hierarchical Structure

### 2.1 Complete Hierarchy Diagram

```
User Global Level
  └── ~/.dsh/AGENTS.md              ← User-level global instructions (shared across all projects)

Project Level (loaded layer by layer from root to working directory)
  ├── AGENTS.md (root)              ← Repository-level standing orders
  │    Repository layout, command list, 20+ coding conventions, defense patterns, type safety, doc requirements
  │
  ├── packages/AGENTS.md            ← Package development-specific rules
  │    Plugin export format, service design principles, invariant checks, HMR safety, naming conventions
  │    │
  │    ├── packages/web/AGENTS.md   ← Web package-specific
  │    │    Redirect rejection policy for credentialed requests
  │    │
  │    ├── packages/client/AGENTS.md ← Client package-specific
  │    │
  │    └── packages/schedule/AGENTS.md
  │
  ├── docs/AGENTS.md                ← Documentation standards
  │    Document structure, hierarchy taxonomy, writing rules, word budgets, slop checklist
  │
  ├── examples/AGENTS.md            ← Examples directory rules
  │    e2e smoke test requirements, cordis.yml comment conventions
  │
  ├── scripts/AGENTS.md             ← Scripts directory rules
  │    Gate script calling conventions
  │
  ├── vendor/AGENTS.md              ← Vendored code rules
  │    No casual modification of vendor/src
  │
  ├── .github/AGENTS.md             ← GitHub workflow rules
  │
  ├── .agents/notes/AGENTS.md       ← Agent Notes writing rules
  │    │
  │    └── .agents/notes/implemented/AGENTS.md
  │
  ├── website/AGENTS.md             ← Documentation website rules
  │
  └── native/landlock-run/AGENTS.md ← Native module rules
```

### 2.2 Hierarchy Design Principle: Supplement, Don't Repeat

Each subtree's AGENTS.md **only defines rules specific to that subtree**, never repeating rules already established at the root level.

Typical opening phrases:
- `packages/AGENTS.md`: *"These package-specific rules supplement the repo-wide conventions."*
- `packages/web/AGENTS.md`: *"These rules supplement the package conventions in packages/AGENTS.md."*

This forms an **inheritance-based rule system**:
- AI working at root → only reads root AGENTS.md
- AI working in `packages/web/` → root + packages/ + packages/web/ three layers stacked
- The deeper you go, the more specific and domain-targeted the rules become

---

## 3. Dynamic Loading Mechanism

### 3.1 Responsible Package

The `@deepseek-ai/dsh-agent-instructions` package implements the complete AGENTS.md auto-loading mechanism. It is a plugin that dynamically injects instructions during Agent runtime.

### 3.2 Baseline Loading (At Startup)

At the first `agent/pre-step` of an Agent session, baseline instructions are assembled:

```
Loading order (broad to narrow):

  1. $DSH_HOME/AGENTS.md           ← User global
  2. Project root/AGENTS.md        ← Repository root
  3. Project root/CLAUDE.md        ← (only loaded if different after content dedup)
  4. packages/AGENTS.md
  5. packages/CLAUDE.md
  6. packages/web/AGENTS.md
  7. packages/web/CLAUDE.md
     ...
  N. Current working directory (cwd) AGENTS.md
```

**Same-directory dedup rule**: If `AGENTS.md` and `CLAUDE.md` in the same directory have identical byte content (after trimming leading/trailing whitespace), only the earlier candidate is loaded (by default `AGENTS.md` takes priority over `CLAUDE.md`).

This is why `CLAUDE.md` in the project is a symlink pointing to `AGENTS.md` — identical content, automatically deduplicated, no double-loading.

### 3.3 Baseline Injection Format

Baseline instructions are wrapped into a persistent user-role message and injected into the conversation history:

```markdown
<system-reminder>
The following workspace instructions may be relevant to your work. Use them as guidance when applicable. More specific instructions take precedence over broader ones. They do not override system, developer, or direct user instructions.

Instructions from: ~/.dsh/AGENTS.md

<user global instruction content>

Instructions from: AGENTS.md

<project root instruction content>

Instructions from: packages/AGENTS.md

<package development rule content>
</system-reminder>
```

Key points:
- **Ordered broad-to-narrow**: global first, most specific last
- **Explicit priority declaration**: "More specific instructions take precedence over broader ones."
- **Does not override system/developer/user direct instructions**: just guidance, not the highest authority

### 3.4 Dynamic Discovery (During Work)

As the Agent works, when it reaches deeper directories through filesystem tools, new instructions are **dynamically appended**.

**Trigger conditions**: After successful `read`, `write`, `edit` calls

**Detection logic**:
1. Check newly reached descendant directories
2. Check each previously loaded scope for changes
3. Newly appeared files → append "Additional instructions"
4. Changed files → append "Updated instructions"
5. Disappeared files → append "Instructions removed"

**Append format**:

```markdown
<system-reminder>
Additional instructions from: packages/web/AGENTS.md

These instructions apply to work under `packages/web`. Use them as guidance when relevant; more specific instructions take precedence. They do not override system, developer, or direct user instructions.

<Web package-specific rule content>
</system-reminder>
```

### 3.5 What Does NOT Trigger

- ❌ **shell `cd` does not trigger**: bash is a new shell each call, parsing arbitrary shell syntax is unreliable
- ❌ **No file watcher**: external edits are not discovered in real-time, must wait for next file operation
- ❌ **Not loaded just by entering a directory**: must be reached through structured filesystem tools

### 3.6 Session Resume

When resuming a session:
- If the visible baseline is compatible (discovery rules, priorities, project root, budget all unchanged) → reuse existing baseline message
- If incompatible → append a complete replacement baseline
- File changes during offline period (additions/edits/deletions) → append corresponding transition messages on resume

### 3.7 Reconstruction After Compaction

When compaction pushes baseline messages out of the visible surface, the next incoming `pre-step` will:
1. Recompose the current baseline
2. Record it in the same request
3. Ensure the Agent always has complete instruction context

---

## 4. Budget Control

### 4.1 Why Budget Is Needed

AGENTS.md instructions consume token budget. If there are too many layers or content is too long, it squeezes space for actual work. Hence strict size limits.

### 4.2 Budget Configuration

| Config Item | Default | Description |
|---|---|---|
| `maxBytes` | CLI default 65,536 bytes | Total byte limit after rendering |
| `maxSourceBytes` | 1 MiB | Single source file read limit |
| `instructionFileCandidates` | `['AGENTS.md', 'CLAUDE.md']` | Candidate filenames (in priority order) |
| `localInstructionFileCandidates` | `['AGENTS.local.md', 'CLAUDE.local.md']` | Local overlay filenames |

### 4.3 Over-Budget Strategy: Keep Specific, Drop Broad

When over budget, **prefer dropping broader files, only truncate the most specific file last**:

```
Drop order when budget insufficient (drop least important first):

  1. ~/.dsh/AGENTS.md          ← dropped first (broadest)
  2. Project root/AGENTS.md
  3. packages/AGENTS.md
  4. packages/web/AGENTS.md
     ...
  N. Most specific directory AGENTS.md  ← truncated last (most relevant)
```

When omitted or truncated, it is clearly marked in the instructions:
> *"Workspace instruction budget ..."* — lists omitted and truncated paths

### 4.4 Content Dedup and Caching

- **Same-directory dedup**: Based on content (SHA-1 over trimmed content), files with identical bytes are loaded only once
- **Cross-session caching**: Each scope caches `{ path, version, digest, trimmedDigest }`, no re-read if version and content unchanged
- **Change detection**: Only bounded read + SHA-1 confirmation when version changes

---

## 5. Detailed Content of Each AGENTS.md Level

### 5.1 Root AGENTS.md (~25 rules)

**Position**: Repository-level standing orders, rules AI needs every session

**Core content**:
- **Repository layout**: what each directory does, package grouping rules
- **Command list**: all available build, test, check commands
- **Coding conventions** (20+ rules):
  - Registrations are effects
  - Runtime invariants assert owned relationships
  - Typed events use declaration merging
  - Switch on discriminant tags
  - Waterfall listeners must call `next()`
  - Model-visible ⟺ recorded
  - Plugins, not loop changes
  - Capability seams include service definition/provider/consumer
  - Prefer maintained dependencies over hand-rolled
  - Explicit over implicit at package boundaries
  - No hardcoded tunables in plugins
  - Misconfiguration fails loudly
  - Opaque cross-boundary IDs are branded
  - Trust TypeScript at typed in-process boundaries
  - Source plane vs artifact plane, don't mix
  - Keep compiler surface explicit
  - Empty catch names what it swallows
  - Don't comment the obvious in code
  - Parallel values prefer symmetry
  - Tests describe behavior, not correctness
  - Non-trivial changes must include an Agent Note
  - Tool UI rendering intent is part of design
  - Plan unit, e2e, and snapshot coverage
  - Choose PR history deliberately
  - Label conventions
  - TODO tag grading
- **Defense patterns**: lifecycle, concurrency, subprocesses, disposal
- **Type safety and documentation**: strict TypeScript + JSDoc requirements
- **Vendoring policy**

### 5.2 packages/AGENTS.md (~18 rules)

**Position**: Package development-specific rules, supplementing root conventions

**Core content**:
- **Plugin export rules**: service packages default-export service classes; function plugins named-export `name`/`inject`/`Config`/`apply`
- **Optional services use `ctx.get(name)`**: reserve `ctx.<name>` for declared injections
- **Product-visible plugins need real composition tests**: manual `ctx.plugin(...)` suites are not enough
- **Initiator-owned private chains**: derive first, then capture
- **One async operation represented by one lifecycle controller**
- **Design service definitions for all current consumers**: don't let one consumer dictate the service contract
- **Require current owners and demand**: every abstraction must have current consumers
- **Public choices need evidence**: configurability doesn't justify unsupported defaults
- **Write model-facing contracts from the model's perspective**: prompts, tool schemas contain only task-relevant concepts
- **Enforce decisions in the operation that makes them**: schema omission, prompt filtering are not enforcement
- **Publish state only at commit points**: emit notifications and update derived state after operation succeeds
- **Apply boundaries to complete results**: enforce limits where complete values are known
- **Registry contributions prove disposable**: pass HMR safety tests
- **Each package owns `./invariant`**: registration manifest name, checks event/data relationships
- **Naming rules**: tsconfig, types.ts, test locations, README format

### 5.3 docs/AGENTS.md (Documentation Standards)

**Position**: Document structure, writing rules, word budgets

**Core content**:
- **Document structure**: tutorial vs reference classification
- **Hierarchy taxonomy** (each fact has one home):
  - Root AGENTS.md → standing orders
  - Subtree AGENTS.md → subtree-specific instructions
  - architecture.md → ordered map
  - subsystems/ → subsystem reference pages
  - Agent Notes → decision records
  - postmortem/ → incident stories
  - cookbook/ → step-by-step operation guides
  - user/ → product user-facing guides
  - Package README → each package's contract
  - development.md → contributor setup
- **Writing rules**:
  - Record current state, not change history
  - One physical line per paragraph
  - Fenced ts blocks must compile
  - Comments and JSDoc state complete contracts, not reasoning
  - Write directly: name actors and facts
- **Word budgets**:
  - Root AGENTS.md ≤ 1,600 words
  - architecture.md ≤ 1,800 words
  - Subtree AGENTS.md ≤ 600 words
- **Slop checklist**: remove redundancy, historical narration, status markers, directory restatement, reasoning transcription, etc.

### 5.4 Other Subtree AGENTS.md

| Subtree | Core Rules | Rule Count |
|---|---|---|
| **examples/** | e2e smoke test requirements (keyless + with key), cordis.yml comment conventions | ~5 |
| **scripts/** | Gate script calling conventions (shell-free, path normalization) | 1 |
| **vendor/** | No casual modification of vendor/src, modifications must be recorded | 1 core prohibition |
| **packages/web/** | Redirect rejection policy for credentialed requests | 1 |
| **.agents/notes/** | Agent Note staleness checks, archiving policy | ~3 |

---

## 6. CLAUDE.md Symlink Mechanism

### 6.1 Why CLAUDE.md Exists

Different AI coding tools use different convention filenames:
- **Codex** → natively uses `AGENTS.md`
- **Claude Code** → uses `CLAUDE.md`
- **opencode** → supports both

DeepSeek Harness needs **cross-tool compatibility**, so both names are supported.

### 6.2 Purpose of Symlinks

In every directory that has an AGENTS.md, there is also a `CLAUDE.md` file with just one line:

```
AGENTS.md
```

It is a **symbolic link** (or content-equivalent file) pointing to the same directory's `AGENTS.md`.

Purpose:
1. **Cross-tool compatibility**: whether the tool looks for `AGENTS.md` or `CLAUDE.md`, it finds the rules
2. **Automatic content dedup**: because content is identical, it gets deduplicated during loading, no double-loading
3. **Single source of truth**: only need to edit `AGENTS.md`, `CLAUDE.md` syncs automatically

### 6.3 Candidate Priority

Default candidate order: `['AGENTS.md', 'CLAUDE.md']`

When both files exist in the same directory with different content, `AGENTS.md` takes priority (listed first). When content is identical, only one is loaded.

---

## 7. Local Overlay

### 7.1 What Is Local Overlay

In addition to the base `AGENTS.md` / `CLAUDE.md`, each directory can also have local overlay files:

- `AGENTS.local.md`
- `CLAUDE.local.md`

### 7.2 Loading Rules

- Local overlays are **loaded after base files** (higher priority)
- Also follow same-directory content dedup
- Empty list disables overlays
- User global level (`$DSH_HOME`) has no local overlay

### 7.3 Use Cases

- Personal work habit private rules (not committed to repo)
- Temporary debugging instructions
- Machine-specific configuration

---

## 8. Relationship with Skills

### 8.1 Differences

| Dimension | AGENTS.md | Skills (`.agents/skills/`) |
|---|---|---|
| **Nature** | Standing rules (what you must know) | Workflow procedures (how to do a task) |
| **Loading method** | Auto-injected into conversation history | Explicitly invoked via Skill tool |
| **Scope** | All work under the corresponding directory | Specific task scenarios |
| **Format** | Markdown rule list | SKILL.md + optional agents/ config |
| **Quantity** | One per level, ~15 total | ~11 specialized skills |

### 8.2 How They Work Together

AGENTS.md tells AI "what rules to follow when working in this directory", while Skills tell AI "what process to follow for a specific task". Together they form a complete guidance system:

- `docs/AGENTS.md` defines document structure and word budgets
- `dsh-doc-standards` Skill provides specific document placement and verification workflows
- `dsh-prose-standard` Skill provides detailed prose review standards

---

## 9. Design Philosophy

### 9.1 Proximity Principle

AI reads whichever level of AGENTS.md corresponds to the directory it's working in. The closer to the work area, the more specific the rules.

### 9.2 Concise and Controllable

Word budgets ensure rules don't bloat, AI can quickly digest them. When exceeded, prefer repositioning content rather than casually raising the limit.

### 9.3 Dynamic Adaptation

The Agent doesn't get all rules at once, but progressively gains more specific guidance as it works — just like human developers, who only need to learn detailed rules of a module when diving into it.

### 9.4 Clear Priority

"More specific instructions take precedence over broader ones." — specific rules override broad rules, but neither overrides system/developer/user direct instructions.

### 9.5 Trust Boundary

Symlinked instruction files are followed, meaning cloned repos can load file content outside the tree. But this content is always **low-authority workspace guidance**, never overriding system instructions. When needed, it can be restricted via filesystem policy gates or OS sandboxing.

---

## 10. Summary

The AGENTS.md system is one of the core infrastructure pieces of the AI development paradigm in the DeepSeek Harness project. It is not simply a "rule file", but a **dynamic, hierarchical instruction system that联动 with work depth**.

```
┌──────────────────────────────────────────────────────────┐
│                    Same Agent Session                     │
│                                                            │
│  Startup: load baseline instructions layer by layer        │
│           from root to cwd                                 │
│  Working: dynamically append more specific rules           │
│           when reaching deeper directories                 │
│  Resume:  check baseline compatibility,                    │
│           incrementally update offline changes             │
│  After compaction: rebuild baseline,                       │
│                     ensure complete instruction context     │
│                                                            │
│  Priority: specific > broad  (but both below               │
│            system/developer/user direct instructions)      │
│  Budget control: keep specific, drop broad;                │
│                  most relevant rules cut last              │
│                                                            │
│  Cross-tool compatible: AGENTS.md + CLAUDE.md              │
│  (symlink + content dedup)                                 │
└──────────────────────────────────────────────────────────┘
```

**One-sentence summary**: The AGENTS.md system is an **onion-style dynamic instruction layer** — the deeper AI goes into the project, the more precise and specific guidance it receives.
