---
description: 本篇整理了 Cloud Code 在真实开发中最常用、最强的 30 个高阶技巧，涵盖上下文管理、MCP（模型上下文协议）扩展、IDE 联动、权限系统、自动化、子代理、命令扩展、版本回溯、批处理能力等全栈工程化能力。掌握这些技巧，你就能把 Cloud Code 从“一个聊天 AI”升级为 可写代码、可执行命令、可管理项目、可自动化任务的全能开发副驾。
title: Cloud Code 高阶技巧一览
tag:
  - AI
  - Claude code
sidebar: true
comment: true
recommend: 1
---
# 30个Cloud Code高阶技巧

## INIT

`\INIT`命令让Cloud Code通读所有文件，保存项目知识到`.claude.md`文件。



## COMPACT 

`\COMPACT`压缩上下文，提高AI专注力，降低token消耗。



## CLEAR

 `\CLEAR`清除对话记录，开始新任务时使用。



## think think hard

使用`think think hard`等增加AI思考长度，官方支持。



## 叹号`!`

叹号`!`切换到命令行模式，执行临时命令，结果加入上下文。

![image-20251008180835970](..\image\image-20251008180835970.png)

## 井号`#`

井号`#`进入记忆模式，记录为文件，作为AI长期记忆，可选择项目或用户级别。



## IDE

`\IDE`命令打通Cloud Code与IDE（如VS Code），实现代码感知和修改对比。

![image-20251008181117906](..\image\image-20251008181117906.png)

## claude -p

非交互模式使用`cloud -p`，使Cloud Code成为命令行AI助手。

![image-20251008181244656](..\image\image-20251008181244656.png)



## claude mcp add

MCP (模型上下文协议) 作为AI与外部工具的中间层，使用`cloud mcp add`安装，可选择项目或用户级别。

![image-20251008181902443](..\image\image-20251008181902443.png)

![image-20251008182134414](..\image\image-20251008182134414.png)

## claude mcp remove

使用`cloud mcp remove`删除MCP server。



## permissions

 `\permissions`自定义权限规则，允许或禁止Cloud Code使用工具或MCP。



## --dangerously-allow-permissions

 启动Cloud Code时添加`--dangerously-allow-permissions`赋予最高权限。



## .cloud/commands

可自定义命令，在`.cloud/commands`文件夹下创建文件，文件名为命令名。



## settings.json

Hook让Cloud Code在特定节点执行操作，如使用Prettier检查代码格式，通过`settings.json`配置。





## agents

Sub Agent类似子线程，可并行执行多个子任务，提高效率，通过`\agents`创建。





## resume

 `\resume`找回历史话题，敲击ESC跳转到具体对话，cc-undo可回退对话和代码。



## export

`\export`导出对话内容，可保存或用于其他AI验证。



##  CLAUDIA

 CLAUDIA是基于Cloud Code的可视化桌面应用，提供安装包，需配置API环境变量（如CCR）。





## CLAUDIA

CLAUDIA提供时间导航线和检查点功能，可回退文件改动和对话历史。