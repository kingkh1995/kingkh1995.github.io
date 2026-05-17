## 核心选择树

根据任务复杂度，有三种递进的使用方式：

```
简单任务/快速修复？
  └─ 是 → 直接提需求
  └─ 否 → 解释上下文很麻烦？
              └─ 是 → 输入 "ulw" 让智能体自己搞定
              └─ 否 → 需要精确可验证的执行？
                         └─ 是 → @plan 访谈规划 → /start-work 执行
                         └─ 否 → 直接 "ulw"
```

---

## 场景一：简单任务 / 快速修复

**什么时候用**：单文件改动、改个变量名、修个 typo、改一行 CSS

**怎么做**：直接打字说需求就行。Sisyphus（Claude Opus 4.7）的 IntentGate 会自动判断你的意图，简单任务不走复杂流程。

**示例**：
```
把按钮颜色改成蓝色
把 calculateTotal 函数重命名为 computeSum
修复这个拼写错误
```

---

## 场景二：复杂功能开发（懒人模式）

**什么时候用**：功能比较复杂，但你懒得从头交代全部背景；或者不想写详细描述，一句话就够了

**怎么做**：输入 `ulw` 或 `ultrawork`

Sisyphus 会进入超工作模式，自动切换到更强的推理模型，自主探索代码库、研究现有模式、实现功能、用诊断工具验证。全自动，持续工作直到完成。

Ulw 的核心优势是**编排能力**——它仍然是 Sisyphus，会拆解任务、委派给子智能体并行执行、验证每个步骤的结果。适合需要多步骤协调的工作流。

**适合 Ulw 的场景（日常开发 80-90% 的工作量）**：
- **日常功能开发**：`ulw 实现 JWT 认证`——需要规划 + 多 agent 并行执行
- **多步骤任务**：需要拆解、委派、验证的工作流，比如"添加用户注册 + 邮件验证 + 权限控制"
- **懒模式**：不想写详细描述，输入 ulw + 一句话就够
- **有 Prometheus 计划时**：ulw 可以直接接入已有计划继续执行

**示例**：
```
ulw
给这个项目添加用户注册功能
```

> **Ulw vs Hephaestus**：Ulw 是编排者思维——拆解任务、委派执行、验证结果。Hephaestus 是独立深潜思维——一个人关起门来从头到尾写完。日常功能开发用 Ulw，深度推理链和跨域知识综合用 Hephaestus（见场景六）。

---

## 场景三：精确可控的复杂项目

**什么时候用**：多日项目、关键生产变更、复杂重构、需要记录决策轨迹

**怎么做**：按 **Tab** 键进入 Prometheus 模式（或输入 `@plan 你的任务`）

1. **Prometheus**（继承 Sisyphus，Claude Opus 4.7）像真正的工程师一样访谈你——问澄清性问题、识别范围歧义
2. **Metis** 做差距分析，补全 Prometheus 遗漏的点
3. **Momus**（GPT-5.5 xhigh）审核计划的质量
4. 运行 `/start-work`，**Atlas**（Claude Sonnet 4.6）接手执行，将任务分发给专门的子智能体，每个完成项独立验证

**示例**：
```
@plan 把这个单体后端拆分成微服务架构
# Prometheus 会问你：服务边界怎么划分？数据一致性怎么处理？...
# 确认计划后运行：
/start-work
```

---

## 场景四：架构决策 / 复杂调试

**什么时候用**：遇到不熟悉的模式、安全问题、多系统权衡

**怎么做**：咨询 Oracle（GPT-5.5 high）

Oracle 是只读高智商顾问，不写代码只分析。遇到棘手问题先问 Oracle 再动手。

**示例**：
```
帮我咨询一下 Oracle：这个分布式锁方案有什么潜在问题？
```

---

## 场景五：前端 / UI 开发

**什么时候用**：UI 组件、CSS 样式、布局、动画、设计

**需要的模型能力**：Gemini 3.1 Pro，前端 / UI 天生擅长

**怎么做**：任务会自动路由到 `visual-engineering` 类别。

**示例**：
```
帮我写一个响应式导航栏组件，包含下拉菜单和移动端汉堡菜单
```

---

## 场景六：深度编码 / 自主开发

**什么时候用**：需要 GPT-5.5 的深度推理链和自主探索能力，Sisyphus 的编排模式无法胜任的场景

**需要的模型能力**：GPT-5.5（Hephaestus）或 GPT-5.3 Codex（deep 类别）——原则驱动的自主推理，不依赖外部指令

**怎么做**：按 **Tab** 切换到 Hephaestus 智能体，或者显式使用 `deep` 类别

Hephaestus 的核心工作方式是**自主深潜**——给目标不给步骤，他自己探索代码库、研究模式、端到端执行。它可以委派给 explore / librarian / oracle 做辅助调研，但核心编码工作全部自主完成。

**适合 Hephaestus 的场景（约 10-20% 的工作量）**：
- **深度推理链 bug**：那种需要追踪 15 个文件的因果链，GPT-5.5 的推理能力在这里是杀手锏
- **架构设计决策**：需要从多个方案中推理出最优解时，Hephaestus 的 principle-driven 风格比编排式工作流更有效
- **跨域知识综合**：Rust 核心与 TS 前端集成、零停机数据库迁移这类需要深层理解多个技术栈的场景

**示例**：
```
切换到 Hephaestus，帮我实现一个基于 WebSocket 的实时协作编辑功能
# 或者用 deep 类别：
用 deep 帮我排查这个并发竞态条件，调用链跨越了 15 个文件
```

> **Hephaestus vs Ulw**：Hephaestus 是"关起门来的资深工程师"——一个人从头到尾搞定，不需要协调任何人。Ulw 是"项目经理 + 全团队"——拆解任务、分配工作、协调并行、验证交付。绝大多数日常开发用 Ulw 就够了，只有真正需要深度推理链的场景才值得切 Hephaestus。

---

## 场景七：硬核逻辑 / 算法

**什么时候用**：复杂算法、架构设计、需要最强推理能力

**需要的模型能力**：GPT-5.5 xhigh，最强推理

**怎么做**：使用 `ultrabrain` 类别

**示例**：
```
用 ultrabrain 来分析这个一致性哈希环的实现，找出热点问题和优化方案
```

---

## 场景八：文档 / 写作

**什么时候用**：技术文档、README、散文、API 文档

**怎么做**：使用 `writing` 类别（Gemini 3 Flash）

**示例**：
```
用 writing 帮我写一份这个项目的 API 使用文档
```

---

## 场景九：创意 / 非传统问题

**什么时候用**：需要跳出常规思维的创造性问题

**怎么做**：使用 `artistry` 类别（Gemini 3.1 Pro）

**示例**：
```
用 artistry 帮我设计一个非传统的用户引导流程
```

---

## 场景十：代码搜索 / 研究

**什么时候用**：在代码库中找模式、查 API 文档、搜索开源实现

**怎么做**：使用 **Explore** 或 **Librarian**（GPT-5.4 Mini Fast）

这些是工具型智能体，速度快于智能，搜索不需要深度推理。

**示例**：
```
Explore 帮我找一下这个项目里所有用到 Redis 分布式锁的地方
```

---

## 智能体切换技巧

按 **Tab** 键可以在核心智能体之间轮转切换，固定顺序是：

**Sisyphus (0) → Hephaestus (1) → Prometheus (2) → Atlas (3)** → 其余智能体

---

## 一句话速查表

| 你想要的 | 怎么做 | 推荐模型 |
|---|---|---|
| 简单改点东西 | 直接说 | Claude Opus 4.7 |
| 不想动脑子 | 输入 `ulw` | Claude Opus 4.7 |
| 要把事情做完美 | `@plan` → `/start-work` | Prometheus: Claude Opus 4.7 → Atlas: Claude Sonnet 4.6 |
| 拿不准主意 | 问 Oracle | GPT-5.5 high |
| 前端 UI | 交给 visual-engineering | Gemini 3.1 Pro |
| 深度编码 | 交给 Hephaestus / deep | GPT-5.5 / GPT-5.3 Codex |
| 硬核逻辑 | 用 ultrabrain | GPT-5.5 xhigh |
| 写文档 | 用 writing | Gemini 3 Flash |
| 搜代码 | 用 Explore / Librarian | GPT-5.4 Mini Fast |
| 创意任务 | 用 artistry | Gemini 3.1 Pro |

---

## 配置参考：仅 OpenCode Go 订阅最佳实践

如果你只有 OpenCode Go 订阅（无 OpenAI / Anthropic / Google 额外付费），以下是当前使用的纯 `opencode-go` 模型配置。与上表官方推荐模型的偏离点及理由见配置思路说明。

### 1. 基础配置（`opencode.jsonc`）

```json
{
  "plugin": ["oh-my-openagent@latest"],
  "$schema": "https://opencode.ai/config.json",
  "provider": {}
}
```

### 2. Oh My OpenAgent 配置（`oh-my-openagent.json`）

```json
{
  "agents": {
    "sisyphus": {
      "model": "opencode-go/deepseek-v4-pro",
      "variant": "high",
      "ultrawork": {
        "model": "opencode-go/deepseek-v4-pro",
        "variant": "xhigh"
      }
    },
    "hephaestus": {
      "model": "opencode-go/deepseek-v4-pro"
    },
    "oracle": {
      "model": "opencode-go/glm-5.1",
      "variant": "high"
    },
    "prometheus": {
      "model": "opencode-go/deepseek-v4-pro",
      "prompt_append": "Leverage deep & quick agents heavily, always in parallel."
    },
    "metis": {
      "model": "opencode-go/glm-5.1"
    },
    "momus": {
      "model": "opencode-go/glm-5.1",
      "variant": "high"
    },
    "atlas": {
      "model": "opencode-go/kimi-k2.6"
    },
    "explore": {
      "model": "opencode-go/deepseek-v4-flash"
    },
    "librarian": {
      "model": "opencode-go/deepseek-v4-flash"
    },
    "multimodal-looker": {
      "model": "opencode-go/qwen3.6-plus"
    }
  },
  "categories": {
    "visual-engineering": {
      "model": "opencode-go/qwen3.6-plus",
      "variant": "high"
    },
    "ultrabrain": {
      "model": "opencode-go/deepseek-v4-pro",
      "variant": "xhigh"
    },
    "deep": {
      "model": "opencode-go/deepseek-v4-pro"
    },
    "artistry": {
      "model": "opencode-go/qwen3.6-plus",
      "variant": "high"
    },
    "quick": {
      "model": "opencode-go/deepseek-v4-flash"
    },
    "unspecified-low": {
      "model": "opencode-go/deepseek-v4-flash"
    },
    "unspecified-high": {
      "model": "opencode-go/deepseek-v4-pro",
      "variant": "medium"
    },
    "writing": {
      "model": "opencode-go/kimi-k2.6"
    }
  },
  "$schema": "https://raw.githubusercontent.com/code-yeongyu/oh-my-openagent/dev/assets/oh-my-opencode.schema.json",
  "claude_code": {
    "mcp": false,
    "commands": false,
    "skills": false,
    "agents": false,
    "hooks": false,
    "plugins": false
  },
  "background_task": {
    "providerConcurrency": {
      "opencode-go": 12
    },
    "modelConcurrency": {
      "opencode-go/deepseek-v4-pro": 5,
      "opencode-go/deepseek-v4-flash": 8,
      "opencode-go/kimi-k2.6": 2,
      "opencode-go/glm-5.1": 3,
      "opencode-go/qwen3.6-plus": 4
    }
  },
  "experimental": {
    "aggressive_truncation": true,
    "task_system": true,
    "auto_resume": true
  }
}
```

---

## 配置参考：OpenCode Zen Free

```json
{
  "$schema": "https://raw.githubusercontent.com/code-yeongyu/oh-my-openagent/dev/assets/oh-my-opencode.schema.json",
  "agents": {
    "sisyphus": {
      "model": "opencode/qwen3.6-plus-free"
    },
    "atlas": {
      "model": "opencode/qwen3.6-plus-free"
    },
    "oracle": {
      "model": "opencode/deepseek-v4-flash-free",
      "variant": "max"
    },
    "librarian": {
      "model": "opencode/deepseek-v4-flash-free"
    },
    "explore": {
      "model": "opencode/deepseek-v4-flash-free"
    },
    "multimodal-looker": {
      "model": "opencode/qwen3.6-plus-free"
    },
    "metis": {
      "model": "opencode/qwen3.6-plus-free"
    },
    "momus": {
      "model": "opencode/deepseek-v4-flash-free",
      "variant": "max"
    }
  },
  "categories": {
    "visual-engineering": {
      "model": "opencode/qwen3.6-plus-free"
    },
    "ultrabrain": {
      "model": "opencode/deepseek-v4-flash-free",
      "variant": "max"
    },
    "deep": {
      "model": "opencode/deepseek-v4-flash-free",
      "variant": "high"
    },
    "artistry": {
      "model": "opencode/qwen3.6-plus-free"
    },
    "quick": {
      "model": "opencode/deepseek-v4-flash-free"
    },
    "unspecified-low": {
      "model": "opencode/deepseek-v4-flash-free"
    },
    "unspecified-high": {
      "model": "opencode/deepseek-v4-flash-free",
      "variant": "max"
    },
    "writing": {
      "model": "opencode/qwen3.6-plus-free"
    }
  },
  "claude_code": {
    "agents": false,
    "commands": false,
    "hooks": false,
    "mcp": false,
    "plugins": false,
    "skills": false
  }
}
```
