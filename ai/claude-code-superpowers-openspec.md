## 三个痛点

如果你用 Claude Code 或其他 AI 编程工具做过正经项目，大概率遇到过这三个问题：

**需求偏差**：你说"加个用户登录"，AI 给你做了 Session 认证，但你想要的是 JWT。你说"支付扫描"，AI 集成了真实支付 SDK，而你只是要一个 Demo。等你审完代码才发现不对，Token 和时间已经烧了。

**缺乏工程纪律**：Claude Code 的默认行为是"收到需求就开始写代码"。没有 Git 分支、没有测试、没有 Code Review。出了问题不知道哪里错了，回滚也很痛苦——因为它直接改了 main 分支。

**决策不可追溯**：为什么用 bcrypt 而不是 argon2？为什么 API 前缀是 `/api` 而不是 `/v1`？上周的设计决策跟着对话一起消失了。三个月后没人记得原因，新来的同事没有任何上下文。

这三个问题靠更好的 prompt 解决不了——需要不同层面的工具配合。按约束强度从弱到强：Claude Code（无约束）→ Superpowers（工程纪律约束）→ OpenSpec（规格驱动约束）。

## 核心流程与原理

### Claude Code：直接执行

Claude Code 的核心流程：进入 Plan 模式规划 → 确认后编码。Plan 模式下 AI 会探索代码库、设计方案，输出一份计划文档供你审阅。确认后退出 Plan 模式，AI 开始写代码。

流程简洁，上手快，但有两个问题：规划阶段的理解只存在于当前对话，关掉就丢了；编码阶段没有任何质量约束——没有测试先行的要求，没有 Code Review，没有 Git 分支隔离。

### Superpowers：工程纪律约束

Superpowers 是安装在 Claude Code 中的 Skill 框架。安装后，Claude Code 不再拿到需求就直接编码，而是自动进入强制流程：

```
brainstorming → writing-plans → TDD 编码 → code-review → verification
```

**brainstorming**：创建功能或组件前自动触发。先和你讨论需求边界、技术选型、实现方案，确认对齐后才进入下一步。假设你要加一个缓存层，它会先问：用 Caffeine 还是 Redis？TTL 还是事件失效？需要穿透保护吗？这些答案直接影响代码结构。

**writing-plans**：需求确认后分解为任务清单，每个任务有明确验收标准。

**编码纪律类 Skill**（由 `using-superpowers` 主 Skill 根据上下文自动匹配并触发）：

| Skill | 触发时机 | 作用 |
| --- | --- | --- |
| test-driven-development | 实现功能或修复 bug 前 | 先写失败测试再写实现。甚至会删除先于测试编写的代码——防止先写代码再补测试 |
| systematic-debugging | 遇到 bug、测试失败时 | 按固定流程定位问题，而非随机尝试 |
| requesting-code-review | 完成主要实现步骤后 | 自动审查代码质量 |
| verification-before-completion | 声称工作完成前 | 强制运行验证命令，有证据才能声称完成 |
| dispatching-parallel-agents | 2+ 个独立任务可并发时 | 自动触发并行执行 |

Superpowers 解决了工程纪律问题：有 TDD 兜底、有 Code Review 把关、有 Git Worktree 分支隔离。`using-superpowers` 作为主 Skill，会根据当前上下文自动匹配并触发对应 Skill——编码前触发 TDD、遇到 bug 触发 debugging、完成实现后触发 code-review。每个 Skill 的 `description` 字段定义了触发条件，Claude Code 在 `using-superpowers` 框架下主动检查并调用匹配的 Skill。CLAUDE.md 可用于覆盖或禁用特定 Skill（用户指令优先级高于 Skill）。

但 brainstorming 的设计文档仅保存为单次文件（`docs/superpowers/specs/YYYY-MM-DD-<topic>-design.md`），不具备 OpenSpec 那样按版本归档、增量合并的能力。下次 brainstorm 创建新文件覆盖工作目录中的上下文，旧版本仅存于 git 历史。下次迭代或新成员加入时，需要手动定位文件，没有 OpenSpec 自动加载和结构化的 4 文档体系。

### OpenSpec：规格驱动约束

OpenSpec 的核心理念是把一句话需求展开为四个结构化文档，让后续所有工作基于文档而非原始描述。核心流程围绕五个命令（均为默认 core profile）：

| 命令 | 作用 |
| --- | --- |
| `/opsx:explore` | 探索模式，和 AI 讨论想法、梳理思路 |
| `/opsx:propose <需求>` | 生成四份文档：proposal + spec + design + tasks |
| `/opsx:apply` | 按 spec 逐任务实现代码 |
| `/opsx:sync` | 将增量 Spec 合并到主 Spec 目录 |
| `/opsx:archive` | 归档变更至 archive 目录，保持规格与代码同步 |

最简流程：**propose → apply → sync → archive**。explore 按需使用。

propose 阶段生成的四份文档各有明确用途：

- `proposal.md`：为什么做、范围是什么、**关键是不做什么**（防止 AI 自行加戏）
- `design.md`：技术决策及其理由——为什么选 Redis 不选 Caffeine、TTL 设多久、Key 命名规范
- `specs/`：用 GIVEN/WHEN/THEN 描述行为规格，描述的是**期望行为**而非实现步骤
- `tasks.md`：实施清单，每个任务 2-5 分钟可完成

关键价值：**这些文档不会随对话消失。** 三个月后你可以打开 `design.md` 看到当初的设计理由，新同事可以从 `specs/` 理解行为契约。archive 会同步增量 Spec 并归档变更，保持规格与代码的长期一致。

但 OpenSpec 的局限在 apply 阶段：有规格约束但没有质量兜底。AI 可能在实现时偏离规格，没有 TDD 先测试后编码的要求，没有 Code Review 把关。就像有完美的建筑图纸但没有施工监理。

> `/opsx:ff`、`/opsx:verify`、`/opsx:new`、`/opsx:continue`、`/opsx:bulk-archive`、`/opsx:onboard` 不在默认 core profile 中（`/opsx:sync` 已在 core 内）。需运行 `openspec config profile` 切换到扩展 profile 后执行 `openspec update`。core profile 的典型流程为 propose → apply → sync → archive，大多数场景足够。要修改生成的文档，直接编辑文件即可。

## 三者配合：对比与组合

### 各自的短板

| 能力 | Claude Code | + Superpowers | + OpenSpec |
| --- | --- | --- | --- |
| 需求探索 | ❌ Plan 模式，无结构化产出 | ✅ brainstorming | ✅ propose（产出文档） |
| 任务分解 | ❌ 无强制 | ✅ writing-plans | ✅ tasks.md（持久化） |
| 规格持久化 | ❌ 对话级 | ⚠️ 单次设计文档 | ✅ 文件级，归档保存 |
| 决策可追溯 | ❌ 聊天关闭即消失 | ⚠️ 单次覆盖，无归档 | ✅ design.md 永久记录 |
| TDD 强制 | ❌ | ✅ 自动触发 | ❌ |
| Code Review | ❌ | ✅ 自动触发 | ❌ |
| Git 分支隔离 | ❌ | ✅ Worktree | ❌ |
| 系统化调试 | ❌ | ✅ 自动触发 | ❌ |
| 完成前验证 | ❌ | ✅ 自动触发 | ❌ |

重叠的只有前两行（需求探索和任务分解），其余完全互补。

**只用 Claude Code**：没有规格约束，不同开发者得到不同代码风格，返回格式不一致（`{code: 200}` vs `{success: true}`），安全约束缺失。实际案例中出现过密码明文存储、私有端点无鉴权——上线才暴露。

**Claude Code + Superpowers**：有工程纪律保障，但设计决策保存在单次设计文档中（`docs/superpowers/specs/`），每次 brainstorming 创建新的设计文件，不具备 OpenSpec 的多版本归档能力。下一次迭代没有结构化的可复用规格文档供团队共享。

**Claude Code + OpenSpec**：规格持久、决策可追溯，但 apply 阶段没有工程纪律。AI 可能在实现时偏离规格，没有 TDD 和 Code Review 兜底。有图纸没监理。

### 组合方式

三者齐全时，**OpenSpec 主导规划阶段，Superpowers 主导编码阶段**：

```
OpenSpec（规划）               Superpowers（编码纪律）
想清楚"做什么"                确保"怎么做"的质量
┌──────────┐                  ┌──────────────┐
│ propose  │                  │ brainstorming │ ← 被 propose 替代
│ tasks.md │                  │ writing-plans │ ← 被 tasks.md 替代
│ apply ───┼────────────────→ │ TDD          │ ← 自动触发
│          │                  │ debugging    │ ← 自动触发
│          │                  │ verification │ ← 自动触发
│ archive  │                  │ code-review  │ ← 自动触发
└──────────┘                  └──────────────┘
```

OpenSpec 的 propose 完成了需求探索和设计决策，天然替代了 Superpowers 的 brainstorming 和 writing-plans。当 apply 进入编码阶段时，Superpowers 的 TDD、debugging、verification、code-review 等技能由 `using-superpowers` 根据上下文字段自动匹配触发，无需手动调用。

关键点在于 CLAUDE.md 的路由：brainstorming 在收到新需求时也会触发，若不覆盖则与 propose 产生两套竞争文档。`using-superpowers` 规定了 `CLAUDE.md > Skills > 系统提示词` 的优先级，因此在 CLAUDE.md 中声明即可精确控制：

```
## OpenSpec + Superpowers 工作流

### 规划阶段（OpenSpec 主导）
- 任何新功能、重大变更必须使用 /opsx:propose 启动
- **禁止**使用 brainstorming 和 writing-plans 技能 — OpenSpec 的 propose 已替代
- /opsx:propose 完成后必须审阅 proposal.md（重点检查 Out of Scope）和 design.md

### 编码阶段（Superpowers 主导）
- /opsx:apply 实施时，每个任务遵循 TDD：先写失败测试 → 最小实现 → 重构
- tasks.md 中存在 2+ 个无依赖的独立任务时，并行派发子代理执行
- 每个任务完成后触发 code-review
- **禁止**在未运行验证命令的情况下声明完成
- 遇到测试失败或异常行为时执行 systematic-debugging，禁止猜测
- 每个新功能在独立 Git Worktree 中开发，禁止直接在 main 分支修改

### 完整流程
propose → 审阅文档 → apply（TDD + Code Review + 并行派发自动触发）→ archive → 推送到 main
```

## 完整工作流实战

用一个用户认证 API（Express + MongoDB + JWT）演示三者配合的完整流程。

### 阶段一：需求 → 规格

```bash
claude
> /opsx:propose User auth API with Express + MongoDB + JWT.
> Features: registration (username+email+password), login (return JWT),
> get current user (requires auth).
> Security: bcrypt password encryption, JWT auth for private endpoints.
```

OpenSpec 生成四份文档。**你的动作**：打开 `proposal.md`，重点检查 **Out of Scope** 部分——确认 AI 没有自行添加 OAuth 或密码重置功能。如果有偏差，直接编辑文件修正。

产出：结构化的需求蓝图 + 设计决策记录（密码算法、JWT 过期时间、ORM 选择全部记录在 `design.md` 中）。

### 阶段二：审阅与执行

花 5 分钟审阅 `tasks.md`：任务顺序是否合理、验收标准是否清晰、有没有遗漏。确认后执行：

```bash
> /opsx:apply
```

Superpowers 的编码纪律技能自动接管 apply 阶段：

```
[Task 1/6] Project Init ✓
[Task 2/6] Database Connection
├─ Write test → Fails (RED) ✓
├─ Write implementation → Passes (GREEN) ✓
├─ Code Review → Pass ✓
└─ Git commit ✓

[Tasks 3-5 无依赖 → 并行派发]
├─ Agent A → Task 3: Registration
├─ Agent B → Task 4: Login
└─ Agent C → Task 5: Get Current User
...
```

AI 在独立 Git Worktree 上工作，遵循规格，有 TDD 约束。出问题直接丢弃分支——主分支不受影响。

### 阶段三：归档与验证

```bash
> /opsx:archive
```

**不要跳过 archive。** 跳过的后果：下次会话 AI 读到旧 Spec，重新实现已有功能。归档后做最终验证：启动服务、跑 curl 测试接口。你的工作集中在：确认需求 → 审阅计划 → 验证结果。

## 总结

这套组合的本质是**把人类的工程最佳实践（需求对齐、TDD、Code Review、决策记录）编码为 AI 必须遵守的规则**。Claude Code 提供执行能力，Superpowers 补上工程纪律，OpenSpec 补上规格持久和决策追溯。三者不冲突，各管一段。

从小处开始：先用 Claude Code，觉得缺纪律就加 Superpowers，觉得需求总对不齐就加 OpenSpec。工具服务于目标，不是反过来。最终目标不是"让 AI 写更多代码"，而是**让 AI 生成的代码和有纪律的人类工程师写的代码一样可靠、可维护、可追溯**。

当前 Superpowers 和 OpenSpec 的集成没有官方文档，路由规则完全依赖 CLAUDE.md 的优先级机制。但这也是组合的优势——两者各自独立演进，通过指令优先级松耦合，互不绑定。未来如果官方提供原生集成，CLAUDE.md 的配置也只需删掉覆盖规则即可平滑迁移。

## 参考资料

- [Fission-AI/OpenSpec — GitHub](https://github.com/Fission-AI/OpenSpec)
- [obra/superpowers — GitHub](https://github.com/obra/superpowers)
- [Claude Code + OpenSpec + Superpowers: From Requirements to Production-Ready Code](https://www.heyuan110.com/posts/ai/2026-04-09-claude-code-openspec-superpowers/)
