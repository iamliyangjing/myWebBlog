---
title: 视野扩展MCP - 第一节基于TypeScript开发STDIO模式的MCP Server
tag:
  - AI
  - MCP
sidebar: true
comment: true
recommend: 1
sticky: 1
---
## 一、前言

今年MCP越来越火了，因此在AI时代，掌握MCP的开发已经成为Java开发人员的AI能力之一了，除了要学会使用Java开发MCP Server，我们也需要系统化的学习MCP知识，体系化的学习MCP的知识。

AI 从最初只能对话的 Chatbot，辅助人类决策的 Copilot，再到能自主感知和行动的 Agent，AI 在任务中的参与度不断提升。这要求 AI 拥有更丰富的任务上下文 **（Context）**，并拥有执行行动所需的工具集 **（Tools）**。

在这个过程中的痛点如下：

缺少标准化的上下文和工具集导致开发者的三大痛点：

1. **开发耦合度高**：工具开发者需要深入了解 Agent 的内部实现细节，并在 Agent 层编写工具代码。这导致在工具的开发与调试困难。
2. **工具复用性差**：因每个工具实现都耦合在 Agent 应用代码内，即使是通过 API 实现适配层在给到 LLM 的出入参上也有区别。从编程语言角度来讲，没办法做到跨编程语言进行复用。
3. **生态碎片化**：工具提供方能提供的只有 OpenAPI，由于缺乏标准使得不同 Agent 生态中的工具 Tool 互不兼容。

而2025年的新思想，就是将工具从 Agent 层解耦出来，单独变成一层 MCP Server 层，并对开发、调用进行标准化。 MCP Server 为上层 Agent 提供上下文、工具的标准化调用方式。

![image-20251226000024254](.\image\image-20251226000024254.png)

目前已经有了很多的MCP的市场，比如：

https://mcp.so/

![image-20251226000044350](.\image\image-20251226000044350.png)

https://mcp.aibase.cn/

![image-20251226000054946](.\image\image-20251226000054946.png)

https://sai.baidu.com/mcp

![image-20251226000106399](.\image\image-20251226000106399.png)

本节分享下如何基于TypeScript SDK来开发一个基于STDIO模式的MCP Server。

目前可以开发3种模式的MCP Server:																																													

![image-20251227115115265](.\image\image-20251227115115265.png)

每种通信方法在不同的应用场景中具有不同的优劣势，适用于不同的需求。本文分享的使Stdio的模式开发。

**Stdio 传输 (Standard input/output)**

stdio 传输方式是最简单的通信方式，通常在本地工具之间进行消息传递时使用。它利用标准输入输出 (stdin/stdout）作为数据传输通道，适用于本地进程间的交互。

**工作方式**：客户端和服务器通过标准输入输出流(stdlin/stdlout）进行通信。客户端向服务器发送命令和数据，服务器执行并通过标准输出返回结果。

**应用场景**：适用于本地开发、命令行工具、调试环境，或者模型和工具服务在同一进程内运行的情况。

**Server-Sent Events (SSE)**

SSE 是基于 HTTP 协议的流式传输机制，它允许服务器通过 HTTP 单向推送事件到客户端。SSE 适用于客户端需要接收服务器推送的场景，通常用于实时数据更新。

**工作方式**：客户端通过 HTTP GET 请求建立与服务器的连接，服务器以流式方式持续向客户端发送数据，客户端通过解析流数据来获取实时信息

**应用场景**：适用于需要服务器主动推送数据的场景，如实时聊天、天气预报、新闻更新等。

**Streamable HTTP**

它是 MCP 协议中新引入的一种传输方式，它基于 HTTP 协议支持双向流式传输。与传统的 HTTP 请求响应模型不同，Streamable HTTP 允许服务器在一个长连接中实时向客户端推送数据，并且可以支持多个请求和响应的流式传输。

不过需要注意的是，MCP只提供了Streamable HTTP协议层的支持，也就是规范了MCP客户端在使用Streamable HTTP通信时的通信规则，而并没有提供相关的SDK客户端。开发者在开发Streamable HTTP机制下的客户端和服务器时，可以使用比如Python httpx库进行开发。

1. **工作方式**：客户端通过 HTTP POST 向服务器发送请求，并可以接收流式响应（如 JSON-RPC 响应或 SSE 流）。当请求数据较多或需要多次交互时，服务器可以通过长连接和分批推送的方式进行数据传输。
2. **应用场景**：适用于需要支持高并发、低延迟通信的分布式系统，尤其是跨服务或跨网络的应用。适合高并发的场景，如实时流媒体、在线游戏、金融交易系统等。

使用TypeScript SDK开发一个Server是非常方便的，也容易快速学习。

## 二、正文

我们需要提前准备好：NodeJS环境，建议使用NVM的方式安装，这样可以随时切换不同版本的依赖。

一般常见的开发流程如下：

![image-20251227115141453](.\image\image-20251227115141453.png)

1、环境搭建完成后，首先我们创建一个文件夹，如下所示：

![image-20251227115203088](.\image\image-20251227115203088.png)

2、然后我们在当前目录下新建一个package.json文件，这里可以直接使用我的准备好的内容：

```json
{
  "name": "@wuchubuzai/tianqi-mcp-server",
  "version": "0.0.1",
  "description": "基于天气的MCP Server",
  "author": "Cooper",
  "type": "module",
  "license": "MIT",
  "main": "bin/index.js",
  "bin": {
    "tianqi-mcp-server": "dist/index.js"
  },
  "files": [
    "dist"
  ],
  "scripts": {
    "build": "tsc && shx chmod +x dist/*.js",
    "prepare": "npm run build",
    "watch": "tsc --watch"
  },
  "dependencies": {
    "@modelcontextprotocol/sdk": "1.10.2",
    "@types/node-fetch": "^2.6.12",
    "node-fetch": "^3.3.2"
  },
  "devDependencies": {
    "@types/node": "^22.15.3",
    "shx": "^0.3.4",
    "typescript": "^5.8.3"
  },
  "publishConfig": {
    "access": "public",
    "registry": "https://registry.npmjs.org"
  }
}
```

3、在新建一个tsconfig.json的文件，初始化了typescript项目的基本配置，文件内容如下：

```json
{
    "compilerOptions": {
      "target": "ES2022",
      "module": "Node16",
      "moduleResolution": "Node16",
      "strict": true,
      "esModuleInterop": true,
      "skipLibCheck": true,
      "forceConsistentCasingInFileNames": true,
      "resolveJsonModule": true,
      "outDir": "./dist",
      "rootDir": "."
    },
    
    "include": [
      "./**/*.ts"
    ],
    "exclude": ["node_modules"]
  }
```

3、然后我们新建一个空的index.ts文件，编辑完成后，我们在当前目录下执行npm install命令，安装当前的工程依赖。

![image-20251227115247204](.\image\image-20251227115247204.png)

4、接下来在index.ts文件中，写一个最简单版本的内容：

```ts
#!/usr/bin/env node
// 第一部分代码:声明导入的依赖内容等
import { Server } from "@modelcontextprotocol/sdk/server/index.js";
import { StdioServerTransport } from "@modelcontextprotocol/sdk/server/stdio.js";
import {
  CallToolRequestSchema,
  ListToolsRequestSchema,
  Tool,
} from "@modelcontextprotocol/sdk/types.js";
import fetch from "node-fetch";

// 第二部分：定义工具数组和工具的定义
const SUPPORT_TOOLS = [

  ] as const;


// 辅助函数：构建响应
function buildSuccessResponse(text: string) {
    return {
        content: [{ type: "text", text }],
        isError: false
    };
}
function buildErrorResponse(text: string) {
    return {
        content: [{ type: "text", text }],
        isError: true
    };
}


// 第三部分：服务器的初始化
const server = new Server({
    name: "tianqi-mcp-server",
    version: "0.0.1",
}, {
    capabilities: {
        tools: {},
    },
});

// Set up request handlers
server.setRequestHandler(ListToolsRequestSchema, async () => ({
    tools: SUPPORT_TOOLS,
}));

server.setRequestHandler(CallToolRequestSchema, async (request) => {
    try {
      switch (request.params.name) {
        case "xxxxx": {
          
        }
        default:
          return {
            content: [{
              type: "text",
              text: `Unknown tool: ${request.params.name}`
            }],
            isError: true
          };
      }
    } catch (error) {
      return {
        content: [{
          type: "text",
          text: `Error: ${error instanceof Error ? error.message : String(error)}`
        }],
        isError: true
      };
    }
  });

// 第四部分，服务器启动
async function runServer() {
    const transport = new StdioServerTransport();
    await server.connect(transport);
    console.error("Tianqi MCP Server running on stdio");
}
runServer().catch((error) => {
    console.error("Fatal error running server:", error);
    process.exit(1);
});
```

这个文件中的StdioServerTransport代表的是基于STDIO模式的传输类，也是非常方便我们使用的。

这样我们的一个基于STDIO模式的基于架子就搭建好了，接下来我们模拟一个天气查询的MCP Server。

这里我使用的免费的API接口地址：**https://api.aa1.cn/**

这个上面有非常多的免费的API地址，可以找到一些可以用的进行测试，这里我找到的一个地址如下：

https://api.苏苏.cn/API/moji.php?city=哈尔滨&n=1

访问后可以如下数据：

```
{
    "code": 200,
    "城市": "黑龙江省 - 哈尔滨市",
    "数据": {
        "今天": {
            "天气": "多云",
            "温度": "4° / 16°",
            "空气质量": "35 优",
            "风向风度": "西北风 1级"
        },
        "明天": {
            "天气": "多云",
            "温度": "3° / 18°",
            "空气质量": "56 良",
            "风向风度": "西风 3级"
        },
        "后天": {
            "天气": "晴",
            "温度": "6° / 19°",
            "空气质量": "88 良",
            "风向风度": "西北风 1级"
        }
    },
    "建议": "哈尔滨市今天实况：8度 多云，湿度：48%，西风：1级。白天：16度,晴。 夜间：多云，4度，天凉了，墨迹天气建议您在羊毛衫外面套上厚外套，年老体弱者可以穿着呢大衣增加保暖系数。",
    "Tips": "苏苏Api提供技术支持"
}
```

截图如下：

![image-20251227115338072](.\image\image-20251227115338072.png)

因此，本节我们希望实现一个天气查询的MCP Server。

首先我们需要定义天气查询的基本的工具方法说明如下：

```ts
const GET_TIANQI:Tool = {
    name: "get_city_tianqi",
    description: "根据城市名称，返回当前查询城市的天气情况信息",
    inputSchema: {
        type: "object",
        properties: {
            city: {
                type: "string",
                description: "需要查询天气的城市名称"
            }
        },
        required: ["city"]
    }
};
const SUPPORT_TOOLS = [
    GET_TIANQI
  ] as const;
```

这样我们就定义好了这个工具对外的暴露说明和基本的结构，这里定义了一个城市的参数。

接下来我们写一个方法， 根据参数请求接口，返回天气的数据，代码如下所示：

```ts
async function handleGetTianqiData(city :string) {
    // 1.参数校验
    if(!city){
        return buildErrorResponse('参数不完整，请确保提供了city参数');
    }
    const url = "https://api.苏苏.cn/API/moji.php?city="+city+"&n=1";


  // 先发送第一个请求，根据城市名称获取编码数据
    const response1 = await fetch(url, {
            method: 'GET'
    });

    const respBody:any = await response1.json();
   
    // 返回正确的结果
    return buildSuccessResponse(JSON.stringify(respBody, null, 2));
}
```

然后我们在setRequestHandler的注册逻辑中，补充上这个方法的逻辑，最终完整的代码如下：

```ts
#!/usr/bin/env node
// 第一部分代码:声明导入的依赖内容等
import { Server } from "@modelcontextprotocol/sdk/server/index.js";
import { StdioServerTransport } from "@modelcontextprotocol/sdk/server/stdio.js";
import {
  CallToolRequestSchema,
  ListToolsRequestSchema,
  Tool,
} from "@modelcontextprotocol/sdk/types.js";
import fetch from "node-fetch";

// 第二部分：定义工具数组和工具的定义
const GET_TIANQI:Tool = {
    name: "get_city_tianqi",
    description: "根据城市名称，返回当前查询城市的天气情况信息",
    inputSchema: {
        type: "object",
        properties: {
            city: {
                type: "string",
                description: "需要查询天气的城市名称"
            }
        },
        required: ["city"]
    }
};
const SUPPORT_TOOLS = [
    GET_TIANQI
  ] as const;



// 辅助函数：构建响应
function buildSuccessResponse(text: string) {
    return {
        content: [{ type: "text", text }],
        isError: false
    };
}
function buildErrorResponse(text: string) {
    return {
        content: [{ type: "text", text }],
        isError: true
    };
}


async function handleGetTianqiData(city :string) {
    // 1.参数校验
    if(!city){
        return buildErrorResponse('参数不完整，请确保提供了city参数');
    }
    const url = "https://api.苏苏.cn/API/moji.php?city="+city+"&n=1";


  // 先发送第一个请求，根据城市名称获取编码数据
    const response1 = await fetch(url, {
            method: 'GET'
    });

    const respBody:any = await response1.json();
   
    // 返回正确的结果
    return buildSuccessResponse(JSON.stringify(respBody, null, 2));
}


// 第三部分：服务器的初始化
const server = new Server({
    name: "tianqi-mcp-server",
    version: "0.0.1",
}, {
    capabilities: {
        tools: {},
    },
});

// Set up request handlers
server.setRequestHandler(ListToolsRequestSchema, async () => ({
    tools: SUPPORT_TOOLS,
}));

server.setRequestHandler(CallToolRequestSchema, async (request) => {
    try {
      switch (request.params.name) {
        case "get_city_tianqi": {
          const { city } = request.params.arguments as { city :string};
          return await handleGetTianqiData(city);
        }
        default:
          return {
            content: [{
              type: "text",
              text: `Unknown tool: ${request.params.name}`
            }],
            isError: true
          };
      }
    } catch (error) {
      return {
        content: [{
          type: "text",
          text: `Error: ${error instanceof Error ? error.message : String(error)}`
        }],
        isError: true
      };
    }
  });

// 第四部分，服务器启动
async function runServer() {
    const transport = new StdioServerTransport();
    await server.connect(transport);
    console.error("Tianqi MCP Server running on stdio");
}
runServer().catch((error) => {
    console.error("Fatal error running server:", error);
    process.exit(1);
});
```

这样我们的MCP Server开发完成。然后本地执行命令进行构建，构建完成后会在当前目录下输出dist/index.js文件：

npm run build     

在执行命令进行启动调试，命令如下：

npx -y @modelcontextprotocol/inspector node dist/index.js

进入到浏览器的控制台，我们选择STDIO模式，选择List Tool后，单击制定的tool，输入我们的测试数据，确保可以成功，如下所示：![image-20251227115437095](.\image\image-20251227115437095.png)

接下来，我们使用Cherry Studio客户端，注册上这个开发环境的MCP Server，（windows电脑）如下所示：

![image-20251227115511780](.\image\image-20251227115511780.png)

然后保存和启用。

然后我们问一个问题测试下，可以看到也是成功的：

![image-20251227115522615](.\image\image-20251227115522615.png)

这样基于Typescript开发一个MCP Server我们就完成了。

这里我们使用的是TypeScript，因此我们可以发布到NPM的远程仓库，给其他人使用，首先可以将源代码发布到Github仓库，然后在npmjs网站注册个账号，然后本地执行如下命令进行打包上传：

npm login

npm publish

## 三、总结
本文分享了基于TypeScript SDK的MCP的开发教程，之所以使用TypeScript作为MCP的学习入门来说是非常方便的，非常容易和掌握开发一个MCP Server的步骤，也非常容易制作自己的第一个MCP Server，并发布到远程的市场。
MCP 的演进本质是加速 AI 基础设施的普及：
​    技术层：通过协议标准化降低开发门槛，使中小开发者能快速构建复杂 AI 应用。
​    商业层：催生“MCP 工具商店”等新商业模式，工具开发者可通过协议分成获利。
​    社会层：推动 AI 从“专家系统”转向“普惠技术”，例如农民通过自然语言指令操作智能农业设备。