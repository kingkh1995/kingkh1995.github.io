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

**怎么做**：直接打字说需求就行。Sisyphus 的 IntentGate 会自动判断你的意图，简单任务不走复杂流程。

**示例**：
```
把按钮颜色改成蓝色
把 calculateTotal 函数重命名为 computeSum
修复这个拼写错误
```

---

## 场景二：复杂功能开发（懒人模式）

**什么时候用**：功能比较复杂，但你懒得从头交代全部背景

**怎么做**：输入 `ulw` 或 `ultrawork`

Sisyphus 会自动探索你的代码库、研究现有模式、实现功能、用诊断工具验证。全自动，持续工作直到完成。

**示例**：
```
ulw
给这个项目添加用户注册功能
```

---

## 场景三：精确可控的复杂项目

**什么时候用**：多日项目、关键生产变更、复杂重构、需要记录决策轨迹

**怎么做**：按 **Tab** 键进入 Prometheus 模式（或输入 `@plan 你的任务`）

1. **Prometheus** 像真正的工程师一样访谈你——问澄清性问题、识别范围歧义
2. **Metis** 做差距分析，补全 Prometheus 遗漏的点
3. **Momus** 审核计划的质量
4. 运行 `/start-work`，**Atlas** 接手执行，将任务分发给专门的子智能体，每个完成项独立验证

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

**怎么做**：咨询 Oracle

Oracle 是只读高智商顾问，最佳配置是 GPT-5.5（最高推理强度），不写代码只分析。遇到棘手问题先问 Oracle 再动手。仅 OpenCode Go 订阅时，用 DeepSeek V4 Pro high 变体降级替代，依然能胜任大多数架构分析场景。

**示例**：
```
帮我咨询一下 Oracle：这个分布式锁方案有什么潜在问题？
```

---

## 场景五：前端 / UI 开发

**什么时候用**：UI 组件、CSS 样式、布局、动画、设计

**底层模型**：Qwen 3.6 Plus（天生擅长视觉任务）

**怎么做**：任务会自动路由到 `visual-engineering` 类别。

**示例**：
```
帮我写一个响应式导航栏组件，包含下拉菜单和移动端汉堡菜单
```

---

## 场景六：深度编码 / 自主开发

**什么时候用**：跨多个文件的复杂编码、深度架构推理、需要 DeepSeek V4 Pro 的独特优势

**怎么做**：切换到 Hephaestus 智能体，或者显式使用 `deep` 类别

Hephaestus 最佳配置是 GPT-5.5，给目标不给步骤——他自己探索代码库、研究模式、端到端执行。仅 OpenCode Go 订阅时，用 DeepSeek V4 Pro medium 变体降级替代，依然能实现高质量的跨文件自主开发。

**示例**：
```
切换到 Hephaestus，帮我实现一个基于 WebSocket 的实时协作编辑功能
```

---

## 场景七：硬核逻辑 / 算法

**什么时候用**：复杂算法、架构设计、需要最强推理能力

**底层模型**：DeepSeek V4 Pro xhigh（最高推理强度配置）

**怎么做**：使用 `ultrabrain` 类别

**示例**：
```
用 ultrabrain 来分析这个一致性哈希环的实现，找出热点问题和优化方案
```

---

## 场景八：文档 / 写作

**什么时候用**：技术文档、README、散文、API 文档

**底层模型**：Kimi K2.6

**怎么做**：使用 `writing` 类别

---

## 场景九：创意 / 非传统问题

**什么时候用**：需要跳出常规思维的创造性问题

**底层模型**：Qwen 3.6 Plus（与 visual-engineering 共用，不同推理风格）

**怎么做**：使用 `artistry` 类别

---

## 场景十：代码搜索 / 研究

**什么时候用**：在代码库中找模式、查 API 文档、搜索开源实现

**怎么做**：使用 **Explore**（代码库搜索）或 **Librarian**（外部文档查询）

这些是工具型智能体，用最快最便宜的模型，不花冤枉钱。

---

## 智能体切换技巧

按 **Tab** 键可以在核心智能体之间轮转切换，固定顺序是：

**Sisyphus (0) → Hephaestus (1) → Prometheus (2) → Atlas (3)** → 其余智能体

---

## 一句话速查表

| 你想要的 | 怎么做 |
|---|---|
| 简单改点东西 | 直接说 |
| 不想动脑子 | 输入 `ulw` |
| 要把事情做完美 | `@plan` → `/start-work` |
| 拿不准主意 | 问 Oracle |
| 前端 UI | 交给 visual-engineering |
| 深度编码 | 交给 Hephaestus / deep |
| 硬核逻辑 | 用 ultrabrain |
| 写文档 | 用 writing |
| 搜代码 | 用 Explore / Librarian |

---

## 配置参考：仅 OpenCode Go 订阅最佳实践

如果你只有 OpenCode Go 订阅（无 OpenAI / Anthropic / Google 额外付费），以下是我当前使用的纯 opencode-go 模型配置。这是我认为在仅 OpenCode Go 订阅下，各智能体和任务类别的最佳模型分配：

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
  "$schema": "https://raw.githubusercontent.com/code-yeongyu/oh-my-openagent/dev/assets/oh-my-opencode.schema.json",

  "agents": {
    "sisyphus": {
      "model": "opencode-go/kimi-k2.6",
      "ultrawork": { "model": "opencode-go/kimi-k2.6" }
    },
    "hephaestus": { "model": "opencode-go/deepseek-v4-pro", "variant": "medium" },
    "oracle": { "model": "opencode-go/deepseek-v4-pro", "variant": "high" },
    "prometheus": {
      "model": "opencode-go/kimi-k2.6"
    },
    "atlas": {
      "model": "opencode-go/kimi-k2.6"
    },
    "explore": { "model": "opencode-go/deepseek-v4-flash" },
    "librarian": { "model": "opencode-go/deepseek-v4-flash" }
  },

  "categories": {
    "visual-engineering": { "model": "opencode-go/qwen3.6-plus" },
    "deep": { "model": "opencode-go/deepseek-v4-pro", "variant": "medium" },
    "ultrabrain": { "model": "opencode-go/deepseek-v4-pro", "variant": "xhigh" },
    "quick": { "model": "opencode-go/deepseek-v4-flash" },
    "unspecified-low": {
      "model": "opencode-go/deepseek-v4-flash"
    },
    "unspecified-high": {
      "model": "opencode-go/kimi-k2.6"
    },
    "writing": {
      "model": "opencode-go/kimi-k2.6"
    },
    "artistry": { "model": "opencode-go/qwen3.6-plus" }
  },

  "background_task": {
    "providerConcurrency": {
      "opencode-go": 10
    }
  }
}
```

### 3. 配置思路说明

#### 编排层（Sisyphus / Prometheus / Atlas）

Kimi K2.6 是当前 opencode-go 订阅中性价比最高的编排模型：
- **Context 窗口大**：128K，适合理解整个代码库
- **指令遵循能力强**：适合 IntentGate 解析和任务分发
- **速度快**：相比 DeepSeek V4 Pro，响应更及时

#### 深度推理（Oracle / Hephaestus / ultrabrain / deep）

DeepSeek V4 Pro 是 opencode-go 中最强的推理模型：
- **Oracle 用 high 变体**：纯分析场景不需要最高推理强度，balance 速度和深度
- **Hephaestus 用 medium 变体**：编码任务需要推理但不需要推到极致
- **ultrabrain 用 xhigh 变体**：最硬核的算法和架构问题才需要全力以赴

#### 前端视觉与创意（visual-engineering / artistry）

Qwen 3.6 Plus 在 opencode-go 的视觉和创意能力最强：
- **多模态理解**：对 UI 截图、设计稿的理解能力突出
- **CSS 生成**：生成的样式代码质量高，移动端适配考虑周全
- **不同推理风格**：visual-engineering 用于结构化视觉任务，artistry 用于开放性创意任务，两者共用同一模型但系统提示不同

#### 工具型智能体（Explore / Librarian / quick / unspecified-low）

DeepSeek V4 Flash 是最快的模型：
- **代码搜索**：不需要深度推理，快就行
- **文档查询**：快速定位信息，低成本
- **简单任务**：`quick` 类别专门处理 trivial 工作
- **通用低工作量任务**：`unspecified-low` 用于不符合其他专门类别的一般性低工作量任务，同样遵循 fast cheap 原则

#### 背景任务并发

`providerConcurrency.opencode-go: 10` 允许最多 10 个并发的 opencode-go 请求。这是 Oh My OpenAgent 并行执行多个子任务时的关键配置——比如 Atlas 同时分发 5 个独立的文件修改任务时，它们可以并行而不是排队。

### 4. 各场景对应的实际模型

| 场景 | 智能体/类别 | 实际调用的模型 | 变体 |
|---|---|---|---|
| 简单修复 | Sisyphus (默认) | Kimi K2.6 | - |
| 懒人开发 | Sisyphus (ultrawork) | Kimi K2.6 | - |
| 精确项目 | Prometheus → Atlas | Kimi K2.6 | - |
| 架构咨询 | Oracle | DeepSeek V4 Pro | high |
| 前端开发 | visual-engineering | Qwen 3.6 Plus | - |
| 深度编码 | Hephaestus / deep | DeepSeek V4 Pro | medium |
| 硬核算法 | ultrabrain | DeepSeek V4 Pro | xhigh |
| 文档写作 | writing | Kimi K2.6 | - |
| 创意任务 | artistry | Qwen 3.6 Plus | - |
| 通用低工作量 | unspecified-low | DeepSeek V4 Flash | - |
| 代码搜索 | Explore / Librarian | DeepSeek V4 Flash | - |
| 快速任务 | quick | DeepSeek V4 Flash | - |

这个配置的核心原则是：**把钱花在刀刃上**。编排层用 Kimi K2.6（指令遵循强、响应快），真正需要深度推理的时候才请出 DeepSeek V4 Pro，所有 fast cheap 场景（工具型任务、快速任务、通用低工作量任务）统一用 DeepSeek V4 Flash。在仅 OpenCode Go 订阅的约束下，这是覆盖所有场景的最优解。
