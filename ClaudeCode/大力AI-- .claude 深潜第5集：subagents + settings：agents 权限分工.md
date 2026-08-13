 https://v.douyin.com/6ZaKZiFdHZU/ 

这是一期关于 Claude 深潜的技术分享视频，主题是 “subagents + settings：agents 权限分工”，核心是讲解如何通过拆分角色和设置权限来优化 AI 项目。
 
视频首先指出在真实项目中，若一个 AI 既写方案又审方案还改代码查安全，会导致权限边界混乱。因此，本集聚焦于 subagents 和 settings.json，核心是拆分角色和权限。
 
接着对比了流程层（skill）和角色层（subagent）：skill 定义流程顺序、触发时机并带模板支持文件；subagent 定义角色身份、隔离上下文窗口和模型工具权限。skill 解决“流程怎么跑”，subagent 解决“谁来跑流程”。若角色不拆分，会出现边界互污染，写和审的角色混为一谈的问题。
 
然后说明何时使用 agent，比如需要固定只读的 reviewer、安全检查角色、方案写手时。并展示了两个实用角色示例：写手 agent（使用 sonnet 模型，允许 Read、Write 等工具，根据 brief 生成第一版方案，不做最终审批）和审核 agent（使用 haiku 模型，仅允许 Read 类工具，只审核完整性和风险，不允许改文件）。对比角色拆分前后，拆分后方案和审查分离、权限更小、review 结果更可信，而之前一个 AI 全干会导致自我打分、权限过大、易越界修改。同时建议从一写一审的双人组开始，不要一开始就建太多角色。
 
关于 settings.json，它并非只是偏好设置，而是权限收口工具。其设置重点包括 permissions.allow、permissions.deny、hooks 以及模型和审批相关默认值。角色拆分后，权限边界需同步拆分，它能收口工具权限、命令审批、危险目录，按用户级、项目级、本地级叠加，实现更可控的默认行为。settings 有三层作用域：个人默认（~/.claude/settings.json）、项目共享边界（.claude/settings*.json）、团队协作约束和本地例外临时调试。更稳的分层方式是用户级管默认模型和个人命令习惯，项目级管工具的允许/禁止和敏感目录，local 级管本机特例且不提交团队。判断设置层级的方法是先问是否跨项目偏好，再问是否团队共享边界，都不是则归为 local 例外，且不要将个人特例提交到团队项目中。
 
之后展示了如何将 settings 和角色结合，以审核 agent 为例，先在 agent 文件中限制其工具（仅 Read、Glob、Grep），再在项目 settings 中限制敏感写动作（deny Edit、Write，对敏感目录开启额外确认），这样角色提示词和 settings 两层一起收口，才能让“只读审核员”角色在工具层也成立。
 
最后强调，不要急于扩展角色，当流程跑顺、出现明显分工、权限边界需要更细时，才值得上角色层。更稳的升级顺序是先 rules，再 commands，再 skills，最后上 subagents + settings。多角色的价值在于边界清晰而非数量多，若流程、规则、settings 还不稳定，上多角色只会复制混乱。本集的核心收口点是：skill 解决流程怎么跑，subagent 解决谁来跑，settings 解决权限怎么收。建议先建一个能写的 proposal-writer 和一个只读的 reviewer，再在项目级 settings 中限制危险写动作，就能明显感受到边界的清晰。下期将做大结局，讲解 .claude 的全局层和记忆系统，对整套内容进行收口。