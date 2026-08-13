
# 凡事预则立，不预则废。

公司里发生了这样一件事。

那是一个深夜，小张正在处理一个紧急需求。他用Claude Code编写代码，本地测试通过后，便执行了`git push`命令将代码推送至远程仓库，随即关机回家休息。

次日早晨九点，团队群消息瞬间炸锅：“谁把env文件推上去了？API key已暴露，所有密钥必须立即作废并重发！”

小张并非不知道env文件严禁提交，只是在深夜加班的疲惫状态下，疏忽了提交前的检查。然而，那个被误提交的env文件中，竟包含3个环境的数据库连接字符串、一个内部API密钥和一个支付网关的测试账号密码。这一失误导致整个团队耗费半天时间，才完成密钥轮换、访问日志排查以及所有引用位置的修正。

小冰作为他的直接领导，第一反应是：“这种错误明明可以自动拦截，为什么没有任何机制能阻止它？”

“因为之前的所有机制，仅仅是在‘建议’Claude该做什么。”咖哥解释道，“CLAUDE.md是建议，Skills是指导，但从来没有人告诉Claude‘这件事你绝对不能做’。而Hooks不同，它不是建议，而是强制执行。”

小雪随即接话：“就像Web开发里的中间件？在请求到达之前先过一道检查？”

“准确。”咖哥点了点头，“无论是Express的中间件、Django的请求处理层，还是Spring的拦截器，它们的核心逻辑都是一致的：在操作执行的前后，插入额外的处理逻辑。Hooks正是这一思想在AI Agent场景中的落地实现。”

## 5.1 Hooks在Claude扩展体系中的定位

在深入分析技术细节之前，我们先将Hooks置于已学的扩展体系中进行定位。

前4章介绍的3种机制构成了一条“影响力递增”的谱系：CLAUDE.md确立项目规范，由Claude在每次对话启动时读取；Skills封装领域工作流，可在Claude需要时自动或手动触发；而子智能体则通过隔离上下文来实现任务委派。

这3种机制存在一个共同特征：它们均作用于Claude的认知层面。CLAUDE.md告诉Claude“应遵守何种规范”，Skills指导其“遇到此类问题该如何处理”，子智能体则指示其“应将子任务委派给谁”。然而，这些本质上仍属于建议性指导。作为语言模型，Claude在理论上具备忽略Prompt中任何约束的能力。

Hooks的工作层面截然不同。它不作用于Claude的认知层，而是直接在系统执行层拦截其行为。

当Claude试图调用Bash工具执行`rm -rf /`时，PreToolUse Hook并非“劝说”Claude放弃该操作，而是在系统层面直接阻断该工具调用的执行——这意味着Claude的决策被强制推翻，无法落地。

这种区别，用软件工程的术语来说，正是策略(Policy)与机制(Mechanism)的分离。CLAUDE.md和Skills定义的是策略，即“应该怎么做”；而Hooks提供的是机制，即“一旦违反策略，将被物理阻止”。正如操作系统的文件权限管理不再依赖于应用程序的“自觉”，Hooks对Claude的约束也不再依赖于Prompt的“引导”，而是通过系统底层的强制力来保障执行。

表5-1从4个维度对比了CLAUDE.md、Skills与Hooks三者的差异。

**表5-1 CLAUDE.md、Skills与Hooks的差异**

|机制|工作层面|触发方式|约束性质|对Claude的控制|类比|
|---|---|---|---|---|---|
|CLAUDE.md|认知层|始终加载|建议|引导行为|交通标志|
|Skills|认知层|语义匹配/显式触发|指导|规范工作流|驾驶手册|
|Hooks|系统执行层|事件自动触发|强制|拦截/阻止|路障/限速器|

这就好比，交通标志(CLAUDE.md)告诉你“这里限速60 km/h”，驾驶手册(Skills)教你“弯道应该减速”，而路障/限速器(Hooks)则直接让你的车速达不到120 km/h。三者配合使用时效果最佳：交通标志让你知晓规则，驾驶手册让你掌握操作方法，而路障/限速器确保在你忘记规则或判断失误时，系统依然能守住安全底线。

## 5.2 事件生命周期：17个事件

Claude Code的事件系统已从早期版本的7个扩展至17个，全面覆盖AI会话从启动到终止、从主对话到子智能体协作的完整生命周期。本节将这17个事件分为会话级事件、工具调用事件、子智能体事件、完成事件与较新的事件类型。我们不需要死记硬背全部17个事件，关键在于掌握其内在的分类逻辑与核心特征。

### 5.2.1 会话级事件

会话级事件负责管理整个会话的生命周期，主要包含以下3个关键事件。

**1 SessionStart**

在会话启动或恢复时触发。其核心能力是通过CLAUDE_ENV_FILE注入环境变量。Hook脚本可向该文件写入export语句，使变量在后续所有Bash命令中生效。这意味着你可以在会话开始时自动配置开发环境。请看以下代码示例。

```
#!/bin/bash
# session-init.sh - SessionStart Hook

if [ -n "$CLAUDE_ENV_FILE" ]; then
    echo 'export NODE_ENV=development' >> "$CLAUDE_ENV_FILE"
    echo 'export DEBUG_LOG=true' >> "$CLAUDE_ENV_FILE"
fi
exit 0
```

**2 SessionEnd**

在会话终止时触发。其匹配器(matcher)可区分不同的终止原因：clear（用户清除）、logout（登出）或prompt_input_exit（用户退出输入）。该事件常用于清理临时资源或记录会话统计信息。

**3 PreCompact**

在上下文压缩前触发。其匹配器可区分manual（用户主动执行/compact）和auto（上下文窗口满后自动压缩）。此事件适合在压缩前备份完整的对话记录。

### 5.2.2 工具调用事件

这是最核心的事件类别，涵盖了Claude每次工具调用的完整生命周期。工具调用事件主要包含以下5个关键事件。

**1 PreToolUse（整个Hooks系统中最强大的事件）**

在Claude决定调用某个工具之后、工具实际执行之前触发。它支持3种操作：
- **允许**（allow，绕过权限检查直接执行）
- **拒绝**（deny，阻止执行并说明原因）
- **修改**（updatedInput，调整输入参数后执行）。

“调整输入参数”这一能力尤其强大：它允许你在不中断操作的前提下，静默地为命令添加安全参数。请看以下代码示例。

```
{
  "hookSpecificOutput": {
    "hookEventName": "PreToolUse",
    "permissionDecision": "allow",
    "updatedInput": {
      "command": "rm -rf /tmp/test --dry-run"
    }
  }
}
```

上述示例将原本危险的`rm -rf /tmp/test`命令静默修改为`rm -rf /tmp/test --dry-run`，既让Claude继续完成任务，又避免了文件被真正删除的风险。

**2 PostToolUse**

在工具成功执行后触发。虽然它无法撤销已发生的操作，但具备两项核心功能：一是通过additionalContext向Claude反馈额外信息（如代码Lint检查结果）；二是对输出进行后处理（如自动格式化刚写入的文件）。此外，针对MCP场景，该事件还拥有专属能力：可通过updatedMCPToolOutput字段直接替换MCP工具的原始输出内容。

**3 PostToolUseFailure**

在工具执行失败后触发，主要用于错误告警以及提供纠正性反馈。

**4 PermissionRequest**

在权限对话框即将弹出时触发。它与PreToolUse的关键区别在于触发时机：PreToolUse会在每次工具调用前无条件触发，而PermissionRequest仅在Claude需要用户手动确认权限时才被激活。通过该事件，你可以以编程方式自动批准或拒绝权限请求。请看以下代码示例。

```
{
  "hookSpecificOutput": {
    "hookEventName": "PermissionRequest",
    "decision": {
      "behavior": "allow",
      "updatedPermissions": {}
    }
  }
}
```

**5 UserPromptSubmit**

在用户提交输入后、Claude开始处理之前触发。该事件常用于输入预处理或上下文注入场景，例如，在用户每次发送消息时，自动附加当前的Git分支信息。

### 5.2.3 子智能体事件

子智能体事件可与第3章介绍的子智能体机制协同使用。

**1 SubagentStart**

在子智能体启动时触发。其匹配器可根据子智能体的类型名称进行筛选，既支持内置类型（如Bash、Explore、Plan），也涵盖在.claude/agents/目录中定义的自定义子智能体（如code-reviewer）。虽然SubagentStart无法阻止子智能体的启动，但运行时会通过additionalContext注入关键上下文信息，例如，每当启动code-reviewer时，自动加载团队的编码规范。

**2 SubagentStop**

在子智能体完成任务后触发。其行为逻辑与全局Stop事件完全一致：既可以放行停止操作，也可以拦截该请求，强制子智能体继续工作直至满足特定的质量标准。此外，SubagentStop的输入数据包含两个关键路径——transcript_path（主会话记录）和agent_transcript_path（子智能体自身的对话记录）。借助这些信息，Hook脚本能够复盘子智能体的完整工作流程，从而对其产出质量进行精准评估。

### 5.2.4 完成事件

**1 Stop**

在Claude完成整轮响应时触发。这是实现“质量门控”机制的核心：如果检测到输出内容未满足预设标准（如代码规范、安全策略等），可以通过设置decision: "block"阻止会话结束，强制Claude继续修改或完善工作，直至符合要求。

**2 Notification**

在Claude发送系统通知时触发。其匹配器能够精准区分不同类型的通知，如permission_prompt（权限请求）、idle_prompt（空闲提示）或auth_success（认证成功）等。该事件常用于自定义通知渠道的集成，例如，将关键警报转发至Slack或钉钉，或在本地触发桌面弹窗提醒。

### 5.2.5 较新的事件类型

在2025年到2026年间，Claude Code扩展了其事件体系，新增了几个关键事件类型以支持更复杂的协作与运维场景。

**1 TeammateIdle与TaskCompleted**

专为多智能体团队协作设计。前者在队友智能体即将进入空闲状态时触发，后者在任务被标记为完成时触发。

**2 ConfigChange**

在配置文件发生变更时触发。该事件主要用于审计与合规，帮助开发者追踪设置变化历史，防止未经授权的配置修改。

**3 WorktreeCreate 与 WorktreeRemove**

分别对应GitWorktree的创建与删除操作。通过拦截这些事件，用户可以自定义版本控制工作流的初始化设置（如自动安装依赖）或清理逻辑（如删除临时构建产物）。

### 5.2.6 “能否阻止”：最关键的维度

在全部17个事件中，“能否阻止”是最核心的分类维度，它决定了事件是用于“控制流程”还是仅用于“观察记录”。

具备阻止能力的事件包括PreToolUse、PermissionRequest、UserPromptSubmit、Stop、SubagentStop、TeammateIdle、TaskCompleted、ConfigChange、WorktreeCreate。

其余事件（如PostToolUse、Notification、SubagentStart等）属于只读模式。它们主要用于读取上下文、注入额外的信息或触发侧边效应（如发送通知），但无法直接阻止或修改Claude的核心执行逻辑。

在日常开发与运维中，最常用的3个事件是：**PreToolUse**（工具执行前的“守门员”）、**PostToolUse**（工具执行后的“质量守卫”）、**Stop**（任务完成时的“质量门控”）。如果时间有限，优先精通PreToolUse、PostToolUse和Stop，即可构建出健壮的自动化闭环。

## 5.3 配置体系：6个位置，6种用途

Claude Code的配置系统采用分层叠加机制，优先级从上至下依次降低。Hooks的配置采用标准的JSON格式，并严格遵循六层优先级架构（见图5-1）。这种设计允许开发者根据作用域灵活部署自动化逻辑。

![[Pasted image 20260701132437.png]]

图5-1 Hooks配置的6个层级与作用域)
- 企业策略配置(managed-settings.json) 
- 项目配置(.claude/settings.json) 
- 项目本地配置(.claude/settings.local.json) 
- 用户全局配置(~/.claude/settings.json) 
- 插件内置Hooks(hooks/hooks.json) 
- 子智能体frontmatter内联Hooks)

**项目配置**(.claude/settings.json)是团队协作的核心载体。将其提交至Git仓库后，所有成员在克隆项目时即可自动同步团队约定的安全检查与自动化规则。**项目本地配置**(.claude/settings.local.json)已被.gitignore忽略，适用于需要覆盖团队默认配置的个人场景。**用户全局配置**(~/.claude/settings.json)则用于管理跨项目的个人偏好，如自定义日志格式或桌面通知方式。

配置结构采用3层嵌套设计：事件类型→matcher组→Hook处理器列表。请看以下代码示例。

```
{
  "hooks": {
    "PreToolUse": [
      {
        "matcher": "Bash",
        "hooks": [
          {
            "type": "command",
            "command": "./.claude/hooks/block-dangerous.sh",
            "timeout": 30
          }
        ]
      }
    ],
    "PostToolUse": [
      {
        "matcher": "Write|Edit",
        "hooks": [
          {
            "type": "command",
            "command": "prettier --write \"$CLAUDE_FILE_PATH\""
          }
        ]
      }
    ]
  }
}
```

**matcher**字段用于指定该组Hook适用的工具范围。

- `Bash`：匹配所有Bash调用。
- `Write|Edit`：匹配Write或Edit工具（管道符`|`表示逻辑“或”）。
- `*`：匹配所有工具。

对于Stop、Notification、UserPromptSubmit等生命周期事件，matcher字段将被忽略，因为这些事件不针对特定工具。而在SubagentStart或SubagentStop事件中，matcher匹配的是子智能体类型名称，而非工具名称。

## 5.4 3种处理器类型：确定性的阶梯

Hook处理器包含3种类型，构成了一个“确定性递减、灵活性递增”的阶梯。具体选择取决于验证逻辑所需的判断力度。

### 5.4.1 command类型：确定性规则

该类型用于执行Shell命令或脚本。作为最常用且最可靠的类型，确定性规则永远比大模型的判断更为可信。请看以下代码示例。

```
{
  "type": "command",
  "command": "./.claude/hooks/check-security.sh",
  "timeout": 30
}
```

command类型的Hook通过stdin接收JSON格式的上下文数据（包含session_id、tool_name、tool_input等），通过stdout输出JSON格式的决策，并依据退出码表达最终意图。

- 退出码0：表示成功。系统将stdout中的JSON解析结果作为决策依据。
- 退出码2：表示有意阻止。系统将stderr的内容作为错误原因反馈给Claude。
- 其他退出码：表示脚本异常。stderr内容仅在调试模式下可见，但不会阻断主流程。

退出码2的设计至关重要，它严格区分了“有意阻止操作”与“脚本自身故障”。脚本自身故障不应阻碍正常工作流——这正如烟雾报警器自身发生故障时，不应因此禁止人员进出大楼。

### 5.4.2 prompt类型：单次大模型评估

当验证逻辑需要一定的判断力，但不需要执行多步操作时，建议使用prompt类型。该类型会调用小型的模型（通常为Haiku）对当前情况进行评估。请看以下代码示例。

```
{
  "type": "prompt",
  "prompt": "评估这段代码修改是否引入了安全漏洞。$ARGUMENTS",
  "model": "claude-haiku-4-5",
  "timeout": 30
}
```

其中，$ARGUMENTS为占位符，运行时将被替换为Hook接收到的完整输入JSON。大模型的响应需要遵循以下JSON格式。

允许通过的JSON格式。

```
{"ok": true, "reason": "代码修改安全，未引入已知漏洞模式"}
```

拒绝操作的JSON格式。

```
{"ok": false, "reason": "检测到潜在的SQL注入风险：用户输入未经转义直接拼接到查询字符串"}
```

### 5.4.3 agent类型：多轮子智能体验证

当验证逻辑需要实际查看代码文件、执行搜索或多步操作才能得出结论时，应使用agent类型。该类型会启动一个子智能体，以便能够利用Read、Grep、Glob等工具进行多轮深度验证。请看以下代码示例。

```
{
  "type": "agent",
  "prompt": "检查所有修改的文件是否通过了单元测试。运行测试套件并验证结果。$ARGUMENTS",
  "timeout": 120
}
```

agent类型的子智能体最多运行50轮对话/操作后必须返回决策。其响应格式与prompt类型完全一致，返回包含ok（布尔值）和reason（字符串）的JSON对象。

选择Hook处理器类型时，应遵循“能用command类型的不建议用prompt类型，能用prompt类型的不建议用agent类型”的降级原则（见图5-2）。
![[Pasted image 20260701132736.png]]
_(图5-2 3种Hook处理器类型的确定性与灵活性对比：
- command类型(模式匹配) 约0ms  
- prompt类型(单次评估) 约2s 
- agent类型(多轮验证) 约30s)

确定性规则（如模式匹配、文件名检查、正则表达式）在速度和可靠性上永远优于大模型判断。只有当验证逻辑确实需要“理解力”（语义分析）或“检查代码能力”（多文件上下文检索）时，才考虑升级到prompt类型或agent类型。

## 5.5 hookSpecificOutput：与Claude交流的协议

PreToolUse Hook的输出格式于2025年迎来了重要升级。早期版本采用顶层的decision和reason字段，而新版本则推荐使用嵌套在hookSpecificOutput对象中的permissionDecision格式。尽管目前两种格式均受支持，但新代码应遵循新规范。请看以下代码示例。

```
{
  "hookSpecificOutput": {
    "hookEventName": "PreToolUse",
    "permissionDecision": "deny",
    "permissionDecisionReason": "此命令试图删除受保护的系统目录",
    "additionalContext": "受保护的路径模式: /etc, /usr, /var"
  }
}
```

permissionDecision支持3种值。

- `allow`：绕过权限系统直接执行。
- `deny`：阻止执行。
- `ask`：交由用户确认。

其中，`ask`是一个微妙却实用的选项——它并非自动拒绝，而是表达“我不确定，请由人来决定”的态度。

`additionalContext`字段适用于所有事件类型，其内容将被注入Claude的上下文中。这一机制构建了一个高效的反馈闭环，例如，PostToolUse Hook可通过additionalContext将代码静态分析(Lint)结果反馈给Claude，Claude在接收到这些信息后会自动修复问题，整个过程不需要人工干预。

此外，所有事件均支持以下几个通用的顶层字段。

```
{
  "continue": false,
  "stopReason": "检测到安全违规，会话已终止",
  "suppressOutput": false,
  "systemMessage": "警告：此操作已被安全策略拦截"
}
```

`continue: false`：相当于“紧急制动”。无论当前处于何种事件阶段，该设置都会立即终止Claude的处理。

`systemMessage`：该字段的内容将直接显示给用户，而不会传递给Claude。

## 5.6 工程实战一：安全防护体系

让我们回到本章开篇那个令人棘手的案例。现在，我们将利用Hooks构建一套完整的安全防护体系，从以下3道防线为项目保驾护航：危险命令拦截、敏感文件保护及全量操作审计。

### 5.6.1 PreToolUse：危险命令拦截

第一道防线旨在拦截可能引发灾难的Bash命令。该脚本在设计上蕴含了几个关键细节。

- 调试信息输出至标准错误（`stderr`，即`>&2`），而非标准输出（`stdout`）。这是因为`stdout`必须严格保留用于输出JSON格式的决策结果。
- 使用`jq`工具解析输入的JSON数据，避免了脆弱的手动字符串匹配。
- 每一次拦截操作都附带清晰、具体的原因说明。

请看以下代码示例。

```
#!/bin/bash
# .claude/hooks/block-dangerous.sh

set -e
INPUT=$(cat)

# 提取命令(调试信息输出至stderr)
COMMAND=$(echo "$INPUT" | jq -r '.tool_input.command // ""')
echo "DEBUG: Checking command: $COMMAND" >&2

# 危险命令模式列表
DANGEROUS_PATTERNS=(
    "rm -rf /"
    "rm -rf ~"
    "rm -rf \$HOME"
    "> /dev/sd"
    "mkfs."
    ":(){:|:&};:"               # Fork bomb (Fork炸弹)
    "chmod -R 777 /"
    "git push --force origin main"
    "git push --force origin master"
    "git reset --hard origin"
    "DROP DATABASE"
    "DROP TABLE"
    "TRUNCATE"
    "curl.*| sh"               # 危险的管道执行
    "curl.*| bash"
)

for pattern in "${DANGEROUS_PATTERNS[@]}"; do
    if [[ "$COMMAND" == *"$pattern"* ]]; then
        echo "BLOCKED: $pattern" >&2
        cat <<EOF
{
    "hookSpecificOutput": {
        "hookEventName": "PreToolUse",
        "permissionDecision": "deny",
        "permissionDecisionReason": "拦截危险命令模式: $pattern"
    }
}
EOF
        exit 2
    fi
done

echo '{"hookSpecificOutput":{"hookEventName":"PreToolUse","permissionDecision":"allow"}}'
exit 0
```

### 5.6.2 PreToolUse：敏感文件保护

第二道防线专注于保护敏感文件免受意外修改。该Hook的匹配器配置为“Write|Edit”，确保仅在发生文件写入或编辑操作时触发，从而在源头阻断风险。请看以下代码示例。

```
#!/bin/bash
# .claude/hooks/protect-files.sh

set -e
INPUT=$(cat)
FILE_PATH=$(echo "$INPUT" | jq -r '.tool_input.file_path // ""')

if [ -z "$FILE_PATH" ]; then
    echo '{"hookSpecificOutput":{"hookEventName":"PreToolUse","permissionDecision":"allow"}}'
    exit 0
fi

FILENAME=$(basename "$FILE_PATH")

# 受保护的文件名模式
PROTECTED_FILES=(".env" ".env.local" ".env.production" "credentials.json" "secrets.yaml" "secrets.json" "id_rsa" "id_ed25519")

# 受保护的扩展名
PROTECTED_EXTENSIONS=("pem" "key" "p12" "pfx")

# 受保护的目录
PROTECTED_DIRS=(".git/" ".ssh/" "node_modules/")

# 检查目录
for dir in "${PROTECTED_DIRS[@]}"; do
    if [[ "$FILE_PATH" == *"$dir"* ]]; then
        cat <<EOF
{
    "hookSpecificOutput": {
        "hookEventName": "PreToolUse",
        "permissionDecision": "deny",
        "permissionDecisionReason": "不允许修改受保护目录中的文件: $dir"
    }
}
EOF
        exit 2
    fi
done

# 检查文件名
for name in "${PROTECTED_FILES[@]}"; do
    if [[ "$FILENAME" == "$name" ]]; then
        cat <<EOF
{
    "hookSpecificOutput": {
        "hookEventName": "PreToolUse",
        "permissionDecision": "deny",
        "permissionDecisionReason": "不允许修改敏感文件: $name"
    }
}
EOF
        exit 2
    fi
done

# 检查扩展名
EXT="${FILENAME##*.}"
for ext in "${PROTECTED_EXTENSIONS[@]}"; do
    if [[ "$EXT" == "$ext" ]]; then
        cat <<EOF
{
    "hookSpecificOutput": {
        "hookEventName": "PreToolUse",
        "permissionDecision": "deny",
        "permissionDecisionReason": "不允许修改密钥类文件: *.$ext"
    }
}
EOF
        exit 2
    fi
done

echo '{"hookSpecificOutput":{"hookEventName":"PreToolUse","permissionDecision":"allow"}}'
exit 0
```

### 5.6.3 PostToolUse：全量操作审计

第三道防线是全量操作审计。通过配置matcher: "*"，该工具能够捕获并记录所有的工具调用。在合规性要求严格的企业环境中，这是不可或缺的机制，它清晰地回答了“Claude在什么时间对什么目标执行了什么操作”这一核心审计问题。请看以下代码示例。

```
#!/bin/bash
# .claude/hooks/audit-log.sh

INPUT=$(cat)
LOG_DIR="${CLAUDE_PROJECT_DIR:-.}/.claude/logs"
mkdir -p "$LOG_DIR"
LOG_FILE="$LOG_DIR/audit-$(date +%Y-%m-%d).log"

TIMESTAMP=$(date -Iseconds)
TOOL_NAME=$(echo "$INPUT" | jq -r '.tool_name // "unknown"')
TOOL_INPUT=$(echo "$INPUT" | jq -c '.tool_input // {}')

echo "[$TIMESTAMP] $TOOL_NAME: $TOOL_INPUT" >> "$LOG_FILE"
echo '{}'
exit 0
```

### 5.6.4 完整配置

将5.6.1小节到5.6.3小节的3个独立的脚本整合进.claude/settings.json，就形成了一套严密的纵深防御体系。具体配置如下。

```
{
  "hooks": {
    "PreToolUse": [
      {
        "matcher": "Bash",
        "hooks": [
          { "type": "command", "command": "./.claude/hooks/block-dangerous.sh" }
        ]
      },
      {
        "matcher": "Write|Edit",
        "hooks": [
          { "type": "command", "command": "./.claude/hooks/protect-files.sh" }
        ]
      }
    ],
    "PostToolUse": [
      {
        "matcher": "*",
        "hooks": [
          { "type": "command", "command": "./.claude/hooks/audit-log.sh" }
        ]
      }
    ]
  }
}
```

这套配置构建了一个涵盖“**事前拦截、事中防护、事后审计**”的完整安全闭环：危险命令拦截→敏感文件保护→全量操作审计。通过这套组合机制，我们将原本依赖“人工谨慎”的脆弱流程，升级为“代码即法律”的自动化安全体系。正如本章开篇案例所示，若应用第二道防护，小张的“悲剧”本可完全避免。

## 5.7 工程实战二：代码质量自动化

安全防护旨在“防患于未然”，而代码质量自动化则致力于“确保卓越交付”。本节的实战将深入展示PostToolUse与Stop Hook的协同工作机制。

### 5.7.1 PostToolUse：自动格式化

每次Claude写入文件后，系统将自动触发格式化工具。该Hook的精妙之处在于实现了“关注点分离”：Claude不需要感知项目具体采用何种格式化规范，只需要专注于代码逻辑编写，格式化过程将在后台无感完成。请看以下代码示例。

```
#!/bin/bash
# .claude/hooks/auto-format.sh

set -e
INPUT=$(cat)
FILE_PATH=$(echo "$INPUT" | jq -r '.tool_input.file_path // ""')

if [ -z "$FILE_PATH" ] || [ ! -f "$FILE_PATH" ]; then
    echo '{}'
    exit 0
fi

EXTENSION="${FILE_PATH##*.}"

case "$EXTENSION" in
    js|jsx|ts|tsx|json|md|css|scss|html)
        if command -v npx &> /dev/null; then
            npx prettier --write "$FILE_PATH" 2>/dev/null
            echo '{"hookSpecificOutput":{"additionalContext":"已用 Prettier 格式化"}}'
        fi
        ;;
    py)
        if command -v black &> /dev/null; then
            black "$FILE_PATH" 2>/dev/null
            echo '{"hookSpecificOutput":{"additionalContext":"已用 Black 格式化"}}'
        fi
        ;;
    go)
        if command -v gofmt &> /dev/null; then
            gofmt -w "$FILE_PATH" 2>/dev/null
            echo '{"hookSpecificOutput":{"additionalContext":"已用 gofmt 格式化"}}'
        fi
        ;;
    *)
        echo '{}'
        ;;
esac
exit 0
```

脚本中引入了`command -v`进行环境检查。如果检测到格式化工具未安装，Hook将静默跳过而非抛出错误。这体现了“优雅降级”的设计原则：Hook自身的异常不应阻塞核心工作流的正常运行。

### 5.7.2 PostToolUse：Lint反馈循环

自动格式化确保了代码的“美观”，而Lint检查则保障了代码的“正确”。本小节介绍的示例的核心在于利用additionalContext将Lint检查结果反馈给Claude，从而构建起“修改→检查→反馈→修复”的自动化闭环。请看以下代码示例。

```
#!/bin/bash
# .claude/hooks/lint-check.sh

set -e
INPUT=$(cat)
FILE_PATH=$(echo "$INPUT" | jq -r '.tool_input.file_path // ""')

if [[ "$FILE_PATH" == *.js || "$FILE_PATH" == *.ts || "$FILE_PATH" == *.jsx || "$FILE_PATH" == *.tsx ]]; then
    LINT_RESULT=$(npx eslint "$FILE_PATH" 2>&1) || true

    if [ $? -ne 0 ]; then
        ESCAPED=$(echo "$LINT_RESULT" | head -30 | jq -Rs '.')
        echo "{\"hookSpecificOutput\":{\"additionalContext\":
\"ESLint 发现问题: \n${ESCAPED}\"}}"
    else
        echo '{"hookSpecificOutput":{"additionalContext":"ESLint检查通过"}}'
    fi
else
    echo '{}'
fi
exit 0
```

当Claude完成JavaScript/TypeScript文件的修改后，如果触发Lint错误，脚本会将错误详情注入additionalContext。Claude读取到该上下文后，将自动分析并修复问题。整个流程不需要人工干预，实现了从代码生成到质量达标的全自动迭代。

### 5.7.3 Stop Hook：测试质量门控

Stop Hook是质量保证的一道防线：当Claude宣称任务完成时，自动触发测试套件。如果测试失败，系统将阻止会话结束并强制要求继续修复。请看以下代码示例。

```
#!/bin/bash
# .claude/hooks/run-tests.sh

INPUT=$(cat)

# 【关键机制】防止无限循环：检查stop_hook_active标志
# 如果该标志为true，说明已经是重试过一次，本次必须放行以避免死锁
STOP_ACTIVE=$(echo "$INPUT" | jq -r '.stop_hook_active // false')
if [ "$STOP_ACTIVE" = "true" ]; then
    exit 0 # 终止拦截，允许Claude停止
fi

# 切换到项目目录
if [ -n "$CLAUDE_PROJECT_DIR" ]; then
    cd "$CLAUDE_PROJECT_DIR"
fi

# 检测项目类型并运行测试
TEST_PASSED=true
TEST_RESULT=""

if [ -f "package.json" ] && grep -q '"test"' package.json; then
    TEST_RESULT=$(npm test 2>&1) || TEST_PASSED=false
elif [ -f "pyproject.toml" ] || [ -f "pytest.ini" ]; then
    TEST_RESULT=$(pytest 2>&1) || TEST_PASSED=false
elif [ -f "go.mod" ]; then
    TEST_RESULT=$(go test ./... 2>&1) || TEST_PASSED=false
else
    echo '{"hookSpecificOutput":{"additionalContext":"未检测到测试框架"}}'
    exit 0
fi

if [ "$TEST_PASSED" = true ]; then
    echo '{"hookSpecificOutput":{"additionalContext":"所有测试通过"}}'
else
    # 截取前50行错误日志并转义
    TEST_ESCAPED=$(echo "$TEST_RESULT" | head -50 | jq -Rs '.')
    # 返回block决策，强制Claude继续工作
    cat <<EOF
{
    "decision": "block",
    "reason": "测试失败，请修复后再停止",
    "hookSpecificOutput": {
        "additionalContext": $TEST_ESCAPED
    }
}
EOF
fi
exit 0
```

脚本中的`stop_hook_active`字段是防止系统陷入“死循环”的关键所在，其逻辑类似于递归函数的终止条件。当Stop Hook执行失败时，若系统尝试修复并再次触发该Hook，`stop_hook_active`将被置为`true`。随后，脚本检测到该标志位便会选择放行，从而退出循环——正如递归函数必须设定终止条件，Stop Hook也必须具备明确的退出机制。如果缺失这一检查，Claude将陷入“修复→测试失败→再修复”的死循环，无法自行脱困。

## 5.8 子智能体Hooks：精准的上下文管理

在第3章中，我们了解到子智能体通过隔离上下文来实现任务委派。Hooks系统为此提供了两种专属事件——SubagentStart和SubagentStop。然而，更为关键的是第三种机制是直接在子智能体的Frontmatter中定义Hooks。

### 5.8.1 全局与Frontmatter：精度问题

假设你拥有一个名为`db-reader`的子智能体，专门用于执行SQL查询。若需要审查其执行的每一条Bash命令以防范SQL注入风险，在全局settings.json中配置Hook并非最佳方案。因为全局配置会无差别地拦截所有Bash命令，涵盖编译代码、运行测试或安装依赖等与数据库无关的操作。这不仅浪费系统性能，还极易引发误拦截。

更优的解决方案是直接在子智能体的Frontmatter中定义Hook。请看以下代码示例。

```
---
name: db-reader
description: 只读数据库分析工具
tools: Read, Grep, Glob, Bash
hooks:
  PreToolUse:
    - matcher: "Bash"
      hooks:
        - type: command
          command: "./.claude/hooks/check-sql-injection.sh"
  Stop:
    - hooks:
        - type: prompt
          prompt: "检查查询结果是否包含PII (如姓名、邮箱、手机号)。若包含，请回复 ok: false 并要求进行脱敏处理。"
---

你是一名数据库分析专家。仅执行SELECT查询，严禁执行任何修改数据的SQL语句。
```

采用Frontmatter定义Hook的核心优势在于**生命周期的紧密绑定**：Hook随着子智能体的启动而激活，并在任务完成后自动清理。此外，配置与子智能体定义集成于同一文件中，可随md文件一同分发，用户不需要额外修改全局settings.json，极大地降低了配置复杂度。

### 5.8.2 SubagentStart：自动注入上下文

SubagentStart Hook的典型应用场景是在子智能体启动时动态注入团队规范。例如，每当启动code-reviewer子智能体时，系统可自动注入团队的编码标准。请看以下代码示例。

```
{
  "hooks": {
    "SubagentStart": [
      {
        "matcher": "code-reviewer",
        "hooks": [
          {
            "type": "command",
            "command": "echo '{\"hookSpecificOutput\":{\"hookEventName\":\"SubagentStart\",\"additionalContext\":\"团队编码规范：使用camelCase命名，行长上限100个字符，公共API必须包含JSDoc 注释\"}}'"
          }
        ]
      }
    ]
  }
}
```

### 5.8.3 SubagentStop：验证输出质量

SubagentStop Hook可用于验证子智能体的工作成果是否达标。通过结合agent_transcript_path读取子智能体的完整交互记录，系统能够执行细粒度的质量验收。请看以下代码示例。

```
#!/bin/bash
# verify-review-quality.sh

INPUT=$(cat)
AGENT_TYPE=$(echo "$INPUT" | jq -r '.agent_type')
STOP_ACTIVE=$(echo "$INPUT" | jq -r '.stop_hook_active')

# 仅验证code-reviewer子智能体
if [ "$AGENT_TYPE" != "code-reviewer" ]; then exit 0; fi

# 防止死循环(若当前已是Stop Hook触发阶段，则跳过)
if [ "$STOP_ACTIVE" = "true" ]; then exit 0; fi

TRANSCRIPT=$(echo "$INPUT" | jq -r '.agent_transcript_path')

if [ -f "$TRANSCRIPT" ]; then
    HAS_ISSUES=$(grep -c "issue\|问题\|bug" "$TRANSCRIPT" || true)
    HAS_SUGGESTIONS=$(grep -c "suggest\|建议\|recommend" "$TRANSCRIPT" || true)

    if [ "$HAS_ISSUES" -gt 0 ] && [ "$HAS_SUGGESTIONS" -eq 0 ]; then
        echo '{"decision":"block","reason":"发现问题但未提供修复建议，请补充每个问题的改进方案"}'
        exit 0
    fi
fi

exit 0
```

5.8.1小节到5.8.3小节介绍的这3层防护机制各司其职，构建了完整的质量保障闭环。

- `Frontmatter Hook`：负责内部自检，确保子智能体自问“我的输出是否完整？”。
- `SubagentStart Hook`：负责外部注入，在启动时赋予其必要的上下文（“给它必要的背景信息”）。
- `SubagentStop Hook`：负责外部验收，在结束时严格核查“它的工作成果是否达标？”。

## 5.9 异步Hooks：后台执行不阻塞

默认情况下，Hook脚本的执行是同步阻塞的——Claude会暂停当前工作流，直至脚本执行完毕并返回结果。对于快速的模式匹配检查，这几毫秒的延迟通常可以忽略不计；然而，对于运行测试套件、调用外部API或发送通知等耗时操作，同步阻塞会显著拖慢Claude的响应速度，影响用户体验。

为此，2026年年初发布的Claude Code引入了异步Hooks支持。

```
{
  "type": "command",
  "command": "./.claude/hooks/run-tests-background.sh",
  "async": true,
  "timeout": 300
}
```

配置`"async": true`后，Hook将在后台非阻塞运行。Claude不需要等待其完成即可立即继续后续工作。当异步Hooks执行完毕后，其输出结果将在下一个对话轮次中自动传递给Claude，供其参考或处理。

异步Hooks的两个关键限制决定了其适用边界。

- 类型限制：仅`command`类型的Hook支持异步执行。`prompt`和`agent`类型的Hook必须同步运行。
- 拦截能力限制：异步Hooks无法阻止当前操作。由于主流程在Hook启动的瞬间即已继续执行，异步Hooks失去了在操作发生前进行干预的时机。

因此，**异步Hooks适用于日志记录、异步通知、后台数据验证、非关键性质量审计等“事后处理”任务，不适用于需要实时阻断的安全检查（如SQL注入防御、敏感信息过滤）**。

## 5.10 环境变量与调试

### 5.10.1 Hooks可用的环境变量

表5-2列出了Hooks可用的环境变量，涵盖了变量的作用域及其核心用途。

**表5-2 Hooks可用的环境变量**

|环境变量|作用域|核心用途|
|---|---|---|
|CLAUDE_PROJECT_DIR|所有Hook|获取当前项目的根目录绝对路径|
|CLAUDE_SESSION_ID|所有Hook|当前会话的唯一标识符|
|CLAUDE_TOOL_NAME|所有Hook|触发当前Hook的工具名称|
|CLAUDE_FILE_PATH|所有Hook|当前操作涉及的文件绝对路径(若适用)|
|CLAUDE_ENV_FILE|仅SessionStart|环境变量持久化文件的路径|
|CLAUDE_NOTIFICATION|仅Notification|包含具体的通知消息内容|
|CLAUDE_CODE_REMOTE|所有Hook|布尔值标识，指示是否在远程Web环境中运行|
|CLAUDE_PLUGIN_ROOT|仅Plugin Hook|插件安装的根目录路径|

### 5.10.2 调试“三板斧”

调试Claude Hook脚本的3种核心方法如下。

**第一种，将调试信息输出至stderr。** 由于stdout专用于输出JSON决策结果，所有调试信息必须重定向至stderr。

```
echo "DEBUG: Checking file $FILE_PATH" >&2      # 调试信息
echo '{"decision": "allow"}'                    # JSON决策
```

**第二种，手动测试Hook脚本。** 通过构造模拟输入直接验证脚本逻辑。

```
echo '{"tool_name":"Bash","tool_input":{"command":"rm -rf /"}}' | ./.claude/hooks/block-dangerous.sh
echo "Exit code: $?"
```

**第三种，使用claude --debug查看完整的Hook执行细节。** 调试模式将显示匹配的Hook列表、各脚本的执行耗时和返回结果。

### 5.10.3 常见陷阱

在使用Hooks的过程中，有两个经常被忽视的问题。

问题一，若Shell配置文件（如~/.zshrc或~/.bashrc）中包含无条件的echo语句（如用于输出欢迎信息），这些输出会污染标准输出(stdout)，从而导致JSON解析失败。解决方法是使用`[[ $- == *i* ]]`条件判断将这些echo语句包裹起来，确保它们仅在交互式Shell中执行。

问题二，直接编辑settings.json后，Hook往往不会立即生效。这是因为Claude Code仅在启动时捕获配置状态，运行期间对文件的修改不会自动同步。若需生效，用户需要在/hooks菜单中确认变更，或重启当前会话。

## 5.11 工程设计方法论

面对具体的自动化需求，设计Hook方案需要明确以下3个核心维度。

- **拦截时机（事件选择）**：操作前拦截，选用PreToolUse或UserPromptSubmit；操作后反馈，选用PostToolUse；完成时检查，选用Stop或SubagentStop；生命周期管理，选用SessionStart或SessionEnd。
- **判断方式（类型选择）**：规则明确（如模式匹配、文件检查），选用command类型；需要语义判断但输入充分，选用prompt类型；需要深度代码分析，选用agent类型。
- **配置作用域（位置选择）**：团队通用规范，配置于.claude/settings.json；个人偏好设置，配置于~/.claude/settings.json；子智能体专属检查，配置于Frontmatter。

设计过程通常遵循“三步走”策略。

**第一步**，首先配置基于PostToolUse事件且匹配器为matcher:"*"的审计日志Hook，以此观察Claude的实际工具调用模式，并积累数日的真实运行数据。

**第二步**，基于审计数据识别高风险操作模式，进而设计针对性的PreToolUse拦截规则。

**第三步**，逐步收紧拦截规则，同时始终保留日志记录功能，确保在发生误拦截时能够快速定位问题根源。

**咖哥发言**
Hooks是团队级的基础设施，而非个人实验玩具。在.claude/settings.json中配置的Hook将对所有克隆该仓库的成员生效。若成员因不明原因被意外拦截，将严重阻碍工作流并引发挫败感。因此，务必遵循以下准则。

- 在提交Hook配置前，必须与团队充分讨论并达成共识。
- 每个拦截规则都必须附带清晰的原因说明，告知用户被拦截的具体原因。
- 利用审计日志实时监控Hook的触发频率，以便及时发现并修正误拦截情况。

## 本章小结

Hooks填补了CLAUDE.md和Skills无法覆盖的空白：它能在任务执行的关键节点，以系统级方式强制执行团队约定。作为Claude Code扩展机制中唯一运行于系统执行层而非认知层的组件，Hooks不依赖Claude的理解能力或Prompt的引导，而是通过脚本逻辑直接拦截工具调用。

**17种事件类型**覆盖了从会话启动到结束、从主对话到子智能体的完整流程。

**3种处理器类型**（包括command、prompt、agent）形成了“确定性递减、灵活性递增”的能力阶梯。选型原则为：首先选用command类型，其次是prompt类型，最后考虑agent类型。

支持在子智能体的Frontmatter中配置内联Hook，实现比全局配置更精准的管控。

引入**异步Hooks**，可以有效解决长时间操作导致的阻塞问题。

从工程方法论的角度审视，Hooks的设计严格遵循“先观测，后管控”的演进原则：首先利用PostToolUse事件记录审计日志，深入分析真实的工具调用模式与行为特征；然后基于积累的数据洞察，针对性地设计PreToolUse拦截规则，确保策略的准确性与必要性。内置stop_hook_active标志位，有效防止Hook逻辑自身触发无限重试调用。规定退出码2代表“有意阻止”，将其与系统错误明确区分，体现了系统在异常处理与状态定义上的深思熟虑。

Hooks的核心价值不在于“扩展能力边界”（让Claude做更多），而在于“夯实执行底座”（让Claude做的每一件事都更可靠）。它将那些“理应发生却常被遗忘”的关键检查（如敏感文件保护、代码格式化规范、测试验收流程）从依赖自觉的“软约束”，升级为不可绕过的系统级“硬约束”。开篇案例中提到的惨痛教训，仅需要一个简单的PreToolUse Hook即可从根本上避免。

至此，我们已深入剖析了Claude Code的内部扩展机制。在第6章中，我们将视野投向外部，探索它如何连接广阔的外部世界。

## 思考题

1. 在日常开发流程中，有哪些“本不该发生却会因粗心而偶发”的错误？请选取其中一个场景，设计一套完整的Hook方案：明确触发事件、Hook类型、Matcher配置规则，以及脚本需要校验的具体条件。
2. Stop Hook的“质量门控”模式与CI/CD流水线中的自动测试有何异同？在哪些场景下应优先采用Stop Hook，哪些场景更适合依赖CI/CD？
3. PreToolUse具备拦截能力，而PostToolUse仅用于反馈。这种不对称设计有何工程意义？若赋予PostToolUse拦截的能力（即撤销已执行的操作），会引发哪些问题？
4. 请对比全局settings.json与子智能体Frontmatter这两种Hook配置方式。在你的项目中，哪些Hook应置于全局配置，哪些更适合定义在特定子智能体的Frontmatter中？