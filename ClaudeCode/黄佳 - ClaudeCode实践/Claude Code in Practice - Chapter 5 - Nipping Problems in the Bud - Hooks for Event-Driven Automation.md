
## Nipping Problems in the Bud: Hooks for Event-Driven Automation

### Preparedness ensures success, and unpreparedness spells ruin.

An incident happened in the company. It was late at night, and Xiao Zhang was handling an urgent requirement. He used Claude Code to write code, and after passing the local tests, he executed the `git push` command to push the code to the remote repository, then immediately shut down his computer and went home to rest. At 9 AM the next morning, the team group chat instantly blew up: "Who pushed the env file? The API key has been exposed, all keys must be invalidated and reissued immediately!" It wasn't that Xiao Zhang didn't know that submitting env files was strictly prohibited; it was just that in the exhausted state of working overtime late at night, he neglected the pre-commit check. However, that mistakenly submitted env file actually contained database connection strings for 3 environments, an internal API key, and a test account password for a payment gateway. This mistake caused the entire team to spend half a day to complete the key rotation, access log troubleshooting, and correction of all referenced locations. As his direct supervisor, Xiao Bing's first reaction was: "This kind of error obviously could be intercepted automatically, why wasn't there any mechanism to prevent it?" "Because all the previous mechanisms were merely 'suggesting' to Claude what it should do," Brother Ka explained, "`CLAUDE.md` is a suggestion, Skills are guidance, but no one has ever told Claude 'you absolutely must not do this'. Hooks are different; they are not suggestions, but mandatory executions." Xiao Xue immediately chimed in: "Just like middleware in Web development? Passing a check before the request arrives?" "Exactly." Brother Ka nodded, "Whether it is Express's middleware, Django's request processing layer, or Spring's interceptor, their core logic is consistent: inserting additional processing logic before and after the execution of an operation. Hooks are exactly the practical implementation of this idea in the AI Agent scenario."

#### 5.1 The Positioning of Hooks in Claude's Extension System

Before diving into the technical details, let's first position Hooks within the extension system we have already learned. The 3 mechanisms introduced in the previous 4 chapters constitute a spectrum of "increasing influence": `CLAUDE.md` establishes project specifications and is read by Claude at the start of every conversation; Skills encapsulate domain workflows and can be triggered automatically or manually when Claude needs them; and SubAgents realize task delegation by isolating context. These 3 mechanisms share a common characteristic: they all act on Claude's cognitive level. `CLAUDE.md` tells Claude "what specifications to follow," Skills instruct it "how to handle such problems," and SubAgents direct it "who to delegate subtasks to." However, these are essentially still advisory guidance. As a language model, Claude theoretically possesses the ability to ignore any constraints in the Prompt. The working level of Hooks is completely different. It does not act on Claude's cognitive layer, but directly intercepts its behavior at the system execution layer. When Claude attempts to call the Bash tool to execute `rm -rf /`, the PreToolUse Hook does not "persuade" Claude to abandon the operation, but directly blocks the execution of the tool call at the system level—this means Claude's decision is forcibly overruled and cannot be implemented. This difference, in software engineering terms, is exactly the separation of Policy and Mechanism. `CLAUDE.md` and Skills define the policy, which is "what should be done"; whereas Hooks provide the mechanism, which is "once the policy is violated, it will be physically blocked." Just as operating system file permission management no longer relies on the application's "consciousness," the constraint of Hooks on Claude no longer relies on the "guidance" of the Prompt, but ensures execution through the coercive force at the bottom of the system.

Table 5-1 compares the differences among `CLAUDE.md`, Skills, and Hooks from 4 dimensions. **Table 5-1 Differences among CLAUDE.md, Skills, and Hooks**

|Mechanism|Working Level|Trigger Method|Constraint Nature|Control over Claude|Analogy|
|---|---|---|---|---|---|
|`CLAUDE.md`|Cognitive level|Always loaded|Suggestion|Guides behavior|Traffic signs|
|Skills|Cognitive level|Semantic match/Explicit trigger|Guidance|Standardizes workflows|Driving manual|
|Hooks|System execution level|Auto-triggered by events|Mandatory|Intercepts/Blocks|Roadblocks/Speed limiters|

This is just like traffic signs (`CLAUDE.md`) telling you "the speed limit here is 60 km/h," the driving manual (Skills) teaching you to "slow down at curves," while the roadblocks/speed limiters (Hooks) directly prevent your vehicle speed from reaching 120 km/h. They work best when used in coordination: traffic signs let you know the rules, the driving manual helps you master operation methods, and roadblocks/speed limiters ensure that when you forget the rules or make a misjudgment, the system can still hold the safety baseline.

#### 5.2 Event Lifecycle: 17 Events

Claude Code's event system has expanded from 7 in early versions to 17, fully covering the complete lifecycle of an AI session from start to termination, and from the main dialogue to SubAgent collaboration. This section divides these 17 events into session-level events, tool invocation events, SubAgent events, completion events, and newer event types. We don't need to memorize all 17 events; the key is to grasp their internal classification logic and core characteristics.

##### 5.2.1 Session-Level Events

Session-level events are responsible for managing the lifecycle of the entire session, mainly including the following 3 key events. **1 SessionStart** Triggered when a session starts or resumes. Its core capability is injecting environment variables via `CLAUDE_ENV_FILE`. Hook scripts can write `export` statements to this file, making the variables effective in all subsequent Bash commands. This means you can automatically configure the development environment at the start of the session. Please see the following code example. **2 SessionEnd** Triggered when a session terminates. Its matcher can distinguish different termination reasons: `clear` (cleared by user), `logout` (logged out), or `prompt_input_exit` (user exited input). This event is often used to clean up temporary resources or record session statistics. **3 PreCompact** Triggered before context compaction. Its matcher can distinguish `manual` (user actively executes `/compact`) and `auto` (automatically compacted after the context window is full). This event is suitable for backing up complete conversation records before compaction.

##### 5.2.2 Tool Invocation Events

This is the most core event category, covering the complete lifecycle of every tool invocation by Claude. Tool invocation events mainly include the following 5 key events. **1 PreToolUse (The most powerful event in the entire Hooks system)** Triggered after Claude decides to call a certain tool and before the tool is actually executed. It supports 3 operations:

- **Allow** (bypass permission checks and execute directly)
- **Deny** (block execution and explain the reason)
- **Modify** (`updatedInput`, execute after adjusting input parameters). The capability to "adjust input parameters" is particularly powerful: it allows you to silently add safety parameters to commands without interrupting the operation. Please see the following code example. The above example silently modifies the originally dangerous `rm -rf /tmp/test` command to `rm -rf /tmp/test --dry-run`, which not only allows Claude to continue completing the task but also avoids the risk of files being truly deleted.

**2 PostToolUse** Triggered after a tool is successfully executed. Although it cannot undo operations that have already occurred, it has two core functions: first, feeding back additional information to Claude via `additionalContext` (such as code Lint check results); second, post-processing the output (such as automatically formatting a newly written file). Furthermore, for MCP scenarios, this event also has an exclusive capability: it can directly replace the original output content of the MCP tool via the `updatedMCPToolOutput` field.

**3 PostToolUseFailure** Triggered after a tool execution fails, mainly used for error alerting and providing corrective feedback.

**4 PermissionRequest** Triggered just before a permission dialog box is about to pop up. Its key difference from PreToolUse lies in the trigger timing: PreToolUse triggers unconditionally before every tool invocation, while PermissionRequest is activated only when Claude requires the user to manually confirm permissions. Through this event, you can automatically approve or deny permission requests programmatically. Please see the following code example.

**5 UserPromptSubmit** Triggered after the user submits input and before Claude starts processing. This event is commonly used for input preprocessing or context injection scenarios, for example, automatically appending the current Git branch information every time the user sends a message.

##### 5.2.3 SubAgent Events

SubAgent events can be used in coordination with the SubAgent mechanism introduced in Chapter 3. **1 SubagentStart** Triggered when a SubAgent starts. Its matcher can filter based on the SubAgent's type name, supporting both built-in types (like Bash, Explore, Plan) and custom SubAgents defined in the `.claude/agents/` directory (like `code-reviewer`). Although SubagentStart cannot prevent the startup of a SubAgent, it will inject critical context information via `additionalContext` at runtime; for example, automatically loading the team's coding standards whenever `code-reviewer` starts.

**2 SubagentStop** Triggered after a SubAgent completes its task. Its behavioral logic is completely consistent with the global Stop event: it can either allow the stop operation or intercept the request, forcing the SubAgent to continue working until specific quality standards are met. Additionally, the input data of SubagentStop includes two key paths—`transcript_path` (main session record) and `agent_transcript_path` (the SubAgent's own conversation record). Leveraging this information, the Hook script can review the complete workflow of the SubAgent, thereby accurately assessing the quality of its output.

##### 5.2.4 Completion Events

**1 Stop** Triggered when Claude completes an entire round of responses. This is the core for implementing the "quality gating" mechanism: if it is detected that the output content does not meet preset standards (like code conventions, security policies, etc.), you can prevent the session from ending by setting `decision: "block"`, forcing Claude to continue modifying or perfecting the work until it meets the requirements.

**2 Notification** Triggered when Claude sends a system notification. Its matcher can precisely distinguish different types of notifications, such as `permission_prompt` (permission request), `idle_prompt` (idle prompt), or `auth_success` (authentication success). This event is frequently used for integrating custom notification channels, such as forwarding critical alerts to Slack or DingTalk, or triggering desktop pop-up reminders locally.

##### 5.2.5 Newer Event Types

Between 2025 and 2026, Claude Code expanded its event system, adding several key event types to support more complex collaboration and operations scenarios. **1 TeammateIdle and TaskCompleted** Designed specifically for multi-agent team collaboration. The former is triggered when a teammate agent is about to enter an idle state, and the latter is triggered when a task is marked as completed.

**2 ConfigChange** Triggered when a configuration file changes. This event is primarily used for auditing and compliance, helping developers track the history of settings changes and preventing unauthorized configuration modifications.

**3 WorktreeCreate and WorktreeRemove** Correspond to the creation and deletion operations of a Git Worktree, respectively. By intercepting these events, users can customize initialization settings for version control workflows (like automatically installing dependencies) or cleanup logic (like deleting temporary build artifacts).

##### 5.2.6 "Can it block": The Most Critical Dimension

Among all 17 events, "can it block" is the most core classification dimension; it determines whether an event is used for "controlling the flow" or solely for "observation and recording". Events with blocking capabilities include PreToolUse, PermissionRequest, UserPromptSubmit, Stop, SubagentStop, TeammateIdle, TaskCompleted, ConfigChange, and WorktreeCreate. The remaining events (such as PostToolUse, Notification, SubagentStart, etc.) belong to a read-only mode. They are mainly used to read context, inject additional information, or trigger side effects (like sending notifications), but cannot directly block or modify Claude's core execution logic. In daily development and operations, the 3 most commonly used events are: **PreToolUse** (the "goalkeeper" before tool execution), **PostToolUse** (the "quality guard" after tool execution), and **Stop** (the "quality gate" upon task completion). If time is limited, prioritizing the mastery of PreToolUse, PostToolUse, and Stop will allow you to build a robust automated closed loop.

#### 5.3 Configuration System: 6 Locations, 6 Purposes

Claude Code's configuration system adopts a layered stacking mechanism, with priority decreasing from top to bottom. Hooks' configurations use standard JSON format and strictly follow a six-layer priority architecture (see Figure 5-1). This design allows developers to deploy automation logic flexibly based on scope. ![[Pasted image 20260701132437.png]] _(Figure 5-1 The 6 layers and scopes of Hooks configuration)_

- Enterprise policy configuration (`managed-settings.json`)
- Project configuration (`.claude/settings.json`)
- Project local configuration (`.claude/settings.local.json`)
- User global configuration (`~/.claude/settings.json`)
- Plugin built-in Hooks (`hooks/hooks.json`)
- SubAgent Frontmatter inline Hooks

**Project configuration** (`.claude/settings.json`) is the core carrier for team collaboration. After committing it to the Git repository, all members can automatically synchronize the team's agreed-upon security checks and automation rules when cloning the project. **Project local configuration** (`.claude/settings.local.json`) is ignored by `.gitignore` and is suitable for personal scenarios that need to override team default configurations. **User global configuration** (`~/.claude/settings.json`) is used to manage cross-project personal preferences, such as custom log formats or desktop notification methods.

The configuration structure adopts a 3-layer nested design: event type -> matcher group -> Hook processor list. Please see the following code example. The **matcher** field is used to specify the tool range applicable to this group of Hooks.

- Bash: Matches all Bash calls.
- Write|Edit: Matches Write or Edit tools (the pipe symbol `|` represents logical "OR").
- *: Matches all tools. For lifecycle events like Stop, Notification, and UserPromptSubmit, the matcher field will be ignored because these events do not target specific tools. In contrast, in SubagentStart or SubagentStop events, the matcher matches the SubAgent's type name, not the tool name.

#### 5.4 3 Processor Types: A Ladder of Certainty

Hook processors contain 3 types, constituting a ladder of "decreasing certainty and increasing flexibility". The specific choice depends on the level of judgment required by the validation logic.

##### 5.4.1 command type: Deterministic Rules

This type is used to execute Shell commands or scripts. As the most commonly used and most reliable type, deterministic rules are always more trustworthy than the judgments of large models. Please see the following code example. The `command` type Hook receives JSON-formatted context data (including `session_id`, `tool_name`, `tool_input`, etc.) via stdin, outputs JSON-formatted decisions via stdout, and expresses its final intent based on the exit code.

- Exit code 0: Indicates success. The system parses the JSON in stdout as the basis for decision-making.
- Exit code 2: Indicates intentional blocking. The system feeds the content of stderr back to Claude as the reason for the error.
- Other exit codes: Indicate script anomalies. The content of stderr is only visible in debug mode but will not block the main workflow.

The design of exit code 2 is crucial; it strictly distinguishes between "intentionally blocking an operation" and "script self-failure." A script self-failure should not hinder the normal workflow—just as when a smoke alarm itself malfunctions, it should not prohibit people from entering and exiting the building.

##### 5.4.2 prompt type: Single Large Model Evaluation

When validation logic requires a certain degree of judgment but does not need to execute multi-step operations, it is recommended to use the `prompt` type. This type calls a small model (usually Haiku) to evaluate the current situation. Please see the following code example. Among them, `$ARGUMENTS` is a placeholder that will be replaced at runtime by the complete input JSON received by the Hook. The response of the large model needs to follow the JSON format below. JSON format for allowing pass-through. JSON format for denying operations.

##### 5.4.3 agent type: Multi-round SubAgent Validation

When validation logic requires actually reviewing code files, executing searches, or multi-step operations to reach a conclusion, the `agent` type should be used. This type launches a SubAgent so that it can utilize tools like Read, Grep, and Glob to conduct multi-round deep validation. Please see the following code example. The `agent` type SubAgent must return a decision after running a maximum of 50 rounds of dialogue/operations. Its response format is completely identical to the `prompt` type, returning a JSON object containing `ok` (boolean) and `reason` (string).

When selecting a Hook processor type, one should follow the degradation principle of "if `command` type can be used, `prompt` type is not recommended; if `prompt` type can be used, `agent` type is not recommended" (see Figure 5-2). ![[Pasted image 20260701132736.png]] *(Figure 5-2 Comparison of Certainty and Flexibility among 3 Hook Processor Types:

- `command` type (Pattern Match) ~0ms
- `prompt` type (Single Evaluation) ~2s
- `agent` type (Multi-round Validation) ~30s)*

Deterministic rules (such as pattern matching, filename checking, and regular expressions) always outperform large model judgments in speed and reliability. Only when the validation logic truly requires "comprehension" (semantic analysis) or "code inspection capability" (multi-file context retrieval) should you consider upgrading to the `prompt` or `agent` type.

#### 5.5 hookSpecificOutput: The Protocol for Communicating with Claude

The output format of the PreToolUse Hook underwent an important upgrade in 2025. Early versions used top-level `decision` and `reason` fields, while the new version recommends using the `permissionDecision` format nested within the `hookSpecificOutput` object. Although both formats are currently supported, new code should follow the new specification. Please see the following code example. `permissionDecision` supports 3 values.

- `allow`: Bypass the permission system and execute directly.
- `deny`: Block execution.
- `ask`: Hand over to the user for confirmation.

Among them, `ask` is a subtle yet practical option—it does not automatically reject, but expresses the attitude of "I'm not sure, please let a human decide."

The `additionalContext` field applies to all event types, and its content will be injected into Claude's context. This mechanism builds an efficient feedback loop; for example, the PostToolUse Hook can feed static code analysis (Lint) results back to Claude via `additionalContext`, and upon receiving this information, Claude will automatically fix the issues, requiring no human intervention throughout the process.

In addition, all events support the following general top-level fields. `continue: false`: Equivalent to an "emergency brake". Regardless of the current event stage, this setting will immediately terminate Claude's processing. `systemMessage`: The content of this field will be displayed directly to the user and will not be passed to Claude.

#### 5.6 Engineering Practice 1: Security Defense System

Let's return to the thorny case at the beginning of this chapter. Now, we will use Hooks to build a complete security defense system, protecting the project from the following 3 lines of defense: interception of dangerous commands, protection of sensitive files, and full operational auditing.

##### 5.6.1 PreToolUse: Interception of Dangerous Commands

The first line of defense is aimed at intercepting Bash commands that could cause a disaster. This script incorporates several key details in its design.

- Debug information is output to standard error (stderr, i.e., `>&2`), rather than standard output (stdout). This is because stdout must be strictly reserved for outputting decision results in JSON format.
- Uses the `jq` tool to parse input JSON data, avoiding fragile manual string matching.
- Every interception operation is accompanied by a clear, specific explanation of the reason. Please see the following code example.

##### 5.6.2 PreToolUse: Sensitive File Protection

The second line of defense focuses on protecting sensitive files from accidental modifications. The matcher for this Hook is configured as "Write|Edit", ensuring it is only triggered when file writing or editing operations occur, thereby blocking risks at the source. Please see the following code example.

##### 5.6.3 PostToolUse: Full Operational Auditing

The third line of defense is full operational auditing. By configuring `matcher: "*"`, this tool can capture and record all tool invocations. In enterprise environments with strict compliance requirements, this is an indispensable mechanism; it clearly answers the core auditing question: "What operation did Claude execute on what target at what time?" Please see the following code example.

##### 5.6.4 Complete Configuration

Integrating the 3 independent scripts from sections 5.6.1 to 5.6.3 into `.claude/settings.json` forms a rigorous defense-in-depth system. The specific configuration is as follows. This configuration builds a complete safety closed loop covering "**pre-interception, in-process protection, and post-audit**": interception of dangerous commands -> protection of sensitive files -> full operational auditing. Through this combined mechanism, we upgrade fragile processes originally relying on "manual caution" to an automated security system where "code is law." Just as the case at the beginning of this chapter showed, if the second line of defense had been applied, Xiao Zhang's "tragedy" could have been completely avoided.

#### 5.7 Engineering Practice 2: Code Quality Automation

Security defense aims to "nip problems in the bud," while code quality automation strives to "ensure excellent delivery." The practical examples in this section will deeply demonstrate the collaborative mechanism between PostToolUse and Stop Hooks.

##### 5.7.1 PostToolUse: Auto-Formatting

After Claude writes a file each time, the system will automatically trigger a formatting tool. The brilliance of this Hook lies in achieving "separation of concerns": Claude does not need to perceive what specific formatting standards the project adopts; it only needs to focus on writing code logic, and the formatting process will be completed seamlessly in the background. Please see the following code example. The script introduces `command -v` for environmental checking. If it detects that the formatting tool is not installed, the Hook will quietly skip it rather than throw an error. This reflects the design principle of "graceful degradation": exceptions from the Hook itself should not block the normal operation of core workflows.

##### 5.7.2 PostToolUse: Lint Feedback Loop

Auto-formatting ensures the "beauty" of the code, while Lint checking guarantees the code's "correctness." The core of the example introduced in this subsection lies in using `additionalContext` to feed Lint check results back to Claude, thereby building an automated closed loop of "modify -> check -> feedback -> fix." Please see the following code example. When Claude completes modifying JavaScript/TypeScript files, if a Lint error is triggered, the script will inject the error details into `additionalContext`. After Claude reads this context, it will automatically analyze and fix the problem. The entire process requires no human intervention, achieving a fully automated iteration from code generation to meeting quality standards.

##### 5.7.3 Stop Hook: Test Quality Gating

The Stop Hook is a line of defense for quality assurance: when Claude declares a task complete, the test suite is automatically triggered. If the test fails, the system will prevent the session from ending and forcefully require continued fixing. Please see the following code example. The `stop_hook_active` field in the script is the key to preventing the system from falling into an "infinite loop," and its logic is similar to the termination condition of a recursive function. When the Stop Hook execution fails, if the system attempts to fix and triggers the Hook again, `stop_hook_active` will be set to `true`. Subsequently, upon detecting this flag, the script will choose to pass it through, thereby exiting the loop—just as a recursive function must set a termination condition, the Stop Hook must also possess a clear exit mechanism. If this check is missing, Claude will fall into a dead loop of "fix -> test fails -> fix again," unable to extricate itself.

#### 5.8 SubAgent Hooks: Precise Context Management

In Chapter 3, we learned that SubAgents achieve task delegation by isolating context. The Hooks system provides two exclusive events for this—SubagentStart and SubagentStop. However, what is even more critical is the third mechanism: defining Hooks directly within the SubAgent's Frontmatter.

##### 5.8.1 Global vs. Frontmatter: The Question of Precision

Suppose you have a SubAgent named `db-reader`, specifically used to execute SQL queries. If you need to review every Bash command it executes to prevent SQL injection risks, configuring a Hook in the global `settings.json` is not the optimal solution. Because global configurations will indiscriminately intercept all Bash commands, covering operations unrelated to the database such as compiling code, running tests, or installing dependencies. This not only wastes system performance but is also highly prone to causing false interceptions. A better solution is to define the Hook directly within the SubAgent's Frontmatter. Please see the following code example. The core advantage of defining a Hook using Frontmatter lies in the **tight binding of lifecycles**: the Hook activates as the SubAgent starts and is automatically cleaned up after the task concludes. Additionally, the configuration is integrated within the same file as the SubAgent definition, allowing it to be distributed along with the `.md` file; users do not need to additionally modify the global `settings.json`, significantly reducing configuration complexity.

##### 5.8.2 SubagentStart: Auto-Injecting Context

A typical application scenario for the SubagentStart Hook is dynamically injecting team standards when a SubAgent starts. For example, whenever the `code-reviewer` SubAgent is launched, the system can automatically inject the team's coding standards. Please see the following code example.

##### 5.8.3 SubagentStop: Validating Output Quality

The SubagentStop Hook can be used to verify whether a SubAgent's work product meets the standard. By reading the SubAgent's complete interaction record using `agent_transcript_path`, the system can execute fine-grained quality acceptance. Please see the following code example. The 3 layers of defense mechanisms introduced from sections 5.8.1 to 5.8.3 each perform their own duties, constructing a complete closed loop of quality assurance.

- Frontmatter Hook: Responsible for internal self-inspection, ensuring the SubAgent asks itself "Is my output complete?".
- SubagentStart Hook: Responsible for external injection, providing it with the necessary context upon startup ("giving it necessary background information").
- SubagentStop Hook: Responsible for external acceptance, strictly verifying "did its work product meet the standard?" upon completion.

#### 5.9 Asynchronous Hooks: Background Execution Without Blocking

By default, the execution of Hook scripts is synchronous and blocking—Claude will pause the current workflow until the script finishes executing and returns a result. For fast pattern matching checks, this delay of a few milliseconds is usually negligible; however, for time-consuming operations like running test suites, calling external APIs, or sending notifications, synchronous blocking will significantly slow down Claude's response speed, affecting user experience. For this reason, Claude Code, released in early 2026, introduced support for asynchronous Hooks. After configuring `"async": true`, the Hook will run non-blocking in the background. Claude does not need to wait for it to complete and can immediately proceed to subsequent work. Once the asynchronous Hook finishes executing, its output results will automatically be passed to Claude in the next dialogue turn for its reference or processing. Two key limitations of asynchronous Hooks determine their applicable boundaries.

- Type restriction: Only `command` type Hooks support asynchronous execution. `prompt` and `agent` type Hooks must run synchronously.
- Interception capability restriction: Asynchronous Hooks cannot block the current operation. Since the main flow continues to execute the moment the Hook starts, asynchronous Hooks lose the opportunity to intervene before the operation occurs.

Therefore, **asynchronous Hooks are suitable for "post-processing" tasks like log recording, asynchronous notifications, background data validation, and non-critical quality audits, but are not suitable for security checks that require real-time blocking (such as SQL injection defense and sensitive information filtering)**.

#### 5.10 Environment Variables and Debugging

##### 5.10.1 Environment Variables Available for Hooks

Table 5-2 lists the environment variables available for Hooks, covering the scopes of the variables and their core uses. **Table 5-2 Environment Variables Available for Hooks**

|Environment Variable|Scope|Core Use|
|---|---|---|
|`CLAUDE_PROJECT_DIR`|All Hooks|Obtain the absolute path of the current project's root directory|
|`CLAUDE_SESSION_ID`|All Hooks|Unique identifier for the current session|
|`CLAUDE_TOOL_NAME`|All Hooks|Name of the tool that triggered the current Hook|
|`CLAUDE_FILE_PATH`|All Hooks|Absolute path of the file involved in the current operation (if applicable)|
|`CLAUDE_ENV_FILE`|SessionStart only|Path to the persistent file for environment variables|
|`CLAUDE_NOTIFICATION`|Notification only|Contains specific notification message content|
|`CLAUDE_CODE_REMOTE`|All Hooks|Boolean flag, indicating whether running in a remote Web environment|
|`CLAUDE_PLUGIN_ROOT`|Plugin Hook only|Path to the root directory where the plugin is installed|

##### 5.10.2 The "Three Axes" of Debugging

The 3 core methods for debugging Claude Hook scripts are as follows. **First, output debug information to stderr.** Because stdout is dedicated to outputting JSON decision results, all debug information must be redirected to stderr. **Second, manually test Hook scripts.** Verify the script logic directly by constructing mock inputs. **Third, use `claude --debug` to view complete Hook execution details.** Debug mode will show the list of matched Hooks, the execution time of each script, and the returned results.

##### 5.10.3 Common Pitfalls

During the use of Hooks, there are two frequently overlooked problems. Problem 1: If Shell configuration files (like `~/.zshrc` or `~/.bashrc`) contain unconditional `echo` statements (e.g., used to output welcome messages), these outputs will pollute the standard output (stdout), resulting in a JSON parsing failure. The solution is to wrap these `echo` statements using the `[[ $- == *i* ]]` condition check, ensuring they are executed only in an interactive Shell. Problem 2: After directly editing `settings.json`, the Hook often does not take effect immediately. This is because Claude Code only captures the configuration state at startup; file modifications during runtime are not automatically synchronized. If it needs to take effect, the user must confirm changes in the `/hooks` menu or restart the current session.

#### 5.11 Engineering Design Methodology

When facing specific automation requirements, designing a Hook scheme requires clarifying the following 3 core dimensions.

- **Interception Timing (Event Selection)**: For pre-operation interception, use PreToolUse or UserPromptSubmit; for post-operation feedback, use PostToolUse; for check upon completion, use Stop or SubagentStop; for lifecycle management, use SessionStart or SessionEnd.
- **Judgment Method (Type Selection)**: If rules are clear (like pattern matching, file checking), use `command` type; if semantic judgment is needed but input is sufficient, use `prompt` type; if deep code analysis is needed, use `agent` type.
- **Configuration Scope (Location Selection)**: For general team norms, configure in `.claude/settings.json`; for personal preference settings, configure in `~/.claude/settings.json`; for SubAgent-exclusive checks, configure in Frontmatter.

The design process usually follows a "three-step" strategy. **Step 1**, first configure an audit log Hook based on the PostToolUse event with a matcher of `matcher: "*"` to observe Claude's actual tool invocation patterns and accumulate several days of real running data. **Step 2**, based on the audit data, identify high-risk operational patterns, and then design targeted PreToolUse interception rules. **Step 3**, gradually tighten interception rules while always retaining logging functions to ensure that when false interceptions occur, the root cause of the problem can be quickly located.

**Brother Ka's Remarks** Hooks are team-level infrastructure, not personal experimental toys. Hooks configured in `.claude/settings.json` will take effect for all members who clone that repository. If a member is accidentally intercepted for unknown reasons, it will severely hinder the workflow and cause frustration. Therefore, be sure to follow these guidelines.

- Before submitting Hook configurations, you must fully discuss and reach a consensus with the team.
- Every interception rule must be accompanied by a clear explanation of the reason, informing the user of the specific cause of interception.
- Use audit logs to monitor the trigger frequency of Hooks in real-time, in order to promptly discover and correct false interceptions.

#### Chapter Summary

Hooks fill the gap that `CLAUDE.md` and Skills cannot cover: they can enforce team conventions at the system level during critical nodes of task execution. As the only component in Claude Code's extension mechanism that runs at the system execution layer rather than the cognitive layer, Hooks do not rely on Claude's comprehension capability or the Prompt's guidance, but intercept tool calls directly through script logic. **17 event types** cover the complete flow from session startup to completion, and from main dialogue to SubAgents. **3 processor types** (including `command`, `prompt`, and `agent`) form a capability ladder of "decreasing certainty and increasing flexibility". The selection principle is: first choose the `command` type, next is the `prompt` type, and finally consider the `agent` type. Supporting the configuration of inline Hooks in SubAgent Frontmatter allows achieving more precise control than global configurations. Introducing **asynchronous Hooks** can effectively solve blocking issues caused by long-duration operations.

Examining from the perspective of engineering methodology, the design of Hooks strictly follows the evolutionary principle of "observe first, control later": first, use the PostToolUse event to record audit logs to deeply analyze actual tool invocation patterns and behavioral characteristics; then, based on the accumulated data insights, specifically design PreToolUse interception rules to ensure the accuracy and necessity of the policies. Built-in `stop_hook_active` flags effectively prevent Hook logic from triggering infinite retry calls on its own. Stipulating that exit code 2 represents "intentional blocking" and clearly distinguishing it from system errors reflects the system's deep deliberation on exception handling and state definition.

The core value of Hooks lies not in "expanding capability boundaries" (letting Claude do more), but in "solidifying the execution foundation" (making everything Claude does more reliable). It upgrades those critical checks that "should happen but are often forgotten" (such as sensitive file protection, code formatting specifications, and test acceptance processes) from "soft constraints" relying on consciousness to unavoidable system-level "hard constraints." The painful lesson mentioned in the opening case of this chapter only required a simple PreToolUse Hook to be fundamentally avoided.

At this point, we have deeply analyzed Claude Code's internal extension mechanisms. In Chapter 6, we will cast our vision outward to explore how it connects to the vast external world.

#### Thought Questions

1. In the daily development workflow, what are some errors that "should not happen but occasionally occur due to carelessness"? Please select one scenario and design a complete Hook scheme: clarify the trigger event, Hook type, Matcher configuration rules, and the specific conditions the script needs to validate.
2. What are the similarities and differences between the "quality gating" mode of the Stop Hook and automated testing in CI/CD pipelines? In what scenarios should a Stop Hook be prioritized, and which scenarios are more suitable to rely on CI/CD?
3. PreToolUse possesses interception capability, whereas PostToolUse is merely used for feedback. What is the engineering significance of this asymmetrical design? If PostToolUse were granted interception capability (i.e., undoing an operation that has already been executed), what problems would it cause?
4. Please compare the two Hook configuration methods: global `settings.json` and SubAgent Frontmatter. In your project, which Hooks should be placed in the global configuration, and which are more suitable to be defined within the Frontmatter of specific SubAgents?