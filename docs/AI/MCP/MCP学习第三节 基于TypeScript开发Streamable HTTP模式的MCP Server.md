## 一、前言

2025年，MCP的发展是所有程序员都需要去关注和掌握的技术，也许有一天，AI 程序员不懂 MCP，就犹如前端程序员不懂 JavaScript！

在前两节中，我分享了MCP的2种开发模式：基于本地环境的Stdio命令模式、基于服务端环境的HTTP SSE模式，对于MCP技术来说，这两种模式是2024年的第一版的MCP协议玩法，即协议版本为：2024-11-05。

SSE 的最大缺陷之一，显而易见：SSE 需要 server 端保持一个长连接，而且，根据 MCP 的协议，在 MCP Client 与 MCP

 Server 建立 SEE 连接后，在整个 connection 的生命周期中，MCP Server 需要一直保持着这个 SSE 连接。

![image-20251228004832445](E:\CodeRepositroy\myWebBlog\docs\AI\MCP\image\image-20251228004832445.png)

那么，对于要支持 SSE 的运行在 remote 的 MCP Server 来说，就需要保证高可靠性。在高并发的情况下，对 MCP Server 的负载更是一个挑战。

在 2025年3 月 26 日，MCP 发布了最新的第二代MCP协议标准，用 Streamable HTTP “取代”了HTTP SSE这种模式。

![image-20251228004844870](E:\CodeRepositroy\myWebBlog\docs\AI\MCP\image\image-20251228004844870.png)

简单来说，在新的 Streamable HTTP 协议中，MCP Server 可以根据自己实际的使用场景来决定自己是 Stateless 还是 Stateful 的，而不是像 SSE 那样，一定是 Stateful 的。

这对开发 Remote MCP Server 的开发者来说，真是一个极好的消息，因为在不少场景中，Stateless server 会对 MCP Server 的要求降低很多！

Streamable HTTP 并不是传统意义上的 **流式 HTTP**（Streaming HTTP），它指的是一种 **兼具以下特性的传输机制**：

1. 以普通 HTTP 请求为基础，客户端用 POST/GET 发请求；
2. 服务器可选地将响应升级为 SSE 流，实现 **流式传输** 的能力（当需要时）；
3. 去中心化、无强制要求持续连接，支持 stateless 模式；
4. 客户端和服务端之间的消息传输更加灵活，比如同一个 /message 端点可用于发起请求和接收 SSE 流；
5. 不再需要单独的 /sse 端点，一切通过统一的 /message 协议层处理。

本节在之前的基础之上，还是基于TypeScript-SDK快速开发一个 Streamable HTTP 的 MCP Server！

![image-20251228004859502](E:\CodeRepositroy\myWebBlog\docs\AI\MCP\image\image-20251228004859502.png)

TypeScript-SDK的设计理念是"简单易用但功能强大"，即使你不是TypeScript专家，也能轻松上手并构建复杂的MCP应用。

## 二、正文

我们需要提前准备好：NodeJS环境，建议使用NVM的方式安装，这样可以随时切换不同版本的依赖。

1、环境搭建完成后，首先我们创建一个文件夹，也可以复制第一节的目录，这样改动会非常的小，如下所示：

![image-20251228004940129](E:\CodeRepositroy\myWebBlog\docs\AI\MCP\image\image-20251228004940129.png)2、然后我们复制上一节httpsse的文件夹内容，作为本节的基础代码：

![image-20251228005016705](E:\CodeRepositroy\myWebBlog\docs\AI\MCP\image\image-20251228005016705.png)

3、接下来，我们继续修改index.ts文件，更改实现为Streamable HTTP模式的MCP Server代码。本节分成6个部分，其中核心是创建StreamableHTTPServerTransport对象，就可以实现一个MCP Server。

```
 //  无会话方式的创建
      const transport: StreamableHTTPServerTransport = new StreamableHTTPServerTransport({
        sessionIdGenerator: undefined,
      });

      const server = getServer();
      await server.connect(transport);
      await transport.handleRequest(req, res, req.body);
```

接下来分享下针对这个无状态的Streamable HTTP MCP Server开发的几个部分：

![image-20251228005034446](E:\CodeRepositroy\myWebBlog\docs\AI\MCP\image\image-20251228005034446.png)

4、首先在引入依赖的时候，更改为StreamableHTTPServerTransport：

```json
// 第1部分代码:声明导入的依赖内容等
import { Server } from "@modelcontextprotocol/sdk/server/index.js";
import { StreamableHTTPServerTransport } from "@modelcontextprotocol/sdk/server/streamableHttp.js";
import express, { Request, Response } from 'express';
import {
  CallToolRequestSchema,
  ListToolsRequestSchema,
  Tool,
  isInitializeRequest
} from "@modelcontextprotocol/sdk/types.js";
import fetch from "node-fetch";
```

5、第2部分的工具定义不变，第3部分，我们将上一节的Server创建的过程单独抽取为一个小方法：

```json
// 第4部分：定义一个mcpserver的基本信息
function getServer() {
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
          const { city } = request.params.arguments as { city: string; };
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
  return server;
}
```

6、然后我们删除掉上一节的SSE的端点，更改为/mcp这个端点，这里依然是接收请求参数和响应，然后创建对象：

```json
// 定义一个MCP的端点
app.post('/mcp', async (req:any, res:any) => {
    console.log('Received MCP request:', req.body);
    try {
      //  无会话方式的创建
      const transport: StreamableHTTPServerTransport = new StreamableHTTPServerTransport({
        sessionIdGenerator: undefined,
      });

      const server = getServer();
      await server.connect(transport);
      await transport.handleRequest(req, res, req.body);
      res.on('close', () => {
        console.log('Request closed');
        transport.close();
        server.close();
      });
    }catch (error) {
      console.error('Error handling MCP request:', error);
      if (!res.headersSent) {
        res.status(500).json({
          jsonrpc: '2.0',
          error: {
            code: -32603,
            message: 'Internal server error',
          },
          id: null,
        });
      }
    }

 
});


app.get('/mcp', async (req: Request, res: Response) => {
  console.log('Received GET MCP request');
  res.writeHead(405).end(JSON.stringify({
    jsonrpc: "2.0",
    error: {
      code: -32000,
      message: "Method not allowed."
    },
    id: null
  }));
});

app.delete('/mcp', async (req: Request, res: Response) => {
  console.log('Received DELETE MCP request');
  res.writeHead(405).end(JSON.stringify({
    jsonrpc: "2.0",
    error: {
      code: -32000,
      message: "Method not allowed."
    },
    id: null
  }));
});

```

这段代码也是核心的代码，这样本文的无状态管理的MCP Server的index.ts代码如下所示：

```typescript
#!/usr/bin/env node
// 第1部分代码:声明导入的依赖内容等
import { Server } from "@modelcontextprotocol/sdk/server/index.js";
import { StreamableHTTPServerTransport } from "@modelcontextprotocol/sdk/server/streamableHttp.js";
import express, { Request, Response } from 'express';
import {
  CallToolRequestSchema,
  ListToolsRequestSchema,
  Tool,
  isInitializeRequest
} from "@modelcontextprotocol/sdk/types.js";
import fetch from "node-fetch";

// 第2部分：定义工具描述和工具的定义
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

// 第3部分：定义工具处理的方法
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

// 第4部分：定义一个mcpserver的基本信息
function getServer() {
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
          const { city } = request.params.arguments as { city: string; };
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
  return server;
}



// 第5部分：定义一个服务器
const app = express();
app.use(express.json());

// 定义一个MCP的端点
app.post('/mcp', async (req:any, res:any) => {
    console.log('Received MCP request:', req.body);
    try {
      //  无会话方式的创建
      const transport: StreamableHTTPServerTransport = new StreamableHTTPServerTransport({
        sessionIdGenerator: undefined,
      });

      const server = getServer();
      await server.connect(transport);
      await transport.handleRequest(req, res, req.body);
      res.on('close', () => {
        console.log('Request closed');
        transport.close();
        server.close();
      });
    }catch (error) {
      console.error('Error handling MCP request:', error);
      if (!res.headersSent) {
        res.status(500).json({
          jsonrpc: '2.0',
          error: {
            code: -32603,
            message: 'Internal server error',
          },
          id: null,
        });
      }
    }

 
});


app.get('/mcp', async (req: Request, res: Response) => {
  console.log('Received GET MCP request');
  res.writeHead(405).end(JSON.stringify({
    jsonrpc: "2.0",
    error: {
      code: -32000,
      message: "Method not allowed."
    },
    id: null
  }));
});

app.delete('/mcp', async (req: Request, res: Response) => {
  console.log('Received DELETE MCP request');
  res.writeHead(405).end(JSON.stringify({
    jsonrpc: "2.0",
    error: {
      code: -32000,
      message: "Method not allowed."
    },
    id: null
  }));
});

// 第6部分：监听3000端口并启动
const PORT = process.env.PORT || 3000;
app.listen(PORT, () => {
    console.log(`Streamable HTTP Server listening on port ${PORT}`);
  })
  .on('error', (err) => {
    console.error(`Server error`, err);
  });


// Handle server shutdown
process.on('SIGINT', async () => {
  console.log('Shutting down server...');
  console.log('Server shutdown complete');
  process.exit(0);
});

```

文件中我使用还是免费的API接口地址：**https://api.aa1.cn/**

这个上面有非常多的免费的API地址，可以找到一些可以用的进行测试，这里我找到的一个地址如下：

https://api.苏苏.cn/API/moji.php?city=北京&n=1

这样就开发完成了一个没有状态管理的，Streamable HTTP模式的MCP Server，接下来本地执行命令进行构建，构建完成后会在当前目录下输出dist/index.js文件：

npm run build

那么本节的MCP Server的启动命令如下所示：

node dist/index.js

这样我们就完成了一个HTTP SSE模式的接口服务端开发。

然后我们依然可以使用调试工具，在线调试这个接口，这里我就不分享了。

npx -y @modelcontextprotocol/inspector

接下来，我们使用Cherry Studio客户端，体验一下这个基于Streamable HTTP方式的MCP Server，设置好信息后，单击保存按钮即可，如下所示：

![image-20251228012519961](E:\CodeRepositroy\myWebBlog\docs\AI\MCP\image\image-20251228012519961.png)

然后我们去聊天界面选中这个MCP工具，然后去测试下，可以看到也是成功的。

在控制台中，我们也可以看到自己的相关请求日志：

![image-20251228012530194](E:\CodeRepositroy\myWebBlog\docs\AI\MCP\image\image-20251228012530194.png)

通过这个日志可以看到，对于无状态管理的MCP Server，每次都会有个Server和Transport的创建过程，有些时候我们可以更改为有状态的代码控制，针对本文的这个Streamable HTTP模式的MCP Server，完整代码如下：

```typescript
#!/usr/bin/env node
// 第一部分代码:声明导入的依赖内容等
import { Server } from "@modelcontextprotocol/sdk/server/index.js";
import { StreamableHTTPServerTransport } from "@modelcontextprotocol/sdk/server/streamableHttp.js";
import express, { Request, Response } from 'express';
import { randomUUID } from 'node:crypto';
import {
  CallToolRequestSchema,
  ListToolsRequestSchema,
  Tool,
  isInitializeRequest
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

// 定义一个mcpserver的基本信息
function getServer() {
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
          const { city } = request.params.arguments as { city: string; };
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
  return server;
}

// 第3部分：定义一个服务器
const app = express();
app.use(express.json());

// 存储每个会话的信息
const transports: { [sessionId: string]: StreamableHTTPServerTransport } = {};

// 定义一个MCP的端点
app.post('/mcp', async (req:any, res:any) => {
    console.log('Received MCP request:', req.body);
    console.log('Received MCP request HEADER:', req.headers);
    try {
        //1.获取前端传递的会话标识等信息
        const sessionId = req.headers['mcp-session-id'] as string | undefined;
        let transport: StreamableHTTPServerTransport;

        // 2.判断是否存在会话标识
        if (sessionId && transports[sessionId]) {
          // 如果存在，则获取出来
          transport = transports[sessionId];
        } else if (!sessionId && isInitializeRequest(req.body)) {
          // 初始化的请求 
          // 如果指定配置代表启用json模式：enableJsonResponse: true,
          transport = new StreamableHTTPServerTransport({
            sessionIdGenerator: () => randomUUID(),
            enableJsonResponse: true,
            onsessioninitialized: (sessionId) => {
              // Store the transport by session ID when session is initialized
              // This avoids race conditions where requests might come in before the session is stored
              console.log(`Session initialized with ID: ${sessionId}`);
              transports[sessionId] = transport;
            }
          });

          // Set up onclose handler to clean up transport when closed
          transport.onclose = () => {
              const sid = transport.sessionId;
              if (sid && transports[sid]) {
                console.log(`Transport closed for session ${sid}, removing from transports map`);
                delete transports[sid];
              }
            };

          const server = getServer();
          
          await server.connect(transport);
          console.log("streamable http request connect")   
          await transport.handleRequest(req, res, req.body);
          return; // Already handled
        }else{
          // Invalid request - no session ID or not initialization request
          res.status(400).json({
           jsonrpc: '2.0',
           error: {
             code: -32000,
             message: 'Bad Request: No valid session ID provided',
           },
           id: null,
         });
         return;
       }

        // Handle the request with existing transport - no need to reconnect
        // The existing transport is already connected to the server
        await transport.handleRequest(req, res, req.body);

    }catch (error) {
      console.error('Error handling MCP request:', error);
      if (!res.headersSent) {
        res.status(500).json({
          jsonrpc: '2.0',
          error: {
            code: -32603,
            message: 'Internal server error',
          },
          id: null,
        });
      }
    }

 
});


app.get('/mcp', async (req: Request, res: Response) => {
  console.log('Received GET MCP request');
  res.writeHead(405).end(JSON.stringify({
    jsonrpc: "2.0",
    error: {
      code: -32000,
      message: "Method not allowed."
    },
    id: null
  }));
});

app.delete('/mcp', async (req: Request, res: Response) => {
  console.log('Received DELETE MCP request');
  res.writeHead(405).end(JSON.stringify({
    jsonrpc: "2.0",
    error: {
      code: -32000,
      message: "Method not allowed."
    },
    id: null
  }));
});

// 第4部分：监听3000端口并启动
const PORT = process.env.PORT || 3000;
app.listen(PORT, () => {
    console.log(`Streamable HTTP Server listening on port ${PORT}`);
  })
  .on('error', (err) => {
    console.error(`Server error`, err);
  });


// Handle server shutdown
process.on('SIGINT', async () => {
  console.log('Shutting down server...');

  // Close all active transports to properly clean up resources
  for (const sessionId in transports) {
    try {
      console.log(`Closing transport for session ${sessionId}`);
      await transports[sessionId].close();
      delete transports[sessionId];
    } catch (error) {
      console.error(`Error closing transport for session ${sessionId}:`, error);
    }
  }
  console.log('Server shutdown complete');
  process.exit(0);
});
```

针对这种有状态管理的代码，我们会生成一个sessionId，然后存下来，然后我们通过日志，可以看到第二次请求的时候，客户端会携带MCP标准的请求头的参数：

![image-20251228012605111](E:\CodeRepositroy\myWebBlog\docs\AI\MCP\image\image-20251228012605111.png)

这样，我们就基于TypeScript开发了一个Streamable HTTP模式的MCP Server，部署的时候也是通过node dist/index.js方式进行部署。

感兴趣的小伙伴可以去学习尝试下哦。目前这个Streamable HTTP模式的客户端SDK官方支持了Python、TypeScript，Java的暂时未提供，Spring AI也未提供，但是Spring AI Alibaba提供了Streamable HTTP模式。

## 三、总结



本文分享了基于TypeScript开发Streamable HTTP模式的MCP Server的开发教程，之所以使用TypeScript作为MCP的学习入门来说是非常方便的，非常容易和掌握开发一个MCP Server的步骤，也非常容易制作自己的第一个MCP Server，并发布到远程的市场。

MCP TypeScript-SDK是模型上下文协议的官方TypeScript实现，它为开发者提供了一套完整的工具，用于创建符合MCP规范的客户端和服务器应用。简单来说，这个SDK就像是MCP世界的"瑞士军刀"，帮助你：

创建MCP服务器，向AI模型提供各种资源和工具

开发MCP客户端，连接并使用这些资源和工具

处理所有MCP协议消息和生命周期事件

TypeScript-SDK的设计理念是"简单易用但功能强大"，即使你不是TypeScript专家，也能轻松上手并构建复杂的MCP应用。

这样我分享了MCP开发过程中的3种对接模式。

1. **标准输入/输出（stdio）**：适用于命令行工具和本地集成
2. **HTTP + SSE**：用于远程服务器通信
3. **可流式HTTP**：支持双向通信的现代HTTP传输

对比如下：

![image-20251228012619080](E:\CodeRepositroy\myWebBlog\docs\AI\MCP\image\image-20251228012619080.png)