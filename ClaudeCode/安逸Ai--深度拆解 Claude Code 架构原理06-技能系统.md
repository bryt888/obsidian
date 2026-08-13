 https://v.douyin.com/xpcwcxEtDws/

这是一期关于Claude Code架构原理中“的深度拆解视频，核心是解决agent在不同任务中专业知识需求不同，如何设计轻量灵活的知识按需加载机制。
 
技能系统要解决的问题
 
agent具备干活能力，但不同任务所需专业知识不同，无法将所有知识都塞进system prompt，否则会过于臃肿。
 
背景问题与解决方案
 
- 问题：
- 若把所有知识包都塞进system prompt，大部分Token会浪费在当前用不到的说明上；
- Prompt会越来越臃肿，主线规则看不清。
- 解决方案：将长期可选知识从system prompt中拆分出来，改成按需加载、动态挂载。
 
三个核心概念
 
- Skill：围绕某类任务的可复用说明书，告诉agent何时使用、任务步骤有哪些。
- Discovery：发现有哪些Skill可用，只需轻量信息，如名字和一句描述。
- Loading：把完整内容放进当前上下文，触发知识读取，这一步是“昂贵”的操作。
 
最小心智模型的两层架构
 
- 第一层（轻量目录）：让模型知道有哪些Skill可用。
- 第二层（按需正文）：模型真正需要时，通过工具调用注入完整内容。
- 优势：平时仅展示知识包清单，工作时才展开对应知识包。
 
代码层面的三个核心数据结构
 
- SkillManifest：轻量元信息，只有name和description，让模型知道Skill存在。
- SkillDocument：实际加载时的结构，包含SkillManifest和完整body。
- SkillRegistry：统一注册表，回答“有哪些Skill可用”和“某个Skill的完整内容是什么”。
 
最小实现五步法
 
1. 目录结构：每个Skill对应一个目录，目录下有skill MD文件。
2. 注册表单构建：从skill MD读取front matter原数据，构建SkillRegistry。
3. System Prompt配置：将Skill目录信息（非完整正文）放入system prompt。
4. 提供Load Tool工具：调用时返回完整Skill正文作为two resaleed。
5. 控制加载：确保只有调用load skill时，Skill正文才进入上下文。
 
skill、memory、CLAUDE.md的边界
 
- skill：某类任务才需要的做法或知识，按需加载、按需卸载。
- memory：需要长期记住的事实或偏好，跨会话记住。
- CLAUDE.md：更稳定的全局规则说明，长期稳定。
 
系统输入的两层结构
 
- 稳定层（始终存在）：包含身份说明、规则定义、工具列表、Skill目录。
- 按需层（动态注入）：当前加载的Skill正文。
 
初学者易犯的错误
 
1. 把Skill正文塞进system prompt，无法发挥Skill的独立优势。
2. Skill目录信息写得太弱，只有名字没有描述，模型不知何时加载。
3. 把Skill当成绝对规则，Skill是可选工作手册，非所有论词都必须用。
4. 混淆Skill和memory，Skill解决“怎么做”，memory解决“记住长期事实”。
5. 一上来就添加过多加载细节，应先讲清轻量发现、重内容按需加载。
 
核心结论
 
Skill系统的核心不是“多一个工具”，而是把可选知识从常驻prompt里拆出来，改成按需加载。