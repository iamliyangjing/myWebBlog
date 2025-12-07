---
description: 本指南系统介绍了 OpenAI CodeX 的四种主要运行方式（CLI、IDE 插件、SDK、云端），并提供完整的安装步骤、模型配置方法（含本地 ~/.codex/config.toml 示例）、常用命令说明、MCP 工具集成（Context7、Excel MCP）、以及在 IDE 与云端环境中的使用方式。适用于希望在本地或云端高效使用 CodeX 进行代码生成、自动化开发与项目增强的开发者。
title: CodeX 全面使用指南：安装、配置与四大运行环境详解
tag:
  - AI
  - 效率提升
sidebar: true
comment: true
recommend: 1
sticky: 1
---
# 如何使用CodeX

> Codex四种运行环境
>
> 1. Cli(命令行)
> 2. IDE插件
> 3. SDK
> 4. 云端

## 如何安装

CodeX github官网：https://github.com/openai/codex

### Installing and running Codex CLI

Install globally with your preferred package manager. If you use npm:

```
npm install -g @openai/codex
```

Alternatively, if you use Homebrew:

```
brew install codex
```

Then simply run `codex` to get started:

```
codex
```



### 配置其他模型API

Codex CLI supports a rich set of configuration options, with preferences stored in C：`~/.codex/config.toml`. For full configuration options, see [Configuration](https://github.com/openai/codex/blob/main/docs/config.md).

如果没有这个文件夹，创建一个文件命名为这个，配置上如下内容

```toml
model= "ZhipuAI/GLM-4.6"
model_provider = "modelscope"
windows_wsl_setup_acknowledged = true

[model_providers.modelscope]
name = "Modelscope"
base_url='https://api-inference.modelscope.cn/v1'
env_key= "MODELSCOPE_API_KEY"
```

在环境变量编辑上api_key

![image-20251019145540958](.\\image\image-20251019145540958.png)
 
输入codex 即可运行

![image-20251019145612953](.\\image\image-20251019145612953.png)



## 重要命令

/init 通读当前文件夹，会将读到内容生成一个AGENTS.md文件

![image-20251019150037837](.\\image\image-20251019150037837.png)

/new  新开一个对话



/compat 压缩对话的上下文，降低Token消耗



/approvals  可以调整Codex的运行权限，是否需要人工权限



/mcp 可以列出使用的mcp工具

> https://github.com/upstash/context7
>
> Context7 MCP pulls up-to-date, version-specific documentation and code examples straight from the source — and places them directly into your prompt.
>
> https://github.com/haris-musa/excel-mcp-server

添加以下内容到~/.codex/config.toml里面

```toml
[mcp_servers.context7]
args = ["-y", "@upstash/context7-mcp", "--api-key", "YOUR_API_KEY"]
command = "npx"

[mcp_servers.excel]
command = "cmd"
args = [
    "/c",
    "uvx",
    "excel-mcp-server",
	"stdio"
]
```

![image-20251019184410088](.\\image\image-20251019184410088.png)



## IDE插件

安装CodeX插件，只要本地配置了，无需再配置

![image-20251019185121600](.\\image\image-20251019185121600.png)

## 云端运行

需要开启plus会员，登录GIHUB账号，拉取项目自动更新提交PR