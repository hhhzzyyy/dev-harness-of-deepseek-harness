# AGENTS.md 层级体系与加载机制

本文档系统整理 DeepSeek Harness 项目中 AGENTS.md 的完整体系——包括层级结构、动态加载机制、各层内容、设计原则，以及它如何指导 AI 在项目中工作。

---

## 一、核心概念

### 1.1 什么是 AGENTS.md

`AGENTS.md` 是给 AI Agent（以及人类开发者）的**常设指令（standing orders）**。它不是普通的项目文档，而是 AI 每次进入项目工作时都需要读取和遵守的"工作守则"。

> DeepSeek Harness 项目本身就是一个 AI Agent 框架，它用自己构建的理念来构建自己——AGENTS.md 体系就是这一理念的体现。

### 1.3 双重定位：既是产品功能，也是开发规范

AGENTS.md 体系具有**双重身份**，这是理解它的关键：

**作为产品功能**：`@deepseek-ai/dsh-agent-instructions` 是一个正式发布的 npm 包，被包含在 base bundle（基础套件）中，是 DeepSeek Harness 框架的标准插件。**任何使用 Harness 构建的 AI Agent，默认都带有 AGENTS.md 自动加载能力**。它和 session、tools、agent-loop 一样，是框架的核心能力之一。

**作为开发规范**：DeepSeek Harness 仓库里的各级 AGENTS.md 文件（根、packages/、docs/、examples/ 等），就是**用自己的产品来开发自己**的实践。开发这个框架的 AI Agent 在工作时，会加载这些 AGENTS.md 来指导自己写代码、写文档、做测试。

```
┌─────────────────────────────────────────────┐
│  DeepSeek Harness 框架（产品）                │
│  包含功能：AGENTS.md 动态加载机制             │
└──────────────┬──────────────────────────────┘
               │ 用这个框架来开发
               ▼
┌─────────────────────────────────────────────┐
│  DeepSeek Harness 项目本身（代码仓库）         │
│  仓库里有：各级 AGENTS.md（指导开发 AI）       │
└─────────────────────────────────────────────┘
```

这种**自反性（Reflexivity）**——框架用自己构建的理念来构建自己——是 DeepSeek Harness 项目最核心的特征之一。

### 1.2 不是什么

- ❌ **不是每个文件夹一个独立 Agent**——始终是同一个 Agent，动态加载不同层级的指令
- ❌ **不是 E2E 驱动开发**——指令是规则约束，不是测试用例
- ❌ **不是单一文件**——是一个分层级的指令网络

---

## 二、层级结构

### 2.1 完整层级图

```
用户全局层
  └── ~/.dsh/AGENTS.md              ← 用户级全局指令（所有项目共享）

项目层（从根到工作目录，逐层加载）
  ├── AGENTS.md（根）                ← 仓库级常设指令
  │    仓库布局、命令清单、20+ 编码约定、防御模式、类型安全、文档要求
  │
  ├── packages/AGENTS.md             ← 包开发专属规则
  │    插件导出格式、服务设计原则、不变量检查、HMR 安全、命名规范
  │    │
  │    ├── packages/web/AGENTS.md    ← Web 包专属
  │    │    带凭证请求的重定向策略
  │    │
  │    ├── packages/client/AGENTS.md ← 客户端包专属
  │    │
  │    └── packages/schedule/AGENTS.md
  │
  ├── docs/AGENTS.md                 ← 文档标准
  │    文档结构、层级分类法、写作规则、字数预算、slop 检查
  │
  ├── examples/AGENTS.md             ← 示例目录规则
  │    e2e smoke 测试要求、cordis.yml 注释规范
  │
  ├── scripts/AGENTS.md              ← 脚本目录规则
  │    gate 脚本调用约定
  │
  ├── vendor/AGENTS.md               ← vendored 代码规则
  │    禁止随意修改 vendor/src
  │
  ├── .github/AGENTS.md              ← GitHub 工作流规则
  │
  ├── .agents/notes/AGENTS.md        ← Agent Notes 写作规则
  │    │
  │    └── .agents/notes/implemented/AGENTS.md
  │
  ├── website/AGENTS.md              ← 文档网站规则
  │
  └── native/landlock-run/AGENTS.md  ← 原生模块规则
```

### 2.2 层级设计原则：补充不重复

每个子树的 AGENTS.md **只定义该子树特有的规则**，绝不重复根级已有的规则。

典型的开头句式：
- `packages/AGENTS.md`: *"These package-specific rules supplement the repo-wide conventions."*
- `packages/web/AGENTS.md`: *"These rules supplement the package conventions in packages/AGENTS.md."*

这形成了一个**继承式的规则体系**：
- AI 在根目录工作 → 只读取根 AGENTS.md
- AI 进入 `packages/web/` 工作 → 根 + packages/ + packages/web/ 三层规则叠加
- 越往下，规则越具体、越针对特定领域

---

## 三、动态加载机制

### 3.1 负责的包

`@deepseek-ai/dsh-agent-instructions` 包实现了完整的 AGENTS.md 自动加载机制。它是一个插件，在 Agent 运行时动态注入指令。

### 3.2 基线加载（启动时）

Agent 会话的第一个 `agent/pre-step` 时，组装基线指令：

```
加载顺序（从宽到窄）：

  1. $DSH_HOME/AGENTS.md           ← 用户全局
  2. 项目根/AGENTS.md               ← 仓库根
  3. 项目根/CLAUDE.md               ← （内容去重后如果不同才加载）
  4. packages/AGENTS.md
  5. packages/CLAUDE.md
  6. packages/web/AGENTS.md
  7. packages/web/CLAUDE.md
     ...
  N. 当前工作目录（cwd）的 AGENTS.md
```

**同目录去重规则**：同一目录下的 `AGENTS.md` 和 `CLAUDE.md`，如果内容（去掉首尾空白后）字节完全相同，则只加载较早的候选者（默认 `AGENTS.md` 优先于 `CLAUDE.md`）。

这就是为什么项目中 `CLAUDE.md` 是指向 `AGENTS.md` 的符号链接——内容完全相同，自动去重，不会重复加载。

### 3.3 基线注入形式

基线指令被包装成一条持久化的 user 角色消息，注入到会话历史中：

```markdown
<system-reminder>
The following workspace instructions may be relevant to your work. Use them as guidance when applicable. More specific instructions take precedence over broader ones. They do not override system, developer, or direct user instructions.

Instructions from: ~/.dsh/AGENTS.md

<用户全局指令内容>

Instructions from: AGENTS.md

<项目根指令内容>

Instructions from: packages/AGENTS.md

<包开发规则内容>
</system-reminder>
```

关键点：
- **从宽到窄排列**：全局在前，最具体的在后
- **明确优先级声明**："More specific instructions take precedence over broader ones."
- **不覆盖系统/开发者/用户直接指令**：只是 guidance，不是最高权威

### 3.4 动态发现（工作中）

Agent 工作过程中，当通过文件系统工具触达更深的目录时，会**动态追加**新的指令。

**触发条件**：成功的 `read`、`write`、`edit` 调用后

**检测逻辑**：
1. 检查新触达的后代目录
2. 检查每个之前已加载的作用域是否有变化
3. 新出现的文件 → 追加 "Additional instructions"
4. 改变的文件 → 追加 "Updated instructions"
5. 消失的文件 → 追加 "Instructions removed"

**追加形式**：

```markdown
<system-reminder>
Additional instructions from: packages/web/AGENTS.md

These instructions apply to work under `packages/web`. Use them as guidance when relevant; more specific instructions take precedence. They do not override system, developer, or direct user instructions.

<Web 包专属规则内容>
</system-reminder>
```

### 3.5 不触发的情况

- ❌ **shell `cd` 不触发**：bash 每次调用都是新 shell，解析任意 shell 语法不可靠
- ❌ **没有文件监视器**：外部编辑不会实时发现，要等下一次文件操作
- ❌ **不是进入目录就加载**：必须通过结构化的文件系统工具触达

### 3.6 会话恢复（Resume）

恢复会话时：
- 如果可见的基线兼容（发现规则、优先级、项目根、预算都没变）→ 复用已有基线消息
- 如果不兼容 → 追加一条完整的替换基线
- 离线期间的文件变更（新增/编辑/删除）→ 在恢复时追加对应的转换消息

### 3.7 压缩（Compaction）后的重建

当压缩把基线消息挤出可见表面后，下一个进入的 `pre-step` 会：
1. 重新组合当前基线
2. 在同一个请求中记录下来
3. 保证 Agent 始终有完整的指令上下文

---

## 四、预算控制

### 4.1 为什么需要预算

AGENTS.md 指令会占用 token 预算。如果层级太多、内容太长，会挤压实际工作的空间。因此有严格的大小限制。

### 4.2 预算配置

| 配置项 | 默认值 | 说明 |
|---|---|---|
| `maxBytes` | CLI 默认 65,536 字节 | 渲染后的总字节上限 |
| `maxSourceBytes` | 1 MiB | 单个源文件读取上限 |
| `instructionFileCandidates` | `['AGENTS.md', 'CLAUDE.md']` | 候选文件名（按优先级） |
| `localInstructionFileCandidates` | `['AGENTS.local.md', 'CLAUDE.local.md']` | 本地覆盖层文件名 |

### 4.3 超预算策略：保具体、舍宽泛

超预算时，**优先丢弃更宽泛的文件，最后才截断最具体的文件**：

```
预算不足时的丢弃顺序（先丢最不重要的）：

  1. ~/.dsh/AGENTS.md          ← 最先丢（最宽泛）
  2. 项目根/AGENTS.md
  3. packages/AGENTS.md
  4. packages/web/AGENTS.md
     ...
  N. 最具体目录的 AGENTS.md     ← 最后才截断（最相关）
```

被省略或截断时，会在指令中明确标注：
> *"Workspace instruction budget ..."* —— 列出被省略和被截断的路径

### 4.4 内容去重与缓存

- **同目录去重**：基于内容（SHA-1 over trimmed content），字节相同的文件只加载一次
- **跨会话缓存**：每个作用域缓存 `{ path, version, digest, trimmedDigest }`，版本和内容都没变就不重读
- **变更检测**：版本变了才做有界读取 + SHA-1 确认

---

## 五、各层 AGENTS.md 内容详解

### 5.1 根 AGENTS.md（约 25 条规则）

**定位**：仓库级常设指令，AI 每个会话都需要的规则

**核心内容**：
- **仓库布局**：每个目录做什么，包的分组规则
- **命令清单**：所有可用的构建、测试、检查命令
- **编码约定**（20+ 条）：
  - 注册是效果（Registrations are effects）
  - 运行时不变量断言拥有的关系
  - 类型化事件使用声明合并
  - 切换判别标签
  - 瀑布流监听器必须调用 `next()`
  - 模型可见 ⟺ 已记录
  - 插件，而非循环变更
  - 能力接缝包含服务定义/提供者/消费者
  - 优先维护的依赖而非手搓
  - 包边界处显式优于隐式
  - 插件中没有硬编码的可调参数
  - 错误配置大声失败
  - 不透明的跨边界 ID 是品牌化的
  - 在类型化的同进程边界处信任 TypeScript
  - 源平面 vs 工件平面，不混合
  - 保持编译器面显式
  - 空 catch 命名它吞掉了什么
  - 不评论代码中显而易见的事实
  - 平行值优先对称
  - 测试描述行为，不描述正确性
  - 非平凡变更必须包含 Agent Note
  - 工具的 UI 渲染意图是设计的一部分
  - 计划单元、e2e 和快照覆盖
  - 审慎选择 PR 历史
  - 标签规范
  - TODO 标记分级
- **防御模式**：生命周期、并发、子进程、销毁
- **类型安全与文档**：严格 TypeScript + JSDoc 要求
- **Vendoring 政策**

### 5.2 packages/AGENTS.md（约 18 条规则）

**定位**：包开发专属规则，补充根级约定

**核心内容**：
- **插件导出规则**：服务包 default-export 服务类；函数插件 named-export `name`/`inject`/`Config`/`apply`
- **可选服务使用 `ctx.get(name)`**：保留 `ctx.<name>` 给声明的注入
- **产品可见插件需要真实组合测试**：手工 `ctx.plugin(...)` 不够
- **发起者拥有的私有链**：先派生再捕获
- **一个异步操作用一个生命周期控制器表示**
- **为所有当前消费者设计服务定义**：不让一个消费者 dictate 服务契约
- **要求当前所有者和需求**：每个抽象都要有当前消费者
- **公共选择需要证据**：配置性不证明不受支持的默认值合理
- **从模型视角写面向模型的契约**：提示词、工具模式只包含任务相关概念
- **在做决策的操作中执行决策**：模式省略、提示词过滤不是强制执行
- **只在提交点发布状态**：操作成功后才发通知和更新派生状态
- **对完整结果应用边界**：在完整值已知的地方强制执行限制
- **注册表贡献证明可处置**：通过 HMR 安全测试
- **每个包拥有 `./invariant`**：注册清单名，检查事件/数据关系
- **命名规则**：tsconfig、types.ts、测试位置、README 格式

### 5.3 docs/AGENTS.md（文档标准）

**定位**：文档结构、写作规则、字数预算

**核心内容**：
- **文档结构**：教程 vs 参考的分类
- **层级分类法**（每个事实一个家）：
  - 根 AGENTS.md → 常设指令
  - 子树 AGENTS.md → 特定子树的指令
  - architecture.md → 有序地图
  - subsystems/ → 子系统参考页
  - Agent Notes → 决策记录
  - postmortem/ → 事故故事
  - cookbook/ → 分步操作指南
  - user/ → 产品面向用户指南
  - 包 README → 每个包的契约
  - development.md → 贡献者设置
- **写作规则**：
  - 记录当前状态，不记录变更历史
  - 每段一个物理行
  - 围栏 ts 块必须编译
  - 注释和 JSDoc 陈述完整契约，不陈述推理过程
  - 直接写作：命名行动者和事实
- **字数预算**：
  - 根 AGENTS.md ≤ 1,600 词
  - architecture.md ≤ 1,800 词
  - 子树 AGENTS.md ≤ 600 词
- **Slop 检查清单**：清除冗余、历史叙述、状态标注、重述目录、推理转录等

### 5.4 其他子树 AGENTS.md

| 子树 | 核心规则 | 规则数量 |
|---|---|---|
| **examples/** | e2e smoke 测试要求（无 key + 有 key）、cordis.yml 注释规范 | ~5 条 |
| **scripts/** | gate 脚本调用约定（shell-free、路径规范化） | 1 条 |
| **vendor/** | 禁止随意修改 vendor/src、修改必须记录 | 1 条核心禁令 |
| **packages/web/** | 带凭证请求的重定向拒绝策略 | 1 条 |
| **.agents/notes/** | Agent Note 超期检查、归档政策 | ~3 条 |

---

## 六、CLAUDE.md 符号链接机制

### 6.1 为什么有 CLAUDE.md

不同的 AI 编码工具使用不同的约定文件名：
- **Codex** → 原生使用 `AGENTS.md`
- **Claude Code** → 使用 `CLAUDE.md`
- **opencode** → 两者都支持

DeepSeek Harness 需要**跨工具兼容**，所以同时支持两个名称。

### 6.2 符号链接的作用

项目中每个有 AGENTS.md 的目录下，都有一个 `CLAUDE.md` 文件，内容只有一行：

```
AGENTS.md
```

它是一个**符号链接**（或内容等价的文件），指向同目录的 `AGENTS.md`。

作用：
1. **兼容不同工具**：不管工具找 `AGENTS.md` 还是 `CLAUDE.md`，都能找到规则
2. **内容自动去重**：因为内容完全相同，加载时会被去重，不会重复加载
3. **单一真相来源**：只需要编辑 `AGENTS.md`，`CLAUDE.md` 自动同步

### 6.3 候选优先级

默认候选顺序：`['AGENTS.md', 'CLAUDE.md']`

同目录下两个文件都存在且内容不同时，`AGENTS.md` 优先（排在前面）。内容相同时，只加载一次。

---

## 七、本地覆盖层（Local Overlay）

### 7.1 什么是本地覆盖层

除了基础的 `AGENTS.md` / `CLAUDE.md`，每个目录还可以有本地覆盖文件：

- `AGENTS.local.md`
- `CLAUDE.local.md`

### 7.2 加载规则

- 本地覆盖层**在基础文件之后加载**（优先级更高）
- 同样遵循同目录内容去重
- 空列表禁用覆盖层
- 用户全局层（`$DSH_HOME`）没有本地覆盖层

### 7.3 用途

- 个人工作习惯的私有规则（不提交到仓库）
- 临时调试指令
- 机器特定的配置

---

## 八、与 Skills 的关系

### 8.1 区别

| 维度 | AGENTS.md | Skills（`.agents/skills/`） |
|---|---|---|
| **性质** | 常设规则（what you must know） | 工作流程序（how to do a task） |
| **加载方式** | 自动注入到会话历史 | 显式调用 Skill 工具触发 |
| **适用范围** | 对应目录下的所有工作 | 特定任务场景 |
| **格式** | Markdown 规则列表 | SKILL.md + 可选的 agents/ 配置 |
| **数量** | 每层一个，总共约 15 个 | 约 11 个专门技能 |

### 8.2 配合方式

AGENTS.md 告诉 AI "在这个目录工作要遵守什么规则"，Skill 告诉 AI "做某项具体工作时按什么流程"。两者配合形成完整的指导体系：

- `docs/AGENTS.md` 定义了文档的结构和字数预算
- `dsh-doc-standards` Skill 提供具体的文档放置和验证流程
- `dsh-prose-standard` Skill 提供文案审查的详细标准

---

## 九、设计哲学

### 9.1 就近原则

AI 在哪个目录工作，就读取哪个层级的 AGENTS.md。越贴近工作区域，规则越具体。

### 9.2 精炼可控

字数预算保证规则不膨胀，AI 能快速消化。超出时优先重新定位内容，而不是随意提高上限。

### 9.3 动态适应

Agent 不是一次性拿到所有规则，而是在工作过程中逐步获得更具体的指导——就像人类开发者一样，深入某个模块时才需要了解该模块的细节规则。

### 9.4 优先级明确

"More specific instructions take precedence over broader ones."——具体规则优先于宽泛规则，但都不覆盖系统/开发者/用户的直接指令。

### 9.5 信任边界

符号链接的指令文件会被跟随（follow symlinks），这意味着克隆的仓库可以加载树外的文件内容。但这些内容始终是**低权威的 workspace guidance**，永远不会覆盖系统指令。需要时可以通过文件系统策略门或 OS 沙箱来限制。

---

## 十、总结

AGENTS.md 体系是 DeepSeek Harness 项目中 AI 开发范式的核心基础设施之一。它不是简单的"规则文件"，而是一个**动态的、层级化的、与工作深度联动的指令系统**。

```
┌──────────────────────────────────────────────────────────┐
│                     同一个 Agent 会话                      │
│                                                            │
│  启动时：从根到 cwd 逐层加载基线指令                         │
│  工作中：触达更深目录时动态追加更具体的规则                   │
│  恢复时：核对基线兼容性，离线变更增量更新                    │
│  压缩后：重建基线，保证指令上下文完整                        │
│                                                            │
│  优先级：具体 > 宽泛  （但都低于系统/开发者/用户直接指令）    │
│  预算控制：保具体、舍宽泛，最相关的规则最后才被裁             │
│                                                            │
│  跨工具兼容：AGENTS.md + CLAUDE.md（符号链接 + 内容去重）    │
└──────────────────────────────────────────────────────────┘
```

**一句话总结**：AGENTS.md 体系是一个**洋葱式的动态指令层**——AI 越深入项目，获得的指导越精准、越具体。
