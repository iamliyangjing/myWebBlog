---
description: MCP（Model Context Protocol）正在成为 AI Agent 生态的重要基础设施。本文整理了 8 个实用的 MCP Server，包括 Chrome DevTools、Neon、Supabase、Context7、Replicate、Vercel 和 GitHub，并介绍如何通过 MCP 让 AI 直接操作浏览器、数据库、代码仓库和部署平台，构建完整的 AI 开发工作流。
title: 8 个必备 MCP Server：打造完整 AI Agent 开发环境（Claude Code / Cursor）
tag:
  - AI
  - MCP
sidebar: true
comment: true
commend: 100
---
# MCP 工具推荐（AI Agent 开发必备）

MCP（Model Context Protocol）正在成为 AI Agent 生态的重要基础设施。
 通过 MCP Server，大模型可以直接访问浏览器、数据库、文档、部署平台等资源。

本文整理了一些 **实用的 MCP Server**，适用于 **Claude Code / Cursor / AI Agent 开发环境**。

## 1. Chrome DevTools

用于 **自动化控制 Chrome 浏览器**。

AI 可以通过命令执行：

- 打开网页
- 操作 DOM
- 自动化测试
- 页面验证
- UI 自动化

适合场景：

- 自动化测试
- 前端调试
- AI 自动操作浏览器

项目地址：

https://github.com/ChromeDevTools/chrome-devtools-mcp

安装：

```bash
claude mcp add chrome-devtools --scope user npx chrome-devtools-mcp@latest
```

## 2. Neon

Neon 是一个 **Serverless Postgres 数据库**，非常适合 AI Agent 使用。

特点：

- Serverless Postgres
- 自动扩缩容
- 免费额度
- 非常适合 AI 应用

使用 MCP 后，AI 可以直接：

- 查询数据库
- 创建表
- 执行 SQL
- 管理数据

文档：

https://neon.com/docs/ai/neon-mcp-server

## 3. Supabase

Supabase 是一个完整的 **Postgres 开发平台**，提供：

- Postgres 数据库
- Authentication
- Storage
- Realtime
- Edge Functions
- Vector Embedding

非常适合构建 **AI SaaS / Agent 系统**。

文档：

https://supabase.com/docs/guides/functions/examples/mcp-server-mcp-lite



## 4. Context7

用于 **检索最新技术文档**。

很多 AI 模型训练数据较旧，而 Context7 可以：

- 搜索最新文档
- 获取 API 文档
- 获取 SDK 使用说明
- 实时技术资料

项目地址：

https://github.com/upstash/context7

适合：

- AI 编程助手
- 自动代码生成
- API 查询

## 5. Ref

Ref MCP 是一个 **文档检索服务器**，专门给 AI Agent 使用。

功能：

- 检索 API 文档
- 检索库文档
- 获取开发资料
- Token 消耗更低

特点：

- 数据源更广
- 可访问一些较冷门文档
- 适合 AI Coding Agent

项目地址：

https://github.com/ref-tools/ref-tools-mcp

## 6. Replicate

Replicate MCP 用于 **AI 图片生成**。

可以让 AI 直接调用 Replicate 的模型生成图片，例如：

- 网站默认图片
- Banner
- AI 插图
- 产品图

项目：

GitHub
 https://github.com/awkoy/replicate-flux-mcp

官方文档：

https://replicate.com/docs

安装：

```
claude mcp add replicate npx replicate-flux-mcp
```

配置步骤：

### 1 安装 MCP

```
claude mcp add replicate npx replicate-flux-mcp
```

### 2 创建 API Key

登录 Replicate 官网创建 API Key。![image-20260315154148961](.\image\image-20260315154148961.png)

### 3 验证 MCP

运行：

```
claude mcp list
```

确认 replicate 已安装成功。![image-20260315154219378](.\image\image-20260315154219378.png)

## 7. Vercel

Vercel MCP 允许 AI 直接操作 **网站部署平台**。

AI 可以：

- 部署项目
- 查看部署日志
- 管理环境变量
- 发布网站

文档：

https://vercel.com/docs/agent-resources/vercel-mcp#claude-code

使用前需要：

- 登录 Vercel
- 绑定账户

## 8. Github

GitHub MCP 允许 AI 直接操作 GitHub。

支持功能：

- 创建 Issue
- 管理 Pull Request
- 读取代码
- 自动提交代码
- 仓库管理

项目地址：

https://github.com/github/github-mcp-server

安装：

```
claude mcp add github --transport http https://api.githubcopilot.com/mcp/
```

配置 Token：

```
"github": {
  "type": "http",
  "url": "https://api.githubcopilot.com/mcp/",
  "headers": {
    "Authorization": "Bearer ${input:github_mcp_pat}"
  }
}
```

## MCP 工具推荐组合

如果你在做 **AI Coding Agent**，推荐组合：

| 功能       | MCP             |
| ---------- | --------------- |
| 浏览器操作 | Chrome DevTools |
| 数据库     | Neon / Supabase |
| 技术文档   | Context7        |
| 文档检索   | Ref             |
| 图片生成   | Replicate       |
| 部署       | Vercel          |
| 代码仓库   | GitHub          |

这样 AI Agent 就可以拥有完整能力：

```
浏览器 + 数据库 + 文档 + 代码仓库 + 部署 + AI生成
```

基本等于一个 **完整 AI 工程助手**。