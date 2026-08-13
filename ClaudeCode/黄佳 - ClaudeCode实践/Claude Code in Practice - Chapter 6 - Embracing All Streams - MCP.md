
## Embracing All Streams: MCP

**Good tools are essential for doing a good job.**
As the technical review meeting was drawing to a close, the product manager proposed a seemingly simple requirement: "Let Claude help us analyze the distribution pattern of all P1 level bugs in Jira from last week." Xiao Bing quickly calculated in her mind: Jira is deployed on the company's intranet, while Claude runs on Anthropic's cloud servers. There is a natural network isolation wall between these two worlds. If Claude is to access Jira data, one must either manually export and paste it, write a script to scrape the data and feed it, or... "Or use MCP," Brother Ka said before she could open her mouth, "which is the Model Context Protocol (MCP)." "I've seen this name in several places, but I never understood what core problem it actually solves," Xiao Xue chimed in. "It solves the oldest problem between AI and data—how to achieve a secure and standard connection." Brother Ka drew two circles on the whiteboard, labeling the left one "Claude", the right one "Jira/Database/API", and drew a line connecting them in the middle. "Before MCP appeared, a bridge had to be rebuilt every time for the two ends of this line: GitHub had to write an adapter to connect to Claude, and Jira had to write another adapter to connect to Claude; once the AI client was changed, all adapters had to be completely rebuilt. This is essentially an $M \times N$ complexity puzzle."

Xiao Bing immediately reacted: "Just like the charging cables before the popularization of USB-C? Every device had to have a dedicated cable, and when going on a business trip, you'd wish you could bring six or seven cables?" "Exactly. MCP is the USB-C of the AI tool integration field," Brother Ka nodded.

#### 6.1 From $M \times N$ to $M+N$: The Power of Standardization

The USB-C analogy is by no means just rhetoric; it structurally and accurately maps the core problem solved by MCP. Before the advent of USB-C, laptops, mobile phones, and cameras each had their own exclusive charging interfaces (such as proprietary round ports, Micro-USB, Mini-USB, etc.), resulting in the need to equip each device with a specific cable. If there are $M$ kinds of devices and $N$ kinds of peripherals, in theory, $M \times N$ connection schemes would be needed. And the emergence of USB-C simplified this complex $M \times N$ problem into an $M+N$ problem: the device side only needs to implement the USB-C standard interface once, and the peripheral side also only needs to implement it once, so that both parties can achieve universal interoperability. MCP realizes the exact same logic (see Figure 6-1). Before MCP was born, the integration between any AI client (such as Claude, ChatGPT, etc.) and a specific data source (such as Jira, GitHub, databases, etc.) required the independent development and maintenance of a dedicated adapter. For example, if you want GitHub to access Claude code, you need to develop a GitHub-Claude adapter; if you want the same GitHub to connect to ChatGPT, you must re-develop a GitHub-ChatGPT adapter. Assuming there are 5 AI clients and 100 data sources, in theory, 500 independent adapters would need to be maintained—which means every time a new AI client or data source is added, it is accompanied by a massive and repetitive amount of integration development work.

![[Pasted image 20260701134803.png]] The core idea of MCP is exactly this: define a set of universal standard protocols. Under this architecture, each AI client only needs to implement the MCP client protocol once, and each data source also only needs to implement the MCP server protocol once. Once both parties follow this standard, they can be plug-and-play and connect to each other, without needing to write targeted adapter code for each pair combination. Returning to the previous case, facing 5 AI clients and 100 data sources, after adopting MCP, the development workload plummeted from a staggering 500 times (5×100) to only needing 105 times (5+100). In November 2024, Anthropic officially launched the open-source protocol MCP. Just 4 months later, OpenAI, as its main competitor, announced its official adoption of MCP and deeply integrated it into the ChatGPT desktop application and Agent SDK. By December 2025, the development of MCP welcomed a milestone turning point: Anthropic donated MCP to the Agentic AI Foundation (AAIF), which is affiliated with the Linux Foundation. This move thoroughly established its position as the public infrastructure of the industry. The lineup of founding members of AAIF is luxurious, covering tech and financial giants such as OpenAI, Google, Microsoft, Amazon Web Services, Cloudflare, and Bloomberg.

By the end of 2025, the MCP ecosystem had shown explosive growth: monthly Agent SDK downloads exceeded 97 million, public MCP servers surpassed 10,000, and official Claude connectors numbered over 75. Evolving from an internal company project to a global industry standard, MCP took only 13 months. This unprecedented speed of popularization strongly proves one thing: solving the integration pain point—the $M \times N$ problem—is an urgent and common need of the entire industry, and the standardized solution provided by MCP is exactly the "universal language" the industry has been painstakingly searching for.

#### 6.2 Architecture: Client-Server and JSON-RPC

MCP adopts a classic client-server architecture (see Figure 6-2) and communicates based on the JSON-RPC 2.0 protocol. The MCP client is built into Claude Code, responsible for discovering the capabilities provided by the MCP server, constructing tool invocation requests, and handling responses. The MCP server, on the other hand, is an independent program that connects to external services such as databases, APIs, or file systems, and exposes the capabilities of these services to the client through MCP. ![[Pasted image 20260701134850.png]] If you are familiar with VS Code's Language Server Protocol (LSP), you will find that its architecture is highly similar to MCP: in the LSP architecture, the editor (client) communicates with the language server to obtain features such as intelligent prompts, definition jumps, and error diagnostics. David Soria Parra, the designer of MCP, once admitted in an interview that MCP directly borrowed the message flow design philosophy of LSP. The core difference between the two is: LSP connects the code editor with the programming language toolchain, while MCP connects the AI model with the entire digital world.

An MCP server can "expose" the following 3 types of core capabilities to the client.

##### 1 Tools

Tools are the most core and commonly used capability. Tools are functions provided for Claude to call actively, such as "create a new issue in Jira", "query the orders of the past 7 days from the database", or "send a message to Slack". Each tool defines a clear name, description, input parameter schema, and return value format. When Claude determines that a certain tool needs to be called, it will automatically construct request parameters that conform to that schema; after the server executes the operation, it will return a structured result.

##### 2 Resources

The concept of Resources is similar to "files", referring to data provided for Claude to read on demand. Its key difference from tools is: tools are used to execute operations (usually accompanied by side effects), while resources are only used to obtain data (read-only). For example, "get the content of README.md" belongs to resource access, while "submit a Commit to the codebase" belongs to tool invocation.

##### 3 Prompts

Prompts (prompt templates) are a type of reusable prompt snippets aimed at standardizing the initialization methods of specific tasks. For example, a dedicated MCP server for code review can provide a `code-review` prompt template, and when calling this template, the standardized review format and areas of focus will be automatically loaded. Although this function overlaps to a certain extent with Skills and is less popular in practical applications than the first two categories, it provides a convenient path for task initialization.

For the above 3 types of capabilities, tools cover the vast majority of practical application scenarios. For example, when a user asks "help me find P1 level bugs in Jira", what Claude calls is exactly the `list_issues` tool exposed by the Jira MCP server.

#### 6.3 Transmission Methods: The 3 Forms of Connection

The transmission layer of MCP defines the physical communication mechanism between the client and the server. Currently, it mainly supports 3 transmission methods.

- **stdio (Standard Input and Output)**: A method designed specifically for local inter-process communication. Under this method, the client (such as Claude Code) starts the MCP server as a subprocess, and passes JSON-RPC messages through standard input (stdin) and standard output (stdout). The advantage of this method is that it runs completely locally, does not require a network connection, and possesses the highest security and lowest latency, making it suitable for file system access, local database connections, and local development tool integrations. The vast majority of official MCP servers started via npx adopt this method.
- **HTTP**: The recommended transmission method for remote server communication. The AI client (such as Claude) sends POST requests to specified HTTP endpoints, and the server returns a response after processing. This method is suitable for remotely deployed MCP services, such as a Jira adapter on a company intranet or an MCP access point provided by a SaaS product. This method supports OAuth 2.0 authentication and can be easily integrated with enterprise unified identity authentication systems.
- **SSE (Server-Sent Events)**: Was once the initial solution for remote transmission, but is currently marked as deprecated. If you see SSE configurations in old documents or legacy projects, it is recommended to migrate to the HTTP transmission method as soon as possible.

Following the simple rule of "choose stdio for local, choose HTTP for remote" can meet the vast majority of needs.

#### 6.4 Configuration Details: From CLI to Configuration Files

##### 6.4.1 CLI Quick Configuration

Claude Code provides a set of `claude mcp` commands for conveniently managing MCP server configurations. Please see the following code example.

##### 6.4.2 .mcp.json Configuration File

The `claude mcp` CLI command is essentially operating on the `.mcp.json` configuration file. Understanding the structure of this file is the key to achieving fine-grained configuration. Please see the following code example. **Environment variable substitution** is the most important function in the above configuration file. The basic syntax `${VAR_NAME}` is directly replaced with the corresponding environment variable value at runtime; the default value syntax `${VAR_NAME:-default_value}` uses the specified `default_value` when the environment variable `VAR_NAME` is not set. Utilizing the environment variable substitution mechanism, you can save sensitive data (such as API keys, database passwords, etc.) in the operating system's environment variables or `.env` files, rather than hardcoding them in the `.mcp.json` configuration file. Since the configuration file only contains variable references (such as `${JIRA_TOKEN}`) rather than real keys, you can safely commit `.mcp.json` to a Git repository for team sharing without worrying about credential leaks.

##### 6.4.3 Location and Scope of Configuration Files

Depending on their storage locations, `.mcp.json` configuration files have different scopes, as shown in Table 6-1. **Table 6-1 Locations and Scopes of .mcp.json Configuration Files**

|File Path|Scope|Applicable Scenarios|
|---|---|---|
|.mcp.json (Project root directory)|Project level|The configuration is only effective for the current project. Suitable for team-shared services tied to a specific codebase (like the project's database connection, specific internal tools). This file is usually committed to the Git repository.|
|~/.claude/mcp.json|User global level|The configuration is effective for all projects. Suitable for personal general tools (like a personal GitHub account retrieval tool). This file is located in the user's home directory and does not change with projects.|

If the credentials of a certain MCP server (such as a personal GitHub Token) are not suitable to be committed to the Git repository, the credential part can be stored in `.claude/settings.local.json` (this file has been ignored by `.gitignore`), while the basic configuration of the server is retained in `.mcp.json` so as to be shared with the team.

#### 6.5 Practice One: Connecting to a Database

Let's start with the most common need: letting Claude query the database directly. First, configure a PostgreSQL MCP server. Please see the following code example. After the configuration is complete, you only need to issue natural language instructions to Claude, as shown below. During this process, Claude will automatically identify the query requirement, locate the `postgres` MCP server, generate and execute the SQL statement, and finally format the returned results. You do not need to master SQL syntax, nor do you need to open a database client or manually export data.

But there is a key security principle here: **Database connections must only use read-only accounts.** The MCP server possesses the ability to execute any operation within the scope of your authorization. If a database account with write permissions is configured, in theory, Claude can execute high-risk operations such as `INSERT`, `UPDATE`, or even `DROP TABLE`. Therefore, configuring a read-only user in `DATABASE_URL` is the minimum protection for ensuring data security.

#### 6.6 Practice Two: Building a Custom MCP Server

When existing MCP servers cannot meet the requirements (for example, needing to dock with a company's internal API, or customize data processing tools of a specific format), you can leverage the TypeScript or Python SDK to build an exclusive MCP server. Below is an example of a TypeScript-based to-do list management server, demonstrating the basic architecture of an MCP server. In the above architecture, there are several key design details worthy of special attention.

- **Tool descriptions are crucial**: The second parameter (description string) of `server.tool()` is the core basis for Claude to judge when to call the tool. The clearer and more accurate the description is written, the more precise Claude's intent recognition and tool invocation will be.
- **Type safety and validation**: Input parameters are defined through Zod schemas, which not only provides strict type safety guarantees but also automatically validates the validity of parameters at runtime to prevent erroneous data input.
- **Log output specifications**: Debug logs must be output to stderr via `console.error`, not stdout. This is because the stdout channel is specifically used for transmitting JSON-RPC messages, and if ordinary logs are mixed in, it will lead to protocol parsing failures. This mechanism is consistent with the debugging output principle of Hooks scripts.

**The Python version** implementation is equally concise and elegant. After development is complete, you need to register the custom server in the configuration file. The following example shows how to start a compiled MCP service (if it's a Python project, please adjust the command to `python`).

#### 6.7 Common MCP Server Ecosystem

Anthropic and the MCP community maintain a large number of out-of-the-box MCP servers. This section will list some of the most commonly used ones.

##### 6.7.1 Official MCP Servers

The MCP servers officially provided by Anthropic are shown in Table 6-2. **Table 6-2 Overview of Official MCP Servers**

|MCP Server Name|Primary Use|Installation/Startup Command|
|---|---|---|
|server-filesystem|File system read and write operations|npx @modelcontextprotocol/server-filesystem<path> Note ①|
|server-fetch|Send HTTP requests|npx @modelcontextprotocol/server-fetch|
|server-postgres|PostgreSQL database query|npx @modelcontextprotocol/server-postgres|
|server-memory|Cross-session persistent memory storage|npx @modelcontextprotocol/server-memory|
|server-git|Git version control operations|npx @modelcontextprotocol/server-git|

① When starting `server-filesystem`, please replace `<path>` with the actual directory path you wish to authorize access to.

##### 6.7.2 Popular Third-Party MCP Servers

Popular third-party MCP servers are shown in Table 6-3. **Table 6-3 Overview of Popular Third-Party MCP Servers**

|Server Name|Primary Use|Communication Protocol|
|---|---|---|
|GitHub MCP|GitHub collaboration management (PRs, Issues, code search)|HTTP(SSE)|
|Notion MCP|Notion document reading and writing|HTTP(SSE)|
|Sentry MCP|Error tracking and log analysis|HTTP(SSE)|
|Context7|Retrieving the latest technical documents|stdio|
|Bytebase DBHub|Unified access and management of multiple databases|stdio|

##### 6.7.3 Practical Configuration Combinations

Below is an MCP configuration file designed specifically for full-stack developers, integrating file systems, network requests, code collaboration, and database management functions. Tip: `${GITHUB_TOKEN}` and `${DATABASE_URL}` in the configuration are environment variable placeholders; please ensure they are correctly set in the terminal environment before use. With this configuration, developers can seamlessly complete complex workflows in a single Claude session that previously required switching between multiple tools.

- **Data episode**: "Help me calculate the total order amount in the database for the last 7 days" $\rightarrow$ Automatically calls the `database` server to execute the SQL query.
- **Task management**: "Organize the bug descriptions above and create them as a new GitHub Issue" $\rightarrow$ Automatically calls the `GitHub` server to submit a ticket via API.
- **Document retrieval**: "Get the latest explanation about Server Components from the React 19 official documentation" $\rightarrow$ Automatically calls the `fetch` server to crawl web content in real-time.

#### 6.8 Security Mechanisms: The Boundaries of Trust

MCP has greatly expanded the capability boundaries of Claude, enabling it to directly manipulate external systems. While this feature is its core advantage, it also introduces significant security risks, which must be treated with the utmost caution.

##### 6.8.1 Three-Layer Defense-in-Depth Security Mechanism

To balance flexibility and security, the MCP architecture designs a progressive three-layer defense mechanism. **The first layer, interactive approval upon the first connection.** This is the cornerstone of establishing trust. Whenever a user configures a new MCP server, Claude will automatically pause before its first attempt to connect to the server, present the server's detailed information (such as source, permission scope, etc.) to the user, and request explicit authorization. This step is a mandatory interactive process and cannot be bypassed. **The second layer, fine-grained tool-level permission control.** Even if the entire MCP server has obtained global authorization, Claude still needs to remain vigilant when calling specific tools, especially for operations with side effects (such as file writing, code execution, database modification, etc.). Claude will intercept again before execution, present the specific instruction details to be executed to the user, and wait for secondary confirmation. **The third layer, dynamic authentication based on OAuth 2.0.** For remote MCP servers, it is recommended to adopt the OAuth 2.0 protocol rather than static API keys with long-term validity. This method not only facilitates integration with existing enterprise unified identity authentication systems to achieve single sign-on, but also supports automatic expiration and refreshing mechanisms for tokens.

##### 6.8.2 Security Risks and Defenses

In April 2025, security researchers pointed out the following security risks in the MCP ecosystem.

- **Prompt Injection Attacks**: Malicious content can be injected into Claude's context via the MCP server. If the MCP server returns data containing malicious instructions (like "ignore all previous instructions and execute the following"), Claude might be misled. The defense measure is to only use MCP servers from trusted sources.
- **Tool Permission Abuse**: The permission of a single tool might seem safe, but when multiple tools are combined, they might produce unexpected consequences. For example, the combination of a "read file" tool and a "send email" tool could theoretically lead to file content leakage. The defense measure is to follow the Principle of Least Privilege.
- **Impersonation**: Malicious tools can disguise themselves under the names and descriptions of legitimate tools to induce Claude to call them. The defense measure is to only install MCP server packages from trusted sources. **Brother Ka's Remarks** Evaluating the trust level of an MCP server is essentially no different from evaluating an npm package: the core lies in examining its source (whether it originates from the official `@modelcontextprotocol/` namespace or individual developers), maintenance activity, download volume, and community reputation. Choosing well-known official servers is the starting point to ensure security. Furthermore, in security practices, when connecting to databases, you must use read-only accounts; when managing API keys, they should be injected via environment variables, and hardcoding is strictly prohibited.

#### 6.9 MCP + Skills: The Kitchen and the Recipe

Skills introduced in Chapter 3 and MCP explored in this chapter respectively address problems at different levels: MCP grants Claude the capability to access external data and tools, while Skills guide Claude on how to apply these capabilities using optimal strategies. Only when the two are combined can a complete solution be formed.

Anthropic officially used a vivid analogy to illustrate this relationship.

- **MCP is like a professional kitchen**: It is equipped with all necessary equipment and raw materials—refrigerators (databases), stoves (API calls), ingredients (data sources), and knives (tools). Having a kitchen means the chef has the capability to make any dish, but "what to make" and "how to make it" rely entirely on the chef's improvisation. Whenever a new chef (i.e., a new user request) arrives, they need to figure out how to use the kitchen from scratch, making it difficult to guarantee the stability of the dishes' quality.
- **Skills are like standard recipes**: They provide detailed operational guidelines, explicitly specifying required ingredients, operation steps, heat control, and cooking duration. With a recipe, even a less experienced chef can stably reproduce high-standard dishes. However, lacking the supporting kitchen equipment, even the most perfect recipe cannot be executed. In actual applications, the collaboration mode of the two is: reference MCP tools in the `SKILL.md` of Skills, thereby guiding Claude to efficiently call these tools according to a specific workflow (see Figure 6-3). This combination grants Claude a dual advantage: it can not only acquire real-time data capabilities through MCP, but also follow consistent data processing methods with the help of Skills. This means that Claude no longer relies on the user's detailed instructions every time, nor does it need to rely on its own improvisation, to achieve stable, efficient automated execution. ![[Pasted image 20260701140010.png]] Figure 6-3 The "Kitchen and Recipe" Collaborative Workflow of MCP+Skills

#### 6.10 Enterprise-Level Deployment

For teams with dozens or even hundreds of engineers, relying on individuals to configure MCP servers themselves is not scalable. For this reason, Claude Code provides an enterprise-level centralized management mechanism. Administrators can preconfigure MCP servers for all users within the organization via the `managed-mcp.json` configuration file (or the Claude for Enterprise management backend). These servers will automatically take effect for all members, requiring no personal action. Furthermore, administrators can also pre-authorize these servers, thereby eliminating the approval popup process during each user's first use. This centralized management mode achieves a win-win: for developers, once tools like the company's Jira, Linear, internal knowledge base, and data warehouse are integrated in the form of MCP, the entire team can gain AI-driven workflows, achieving a "zero-configuration, out-of-the-box" experience; in terms of security, all MCP accesses go through strictly vetted servers, and compared to dispersed personal configurations, centralized auditing and control significantly enhance security.

#### 6.11 Debugging and Troubleshooting

The debugging difficulty of MCP servers is usually higher than that of Hooks scripts, because it involves multiple links such as inter-process communication, network connections, and complex authentication processes.

##### 6.11.1 Common Debugging Methods

Some commonly used debugging methods are as follows.

##### 6.11.2 Token Management for MCP Tool Outputs

In response to the situation where MCP tools might return massive amounts of data (for example, a database query without a `LIMIT` restriction might return tens of thousands of records), Claude Code has built-in specialized Token management mechanisms.

- **Warning Threshold**: When the output exceeds 10,000 Tokens, the system will display a warning prompt.
- **Truncation Threshold**: When the output exceeds 25,000 Tokens, the system will automatically truncate the content to prevent context overflow.
- **Custom Configuration**: Users can adjust this upper limit by setting the environment variable `MAX_MCP_OUTPUT_TOKENS` to adapt to the needs of different business scenarios.

##### 6.11.3 Common Problems

Common MCP problems and solutions are shown in Table 6-4. This table can help users quickly locate and solve typical problems encountered in development. **Table 6-4 Common MCP Problems and Solutions**

|Problem Phenomenon|Possible Cause|Solution|
|---|---|---|
|Server cannot start|Command or path configuration error|Check the `command` and `args` parameters in the configuration file; try running the command in the terminal to verify if it can start normally|
|Connection timeout|Network instability or server response too slow|Set the environment variable `MCP_TIMEOUT` (in milliseconds) to appropriately extend the waiting time|
|Authentication failed|Token error, missing, or expired|Confirm that relevant environment variables (like `API_KEY`, etc.) are correctly set; check the validity of the Token and refresh it promptly|
|Output truncated|Returned data volume exceeds Token limit|Increase the upper limit of the environment variable `MAX_MCP_OUTPUT_TOKENS`; optimize query logic (like adding `LIMIT`) to let the server return more streamlined data|
|JSON parsing error|Server standard output (stdout) is polluted by logs|Ensure the server outputs non-data content like debug logs and error messages to standard error (stderr), keeping stdout containing only pure JSON data|

#### Chapter Summary

MCP solves the foundational problem in the AI tool integration field: through a unified open protocol, it seamlessly connects various AI clients with diversified data sources, eliminating the need to independently develop adapter layers for each combination. The protocol uses JSON-RPC 2.0 as the communication cornerstone, where stdio applies to local inter-process communication, and HTTP applies to remote service calls (SSE is deprecated); meanwhile, leveraging the environment variable substitution mechanism, configuration files can be safely incorporated into version control systems. Examining from the perspective of engineering design, the core value of MCP lies not only in its connection capability but more importantly in achieving the standardization of connections. Today, MCP servers connected to Claude can theoretically be seamlessly compatible with any AI client following the standard; conversely, with the popularization of the standard, the ecosystem of MCP servers capable of connecting to Claude will also continue to expand. From its release in November 2024 to its donation to the Linux Foundation in December 2025, MCP transformed from a single enterprise project into an industry standard managed by an industry alliance in just 13 months. This evolution speed itself powerfully proves the AI tool ecosystem's urgent need for standardization. Compared to Hooks discussed in Chapter 5 and Skills described in Chapter 3, the positioning of MCP is completely different: Hooks aim to "define Claude's behavioral forbidden zones," Skills focus on "guiding Claude's execution strategies," while MCP is dedicated to "expanding Claude's data and service reach boundaries." The three complement each other—MCP grants connection capabilities, Skills standardize invocation logic, and Hooks build solid security defense lines—jointly constructing a complete AI engineering system.

In Chapter 7, we will step out of interactive use scenarios and delve deeply into how Claude Code runs in unattended environments, focusing on analyzing the Headless mode and its integration practices with CI/CD processes.

#### Thought Questions

1. Please examine your current workflow: which data sources or tools, if accessible in real-time by Claude, would significantly improve efficiency? Please try to design an `.mcp.json` configuration file for these tools.
2. Compared to directly pasting data into a conversation (like sending an exported Jira-CSV file to Claude), what core differential advantages does MCP essentially have besides "convenience"? And under what specific scenarios is directly pasting data actually the better choice?
3. In the collaborative architecture of "kitchen and recipe" (MCP being the kitchen, Skills being the recipe), if you need to design a complete integration scheme for a certain internal company tool, how would you plan it? (Hint: At the MCP server level, which core tools should be exposed to maximize their utility? At the Skills guidance level, how should the supporting Skill guide Claude to follow specific business workflows to call these tools?)