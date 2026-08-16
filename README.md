# DeepSeek Harness 项目探索

本仓库用于探索 [DeepSeek Harness](deepseek-harness/) 项目——一个基于插件架构的 AI Agent 运行时框架。

## 文档索引

| 文档 | 内容 |
|---|---|
| [DeepSeek Harness 架构分析](deepseek-harness-harness-architecture.md) | Harness 产品本身的架构设计、核心子系统、插件机制 |
| [AGENTS.md 层级体系与加载机制（中文）](deepseek-harness-agents-system-zh.md) | AGENTS.md 的层级结构、动态加载机制、各层内容详解 |
| [AGENTS.md Hierarchy System (English)](deepseek-harness-agents-system-en.md) | English version of AGENTS.md hierarchy and loading mechanism |
| [记忆机制（中文）](deepseek-harness-memory-system-zh.md) | Session 事件溯源、压缩、持久化、MCP 语义记忆等完整记忆体系 |
| [Memory System (English)](deepseek-harness-memory-system-en.md) | English version of memory system architecture |
| [AI 开发约定与方法论](#ai-开发约定与方法论) | 项目作为 AI 开发产物的开发流程、规范与方法论 |

---

## AI 开发约定与方法论

### 核心结论：不是 End-to-End 驱动开发

DeepSeek Harness 项目**不是**传统意义上的 End-to-End（端到端）驱动开发。它采用的是一种**高度规范化的、以 Agent 为中心的、多层验证驱动的开发方法论**——可以称之为 **"Agent-Native 开发范式"** 或 **"快照驱动 + 决策记录 + 多层门禁"** 的开发模式。

---

### 一、开发流程的核心支柱

#### 1. AGENTS.md — AI 开发者的"宪法"

项目根目录的 `AGENTS.md` 是给 AI Agent（以及人类开发者）的**常设指令（standing orders）**，相当于开发工作的"宪法"。它规定了：

- **仓库布局**：每个目录做什么，包的分组规则
- **命令清单**：所有可用的构建、测试、检查命令
- **编码约定**：20+ 条具体的编码规则
- **防御模式**：生命周期、并发、子进程、销毁等方面的注意事项
- **类型安全与文档**：严格的 TypeScript 规范和文档要求

> 关键特征：AI Agent 每次打开项目都要读取这些规则，确保所有工作在统一的规范下进行。

#### 2. Agent Notes — 决策记录系统

`.agents/notes/` 目录下的 **Agent Notes** 是这个项目最独特的开发约定之一。它们本质上是**由 AI 撰写的 RFC/ADR（架构决策记录）**。

##### 生命周期

```
proposed/    — 提案，在实现前评审
implemented/ — 已实施，记录实际交付的决策（保持与代码同步更新）
rejected/    — 被否决的提案（保留以防重蹈覆辙）
archived/    — 已归档的冻结历史快照
```

##### 分类

| 类别 | 内容 |
|---|---|
| `feature` | 新的用户或模型可见能力 |
| `bug-fix` | 修复缺陷或弥补事后分析暴露的差距 |
| `simplification` | 移除代码、行为或表面积 |
| `architecture` | 关于已交付源码的结构性决策 |
| `process` | 代码周围的工具、策略或工作流 |
| `testing` | 测试基础设施和策略 |

##### 格式要求

每个 Agent Note 都有严格的格式：
- **Header**：标题 + 状态（proposed/implemented/rejected）
- **Problem**：动机，独立于解决方案存在
- **Decision / Proposal**：决策内容或提案
- **Alternatives considered**：**强制要求**——每个决策必须记录考虑过的替代方案及为什么放弃
- **Consequences**：权衡的代价和收益

> **核心规则**：每个非平凡的变更必须在同一个 PR 中添加或更新至少一个 Agent Note。只有纯机械的或局部的编辑可以豁免。

这意味着：**每一个重要决策都有明确的"为什么"记录在案**，AI 和人类开发者都可以追溯决策的来龙去脉。

---

### 二、测试方法论：多层验证体系

项目的测试策略不是简单的 E2E 驱动，而是**五层验证体系**，每一层有明确的职责和准入条件。

#### 测试层级

| 层级 | 命令 | 职责 | 特点 |
|---|---|---|---|
| **单元测试** | `pnpm run test` | 包级别的 Vitest 测试 | 测试与被测代码放在一起 |
| **覆盖率门禁** | `pnpm run test:coverage` | `packages/*/*/src` 逐文件 100% 覆盖率 | 未覆盖行通常是应删除的死代码 |
| **真实 API e2e** | `pnpm run test:e2e` | 针对真实 DeepSeek API 的测试 | 无 key 时自动跳过 |
| **快照测试** | `pnpm run test:snapshot` | 无 key 的预期输出对比 | 重放录制的会话，对比输出 |
| **Web 浏览器快照** | `pnpm run test:web` | Chromium 浏览器渲染对比 | Linux PR 门禁 |

#### 关键测试理念

##### 1. "宁用真实实现，不用 mock"

> Mock 只用于昂贵或非确定性的边界（LLM 适配器、网络、时钟）；下游的一切都保持真实。

这意味着测试尽可能接近真实运行环境，而不是构建大量模拟对象。

##### 2. "验证外部世界，不验证自我报告"

> e2e 断言要重新运行命令或从外部重读文件；对 agent 自己输出的关键词探测等于让作弊的 agent 通过。

测试关注**外部可观察的结果**，而不是内部状态或 agent 自己的报告。

##### 3. "测试真实入口路径"

> 产品可见的插件需要一个非单元的**真实组合测试**。手工 `ctx.plugin(...)` 套件是不够的。

测试必须通过真实的 Loader、真实的配置文件、真实的 bin 入口来运行，而不是手工拼装组件。

##### 4. "快照测试是必需的"

> 每个非平凡的模型、协议或人类可见的变更，必须在同一个 PR 中通过可运行示例的快照套件添加或更新一个无 key 场景。

**快照测试是这个项目最核心的验证手段**。它不是传统的单元测试驱动开发（TDD），也不是 E2E 驱动开发，而是：

**快照驱动开发（Snapshot-Driven Development）**

开发流程大致是：
1. 实现功能
2. 用真实模型录制一次运行（生成预期快照）
3. 后续无 key 重放验证输出一致性
4. 行为变化时更新快照并审查差异

##### 5. "测试描述行为，不描述正确性"

> 随着行为改变，过时的测试也要一起改；在 PR 中解释为什么。

这与传统 TDD 的"测试先于实现"不同——测试是行为的描述，行为变化时测试同步变化。

---

### 三、AI 技能系统（Skills）

`.agents/skills/` 目录下的技能是 AI 开发者的**专业工作流**。每个 Skill 都是一个可复用的、专门化的决策标准或操作流程。

#### 现有技能

| 技能 | 用途 |
|---|---|
| `dsh-pre-push-checks` | 推送前选择最小相关测试集 |
| `dsh-code-review` | PR 代码审查指南 |
| `dsh-doc-standards` | 文档放置和验证标准 |
| `dsh-prose-standard` | 注释、文档、提示词的文案标准 |
| `dsh-translate-docs` | 文档翻译工作流 |
| `dsh-archive-agent-notes` | Agent Note 归档工作流 |
| `dsh-find-simplifications` | 发现可简化的代码 |
| `dsh-merging-stacked-prs` | 合并堆叠 PR 的流程 |
| `dsh-trim-cot-leakage` | 检查思维链泄露 |
| `dsh-doc-site-sync` | 文档网站同步 |
| `record-browser-gif` | 录制浏览器 GIF 演示 |

#### 推送前检查技能的方法论

`dsh-pre-push-checks` 技能体现了项目的核心开发哲学：

1. **先评估变更范围**：用 `change-scope` 命令精确知道改了什么
2. **选择最小相关证据**：不跑全量测试，只跑与变更表面相关的检查
   - 包/脚本行为 → 对应 Vitest 文件
   - 文档 → `doc-sync`
   - 模型/UI 输出 → 对应的快照测试
   - 构建配置 → build + hygiene
   - 真实提供者行为 → 有 key 时的 e2e 测试
3. **CI 拥有全面覆盖**：本地只做最小必要验证，CI 跑完整矩阵

> 这不是"测试驱动"，而是**"证据驱动"**——每个变更需要有对应的证据证明它是正确的。

---

### 四、代码审查标准

`dsh-code-review` 技能定义了审查的标准，体现了项目的质量观：

#### 阻塞性要求

1. **新文本接受语义审查**：每个新增或修改的 Markdown、JSDoc、注释、提示词、可见字符串都要经过语义审查
2. **文档与代码匹配**：配置、默认值、错误、线字段、事件、公共行为在同一个 diff 中更新包 README 和 JSDoc
3. **核心类型文档匹配**：脊柱或接缝词汇的变化更新相应的子系统页面
4. **注册可清理**：每个新的注册表贡献通过处置测试
5. **不变量伴随是语义的**：检查权威事件流或可变数据，而不是服务/方法存在性
6. **必需证据存在**：作者运行了相关的本地检查

#### 手动检查清单

- 意图和接口契约
- 生命周期和并发性
- 能力与消费者适配
- 作用域、所有权和必要性
- 配置和公共选择
- 模型视角
- 强制执行路径
- 借用和派生状态
- 边界覆盖最终操作
- 真实入口路径
- 测试强度
- 不变量生命周期和负控制
- 已实现的 Agent Notes 与交付现实匹配
- 转录变更
- 双语变更

---

### 五、事后分析（Post-mortems）

`docs/postmortem/` 目录记录了已经发生的线上/集成事故。每个事后分析关注的是：

> **为什么我们的流程让它通过了，而不仅仅是一行修复。**

事后分析的写作标准：
- 以**执行摘要**开头（30 秒内能读完）
- 记录故障机制、为什么每个安全网都漏掉了
- 添加具体的护栏（测试、AGENTS.md 规则、ADR）确保同类故障下次会大声失败

---

### 六、文档纪律

项目对文档有极其严格的要求，体现在 `docs/AGENTS.md` 中：

#### 层级分类法：每个事实一个家

| 层级 | 职责 |
|---|---|
| 根 `AGENTS.md` | 常设指令：agent 每个会话都需要的规则 |
| 子树 `AGENTS.md` | 特定子树的指令 |
| `architecture.md` | 有序地图：组合、核心包、循环、接缝、扩展点 |
| `subsystems/` | 每个子系统的参考页：类型定义、语义、生成的 Cordis API |
| Agent Notes | 活跃的决策记录：为什么、放弃了什么、需要的验证 |
| `postmortem/` | 事故故事 |
| `cookbook/` | 分步操作指南 |
| `user/` | 产品面向用户的指南 |
| 包 README | 每个包的契约 |
| `development.md` | 贡献者设置、日常工作流、CI 摘要 |

#### 写作规则

- **记录当前状态，不记录变更历史**
- **每个非平凡变更都包含至少一个 Agent Note**
- **每段一个物理行**（`verify-md-wrap` 检查）
- **注释和 JSDoc 陈述完整的契约，不陈述推理过程**
- **直接、具体的术语**，不用比喻
- **机械可检查的不变量要接入执行的顶层门禁**

#### 字数预算

文档有严格的字数上限（由 `verify-doc-budgets` 执行）：
- 根 `AGENTS.md` ≤ 1,600 词
- `architecture.md` ≤ 1,800 词
- 子树 `AGENTS.md` ≤ 600 词

超出时必须：重新定位内容、压缩、或提升上限（并在 PR 中说明理由）。

---

### 七、CI/CD 与质量门禁

#### CI 工作流

- **ci.yml**：无 key 的主 CI，分组为多个独立通道
- **e2e.yml**：真实 API 端到端测试
- **sandbox.yml**：沙箱测试
- **e2b-e2e.yml**：E2B 相关 e2e
- **release.yml**：发布流程

#### 质量门禁清单

```
pnpm run test:coverage    # 逐文件 100% 覆盖率
pnpm run test:snapshot    # 无 key 快照测试
pnpm run typecheck        # 类型检查
pnpm run lint             # 代码检查
pnpm run duplication      # 跨文件 TypeScript 克隆检测
pnpm run build            # 构建
pnpm run hygiene          # knip + publint + 工作区约束 + NodeNext 消费者检查
pnpm run doc-sync         # 所有文档门禁
pnpm run website:build    # VitePress 构建（兼作死链检查）
```

---

### 八、与传统开发范式的对比

#### 不是什么

- **不是 TDD（测试驱动开发）**：测试描述行为，行为变化时测试一起改；不要求先写测试
- **不是 E2E 驱动开发**：E2E 只是五层中的一层，且有 key 时才运行
- **不是敏捷/Scrum**：没有迭代、故事点、站会等概念
- **不是传统的 ADR**：Agent Notes 更详细、更频繁、由 AI 撰写、有生命周期流转

#### 是什么

| 特征 | 描述 |
|---|---|
| **快照驱动** | 用真实运行的输出快照作为主要验证手段 |
| **决策记录驱动** | 每个重要决策都有 Agent Note 记录"为什么" |
| **多层门禁** | 单元→覆盖率→快照→e2e→浏览器快照，层层递进 |
| **AI 原生** | 所有规范、技能、流程都是为 AI 开发者设计的 |
| **证据导向** | 每个变更需要匹配最小相关证据，不跑无关测试 |
| **文档即代码** | 文档有格式检查、字数预算、类型等价校验 |
| **防御性编程** | 明确的防御模式指南，处理生命周期/并发/子进程/销毁 |
| **事后驱动改进** | 每个事故都转化为永久的自动化护栏 |

---

### 九、总结：DeepSeek Harness 的开发约定本质

DeepSeek Harness 作为一个**由 AI 开发的 AI Agent 框架**，它的开发方法论是**自反的**——它用自己构建的理念来构建自己。

核心可以概括为：

> **"规范 + 证据 + 记录"三位一体的 AI 原生开发范式**

1. **规范（AGENTS.md + Skills）**：告诉 AI 怎么做，有明确的规则和工作流
2. **证据（多层测试 + 快照）**：证明做对了，用最小相关证据验证变更
3. **记录（Agent Notes + Postmortems）**：记录为什么这么做，每个决策可追溯

这不是 End-to-End 驱动，而是一种更精细、更适合 AI 协作的开发模式——**快照验证保证行为一致性，决策记录保证可追溯性，多层门禁保证质量底线**。
