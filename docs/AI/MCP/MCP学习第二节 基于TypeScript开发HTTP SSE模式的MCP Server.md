---
title: 视野扩展MCP - 第二节基于TypeScript开发HTTP SSE模式的MCP Server
tag:
  - AI
  - MCP
sidebar: true
comment: true
commend: 99
sticky: 2
---
## 一、前言

今年MCP越来越火了，因此在AI时代，掌握MCP的开发已经成为Java开发人员的AI能力之一了，除了要学会使用Java开发MCP Server，我们也需要系统化的学习MCP知识，体系化的学习MCP的知识。

本节分享下如何基于TypeScript SDK来开发一个基于HTTP SSE模式的MCP Server。

这种模式的MCP Server开发完成后，对于客户端的引用来说，只需要配置一个远程的HTTP SSE地址即可。非常方便，同时基于TypeScript SDK这种开发方式来说，我们也容易学习到一些原理方面的知识。

官方画了这个图非常清晰的表达了远程MCP和本地MCP的概念:

![image-20251227133203234](.\image\image-20251227133203234.png)

标红的区域是本节我们开发的这种模式。

Spring AI也分享了一个关于Stdio模式和SSE模式的对比图，大家可以感觉一下：

![image-20251227133220962](.\image\image-20251227133220962.png)

对于上一节分享的Stdio模式和本节的HTTP SSE模式来说，区别如下：

![image-20251227133231828](.\image\image-20251227133231828.png)

## 二、正文

我们需要提前准备好：NodeJS环境，建议使用NVM的方式安装，这样可以随时切换不同版本的依赖。

1、环境搭建完成后，首先我们创建一个文件夹，也可以复制第一节的目录，这样改动会非常的小，如下所示：

```
mkdir typescript-httpsse-demo-mcpserver
```

2、然后我们在当前目录下新建一个package.json文件，这次文件相较于上一节中的内容，我们多了几个新的依赖：

```
express
@types/express
```

这里可以直接使用我的准备好的内容，完整的样例代码如下所示：

```
{
  "name": "@wuchubuzai/tianqi-mcp-server",
  "version": "0.0.1",
  "description": "基于天气的MCP Server",
  "author": "爱海贼的无处不在",
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
    "express": "^4.18.2",
    "node-fetch": "^3.3.2"
  },
  "devDependencies": {
    "@types/express": "^5.0.1",
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

3、在新建一个tsconfig.json的文件，初始化了typescript项目的基本配置，这个文件的内容与第一节的Stdio模式的内容一致，文件内容如下：

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

![image-20251227140534701](.\image\image-20251227140534701.png)

4、接下来在index.ts文件中，我们基于上一节的基础之上做几点修改：

（1）、首先在引入的时候，替换StdioServerTransport为SSEServerTransport

（2）、引入依赖：

import express from "express";

（3）、定义服务器代码：

const app = express();

app.use(express.json());

（4）、定义对外的两个端点地址：/sse、/messages

（5）、在/sse请求逻辑中增加和原理类似的初始化链接的相关方法。

本节的index.ts文件的完整代码如下所示：

```typescript
#!/usr/bin/env node
// 第一部分代码:声明导入的依赖内容等
import { Server } from "@modelcontextprotocol/sdk/server/index.js";
import { SSEServerTransport } from "@modelcontextprotocol/sdk/server/sse.js";
import express from "express";

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


// 定义一个服务器
const app = express();
app.use(express.json());

// Store transports for each session type
const transports = {
  sse: {} as Record<string, SSEServerTransport>
};

// HTTP SSE模式需要的请求地址
app.get('/sse', async (req:any, res:any) => {
      // Create SSE transport for legacy clients
      console.log("http /sse request start.......") 
      const transport = new SSEServerTransport('/messages', res);
      transports.sse[transport.sessionId] = transport;
      
      res.on("close", () => {
        delete transports.sse[transport.sessionId];
      });
      
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
      
      await server.connect(transport);
      console.log("http /sse request connect") 
});


// HTTP SSE模式需要的请求地址messages
app.post('/messages', async (req:any, res:any) => {
  console.log("http /sse request receive message"); 
  const sessionId = req.query.sessionId as string;
  const transport = transports.sse[sessionId];
  if (transport) {
    await transport.handlePostMessage(req, res, req.body);
  } else {
    res.status(400).send('No transport found for sessionId');
  }
});

// 监听3000端口并启动
const PORT = process.env.PORT || 3000;
app.listen(PORT, () => {
    console.log("HTTP SSE Server Started.....") 
    console.log(`Server is running on port ${PORT}`);
  })
  .on('error', (err) => {
    console.error(`Server error`, err);
  });
```

文件中我使用的免费的API接口地址：**https://api.aa1.cn/**

这个上面有非常多的免费的API地址，可以找到一些可以用的进行测试，这里我找到的一个地址如下：

https://api.苏苏.cn/API/moji.php?city=北京&n=1

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

因此，本节我们希望实现一个天气查询的MCP Server。

首先我们需要定义天气查询的基本的工具方法说明如下：

```typescript
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

```typescript
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

然后本节我们将初始化链接的代码放到了/sse的请求地址中，这样就基本方便的开发了一个基于HTTP SSE模式的MCP Server。

![image-20251227140704235](.\image\image-20251227140704235.png)

接下来本地执行命令进行构建，构建完成后会在当前目录下输出dist/index.js文件：

npm run build

那么本节的MCP Server的启动命令如下所示：

node dist/index.js

这样我们就完成了一个HTTP SSE模式的接口服务端开发。

然后我们依然可以使用调试工具，在线调试这个接口，这里我就不分享了。

npx -y @modelcontextprotocol/inspector 

接下来，我们使用Cherry Studio客户端，体验一下这个基于HTTP SSE方式的MCP Server，设置好信息后，单击保存按钮即可，如下所示：

![image-20251227140759497](.\image\image-20251227140759497.png)

![image-20251227140749171](.\image\image-20251227140749171.png)

在控制台中，我们也可以看到自己的相关请求日志：

![image-20251227140812250](.\image\image-20251227140812250.png)

这样基于Typescript开发一个HTTP SSE模式的MCP Server我们就完成了。

如果想发布给别人使用，我们可以部署到自己的服务器上后，对外提供IP:PORT/sse这样的地址即可。

## 三、总结

本文分享了基于TypeScript SDK的HTTP SSE模式的MCP Server的开发教程，之所以使用TypeScript作为MCP的学习入门来说是非常方便的，非常容易和掌握开发一个MCP Server的步骤，也非常容易制作自己的第一个MCP Server，并发布到远程的市场。