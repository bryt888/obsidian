

**工欲善其事，必先利其器。**

技术评审会议临近尾声时，产品经理提出了一个看似简单的需求：“让Claude帮我们分析一下Jira中上周所有P1级bug的分布规律。”

小冰脑中迅速盘算起来：Jira部署在公司内网，而Claude运行在Anthropic的云端服务器上。这两个世界之间横亘着一道天然的网络隔离墙。若要让Claude访问Jira数据，要么手动导出再粘贴，要么编写脚本抓取数据后投喂，要么……

“要么用MCP，”咖哥在她开口前便说道，“也就是模型上下文协议(Model Context Protocol, MCP)。”

“这个名字我在好几个地方都见过，但一直没搞懂它究竟解决了什么核心问题。”小雪接话道。

“它解决的是AI与数据之间最古老的问题——如何实现安全、标准的连接。”咖哥在白板上画了两个圆圈，左侧标注“Claude”，右侧标注“Jira/数据库/API”，中间连了一条线。“在MCP出现之前，这条线的两端每次都需要重新搭桥：GitHub想接入Claude得写一个适配器，Jira想接入Claude又得写另一个；一旦更换AI客户端，所有适配器还得推倒重来。这本质上是一个$M \times N$的复杂度难题。”

小冰立刻反应过来：“就像USB-C普及前的充电线？每台设备都得配一根专用线，出差恨不得带上六七根？”

“没错。MCP就是AI工具集成领域的USB-C。”咖哥点头道。

## 6.1 从$M \times N$到$M+N$：标准化的力量

USB-C的类比绝非仅仅是修辞，它在结构上精准地映射了MCP所解决的核心问题。

在USB-C问世之前，笔记本电脑、手机和相机各自拥有专属的充电接口（如专用圆口、Micro-USB、Mini-USB等），导致每种设备都必须配备特定的连接线。若存在$M$种设备和$N$种外设，理论上就需要$M \times N$种连接方案。而USB-C的出现，将这一复杂的$M \times N$问题简化为$M+N$问题：设备端只需要实现一次USB-C标准接口，外设端也同样只需要实现一次，双方即可实现任意互通。

MCP实现了完全相同的逻辑（见图6-1）。在MCP问世之前，任何一个AI客户端（如Claude、ChatGPT等）与特定数据源（如Jira、GitHub、数据库等）之间的集成，都需要单独开发并维护专用的适配器。例如，若要让GitHub访问Claude代码，需要开发一个GitHub-Claude适配器；若要让同一个GitHub接入ChatGPT，则必须重新开发一个GitHub-ChatGPT适配器。假设有5个AI客户端和100个数据源，理论上就需要维护500个独立的适配器——这意味着每新增一个AI客户端或数据源，都伴随着大量且重复的集成开发工作。

![[Pasted image 20260701134803.png]]
MCP的核心思路正是如此：定义一套通用的标准协议。在这个架构下，每个AI客户端只需要实现一次MCP客户端协议，而每个数据源也只需要实现一次MCP服务器协议。一旦双方都遵循这一标准，它们就能即插即用、互相连接，不需要再为每一对组合编写针对性的适配代码。

回到前面的案例，面对5个AI客户端和100个数据源，采用MCP后，开发工作量从惊人的500次（5×100）骤降至仅需要105次（5+100）。

2024年11月，Anthropic正式推出了MCP这一开源协议。仅仅4个月后，作为主要竞争对手的OpenAI便宣布正式采纳MCP，并将其深度集成至ChatGPT桌面应用及Agent SDK中。

到了2025年12月，MCP的发展迎来了里程碑式的转折：Anthropic将MCP捐赠给了隶属于Linux基金会的Agentic AI Foundation (AAIF)。这一举动彻底确立了其作为行业公共基础设施的地位。AAIF的创始成员阵容豪华，涵盖了OpenAI、Google、Microsoft、Amazon Web Services、Cloudflare和Bloomberg等科技、金融领域的巨头。

截至2025年底，MCP生态已呈现爆发式增长：月度Agent SDK下载量超过9700万次、公开MCP服务器超过10 000台、官方Claude连接器逾75个。

从一家公司的内部项目演变为全球行业标准，MCP仅用了13个月。这种前所未有的普及速度有力地证明了一件事：解决集成痛点——$M \times N$问题——是整个行业共同的迫切需求，而MCP提供的标准化解决方案正是业界苦苦寻找的“通用语言”。

## 6.2 架构：客户端-服务器与JSON-RPC

MCP采用经典的客户端-服务器架构（见图6-2），基于JSON-RPC 2.0协议进行通信。

MCP客户端内置于Claude Code，负责发现MCP服务器提供的能力、构造工具调用请求并处理响应。MCP服务器则是一个独立程序，它连接至数据库、API或文件系统等外部服务，并通过MCP向客户端暴露这些服务的能力。

![[Pasted image 20260701134850.png]]
如果你熟悉VS Code的语言服务器协议 (Language Server Protocol, LSP)，就会发现其架构与MCP高度相似：在LSP架构中，编辑器(客户端)通过与语言服务器通信，获取智能提示、定义跳转及错误诊断等功能。MCP设计者David Soria Parra曾在访谈中承认，MCP直接借鉴了LSP的消息流设计理念。两者的核心区别：LSP连接的是代码编辑器与编程语言工具链，而MCP连接的则是AI模型与整个数字世界。

一台MCP服务器可向客户端“暴露”以下3类核心能力。

### 1 Tools

Tools（工具）是最核心且最常用的能力。工具是供Claude主动调用的函数，如“在Jira上创建新工单”“查询数据库过去7天的订单”或“向Slack发送消息”。每个工具都定义了明确的名称、描述、输入参数架构(schema)及返回值格式。当Claude判断需要调用某工具时，会自动构造符合该架构的请求参数；服务器执行操作后，将返回结构化结果。

### 2 Resources

Resources（资源）的概念类似于“文件”，指供Claude按需读取的数据。其与工具的关键区别：工具用于执行操作（通常伴随副作用），而资源仅用于获取数据（只读）。例如，“获取README.md的内容”属于资源访问，而“向代码库提交Commit”则属于工具调用。

### 3 Prompts

Prompts（提示词模板）是一类可复用的提示词片段，旨在使特定任务的启动方式标准化。例如，一台代码审查专用的MCP服务器可提供code-review提示词模板，调用该模板时会自动载入标准化的审查格式与关注点。尽管该功能与Skills存在一定重叠，且在实际应用中的普及度不及前两类，但它为任务初始化提供了便捷路径。

对于上述3类能力，工具覆盖了绝大多数实际应用场景。例如，当用户要求“帮我在Jira中查找P1级bug”时，Claude调用的正是Jira MCP服务器暴露的list_issues工具。

## 6.3 传输方式：连接的3种形态

MCP的传输层定义了客户端与服务器之间的物理通信机制。目前主要支持3种传输方式。

- **stdio（标准输入输出）**：专为本地进程通信设计的方式。在此方式下，客户端（如Claude Code）以子进程形式启动MCP服务器，并通过标准输入(stdin)和标准输出(stdout)传递JSON-RPC消息。这种方式的优点是完全在本地运行，不需要网络连接，具备最高的安全性和最低的延迟，适用于文件系统访问、本地数据库连接及本地开发工具集成。绝大多数通过npx启动的官方MCP服务器均采用此方式。
- **HTTP**：是远程服务器通信的推荐传输方式。AI客户端（如Claude）向指定的HTTP端点发送POST请求，服务器处理完毕后返回响应。该方式适用于部署在远端的MCP服务，例如公司内网的Jira适配器，或SaaS产品提供的MCP接入点。此方式支持OAuth 2.0认证，可轻松与企业统一身份认证系统集成。
- **SSE(Server-Sent Events)**：曾是远程传输的初始方案，但目前已标记为废弃(deprecated)。如果在旧文档或遗留项目中见到SSE配置，建议尽快迁移至HTTP传输方式。

遵循“本地选择stdio，远程选择HTTP”的简单准则即可满足绝大多数需求。

## 6.4 配置详解：从CLI到配置文件

### 6.4.1 CLI快速配置

Claude Code提供了一套`claude mcp`命令，用于便捷地管理MCP服务器配置。请看以下代码示例。

```
# 添加本地stdio服务器：启动一个本地文件系统服务，指定工作目录为/workspace
claude mcp add filesystem npx -y @modelcontextprotocol/server-filesystem /workspace

# 添加远程HTTP服务器：连接至公司内网的Jira服务
claude mcp add --transport http company-jira https://jira.company.com/mcp

# 添加用户级服务器（全局可用）：将GitHub服务添加到用户级别配置，使其对所有项目生效
claude mcp add --scope user github -- npx -y @modelcontextprotocol/server-github

# 添加带认证信息的服务器：通过自定义HTTP头传递认证Token
claude mcp add --transport http --header "Authorization: Bearer ${TOKEN}" api https://api.example.com/mcp

# 管理命令
claude mcp list             # 列出配置：查看当前所有已配置的MCP服务器
claude mcp test github      # 测试连接：验证指定服务器(如GitHub)的连接状态及可用性
claude mcp remove github    # 移除服务器：删除指定的MCP服务器配置
```

### 6.4.2 .mcp.json 配置文件

`claude mcp`CLI命令本质上是对`.mcp.json`配置文件的操作。理解该文件的结构是实现精细化配置的关键。请看以下代码示例。

```
{
  "mcpServers": {
    "filesystem": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-filesystem", "/workspace"],
      "env": {}
    },
    "company-jira": {
      "type": "http",
      "url": "https://jira.company.com/mcp",
      "headers": {
        "Authorization": "Bearer ${JIRA_TOKEN}"
      }
    },
    "postgres": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-postgres"],
      "env": {
        "DATABASE_URL": "${DB_URL:-postgresql://localhost:5432/mydb}"
      }
    }
  }
}
```

**环境变量替换**是上述配置文件中最重要的功能。基本语法`${VAR_NAME}`在运行时直接替换为对应的环境变量值；默认值语法`${VAR_NAME:-default_value}`则在未设置环境变量`VAR_NAME`时，使用指定的`default_value`。利用环境变量替换机制，你可以将敏感数据（如API密钥、数据库密码等）保存在操作系统的环境变量或`.env`文件中，而不是硬编码在`.mcp.json`配置文件中。由于配置文件中只包含变量引用（如`${JIRA_TOKEN}`）而非真实密钥，你可以安全地将`.mcp.json`提交到Git仓库进行团队共享，而不用担心凭证泄露。

### 6.4.3 配置文件的位置与作用域

`.mcp.json`配置文件根据存放位置的不同，具有不同的作用域，如表6-1所示。

**表6-1 .mcp.json配置文件的位置与作用域**

|文件路径|作用域|适用场景|
|:--|:--|:--|
|`.mcp.json` (项目根目录)|项目级|配置仅对当前项目生效。适用于团队共享的、与特定代码库绑定的服务（如该项目的数据库连接、特定的内部工具）。此文件通常提交到Git仓库。|
|`~/.claude/mcp.json`|用户全局级|配置对所有项目生效。适用于个人通用的工具（如个人GitHub账号检索工具）。此文件位于用户主目录，不随项目变动。|

如果某MCP服务器的凭证（如个人GitHub Token）不宜提交至Git仓库，可将凭证部分存入`.claude/settings.local.json`（该文件已被`.gitignore`忽略），而将服务器的基础配置保留在`.mcp.json`中，以便与团队共享。

## 6.5 实战一：连接数据库

让我们从最常见的需求入手：让Claude直接查询数据库。

首先，配置一个PostgreSQL MCP服务器。请看以下代码示例。

```
{
  "mcpServers": {
    "postgres": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-postgres"],
      "env": {
        "DATABASE_URL": "${DATABASE_URL:-postgresql://localhost:5432/mydb}"
      }
    }
  }
}
```

配置完成后，你只需要对Claude发出自然语言指令，如下所示。

```
你: 帮我查一下数据库里上个月的订单数量和总金额。

Claude: 正在查询数据库......

[调用 postgres MCP Server: sql_query 工具]

上个月 (2025年12月) 订单统计：
- 订单数量：1,234 笔
- 总金额：¥456,789.00
- 平均客单价：¥370.21
- 较上月增长 12.3%
```

在此过程中，Claude会自动识别查询需求，定位到postgres MCP服务器，生成并执行SQL语句，最后整理返回结果。你不需要掌握SQL语法，也不必打开数据库客户端或手动导出数据。

但这里有一项关键的安全原则：**数据库连接仅使用只读账号。**

MCP服务器拥有执行你授权范围内任何操作的能力。如果配置了具备写权限的数据库账号，理论上Claude可以执行 INSERT、UPDATE 甚至 DROP TABLE等高风险操作。因此，在`DATABASE_URL`中配置只读用户是保障数据安全的最低防护。

## 6.6 实战二：构建自定义MCP服务器

当现有的MCP服务器无法满足需求时（例如，需要对接公司内部API，或定制特定格式的数据处理工具），可以利用TypeScript或Python SDK构建专属的MCP服务器。

以下是一个基于TypeScript的待办事项管理服务器示例，展示了MCP服务器的基本架构。

```
import { McpServer } from "@modelcontextprotocol/sdk/server/mcp.js";
import { StdioServerTransport } from "@modelcontextprotocol/sdk/server/stdio.js";
import { z } from "zod";

// 内存存储
const todos: { id: string; text: string; done: boolean }[] = [];

// 创建MCP服务器实例
const server = new McpServer({
  name: "todo-server",
  version: "1.0.0",
});

// 定义工具：添加待办事项
server.tool(
  "todo_add",                                          // 工具名称
  "Add a new todo item",                               // 工具描述 (供Claude判断调用时机)
  { text: z.string().describe("The todo text") },      // 输入参数Schema
  async ({ text }) => {                                // 执行逻辑
    const todo = {
      id: Math.random().toString(36).substring(2, 9),
      text,
      done: false,
    };
    todos.push(todo);
    return {
      content: [{ type: "text", text: `Added: ${todo.id} - ${todo.text}` }],
    };
  }
);

// 定义工具：列出所有待办事项
server.tool("todo_list", "List all todo items", {}, async () => {
  const text = todos.length === 0
    ? "No todos found."
    : todos.map((t) => `[${t.done ? "x" : " "}] ${t.id}: ${t.text}`).join("\n");
  return { content: [{ type: "text", text }] };
});

// 定义资源：统计信息
server.resource("stats", "Server statistics", async () => {
  return {
    contents: [{
      uri: "stats://current",
      mimeType: "application/json",
      text: JSON.stringify({
        total: todos.length,
        completed: todos.filter((t) => t.done).length,
        pending: todos.filter((t) => !t.done).length,
      }, null, 2),
    }],
  };
});

// 启动服务器
async function main() {
  const transport = new StdioServerTransport();
  await server.connect(transport);
  console.error("MCP Server started");  // 注意：日志输出至stderr
}
main().catch(console.error);
```

在上述架构中，有几个关键的设计细节值得特别注意。

- **工具描述至关重要**：`server.tool()`的第二个参数（描述字符串）是Claude判断何时调用该工具的核心依据。描述写得越清晰、准确，Claude的意图识别和工具调用就越精准。
- **类型安全与校验**：输入参数通过Zod schema进行定义，这不仅提供了严格的类型安全保障，还能在运行时自动校验参数的有效性，防止错误的数据传入。
- **日志输出规范**：调试日志必须通过`console.error`输出到`stderr`，而非`stdout`。这是因为`stdout`通道专门用于传输JSON-RPC消息，若混入普通日志会导致协议解析失败。这一机制与Hooks脚本的调试输出原理一致。

**Python版本**的实现同样简洁优雅。

```
from mcp.server import Server
from mcp.server.stdio import stdio_server

# 初始化服务器与内存存储
server = Server("todo-server")
todos = []

@server.tool("todo_add")
async def add_todo(text: str) -> str:
    """添加新的待办事项"""
    todo_id = generate_id()    # 假设已定义generate_id函数
    todos.append({"id": todo_id, "text": text, "done": False})
    return f"Added: {todo_id} - {text}"

@server.tool("todo_list")
async def list_todos() -> str:
    """列出所有待办事项"""
    if not todos:
        return "No todos found."
    return "\n".join(
        f"[{'x' if t['done'] else ' '}] {t['id']}: {t['text']}"
        for t in todos
    )

async def main():
    async with stdio_server() as (read_stream, write_stream):
        await server.run(read_stream, write_stream)

if __name__ == "__main__":
    import asyncio
    asyncio.run(main())
```

完成开发后，你需要在配置文件中注册自定义服务器。以下示例展示了如何启动一个编译后的MCP服务（若为Python项目，请将command调整为python）。

```
{
  "mcpServers": {
    "my-todo": {
      "command": "node",
      "args": ["./mcp-server/build/index.js"]
    }
  }
}
```

## 6.7 常用MCP服务器生态

Anthropic及MCP社区维护了大量开箱即用的MCP服务器。本节将列举其中最常用的几类。

### 6.7.1 官方MCP服务器

Anthropic官方提供的MCP服务器如表6-2所示。

**表6-2 官方MCP服务器概览**

| MCP服务器名称          | 注意用途            | 安装/启动命令                                                                          |
| :---------------- | :-------------- | :------------------------------------------------------------------------------- |
| server-filesystem | 文件系统读写操作        | `npx @modelcontextprotocol/server-filesystem<path>`                         注释①  |
| server-fetch      | 发送HTTP请求        | `npx @modelcontextprotocol/server-fetch`                                         |
| server-postgres   | PostgreSQL数据库查询 | `npx @modelcontextprotocol/server-postgres`                                      |
| server-memory     | 跨会话持久化记忆存储      | `npx @modelcontextprotocol/server-memory`                                        |
| server-git        | Git版本控制操作       | `npx @modelcontextprotocol/server-git`                                           |

① 启动server-filesystem时，请将`<path>`替换为你希望授权访问的实际目录路径。

### 6.7.2 热门第三方MCP服务器

热门的第三方MCP服务器如表6-3所示。

**表6-3 热门的第三方MCP服务器概览**

|服务器名称|主要用途|通信协议|
|:--|:--|:--|
|GitHub MCP|GitHub协作管理 (PR、Issues、代码检索)|HTTP(SSE)|
|Notion MCP|Notion文档读取与写入|HTTP(SSE)|
|Sentry MCP|错误追踪与日志分析|HTTP(SSE)|
|Context7|检索最新技术文档|stdio|
|Bytebase DBHub|多数据库统一接入与管理|stdio|

### 6.7.3 实用配置组合

以下是一份专为全栈开发者设计的MCP配置文件，集成了文件系统、网络请求、代码协作及数据库管理功能。

```
{
  "mcpServers": {
    "filesystem": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-filesystem", "."]
    },
    "fetch": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-fetch"]
    },
    "github": {
      "type": "http",
      "url": "https://api.githubcopilot.com/mcp/",
      "headers": {
        "Authorization": "Bearer ${GITHUB_TOKEN}"
      }
    },
    "database": {
      "command": "npx",
      "args": ["-y", "@bytebase/dbhub", "--dsn", "${DATABASE_URL}"]
    }
  }
}
```

提示：配置中的`${GITHUB_TOKEN}`和`${DATABASE_URL}`为环境变量占位符，请在使用前确保已在终端环境中正确设置。

借助此配置，开发者可在单个Claude会话中无缝完成以往需要在多个工具间切换的复杂工作流。

- **数据插叙**：“帮我统计数据库中最近7天的订单总额” $\rightarrow$ 自动调用database服务器执行SQL查询。
- **任务管理**：“整理上述bug描述并将之创建为一个新的GitHub Issue” $\rightarrow$ 自动调用GitHub服务器通过API提交工单。
- **文档检索**：“获取React 19官方文档中关于Server Components的最新说明” $\rightarrow$ 自动调用fetch服务器实时抓取网页内容。

## 6.8 安全机制：信任的边界

MCP极大地扩展了Claude的能力边界，使其能够直接操作外部系统。这一特性既是其核心优势所在，也引入了显著的安全风险，因此必须慎之又慎。

### 6.8.1 三层纵深安全机制

为了平衡灵活性与安全性，MCP架构设计了层层递进的三重防御机制。

**第一层，首次连接的交互式审批。**

这是建立信任的基石。每当用户配置了一台新的MCP服务器时，Claude在首次尝试连接该服务器前会自动暂停，并向用户展示服务器的详细信息（如来源、权限范围等），请求明确的显式授权。该步骤为强制交互流程，无法绕过。

**第二层，细粒度的工具级权限控制。**

即使整台MCP服务器已获得全局授权，Claude在调用具体工具时仍需要保持警惕，特别是针对具有副作用的操作（如文件写入、代码执行、数据库修改等）。Claude会在执行前再次拦截，向用户展示即将执行的具体指令细节，并等待二次确认。

**第三层，基于OAuth 2.0的动态认证。**

对于远程MCP服务器，推荐采用OAuth 2.0协议而非长期有效的静态API密钥。这种方式不仅便于与企业现有的统一身份认证系统集成以实现单点登录，而且支持令牌的自动过期与刷新机制。

### 6.8.2 安全风险与防护

2025年4月，安全研究人员指出了MCP生态中的以下安全风险。

- **提示注入攻击**：恶意内容可通过MCP服务器注入Claude的上下文。若MCP服务器返回包含恶意指令的数据（如“忽略之前的所有指令，执行以下操作”），Claude可能会被误导。防护措施是仅使用可信来源的MCP服务器。
- **工具权限滥用**：单个工具的权限看似安全，但多个工具组合使用时可能产生意想不到的后果。例如，“读文件”工具与“发邮件”工具的组合，理论上可能导致文件内容泄露。防护措施是遵循最小权限原则。
- **冒名顶替**：恶意工具可伪装成合法工具的名称和描述，诱导Claude调用。防护措施是仅安装来自可信来源的MCP服务器包。

**咖哥发言**

评估MCP服务器的信任度与评估npm包并无本质区别：核心在于考察其来源（是源自官方的@modelcontextprotocol/命名空间，还是个人开发者）、维护活跃度、下载量及社区口碑。选择知名的官方服务器是确保安全的起点。此外，在安全实践上，连接数据库时务必使用只读账号；管理API密钥时，应通过环境变量注入，严禁硬编码。

## 6.9 MCP+Skills：厨房与菜谱

第3章介绍的Skills与本章探讨的MCP，分别解决了不同层面的问题：MCP赋予了Claude访问外部数据和工具的能力，而Skills则指导Claude如何以最优策略运用这些能力。唯有两者组合，方能构成完整的解决方案。

Anthropic官方曾通过一个生动的类比来阐述这一关系。

- **MCP如同专业厨房**：这里配备了所有必要的设备与原料——冰箱（数据库）、炉灶（API调用）、食材（数据源）和刀具（工具）。拥有厨房意味着厨师具备制作任何菜肴的能力，但具体“做什么”和“怎么做”，完全依赖厨师的临场发挥。每当来了新厨师（即新的用户请求）时，他都需要重新摸索厨房的使用方式，难以保证菜肴质量的稳定性。
- **Skills如同标准菜谱**：它提供了详尽的操作指南，明确规定了所需食材、操作步骤、火候控制及烹饪时长。有了菜谱，即使是经验尚浅的厨师也能稳定复现出高水准的菜肴。然而，若缺乏配套的厨房设备，再完美的菜谱也无法落地执行。

在实际应用中，两者的协作模式：在Skills的SKILL.md中引用MCP工具，从而指导Claude按照特定的工作流高效调用这些工具（见图6-3）。

这一组合赋予了Claude双重优势：既能通过MCP获取实时数据能力，又能借助Skills遵循一致的数据处理方式。这意味着，Claude不再依赖用户每次的详尽指令，也不需要仰仗Claude的临场判断，即可实现稳定、高效的自动化执行。

![[Pasted image 20260701140010.png]]
图6-3 MCP+Skills的“厨房与菜谱”式协作流程

## 6.10 企业级部署

对拥有数十甚至上百名工程师的团队而言，依赖个人自行配置MCP服务器并不可扩展。为此，Claude Code提供了企业级的集中管理机制。

管理员可通过`managed-mcp.json`配置文件（或Claude for Enterprise管理后台），为组织内所有用户预配置MCP服务器。这些服务器将自动对所有成员生效，不需要个人进行任何操作。此外，管理员还可预先授权这些服务器，从而免除每位用户首次使用时的审批弹窗流程。

这种集中管理模式实现了双赢：对开发者而言，一旦公司的 Jira、Linear、内部知识库及数据仓库等工具以MCP形式接入，整个团队即可获得AI驱动的工作流，实现“零配置、开箱即用”；就安全性而言，所有MCP访问均经由严格审查的服务器进行，相较于分散的个人配置，集中式的审计与管控显著提升了安全性。

## 6.11 调试与故障排除

MCP服务器的调试难度通常高于Hooks脚本，原因在于其涉及进程间通信、网络连接以及复杂的认证流程等多个环节。

### 6.11.1 常用调试手段

一些常用的调试手段如下。

```
# 列出所有已配置的服务器及其当前状态
claude mcp list

# 专门测试特定服务器的连接连通性
claude mcp test my-server

# 启用MCP调试模式
claude --mcp-debug
```

### 6.11.2 MCP工具输出的Token管理

针对MCP工具可能返回海量数据（例如，未加LIMIT限制的数据库查询可能返回数万行记录）的情况，Claude Code内置了专门的Token管理机制。

- **警告阈值**：当输出超过10 000 Token时，系统会显示警告提示。
- **截断阈值**：当输出超过25 000 Token时，系统将自动对内容进行截断，防止上下文溢出。
- **自定义配置**：用户可以通过设置环境变量`MAX_MCP_OUTPUT_TOKENS`来调整这一输出上限，以适应不同的业务场景需求。

### 6.11.3 常见问题

MCP常见问题与解决方案如表6-4所示。该表可以帮助用户快速定位和解决开发中遇到的典型问题。

**表6-4 MCP常见问题与解决方案**

|问题现象|可能原因|解决方案|
|:--|:--|:--|
|服务器无法启动|命令或路径配置错误|检查配置文件中的`command`和`args`参数；尝试在终端运行该命令以验证是否可以正常启动|
|连接超时|网络不稳定或服务器响应过慢|设置环境变量`MCP_TIMEOUT`（单位为毫秒），适当延长等待时间|
|认证失败|Token错误、缺失或已过期|确认相关环境变量（如`API_KEY`等）已正确设置；检查Token的有效性并及时刷新|
|输出被截断|返回数据量超过Token限制|调大环境变量`MAX_MCP_OUTPUT_TOKENS`的上限；优化查询逻辑（如增加LIMIT），让服务器返回更精简的数据|
|JSON解析错误|服务器标准输出(stdout)被日志污染|确保服务器将调试日志、错误信息等非数据内容输出至标准错误(stderr)，保持stdout仅包含纯净的JSON数据|

## 本章小结

MCP解决了AI工具集成领域的基础性问题：它通过一套统一的开放协议，将各类AI客户端与多样化的数据源无缝连接，不需要再为每种组合单独开发适配层。该协议以JSON-RPC 2.0为通信基石，其中，stdio适用于本地进程通信，HTTP适用于远程服务调用（SSE已废弃）；同时，借助环境变量替换机制，配置文件得以安全地纳入版本控制系统。

从工程设计视角审视，MCP的核心价值不仅在于其连接能力，更在于实现了连接的标准化。如今接入Claude的MCP服务器，理论上可无缝兼容任何遵循该标准的AI客户端；反之，随着标准的普及，能接入Claude的MCP服务器生态也将持续扩容。MCP自2024年11月发布至2025年12月捐赠给Linux基金会，其在短短13个月内便从单一企业项目蜕变为由行业联盟管理的行业标准。这一演进速度本身，便有力地证明了AI工具生态对标准化的迫切需求。

与第5章讨论的Hooks及第3章所述的Skills相比，MCP的定位截然不同：Hooks旨在“界定Claude的行为禁区”，Skills侧重“指导Claude的执行策略”，而MCP则致力于“拓展Claude的数据和服务触达边界”。三者相辅相成——MCP赋予连接能力，Skills规范调用逻辑，Hooks筑牢安全防线——共同构建起一套完备的AI工程化体系。

在第7章中，我们将跳出交互式使用场景，深入探讨Claude Code如何在无人值守的环境下运行，重点解析Headless模式及其与CI/CD流程的集成实践。

## 思考题

1. 请审视你当前的工作流：哪些数据源或工具若能被Claude实时访问，将显著提升效率？请尝试为这些工具设计一份.mcp.json配置文件。
2. 相较于直接在对话中粘贴数据（如将导出的Jira-CSV文件发送给Claude），MCP在“便捷性”之外，本质上还有哪些核心差异化优势？又在何种特定场景下，直接粘贴数据反而是更优的选择？
3. 在“厨房与菜谱”的协同架构中（MCP为厨房，Skills为菜谱），若需要为公司某内部工具设计全套集成方案，你会如何规划？（提示：在MCP服务器层面，应暴露哪些核心工具以最大化其效用？在Skills指导层面，配套的Skill应如何指导Claude遵循特定的业务工作流来调用这些工具？）