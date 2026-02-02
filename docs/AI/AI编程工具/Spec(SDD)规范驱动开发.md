---
description: 本文系统阐述了Spec规范驱动开发（SDD）的核心流程，通过意图定义（spec.md）、技术规划（plan.md）、任务分解（tasks.md）、自动化实现四个递进阶段，将模糊需求转化为结构化规范，并借助AI Agent实现从设计到代码的自动化生成。文章对比了传统开发与SDD的差异，解答了遗留系统集成、行业术语理解等常见问题，并提供了工具推荐与实践指南，为开发者提供一套可复用、可协作的AI原生开发方法论。
title: Spec（SDD）规范驱动开发：从意图到代码的AI编程新范式
tag:
  - AI
  - 效率提升
sidebar: true
comment: true
recommend: 1
---

## 一、前言



之前分享了一点关于Spec规范驱动开发的实践经验和整理，最近几个月无论是前端还是后端，都已经针对特定的任务场景、开发场景进行Spec规范驱动编程，通过一份高质量的、可被机器精确理解的规范（Specification），驱动整个开发工作流的开始。也有老师说它是AI时代，AI 原生开发的“第一性原理”，很合适。

## 二、正文

2025年，每个人学会运用它的核心产物—— spec.md、plan.md 和 tasks.md能够极大的提供AI编程的质量和效率。

技术大佬在前几个月技术大会上的分享截图：

![image-20260203002716535](.\image\image-20260203002716535.png)

AI编程中的常见问题让我们感觉很真实：

![image-20260203002726813](.\image\image-20260203002726813.png)

而Spec工作流解决了AI变成开发的三个痛点：

![image-20260203002734450](.\image\image-20260203002734450.png)

也正是因此Spec开发，会让我们更加和标准研发流程产生联系。

![image-20260203002740736](.\image\image-20260203002740736.png)

再来看下Vibe Coding和Spec的对比：

![image-20260203002751059](.\image\image-20260203002751059.png)

Spec 驱动开发（Spec-Driven Development）这个东西把整个从“想法”到“代码”的过程拆成 **四个递进阶段**。每个阶段只关心一个核心问题，并产出一个关键文档。四个文档首尾相接，构成一个完整的“从意图到实现”的流水线。

**⭐ 阶段 1：意图定义（Intent Definition）**

🎯 目标

这是把“灵感”变成“需求”的阶段。把“我想做一个 XXX”变成“这个东西到底为谁服务、解决什么问题、怎么判断做成功了”。回答两个最关键的问题：

1. **WHAT**：要做什么？
2. **WHY**：为什么要做这件事？

📝 输入

1. 开发者或产品经理给出的一个模糊想法，（往往只有一句话，比如：“做个会员系统吧”）

🔍 核心活动

1. 人与 AI 一起头脑风暴，挖边缘场景（用户可能怎么用？会不会出问题？），模糊点全部问清楚，写出明确的 **验收标准**（做完必须满足什么）

📄 输出产物：spec.md（纯需求规范）

特点：

1. 只讲“需求”“用户故事”“业务目的”，**不谈技术**（不谈框架、不谈数据库、不谈语言），之后所有阶段都必须严格遵守这个文档

🪄 示例如下：

初始想法：

“我们需要一个任务提醒功能。”

整理成 spec.md 中的需求示例：

1. 用户可以创建一个任务，并设置截止时间
2. 系统需要在截止前 10 分钟通知用户
3. 边缘场景：时间过去了怎么办？重复任务如何处理？
4. 验收标准：
5. 用户确实能收到通知
6. 不会出现多次重复通知

**⭐ 阶段 2：技术规划（Technical Planning）**

🎯 目标：把“要做什么”翻译成“应该怎么实现”。这玩意儿在技术上怎么做？就像“客户点菜 → 厨师拿到菜谱”的过程。

📝 输入

1. spec.md
2. 技术栈限制（语言、框架、公司已有基础设施）

🔍 核心活动

1. AI Agent 做工程决策：
2. 技术选型
3. 架构设计
4. 模块拆分
5. 数据结构设计
6. API 设计
7. 要遵守项目的“constitution.md”（工程规则、风格、原则）

📄 输出产物

1. plan.md（技术方案）
2. 附属文档：
3. data-model.md（数据模型）
4. api-spec.json（API 契约）
5. …任何必要的技术设计文档

🪄 示例

1. 用 PostgreSQL 存任务
2. 用 REST API 提供任务查询
3. 提醒功能用系统级 cron 或后台 job 处理
4. API 格式：
5. POST /tasks
6. GET /tasks/:id
7. POST /tasks/:id/remind

**⭐ 阶段 3：任务分解（Task Breakdown）**

🎯 目标：一步一步具体要做什么？这是“把做菜步骤写成菜谱”的阶段。每一步都必须明确到“会做饭的机器人能直接执行”。

📝 输入

1. plan.md
2. 所有附属设计文档

🔍 核心活动

1. AI Agent 分析整个技术方案
2. 拆成一系列 **原子化任务（粒度足够小、可直接执行）**
3. 识别任务之间的：
4. 依赖关系
5. 可并行执行的部分

📄 输出产物：tasks.md

特点：

1. 机器友好
2. 任务必须明确可执行
3. 如：
4. 创建文件
5. 实现函数
6. 编写接口
7. 创建 schema
8. 写测试

🪄 示例输出

1. 创建 models/task.py
2. 实现 Task 数据模型
3. 创建 /tasks API endpoint
4. 实现提醒 job 的 cron 配置
5. 编写提醒功能的单元测试

**⭐ 阶段 4：自动化实现（Automated Execution）**

🎯 目标

完成所有任务，产出实际的软件。这是“机器人开始照着菜谱做菜”的阶段。人类负责尝尝味道、确认没有下错料。

📝 输入

1. tasks.md

🔍 核心活动

1. AI Agent 按照 tasks 一项项执行：
2. 写文件
3. 写代码
4. 运行测试
5. 编辑文档
6. 开发者主要负责：
7. 监督
8. 审批（尤其是破坏性操作）
9. 最终验收

📄 输出产物

1. 可运行的代码
2. 测试套件
3. 文档
4. 配置文件等

在之前分享的Github开源的spec-kit工具包，是规范驱动开发思想的一个杰出实现。解决了项目从0-1的建设问题，目前也有openspec能够较好的解决项目从1-N的建设开发问题。

在SpecKit提供的工具包中提供了相关的斜杠命令：

**意图输入（/speckit.specify）：**你向 AI Agent 输入一个自然语言的想法：“Build an application that can help me organize my photos...”。“编译”成规范（spec.md）：AI Agent 扮演产品经理的角色，与你互动（或基于内置模板），将你的模糊想法“编译”成一份结构化的 spec.md。同时，它会自动为你创建一个新的 Git 分支，如 001-photo-albums。

**技术选型（/speckit.plan）：**你进入技术决策阶段，告诉 AI：“The application uses Vite…vanilla HTML, CSS, and JavaScript…SQLite database.”。“编译”成方案 (plan.md)：AI Agent 扮演架构师的角色，将技术选型与 spec.md 的需求结合，生成一份详尽的 plan.md。

**生成任务列表 (/speckit.tasks)：**你下达指令，不需额外参数。“编译”成“字节码” (tasks.md)：AI Agent 扮演技术组长的角色，它读取 plan.md，将其分解为上百个具体的、带依赖关系和并行标记的原子任务，生成 tasks.md。

**执行实现 (/speckit.implement)：**你下达最终执行指令。

传统的软件开发是一个以代码为中心的 Code -> Test -> Refactor 循环。始终围绕着“实现”这个层面。当需求发生较大变更时，这个小循环往往需要被打破，回到更上游的设计阶段，成本很高。

而在 SDD 范式下，开发循环被提升到了一个更高的维度，变成了一个以“规范”为中心的 Spec -> Generate -> Validate 循环。SDD 通过结构化的规范，将模糊的自然语言，转化为 AI 可以精确执行的“机器语言”，极大地提升了输出的可靠性。你不再需要祈祷 AI“猜对”你的心思，而是通过一份严谨的规范来精确地“编程”AI 的行为。

## 三、常见问题

有网友在其他网站上问：

> （1）、如果使用spec生成的代码，在测试阶段发现实现bug，我是应该修改spec相关文档，让其自主修复？还是直接手动修复代码？

大佬回答我觉得也可以参考：

是取决于 Bug 的性质：是“没想清楚”还是“没写对”。

如果是“意图”错了（没想清楚）：修改 Spec，比如你发现业务逻辑本身有漏洞，或者漏掉了某个边缘场景。这样，最好修改 spec.md，然后让 Agent 重新生成。

如果Spec 写得很清楚，但 AI 生成的代码有逻辑错误、语法错误或性能问题，那么让 Agent 修复代码。这时 Spec没错，你应该把错误日志、失败的测试用例反馈给 Agent，指令它：“实现有误，请根据这个错误信息修复代码”。

总之，一个原则尽量把握住：永远维护“意图”的单一来源。 只有当 Bug 源于意图偏差时才改 Spec，否则让 Agent 去修代码。

> （2）如果利用SDD，面对的是遗留系统上进行新功能开发，这种场景的最佳实践是什么？以及如何能解决以下问题？可能会遇到以下问题:
>
> 新功能是需要基于/调用现有遗留系统的组件提供的API进行功能开发，而非采用重新实现的方式，Agent如何才能做到准确的调用？
>
> 需求内容往往会含有特定行业/应用的“黑话”、“行话”等私有的内部术语描述，而大模型一般是无法进行理解的，Agent如何才能准确理解意图并推进后续步骤执行？

大佬回答如下：

问题一：核心还是在于“上下文注入”。    

提供接口文档： 如果有遗留系统组件的 Swagger/OpenAPI 文档，直接用 @ 喂给 Agent。

提供示例代码： 找一段现有的、调用了相关 API 的老代码，用 @ 喂给 Agent 作为 Few-Shot 示例。Agent 的模仿能力极强，它会照猫画虎。

另外，也可以讲遗留系统代码（如果可以的话）提供给ai agent，像Claude Code可以自己去理解 API 用法。

问题二：

可以在项目级 CLAUDE.md 中专门开辟一个 ## Terminology 章节，把“黑话”翻译成通用的技术语言（例如：“‘打桩’ = 创建 Mock 对象”）。或者

建立docs/term.md，然后在Claude.md中引用。 作为长期记忆： 这样每次启动会话，coding Agent 都会自动加载这份“字典”，从而能听懂你的行话。

这里我个人的补充如下：

这种情况很多项目都会遇到，这个时候我们就需要持续的在项目下逐步创建各个模块的知识库MD文档，包括了架构设计、数据模型、模块接口、术语库、开发规范等，同时可以维护一下个人的记忆偏好等内容。

**SDD利用结构化的规范作为人与 AI 之间的高效沟通桥梁，将 AI 编程从"凭感觉的概率性抽奖"转变为"有章可循的确定性工程"**。

## 四、资料整理

1、https://blog.csdn.net/kof820/article/details/153346393

2、https://time.geekbang.org/column/intro/101089301

3、[https://www.bilibili.com/video/BV1aEHCz1EgZ/?spm_id_from=333.337.search-card.all.click&vd_source=adc21d057db34d730f6d05a77edc26f8](https://www.bilibili.com/video/BV1aEHCz1EgZ?spm_id_from=333.337.search-card.all.click&vd_source=adc21d057db34d730f6d05a77edc26f8)

4、GitHub Spec Kit 操作指南:[https://www.bilibili.com/video/BV1iSp4zVEPk/?spm_id_from=333.337.search-card.all.click&vd_source=adc21d057db34d730f6d05a77edc26f8](https://www.bilibili.com/video/BV1iSp4zVEPk?spm_id_from=333.337.search-card.all.click&vd_source=adc21d057db34d730f6d05a77edc26f8)

5、GitHub Spec Kit 的底层功能是什么？

[https://www.bilibili.com/video/BV1bLpezSEZg/?spm_id_from=333.1387.upload.video_card.click](https://www.bilibili.com/video/BV1bLpezSEZg?spm_id_from=333.1387.upload.video_card.click)

6、https://zhuanlan.zhihu.com/p/1952308738917629967

7、https://blog.csdn.net/wwwzhouhui/article/details/153701945

8、https://cagurzhan.cn/archives/unXJ2HLN
