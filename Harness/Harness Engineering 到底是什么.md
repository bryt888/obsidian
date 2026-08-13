【马克的技术工作坊的作品】https://v.douyin.com/HYN85yQJX_w/

 Harness Engineering 到底是什么
继 Prompt Engineering、Context Engineering 之后，AI 圈最近又冒出了一个新名词——Harness Engineering。本期将为你详解其定义、与前两者的关系、大厂实战案例、来源及是否为噱头的讨论。
 
一、基础概念铺垫
 
- Prompt Engineering：研究如何调整大模型提示词，让回答更贴合需求。例如给橘色小猫起名，仅说“帮我的猫起名”可能得到普通名字，而明确“帮我的橘色小猫起两个字、体现活泼爱玩性格的名字”，就能得到更贴切的“橘宝、橘豆”。
- Context Engineering：研究如何设计大模型接收的上下文信息，包括对话历史、工具列表、技能列表等，且需注意上下文有容量上限。方法有上下文压缩、动态检索外部资料、渐进式披露等。
 
二、Harness Engineering 定义
 
Harness 原意为“马具”，用来控制马匹。类比到 AI 领域，Harness 是用来控制大模型的系统，即 Harness = Agent - Model（Agent 减去大模型本身的部分）。Harness Engineering 则是构建与设计这套系统的技术，聚焦于如何围绕大模型搭建完整可靠的 Agent，涉及权限管控、工作流设计等。
 
三、与 Prompt、Context Engineering 的关系
 
- Prompt Engineering 关注“怎么问问题”，即如何组织提示词更准确。
- Context Engineering 关注“怎么给信息”，研究在合适时机提供包括 Prompt 在内的各类信息，范围比 Prompt Engineering 更广。
- Harness Engineering 关注“怎么搭系统”，研究如何围绕大模型搭建完整的 Agent 系统，范围最广。
 
四、大厂实战案例
 
- OpenAI 案例：用 AI 从零开发真实软件产品，5 个月写了近 100 万行代码。其 Harness 优化分为三部分：
- 上下文管理：将项目文档和决策整理到代码仓库，避免信息分散，且对文档进行分类管理，按需提供给 Agent。
- 验证与反馈：为 Codex 接入 Chrome DevTools 模拟用户操作，实现代码的即时验证与修复；接入可观测性工具栈，实现日志读取、错误捕获、链路追踪，且任务运行在隔离环境，结束后自动销毁。
- 技术债清理：定期扫描代码库和文档库，修正错误，保持代码和文档质量。
- Anthropic 案例：以克隆 claude.ai 为例，其 Harness 核心在任务规划和质量评估：
- 任务规划：通过 Planner 将用户模糊需求拆解为详细功能列表，Generator 按功能列表逐步实现。
- 质量评估：采用独立的 Evaluator Agent 进行评估，避免 Generator 自评的不客观。Planner 拆解需求，Generator 与 Evaluator 讨论交付标准后生成代码，Evaluator 评估后反馈问题，Generator 修改直至通过，如此循环完成所有功能。
 
五、Harness Engineering 的来源
 
Harness 一词原本用于指代支持某功能的一套框架，如 Test Harness（支持测试代码运行的框架）。后来 Mitchell Hashimoto 提出将其用于 Agent 系统，指当 Agent 出错时，设计解决方案使其不再犯同样错误，称为 Harness Engineering。OpenAI 发文后引发行业热议，Martin Fowler 等技术博主也参与讨论，逐渐形成共识：Harness 是 Agent 减去 Model 的部分，Harness Engineering 是构建这套系统的技术。
 
六、是否为噱头的讨论
 
- 观点一：不是噱头
它并非概念炒作，而是实实在在提升了 Agent 的稳定性、自动化程度和生产力，是当下将 AI 能力转化为实际生产力的关键技术。
- 观点二：不是终局
随着大模型能力的提升，当前用于约束、纠正模型的 Harness 设计可能会被模型自身吸收，逐渐淡出视野。它更像是过渡期的关键技术，是当下 AI 工程化的现实答案，谁能掌握它，谁就能更早将 AI 能力转化为真正的生产力。