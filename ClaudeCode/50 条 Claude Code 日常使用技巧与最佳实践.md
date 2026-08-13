


Vishwas
@CodevolutionWeb
·
3 月 19 日

你已经用 Claude Code 用得够久，知道它确实好用，现在你想挖出一切能带来优势的技巧。我整理了 50 条 Claude Code 的最佳实践和使用技巧，不管你才用了一个星期，还是已经深度使用了几个月，都会有帮助。内容来源包括 Anthropic 官方文档、Boris Cherny（打造它的人）、社区经验，以及我自己一年来的日常使用总结。



1. 设置 cc 别名



这是我每次开启 Claude Code 会话时的起手式。把下面这行加到你的 ~/.zshrc（或 ~/.bashrc）里：

bash
alias cc='claude --dangerously-skip-permissions'

运行 source ~/.zshrc 让它生效。这样你以后输入的是 cc 而不是 claude，并且会跳过所有权限确认提示。这个参数名故意起得很吓人。只有在你完全理解 Claude Code 能做什么、会对你的代码库做什么之后，才应该使用它。我在自定义 Claude Code里还讲了这个以及更多别名配置。



2. 用 ! 前缀直接内联运行 bash 命令



输入 !git status 或 !npm test，命令就会立刻执行。命令本身和输出结果都会进入上下文，因此 Claude 能看到结果并继续处理。比起先让 Claude 去运行命令，这样更快。



3. 按 Esc 停止 Claude。按两次 Esc 撤回一切



Esc 会在 Claude 执行到一半时中止它，但不会丢失上下文。你可以立刻改方向。

按两次 Esc（或输入 /rewind）会打开一个可滚动菜单，展示 Claude 创建过的每一个检查点。你可以恢复代码、对话，或两者一起恢复。直接说 “Undo that” 也可以。共有四种恢复选项：代码和对话一起恢复、仅恢复对话、仅恢复代码，或者从某个检查点开始向前总结。

这意味着你可以大胆尝试那些你只有 40% 把握的方法。成了就很好，不成就回退。零损失。唯一要注意的是：检查点只会跟踪文件编辑。通过 bash 命令产生的改动（例如迁移、数据库操作）不会被记录。

想从上次停下的地方继续，claude --continue 会恢复你最近的一次会话，而 claude --resume 会打开一个会话选择器。



4. 给 Claude 一个检查自己工作的办法



给 Claude 建立一个反馈闭环，让它能自己发现错误。你可以在提示词里加入测试命令、lint 检查或者期望输出。

markdown
Refactor the auth middleware to use JWT instead of session tokens. Run the existing test suite after making changes. Fix any failures before calling it done.

Claude 会运行测试、看到失败，然后自己修复，不需要你介入。Boris Cherny表示，仅这一点就能把质量提升 2 到 3 倍。对于 UI 修改，可以搭建 Playwright MCP server，这样 Claude 就能打开浏览器、与页面交互，并验证 UI 是否按预期工作。这个反馈闭环能抓住很多单元测试漏掉的问题。



5. 为你的语言安装代码智能插件



LSP 插件会在 Claude 每次编辑文件后自动提供诊断信息。类型错误、未使用的导入、缺失的返回类型，Claude 都能在你还没注意到之前看到并修复。这是你能安装的、影响力最高的插件。

选一个适合你的，然后运行安装命令：

bash
/plugin install typescript-lsp@claude-plugins-official /plugin install pyright-lsp@claude-plugins-official /plugin install rust-analyzer-lsp@claude-plugins-official /plugin install gopls-lsp@claude-plugins-official

也有适用于 C#、Java、Kotlin、Swift、PHP、Lua 和 C/C++ 的插件。运行 /plugin，进入 Discover 标签页，就能浏览完整列表。你需要先在系统里安装对应的 language server 可执行文件（如果缺失，插件会提示你）。



6. 使用 gh CLI，并教 Claude 学会任何 CLI 工具



gh CLI 可以处理 PR、issue 和评论，不需要额外的 MCP server。CLI 工具比 MCP server 更节省上下文，因为它们不会把工具 schema 加载进你的上下文窗口。jq、curl 和其他标准 CLI 工具也是一样。

对于 Claude 还不认识的工具，你可以这样说：“Use sentry-cli --help to learn about it, then use it to find the most recent error in production.” Claude 会读取帮助输出，搞清语法，然后执行命令。即便是一些小众的内部 CLI，也同样可行。



7. 给复杂推理加上 “ultrathink”



这是一个关键词，会把 effort 设为高，并在 Opus 4.6 上触发自适应推理。Claude 会根据问题动态分配思考量。适合用在架构决策、棘手调试、多步推理，或者任何你希望 Claude 先认真思考再行动的场景。

你也可以用 /effort 永久设置 effort。对于不太复杂的任务，较低的 effort 会更快、更便宜。要让 effort 和问题匹配。给变量重命名这种事，不值得消耗思考 token。



8. 用 skills 按需扩展知识



Skills 是一些 markdown 文件，可以按需扩展 Claude 的知识。不同于每次会话都会加载的 CLAUDE.md，skills 只会在当前任务相关时才加载。这样可以让上下文保持精简。

你可以在 .claude/skills/ 中创建 skills，或者安装打包了预构建 skills 的插件（运行 /plugin 查看可用项）。当 Claude 有时需要、但不是每次都需要某些专门领域知识（比如 API 约定、部署流程、编码模式）时，就很适合用 skills。



9. 用手机控制 Claude Code



运行 claude remote-control 启动一个会话，然后从 claude.ai/code 或 Claude 的 iOS/Android 应用连接它。会话实际运行在你的本地机器上。手机或浏览器只是一个查看窗口。你可以从任何地方发消息、批准工具调用、监控进度。

如果你用了技巧 #1 里的 cc 别名，Claude 已经拥有完整权限，不需要为每一个动作单独批准。这样远程控制会更顺畅：启动任务，离开电脑，等 Claude 完成或遇到异常时，再从手机查看。



10. 把上下文窗口扩展到 100 万 token



Sonnet 4.6 和 Opus 4.6 都支持 100 万 token 的上下文窗口。在 Max、Team 和 Enterprise 计划中，Opus 会自动升级到 100 万上下文。你也可以在会话中途用 /model opus[1m] 或 /model sonnet[1m] 切换模型。

如果你担心更大上下文下的质量，可以先从 50 万开始，再逐步增加。更大的上下文意味着在触发压缩前能容纳更多内容，但回复质量会随任务不同而波动。你可以用 CLAUDE_CODE_AUTO_COMPACT_WINDOW 控制何时触发压缩，用 CLAUDE_AUTOCOMPACT_PCT_OVERRIDE 设置百分比阈值。找到最适合你工作流的平衡点。



11. 不确定该怎么做时，使用 Plan Mode



对于多文件修改、不熟悉的代码和架构决策，使用 Plan Mode。它确实有额外开销（一开始会多花几分钟），但能防止 Claude 花 20 分钟自信满满地解决一个完全错误的问题。

对于范围小、目标清晰的任务就别用了。如果你一句话就能描述清楚 diff，那就直接做。你也可以随时按 Shift+Tab 在 Normal、Auto-Accept 和 Plan 三种权限模式间切换，而不用退出当前对话。



12. 在不相关的任务之间运行 /clear



一个带着明确提示词的干净会话，胜过一个混乱的三小时会话。换任务了？先 /clear。

我知道这感觉像是在丢掉进展，但重新开始通常效果更好。会话会逐渐退化，因为之前累积的上下文会淹没你当前的指令。花 5 秒执行 /clear，再写一个聚焦的起始提示词，能帮你省下 30 分钟越来越差的结果。



13. 不要替 Claude 解释 bug，直接贴原始数据



用文字描述 bug 很慢。你会看到 Claude 猜错、修正、再猜错，不断循环。

直接贴错误日志、CI 输出或者 Slack 线程，然后说一句 “fix”。Claude 可以读分布式系统的日志，追踪故障点。你的解释会增加一层抽象，反而经常丢掉 Claude 真正需要的细节。把原始数据给它，然后别挡路。

这对 CI 也一样有效。“Go fix the failing CI tests”，再附上 CI 输出，是最稳定的模式之一。你也可以贴一个 PR URL 或编号，让 Claude 去检查失败的 checks 并修复它。有了技巧 #6 里的 gh CLI，剩下的事 Claude 都能处理。

你还可以直接把终端输出通过管道送给它：

bash
cat error.log | claude "explain this error and suggest a fix" npm test 2>&1 | claude "fix the failing tests"



14. 用 /btw 提问快速的旁支问题



/btw 会弹出一个覆盖层，让你提一个快速问题，而不会把它写进你的对话历史。我会用它来问和当前会话有关的澄清问题，比如：“Why did you choose this approach?” 或 “What's the tradeoff with the other option?” 回答会显示在一个可关闭的覆盖层中，你的主上下文依然保持精简，而 Claude 会继续工作。



15. 用 --worktree 创建隔离的并行分支



claude --worktree feature-auth 会创建一个带新分支的隔离工作副本。Claude 会替你处理 git worktree 的设置和清理。

Claude Code 团队称这是最大的生产力提升之一。你可以同时开 3 到 5 个 worktree，每个都运行独立的 Claude 会话。我自己通常会同时跑 2 到 3 个。每个 worktree 都有自己的会话、自己的分支，以及独立的文件系统状态。

本地 worktree 的上限取决于你的机器。多个开发服务器、构建进程和 Claude 会话都会争抢 CPU。Builder.io 则把每个 agent 放到独立的云容器里，并带浏览器预览，这样你的本机就能专注处理那些真正需要你大脑的工作。



16. 用 Ctrl+S 暂存你的提示词



你正在写一个很长的提示词，写到一半突然想先问个简短问题。按 Ctrl+S 可以把当前草稿暂存起来。你先输入简短问题并提交，之后刚才暂存的提示词会自动恢复。



17. 用 Ctrl+B 把长时间运行的任务放到后台



当 Claude 启动一个耗时较长的 bash 命令（例如测试套件、构建、迁移）时，按 Ctrl+B 可以把它送到后台运行。Claude 会在进程运行期间继续工作，而你也可以继续聊天。等进程结束，结果就会显示出来。



18. 添加一个实时状态栏



状态栏本质上是一个 shell 脚本，会在 Claude 每轮回复后执行。它会在终端底部显示实时信息：当前目录、git 分支、上下文使用情况，并根据窗口填充程度做颜色标识。

最快的设置方式，是直接在 Claude Code 里输入 /statusline。它会问你想显示什么，然后帮你生成脚本。我在自定义 Claude Code里还写了完整配置和可直接复制粘贴的脚本。



19. 用 subagents 保持主上下文干净



“Use subagents to figure out how the payment flow handles failed transactions.” 这会启动一个独立的 Claude 实例，拥有自己的上下文窗口。它会读取所有文件、分析整个代码库，然后返回一份简洁总结。

你的主会话会保持干净，仍然有足够空间继续构建东西。一次深入调查，很可能在你真正开始写代码前就吃掉一半上下文窗口。subagents 能把这部分成本隔离在主会话之外。内置类型包括 Explore（Haiku，适合快速文件搜索）和 Plan（只读分析）。完整介绍可以看我们的subagents 与 agent teams 指南。



20. 用 agent teams 协调多会话并行工作



这是实验性功能，但很强大。先在设置或环境变量里加入 CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS 以启用。然后告诉 Claude 创建团队：“Create an agent team with 3 teammates to refactor these modules in parallel.” 一个 team lead 会把工作分发给多个 teammate；每个 teammate 都有自己的上下文窗口和共享任务列表。teammate 之间还可以直接互相发消息协调。

建议从 3 到 5 个 teammate 开始，每个 teammate 分配 5 到 6 个任务。避免分配会修改同一文件的任务。两个 teammate 同时编辑同一文件，最终很容易互相覆盖。你可以先从研究型和审查型任务开始（例如 PR review、bug 调查），再尝试并行实现。



21. 用指令引导 compaction



当上下文压缩时（自动触发或手动 /compact），告诉 Claude 要保留什么：/compact focus on the API changes and the list of modified files. 你也可以在 CLAUDE.md 中加入长期指令：“When compacting, preserve the full list of modified files and current test status.”



22. 用 /loop 做周期性检查



/loop 5m check if the deploy succeeded and report back 会安排一个后台循环提示，只要你的会话保持打开，它就会按周期触发。时间间隔可以省略（默认 10 分钟），支持 s、m、h 和 d 单位。你也可以循环执行其他命令：/loop 20m /review-pr 1234。这些任务与会话绑定，3 天后自动过期，所以即使忘了也不会无限运行。你可以用 /loop 监控部署、盯 CI 流水线，或者轮询外部服务，而自己去专注别的事情。



23. 用语音输入获得更丰富的提示词



运行 /voice 启用按住说话，然后按住空格开始口述。你的语音会被实时转写进提示词里，而且同一条消息里可以混合语音与打字。口述的提示词天然会包含更多上下文，因为你会更自然地解释背景、提到约束、描述需求，而不是为了省击键而省略信息。这个功能需要一个 Claude.ai 账号（不是 API key）。你还可以在 ~/.claude/keybindings.json 里把按住说话的按键改成像 meta+k 这样的组合键，从而跳过按住检测的预热时间。



24. 同一个问题纠正两次还没好，就重新开始



如果你和 Claude 在某个问题上不断来回纠正，但问题依然没修好，那当前上下文里已经塞满了失败路线，它们会主动伤害下一次尝试。这时执行 /clear，然后写一个更好的起始提示词，把你刚学到的信息融进去。一个更干净、提示更锐利的新会话，几乎总是胜过一个被死胡同拖累的长会话。



25. 明确告诉 Claude 应该看哪些文件



用 @ 可以直接引用文件：@src/auth
/middleware.ts 里有 session 处理逻辑。@ 前缀会自动解析成文件路径，所以 Claude 能准确知道该看哪里。

Claude 自己也能 grep 和搜索代码库，但它仍然需要先缩小候选范围，再识别正确文件。每一步搜索都会消耗 token 和上下文。从一开始就把正确文件指给 Claude，可以直接跳过整套过程。



26. 用模糊提示探索不熟悉的代码



“What would you improve in this file?” 是一个很好的探索型提示。并不是每个提示都必须非常具体。当你想让 Claude 以新鲜视角审视现有代码时，模糊一点的问题反而能让它提出你原本不会想到的问题。

我在初次接手一个陌生仓库时常这么做。Claude 会指出模式、不一致之处和改进机会，这些都是我第一次阅读时很容易忽略的。



27. 用 Ctrl+G 编辑计划



当 Claude 给出一份计划时，按 Ctrl+G 会在你的文本编辑器里打开它，方便你直接修改。你可以增加约束、删掉步骤、调整方向，而不需要在 Claude 写出一行代码后再返工。当计划大体正确，但你只想微调其中几步时，这特别有用。



28. 运行 /init，然后把结果砍掉一半



CLAUDE.md 是放在项目根目录下的一个 markdown 文件，用于给 Claude 提供持久指令：构建命令、编码规范、架构决策、仓库约定。Claude 每次会话开始时都会读取它。/init 会基于你的项目结构自动生成一个初始版本。它会识别构建命令、测试脚本和目录布局。

但生成结果往往过于臃肿。如果你不能解释某一行为什么存在，就删掉它。去掉噪音，补上真正缺失的部分。关于如何组织这些文件，可以看如何写出优秀的 CLAUDE.md 文件。



29. 判断每一行 CLAUDE.md 是否有用的试金石



对于你 CLAUDE.md 里的每一行，都问自己：如果没有这一行，Claude 会犯错吗？如果 Claude 本来就能正确做到，那这条指令就是噪音。每一条没必要的指令都会稀释那些真正重要的。大约只有 150 到 200 条指令预算，超过后遵循度就会下降，而系统提示本身已经占用了大约 50 条。



30. 当 Claude 犯错后，对它说：“更新你的 CLAUDE.md，别再犯这个错”



当 Claude 犯错时，对它说：“update the CLAUDE.md file so this doesn't happen again.” Claude 会自己把规则写进去。下一次会话，它就会自动遵守。

随着时间推移，你的 CLAUDE.md 会变成一份由真实错误驱动出来的活文档。为了避免它无限膨胀，可以使用 @imports（见技巧 #32）去引用一个单独文件，比如 @docs/solutions.md，把模式和修复方法放进去。这样你的 CLAUDE.md 仍然精简，而 Claude 会在需要时按需读取细节。



31. 把只在特定场景生效的规则放进 .claude/rules/



把 markdown 文件放进 .claude/rules/，按主题组织指令。默认情况下，每个 rule 文件都会在每次会话开始时加载。若想让某个规则只在 Claude 处理特定文件时加载，可以使用 paths frontmatter：

yaml
--- paths:   - "**/*.ts" --- # TypeScript conventions Prefer interfaces over types.

这样可以让主 CLAUDE.md 保持精简。TypeScript 规则会在 Claude 读取 .ts 文件时加载，Go 规则会在它读取 .go 文件时加载。Claude 不会在没碰到某种语言时，还去读这门语言的约定。



32. 用 @imports 让 CLAUDE.md 保持精简



用 @docs/git-instructions.md 引用文档。你也可以引用 @README.md、@package.json，甚至 @~/.claude/my-project-instructions.md。

Claude 会在需要的时候读取这些文件。可以把 @imports 理解为：“如果你需要，这里还有更多上下文”，而不会把 Claude 每次会话都要读的文件撑得过大。



33. 用 /permissions 把安全命令加入允许列表



别再为第一百次执行 npm run lint 而手动点击 “approve” 了。/permissions 允许你把可信命令加入允许列表，这样你就能保持在流中工作。对于不在列表里的内容，你仍然会收到提示。



34. 当你想让 Claude 自由工作时，使用 /sandbox



运行 /sandbox 启用操作系统级隔离。写入操作会被限制在你的项目目录内，网络请求也会被限制到你批准的域名上。在 macOS 上它使用 Seatbelt，在 Linux 上它使用 bubblewrap，因此这些限制会作用于 Claude 启动的每一个子进程。在 auto-allow 模式下，沙箱内的命令无需权限提示即可运行，这相当于给了你几乎完整的自主性，同时又保留防护栏。

对于无人值守的工作（例如过夜迁移、实验性重构），可以把 Claude 放进 Docker 容器里运行。容器能提供完整隔离、便捷回滚，以及让 Claude 连续运行数小时的安全感。



35. 为重复任务创建自定义 subagents



这和即时调用的 subagents（#19）不同。自定义 subagents 是预配置好的 agent，保存在 .claude/agents/ 中。例如，一个使用 Opus 且只读工具的 security-reviewer agent，或者一个为了速度而用 Haiku 的 quick-search agent。

运行 /agents 可以浏览和创建它们。你还可以设置隔离方式：对于需要自己文件系统的 agent，可以使用 worktree。



36. 为你的技术栈挑选合适的 MCP servers



最值得优先启用的 MCP servers 包括：用于浏览器测试和 UI 验证的 Playwright，用于直接查询 schema 的 PostgreSQL/MySQL，用于读取 bug 报告和线程上下文的 Slack，以及用于设计稿到代码工作流的 Figma。

Claude Code 支持动态工具加载，因此 server 只有在 Claude 确实需要时才会加载它们的定义。完整列表可以参考我们的2026 年最佳 MCP servers 指南。



37. 设置你的输出风格



运行 /config 并选择你偏好的风格。内置选项包括 Explanatory（详细、分步骤）、Concise（简短、以行动为导向）和 Technical（精确、术语友好）。

你也可以在 ~/.claude/output-styles/ 中创建自定义输出风格文件。



38. 用 CLAUDE.md 提建议，用 hooks 保证要求一定执行



CLAUDE.md 是建议性的。Claude 大约有 80% 的概率会遵守。hooks 是确定性的，100%。如果某件事必须每次都发生、绝不能漏掉（例如格式化、lint、安全检查），那就把它做成 hook。如果只是希望 Claude 纳入考虑的指导建议，用 CLAUDE.md 就可以。



39. 用 PostToolUse hook 自动格式化



每次 Claude 编辑文件后，你的格式化工具都应该自动运行。在项目里的 .claude/settings.json 中添加一个 PostToolUse hook，让 Prettier（或你的格式化器）在 Claude 编辑或写入任何文件后自动执行：

json
{   "hooks": {     "PostToolUse": [       {         "matcher": "Edit|Write",         "hooks": [           {             "type": "command",             "command": "npx prettier --write \\\\"$CLAUDE_FILE_PATH\\\\" 2>/dev/null || true"           }         ]       }     ]   } }

结尾的 || true 可以防止 hook 失败时阻塞 Claude。你还可以串联其他工具，比如再加一个 npx eslint --fix 作为第二个 hook 项。

如果你在编辑器中也打开了同一批文件，可以考虑在 Claude 工作时关闭 format-on-save。有开发者反馈，编辑器保存动作可能会让提示缓存失效，迫使 Claude 重新读取文件。这时候就让 hook 来负责格式化即可。



40. 用 PreToolUse hooks 阻止破坏性命令



通过给 Bash 配置一个 PreToolUse hook，可以拦截 rm -rf、drop table 和 truncate 之类的模式。Claude 连尝试都不会尝试。这个 hook 会在 Claude 执行工具前触发，因此破坏性命令会在造成伤害之前被拦下。

json
{   "hooks": {     "PreToolUse": [       {         "matcher": "Bash",         "type": "command",         "command": "if echo \\\\"$TOOL_INPUT\\\\" | grep -qE 'rm -rf|drop table|truncate'; then echo 'BLOCKED: destructive command' >&2; exit 2; fi"       }     ]   } }

把它加入项目中的 .claude/settings.json。你也可以通过 /hooks 交互式设置，或者直接对 Claude 说：“Add a PreToolUse hook that blocks rm -rf, drop table, and truncate commands.”



41. 用 hooks 在 compaction 后保留重要上下文



长会话中上下文被压缩后，Claude 可能会丢失当前工作线索。一个带 compact matcher 的 Notification hook，可以在每次压缩触发后，自动重新注入你的关键上下文。

你可以对 Claude 说：“Set up a Notification hook that after compaction reminds you of the current task, modified files, and any constraints.” Claude 会在你的设置里创建这个 hook。适合重新注入的信息包括：当前任务描述、已修改文件列表，以及任何硬约束（比如“不要修改 migration 文件”）。

这在持续数小时、你深陷某个功能开发时尤其有价值，因为那时你最不希望 Claude 丢掉线索。



42. 始终手动审查认证、支付和数据变更相关代码



Claude 很擅长写代码。但这些决策必须由人把关。认证流程、支付逻辑、数据变更、破坏性数据库操作，无论其他部分看起来多么靠谱，这些都必须人工审查。认证范围错了、支付 webhook 配错了，或者一个迁移悄悄删掉了某一列，都可能让你失去用户、金钱或信任。再完善的自动化测试，也抓不住所有这种问题。



43. 用 /branch 在不丢掉当前方案的前提下尝试另一条路



/branch（或 /fork）会在当前节点复制一份会话。你可以在分支里尝试风险较高的重构。成功了，就保留；失败了，原始会话也完全不受影响。这和回退（#3）不同，因为两条路径都会保留下来。



44. 当你无法完整写清功能规格时，让 Claude 来采访你



你知道自己想做什么，但感觉自己还没掌握足够细节，无法让 Claude 高质量地实现它。那就让 Claude 来提问。

markdown
I want to build [brief description]. Interview me in detail using the AskUserQuestion tool. Ask about technical implementation, edge cases, concerns, and tradeoffs. Don't ask obvious questions. Keep interviewing until we've covered everything, then write a complete spec to SPEC.md.

规格写完后，开一个全新的会话去执行，这样上下文更干净，而且你已经有了一份完整 spec。



45. 让一个 Claude 写，另一个 Claude 审



第一个 Claude 实现功能，第二个 Claude以 staff engineer 的新鲜视角来审查。审查者并不知道实现过程中做过哪些取巧，因此会对每一处都提出质疑。

这个思路也适用于 TDD。会话 A 写测试，会话 B 写代码去通过这些测试。



46. 用对话式方式审查 PR



不要让 Claude 只做一次性的 PR review（虽然你当然也可以这么做）。把 PR 打开到一个会话里，然后围绕它展开对话。“Walk me through the riskiest change in this PR.” “What would break if this runs concurrently?” “Is the error handling consistent with the rest of the codebase?”

对话式审查能发现更多问题，因为你可以持续深入到真正重要的区域。一次性 review 往往只会指出一些风格上的小问题，却容易漏掉架构层面的风险。



47. 给会话命名并用颜色区分



/rename auth-refactor 会在提示栏上加一个标签，这样你知道当前会话是什么。/color red 或 /color blue 可以设置提示栏颜色。可用颜色有：red、blue、green、yellow、purple、orange、pink、cyan。当你同时运行 2 到 3 个并行会话时，花 5 秒做命名和着色，能帮你避免把内容输入到错误的终端里。



48. 在 Claude 完成时播放提示音



添加一个 Stop hook，让 Claude 完成回复时播放系统音效。你可以启动一个任务，然后切去做别的事，等听到一声提示音再回来。

json
{   "hooks": {     "Stop": [       {         "matcher": "*",         "hooks": [           {             "type": "command",             "command": "/usr/bin/afplay /System/Library/Sounds/Glass.aiff"           }         ]       }     ]   } }



49. 用 claude -p 扇出执行批量任务



在非交互模式下，循环处理一组文件。--allowedTools 可以限制 Claude 针对每个文件允许使用哪些工具。再配合 & 并行执行，就能把吞吐量拉满。

bash
for file in $(cat files-to-migrate.txt); do   claude -p "Migrate $file from class components to hooks" \\\\     --allowedTools "Edit,Bash(git commit *)" & done wait

这非常适合做文件格式转换、跨代码库更新 import，或者执行那种各文件彼此独立的重复性迁移任务。



50. 自定义 spinner 动词（这个最好玩）



当 Claude 在思考时，终端会显示一个带动词的 spinner，比如 “Flibbertigibbeting...” 和 “Flummoxing...”。你可以把它们换成任何你喜欢的内容。直接告诉 Claude：

Replace my spinner verbs in user settings with these: Hallucinating responsibly, Pretending to think, Confidently guessing, Blaming the context window

你甚至不用自己列清单。只要告诉 Claude 你想要什么风格，比如：“Replace my spinner verbs with Harry Potter spells.” Claude 会帮你生成列表。这是个小细节，但确实能让等待过程更有趣。



总结

你不需要一下子掌握这 50 条。挑一个，选那个刚好能解决你上一次会话里最烦人的问题，明天就试试看。真正用得上的一个技巧，胜过你收藏却从不打开的五十个。

我经常写关于 Claude Code 的内容。也欢迎看看我的其他 Claude Code 指南。

想发布你自己的文章？
升级到 Premium
显示 22 条回复

Vishwas
@CodevolutionWeb
Builder.io 的 DevRel 负责人
@builderio
制作视频 https://youtube.com/codevolution 写作随想 https://vishwas.dev 🟩🟩🟩🟩🟩🟩🟩🟨⬜️⬜️ 100 万订阅者
