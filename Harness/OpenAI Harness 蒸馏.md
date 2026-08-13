
 
阅读需要 8 分钟 | 03-18
 
摘要
 
Harness skills 单次跑了 25 小时
 
OpenAI 2 月发了篇 Harness Engineering 文章，讲他们用 Codex 搭了一个让 Agent 持续工作的执行环境。我读完觉得里面很多东西是可以蒸馏成 Claude Code Skill 的，花了点时间提炼了四个： harness （持久执行）、 closed-loop-testing （闭环测试）、 architecture-guardrails （架构约束）、 harness-marathon （运行策略）。
 
用了几周，目前最长单次跑了 25 小时，任务一次通过率稳定在 80% 左右。
 
这篇文章讲蒸馏思路，不是 Skill 使用手册。
 
 
 
读这篇文章的时候我在想什么
 
OpenAI 那篇文章发布在 2 月 11 号，标题是 Harness engineering: leveraging Codex in an agent-first world，作者 Ryan Lopopolo。
 
核心数据：3 个正工程师，5 个月，100 万行代码，零手写，每人每天 3.5 个 PR。
 
说实话第一遍读完最大的感受不是“好厉害”，而是“这不就是我一直想做的那个东西吗”。
 
我之前让 Agent 跑长任务一直有几个固定痛点：Context 重置之后不知道上一轮干了什么；跑到一半觉得“差不多了”就停；遇到模糊的地方卡住等我确认。OpenAI 那篇文章把这些问题讲得很清楚，而且给出了他们的解法。
 
但他们的解法跟我的工具链不一样——他们是 Codex 云端 + 自研基础设施，我是本地。
 
所以问题变成了：怎么把他们文章里的方法论蒸馏成我能用的东西？
 
 
 
蒸馏的过程
 
我读文章的时候习惯把内容分三类：直接可用的机制、思路可以借鉴但实现要改的、他们能做但我现在不考虑的。
 
直接可用的机制
 
OpenAI 列了几个做法，实现逻辑非常清晰，可以照搬：
 
1. 进度文件即上下文
他们把  execution plans  和  decision logs  存在仓库里，Context 重置之后 Agent 读这些文件就能恢复状态。这个我完全认同——后来  harness  里的  harness-tasks.json  +  harness-progress.txt  就是这个思路，再加一个  git log ，三件事一起读，10 秒内恢复会话。
2. AGENTS.md 当目录，不当百科全书
他们试过“一个大 AGENTS.md”，失败了。总结的原因我都踩过：文件太大挤占有效 Context，规则写太多等于没规则，而且一旦过期 Agent 没法分辨哪些还是真的。现在他们的  AGENTS.md  只有 100 行，当导航地图用，真正的知识在  docs/  里。这个机制直接进了  harness  的知识架构设计。
3. 结构测试机械执行架构规则
文章里说他们用自定义 linter + 结构测试来强制执行分层约束，lint 错误信息本身就包含修复指引。这个设计很聪明——Agent 违规的时候报错，报错信息直接告诉它怎么改，不需要 Agent 再去找文档。这个成了  architecture-guardrails  的核心。
 
思路可以借鉴但实现要改的
 
1. 可观测性直接给 Agent 用
OpenAI 的做法是每个 git worktree 配一个独立的 Victoria Logs + Metrics + Traces，Agent 可以用 LogQL / PromQL / TraceQL 查。这个思路很好：让 Agent 能查日志和指标，“确保启动不超过 800ms”这种 Prompt 才能变成可执行的任务。但我的场景更偏业务流测试，不是性能调优。所以我没有给每个 worktree 装一个可观测性栈，而是换成了“证据包”的概念：每个业务流测试完必须产出  verdict.json  + 请求响应记录 + 数据库快照 + 过滤好的日志，判定来自业务不变式，不来自“脚本跑完了”。这成了  closed-loop-testing  的核心。
2. Entropy 管理
OpenAI 讲到 Agent 会复制仓库里的模式——好的坏的都复制，时间长了会有 drift。他们一开始每周花 20% 时间手清 “AI Slop”，后来换成 golden principles + 后台 Agent 定期扫描修复。思路我认同，但后台自动扫描我还没做。我的方案是把  golden principles  编成结构测试在 CI 里跑，加上  architecture-guardrails  里的 ratchet 策略：遗留违规加到 allowlist，新代码必须通过，慢慢蚕食存量问题。
 
他们能做但我不考虑的
 
1. 宽松合并门槛
文章里有一段让我停顿了一下：
The repository operates with minimal blocking merge gates. Test failures are often addressed with follow-up runs rather than blocking progress indefinitely.
 
这个在他们的场景（内部工具）可能合理，但我的场景有支付、订单这类业务，flaky test 阻塞合并是基本卫生，不是可以权衡的选项。 closed-loop-testing  里明确区分了哪些 check 是 blocking 的，这块我没有跟他们对齐。
2. Agent-to-Agent review
OpenAI 做到了 “humans may review pull requests, but aren’t required to”，review 主要在 Agent 之间完成。这个我也做到了，但实现方式不一样： harness  交给 Codex 负责开发和写 PR，Claude Code 做 review，两边互相 review。Codex 写代码质量高，Claude Code review 快，分上下米效果比单一 Agent 自我 review 好不少。
 
 
 
四个 Skill 是怎么来的
 
蒸馏完之后，我发现要解决的问题自然分成了几层：
 
- 让 Agent 能连续跑、断了能恢复——这是执行引擎问题： harness 。
- 让业务流测试有证据，判定来自不变式而不是脚本退出——这是验证问题： closed-loop-testing 。
- 让架构规则机械化执行，不依赖 Agent 自觉——这是约束问题： architecture-guardrails 。
- 让 Prompt 与 Agent 就不会停——这是运行策略问题： harness-marathon 。
 
最后那个  harness-marathon  是我加的，OpenAI 文章里没有对应的东西。它本质上是一套公标框架：Agent 停下来只有三个原因——工作耗尽了、遇到不确定的地方卡住了、Context 退化了。把这三个原因逐个干掉，Agent 就能跑很久。
 
跑了几周的情况：
 
- 我在  harness  里最长单次跑了 25 小时。这个不是我特意为了刷记录跑的，是一个比较大的 Go 服务重构任务，任务拆了 40 多个，跑充花了这么长时间。
- 任务一次通过率大概在 80%，剩下 20% 里，一半是需要介入的实现坑，另一半是我 Prompt 写得不够清楚导致 Agent 理解偏差。后者可以继续优化，前者的有些坑就该人来处理。
 
几个跑下来的真实感受：
 
- 进度文件救了我好几次。晚上跑着跑着 session 挂了很正常，第二天打开接着跑。 harness-progress.txt  里有完整的操作日志，Agent 读完就知道上次干到哪了。没有这个文件的时候，Agent 要花很长时间重新理解项目状态，还不一定理解对。
- 马折一定是 law 2 最容易被忽略。“Zero decision points”——指把所有外部依赖与讲 Prompt 说太简单，但会从 Agent 中还停了，问不是基本都是漏写了某个依赖、测试数据路径、API key、某个已知 bug 的处理方式。这种信息最浪费时间，也最容易遗忘。
-  bun.lock  检测比我预想的触发频率高。我在  harness  里加了一个机制：同一个文件如果被超过 6 次报警，超过 12 次提醒，每个文件报警基本就是 Agent 在绕圈，必须中断让人工介入。这个 OpenAI 文章里没提，是我自己加的，实测非常有用。
- Agent 在终局的时候当前的人干不动。这个 OpenAI 文章里也没提，我观察到的规律是： v1/v2/v3  会比那一版快很多，小测和 Hotfix 控得很死，但  v1/v2/v3  会比那一版慢很多，因为要做大量的重构和验证。所以我现在会在 Prompt 里明确告诉 Agent：v1/v2/v3 的成功率按 100% 算，v4 如果超过了 v1/v2/v3 的成功率，就不叫新版本，叫 bugfix。
 
 
 
摘要 + Claude Code review 的组合比我预期好
 
 harness  的质量存靠 Agent 的时间，自我  review  能在 60% 左右，加上  harness  的约束之后能到 80%，再加上 Claude Code review 之后能到 90%+。
 
 Codex  做开发， Claude Code  做 review，这个组合我之前在别的项目里试过，效果不错。这次把  harness  的约束加进去之后， Claude Code  的 review 质量不降反升——因为  harness  已经把大部分架构问题拦住了， Claude Code  只需要关注业务逻辑和细节。
 
 
 
最后
 
这篇文章讲的是怎么蒸馏别人的方法论成自己的工具，不是他们做了什么，而是我怎么把他们的东西变成我能用的。
 
核心数据：3 个正工程师，5 个月，100 万行代码，零手写，人均每天 3.5 个 PR。
 
场景不同，问题不同，所以实现不同。
 
有没有更简单的实现能解决同一个问题？不是所有机制都得是最复杂的。OpenAI 的  harness  是为了他们的场景做的，我的  harness  是为了我的场景做的。
 
每个工具干好自己的事，从一篇文章里能提出来的东西其实不多，但提出来的都是实用的。
 
Skill 的代码我先不放，后续会整理开源。有问题欢迎留言。
 
 
 
要不要我帮你把这篇文章里提到的 4 个核心 Skill 整理成一份可直接落地的实现清单？这样你可以快速对照着搭建自己的 Agent 长任务执行环境。