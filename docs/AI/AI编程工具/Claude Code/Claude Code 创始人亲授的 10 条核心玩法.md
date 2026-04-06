---
description: 本篇整理了Claude Code创始人亲授的10条核心玩法，涵盖多任务并行、计划模式、团队知识库、自定义命令、Bug自动修复、高级Prompt技巧、终端优化、子代理应用、数据分析及学习模式等实战技巧。掌握这些方法，你将把Claude Code从简单的聊天AI升级为可写代码、可执行命令、可管理项目、可自动化任务的全能开发副驾。
title: Claude Code创始人亲授的 10条核心玩法
tag:
  - AI
  - Claude Code
sidebar: true
comment: true
recommend: 3
---
# Claude Code创始人亲授的 10条核心玩法

Claude Code 的创始人在他的社交媒体上分享了来自 Claude Code 团队官方的使用的 10 个小技巧，赶紧来学习，Get 新技能。

![image-20260310001618657](..\image\image-20260310001618657.png)

### 1、多开并行干活，效率翻倍

同时开 3–5 个独立的代码目录（用 git worktree 最方便），每个目录里跑一个独立的 Claude 会话。

一个修 bug、一个写新功能、一个看日志分析问题……彻底不等了。

![image-20260310001721028](..\image\image-20260310001721028.png)

### 2、复杂需求先写计划，别直接让它 coding

先让 Claude 出详细计划（Plan Mode），你多花时间改计划，直到靠谱了再让它一次性实现。

一个 Claude 写计划 ， 另一个 Claude 当资深工程师审计划， 一旦跑偏，立刻回到计划阶段重来，别硬推。

![image-20260310001735493](..\image\image-20260310001735493.png)

### 3、建一个 CLAUDE.md，当成团队知识库

每次 Claude 犯错，就立刻说：把这个错误记到 CLAUDE.md 里，以后别再犯了。

Claude 特别擅长给自己写规则，持续迭代这份文件，出错率会明显下降。

![image-20260310001744638](..\image\image-20260310001744638.png)

### 4、重复的事就封装成 Skill 或斜杠命令，用 Git 管理

一天干超过一次的操作，就做成 skill 或 /xxx 命令，提交到 git。

例如 /techdebt（扫描重复代码）、一键拉 7 天 Slack+GitHub 上下文、自动写测试等。

![image-20260310001752864](..\image\image-20260310001752864.png)

### 5、Bug 直接丢给它修，别手把手教

把 Slack bug 讨论直接丢给它，说 fix 就行，或者 “Go fix the failing CI tests”。

分布式系统出问题，直接给 docker logs 让它排查。

开启 Slack MCP 后效果更强。

![image-20260310001814962](..\image\image-20260310001814962.png)

### 6、Prompt 写得越高级越好，逼它出好代码

- 让它当考官：Grill me on these changes，不通过别提 PR
- 修复一般就说：Knowing everything you know now, scrap this and implement the elegant solution
- 写超详细 spec 减少歧义
- 让它对比 main 和 feature 分支，证明改动真的有效

![image-20260310001822754](..\image\image-20260310001822754.png)

### 7、选择好的终端工具

团队最爱 Ghostty。

用 /statusline 自定义状态栏显示上下文用量和 branch。

给终端 tab 配色+命名（一个任务一个 tab）。

强烈推荐语音输入（macOS fn 连按两下），说话比打字快 3 倍，prompt 也能写得更细。

![image-20260310001905469](..\image\image-20260310001905469.png)**

### 8、善用 Subagents（子代理）

复杂任务后面加一句 “use subagents”，让 Claude 投入更多算力。

把独立小任务外包给 subagent，保持主上下文干净。

还可以设置 hook 把权限请求路由给 Opus 4.5 自动审批安全操作。

![image-20260310223933917](..\image\image-20260310223933917.png)

### 9、数据分析直接交给 Claude

让它用 bq CLI 现场跑 BigQuery 查询，拉数据分析。

团队有个共享 BigQuery skill，人人复用。

任何带 CLI/MCP/API 的数据库都能这么玩。

![image-20260310223918085](..\image\image-20260310223918085.png)

### 10、把 Claude 当老师

打开 explanatory/learning 模式，让它解释为什么这么改

让它生成 HTML 幻灯片讲解陌生代码

让它画 ASCII 图解释架构

建间隔重复学习：**你讲理解 → 它追问补漏 → 存下来复习**

![image-20260310223908789](..\image\image-20260310223908789.png)

最后，使用 Claude Code 没有唯一标准答案，每个人的配置与使用场景都各有差异，要多去尝试，摸索出最适合自己的方式。