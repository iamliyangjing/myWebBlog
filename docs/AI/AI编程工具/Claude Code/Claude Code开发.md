---
description: 想要体验 Claude Code 强大的编程能力，又受限于官方 API 额度？本文手把手教你安装配置 claude-code-router，通过接入阿里云 ModelScope 免费版 Qwen3 和 Google Gemini，无需官方 Key 也能流畅运行 AI 命令行工具。附完整 Config 配置与常见端口报错解决指南。
title: 零成本解锁 Claude Code：用 ModelScope 白嫖 Qwen3 打造 AI 编程助手
tag:
  - AI
  - Claude code
sidebar: true
comment: true
recommend: 1
---

## Claude Code

官网：https://claude.com/product/claude-code

安装命令 （需要下载NodeJS > 18.0）

```cmd
npm install -g @anthropic-ai/claude-code
```



## Claude Code router

使用API的方式接入 Claude Code

github 地址：https://github.com/musistudio/claude-code-router

安装命令：

```cmd
npm install -g @musistudio/claude-code-router
```

需要在本地**User目录下创建一个config.json文件。**

> Create and configure your `~/.claude-code-router/config.json` file. For more details, you can refer to `config.example.json`.

![image-20250927211335506](..\image\image-20250927211335506.png)

等待config.json文件配置成功后输入'**ccr code**'就可以直接启动

### ⚠️补充一

有时候我们执行ccr code启动不起来，可能是因为端口被占用了。

![image-20250927212619284](..\image\image-20250927212619284.png)

可以在配置文件conf.json里面加一个PORT自定义一个端口。

![image-20250927212715036](..\image\image-20250927212715036.png)

### ⚠️补充二

每次修改完配置文件需要执行ccr stop，再重新启动保证配置文件刷新。

## ModelScope社区

官网：https://modelscope.cn/my/overview

可以免费白嫖阿里Qwen Qcoder3模型，需要绑定阿里云社区账号

![image-20250927212032564](..\image\image-20250927212032564.png)

绑定完成后创建访问令牌Token

![image-20250927212210649](..\image\image-20250927212210649.png)

可以在个人首页查看每天调用次数。

![image-20250927212919762](..\image\image-20250927212919762.png)

可以在config.json文件自定义API提供商。

```JSON
{
  "Providers": [
    {
      "name": "modelscope",
      "api_base_url": "https://api-inference.modelscope.cn/v1/chat/completions",
      "api_key": "",
      "models": ["Qwen/Qwen3-Coder-480B-A35B-Instruct", "Qwen/Qwen3-235B-A22B-Thinking-2507"],
      "transformer": {
        "use": [
          [
            "maxtoken",
            {
              "max_tokens": 65536
            }
          ],
          "enhancetool"
        ],
        "Qwen/Qwen3-235B-A22B-Thinking-2507": {
          "use": ["reasoning"]
        }
      }
    }
  ],
  "Router": {
    "default": "modelscope,Qwen/Qwen3-Coder-480B-A35B-Instruct,deepseek-chat"
  }
}
```

用其他模型API_key复制给Config.Json的**API_KEY**.

![image-20250927212241427](..\image\image-20250927212241427.png)

## Gemini

官网:https://aistudio.google.com/api-keys

创建API_Key

![image-20250927213756844](..\image\image-20250927213756844.png)

```json
{
  "Providers": [
    {
      "name": "gemini",
      "api_base_url": "https://generativelanguage.googleapis.com/v1beta/models/",
      "api_key": "",
      "models": ["gemini-2.5-flash", "gemini-2.5-pro"],
      "transformer": {
        "use": ["gemini"]
      }
    }
  ],
  "Router": {
    "default": "gemini,gemini-2.5-flash",
    "think":"gemini-2.5-pro"
  }
}
```



## GLM4.5

官网：https://bigmodel.cn/usercenter/settings/account

可以充值GLM-4.5

## Kimi K2

登录moonshot 平台

https://platform.moonshot.cn/console/account

创建一个密码，把密钥复制保存好

![image-20250927210603071](..\image\image-20250927210603071.png)





