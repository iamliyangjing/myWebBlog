---
description: 本文介绍开源项目 Understand Anything——一个基于 Claude Code 的多智能体插件，能够将任意代码库转化为交互式知识图谱。通过静态代码分析与大语言模型结合，它自动提取函数、类与依赖关系，生成可视化数据看板，并支持语义搜索、变更影响分析、引导式学习路径等实用功能。文章以 CRMEB 商城项目为例，演示了从安装、分析到使用的完整流程，并讨论了其适用场景、当前局限与发展潜力，为开发者提供一种从“盲读代码”转向“结构化理解系统”的全新方式。
title: Understand Anything - 用 AI 多智能体将代码库转化为可探索的知识图谱
tag:
  - AI
  - 效率提升
sidebar: true
comment: true
recommend: 1
---

## 一、前言

在软件开发的世界里，代码量往往是令人望而生畏的迷宫。当我们初次加入一个新团队，面对成百上千的文件、数十万行代码，自己是否曾感到无助？文档零散、逻辑复杂，任何新功能的开发都像一次“考古”探险——每一行代码都可能隐藏着未知的陷阱。

正是在这样的困境下，最近有个开源项目**Understand Anything**诞生了。**它可以将任意代码库转化为可探索、可搜索、可对话的交互式知识图谱，**是一个基于 [Claude Code](https://docs.anthropic.com/en/docs/claude-code) 的插件，通过多智能体（multi-agent）架构分析你的项目，构建包含文件、函数、类以及依赖关系的知识图谱，并提供一个可视化交互界面，帮助你理解整个系统。不再“盲读代码”，而是从全局视角理解系统结构。

它并不是普通的工具，而是一种理解代码的全新方式——让你不再盲读代码，而是能够从宏观的视角掌握整个系统。它的使命，就是让复杂的代码库变得清晰、可探索，并且充满智慧。

今天分享下这个开源项目，官网为：

https://lum.is-a.dev/Understand-Anything/

开源的Github地址为：

>  https://github.com/Lum1104/Understand-Anything

目前这个项目开源不到一个月，已经有了6.8K：

![image-20260330230622145](E:\CodeRepositroy\myWebBlog\docs\AI\AI编程工具\Agent\image\image-20260330230622145.png)

**Understand Anything**是一款将任意代码库转化为**可探索、可搜索、可对话的交互式知识图谱**的工具。的核心是结合大语言模型（LLM）与静态代码分析技术，通过多智能体架构深入分析项目，为你构建一个可视化的、可交互的知识地图。

## 二、正文

**为什么我们需要它？**

当我们阅读新系统代码的时候，阅读代码已经很难了，理解整个系统更难。文档往往过时，上手周期长达数周，新功能开发像考古。Understand Anything 通过结合 **大语言模型（LLM）与静态代码分析**去生成一个**动态、可探索的代码知识地图** — 并提供自然语言解释。

想象一下，你可以轻松点击每个函数、类和文件节点，立即看到它的作用、依赖关系以及简明的自然语言解释。对于入门开发者，它像一位耐心的导师，引导你循序渐进地理解整个系统；对于产品经理和设计师，它像一本透明的手册，让你无需阅读代码，也能洞察系统的逻辑；而对于 AI 协同开发者，它提供了深入的分析接口，让你的智能工具真正理解你的项目。

**目前适用哪些人群？**

**👩‍💻 入门级开发者：**不再被陌生代码淹没。通过结构化引导逐步理解系统架构，每个函数和类都有简明易懂的解释。

**📋 产品经理 & 设计师：**无需阅读代码，也能理解系统逻辑。比如直接提问：“认证流程是怎么实现的？” 便可获得基于实际代码库的清晰答案。

**🤖 AI协同开发者：**让你的 AI 工具深入了解你的项目。在代码审查之前使用/understand-diff，在深入任何模块时使用/understand-explain，或在架构分析中使用 /understand-chat。

核心功能如下：

![image-20260330230711045](E:\CodeRepositroy\myWebBlog\docs\AI\AI编程工具\Agent\image\image-20260330230711045.png)

接下来我使用ClaudeCode、MiniMax2.7、开源商城项目：CRMEB商城JAVA版来分享下这个东西。

首先，我们更新为最新版本的Claude Code：

>  npm install -g @anthropic-ai/claude-code

然后通过cc switch软件，设置ClaudeCode的模型为MiniMax2.7模型，这里我购买了CodingPlan计划，目前是比较合适的：

![image-20260330230751732](E:\CodeRepositroy\myWebBlog\docs\AI\AI编程工具\Agent\image\image-20260330230751732.png)

我来访问我自己的RAG 项目，是一个很简单的单体项目

![image-20260330231053099](E:\CodeRepositroy\myWebBlog\docs\AI\AI编程工具\Agent\image\image-20260330231053099.png)

接下来我们在这个项目下打开ClaudeCode的终端窗口，安装下这个Understand Anything插件：

> /plugin marketplace add Lum1104/Understand-Anything
>
> /plugin install understand-anything

![image-20260330231146475](E:\CodeRepositroy\myWebBlog\docs\AI\AI编程工具\Agent\image\image-20260330231146475.png)

安装完成后，我们这里重启下插件，执行reload-plugins命令，然后看到：

![image-20260330231205279](E:\CodeRepositroy\myWebBlog\docs\AI\AI编程工具\Agent\image\image-20260330231205279.png)

如果没有看到可以直接ctrl+C然后claude -c重进。

接下来可以看到Claude Code开始执行这个/understand的命令，它开启了5个并行的subagent，然后开始阶段1和阶段2的分析：

![image-20260330231227509](E:\CodeRepositroy\myWebBlog\docs\AI\AI编程工具\Agent\image\image-20260330231227509.png)

这个步骤比较耗时，当前我下载的这个RAG项目中的java文件大概有100多个，Claude Code帮我搞了40个多个批次去执行，多智能体（multi-agent）架构会：扫描你的项目，提取函数 / 类 / 依赖，构建知识图谱保存至.understand-anything/knowledge-graph.json

/understand 命令调用 5 个 agent：

![image-20260330231319354](E:\CodeRepositroy\myWebBlog\docs\AI\AI编程工具\Agent\image\image-20260330231319354.png)

生成完成后，实际上再本地会产生一个知识图谱的JSON文件，如果我们后续需要自己二次创作也是OK的：

接下来我们继续，我们输入如下命令查看数据看板：

>  /understand-dashboard

打开交互式网页数据看板，您的代码库将以图表形式呈现 — 按架构层级进行颜色编码，支持搜索和点击。选择任意节点即可查看其代码、关系以及简明易懂的解释。

执行命令后，会在浏览器中打开如下界面：

目前看了一些，目前这个软件应该是正在处于发展期，从一些效果层面，还没有到Neo4j这种，但是基本上把一个系统的关系进行了展示。

可以询问下相关内容：

>  询问任意代码库的问题
>
> /understand-chat  购物车模块的相关逻辑

如下：

还有其他可以使用：

> 分析当前修改的影响
>
> /understand-diff
>
>  深入理解某个文件
>
> /understand-explain src/auth/login.ts
>
> 为新团队成员生成指南
>
> /understand-onboard

例如让它可以让他给我解释某个模块文件：

我们可以执行/understand-onboard这个技能，然后让它根据我们的需要给我们写一写指南，例如：

这个运行完成后，可以看下文档，文档中也包含一些学习的路径：

![image-20260330231513284](E:\CodeRepositroy\myWebBlog\docs\AI\AI编程工具\Agent\image\image-20260330231513284.png)

我们也可以根据自己的需求进行提问。

根据作者仓库的介绍，它的功能不仅仅停留在可视化。**语义搜索**可以帮你迅速找到项目中处理身份验证的部分；**变更影响分析**让你在提交修改前知道可能波及的模块；**引导式学习路径**为你梳理系统架构；甚至根据不同角色自动调整界面，让每一位使用者都能高效上手。

目前我感觉当前有一些大的代码仓库，可能不太适合用这个。

同时这种情况也需要一个厉害的模型，例如OPUS4.6，当前MiniMax还是有些欠缺，这个开源项目还需要发展一下，不过目前提供的一些能力也是能够满足了一些简单的场景。

## 三、总结

在代码的世界里，理解胜于盲从。**Understand Anything**不仅仅是一个插件，它是一座桥梁，让开发者、在复杂的系统中找到方向。它将迷宫般的代码转化为可探索的知识图谱，把抽象的逻辑变成触手可及的可视化信息。
