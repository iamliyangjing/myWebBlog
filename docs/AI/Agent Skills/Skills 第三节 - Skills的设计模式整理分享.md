---
description: 本文系统梳理了来自黄佳、Anthropic 与 Google Cloud Tech 的 Agent Skill 设计模式，涵盖模板驱动、脚本增强、知识分层、工具隔离、串行工作流、多 MCP 协调、迭代优化、上下文感知工具选择与领域特定智能等模式。文章不仅阐明各模式的核心思想与适用场景，更通过具体示例展示如何将业务规则、行业知识与工程化流程“编译”为结构化 Skill。帮助开发者从“依赖模型输出”转向“工程化构建智能体”，设计出稳定、安全、可落地到企业级生产环境的高质量技能模块。
title: Agent Skill 设计模式全景解析：从模板驱动到领域智能
tag:
  - Agent Skill
  - AI
sidebar: true
comment: true
recommend: 3
---
## 一、前言

目前，AgentSkills 已进入了快速发展期。随着行业经验和个人实践的不断积累，各类 Skills 如雨后春笋般涌现。在这个阶段，我们已经可以看到一些经过验证的设计模式，它们像路标一样，帮助我们在复杂的 Skill 开发过程中少走弯路。设计得越稳健的 Skills，质量就越高，执行效果也越可靠——这不仅关乎效率，更关乎信任。

现在真正的挑战是**内容设计**。

目前，关于 Skills 设计模式的总结资料已经有一些非常出色的分享，包括：

1. 黄佳老师整理常规场景的 Skills 设计模式
2. Anthropic 分享的 Skills 设计模式
3. 谷歌公司分享的 Skills 设计模式

这些分享为我们提供了丰富的参考和灵感，使我们能够在实际设计中借鉴成熟的方法，而不是从零摸索。

## 二、正文

首先，先来看看黄佳老师总结的几个Agent Skills设计模式，老师总结了4个设计模式：

- **工具隔离模式**
- **脚本增强模式**
- **模板驱动模式**
- **知识分层模式**

**第1个模式**：模板驱动模式的核心是用模板强约束输出结构，让结果稳定、可对比、可自动解析。适用于报告生成、文档输出等需要格式一致性的场景。它解决的是“输出不稳定”的问题，本质是把自然语言生成转化为结构化接口。

比如，我们的有些Skills最终输出的结果希望是一个固定格式的内容情况下可以使用，比如一个固定的HTML格式、一个固定MD格式等等。

举个例子，我们希望做一个周报 Skill：

```powershell
.agents/skills/report/
├── SKILL.md              # 主文件
└── templates/
​    ├── weekly_report_tpl.md   # 周报模板
```

模板驱动的价值在于把输出格式标准化，使结果具备一致性、可比较性和可自动处理能力。它将“生成内容”与“结构定义”分离，让 Skill 更易维护，也让后续流程（例如自动汇总、比对或系统导入）更加稳定可靠。

**第2个模式**：脚本增强模式的核心是把计算、匹配、数据转换等确定性逻辑交给脚本执行，而不是让 Claude 推理完成。适用于公式计算、正则匹配、指标统计等场景。它解决的是“结果不稳定”的问题，本质是把概率型推理替换为确定性执行。

比如我之前做的一些Skills都是通过脚本的方式作为执行层，例如图片大小修改的Skills:

```powershell
.agents/skills/image-resizer/
├── SKILL.md              # 主文件
└── script/
​    ├── resize_image.js  # 对图片进行大小、缩放、格式处理的脚本
```

对于一些Skills可能需要通过一个确定性的脚本实现一些固定的业务处理逻辑，这种确定性计算应当下沉到脚本中完成。模型和工具负责判断和理解，脚本负责计算和执行。脚本不应依赖额外的外部安装环境，优先使用标准库，若确有依赖必，须在说明文档中明确声明；脚本不应包含交互式输入，必须是一次性可执行、无人工干预的流程；同时也不要把所有逻辑都塞进脚本，只有确定性、可计算、可验证的逻辑才适合放入脚本。

**第3个模式**：知识分层模式的核心是按使用频率组织知识，高频内联，中低频按需加载。适用于规则多、领域复杂的 Skill。它解决的是“上下文膨胀”的问题，本质是通过渐进加载控制认知复杂度。

这里以某个图表可视化的Skills为例，我们可以通过在提示词中维护图表的相关触发条件，然后提示AI在什么时候加载references下的什么样的知识文件，从而实现渐进式加载与知识分层：

![image-20260406224025585](.\image\image-20260406224025585.png)

**第4个模式**：老师在分享ClaudeCode相关知识的时候，介绍了自己总结的工具隔离模式，它核心是通过 allowed-tools 明确能力边界，限制 Skill 可以调用的工具。适用于需要安全控制或职责划分的场景。它解决的是“越权风险”的问题，本质是把安全约束前置为结构设计。当你需要确保 Skill 不会做“不该做的事”时——这是安全设计，不是功能设计。

比如在ClaudeCode的Skills中，可以增加和限制Skills的处理边界，增加一些控制来处理Skills:

```powershell
\# 审计类 Skill：只读
allowed-tools: [Read, Grep, Glob]
\# 生成类 Skill：只写不改
allowed-tools: [Read, Grep, Glob, Write]
```

具隔离模式的价值不在于“能做什么”，而在于明确“不能做什么”。

老师的经验是四种模式不是互斥的——一个成熟的 Skill 通常组合使用多种模式。但理解每种模式的核心思想和适用边界，能帮你在设计时做出更好的取舍。

接下来再看看Anthropic公司出品的Skills经验分享，内容来源于Anthropic的The-Complete-Guide-to-Building-Skill-for-Claude内容。

Anthropic分享的第一个设计模式为：串行工作流编排模式。

![image-20260406224037573](.\image\image-20260406224037573.png)

目前这个模式是很多Skills核心使用的模式，也是SOP风格式的Skills的写法，这类Skills通常会非常明确的指定多个处理的步骤，每一个步骤的如何执行与先后顺序，Skills.md类似如下：

```powershell
-# Workflow: Onboard New Customer
   --# Step 1: Create Account
   Call MCP tool: `create_customer`
   Parameters: name, email, company
   --# Step 2: Setup Payment
   Call MCP tool: `setup_payment_method`
   Wait for: payment method verification
   --# Step 3: Create Subscription
   Call MCP tool: `create_subscription`
   Parameters: plan_id, customer_id (from Step 1)
   --# Step 4: Send Welcome Email
   Call MCP tool: `send_email`
   Template: welcome_email_template
```

这种情况下，可以指定每个步骤的执行顺序，步骤的依赖关系，每个步骤的校验、输入与输出等，失败情况的处理办法。

再来看下Anthropic分享的第2个模式：多 MCP 协调模式。

![image-20260406224243728](.\image\image-20260406224243728.png)

**核心思想：** 将复杂工作流拆解为跨越多个独立服务（或工具）的离散阶段，Agent 充当中央调度器（Orchestrator）。

你可以将这种模式直接类比为后端开发中的**微服务编排（Microservices Orchestration）**。在开发复杂的 AgentSkills 时，一个完整的业务流往往需要跨越多个独立的 API 或平台。

1. **图片案例拆解（设计到开发的交接）：**
2. **Phase 1 (Figma MCP):** 负责“读”和“提取”。Agent 从设计工具中导出资源和规范。
3. **Phase 2 (Drive MCP):** 负责“存储”。Agent 将上一步拿到的资源上传到云盘，并获取共享链接。这里发生了关键的**上下文传递（Data passing）**。
4. **Phase 3 (Linear MCP):** 负责“执行任务分配”。Agent 拿着上一步的链接，在项目管理工具中创建开发任务。
5. **Phase 4 (Slack MCP):** 负责“通知”。将所有前置步骤的汇总信息发送给工程团队。
6. **工程视角的关键技术（Key techniques）：**
7. **清晰的阶段分离 (Clear phase separation)：** 必须遵循单一职责原则，Figma 的 Skill 绝不应该去处理 Slack 的发信逻辑。
8. **集中式错误处理 (Centralized error handling)：** 如果 Phase 2 上传云盘失败，Agent 需要有全局视野来决定是重试、还是向用户报错，而不是让流程卡死。
9. **流转前的校验 (Validation before moving to next phase)：** 这是防止级联失败的关键。在创建 Linear 任务前，必须校验 Drive 链接是否真的生成了。

再来看下Anthropic第3个模式：（迭代优化模式）

**核心思想：** 承认大模型很难在 Zero-shot（一次性提示）中完美解决复杂问题，因此引入**生成 -> 校验 -> 修正**的闭环。

![image-20260406224328384](.\image\image-20260406224328384.png)

在实际场景中，比如让 OpenClaw 生成一份深度报告或编写一段复杂的代码，与其堆砌极其复杂的初始 Prompt，不如利用这个循环模式。

1. **图片案例拆解（报告生成）：**
2. **初稿阶段 (Initial Draft)：** 通过 MCP 获取数据，快速生成第一版草稿并落盘。
3. **质量检查 (Quality Check)：** 这是该模式的灵魂。图片中提到运行 scripts/check_report.py。这意味着**验证逻辑被外包给了确定性的代码脚本**（比如检查缺失章节、数据一致性等），而不是完全依赖大模型“自己检查自己”，从而避免了幻觉。
4. **优化循环 (Refinement Loop)：** Agent 接收脚本跑出来的 Issue 列表，针对性地修改对应的段落，然后再去跑一遍脚本。
5. **最终化 (Finalization)：** 当且仅当质量阈值达标（比如跑通了所有测试用例），才进行最终排版和输出。
6. **工程视角的关键技术（Key techniques）：**
7. **明确的质量标准 (Explicit quality criteria)：** 必须有客观的、可被机器评估的标准。
8. **知道何时停止迭代 (Know when to stop iterating)：** 这是最容易踩坑的地方。必须在系统层设置最大迭代次数（Max Iterations），或者在校验脚本中定义绝对的“通过”条件，否则 Agent 极易陷入死循环（Infinite Loop）。

如果说**模式二（多 MCP 协调）解决的是 Agent 能力的广度**问题（如何有条不紊地操作多个外部系统），那么**模式三（迭代优化）解决的就是 Agent 输出的深度和质量**问题（如何通过工程化的 Loop 逼近人类专家的交付标准）。

接下来再看下Anthropic公司分享的第4个模式：Context-aware tool selection（上下文感知工具选择模式）

**核心思想：** 面对同一个业务目标（Outcome），Agent 能够像有经验的工程师一样，根据运行时的上下文（Context）动态路由，选择最合适的底层工具。

这在 Java 开发中非常熟悉，本质上就是 **策略模式 (Strategy Pattern)** 或 **工厂模式 (Factory Pattern)** 的 Agent 化实现。接口是统一的（比如 save_data()），但具体的实现类是动态派发的。

![image-20260406224343445](.\image\image-20260406224343445.png)

**图片案例拆解（智能文件存储）：**

1. **决策树 (Decision Tree)：** 这是该模式的核心。Agent 在调用任何存储 API 前，先进行条件评估（大小、类型）。大于 10MB 走对象存储，代码文件走 GitHub，临时文件放本地。
2. **执行存储 (Execute Storage)：** 根据决策树的结果，动态挂载对应的 MCP 工具，并注入特定平台所需的元数据（Metadata）。
3. **向用户提供上下文 (Provide Context to User)：** 这一步在 UX（用户体验）上极其关键。所谓的“Explainability（可解释性）”，Agent 必须告诉用户*为什么*做了这个选择（“因为文件过大，我已将其存入云盘”），从而建立信任感。
4. **工程视角的关键技术：**
5. **清晰的决策标准 (Clear decision criteria)：** 路由规则必须是确定性的。
6. **降级与回退 (Fallback options)：** 如果首选的 Notion API 宕机了，Agent 需要知道降级策略（比如暂存本地并告警）。
7. **AgentSkills 应用映射：** 假设你在为 OpenClaw 编写一个信息检索的 Skill，你就不该每次都无脑调用 Tavily Search。你可以设计一个 Context-aware 模式：如果是简单的基础常识，直接走模型内置知识或轻量级本地检索；如果是实时的深度技术文档分析，再动态路由并触发 Tavily 搜索。

接下来再看下Anthropic公司分享的第5个模式：Domain-specific intelligence（领域特定智能模式）

领域特定模式要求将**业务规则、行业Know-how 和前置合规逻辑**硬编码到 Skill 的工作流中。

![image-20260406224357498](.\image\image-20260406224357498.png)

1. **图片案例拆解（金融支付合规）：**
2. **处理前置检查 (Before Processing)：** 在真正调用支付接口前，强行插入合规逻辑（查制裁名单、校验管辖权辖区、评估风险等级）。这把该 Skill 从一个“支付工具”提升为了一个“合规风控专家”。
3. **分支处理 (Processing)：** 基于前置检查的结果执行 IF/ELSE 分支。**注意这里的设计：** 如果合规未通过，Agent 并没有直接崩溃或抛出异常，而是优雅地转向“标记人工审核 (Flag for review)”并生成工单。
4. **审计追踪 (Audit Trail)：** 将所有的决策链路、检查项作为日志落盘。
5. **工程视角的关键技术：**
6. **将领域经验嵌入逻辑 (Domain expertise embedded in logic)：** 也就是把行业专家的“SOP（标准作业程序）”翻译成了 Agent 的执行流。
7. **行动前合规 (Compliance before action)：** 永远在产生破坏性/实质性动作（如转账、删库、发布代码）前进行拦截校验。
8. **多智能体（MAS）应用映射：** 在构建复杂的多智能体系统时，这通常体现为**专员智能体**的设计。比如开发一个代码 Review Skill，它不仅仅是调用 GitHub API 获取 PR Diff，而是内置了“审查是否存在硬编码密码、是否符合某种特定的代码规范”等领域智能。

综合这两张图，高质量的 Agent 架构正在迅速抛弃早期的“大力出奇迹”式的长 Prompt 模式。

1. **模式四** 让 Agent 具备了**灵活性（动态路由）**，节省了资源。
2. **模式五** 让 Agent 具备了**安全性（前置风控）**，能够真正落地到严谨的企业级生产环境中。

同样的谷歌近期也发布了Skills的设计模式，资料来源：https://x.com/GoogleCloudTech/status/2033953579824758855

5个模式分别为：

1. **工具包装器模式**（Tool Wrapper）：将调用约定、最佳实践或API封装为可复用模块
2. **审查者/评论家模式**（Reviewer / Critic）：生成与审查逻辑解耦，通过循环机制保证质量
3. **反转询问模式**（Inversion）：Skill 在执行前主动“面试”用户，获取完整上下文
4. **结构化生成器模式**（Generator）：利用模板和质量门禁保证输出一致
5. **流水线与协调者模式**（Pipeline & Coordinator）：分解复杂任务并动态路由到专门化子智能体

这些模式核心都是：将开发者的经验“编译”成结构化 Skill，让智能体具备专业、可组合的能力。

详情参考 我上一次的文章[Skills 第二节 - 5 Agent Skill Design Patterns.md](Skills%20%E7%AC%AC%E4%BA%8C%E8%8A%82%20-%205%20Agent%20Skill%20Design%20Patterns.md)

## 三、总结

通过对各类 Skills 设计模式的梳理，我们可以发现，成熟的 Skills 并非依赖大模型的奇迹式输出，而是依赖工程化的设计与结构化的流程：

1. 某些模式解决了输出稳定性和流程可控性的问题
2. 某些模式让 Agent 灵活、节省资源
3. 某些模式让 Agent 安全、合规，并可落地到企业级生产环境

理解这些模式的核心思想和应用边界，不仅能帮助我们设计更高质量、更可靠的 Skills，还能让我们从“依赖模型”转向“工程化构建智能体”的思路，真正把经验和业务逻辑落到实处。