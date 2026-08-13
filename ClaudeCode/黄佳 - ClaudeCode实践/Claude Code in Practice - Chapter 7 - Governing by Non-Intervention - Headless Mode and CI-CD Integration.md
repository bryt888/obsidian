
## Governing by Non-Intervention: Headless Mode and CI/CD Integration


#### Devising strategies within a command tent to ensure victory a thousand miles away.

The company's technical team has only 3 senior engineers, yet they need to review over twenty PRs every day. Conventionally, each PR must be approved by at least one senior engineer before it can be merged. The consequences are obvious: a massive backlog of PRs, forcing a reduction in release frequency; developers have to switch to other tasks while waiting for reviews, leading to frequent context interruptions and compromised efficiency. Even more severely, during peak review loads, the quality of manual reviews begins to decline—exhausted reviewers are highly prone to missing security vulnerabilities or performance risks, and once these issues flow into the production environment, the cost of fixing them multiplies.

After a week of research, Xiao Xue proposed a solution: introduce Claude as the "preliminary reviewer" for PRs, automatically executing security checks, code quality analysis, and test coverage evaluation, and generating structured review reports. Human reviewers will then make the final decisions based on this report, focusing their energy on core areas such as business logic and architecture design. This solution does not replace manual review, but rather frees engineers from repetitive mechanical checks, allowing them to focus on high-level judgments that only humans can perform.

"Can Claude Code run automatically unattended?" Xiao Bing asked.

"This is exactly the core of the Headless mode," Brother Ka explained. "In the previous 6 chapters, we have been using Claude in 'conversational mode'—you input instructions, it gives feedback, and so on. But starting from this chapter, Claude Code will break away from real-time human operation and be able to work independently in the background, in CI/CD pipelines, or even unattended at 3 AM. The significance of this transformation is comparable to the emergence of Docker, which liberated applications from being 'bound to a specific machine' to being 'runnable in any environment'."

#### 7.1 From Human-Computer Interaction to Unattended Operation: A Crucial Architectural Evolution

In interactive mode, Claude Code plays the role of a "present conversational partner": the user inputs instructions, Claude responds instantly, and waits for the user to confirm the results before continuing to the next round of interaction. This mode is very suitable for exploratory work, such as code debugging, brainstorming architectural solutions, or progressive refactoring of modules. However, in automated scenarios, no one is waiting at the keyboard, nor is there anyone to gatekeep intermediate steps, so the entire process must possess the ability to run independently from start to finish with zero human intervention.

The Headless mode is exactly designed for this. Through the `-p` (or `--print`) parameter, the user can explicitly tell Claude Code: "This is a non-interactive call; please output the result directly after completing the task and exit, without waiting for any user input."

However, `-p` is merely the entry point. What truly transforms Claude Code into a "programmable component" is its comprehensive parameter system. To deeply understand this system, one must first clarify the core differences between the interactive mode and the Headless mode (see Table 7-1).

**Table 7-1 Core Differences Between Interactive Mode and Headless Mode**

|Dimension|User Interface|Input Method|Output Method|Permission Confirmation|Session Persistence|Cost Control|Typical Scenarios|
|---|---|---|---|---|---|---|---|
|Interactive Mode|Terminal TUI, real-time rendering|Multi-turn real-time dialogue|Streaming Markdown rendering|Pop-up interactive prompts, requires manual confirmation|Auto-saved, supports resume|Relies on manual observation and interruption|Development debugging, architectural exploration|
|Headless Mode|No interface, standard output (stdout) only|One-time Prompt (or stdin pipeline)|Plain text/JSON/Streaming JSON|Auto-skips or executes based on whitelist|Not saved by default (can be specified on demand)|Hard limit via `--max-turns`/`--max-budget-usd`|CI/CD pipelines, scheduled tasks, batch processing|

The significance of this evolution goes far beyond simply "having one less interface." In interactive mode, humans actually play three crucial roles: permission approver (making real-time decisions on whether to allow the execution of specific commands), quality checker (instantly judging whether the output results meet expectations), and process controller (dynamically adjusting the next instruction based on intermediate results). When all 3 of these roles are absent, the system must "hardcode" all rules into parameters before running: Which tools are authorized to be called? What is the maximum number of conversation turns allowed? What is the cost budget limit? What format must the output follow for downstream programs to parse? This is exactly the core mission of the massive parameter system of the Headless mode—transforming a fuzzy process originally relying on human intuition and real-time judgment into a deterministic, predictable, and auto-executable set of engineering standards.

#### 7.2 Core Parameter System: Control Across 4 Dimensions

Brother Ka sketched a clear four-quadrant diagram on the whiteboard, illustrating the 4 dimensions of the Headless mode parameter system, as shown in Figure 7-1. ![[Pasted image 20260701141200.png]] _Figure 7-1 The 4 Dimensions of the Headless Mode Parameter System_

"Look," Brother Ka pointed to the whiteboard and said, "the parameters of the Headless mode are not a disorganized mess, but can be summarized into these 4 dimensions. Any production-grade unattended invocation must make clear and firm decisions across these 4 dimensions."

##### 7.2.1 Output Format Control

The `--output-format` parameter is the "protocol converter" connecting Claude Code with downstream automated pipelines. It clarifies the structure of the output data and directly determines how subsequent scripts or systems will consume these results.

This parameter includes 3 core formats—text format, json format, and stream-json format.

**1. text format** The simplest plain text output, preserving the natural flow of language, suitable for direct reading or simple script processing.

**2. json format** Outputs a complete, strictly structured JSON object. It not only contains the task results but also encapsulates rich execution metadata, serving as the cornerstone for building robust CI/CD pipelines.

When using the json format, the returned object contains the full set of information required for auditing and monitoring.

In automated operations and cost optimization, the following fields are crucial.

- `total_cost_usd` can be used for precise cost tracking; in CI/CD pipelines, users can log the cost of each run into a database to generate reports like "AI review cost per PR", or even automatically trigger alerts or blockages for departments that exceed their budget.
- `num_turns` can be used for monitoring execution turns, reflecting the actual depth of the model's interaction (compared to the set `--max-turns` limit).
- `usage.cache_read_input_tokens` reflects the effectiveness of Prompt caching. In scenarios where the same System Prompt or foundational codebase is called frequently, a higher `cache_read_input_tokens` means more computations are skipped, resulting in lower costs and faster speeds.

**3. stream-json format** Outputs JSON events line by line (i.e., NDJSON format), designed specifically for real-time monitoring of long tasks. An example is as follows.

In the output, each line is an independent JSON object and is strictly arranged in chronological order.

Please pay special attention to the `system/init` event in the first line, which reports initialization information such as the model version of the current session, the list of available tools, and the status of MCP servers. This feature is very useful when debugging CI configuration issues.

"The choice among the 3 formats is actually very simple," Brother Ka summarized. "If manual reading is needed, use the plain text format; if machine parsing is needed, use the JSON format; if real-time monitoring is needed, use the stream-json format."

##### 7.2.2 Cost Guardrails

In interactive mode, you can press `Ctrl+C` at any time to interrupt Claude's execution; but in Headless mode, due to the lack of this "manual circuit breaker" mechanism, control must be implemented through parameters. Please see the following code example.

`--max-turns` and `--max-budget-usd` impose constraints from different dimensions.

- `--max-turns`: Limits the number of interaction turns between Claude and the tools. Each time a tool is called, it counts as one turn. For example, a task that requires reading 5 files and then providing a conclusion typically only requires 6 turns.
- `--max-budget-usd`: Limits the total amount in USD for API calls. Even if the number of interaction turns is small (e.g., only 3 turns), if each turn involves a massive amount of Tokens (like reading a giant log file), the cumulative cost can still be high; in such cases, this parameter plays a critical protective role.

The behavioral mechanisms when both are triggered are entirely different.

- When `--max-turns` is exhausted, Claude will output the conclusions it has reached up to the current turn. Although this conclusion might not be complete yet, it usually still holds reference value. In the JSON output, the `subtype` of such events is marked as `error_max_turns`.
- When `--max-budget-usd` is triggered, the execution process will be forcibly stopped immediately, and no subsequent conclusions will be output. In the JSON output, the `subtype` of such events is marked as `error_max_budget_usd`.

"**In a production environment, be sure to configure both parameters simultaneously**." Brother Ka emphasized:

- `--max-turns` is used to prevent Claude from falling into infinite recursion when processing complex tasks,
- and `--max-budget-usd` avoids exhausting the API budget due to an unexpected large-scale analysis. These two lines of defense are both indispensable.

##### 7.2.3 Security Boundaries: Tool Permissions

In Headless mode, due to the lack of a human confirmation step for tool calls, Claude's capability boundaries must be drawn in advance; this is the most critical security setting in CI/CD scenarios. Please see the following code example.

`--allowedTools` (whitelist) and `--disallowedTools` (blacklist) are entirely different in their security semantics.

- **Whitelists are based on the "closed-world assumption"**: They only allow the use of explicitly listed tools, rejecting all other tools universally.
- **Blacklists are based on the "open-world assumption"**: Except for explicitly prohibited tools, all other tools can be used.

In automated scenarios, the security of a whitelist is always higher than that of a blacklist. This is because when using a whitelist, you do not need to foresee all potential dangerous tools; you only need to list any tools that are genuinely required.

Please note that `--allowedTools` supports pattern-matching syntax. For example, `Bash(git log *)` means "allow the use of the Bash tool, but strictly limited to executing commands starting with `git log`". This fine-grained control mechanism is particularly practical in scenarios where partial Shell capabilities (like Git operations) are needed, yet you dare not grant full Bash permissions.

Additionally, there is a more aggressive parameter—`--dangerously-skip-permissions`, used to skip all permission checks. Do not ignore the word `dangerously` in its name; it is by no means decorative. In a CI environment, using this parameter is strictly prohibited unless you are running in a strictly isolated container and are confident that Claude's operations cannot cause impacts beyond the container's scope.

##### 7.2.4 Execution Control: Model Prompt and Structured Output

We can control Claude's "way of thinking" through model selection, System Prompt customization, and structured output.

**1. Model Selection** Different tasks fit different models, requiring trade-offs based on needs. Please see the following code example.

`--fallback-model` is a crucial yet easily overlooked parameter in production. During API peak hours, if the main model is unavailable, causing the CI pipeline to block, all PRs will stagnate. Setting a fallback model ensures the pipeline remains running at the expense of a slight drop in quality, preventing process interruptions.

**2. System Prompt Customization** Customize the System Prompt to define Claude's role and behavioral boundaries. Please see the following code example.

The core difference between the `--system-prompt` and `--append-system-prompt` parameters lies in their different ways of handling Claude's default behavior.

- `--system-prompt`: Completely overwrites Claude's default System Prompt. Claude will no longer know it is a coding assistant, nor will it understand how to correctly use built-in tools like Read, Grep, Bash, etc. It will mechanically follow the new instructions you provide, which easily leads to tool invocation failures or uncontrollable behaviors.
- `--append-system-prompt`: Appends your customized content after Claude's default System Prompt. Claude retains all basic capabilities while strictly following the additional instructions you added. This is suitable for the vast majority of CI/CD automation scenarios.

**3. Structured Output** Forces Claude to output data conforming to a specific JSON Schema, thereby achieving programmatic, seamless integration. Please see the following code example.

After using the `--json-schema` parameter, the system will automatically validate Claude's output. If the generated content does not match the defined Schema, Claude will automatically retry the generation process until it produces valid and legal data. The final validated structured data will appear in the `structured_output` field of the JSON output. This feature is extremely useful in scenarios that require programmatic handling of Claude's output—you don't need to write complex regular expressions or heuristic rules to parse free text; you can directly obtain standard JSON objects for use by downstream scripts, databases, or alerting systems.

#### 7.3 Unix Pipelines: Integrating Claude into the Command Line Workflow

"I noticed an interesting design," Xiao Xue said, "Does Claude Code seem to support standard input (stdin)?"

"That's right," Brother Ka answered, "This is exactly one of the most elegant features of the Headless mode—Claude Code can seamlessly connect to Unix pipelines, becoming a flexible link in your toolchain."

The core of the Unix philosophy is "each tool does one thing well, and combine them through pipelines." Claude Code perfectly aligns with this design concept. Please see the following code example.

What's even more interesting is multi-stage pipeline combinations—Claude can both receive upstream data and pass the processing results to downstream tools. Please see the following code example.

In batch processing scenarios, this pipeline combination capability transforms Claude into a "smart filter." Traditional text processing tools (like Grep, Awk, Sed) excel at exact matching based on patterns but lack semantic understanding capabilities; while Claude is proficient in semantic analysis, efficiently handling large-scale data is not its strong suit. Combining the two via a pipeline achieves complementary advantages: first use Grep to filter out key content, and then hand it over to Claude for deep semantic analysis.

#### 7.4 Practice 1: Automated Code Review with GitHub Actions

The theoretical elaboration up to here is sufficient; next, we will demonstrate through a complete engineering example.

Anthropic provides two GitHub Actions integration schemes: one is adopting the official Action (`anthropics/claude-code-action@v1`), which is highly encapsulated and easy to configure; the other is calling the CLI directly, which offers greater flexibility and finer control.

##### 7.4.1 Adopting the Official Action

Claude Code has built-in convenient installation commands: execute `/install-github-app` in interactive mode, which guides the user through the entire process of GitHub App installation, Secret configuration, and Workflow file creation.

If you choose to configure it manually, the steps are not cumbersome either. First, go to the GitHub repository, then click **Settings -> Secrets -> Actions**, add `ANTHROPIC_API_KEY`, and subsequently create the following Workflow file.

There are 3 points in this configuration worth special clarification.

**1. Two Trigger Modes** The official Action v1 possesses the ability to automatically detect trigger modes.

- **Agent Mode**: Triggered when the `prompt` parameter is provided in the configuration. Every time a PR is created or updated, Claude automatically executes the preset review task.
- **Tag Mode**: Triggered when a user mentions `@claude` in a PR comment. At this time, Claude will act like a team member, interactively responding to specific questions.

These two modes can coexist in the same Workflow file without conflicting.

**2. Parameter Pass-Through Mechanism** The official Action v1 has significantly simplified the configuration structure. All Claude Code CLI parameters are now uniformly passed through the `claude_args` field. Independent fields like `max_turns` and `allowed_tools` from previous Beta versions have been deprecated and are no longer supported.

**3. Concurrency Control Strategy** By setting `cancel-in-progress: true`, efficient concurrency control can be achieved. When multiple commits are pushed consecutively to the same PR, the system automatically cancels the running old tasks and only retains the review process triggered by the latest commit. This mechanism effectively avoids redundant executions, thereby saving API call costs.

##### 7.4.2 Calling the CLI Directly

When you need to implement more complex logic—for example, deciding whether to block a PR merge based on the review results, publishing the review report as an official PR review rather than a regular comment, or applying differentiated review strategies to different types of files—calling the Claude CLI directly will offer greater flexibility.

Although the above configuration is more complex than the official Action, it grants the user 3 key capabilities: **cost transparency** (the specific cost of each review will be displayed directly in the comment), **quality gating** (when a Critical-level severe issue is detected, the Workflow will automatically fail and block the PR merge), and **path filtering** (restricted by `paths`, triggering reviews only when changes occur in the `src/` directory).

In addition, attention should be paid to the `CLAUDE_CODE_DISABLE_NONESSENTIAL_TRAFFIC` environment variable. This is a standard best practice in CI/CD pipelines, used to disable non-essential functions in automated scenarios such as automatic update checks, error reporting, and telemetry data transmission all at once.

#### 7.5 Practice 2: Multi-Stage CI Pipelines

The case introduced in Section 7.4 belongs to a single-stage review. In actual projects, you might need Claude to participate in multiple stages, for example, first reviewing code quality, then scanning for security vulnerabilities, and finally checking test coverage. Each stage often has different focus areas, tool permission limitations, and cost budgets.

Below is a configuration example of a multi-stage AI pipeline.

The above configuration involves 3 design key points.

- **Model Tiering**: For lightweight tasks (such as format checking), choose Haiku (this model has fast response times and low costs); for heavyweight tasks (such as security scanning and test coverage analysis), choose Sonnet (leveraging its stronger reasoning capabilities for deep analysis). Matching the appropriate "computing power ratio" based on the complexity of different tasks achieves the optimal balance between efficiency and effectiveness.
- **Budget Tiering**: Set a strict upper limit for simple tasks (e.g., $0.05), and reserve ample room for complex tasks (e.g., $1). Use the `--max-budget-usd` parameter to precisely control the maximum expenditure of a single run, ensuring the overall pipeline cost is controllable and predictable.
- **Parallel Execution**: Both the `security-scan` and `coverage-check` tasks are set with `needs: lint-check`, meaning they must wait for the first stage to complete before starting. Because there are no mutual dependencies between these two tasks, GitHub Actions will automatically run them in parallel.

#### 7.6 Streaming Output: Real-Time Monitoring of Long-Running Tasks

For analysis tasks that take a long time, the stream-json format allows users to observe Claude's execution progress in real-time, without waiting for the entire job to finish to see the final results. Please see the following code example.

The stream-json format defines a complete suite of event protocols. In addition to `system` (initialization), `assistant` (assistant reply), `user` (user input), and `result` (result summary) involved in the above case, there are two types of special events worth noting.

- `stream_event` (requires adding the `--include-partial-messages` parameter): Provides incremental data of Claude's output, precise to every Token-level text fragment. Suitable for user interfaces that need to implement a "typewriter effect," but it typically does not need to be enabled in CI automation flows.
- `system/compact_boundary`: Context compaction boundary event. When the length of the dialogue history approaches the model's context window limit, Claude automatically triggers a mechanism to compress earlier content to free up space. In long-running analysis tasks, if this event is detected, it indicates that the current task has consumed a large amount of context resources, and one needs to be aware of the potential risk of context loss.

#### 7.7 Session Management: Maintaining Context Across Steps

In Headless mode, the system does not persist session states by default—each call is treated as an independent, entirely new context. However, in complex scenarios involving multiple rounds of interactions, it is often necessary to maintain context continuity across multiple calls. Please see the following code example.

The core value of the `--resume` parameter is allowing Claude to completely "inherit" the context state of the previous round of dialogue, including the file contents read, the analysis steps executed, and the intermediate conclusions reached. Compared to manually repeating context information in a new Prompt, this method not only significantly improves interaction efficiency but also ensures the accuracy and coherence of the model's reasoning.

In addition to basic session recovery, the CLI tool also provides a series of advanced parameters to handle complex interactive scenarios. Please see the following code example.

`--fork-session` is a highly practical parameter, and its application logic is akin to the Git branching mechanism in version control systems. It "forks" an entirely new session instance from a specific point in time of a specified session. The original session (the main branch) remains unchanged and unaffected; the new session (the branch) evolves independently starting from that fork point. This feature perfectly meets the needs of "what-if analysis". For example, after completing the basic analysis of the same codebase, developers can use this parameter to create multiple branches to explore different refactoring directions separately.

#### 7.8 CI Environment Configuration: Production-Grade Checklist

When deploying Claude Code in a real CI/CD environment, besides the core parameters `-p` and `--allowedTools`, a series of environment variables also need to be set. Below is the "CI Environment Configuration Checklist" compiled by Brother Ka.

##### 7.8.1 Mandatory Environment Variables to Set

The environment variables that must be set are as follows.

`DISABLE_AUTOUPDATER` is a crucial environment variable in CI/CD environments. If this variable is not set, Claude Code might attempt to auto-update during execution, leading to the risk of version inconsistencies or unexpected service interruptions. By contrast, `CLAUDE_CODE_DISABLE_NONESSENTIAL_TRAFFIC` is a convenient "master switch" capable of disabling all non-essential background traffic (including auto-updates, telemetry, and error reporting) all at once, simplifying the configuration process.

##### 7.8.2 Performance Tuning Parameters

Parameters related to performance tuning are as follows.

##### 7.8.3 Complete Configuration Template in GitHub Actions

Below is a complete configuration snippet for deploying Claude Code in a GitHub Actions workflow.

##### 7.8.4 MCP Server Configuration in CI

If CI tasks need to leverage MCP servers for extended capabilities (such as database queries, external API calls, etc.), dedicated configuration files can be loaded using the `--mcp-config` parameter, paired with `--strict-mcp-config` to ensure environment isolation. Please see the following code example.

The `--strict-mcp-config` parameter indicates enabling strict mode, forcing Claude to use only the configuration file specified by `--mcp-config`. It ignores other MCP configurations in the user's home directory and the project's root directory. This is important in CI environments—it prevents residual local MCP configurations from developers from being accidentally loaded.

#### 7.9 Cross-Platform CI/CD Integration

Although GitHub Actions has official Action support, Claude Code's Headless mode is essentially platform-agnostic. As long as the runtime environment supports Node.js, it can be deployed on any CI/CD platform.

##### 7.9.1 GitLab CI/CD Configuration

The following configuration demonstrates how to execute code review tasks in a GitLab Runner and output the results in a structured format.

It is worth noting that the partnership between Anthropic and GitLab is deepening. Currently, GitLab has launched official AI features (in Beta), introducing dedicated variables like `AI_FLOW_INPUT` and `AI_FLOW_CONTEXT`, eliminating the need to manually concatenate Prompt contexts; through direct integration with `gitlab-mcp-server`, it allows AI models to access the GitLab API safely.

##### 7.9.2 Jenkins Pipeline Integration Configuration

In Jenkins, Claude Code can be easily integrated via Declarative Pipelines. The following configuration demonstrates how to securely manage credentials, execute code reviews in Headless mode, and parse structured output.

##### 7.9.3 Local Automation Scripts

In addition to CI/CD platforms, Claude Code's Headless mode is also well-suited for encapsulation into local scripts, available for team members to self-check before committing code, or to serve as part of a local automated toolchain.

Below is an enhanced version of a `review.sh` script, equipped with environment checking, dynamic file filtering, structured report generation, and error-handling capabilities.

#### 7.10 Security Principles and Best Practices

"Running an Agent in CI/CD is a security matter that requires serious attention." Brother Ka's tone became a bit serious.

This is not baseless anxiety. An improperly configured Headless mode task could trigger the following severe risks.

- **Secrets Leakage**: Reading and outputting `.env`, `config.json`, or hardcoded Secrets files.
- **Destructive Operations**: Executing malicious Shell commands, damaging the build environment, or overwriting code.
- **Cost Out of Control**: Falling into an infinite loop or being induced to generate massive amounts of meaningless content, consuming hefty API call fees.
- **Content Pollution**: Outputting hallucinated content, inappropriate speech, or even being injected with attack payloads in PR/MR comments.

Defense strategies must build a defense-in-depth system from the following 5 levels.

##### 7.10.1 Principle of Least Privilege

In practical application scenarios, the following measures can be taken.

Granting Claude unrestricted Bash execution permissions is strictly prohibited. Even in scenarios where Shell capabilities are a must, the scope of executable commands must be strictly restricted via pattern matching.

##### 7.10.2 Secrets Management

Regarding Secrets management, a comparison of correct and incorrect practices is explained here.

GitHub Actions will automatically mask referenced secrets values in the logs (displayed as `***`).

Although GitHub has a log-masking mechanism, if the Prompt instructs the model to "list all environment variables" or "print configuration details," the model might regurgitate the key values in its output text. Such model-generated text cannot be caught by the automatic masking mechanism, leading to key leakage.

Requiring the model to access, read, or output any environment variables and key values in the Prompt is strictly prohibited.

##### 7.10.3 Container Isolation

The following shows a GitHub Actions workflow configuration snippet used to build a highly secure "sandbox" environment to run code review tasks.

`--read-only`: Sets the file system as read-only, preventing write operations (used in conjunction with `--tmpfs /tmp` to provide a writable space for temporary files).

`--network none`: Completely severs network connections in review tasks that do not require external access. In this mode, Claude can only read files within the codebase and cannot communicate with the outside world.

##### 7.10.4 Cost Protection

The following demonstrates configuration snippets in a GitHub Actions workflow used to optimize resource usage and reduce operating costs.

Path filtering and concurrency control are the first line of defense for cost optimization. Compared to limiting the resource consumption of a single run, avoiding unnecessary triggers at the source can reduce the total cost more effectively.

##### 7.10.5 Audit Logs

The following shows a configuration snippet for executing an AI task and persistently saving running logs containing full audit information.

The JSON output of every Headless mode call automatically includes key audit fields such as `session_id`, `total_cost_usd`, `num_turns`, and `usage`. To strictly comply with regulatory requirements, it is recommended to push these logs in real-time to a centralized logging system (such as ELK, Datadog, etc.) to achieve long-term archiving and deep analysis.

**Brother Ka's Remarks** The above five layers of defense are by no means "optional questions," but rather "mandatory questions" in a production environment: the Principle of Least Privilege prevents Claude from "doing things it shouldn't do," Secrets management strictly guards against credential leaks, Container Isolation blocks the risk of escape, Cost Protection avoids bill inflation, and Audit Logs ensure full traceability. Missing any single layer means a fatal flaw has appeared in the security defense line.

#### 7.11 From CLI to Agent SDK: The Programming Interface for Headless Mode

The examples introduced previously all call Claude Code via the CLI, but in actual engineering practice, Claude Code also provides an Agent SDK (supporting TypeScript and Python), allowing developers to directly use programming languages rather than command-line parameters to control the Headless mode execution flow.

Below are examples of the TypeScript Agent SDK and Python Agent SDK.

The relationship between the CLI and the Agent SDK is just like the difference between curl and an HTTP client library: the CLI excels in being lightweight and convenient, making it suitable for simple automation scripts and CI/CD pipelines; whereas the Agent SDK is born specifically for complex logical scenarios—whether it involves dynamically adjusting subsequent Prompts based on previous review results, fine-grained process orchestration across multiple Claude calls, or deeply integrating Claude into large application systems, the Agent SDK provides the programming flexibility developers need. In Chapter 8, we will deeply analyze the specific usage of the Agent SDK.

#### 7.12 Progressive Implementation Strategy

"Having talked about so many technical details," Xiao Bing asked, "if our team wants to introduce Headless mode for PR reviews, how should we proceed step by step?"

Brother Ka suggested adopting a four-stage progressive implementation path.

**Stage One: Observer Mode.** Claude only executes read-only analyses, outputting review reports without blocking any processes. The review reports are posted as comments in the PR for team members to reference or ignore. The core goal of this stage is to build trust—allowing the team to intuitively experience Claude's review quality while identifying its limitations (such as scenarios prone to false positives), thereby iteratively optimizing Prompts and parameter configurations.

**Stage Two: Advisor Mode.** After accumulating sufficient positive feedback, incorporate Claude's review results into CI status monitoring. When a Critical issue is discovered, mark the CI status as "Warning" (yellow), but still do not block the merge. Human reviewers need to make final judgments in conjunction with Claude's report. The core goal of this stage is human-machine collaboration.

**Stage Three: Gatekeeper Mode.** Claude's review becomes one of the necessary conditions for code merging. Once a severe security issue is detected, the CI status turns red immediately and blocks the merge. However, an "escape hatch" must be provided at this stage, allowing personnel with specific permissions to manually override Claude's blocking decision (similar to skipping an unstable test case).

**Stage Four: Active Fix Mode.** Claude can not only discover issues but also automatically fix format errors, supplement missing exception-handling logic, or generate unit tests. This stage requires opening up Write tool permissions, but all fixes must be submitted in the form of a new Commit, and directly modifying the PR author's original code is strictly prohibited.

"Avoid trying to reach the final goal in one step," Brother Ka emphasized, "Each stage should run for at least 2 to 4 weeks, closely monitoring false positive rates, cost trends, and team acceptance. The core value of AI-assisted reviews is not about 'perfectly replacing humans,' but rather 'allowing human reviewers to focus their limited energy on the most valuable decisions.'"

#### Chapter Summary

Headless mode has driven a role transformation for Claude Code—evolving from a smart assistant confined to terminal interaction into a programmable component that can be seamlessly embedded into various engineering flows. This shift is defined by parameters across 4 dimensions.

- `-p` (Entry point): Switches Claude Code's running mode from an interactive session to non-interactive execution, laying the foundation for automation.
- `--output-format` (Data interface): Standardizes the output structure to ensure downstream systems can efficiently parse and call review results.
- `--allowedTools` (Security boundary): Strictly limits tool permissions to ensure operations within automated environments are compliant and controllable.
- `--max-turns` and `--max-budget-usd` (Cost guardrails): Sets upper limits for execution turns and budgets to prevent resource loss of control in unattended scenarios.

At the engineering architecture level, the greatest value of Headless mode lies in granting Claude Code the status of an "equal citizen in the CI/CD ecosystem." Just like unit testing, static analysis, or container image builds, AI code review officially becomes a standard component in the pipeline: it requires clear input definitions, deterministic output formats, quantifiable cost budgets, and configuration files that can be incorporated into version control.

Regarding implementation strategies, it is recommended to follow a path starting with "Observer Mode" and gradually evolving into "Gatekeeper Mode." This progressive process aims to reserve ample time for the team to build trust, accumulate practical data, and optimize configurations. Remember not to rush for success—the core value of AI-assisted reviews does not pursue a "perfect replacement" of manual work, but rather uses intelligent filtering to allow human reviewers to focus their precious energy on the most critically valuable decision-making links.

In Chapter 8, we will delve deeply into the transition from command-line parameters to the world of programming interfaces (Agent SDK). You will master how to use TypeScript or Python code to replace tedious CLI parameters and flexibly control Claude's behavioral logic, thereby building a truly customized, highly integrated AI application system.

#### Thought Questions

1. Integrating Claude Code's PR review into existing CI/CD pipelines requires not only overcoming technical configuration hurdles but also properly resolving core issues in the team collaboration process: How to establish the authority of the review results? How to clarify its functional boundaries with existing Linter tools? How to build an efficient false-positive handling mechanism?
2. In which scenarios will `--max-turns 5` and `--max-budget-usd 0.50` be the first to trigger a limit, respectively? When designing Headless mode tasks for a production environment, which metric should you prioritize? Please explain your reasons.
3. Please clarify the CI platform used by your team (such as GitHub Actions, GitLab CI, Jenkins, or others). Based on this, please tailor a Claude Code integration scheme for the project, covering trigger conditions, tool permissions, cost budgets, and security policies, and output the complete configuration file.