---
title: "Claude Code完全命令指南:50+命令深度拆解"
source: "https://www.toutiao.com/article/7626692171114250798/?app=news_article_lite&category_new=__all__&chn_id=-3&group_id=7626692171114250798&module_name=Android_lite_url&req_id=2026040922005473BBDCBA0F8BE4C7690E&req_id_new=2026040922005473BBDCBA0F8BE4C7690E&share_did=MS4wLjACAAAApHyOtM7YybzW-ng5b9JGdvCPEDXhfLsoxrmYjWQm3t3G_oVpnT7wbqI2VTRhnfMy&share_token=70e3b224-b8e8-4655-82da-fb6622aca014&timestamp=1775743255&tt_from=copy_link&upstream_biz=Android_lite_url&use_new_style=1&utm_campaign=client_share&utm_medium=toutiao_android&utm_source=copy_link&source=m_redirect"
author:
  - "[[优美可乐59]]"
published: 2026-04-09
created: 2026-04-09
description: "今天是开张第一篇,我想和你聊聊一个正在改变编程方式的工具——Claude Code。不是教你怎么安装,不是告诉你它有多厉害。我想从最实用的角度,把Claude Code的所有命令都讲清楚:它们是什么、为什么这样设计、什么时候用、怎么用最高效"
tags:
  - "clippings"
---
作品声明：内容取材于网络

![](https://p3-sign.toutiaoimg.com/tos-cn-i-axegupay5k/9b710eb69e654b96a460108d5c94c39e~tplv-tt-origin-web:gif.jpeg?_iz=58558&from=article.pc_detail&lk3s=953192f4&x-expires=1776349572&x-signature=CULB2URK8OmiYRAI1e3Kt2dxSTk%3D)

今天是开张第一篇,我想和你聊聊一个正在改变编程方式的工具—— **Claude Code** 。

不是教你怎么安装,不是告诉你它有多厉害。我想从最实用的角度,把Claude Code的 **所有命令** 都讲清楚:它们是什么、为什么这样设计、什么时候用、怎么用最高效。

这篇文章很长,但我保证:小白能懂,高手有料。建议收藏。

---

## 为什么要写这篇文章?

过去一个月,我把Claude Code的官方文档、GitHub Issues、Reddit讨论、开发者博客全翻了一遍。发现一个问题:

**大部分人在用Claude Code的时候,只会两件事:**

1. 打开终端,输入claude
2. 然后...用自然语言聊天

这当然没错,Claude Code本来就是这样设计的。但如果你只用自然语言,你其实只用了它 **10%的能力** 。

**真正让Claude Code变成编程超能力的,是命令系统。**

想象一下:

- 你可以用/init让AI瞬间理解一个10万行代码的项目
- 你可以用/compact让长达3小时的对话压缩成精华,省下90%的token
- 你可以用/plan模式让AI先规划再执行,避免"一顿操作猛如虎,回头一看全是错"
- 你甚至可以自己创建命令,把重复的工作流一键执行

这些,都藏在\*\*斜杠命令(Slash Commands)\*\*里。

今天,我们一起把它们全部拆清楚。

---

## 第一部分:理解Claude Code的命令哲学

在开始之前,我们先理解一个底层逻辑:

## Claude Code的三层命令架构

Claude Code不是一个AI聊天工具。它更像一个 **可编程的AI操作系统**,命令是操作这个系统的接口。

```ruby
第一层:CLI命令(启动时)
  └─ 控制Claude Code如何启动和运行
  └─ 例如:claude -c, claude --model opus

第二层:斜杠命令(会话中)
  └─ 控制AI的行为和上下文
  └─ 例如:/clear, /init, /model

第三层:自定义命令(你的工作流)
  └─ 把重复任务封装成命令
  └─ 例如:/review, /deploy, /test
```

**为什么要分三层?**

因为它们的作用范围不同:

- CLI命令决定" **这个会话的边界** "(用哪个模型、在哪个目录、权限模式)
- 斜杠命令控制" **对话的状态** "(清空历史、压缩上下文、切换模型)
- 自定义命令封装" **你的工作流程** "(代码审查、部署流程、测试策略)

明白了这个架构,你就理解了为什么有些事情用CLI做,有些事情用斜杠命令做。

---

## 第二部分:CLI命令全解析(启动层)

这是Claude Code的 **第一层命令**:你用来启动和配置会话的。

## 基础启动命令

## claude

最基础的启动方式,在当前目录开始会话。

**什么时候用:**  
当你在项目根目录,想快速开始工作的时候。

**实际场景:**

```bash
cd ~/projects/my-app
claude
```

---

## claude /path/to/project

指定目录启动。

**为什么需要这个?**  
因为有时候你人在A目录,但要处理B目录的项目。

**实际场景:**

```nginx
# 你在家目录,但要处理projects下的项目
claude ~/projects/my-app
```

---

## claude -c (continue)

继续上一次会话。

**这是最常用的命令之一。**

**为什么重要?**  
因为编程工作很少一次性完成。你可能:

- 上午写了一半代码,下午继续
- 昨天修了个bug,今天想接着优化
- 刚才测试发现问题,现在回来改

**实际场景:**

```nginx
# 早上做到一半,去开会
# 下午回来
claude -c
# 所有上下文都在,直接继续
```

**小技巧:**  
如果你有多个项目,-c会继续当前目录的最近会话。如果你想继续其他目录的会话,先cd到那个目录再-c。

---

## claude -n "session-name" (name)

给会话命名。

**什么时候用?**  
当你同时在做多个任务,需要在它们之间切换的时候。

**实际场景:**

```php
# 正在做A功能
claude -n "feature-payment"

# 切换去修B的bug
claude -n "bugfix-auth"

# 回到A功能
claude -r "feature-payment"  # r = resume
```

这就像浏览器的标签页,你可以为不同任务开不同的"标签"。

---

## claude -w branch-name (worktree)

在Git worktree中启动。

**这个功能太聪明了。**

**背景知识:**  
Git worktree让你在不同分支上同时工作,而不用来回切换分支。

**Claude Code的聪明之处:**  
它会自动在worktree里启动,这意味着:

- AI只能看到这个分支的代码
- 所有修改都隔离在这个分支
- 不会污染主分支

**实际场景:**

```coffeescript
# 在feature分支上工作
# Claude只能看到feature分支的代码
claude -w feature-new-api
```

---

## claude --model haiku / --model sonnet / --model opus

指定使用的模型。

**三个模型怎么选?**

| 模型 | 速度 | 成本 | 适用场景 |
| --- | --- | --- | --- |
| Haiku | 最快 | $1/百万token | 简单任务、批量操作 |
| Sonnet | 中等 | $3/百万token | 日常编码、代码审查 |
| Opus | 慢 | $5/百万token | 复杂架构、深度重构 |

**实战建议:**

```nginx
# 快速批量重命名文件
claude --model haiku "rename all test files to *.test.js"

# 日常编码(默认就是Sonnet)
claude

# 重构整个模块
claude --model opus "refactor the auth module"
```

**省钱技巧:**  
在会话中用/model haiku切换模型,只在需要深度推理的时候切回Opus。

---

## claude -p "prompt" (non-interactive)

非交互模式,执行完就退出。

**这是集成到脚本和CI/CD的关键。**

**实际场景1:代码审查**

```nginx
# 在Git hook里自动审查
git diff main | claude -p "review for security issues"
```

**实际场景2:自动化测试**

```nginx
# CI/CD里自动修复测试
claude -p "run tests and fix failures" \
  --allowedTools "Bash,Read,Edit" \
  --max-budget-usd 5.00
```

**实际场景3:日志分析**

```nginx
# 分析错误日志
cat error.log | claude -p "what caused this crash?"
```

---

## claude --permission-mode auto

自动模式:AI自己决定是否需要权限。

**危险但高效。**

**三种权限模式:**

- manual(默认):每个操作都要你确认
- plan:先给你计划,你批准后再执行
- auto:AI自己判断,危险操作会拦截

**什么时候用auto?**

- 你对项目很熟悉
- 任务相对安全(读代码、分析、建议)
- 你想快速迭代

**什么时候别用auto?**

- 陌生的代码库
- 涉及数据库操作
- 生产环境

---

## 高级启动选项

## claude --max-budget-usd 5.00

设置单次会话的花费上限。

**为什么需要这个?**  
防止AI跑飞了一直调用API,把你的钱烧光。

**实际场景:**

```nginx
# 让AI重构代码,但最多花5美元
claude -p "refactor the API layer" --max-budget-usd 5.00
```

---

## claude --max-turns 3

限制AI最多执行3轮操作。

**配合预算限制使用:**

```
# 修bug,最多3轮,最多花2美元
claude -p "fix the login bug" \
  --max-turns 3 \
  --max-budget-usd 2.00
```

---

## claude --output-format json

输出JSON格式,方便脚本处理。

**实际场景:自动化提取信息**

```nginx
# 提取所有TODO注释
claude -p "list all TODO comments" \
  --output-format json | jq '.todos'
```

---

## 第三部分:斜杠命令全解析(会话层)

这是Claude Code的 **第二层命令**:在会话中控制AI的行为。

## 必须掌握的核心命令

## /help

显示所有可用命令。

**这是你的命令字典。**

打开Claude Code,第一件事就是输入/help,看看有哪些命令可用。

**小技巧:**  
/help会显示:

- 内置命令(Claude Code自带)
- 当前项目的自定义命令
- 已连接MCP服务器的命令

---

## /clear

清空对话历史。

**什么时候用?**

- 任务完成,要开始新任务
- 对话跑偏了,想重新开始
- 上下文混乱了,AI答非所问

**注意:**  
清空的是 **对话历史**,你之前让AI写的代码、修改的文件都还在。

---

## /compact

压缩对话历史。

**这是长会话的救星。**

**背景:**  
Claude的上下文窗口虽然很大(100万token),但token是要钱的。一个3小时的会话可能累积几十万token。

**/compact做什么?**  
它把整个对话历史总结成精华,保留关键信息,删掉冗余内容。

**实际效果:**  
原本5万token的对话,压缩后可能只剩5000token,但关键信息都在。

**与/clear的区别:**

- /clear:完全删除,从零开始
- /compact:压缩总结,保留精华

**什么时候用?**

- 对话很长,但还没做完
- Token快用完了,但不想丢失上下文
- 对话有用,但太冗长

**高级用法:**

```bash
# 压缩时保留特定信息
/compact "keep the unresolved bugs"
```

---

## /init

分析项目并生成CLAUDE.md。

**这是Claude Code理解你项目的关键。**

**它做什么?**

1. 扫描整个项目结构
2. 识别技术栈(React? Django? Go?)
3. 找到关键文件(配置、入口、核心模块)
4. 生成一份CLAUDE.md文档

**CLAUDE.md是什么?**  
它是Claude对你项目的"记忆"。每次启动会话,Claude都会先读这个文件,了解:

- 这是什么项目
- 用什么技术栈
- 代码规范是什么
- 常用命令是什么

**实际场景:**

```bash
# 第一次在新项目使用Claude Code
cd ~/projects/new-app
claude
> /init
# Claude会分析项目,生成CLAUDE.md

# 之后每次启动
claude
# Claude自动读取CLAUDE.md,已经知道项目是什么了
```

**CLAUDE.md示例:**

```markdown
# Project: My App

## Tech Stack
- React + TypeScript
- Express API
- PostgreSQL database

## Key Commands
- \`npm run dev\` - Start dev server
- \`npm test\` - Run tests
- \`npm run lint\` - Check linting

## Code Conventions
- Use TypeScript strict mode
- Prefer functional components with hooks
- Write tests for all new features
```

**小技巧:**  
生成后,你可以手动编辑CLAUDE.md,添加AI需要知道但代码里没体现的信息(比如业务逻辑、设计决策)。

---

## /memory

编辑CLAUDE.md。

直接打开CLAUDE.md让你编辑,相当于:

```nginx
vim .claude/CLAUDE.md
```

---

## /model haiku / /model sonnet / /model opus

在会话中切换模型。

**省钱技巧:**

```shell
# 用Haiku做简单任务
/model haiku
> "rename these files"

# 切回Sonnet做复杂任务
/model sonnet
> "refactor this module"
```

---

## /cost

查看当前会话花了多少钱。

**实时成本监控。**

```shell
/cost
# 输出:
# Input tokens: 45,230 ($0.14)
# Output tokens: 12,450 ($1.87)
# Total: $2.01
```

---

## 工作流控制命令

## /plan

进入计划模式。

**什么是计划模式?**  
AI先给你计划,你批准后再执行。

**为什么需要这个?**  
想象你让AI重构一个模块,它可能要:

1. 读10个文件
2. 修改5个文件
3. 运行测试
4. 提交代码

如果你用默认模式,它每一步都要问你"我要读这个文件,可以吗?"

用/plan,它会先给你完整计划:

```markdown
我的计划:
1. 读取auth.js, user.js, db.js了解当前架构
2. 重构auth.js的login函数
3. 更新user.js的验证逻辑
4. 运行测试确保没问题
5. 提交更改

是否执行?(yes/no)
```

你只需要批准一次,它就按计划执行。

**什么时候用?**

- 复杂重构
- 不熟悉的代码库
- 多步骤任务

**高级用法:**

```ruby
# 直接在plan模式开始任务
/plan refactor the auth module
```

---

## /fast

切换快速模式。

**同样的模型,更快的输出。**

适合:

- 批量重复任务
- 简单代码生成
- 你需要快速迭代的时候

---

## 专业级命令

## /debug

内置的debug workflow。

**这是一个内置Skill,不是简单命令。**

**它做什么?**

1. 分析错误信息
2. 定位问题代码
3. 提出修复方案
4. 可选:直接修复

**实际场景:**

```bash
# 程序报错了
/debug
# 贴上错误信息
# Claude会分析并给出修复建议
```

---

## /review

代码审查。

**实际效果:**

- 检查代码质量
- 发现潜在bug
- 提出改进建议
- 安全问题检查

---

## /simplify

简化代码。

**适用场景:**

- 代码太复杂,可读性差
- 有重复代码
- 逻辑可以优化

---

## 第四部分:自定义命令(你的工作流)

这是Claude Code的 **第三层命令**:把你的工作流封装成命令。

## 为什么要自定义命令?

想象你每次代码审查都要这样:

```shell
claude
> "请审查这个PR,重点检查:
> 1. 安全问题(SQL注入、XSS)
> 2. 性能问题
> 3. 代码规范
> 4. 测试覆盖
> 输出格式:Markdown表格,包含位置、问题、建议"
```

每次都打这么多字,很累。

**自定义命令让你变成:**

```
/review
```

一个命令,完成整个工作流。

---

## 如何创建自定义命令?

**新方式:Skills(推荐)**

在项目根目录创建  
.claude/skills/review/SKILL.md:

```yaml
---
name: review
description: 代码安全和质量审查
allowed-tools: Read, Grep, Glob
model: claude-opus-4-6
---

以安全工程师视角审查代码库:

**高危:**
- SQL注入
- XSS攻击
- 命令注入
- 硬编码密钥

**中危:**
- 不安全的反序列化
- 过时依赖
- 敏感信息写入日志

**输出格式:**
| 位置 | 危险级别 | 问题描述 | 修复建议 |
|------|---------|---------|---------|
```

**之后每次:**

```
/review
```

就会用Opus模型,按这个Prompt执行完整的代码审查。

---

## 个人命令 vs 项目命令

**项目命令:**  
.claude/skills/ - 团队共享,进git

**个人命令:**  
~/.claude/skills/ - 只有你用,跨项目通用

**实际场景:**

项目命令(团队共享):

- /review - 代码审查标准
- /deploy - 部署流程
- /test - 测试策略

个人命令(你的习惯):

- /fix-imports - 修复你常见的import问题
- /my-style - 按你的风格重构代码

---

## Skills vs Commands

**旧方式:**  
.claude/commands/xxx.md - 已弃用但仍可用

**新方式:**  
  
.claude/skills/xxx/SKILL.md - 支持更多配置

**核心区别:**

| 特性 | Commands | Skills |
| --- | --- | --- |
| 组织方式 | 单个文件 | 文件夹(可包含多个文件) |
| 自动触发 | ❌ | ✅ |
| 配置项 | 少 | 丰富(model、tools、context) |
| 推荐度 | 旧项目 | 新项目 |

---

## 第五部分:MCP集成命令

**MCP = Model Context Protocol**

这是Claude Code最强大但最少人知道的功能。

## MCP是什么?

简单说:**让Claude Code连接外部服务** 。

比如:

- 连接GitHub,直接管理PR
- 连接数据库,查询和修改数据
- 连接Slack,发送通知
- 连接你的API,调用服务

---

## 如何使用MCP?

**第一步:连接MCP服务器**

在项目根目录创建.mcp.json:

```json
{
  "mcpServers": {
    "github": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-github"],
      "env": {
        "GITHUB_TOKEN": "your_token"
      }
    }
  }
}
```

**第二步:启动Claude Code**

它会自动连接GitHub MCP服务器。

**第三步:使用MCP命令**

```bash
/help
# 会看到新增的命令:
# /mcp__github__list_prs
# /mcp__github__create_pr
# /mcp__github__comment

/mcp__github__list_prs
# 列出所有PR

/mcp__github__create_pr "Fix auth bug"
# 直接创建PR
```

---

## 常用MCP服务器

**GitHub:**

```coffeescript
npm install -g @modelcontextprotocol/server-github
```

**PostgreSQL:**

```coffeescript
npm install -g @modelcontextprotocol/server-postgres
```

**Slack:**

```coffeescript
npm install -g @modelcontextprotocol/server-slack
```

---

## 第六部分:实战工作流

理论讲完了,现在看 **真实场景** 怎么用这些命令。

## 场景1:新项目第一天

```shell
# Day 1: 熟悉项目
cd ~/projects/new-codebase
claude

> /init
# Claude分析项目,生成CLAUDE.md

> "explain the architecture"
# 了解架构

> "what are the key modules?"
# 了解核心模块

> /memory
# 补充CLAUDE.md,添加业务逻辑说明
```

---

## 场景2:修复bug

```shell
claude -c  # 继续昨天的会话

> "I'm getting this error: [paste error]"

> /debug
# Claude分析错误

> /plan fix this bug
# 生成修复计划

# 批准计划

> /test
# 运行测试确认修复
```

---

## 场景3:大型重构

```shell
# 重构auth模块
claude -n "refactor-auth" --model opus

> /plan refactor the auth module to use JWT

# 审查计划

> yes

# 等待完成

> /review
# 审查改动

> /compact "keep the refactoring decisions"
# 压缩上下文,保留关键决策

> /cost
# 检查花费
```

---

## 场景4:代码审查(PR review)

```shell
git checkout pr-42
claude

> /review

# 生成审查报告

> "export the review as a markdown file"

# 复制到GitHub PR评论
```

---

## 场景5:快速原型

```nginx
# 用Haiku快速生成原型
claude --model haiku -p "create a React login form component"

# 生成后切换到Sonnet优化
claude -c
> /model sonnet
> "optimize this component for accessibility"
```

---

## 第七部分:高级技巧和最佳实践

## 技巧1:善用会话命名

```nginx
# 不同任务用不同会话
claude -n "feature-payment"
claude -n "bugfix-auth"  
claude -n "refactor-api"

# 随时切换
claude -r "feature-payment"
```

---

## 技巧2:定期compact

**规则:**  
每30分钟的长会话,执行一次/compact。

**为什么?**

- 省钱(压缩token)
- 保持对话清晰
- 避免上下文污染

---

## 技巧3:用cheap模型做cheap事情

```shell
# ❌ 浪费钱
claude --model opus "rename files"

# ✅ 省钱
claude --model haiku "rename files"

# ✅ 动态切换
claude
> /model haiku
> "batch rename test files"
> /model opus
> "design the architecture"
```

---

## 技巧4:创建任务模板

在.claude/skills/创建常用任务:

```
.claude/skills/
├── quick-fix/
│   └── SKILL.md (快速修复模板)
├── feature/
│   └── SKILL.md (新功能开发模板)
└── review/
    └── SKILL.md (代码审查模板)
```

---

## 技巧5:CLAUDE.md写什么?

**好的CLAUDE.md:**

```markdown
# Project: E-commerce API

## Quick Start
- \`npm run dev\` - Dev server (http://localhost:3000)
- \`npm test\` - Run tests
- \`npm run db:migrate\` - Database migrations

## Architecture
- Express.js REST API
- PostgreSQL database
- Redis for caching
- Stripe for payments

## Key Rules
- All endpoints require JWT auth (except /login)
- Always validate input with Joi
- Write tests for business logic
- Use async/await, not callbacks

## Current Focus
- Implementing payment webhook
- Known issue: Race condition in order creation
```

**不要写的:**

- 重复代码里已有的信息
- 过于详细的函数说明
- 太长(超过200行就该精简了)

---

## 第八部分:常见问题

## Q: /clear 和 /compact 到底什么区别?

**简单记:**

- /clear = 完全删除,适合任务切换
- /compact = 压缩保留,适合长会话

**类比:**

- /clear像清空浏览器历史
- /compact像给长文章写摘要

---

## Q: 为什么我的CLAUDE.md不生效?

**可能原因:**

1. 文件位置错了(应该在项目根目录)
2. 格式错误(确保是valid Markdown)
3. 太长了(超过5000行Claude可能忽略部分)

**解决:**

```nginx
# 检查Claude是否读取了
claude
> "what do you know about this project from CLAUDE.md?"
```

---

## Q: 自定义命令和自然语言,用哪个?

**规则:**

- 重复任务 → 自定义命令
- 一次性任务 → 自然语言

**例如:**

```shell
# ❌ 每次都打一遍
> "Please review this code for security issues, focus on SQL injection and XSS, output as markdown table..."

# ✅ 创建命令
/review
```

---

## Q: MCP值得用吗?

**看场景:**

**值得:**

- 你经常需要操作GitHub
- 你需要Claude直接查询数据库
- 你在做自动化流程

**不值得:**

- 你只用Claude写代码
- 你的任务很简单
- 你不想配置额外服务

---

## 总结:从命令到哲学

我们讲了50多个命令,但如果你只记住一件事,记住这个:

**Claude Code不是一个聊天机器人,它是一个可编程的AI操作系统。**

命令是你编程这个系统的方式。

当你掌握了命令,你不只是在"用"Claude Code,你是在 **指挥** 它,让它按你的方式工作。

**三个层次的进化:**

1. **初级:** 只用自然语言聊天
2. **中级:** 用斜杠命令控制AI行为
3. **高级:** 创建自定义命令,封装你的工作流

大部分人停在第1级,少数人到第2级,极少数人到第3级。

但第3级,才是Claude Code真正强大的地方。

---

如果这篇文章对你有帮助,欢迎转发分享。  
有问题,欢迎留言,我们一起探讨。

---

本文所有命令基于Claude Code v2.1.x版本。  
参考资料:Claude Code官方文档、GitHub社区、开发者博客。