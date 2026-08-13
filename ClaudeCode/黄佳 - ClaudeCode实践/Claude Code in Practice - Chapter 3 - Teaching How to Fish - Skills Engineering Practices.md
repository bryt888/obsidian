
## Teaching How to Fish: Skills Engineering Practices

**Practice makes perfect, but teaching saves labor.** 
Monday morning, Xiao Bing started a brand new Claude Code session to handle the first task of the week—generating API documentation for the payment module. She first spent 3 minutes clarifying the context: the team follows RESTful standards, the document needs to use the OpenAPI 3.0 format, error codes should align with internal company standards, sample data must be real and valid, and the output content needs to be bilingual in Chinese and English. Only after feeding the standard document written by a colleague last week to the model as a reference did she formally instruct it to start working. It was like this last week, and the week before that. Every time a new conversation is started, these prerequisite knowledge points need to be "fed" to Claude again. Even more frustratingly, even in the same session, when the task switches from "generating API documentation" to "reviewing code quality," Claude cannot automatically adapt to the corresponding work mode—she has to spend an extra round of conversation to re-clarify review priorities, output formats, and severity classifications. How is this using a smart assistant? It's clearly like repeatedly conducting onboarding training day after day for a temp worker with "amnesia." Xiao Xue felt the exact same way. When collaborating with the company's data analysis team, every time she asked Claude to conduct a financial analysis, she had to re-explain the gross margin formula, industry benchmark data, and report template formats. This core knowledge remains constant, but Claude acts as if hearing it for the first time every single time. "Is there a way," she asked Brother Ka, "to make Claude permanently remember the work paradigm of certain specific domains? Not just project-level general memory like `CLAUDE.md`, but more precise context awareness, for example, whenever I mention 'API documentation,' it automatically knows the execution standards; whenever I mention 'financial analysis,' it can automatically load the corresponding industry benchmarks and report templates."

Brother Ka nodded slightly, turned around, and wrote a word on the whiteboard—Skills.

#### 3.1 From CLAUDE.md to Skills: Two Dimensions of Knowledge

"Before diving into Skills," Brother Ka began, "I want to first clarify a fundamental question—why can't `CLAUDE.md` meet the needs anymore?" Having said that, he drew a coordinate system of knowledge types and loading strategies on the whiteboard (see Figure 3-1). ![[Pasted image 20260701092447.png]] _(Figure 3-1 CLAUDE.md and Skills knowledge types and loading strategies)_ "`CLAUDE.md` is like an enterprise's 'rules and regulations'," Brother Ka explained. "It defines the basic rules of the project—whether to use TypeScript or Python, whether to indent with Tabs or spaces, and what format PR titles should follow. These rules must take effect at every moment of every conversation, so they must be permanently resident in the context, fully loaded every time. But the cost of this is obvious: 'permanently resident' means that whether you are currently processing related tasks or not, these Tokens will be continuously consumed." He paused, and turned his gaze to Xiao Bing: "Now think about your API documentation knowledge—the OpenAPI 3.0 specification, error code standards, and bilingual formats. This information is only needed when you generate API documentation. If you forcefully stuff them into `CLAUDE.md`, then when you debug a bug, run tests, or write configuration files, these thousands of Tokens of document specifications will still linger in the context, vainly occupying precious space and diluting the weight of truly critical information."

"So, this is the core pain point Skills aims to solve?" Xiao Bing asked thoughtfully. "Accurately speaking, Skills solves the problem of delivering knowledge on demand," Brother Ka said, drawing a complete comparison table on the whiteboard (see Table 3-1). **Table 3-1 Feature Comparison between CLAUDE.md and Skills**

|System File|Hosted Content|Effective Scope|Trigger Method|Loading Strategy|Typical Use|Relationship with Agent|Token Consumption|Enterprise Ontology|
|---|---|---|---|---|---|---|---|---|
|CLAUDE.md|Project general specifications|Current project|Always effective|Full load every time|Using TypeScript|Shared by all Agents|Fixed overhead|Enterprise rules and regulations|
|Skills|Professional workflows, domain knowledge|Cross-project, cross-session|Activated on demand|Progressive on-demand loading|How to conduct code review|Can be bound to specific Agents|Pay-as-you-go based on usage|SOP manual|

"Enterprise Ontology" in the last column of Table 3-1 caught Xiao Xue's attention. "What does this mean?" she asked. Brother Ka explained: "A mature enterprise actually has two vastly different types of knowledge assets. The first category consists of general rules that everyone must obey (such as attendance policies, security red lines, and communication norms). This content is like posters pasted on the office walls, visible to everyone, effective at all times. This corresponds to `CLAUDE.md`. The second category is the Standard Operating Procedure (SOP) for specific roles, such as how finance does monthly closing, how operations handles failures, and how customer service responds to complaints. This content will not be posted on the wall for everyone to see, but neatly stored in the filing cabinets of the respective departments, and only referenced when people in specific roles need to execute specific tasks. This is Skills. `CLAUDE.md` is that wall, and Skills is that filing cabinet. You don't need to carry the financial manual on your back while writing code, nor do you need to bring safety regulations while making reports; only at the very moment of 'need' will the corresponding knowledge be precisely extracted (see Figure 3-2)."

This analogy precisely anchors the core positioning of Skills in the Claude Code architecture. Claude Code's definition of Skills is both straightforward and profound: **A Skill is a folder containing instructions, packaged into a simple directory structure, used to "teach" Claude how to handle specific tasks or workflows.** In this definition, two keywords are worth deep consideration. ![[Pasted image 20260701092635.png]] _(Figure 3-2 Skills can be fetched on demand)_ The first is "folder"—from "string" to "engineering." A Skill is by no means just a piece of Prompt or a simple configuration item; it is a complete engineering directory—capable of holding complex codebases, detailed domain documents, various templates, and even executable scripts. The second is "teach"—from "constraint" to "internalization." The definition uses "teach" rather than "command" or "constrain." This subtly reveals the essence of its operational mechanism. "Teaching" is an endowment and internalization of capabilities. Through Skills, Claude truly understands the operational logic of a domain. It is no longer a tool that passively executes instructions, but becomes an "expert" in that domain.

#### 3.2 Dissecting a Skill: Skeleton and Texture

"In 'Zhuangzi: The Secret of Caring for Life,' the reason why Cook Ding could butcher an ox with such ease was that he had long seen through the ox's skeletal and muscular structure, following the natural grain, striking the large gaps, and guiding the knife through the large hollows, rather than relying on brute force to hack at it," Brother Ka quoted. "Understanding the structure of a Skill requires the same wisdom: first, clearly see its 'skeleton,' understand how the knowledge is placed and organized, and only then can we talk about how to use it exquisitely."

##### 3.2.1 Directory Structure

Brother Ka turned around and listed a typical Skill directory structure next to the coordinate system on the whiteboard. "This is the specific structure of that 'ox' in 'Cook Ding butchering the ox'." Brother Ka pointed to the directory structure just listed on the whiteboard and deconstructed it one by one. "These seemingly tedious 'rules and regulations' are actually error-prevention mechanisms exchanged for blood and tears in engineering practices." Brother Ka picked up a red pen, heavily circled a few key points next to the directory structure on the whiteboard, and his expression turned serious.

**File name: `SKILL.md`—the only "trigger".** The file name must be fully capitalized; `skill.md`, `Skill.md`, and `SKILL.MD` are all invalid. When Claude's loader scans files, it executes exact string matching rather than fuzzy searches.

**Directory name: kebab-case—the cross-platform "lingua franca".** The directory name adopts the format of all lowercase + hyphens (e.g., `api-doc-generator`), up to 64 characters, with a character set limitation of `[a-z0-9-]`. There cannot be spaces, underscores, uppercase letters, leading/trailing hyphens, or consecutive hyphens. This is to smooth out operating system differences. Windows OS does not distinguish between uppercase and lowercase, while Linux OS strictly does. Kebab-case is the "Esperanto" of the internet era, ensuring your Skills can be recognized losslessly in any environment and any toolchain.

**No `README.md` allowed—a pure "instruction space".** Placing `README.md` within the Skills directory is strictly prohibited. Documents intended for humans to read should be placed in the parent directory. When Claude activates a Skill, it greedily reads the Markdown files under the directory as context. `SKILL.md` contains instructions meant for Claude. While `README.md` typically contains introductions meant for humans. If `README.md` is mixed in, it will interfere with Claude's understanding of the instruction content and introduce unnecessary noise.

**Naming neutrality: reject "claude" or "anthropic".** The `name` field must not contain brand words. Skills is a universal knowledge packaging format; its vitality lies in openness. Maintaining brand neutrality is meant to let knowledge itself flow, rather than becoming an accessory to a certain product.

##### 3.2.2 YAML Frontmatter: The "ID Card" of a Skill

The YAML Frontmatter at the top of `SKILL.md` defines the identity and behavioral boundaries of the Skill. The complete field descriptions are as follows. These fields can be divided into three major logical groups, corresponding respectively to the 3 core dimensions of Skills design: trigger mechanism, permission control, and runtime environment.

- **Identity fields (trigger mechanism):** Contains `name`, `description`, and `argument-hint`. They define "what the Skill is," are responsible for clearly conveying the functional positioning and calling method of the Skill to Claude and the user, and form the foundation of the trigger logic.
- **Permission fields (permission control):** Contains `disable-model-invocation`, `user-invocable`, `allowed-tools`, and `model`. They stipulate "who can call" and "what can be done," building strict security and resource boundaries by restricting call sources, available tools, and specified models.
- **Execution fields (runtime environment):** Contains `context`, `agent`, and `hooks`. They determine "where it executes" and "what happens during execution," used to configure isolated environments, sub-agent types, and lifecycle event hooks to ensure tasks run in the expected context.

Mastering these 3 dimensions equates to mastering the complete framework of Skills architecture design. **Brother Ka's Remarks** `disable-model-invocation` and `user-invocable` are easily confused, but they actually control two orthogonal calling directions. **The meaning of `disable-model-invocation: true` is "Claude cannot automatically trigger this Skill; it must be manually executed by the user via `/skill-name`." It applies to operations with side effects or high risks, such as code commits (`/commit`), service deployments (`/deploy`), etc., to ensure explicit confirmation by the user.** The default value of `disable-model-invocation` is `false`. The meaning of `user-invocable: false` is "This Skill will not appear in the `/` menu, and the user cannot manually trigger it, but Claude can still automatically call it within its internal logic." It is suitable for building purely knowledge-based or auxiliary reference Skills; hiding them avoids menu clutter while preserving Claude's ability to automatically load and use them at the right time. The default value of `user-invocable` is `true`. Under default configurations (i.e., Claude's automatic calls are all in an "allowed" state), Skills are open bidirectionally to both Claude and the user.

#### 3.3 Progressive Disclosure: The Return on Investment of Knowledge

"Alright, I understand the structure of Skills," Xiao Bing asked, "but if I have 20 Skills, each containing thousands of Tokens of content, does Claude Code have to fully load them all in every conversation? Then what is the difference between that and stuffing everything into `CLAUDE.md`?"

This is a pertinent question that directly addresses the point, and it exactly reveals the most ingenious design core of the Skills system—Progressive Disclosure.

##### 3.3.1 The Library Model

Brother Ka answered: "We can use a library as an analogy here. Imagine you walk into a library to find information. You would never read all the collections at once, but rather follow an efficient retrieval process—first browse the library's catalog to locate the category, then extract the target book to consult the book's table of contents, and finally carefully read only the required chapters. The Skills system adopts exactly a similar three-layer progressive disclosure model (see Figure 3-3), realizing on-demand calling of massive knowledge at extremely low Token costs." ![[Pasted image 20260701093146.png]] _(Figure 3-3 Three-layer progressive disclosure model)_ The performance improvement brought by this model in practical application is significant. Taking a complete financial analysis Skill (totaling 5300 Tokens) as an example, its file composition is shown in Table 3-2. **Table 3-2 File Composition of a Complete Financial Analysis Skill**

|File Path|Token Count|Description|
|---|---|---|
|SKILL.md|800|Main logic and routing guide|
|reference/revenue.md|1500|Revenue data reference|
|reference/costs.md|1200|Cost structure reference|
|reference/profitability.md|1000|Profitability formula reference|
|templates/report.md|800|Report template|

The comparison of Token consumption across different loading methods is shown in Table 3-3. **Table 3-3 Comparison of Token Consumption Across Different Loading Methods**

|Scenario|Traditional Full Loading|Progressive Disclosure|Savings Percentage|
|---|---|---|---|
|Skill not activated|0|0|-|
|Scan phase (relevance judgment only)|5300|~100|98%|
|Simple request (only main file needed)|5300|800|85%|
|Medium request (one reference file needed)|5300|2300|57%|
|Complex request (all resources needed)|5300|5300|0%|

Data indicates that the vast majority of requests only require partial resources. When a user asks "how to calculate gross margin," Claude Code only needs to load `SKILL.md` (locate routing) and `profitability.md` (fetch formula), totaling about 1800 Tokens. The other 3 files (totaling 3500 Tokens) remain entirely on disk, consuming zero. This is the core economic advantage of "on-demand knowledge delivery": exchanging the minimum context cost for the maximum knowledge coverage.

##### 3.3.2 The Budget Mechanism of Description

The first level of progressive disclosure, namely the `description` of all Skills, is injected into Claude's context in a resident manner. This means they must collectively divide a limited Token budget, constituting the system's "entry bottleneck." Claude Code official rules: the total budget limit for `description` is 2% of the total context window size; if not specified or a calculation exception occurs, it defaults to a fixed 16,000 characters.

This budget is equally divided among all installed Skills, rather than being allocated on demand. The calculation formula is as follows. Available characters per Skill = Total budget / Total number of Skills Assuming you have installed 20 Skills, under the default budget of 16,000 characters, each `description` can only be allocated an average of 800 characters (see Figure 3-4). ![[Pasted image 20260701093242.png]] _(Figure 3-4 Claude Code's description budget pool)_ If the `description` of a certain Skill exceeds 800 characters, it will be silently excluded. Claude Code will be completely unaware of this Skill's existence; it will neither see it during the scanning phase nor be able to load it in subsequent steps. To the user, this Skill seems to have "disappeared."

**Brother Ka's Remarks** Regarding the limitation of the `description` budget, here are 3 key practical tips that can help you avoid pitfalls and flexibly control the system.

- Tip 1, **Set `disable-model-invocation: true` in the Skill configuration. This Skill's `description` will not be injected into the context.**
- Tip 2, Run the diagnostic command `/context` to check whether any Skills exceeding the budget have been "silently excluded."
- Tip 3, By adjusting the environment variable `SLASH_COMMAND_TOOL_CHAR_BUDGET`, you can manually expand the budget pool.

This budget mechanism reveals a core engineering philosophy: **Do not blindly create Skills, but rather pursue a "fewer but better" architectural design.** When you find the need to install more than 20 Skills, this is usually a signal prompting you to re-examine your architectural strategy.

- Merge similar items. Consolidate multiple fragmented Skills into one comprehensive Skill.
- Hide internal tools. Mark purely internally-called sub-tasks with `disable-model-invocation: true` to reduce budget consumption.
- Adopt a "Token investment" return-on-investment mindset. Treat every character in the `description` as an expensive "Token investment"—precise investment yields efficient retrieval and accurate execution; wasteful investment not only wastes your own budget quota but also squeezes the survival space of other Skills.

#### 3.4 Trigger Mechanism: How Claude Code Chooses to Call Skills

The trigger mechanism of Skills is the most crucial and most needs-to-be-deeply-understood core link in the entire system. It directly determines whether a Skill can be activated at the right time—this point is even more important than the quality of the Skill's content itself. After all, if it cannot be successfully triggered, even the most brilliant content will be rendered useless.

##### 3.4.1 Dual-Channel Activation Mechanism

Each Skill supports two activation paths. **1. Explicit Call** When a user enters `/skill-name` in the conversation, Claude Code immediately loads and executes the corresponding Skill. This method is similar to terminal commands, characterized by being clear, direct, and unambiguous. If the Skill defines `argument-hint`, the user can also call it with parameters, such as `/commit fix login bug` or `/review src/auth/login.ts`.

**2. Semantic Matching** After deeply understanding the user's intent, Claude autonomously judges which Skill best fits the current task and thus automatically loads it. In daily applications, this is exactly where the core value of Skills lies—users only need to describe their needs, and Claude will intelligently decide behind the scenes whether to call and how to call the most appropriate Skill. The entire process is completely transparent to the user, achieving a seamless "what you think is what you get" experience. Figure 3-5 shows the execution flow of explicit calling and semantic matching. ![[Pasted image 20260701093520.png]] _(Figure 3-5 Execution flow of explicit calling (left) and semantic matching (right))_

##### 3.4.2 Description—The Soul of Skills

The semantic matching mechanism completely relies on the `description` field. This field is not descriptive text meant for humans to read, but rather the **sole signal** that Claude relies on when deciding "whether to call this Skill." Claude Code's recommended structure formula for writing a `description` is as follows. [Function Definition] (What it does) + [Trigger Scenario] (When to use) + [Core Capability] (What it can do) The best way to understand the above formula is by comparing "inefficient" and "efficient" ways of writing. Below is a comparison of the two ways of writing. The `description` field has a limit of **1024 characters**. This space is not generous, so it requires weighing every word and carefully selecting terminology.

An efficient and practical writing strategy can be divided into the following 3 steps:

- **Step 1, Define core capabilities (What):** Accurately summarize "what this Skill can do" in one sentence, establishing its basic functional positioning.
- **Step 2, Clarify trigger scenarios (When):** Use the `Use when user...` sentence pattern to detail various user instructions, phrases, or keywords that might trigger the Skill, improving the hit rate of semantic matching.
- **Step 3, Define exclusion scopes (Not For):** (Optional but recommended) If the Skill is prone to being mistakenly triggered, be sure to add `Not for...` to explicitly point out the scenarios where it is not applicable, in order to optimize decision accuracy.

**Brother Ka's Remarks** The core audience of the `description` you write is Claude, not human readers. When humans read documents, they tend to scan titles, browse structures, and quickly grasp the main idea; whereas when Claude reads the `description` field, it is performing deep semantic matching. It needs to capture subtle differences in user intent. In order to improve trigger accuracy, you must exhaust various expressions users might use within the `description`. For example, regarding the need to "generate API documentation," users might say "generate API docs," "write interface docs," "output OpenAPI specifications," or "help me write a Swagger." All these different expressions should be explicitly included in the `description`. The richer the synonym library, the broader the coverage of semantic matching, and the higher the trigger accuracy will be.

##### 3.4.3 Preventing Over-Triggering and Under-Triggering

In practice, the trigger mechanism of Skills often faces two typical failure modes. Understanding and fixing these problems is the key to optimizing Claude's performance.

**1. Under-Triggering** Phenomenon: The Skill should have been called, but was not. Data warning: Vercel's benchmark data shows that without explicit guidance, Agents have a 56% probability of completely failing to check available Skills. Root cause: The `description` is written in an overly technical or academic manner, resulting in a huge semantic gap between it and the user's natural conversational expressions. Fix strategy: "Translate" the user's language. Add a large number of expressions commonly used by users into the `description`, covering synonyms of domain terminology, colloquialisms, and even common but inaccurate expressions (for example, many users confuse "Swagger" and "OpenAPI," so both need to be written in).

**2. Over-Triggering** Phenomenon: The Skill is mistakenly activated in scenarios where it shouldn't be called. Root cause: The `description` is defined too broadly, containing too many high-frequency generic words, leading to a match threshold that is too low. Fix strategy: Introduce reverse constraints. Clearly demarcate boundaries and use the `Not for...` sentence pattern to exclude distractors. Example: "Not for general code questions or debugging. Only for structured documentation generation."

To test whether a Skill's triggering is accurate, you can build a validation set containing 10-20 test cases, simultaneously covering both "should trigger" and "should not trigger" scenarios. Below is an example of a practical test template. The acceptance criteria are: for related tasks, the trigger rate should reach above 90%; for unrelated tasks, the false trigger rate should be kept below 5%.

##### 3.4.4 Reference Skills vs. Task Skills: Two Skill Philosophies

In the design philosophy of Skills, the `disable-model-invocation` field is not just a simple switch; it defines two entirely different Skill interaction modes—Reference and Task.

**1. Reference Skills** Configuration: Default behavior. Core logic: "On-demand loaded knowledge base." Claude will automatically judge whether this type of Skill is needed based on the conversation context. `description` is the trigger condition, injected directly into Claude's context for semantic matching. Use cases: Providing knowledge, specifications, frameworks, or standards. The user does not need to be aware of the Skill's existence; Claude will automatically "flip open the manual" at the appropriate time. Examples: API design specifications, code review checklists, industry standard documents.

**2. Task Skills** Configuration: Explicitly set `disable-model-invocation: true`. Core logic: "Controlled execution tools." Claude cannot trigger them automatically; they must be manually called by the user via the `/skill-name` command. `description` is not injected into Claude's context, serving only as identification instructions for users when selecting Skills in the command line.

Use cases: Operations with side effects, requiring explicit authorization from the user to execute. Examples: `/commit` (commit code), `/deploy` (deploy application), `/notify` (send notifications). A comparison between Reference Skills and Task Skills is shown in Table 3-4. **Table 3-4 Comparison Between Reference Skills and Task Skills**

|Dimension|Reference Skill|Task Skill|
|---|---|---|
|Claude Auto-Trigger|Can (based on semantic matching)|Cannot (must be manual)|
|User /Trigger|Can|Can|
|Role of description|Trigger condition (injected into Claude's context)|For user identification only (not injected into Claude's context)|
|Consumes Token budget|Yes (consumed upon auto-loading)|No (consumed only when manually called)|
|Typical scenario|Providing knowledge, querying specifications|Executing actions, operations with side effects|
|Classic examples|API specs, review checklists|`/commit`, `/deploy`|

**Brother Ka's Remarks** The Side Effect mentioned here refers to an operation that not only returns a result but also changes external system states. For example, committing code alters the Git repository history, deploying an application changes the state of live services, and sending notifications genuinely messages others. Once these actions are executed, they impact the real world, and are typically irreversible. Therefore, this kind of Skill is usually designed to be explicitly triggered by the user, rather than letting the model decide on its own when to execute.

Xiao Bing asked: "Should a task be designed as a Reference Skill or a Task Skill?"

Brother Ka answered: "Please use a simple 'worst-case scenario test' to decide: 'If Claude automatically executes this task, what is the worst-case scenario?' If the answer makes you nervous—for instance, automatically committing untested code, automatically deploying a buggy version, automatically deleting production data, or automatically sending incorrect notifications—you must choose a Task Skill. Please add `disable-model-invocation: true`. Keep the control firmly in the hands of the user, only when the user explicitly inputs `/`" ![[Pasted image 20260701093859.png]] _(Figure 3-6 The role of SKILL.md is a router)_ The core technique of routing design lies in building a "Quick Reference" table. This table can clearly guide Claude through routing conditions across 5 key dimensions with extremely low Tokens (about 3 lines requiring only 50 Tokens). Compared to having Claude scan the entire `SKILL.md` line by line to infer that "when encountering revenue issues, look up `revenue.md`," the efficiency of this structured table is significantly higher, with an information density that can be up to 10 times that of the former.

##### 3.5.2 Contractual Reference

When referencing auxiliary files within `SKILL.md`, do not simply list the paths. A clear "contract" should be established to ensure Claude clearly understands 3 core elements: trigger timing (when to load), resource location (where to look), and expected output (what to fetch).

The following examples demonstrate the writing methods of "weak reference" and "contractual reference." This design philosophy is consistent with the "handover contract" in sub-agent pipelines: downstream consumers not only need to know the upstream's location but must also clearly understand what the upstream can provide.

##### 3.5.3 The 500-Line Rule

Claude Code recommends keeping the length of `SKILL.md` within 500 lines. "Why is it set to 500 lines?" Xiao Bing asked. Brother Ka answered: "This is because 500 lines of code equals 2000-3000 Tokens, which is a reasonable context overhead after a single Skill is activated. Accumulating this with Claude's System Prompt and the current conversation history can ensure the total Token count remains within a controllable range. If it exceeds 500 lines, it usually means you have confused 'reference materials' with 'routing instructions'—the response strategy at this time is not to continue expanding the content, but to immediately refactor." Table 3-5 lists the refactoring signals and countermeasures when Skills content is overloaded. **Table 3-5 Refactoring Signals and Countermeasures When Skills Content is Overloaded**

|Refactoring Signal|Countermeasure|
|---|---|
|Large blocks of formulas or specification descriptions|Move to the `reference/` directory|
|Multiple complete examples (single one exceeding 30 lines)|Move to the `examples/` directory|
|Multiple output templates|Move to the `templates/` directory|
|Independently executable logic|Encapsulate as scripts in `scripts/`|
|Multiple parallel functional modules|Consider splitting into multiple independent Skills|

#### 3.6 allowed-tools: Knowledge Constrains Actions

`allowed-tools` is the core field in the security architecture of Skills. It is not just a permission checklist; it further reflects a deep design principle: **Knowledge should constrain actions**. Specific permission configurations should be based on the "cognition" of the Skills' business logic. Code review Skills: The review process only needs to read code and strictly prohibits modification, so only read-only tools are granted. Documentation generation Skills: Needs to create new files, but should never tamper with existing files, therefore only Write permission is granted, while Edit permission is explicitly forbidden. Test runner Skills: Only needs to execute specific test commands, thus precisely limiting executable Bash commands to prevent arbitrary command injection.

##### 3.6.1 Permission Design Templates

Below are the best practices for `allowed-tools` configurations tailored to different Skill types. These configurations reflect the Principle of Least Privilege, ensuring each Skill only possesses the absolute minimum tool permissions required to complete its specific task.

##### 3.6.2 Fine-Grained Control Syntax for Bash

The Bash tool supports fine-grained management of executable commands through a prefix matching mechanism. Its core syntax is `Bash(prefix:*)`, where `prefix` specifies the allowed command prefix, and `*` acts as a wildcard representing subsequent parameters. Examples of permission control are as follows. When Claude attempts to execute a Bash command, the system conducts preemptive verification.

- Extract prefix: Retrieve the complete command string requested for execution by the user.
- Match rule: Check whether the command starts with the configured `prefix`.

Execution decision: If the match succeeds, allow execution; if the match fails, reject it directly and return a permission error. Following the Principle of Least Privilege is the cornerstone of building secure Skills. Below is a comparison of positive and negative cases regarding `allowed-tools` configurations. `Bash ( * )` is practically equivalent in security to not setting `allowed-tools` at all. This means Claude Code gains full Shell privileges on the host machine. For high-risk Task Skills like `/deploy`, this configuration is fatal. Once the model is hijacked by a prompt injection attack, the attacker can exploit this permission to steal keys, delete production data, or plant backdoors.

#### 3.7 Parameter Passing and Dynamic Injection

Skills are not just static instructions; they also support runtime parameter passing and context pre-injection, enabling their behavior to dynamically adjust based on the calling scenario.

##### 3.7.1 `$ARGUMENTS` and Positional Parameters

When a user calls a Skill via `/skill-name arg1 arg2`, the variables shown in Table 3-6 can be referenced in the main body of `SKILL.md`. **Table 3-6 Variables Referenceable in the Body of SKILL.md and Their Descriptions**

|Variable|Description|
|---|---|
|`$ARGUMENTS`|Complete string of all parameters|
|`$ARGUMENTS`|First parameter (0-indexed)|
|`$ARGUMENTS`|Second parameter|
|`$0, $1, $2`|Shorthand forms for positional parameters|

A reference example is as follows. When executing the call `/migrate-component SearchBar React Vue`, the instruction actually received by Claude is "Migrate the SearchBar component from React to Vue. Preserve all existing behavior and tests."

##### 3.7.2 Dynamic Context Injection

**This is the most powerful and unique feature in the Skills system. The `!command` syntax allows for executing a specified command in the Shell environment before sending `SKILL.md` to Claude, and inlining the output of the command directly into the Prompt.** An example is as follows. When executing the `/pr-create "Add auth"` command, what Claude actually receives is a Prompt already populated with dynamic data. The engineering value of this design is immense. The comparison between not using and using the `!command` feature is shown in Table 3-7. **Table 3-7 Comparison Between Not Using and Using the `!command` Feature**

|Dimension|Unused `!command`|Used `!command`|
|---|---|---|
|Context upon Claude startup|Blank, needs multiple turns of dialogue to explore|Key information pre-injected|
|Tool calls in first response|3-5 times (for gathering info)|0-1 times (directly executes actions)|
|Token consumption|High|Low|
|Response speed|Slow|Fast|
|Result consistency|Low (risk of missing information)|High (fixed injection of the same info)|

#### 3.8 Scope and Priority

_Brother Ka's Remarks_ Strictly demarcate execution boundaries. Claude follows a strict sequence when executing `!command`: first replaces the `$ARGUMENTS` variable, then executes the Shell command. This means the user's input content will be directly concatenated into the Shell command, making it highly vulnerable to Shell injection attacks. Therefore, any Skills using the `!command` syntax must strictly limit their executable command range via the `allowed-tools` configuration item to build necessary security guardrails.

Skills files can be deployed at different tiers, each tier corresponding to a specific effective scope and applicable scenario, as detailed in Table 3-8. **Table 3-8 Effective Scope and Applicable Scenarios of Skills**

|Storage Location|Effective Scope|Typical Uses|
|---|---|---|
|Enterprise Configuration Center|Effective for all employees|Enterprise-level development standards and security policies for mandatory execution|
|`~/.claude/skills/`|Personal to all projects|Personal coding habits, universal toolsets, and cross-project utility scripts|
|`.claude/skills/`|Current project only|Project-specific workflows, tailored business logic, and team collaboration norms|
|Plugin Built-in Resources|When Plugin is enabled|Community-shared capability packages, dedicated command sets for specific frameworks|

When Skills of the same name exist from multiple sources, **the system will parse and load them in order of priority from highest to lowest**. Enterprise Policy > Personal Configuration (`~/.claude/`) > Project Configuration (`.claude/`) > Plugin Built-in

The design logic behind this priority order strictly follows enterprise governance architecture, ensuring an orderly transition from "mandatory baselines" to "personalized preferences."

- **Enterprise Policy (Insurmountable Baseline):** Enterprise-level configurations possess the highest authority and are used to enforce global security and compliance policies. For example, a security audit Skill must mandatorily check for OWASP Top 10 vulnerabilities; such critical rules are never allowed to be overridden by project-level or personal configurations, thereby ensuring that security red lines at the organizational level are not breached.
- **Personal Configuration (Personalized Efficiency Tools):** Configurations located in the personal directory aim to satisfy developers' personal habits. For example, a user can define a `/commit` command to automatically generate commit messages according to the conventional commit standard; this preference only operates in the personal environment and does not affect others.
- **Project Configuration (Business Scenario Tailoring):** Configurations located in the project's root directory serve specific codebases exclusively. For example, a certain frontend project can tailor an exclusive component documentation generation Skill to adapt to its unique tech stack and documentation structure.

Incorporating the Skill directory (`.claude/skills/`) into the project's Git version control is the optimal solution for realizing zero-cost sharing of team knowledge.

- **Plug and Play:** After new team members clone the codebase, the relevant Skills take effect automatically without any manual installation or configuration steps.
- **Synchronized Evolution:** As the project iterates, the Skill repository can be updated alongside the code, ensuring the team always uses the latest workflow standards.

#### 3.9 Practice: Building 3 Types of Skills from Scratch

The theoretical explanation is fully established up to this point; next, we will enter the core engineering practical stage. In this section, we will walk you through building high-value Skills from scratch via 3 complete cases of increasing complexity. These 3 cases were not chosen at random, but rather precisely correspond to the 3 typical application scenarios described earlier.

##### 3.9.1 Reference Skill: Code Review

Xiao Bing's team needs to perform massive code reviews daily. To this end, the team has established the following internal agreements. Principle of priority: Prioritize security issues, followed by performance issues, and finally code style. Feedback requirements: Must provide specific modification suggestions; merely pointing out problems without offering solutions is strictly prohibited. Tiered labeling: Every problem must be labeled with a severity level. Given that relevant standards are scattered across different documents, making it difficult for newcomers to master them all at once, the core guidelines and directory structure are consolidated below. The directory structure is as follows. The core content of `SKILL.md` is as follows. The design of this Skill focuses on three core points.

- **Read-only Permission Guarantee:** Only Read, Grep, and Glob tools are granted, structurally eliminating the risk of accidental code modifications during the review process.
- **Priority Sorting Strategy:** Strictly follows the review order of "Security > Performance > Quality" to ensure critical hazards are discovered first, preventing omissions.
- **Structured Output Norms:** Uniformly adopts the output format of "Severity Level + Problem Description + File Location + Modification Suggestion" to guarantee consistency and traceability of every review result.

##### 3.9.2 Task Skill: Smart Commit

Xiao Bing needs to frequently commit code every day (averaging over ten times a day), and manually drafting commit messages is both time-consuming and prone to non-compliance. To this end, we designed the following Task Skill. Given that this operation has side effects (directly modifying Git history), the system is set up to require active triggering by the user, and auto-execution is strictly prohibited. The above Skill demonstrates a deep combination of multiple advanced features.

- **Security Control (`disable-model-invocation: true`):** Forcibly disables model invocation to ensure the execution process relies purely on preset scripts, preventing accidental AI reasoning intervention.
- **Dynamic Parameters (`$ARGUMENTS`):** Supports a flexible parameter passing mechanism, allowing users to directly specify commit information or leave it blank to trigger automatic generation.
- **Context Pre-Injection (`!command`):** Utilizes Shell commands to instantly capture and inject the current Git state prior to execution. This gives Claude complete context information upon startup, allowing it to perceive content changes without invoking additional tools, significantly improving response speed.
- **Cost and Performance Optimization (`model: haiku`):** Specifies the use of a lightweight model (Haiku). Given that the commit operation mainly relies on rule matching rather than complex reasoning, this configuration effectively reduces latency and resource consumption while ensuring accuracy.

A usage example is as follows.

##### 3.9.3 Composite Skill: Financial Analysis (A Complete Case of Progressive Disclosure)

This case demonstrates a highly modular progressive disclosure architecture. This design achieves efficient management of complex tasks through layered decoupling.

- Control Layer: The main file is responsible for intent recognition and process routing.
- Knowledge Layer: Reference files provide deep knowledge and benchmarks for vertical domains.
- Normative Layer: Templates ensure high consistency of output formats.
- Execution Layer: Scripts encapsulate deterministic calculation logic, eliminating the risk of hallucinations. The directory structure is as follows. The core content of `SKILL.md` is as follows. The above `SKILL.md` is only about 200 lines, fully conforming to the "500-line rule." The "Quick Reference" table inside acts as a router: when Claude identifies "gross margin," it can directly locate `reference/profitability.md` to get the formula, without needing to load redundant files related to revenue and costs. In addition, the `calculate_ratios.py` script encapsulates deterministic calculation logic, meaning Claude doesn't have to execute error-prone floating-point math, but simply needs to call the script to fetch precise results. This is a complete manifestation of the "Progressive Disclosure" concept: **Achieving optimized task completion quality with minimum Token input.**

#### 3.10 Four Design Patterns for Skills

From engineering practices, we extracted 4 verified Skill design patterns (see Figure 3-7). These patterns are not mutually exclusive; mature Skills systems typically use a combination of multiple patterns. ![[Pasted image 20260701094610.png]] _(Figure 3-7 Decision tree for combinations of Skill design patterns)_

- **Template-Driven Pattern:** Utilizes predefined templates to strictly constrain output formats. This pattern applies to scenarios that require standardized outputs, such as weekly reports, incident reports, and review reports. Under this pattern, Claude's output will strictly follow the template structure, ensuring the results possess predictability and comparability, while supporting automated post-processing.
- **Script-Enhanced Pattern:** Encapsulates deterministic computational logic into scripts, called and executed by Claude instead of inferring it on its own. This pattern is suitable for financial calculations, regex matching, data transformations, and similar scenarios. Compared to large model reasoning, script execution is more precise, saves more Tokens, and is more reproducible. A rule of thumb is: if you find yourself writing formulas in `SKILL.md` to make Claude run calculations, please stop immediately—that logic should be moved to a script.
- **Knowledge Layering Pattern:** Organizes knowledge hierarchically based on usage frequency. Following the "80/20 Rule" (meaning 80% of requests only need 20% of the core content), high-frequency knowledge is inlined into `SKILL.md`, while low-frequency knowledge is placed in reference files for on-demand loading. This is exactly the formalized summary of the "Progressive Disclosure" concept.
- **Tool Isolation Pattern:** Strictly defines the capability boundaries of a Skill via the `allowed-tools` mechanism. This falls under the scope of security design rather than purely functional design; its core value lies in clarifying "what is forbidden to do," which is often far more critical than defining "what can be done." For example, audit Skills are not granted Write permissions, and generation Skills are not granted Modification permissions.

#### 3.11 Testing and Iteration

Xiao Bing asked, "Are we done once the Skill is written?" Brother Ka answered, "Of course not. A Skill is a living document that needs continuous polishing." Claude Code's official website recommends 3 core categories of testing methods to ensure the robustness of a Skill.

- **Trigger Test:** Prepare 10 questions that should trigger the Skill and 10 questions that should not, to verify the accuracy of Claude's judgments. The goal is that the trigger rate for related tasks needs to be higher than 90%, while the false trigger rate for unrelated tasks should be lower than 5%.
- **Functional Test:** Verify the execution quality after the Skill is loaded. Checkpoints include whether the output format meets expectations, whether checking items are completely covered, and whether boundary cases are properly handled.
- **Performance Comparison:** Aiming at the same task, execute it 5 times respectively in "With Skill" and "Without Skill" states, comparing Token consumption, user correction frequency, and final output quality.

If you find yourself repeatedly manually correcting Claude's output (for instance, having to remind it every time to "remember to mark authentication requirements"), this is a clear signal that the body of `SKILL.md` needs to be updated. Write the correction logic directly into `SKILL.md`, and the same kind of error won't happen next time. This iterative process is identical to the bug fix cycle in traditional software: discover issue → locate root cause (imprecise `description`? missing steps? vague format definitions?) → fix document → verify effects. Claude Code also provides a dedicated Skill called `skill-creator`, which can assist you in generating the initial version of `SKILL.md` from natural language descriptions. A command example is as follows.

The very existence of this tool reveals an important fact: the Skills system is mature enough to achieve "bootstrapping," meaning leveraging the Skills system itself to build new Skills.

#### 3.12 Viewing Skills Through Software Engineering

Just like sub-agents, Skills is also a completely new concept. Its core essence is migrating and adapting the time-tested classical wisdom of software engineering into the knowledge architecture of AI Agents.

**1. Separation of Concerns** Core philosophy: "Give a man a fish and you feed him for a day; teach a man to fish and you feed him for a lifetime." Skills precipitate problem-solving methods, steps, and experiences into reusable knowledge assets, rather than just providing one-time answers. This enables Agents to shift from relying on "temporary inspiration" to being able to stably reproduce high-quality workflows. The three-layer architecture responsibility division is as follows.

- _`CLAUDE.md`:_ Global rules (project background, general standards).
- _Skills:_ Professional workflows (encapsulation of complex logic for specific domains).
- _Sub-Agents:_ Task execution (dynamic planning and real-time operation). Engineer Warning: Forcefully stuffing all logic into `CLAUDE.md` is strictly prohibited. Just like writing all code into the `main` function in software development, this causes context redundancy, maintenance difficulties, and poor extensibility.

**2. Dependency Inversion** Core mechanism: Programming oriented to interfaces, not implementations. Claude does not directly rely on the specific internal implementations of a Skill (like specific scripts or Prompt details), but relies on its abstract interface (i.e., `description` and output contract). As long as the interface contract remains unchanged, developers can replace, refactor, or upgrade the internal logic of a Skill at any time. To Claude and users, this change is completely transparent, greatly reducing the system's coupling degree.

**3. Cache Optimization and Lazy Loading** Core strategy: Progressive Disclosure = On-Demand Loading. "Progressive disclosure" is a typical Lazy Loading strategy. The system does not preload all knowledge bases upon startup, but loads the relevant resources only when a specific skill is needed for the first time. This shares the same logic with Code Splitting in Web development and delayed loading in databases: loading the right resources at the right time, thereby minimizing initial overhead and improving response speed.

**4. Principle of Least Privilege** Security cornerstone: `allowed-tools` is a direct mapping of security classics in the AI field. Granting a Skill just enough permissions to fulfill its responsibilities, no more, no less. It is akin to using `chmod` in the Linux OS to precisely control file read, write, and execute permissions, or using IAM Policy in AWS to finely restrict service access scopes. Clarifying "what cannot be done" is more capable of ensuring system security than defining "what can be done," preventing unauthorized operations caused by malicious code or hallucinations.

**5. Open Standards** Ecosystem vision: Declarative, Self-contained, Knowledge-centric. Anthropic promotes Skills as the Agent Skills open standard (`agentskills.io`) specification. Since its release in December 2025, over 27 Agent platforms (including OpenAI Codex CLI, Google Gemini CLI, Cursor, GitHub Copilot, etc.) have provided native support.

The three essential attributes for the success of Skills are as follows.

- **Declarative:** Pure Markdown format, readable and understandable by any large model, with no black-box binaries.
- **Self-contained:** One folder contains everything needed; copying equals installation, without requiring complex dependency management.
- **Knowledge-Centric:** The core value lies in the content itself rather than specific formats, unbound to any single platform. A Skill carefully designed by a developer in Claude Code can theoretically be seamlessly migrated to any AI platform supporting this standard, truly realizing "write once, run anywhere." This is not just a story about a product feature, but a story about industry standards. What you have learned is not merely "how to configure a certain function in Claude Code," but "how to build standardized knowledge packages for the AI Agent ecosystem."

**Chapter Summary** Skills directly address developers' daily pain points: how to make Claude permanently remember professional work paradigms, thereby avoiding having to re-"teach" it in every conversation. In the five-layer architecture of Claude Code, Skills reside at the knowledge layer—connecting upwards to the tools layer (defining "what can be done"), and downwards to the agent layer (deciding "who does it"). It works downwards to constrain tool behaviors via `allowed-tools`, works upwards utilizing the `skills` field to inject professional knowledge into sub-agents, and forms a parallel and complementary posture with `CLAUDE.md` (the former being permanently effective, while the latter is loaded on demand).

To understand this mechanism, the key lies in grasping four core designs.

- **Progressive Disclosure:** Realizing on-demand loading of knowledge—from `description` always residing in the context, to the `SKILL.md` body loaded upon trigger, and then to reference files read dynamically during execution. This mechanism can save 50% to 98% of Token consumption in most scenarios.
- **Semantic Trigger:** Ensuring professional capabilities are automatically activated at the appropriate time. Among this, `description` is the soul of this mechanism: it is not an instruction manual meant for humans to read, but a "semantic fingerprint" assisting Claude in its decisions.
- **Security Constraints:** Restricting knowledge within safe action boundaries via `allowed-tools`, achieving a security design that unites "knowledge and action."
- **Dynamic Injection:** Utilizing parameter passing (`$ARGUMENTS`) and command execution (`!command`) to endow Skills with the ability to respond flexibly based on the runtime context. Observing from a more macroscopic perspective, Skills is essentially a digital mapping of decades of human organizational knowledge management experience. In this system, `SKILL.md` is equivalent to the front page of a departmental SOP, `reference/` corresponds to the enterprise knowledge base, `templates/` equates to standardized output templates, and `scripts/` are automated toolsets—these roles can find precise counterparts in any mature enterprise. More critically, through the open standard (`agentskills.io`) specification, Skills extend this mapping from within the enterprise to the industry level: it propels knowledge management from an enterprise-level "SOP" to an industry-level "ISO standard," ensuring the knowledge value of Skills is no longer constrained to a single platform but becomes a universal asset across the entire ecosystem.

In Chapter 4, we will dive deeply into a milestone core feature in Claude Code's extension system—sub-agents. If Skills endows Claude with professional "knowledge," then sub-agents endow it with a more essential "capability": disassembling complex tasks and delegating them to multiple "experts" with independent contexts, allowing them to collaborate deeply in isolated spaces, and finally converging only the refined conclusions back to the main dialogue.

**Thought Questions**

1. In your daily work, what domain knowledge or work standards do you need to repeatedly explain to Claude? You might want to try refining them and encapsulating them into the `description` of a Skill. Under the strict limit of 1024 characters, which core information would you prioritize keeping?
2. Please design a Task Skill named `/deploy/`. In your design scheme, you need to focus on considering the following elements: the necessity of the `disable-model-invocation` configuration, the precise permission scope of `allowed-tools`, the context that needs to be preloaded before executing a command (such as current branch, latest build status, etc.), and a robust error handling path. Finally, please output the complete Frontmatter configuration and the `SKILL.md` skeleton.
3. Suppose 25 Skills have been installed in the project, and the total character budget for `description` is 16,000. How can you maximize the trigger accuracy of each Skill under the premise of strictly adhering to the budget? Which types of Skills are suitable for configuring `disable-model-invocation: true`, thereby freeing up precious `description` space?