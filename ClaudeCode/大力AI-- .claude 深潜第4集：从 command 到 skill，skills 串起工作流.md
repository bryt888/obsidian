 https://v.douyin.com/IJXWLr-sCjE/ 
 
上集把 commands 搭起来后，很多重复动作可一键触发。但做复杂任务时，步骤太碎，客户丢来需求，得先想跑哪条命令，再想下一条，流程繁琐。skills 要解决的是把这些离散动作接成一条工作流。
 
动作 vs 流程
 
- command 感：你手动决定何时触发，一次通常做一件事，更像固定入口。
- skill 感：Claude 根据场景判断，会串多个步骤，会带模板、规则和支持文件一起跑。
现在 Claude Code 官方文档中，custom commands 已被并进 skills 了。command 更像触发入口，skill 更像完整工作流定义。你手动触发一步是 command 感；Claude 读懂场景自己决定整套流程怎么走是 skill 感。
 
skill 解决的核心问题
 
1. 谁来判断时机：不再全靠手动点。
2. 多步怎么衔接：同一套流程顺着跑完。
3. 参考资料怎么带上：模板和 supporting files 一起进来。
 
skill 四层结构（不只是好的提示词正文）
 
1. 触发条件：什么时候该启动，比如用户说了什么、当前在哪个场景。
2. 工具边界：能做什么不能做什么，涉及 allowed-tools、模型选择。
3. 步骤顺序：流程怎么递进，先做什么、后做什么。
4. supporting files 的作用：模板、示例、参考资料不是装饰，决定了 skill 是否真的可复用。
 
skill 要点
 
包含 name、description、allowed-tools、model、触发条件、执行步骤、引用 supporting files。其价值在于把触发条件、工具边界、步骤顺序、支持文件都收进一个稳定入口。skill 稳不稳，关键不在字数，而在边界和支持材料有没有写清楚。可以把它理解成一个带说明书的小系统：什么时候启动、能用什么工具、该先做什么、遇到哪种情况要停。
 
为什么 skill 更稳
 
- 输入：包含触发条件、步骤顺序、支持文件。
- 处理：Claude 按固定流程读取和执行，不再每次靠临场猜。
- 输出：更稳定的输出、更少的跑偏、更容易复用。
 
仓库里的 skill 示例
 
有 tech-share（每日视频从写稿到预览的完整流程）、background-plate（关系图底图和概念板风格约束）、harness-engineering（课程系列的脚本、预览和渲染规矩）等。它们不是只一句“请帮我做某某”，而是把路径、文件命名、工作流约束、什么时候要停下来等全收进去了。
 
有无 skill 对比
 
- 只有 prompt：每次靠口头交代，流程边界容易丢，停在哪全看临场判断。
- 有 skill：路径和命名被固定，关键停顿点写进流程，Claude 更像按规程做事。skill 开始有工程味，它管理的是整条工作流怎么走，而非只管理语气。
 
构建 project-delivery skill 示例
 
- 目录构建：.claude/skills/project-delivery/ 下有 SKILL.md、template.md、pricing-guide.md。
- skill 最小项：触发条件为用户提供需求 brief 时；先读 template.md / pricing-guide.md；流程为分析需求→出方案→出排期报价；停止点为用户未确认前不进入下一步生成。
- 让它变稳的关键顺序：读 brief→读 supporting files→按步骤执行→在确认点停住。模板和参考资料很重要，别把它们当附件，它们决定了 skill 能不能长期复用。
 
command 和 skill 的选择判断
 
- 留在 command：动作单一、你手动触发更合理、不依赖 supporting files、不需要自己判断时机。
- 升级成 skill：多步流程、依赖模板或参考、有明确确认点、希望 Claude 自己判断并串起来。
判断方法：先看是不是多步，再看是否要带文件，最后看是否需要自动识别场景。skill 不是写得越多越好，而是该升级的时候再升级。
 
从上一集到这一集的演进
 
1. rules：先把规则按场景拆开。
2. commands：再把高频动作固定成入口。
3. skills：最后把离散动作接成工作流。
判断标准：需要自己盯住每一步的，留 command；想让它自己沿着流程跑的，升 skill。
 
最后建议，不用上来就写很大的 skill，可先挑一条已跑熟的 command，把它背后的三四步动作收成一份 skill。若在仓库里做视频流程，可复刻一份小一点的 project-delivery skill 练习。下期将讲多个 AI 角色怎么分工，以及 settings.json 怎么把权限边界真正管住。