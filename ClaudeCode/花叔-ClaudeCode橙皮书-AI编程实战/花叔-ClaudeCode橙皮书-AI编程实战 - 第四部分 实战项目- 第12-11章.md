# 第四部分 实战项目


# 第12章 实战项目1：Chrome扩展

本章将从零开始，用Claude Code做一个真正能用、有人在用的Chrome扩展。你将经历从需求分析到打包安装的完整流程，体验最真实的踩坑和重构过程。

我最初做B站UP主运营助手插件，纯粹因为懒。每天需要回复几十条评论，每次打开B站后台，都要一条一条地点回复、打字、发送。两周之后，我实在受不了了，于是打开终端对Claude Code说：“帮我做一个自动回复B站评论的Chrome扩展。”

v1.0（最初的版本）大概2小时就运行起来了。虽然能用，但代码全堆在一个content.js文件中，共1211行，任何一次修改都像在拆炸弹。后来，我花了几天做v3.0重构，将content.js缩到175行，并把业务逻辑拆分为7个独立模块。在重构过程中学到的东西，远比做v1.0时学到的多。

下面我将带你走一遍完整的开发流程。目标是“可维护”，“仅能运行”远远不够。

## 12.1 为什么从Chrome扩展开始

学完前面的章节之后，Chrome扩展是一个很合适的进阶项目。

**技术门槛适中。**不需要学习Swift或者搭建后端。HTML、CSS和JavaScript是Claude Code最擅长的技术栈。虽然Manifest V3(MV3)的规则比Manifest V2(MV2)复杂，但Claude Code对MV3的理解非常到位，你只需要描述功能，它会自动处理Service Worker和权限声明等细节。

**有真实应用场景。**你几乎每天都在用浏览器，一个好的扩展能改善你的日常工作流。相比做一个永远不会打开第二次的“待办清单”，做一个自己天天用的工具，学习动力完全不同。

**反馈循环极短。**改完代码后，在Chrome扩展管理页面单击“重新加载”，刷新页面就能看到效果，不需要编译、部署和审核。这种即时反馈让你能快速迭代。

## 12.2 Phase 0：需求分析——你到底想让它做什么

我一开始犯的错误是直接说“做个自动回复插件”。结果Claude Code做了一个非常简单的版本：检测到新评论就回复“谢谢支持”。虽然能用，但完全不是我想要的。

后来我学会了先在Plan模式下把需求说清楚：

```
我想做一个B站UP主运营助手Chrome扩展，核心需求：
1. 自动扫描视频评论
2. 根据关键词规则匹配评论并自动回复
3. 支持全局规则和单视频规则
4. 不重复回复
5. 提供一个简单的管理面板，用于查看运行状态
先帮我分析这个需求，给出技术方案。
```

Claude Code会输出一份完整的技术方案，包括文件结构、技术选型、难点分析。这时不要急着让它开始做，应先仔细看方案，提出你的疑问和修改意见。

**核心建议** 在_Plan_模式下讨论需求所花的时间，远比事后推翻重做耗费的时间少。我做v1.0时没有经过讨论就直接做，只用了大约2小时，但随后又用了好几天做重构。如果一开始在_Plan_模式下花30分钟把架构梳理清楚，可以避免大部分重构工作。

## 12.3 Phase 1：项目初始化

确认方案后，切回正常模式，让Claude Code创建项目：

```
按我们讨论的方案，创建Chrome扩展项目。先做项目骨架：
- manifest.json(MV3)
- background.js(Service Worker)
- content.js(Content Script)
- popup.html + popup.js (管理面板)
- lib/目录(业务模块)
权限只申请必需的，不要多申请。
```

需要注意以下几个关键决策。

**MV3。**MV2已经被Chrome废弃，Claude Code会自动用MV3的写法。但要注意，网上很多Chrome扩展教程还是基于MV2的，如果你让Claude Code参考那些教程，可能会混用两套API。建议在CLAUDE.md中写清楚，必须使用MV3。

**Service Worker。**MV3最大的变化是background script变成了Service Worker，这意味着它随时可能被浏览器终止。因此，不能再用setInterval做定时任务，而要用chrome.alarms。Claude Code知道这个区别，但如果CLAUDE.md没有明确说明，它偶尔会用以前的命令。

**权限最小化。**只申请真正需要的权限。提交到Chrome Web Store审核时，权限越多越容易被拒，而且用户看到一长串权限请求也会犹豫。

Claude Code会生成一个完整的项目结构，其中manifest.json是Chrome扩展的灵魂文件，用于声明扩展名称、版本与权限，指定后台运行脚本、页面注入脚本，以及单击图标后的弹出界面等。

## 12.4 Phase 2：构建核心架构

开发Chrome扩展最容易踩的坑是“代码放错地方”。MV3有3个运行环境，职责完全不同。

|环 境|文 件|能做什么|不能做什么|
|:--|:--|:--|:--|
|Service Worker|background.js|定时任务、全局状态、消息路由|访问页面 DOM、读取页面 cookie|
|Content Script|content.js|读取页面 cookie、操作页面 DOM|直接调用 chrome.alarms 等 API|
|Popup/Options|popup.js/options.js|用户界面、配置管理|在用户关闭 popup 后运行|

我做v1.0的教训是把所有代码都塞进content.js：扫描逻辑、API调用、规则匹配、回复发送、状态管理等代码全在一个文件中，足足1211行。想要修改任何一个功能都要翻查半天。

v3.0的架构如下：

```
background.js (指挥中心)
├── 拥有定时扫描循环 (chrome.alarms, 每分钟一次)
├── 路由所有消息 (popup/content/widget, 统一处理)
└── 管理全局状态 (开关、限流等级等)
    ↕ chrome.runtime.sendMessage
content.js (前线哨兵)
├── 提取当前视频信息 (BV号、标题等)
├── 代理API调用 (因为需要页面cookie)
└── 转发日志和状态给浮窗
    ↕ 消息通信
lib/ (业务大脑)
├── db.js             ←统一数据层, 所有读写存储的操作都通过该层进行
├── scanner.js        ←评论扫描引擎
├── rules.js          ←规则匹配逻辑
├── api.js            ←B站API封装
├── ai.js             ←AI生成回复
└── rate-limiter.js   ←4级限流降级
```

关键设计原则是：**content.js只做它必须做的事。**哪些是它必须做的？读取页面cookie (background.js在MV3下读不到) 和操作页面DOM。除此之外，一切业务逻辑都在background.js和lib/中。

你在Claude Code中可以这样描述这个架构：

```
架构原则：
1. content.js是薄壳，只负责提取视频信息和代理API调用
2. background.js是指挥中心，拥有扫描循环和状态管理
3. 所有业务逻辑放在lib/目录下的独立模块中
4. 数据存储统一通过lib/db.js进行，不在其他文件里直接操作chrome.storage
5. 模块间通过消息通信，不共享状态
把这些写进项目的CLAUDE.md中。
```

**核心建议**

让Claude Code帮你写项目的CLAUDE.md，并把架构原则写进去。以后每次对话，Claude Code都会自动遵守。这就是第5章内容在实战中的应用。

## 12.5 Phase 3：逐个模块开发

在确定架构之后，开发过程就变成了逐个模块实现。我建议按以下顺序进行。

**1.数据层(db.js)**

这是地基，其他模块都依赖它。

```
先做lib/db.js，统一数据访问层。需要这些接口：
- getSystemState() / setSystemState(partial)
- getConfig() / setConfig(partial)
- getRules() / setRules(rules)
- getVideo(bvid) / setVideo(bvid, data)
- markReplied(bvid, rpid)
存储结构：
- 'sys:state'  →系统状态（开关、限流等级等）
- 'sys:config' →配置（扫描间隔、回复延迟等）
- 'sys:rules'  →规则（全局规则和视频专属规则）
- 'sys:logs'   →日志
- 'v:{bvid}'   → 每个视频的独立数据
所有写操作必须是原子的：先读，再合并，最后写入，防止并发覆盖。
```

从20多个散落的storage key收敛到4个核心命名空间key和一类视频级key，这个改变让后续所有模块都变得简单——没有“数据到底存在哪里”的困惑，全部由db.js处理。

**2.规则匹配(rules.js)**

这是最小的模块，只有62行代码。它的输入是一条评论和一组规则，输出是匹配结果。该模块采用纯函数设计，没有副作用，非常容易测试。

**3.API封装(api.js)**

该模块把B站的评论获取和回复发送封装成干净的函数。注意，B站的API需要页面cookie认证，因此这些函数实际上会在content.js的上下文中执行，通过消息传递结果。

**4.扫描引擎(scanner.js)**

该模块可以把前面3个模块串起来，核心逻辑是：获取评论$\rightarrow$过滤已回复的评论$\rightarrow$匹配规则$\rightarrow$生成回复$\rightarrow$发送。

这里有一个关键设计——依赖注入。scanner.js不直接调用API，而是接收外部传入的函数：

```
async function scanOneVideo(bvid, rules, config, {
    sendReplyFn,        // 发送回复的函数
    fetchCommentsFn,    // 获取评论的函数
    logFn               // 日志函数
}) {
    // 业务逻辑
}
```

为什么这样设计？因为scanner.js在background.js中运行，但API调用需要在content.js的上下文中执行，通过依赖注入，background.js可以传入“通过消息转发到content.js执行”的函数，而scanner.js完全不需要知道这个细节。

**5.限流器(rate-limiter.js)**

B站对评论回复频率有限制。如果你在1秒内发送5条回复，账号会被临时限制。我采用4级限流降级策略。

|等 级|回复间隔|扫描间隔|触发条件|
|:--|:--|:--|:--|
|normal|5 秒|2 秒|默认|
|slow|15 秒|5 秒|收到频率限制警告|
|slower|30 秒|10 秒|连续 3 次错误|
|paused|暂停|暂停|账号被限制，1 小时后恢复|

每次成功回复后，错误计数器会递减，自动恢复到normal。这就像弹簧一样，被压下去还能弹回来。

**6.背景脚本(background.js)**

把所有模块组装起来，实现扫描循环。每分钟被chrome.alarms唤醒一次，按优先级扫描视频（当前打开的标签页优先）。

**7.Content Script和UI**

最后设计界面。content.js只暴露几个简单方法给浮窗，popup.js负责实现管理面板。

## 12.6 Phase 4：调试技巧

Chrome扩展的调试和普通网页不太一样，下面分享几个实用技巧。

**Service Worker调试。**打开Chrome扩展管理页面，单击你的扩展下面的Service Worker链接，即可打开独立的DevTools。background.js中的console.log会出现在这里，而不是在网页的控制台中。

**Content Script调试。**在网页的DevTools控制台中可以直接看到content.js的日志。但要注意，content.js运行在一个隔离的执行环境中，不与网页的JavaScript共享全局变量。

**重新加载。**改完代码后，在Chrome扩展管理页面单击扩展的刷新按钮。popup和option页面需要关闭后重新打开。content.js改动需要刷新目标网页才能生效。Service Worker改动后，需要单击Service Worker旁边的“更新”按钮重载。

Claude Code可以帮你写调试辅助代码。比如，让它在content.js中添加一个**日志系统**，对所有关键操作进行记录，方便排查问题。在v3.0中，我做了一个简单的日志系统，最多保留200条记录，自动滚动清理，并可以在popup面板中查看。

## 12.7 Phase 5：安装和测试

开发阶段不需要将扩展发布到Chrome Web Store。在Chrome扩展管理页面打开“开发者模式”，单击“加载已解压的扩展程序”，然后选择你的项目目录即可。

测试清单（可以让Claude Code帮你生成）主要包括如下内容。

**基本功能**：添加规则$\rightarrow$扫描评论$\rightarrow$自动回复。

**去重**：同一条评论不会被回复两次。

**限流**：快速连续回复会自动降速。

**持久化**：关闭浏览器后重新打开，状态和数据都还在。

**多标签**：同时打开多个视频页面，各自独立运行。

## 12.8 从1211行到175行：一个典型的重构案例

重构过程值得单独讲解，因为它是Claude Code辅助重构的一个典型案例。

v1.0虽然能用，但痛点很多：content.js文件有1211行，API调用、扫描逻辑、规则匹配和UI更新等全混在一起，要修改一个功能得翻查好久才能找到相应代码；MV3的Service Worker可能随时休眠，用setInterval做定时任务很不稳定。

重构并不是我一开始就计划好的。某天，我想加一个新功能（AI辅助回复），却发现在1211行的content.js中根本无下从下手，于是决定重构。

我告诉Claude Code：

```
当前content.js有1211行，所有逻辑都在里面。我要重构成：
- background.js作为运行中枢
- content.js变成薄壳(只做视频信息提取和API代理)
- lib/目录存放所有业务模块
不要一步到位，分步骤来。先帮我分析当前代码，列出每一块逻辑应该去哪里。
```

Claude Code给出了一个详细的迁移计划，把1211行代码按功能拆分为7个独立模块和2个入口文件（content.js精简版、background.js），并标注了每块代码应该去哪个模块。然后我们一步一步执行，每迁移一块就测试一次，确保没有退化。最后的结果如下表所示。

|文 件|行 数|职 责|
|:--|:--|:--|
|background.js|468|消息路由、扫描编排|
|content.js|175|视频信息提取、消息桥接|
|lib/db.js|410|统一数据层|
|lib/scanner.js|322|评论扫描引擎|
|lib/api.js|137|B 站 API 封装|
|lib/ai.js|127|AI 回复生成|

（续）

|文 件|行 数|职 责|
|:--|:--|:--|
|lib/rate-limiter.js|100|限流降级|
|lib/rules.js|62|规则匹配|
|lib/migrate.js|253|MV2 到 MV3 的数据迁移|

重构后，content.js文件从1211行缩至175行，从“什么都做”变成了“只做必须做的事”。虽然项目的总代码量反而增加了（从1211行变为2054行），但每个文件的职责更加清晰，可以独立理解和修改。**重构的意义从来不在于减少代码量，而在于降低认知负担。**

**核心提醒**

重构时最容易犯的错误是“一步到位”。不要让Claude Code一次性重写所有代码，而应分步骤迁移，且每完成一步，都要测试功能是否正常。我在重构过程中，每迁移一个模块就在浏览器中过一遍完整的测试流程。慢一点没关系，稳比快更重要。

## 12.9 项目实践建议

如果你想跟着做一个Chrome扩展，不一定要做B站插件。这里提供几个参考方向。

**网页标注工具：**选中文字后高亮标注，并保存到本地。涉及Content Script操作DOM和storage持久化。

**页面翻译助手：**选中段落后调用AI翻译。涉及Content Script和外部API调用。

**社交媒体定时器：**记录你在各个网站中的停留时间，超时提醒。涉及chrome.alarms和多站点Content Script。

**GitHub增强：**在PR页面显示CI状态，并自动添加标签。涉及GitHub API和Content Script注入。

不管做哪个项目，核心流程都是一样的：在Plan模式下分析需求$\rightarrow$定义架构（谁管什么）$\rightarrow$写CLAUDE.md固化架构原则$\rightarrow$逐模块实现$\rightarrow$调试测试。

Chrome扩展项目是一个很好的起点，因为它证明了一件事：Claude Code的能力远不止于写网页。浏览器插件、自动化工具，甚至操作系统的脚本，只要用代码能解决，Claude Code都能帮你实现。接下来两章，我们将进一步拓展这个边界。


# 第13章 实战项目2：内容创作自动化

Claude Code最被低估的能力，不是写代码。 在本章中，我将展示自己每天真实的工作流——用Claude Code进行内容创作。 从选题、调研、写作、审校到发布，全流程自动化。 当你把Claude Code当作“通用智能体”来用时，它的价值会翻好几倍。

我从事自媒体创作，大概30%的时间用AI进行开发，70%的时间做内容：写公众号文章、做小红书图文、写视频脚本、做调研报告。 很长一段时间，我做开发用Claude Code，写文章用ChatGPT或Claude网页版。 直到有一天，我意识到一个问题：我在Claude Code中积累的所有偏好、规则和风格要求，换到网页版就全丢了。 每次开启新会话都要重新交待一遍：不要用破折号、用中文引号、搜索最近3个月的信息……

那天，我把写作规则写进了CLAUDE.md，然后加了几个skill，逐步搭建起一条完整的内容生产线。 现在回头看，这可能是我用Claude Code做过的最有价值的事情。 它不是某个具体的代码项目，而是每天都在帮我节省时间的内容自动化系统。

本章不讲理论，每一个实现我自己每天都在用。 你可以完全照搬，也可以按自己的需求修改。

## 13.1 CLAUDE.md路由系统

如果你只做一个项目，维护一个CLAUDE.md就够用。 但如果你需要应对多种工作场景，比如同时进行写作、开发、视频制作等，就需要一个路由系统。

所谓路由系统，是指根目录的CLAUDE.md(根CLAUDE.md)充当“指挥”，根据你说的话判断你要做什么，然后读取对应子目录的CLAUDE.md。

我的项目结构如下：

```
写作/
├── CLAUDE.md          <-路由器 (约8KB)
├── 01-公众号写作/
│   ├── CLAUDE.md      <-公众号写作的完整规则
│   └── 项目/
├── 02-小红书写作/
│   ├── CLAUDE.md      <-小红书写作规则
│   └── 项目/
├── 03-视频创作/
│   ├── CLAUDE.md      <-视频脚本规则
│   └── 项目/
├── 04-写作参考/
│   └── SHARED-RULES.md  <-跨工作区共享规则
├── .claude/skills/    <-27个skill
└── 10-橙皮书/           <-你正在看的这本书
```

根CLAUDE.md的核心是路由表：

```
## 工作区路由
收到任务后，先判断工作区，读取对应的CLAUDE.md：
| 关键词 | 工作区 | 读取文件 |
|--------|--------|----------|
| 写文章、商单、公众号 | 公众号写作 | /01-公众号写作/CLAUDE.md |
| 小红书、笔记、图文 | 小红书写作 | /02-小红书写作/CLAUDE.md |
| 视频脚本、视频创作 | 视频创作 | /03-视频创作/CLAUDE.md |
| 橙皮书、EPUB | 橙皮书 | /10-橙皮书/CLAUDE.md |
路由原则：任务模糊->询问确认；涉及多工作区->依次读取。
```

当我可以说“写一篇关于Claude Code的公众号文章”时，Claude Code会执行以下操作。

(1) 读取根CLAUDE.md，看到关键词“公众号”，自动匹配到“公众号写作”。

(2) 自动读取“/01-公众号写作/CLAUDE.md”，加载写作规则。

(3) 按照公众号的风格规范工作。

这样做的的好处是：根CLAUDE.md保持精简（约8KB），不会浪费上下文窗口；每个工作区的规则可以独立维护，互不干扰；新增工作区只需要加一行路由。

**核心建议**

如果你同时管理多个项目（工作项目、副业项目、学习笔记），可以创建一个统一的工作目录，由根CLAUDE.md负责路由，每个项目有各自的规则。 不必完全照搬我的项目结构，核心思想是：一个入口，多个出口。

## 13.2 共享规则，跨项目的一致性

有些规则不属于任何一个工作区，而是所有场景都需要遵守的，比如：

搜索信息要用最近3个月的来源

三遍审校流程（审内容 $\rightarrow$ 审风格 $\rightarrow$ 审细节）

配图流程（截图 $\rightarrow$ AI生成 $\rightarrow$ 上传图床）

文件命名规范

你可以将这类规则放在SHARED-RULES.md中，供所有工作区的CLAUDE.md引用。

最值得展开讲的是三遍审校流程，因为这个规则直接影响输出质量。

**第一遍：审内容。** 检查事实是否正确、逻辑是否严谨、结构是否完整。 这一遍不审查文字风格。

**第二遍：审风格。** 这一遍专门检测 AI 味（详见10.3节）。

**第三遍：审细节。** 比如，句子长度控制在15～25字、段落间留白、加粗标记（全篇约10处）、规范使用引号等。

将三遍审校流程写进SHARED-RULES.md后，不管我在哪个工作区写作，Claude Code都会按这个规则检查。 另外，我可以随时用`/proofreading`这个skill命令来触发审校，不需要每次手动描述流程。

## 13.3 创建你的第一个skill

第7章简单介绍过skill。 这里用一个真实例子展示怎么从零开始创建一个skill。

我创建的第一个skill就是上面提到的三遍审校流程。

创建方法如下：

```
# 在项目目录创建skill文件
mkdir -p .claude/skills/huashu-proofreading
# 写SKILL.md
```

SKILL.md的格式如下：

```
---
name: proofreading
description: |
  三遍审校流程，降低AI检测率到30%以下
  自动触发: 用户说“审校”“减轻AI味”“改一改”
  手动触发: /proofreading
---
# 三遍审校
## 第一遍：审内容
- 事实核查: 所有数据、日期、版本号是否正确
- 逻辑检查: 因果关系是否成立
- 结构检查: 是否有遗漏的要点
## 第二遍：审风格
检测AI味，逐一修改:
1.套话连篇->删除
2.AI句式->改为口语化
3.过度书面->换日常用词
4.结构机械->打乱，用叙事串联
5.态度中立->加个人立场
6.细节缺失->补具体数据
## 第三遍：审细节
- 句子长度: 15~25字为主
- 段落长度: 3~5句
- 加粗标记: 全篇约10处
- 破折号: 全篇最多2处
```

写好之后，输入`/proofreading`，Claude Code就会按这个流程审校当前文档。 你也可以直接说“帮我审校一下”，Claude Code看到关键词“审校”，会自动触发这个skill。

这里有一个重要的心智转变：**skill是写给AI的，不是写给你的。** 你不需要记住三遍审校的所有检查项，只需要记住`/proofreading`命令。 所有细节都被封装进skill中，Claude Code负责执行。

当你养成把重复工作封装成skill的习惯后，skill会迅速增加。 目前，我已经写了27个skill，覆盖内容创作的每一个环节。 以下是我常用的skill。

|类 别|操 作|作 用|
|:--|:--|:--|
|写作|`/proofreading`|三遍审校|
||`/article-edit`|文章编辑（带进度追踪）|
||`/topic-gen`|生成 3～4 个选题方向|
||`/article-to-x`|公众号文章改写为 X 推文|
|视频|`/video-outline`|视频脚本大纲（含标题策略）|
||`/script-polish`|脚本润色|
||`/danmaku-gen`|生成互动弹幕文案|
|调研|`/research`|结构化调研（自动存档）|
||`/info-search`|信息搜索（带来源验证）|
||`/material-search`|搜索个人素材库|
|出版|`/book-pdf`|“橙皮书”全流程构建|
||`/md-to-pdf`|Markdown 转 PDF|
|配图|`/image-upload`|AI 生图 + 上传图床|
||`/design`|信息图设计|
|集成|`/feishu`|飞书文档的创建与发送|

每个skill都经过实战打磨。 这些skill并不是我一次性写好的，而是在日常工作中，每遇到重复性操作就封装成一个skill，大概花了两个月积累的。

## 13.4 完整工作流演示，从选题到发布

介绍了这么多组件，下面我用一个完整的例子把它们串起来。 假设今天我要写一篇公众号文章，完整工作流如下。

**1. 10:00——选题**

我在终端打开Claude Code，进入写作项目目录，输入以下指令：

```
最近Claude Code更新了一些功能，帮我想几个公众号选题方向。
```

Claude Code看到关键词“公众号”，就自动路由到公众号工作区，加载对应的写作规则。 然后调用`/topic-gen`，输出3～4个选题方向，每个选题包含标题建议、大纲、工作量评估（从☆到☆☆☆）。

**2. 10:10——调研**

我选了第二个方向，工作量评估是☆☆。

```
选第二个方向。先做调研，搜集Claude Code最近一个月的重要更新和用户反馈。
```

这时，Claude Code调用`/research`，执行以下操作。

(1) 立刻创建一个调研文件：`_knowledge_base/research-claude-code-更新-20260403.md`。

(2) 每搜索一轮，就把结果追加写入文件（防止因对话中断丢失数据）。

(3) 搜索3轮后，输出阶段性总结。

(4) 给出结构化报告：关键事实、信息来源、空白点、写作建议。

这个“边搜边存”的设计是被“逼”出来的。 有一次，调研做到一半，对话因为上下文太长而被压缩了，之前搜到的信息全部丢失。 从那以后，我的每个调研skill都强制要求实时写入文件。

**3. 10:30——写草稿**

调研结束后，开始写文章。

```
调研差不多了。根据调研结果，写一篇公众号文章。
要求：
- 文章保存到“项目/2026.04-Claude-Code更新/草稿.md”
- 3000字左右
- 有明确的个人观点，不要写成新闻稿
```

Claude Code会创建项目目录，然后开始写作，写完后将文章保存到`.md`文件中。 注意，根CLAUDE.md中有一条偏好：“文章必须写入`.md`文件，不要直接写在回复里。” 这是因为，回复内容在对话结束后就被清空了，但`.md`文件一直在。

**4. 11:00——审校**

写完文章后，还需进行审校。

```
/proofreading 草稿.md
```

Claude Code收到指令后自动执行三遍审校：先读取文件，然后按照三遍审校流程逐项检查，最后输出修改建议。 待我确认后，它直接在文件中修改。 通常一次审校能发现10～20处AI味问题。

**5. 11:20——配图**

根据文章内容，生成配图。

```
给这篇文章配一张封面图。
```

这会触发配图流程。 Claude Code根据文章内容生成图片提示词，然后调用AI生图，完成将图片上传到图床，并把图片链接插入文章的对应位置。 整个过程自动完成，我只需最后审阅图片效果。

**6. 11:30——发布**

完成以上操作后，发布文章。

```
文章审校完了，发到飞书文档。
```

Claude Code调用`/feishu`，把`.md`文件转换成飞书文档格式，通过API创建文档并设置权限后发给我。 我在飞书中进行最后一轮人工检查，然后复制到公众号编辑器发布。

从选题到文档就绪，大概只需一个半小时。 如果没有这套系统，可能需要半天才能完成这些工作。

## 13.5 多智能体并行实践

上面的工作流是串行的：一步做完再做下一步。 但在实际工作中，很多环节可以并行。 比如，写这本书的时候，我同时打开了4个智能体。

```
# 终端1：调研智能体，查找最新的Claude Code更新
claude -p "调研Claude Code最近3个月的重大更新" --dangerously-skip-permissions
# 终端2：写作智能体，写第7章
claude -p "根据以下大纲写第7章" --dangerously-skip-permissions
# 终端3：审校智能体，审校已完成的第5章
claude -p "审校fragments/part5-claude-md.html" --dangerously-skip-permissions
# 终端4：配图智能体，为已审校的章节配图
claude -p "为第3章配图" --dangerously-skip-permissions
```

每个智能体独立运行，互不干扰。 4个智能体同时运行，整本书的构建效率提升了好几倍。 API成本大概是50美元，比雇一个助理便宜得多。

**注意**

并行的前提是任务互相独立。 如果智能体B需要智能体A的输出，就不能并行。 比如，“先调研再写作”必须串行，但“写第3章”和“写第5章”可以并行，因为它们操作不同的文件，不会发生冲突。

## 13.6 CLAUDE.md、skill、hook和MCP的配合

到这里，我们已经用到了CLAUDE.md（规则持久化）和skill（封装重复流程），再加上hook和MCP，就可以构成完整的harness。

**1. hook做什么**

hook是事件驱动的，比如：

每次创建新文件时，自动检查文件名是否符合命名规范；

每次编辑`.md`文件后，自动运行格式化（统一引号样式、删除多余空行）；

每次提交前，检查是否有未完成的TODO标记。

hook的价值在于“防遗忘”。 人可能会忘记检查命名规范，但hook不会。

**2. MCP做什么**

MCP让Claude Code能够连接和调用外部服务。 在我的内容创作工作流中，MCP有3个核心用途。

**连接飞书：**直接创建和编辑飞书文档。

**连接浏览器：**在Chrome中操作页面、读取信息。

**连接文件系统：**读取写作目录之外的参考资料。

**3. 四者的关系**

CLAUDE.md、skill、hook和MCP分别具有以下作用。

**CLAUDE.md：**定义规则和偏好（被动，每次对话自动加载）。

**skill：**封装工作流程（主动触发或关键词触发）。

**hook：**自动化检查（事件驱动，不需要人参与）。

**MCP：**连接外部世界（扩展Claude Code的能力边界）。

如果把Claude Code比作一名员工，那么CLAUDE.md是他的岗位手册，skill是他学会的技能，hook是他养成的好习惯，MCP是他能用的工具。 四者叠加，就是一个完整的“AI员工”系统。

## 13.7 搭建你的内容自动化系统

你不需要一步到位搭建起具有27个skill的系统，我建议你分3步走。

**第1周：创建CLAUDE.md。** 创建根CLAUDE.md，写入3～5条你最在意的规则，比如写作风格偏好、文件命名规范、常用的项目结构等。 不需要写很多，简单够用就行。

**第2周：创建第一个skill。** 观察一周，留意你重复最多的指令是什么，把它封装成一个skill。 可能是“帮我审校”“帮我翻译”或“帮我总结会议记录”……不管是什么，先创建一个skill，用起来。

**从第3周开始：按需扩展。** 每遇到一个重复操作，就问自己：“这个操作值得做成skill吗？” 如果一个操作你一周会执行3次以上，答案就是值得。 3个月后，你会有10～15个skill，覆盖日常工作中的大部分重复性操作。

搭建内容自动化系统的关键不是追求skill的数量，而是养成习惯。 当你习惯了用CLAUDE.md固化规则、用skill封装重复操作、用MCP连接外部服务时，你的工作效率将实现质的提升。 这就是第11章讲的harness工程在实际工作中的应用。

**写给非程序员读者**

本章的所有操作都不需要你会写代码：创建CLAUDE.md本质上是写规则文档，创建skill则是写操作手册，配置MCP只需填几个参数。 无论是做内容、做运营，还是做产品，任何需要重复性输出的工作，都可以直接复用这套方法。 Claude Code不只是编程工具，它是一个通用的AI工作站。 这可能是本书最重要的一个观点。


# 第14章 实战项目3：从零到App Store付费榜第一

本章讲的不是技术，而是一个完整的产品故事。从女朋友的一句话到App Store付费榜第一，中间只隔了一个周末。但1小时的开发不是重点，前面5分钟的判断、后面几个月的迭代，才是这个故事真正值得讲的部分。

2024年11月的一个下午，我在录制Cursor教程视频。女朋友坐在旁边帮我检查画面时突然说了一句：“与其做屏幕手电筒，不如做补光卡片呢。”

我当时有点儿蒙。“补光卡片”是什么需求？我之前从来没听过这个词。但我没有忽略这句话，而是打开小红书搜了一下。搜索结果让我愣了几秒。好几篇与“补光卡片”相关的笔记都获得了十几万个赞。有人用纯色图片当补光板自拍，有人分享不同颜色的补光效果对比，有人在找“能调颜色和亮度的补光App”，但一直找不到好用的。我立刻意识到：这是一个真实的高频需求。需求已经被验证了，只是没人做成产品。

5分钟后，我打开Cursor（当时我还没开始用Claude Code），用自然语言描述了我想要的功能。1小时后，“小猫补光灯”就在App Store上架了。

## 14.1 最重要的不是写代码，而是5分钟的判断

后来复盘这件事时，我发现最关键的不是开发过程，而是在那5分钟里做对了几个判断。

**第一，我没有忽略一个非程序员的建议。**女朋友不懂代码，但她懂用户需求。“补光卡片”是她从小红书上看到的，是真实用户的真实表达。如果我只跟程序员朋友讨论，应该不会想到这个方向。

**第二，我花了3分钟验证需求。**我没有凭直觉认为“这应该有人需要”，而是打开小红书搜了一下。获得十几万个赞的笔记就是最好的市场调研报告。成本：3分钟；回报：确认这是一个真实的高频需求。

**第三，我选择了最小可行产品。**第一版产品只聚焦核心功能：全屏显示一个纯色，支持调颜色和亮度。它没有滤镜、定时器、社交分享和会员系统，只是一个能调色的全屏灯。

这个判断过程和你用Claude Code做项目时的决策逻辑是相通的。技术能力不是瓶颈，Claude Code可以帮你写任何复杂度的代码。真正的瓶颈是：你要做的产品，真的有人需要吗？

**核心建议**

在阅读本章之前，先构思一款你想做的App。不要考虑技术上能不能实现，先思考谁会用、在什么场景下用、市面上有没有替代品。如果你在小红书、B站和抖音等平台上，发现很多人在讨论相关需求，但没有好产品能够满足，那就值得做。

## 14.2 用AI开发一个iOS App

“小猫补光灯”最初是用Cursor开发的。当时，Claude Code还没有现在这么成熟。如果今天重新做这个项目，我会全程用Claude Code。具体怎么做呢？

### 1.创建SwiftUI项目

先用Xcode创建一个空的iOS项目（这一步必须手动操作，因为Xcode项目的初始化需要通过图形界面选择配置）。创建完成后，在项目目录下启动Claude Code并发出如下指令：

```
这是一个SwiftUI项目，我要做一款补光灯App。核心功能：
1. 全屏纯色显示，用户可以选择颜色
2. 亮度可调（从0到最大）
3. 提供几个预设色卡（暖白、冷白、暖黄、粉色等）
4. 界面要简洁，打开就能用
先看看项目结构，然后帮我实现核心功能。
```

Claude Code会读取项目结构，然后开始写SwiftUI代码。补光灯的核心技术其实很简单，就是用Color做一个全屏的视图，再加上颜色选择器和亮度滑块。Claude Code很擅长执行这种“目标明确、技术路径清晰”的任务，通常第一版就能成功运行。

### 2.迭代是真正的工作

第一版能用，但远没到“好用”的程度。用户反馈会告诉你还需要做什么。

“能不能加个摄像头预览？这样我不用切来切去” $\rightarrow$ Pro版的核心功能

“颜色选择太麻烦了，能不能做成预设卡片” $\rightarrow$ 色卡系统

“我想要自定义颜色” $\rightarrow$ HSB颜色选择器

“亮度太低了” $\rightarrow$ 系统亮度API + 屏幕亮度叠加

每一次迭代都是同样的流程：用户反馈 $\rightarrow$ 在Claude Code中描述需求 $\rightarrow$ 实现 $\rightarrow$ 测试 $\rightarrow$ 提交。比如：

```
用户反馈需要补光的同时看到摄像头预览。
帮我加一个功能：屏幕上半部分显示摄像头实时画面，下半部分显示补光颜色，中间有一条可拖动的分界线。
```

Claude Code会修改现有代码，加入AVCaptureSession的摄像头集成。这是一个技术复杂度较高的功能（涉及相机权限、实时预览和画面布局），但Claude Code处理得很好，因为这些都是SwiftUI和AVFoundation的标准开发模式。

## 14.3 App Store上架

写代码可能只占整个开发过程的30%，剩下的70%是与上架相关的工作：注册开发者账号、进行证书配置、了解隐私政策、等待App审核。

### 1.开发者账号

要想在App Store中发布应用，需加入Apple Developer Program（年费约688元）。必须用真实身份注册，审核通常需要1～2天。

### 2.证书和签名

这是iOS开发中最“反直觉”的部分。你需要创建证书、配置描述文件、在Xcode中设置签名。你可以让Claude Code帮你梳理一下从开发到上架的流程：

```
我已经有了Apple开发者账号。帮我梳理一下从开发到上架的完整步骤，包括证书配置、描述文件创建、Xcode打包设置。
列出每一步的具体操作和可能遇到的坑。
```

Claude Code会给你一份详细的清单，告诉你每个选项应该怎么选、为什么这么选。但它不能帮你操作界面，你需要在Xcode与Apple Developer网站上手动完成操作。

### 3.App审核

Apple Store的审核比Chrome Web Store严格得多。常见的拒审原因包括：

缺少隐私政策页面（即使你的App不收集任何数据，也必须提供）

截图和实际功能不符

使用了私有API

功能太简单（可能被判定为“可以用网页实现的功能，没有必要做成App”）

“小猫补光灯”第一次提交就顺利通过审核，因为它的功能明确、界面简洁，没有灰色地带。审核大概用了24小时。

## 14.4 爆火：运气还是必然

App上架后的第一周，我在小红书发了一篇笔记，内容很简单：展示了补光前后的对比效果。

然后，它就“炸”了。

3天内，那篇笔记的阅读量达118万，获得7.3万个赞。“小猫补光灯”App的下载量突破3万次。“小猫补光灯Pro”（加了摄像头功能，售价6元）冲上了App Store付费榜第一名，并保持了一个多月。

后来有媒体问我是怎么做到的。说实话，我自己也有点儿意外。但复盘之后，我总结出以下几个关键点。

**需求强烈且明确。**自拍补光是一个高频、刚需的使用场景。它不是“有了更好”，而是“没有不行”。小红书上大量关于补光卡片的笔记证明了这个需求的真实性。

**产品足够简单。**打开App就是一个全屏色卡，不需要注册，也没有学习成本。这种“打开就能用”的产品最容易传播，因为用户可以在短视频中用3秒展示全部功能。

**争议带来流量。**很多程序员在评论区质疑：“这种App也能卖钱？”“1小时做出来的东西也配收费？”这些争议反而带来更多关注，让更多人看到了这篇笔记，知道了这款App。用户之间的讨论成了免费的流量引擎。

**快速迭代建立口碑。**App爆火的那几天，我连夜改了3个版本：修复用户反馈的问题、加入呼声最高的功能。同时，我刻意没有修改那篇小红书笔记，也没有发布新笔记，因为当时平台正在推荐那篇笔记，任何干预都可能影响分发效果。

**务实的期望管理**

绝大多数“手搓”产品会无声消亡。“小猫补光灯”的成功包含运气成分：选对了赛道、赶上了小红书的推荐、产品形态恰好适合短视频传播。这些因素不是每次都能复制的。做产品的正确心态是：做好每一个环节，但不预设结果。

## 14.5 后来的事：从爆款到产品线

“小猫补光灯”后来的发展超出了我的预期：Pro版持续更新，加入更多专业功能。基于同一个技术底座，我做了小程序版（覆盖没有iOS设备的用户）和鸿蒙版（覆盖华为用户），又做了“小猫相册”——一个面向同一用户群体（自拍爱好者）的新产品。

一个最初仅用1小时做出来的App，最后变成了一条产品线。这在传统开发模式下几乎不可能，光是写代码就要几个月。AI编程把开发成本压缩到接近零，让我可以快速试错和迭代。

《人民日报》把这种模式叫“手搓经济”。

## 14.6 iOS开发的CLAUDE.md建议

如果你打算进行iOS开发，建议在项目的CLAUDE.md中写入以下内容：

```
# iOS项目规则
## 技术栈
- SwiftUI (不用UIKit，除非SwiftUI无法实现)
- 最低支持iOS 16
- Swift Package Manager管理依赖
## 架构
- MVVM模式
- 每个View对应一个ViewModel
- 网络请求放在Service层
- 数据持久化用SwiftData或UserDefaults
## 代码风格
- 文件命名: 大驼峰(ContentView.swift)
- 变量命名: 小驼峰(isLoading)
- 每个文件不超过200行，超了就拆分
- 中文注释
## 常见坑
- 不要在View的body里做异步操作
- 使用@StateObject而不是@ObservedObject来持有ViewModel
- 真机调试需要先在Xcode中设置签名
```

这些规则可以帮助Claude Code生成更符合iOS最佳实践的代码，减少后续修改的工作量。

## 14.7 本章真正想讲什么

在这个故事里，技术是最不重要的部分。SwiftUI代码，Claude Code能写；上架流程，网上查得到；推广方法，每个平台都有教程。

真正决定成败的是3个非技术因素。

**你能不能发现需求。**“小猫补光灯”的需求来自女朋友的一句话。如果那天我戴着耳机没听到，就没有后面的一切。保持对日常生活的观察力，往往比任何编程技能更重要。

**你能不能克制**。专注于产品的核心功能，不盲目添加社交、会员和广告等不必要的功能。极致的简单往往就是极致的竞争力。我见过太多独立开发者把App做成了功能堆砌的“怪物”——每个功能都有，但没有一个做得好。

**你能不能坚持迭代**。爆火是偶然，持续提供价值是选择。在“小猫补光灯”之后，我又做了好几个版本，每个版本都在解决用户真实反馈的问题。这个过程不那么让人兴奋，但这才是产品成功的关键。

AI编程工具让“写代码”不再是瓶颈。这意味着竞争的维度从“谁能把代码写出来”变成了“谁能发现值得解决的问题”。Claude Code给了你实现任何想法的能力，但选择做什么、不做什么，永远是你自己的判断。


# 附录A 国内模型接入指南

如果你访问Anthropic API有困难，或者想尝试用国内模型驱动Claude Code，可以参考这份指南，了解主流平台的接入方法。

## A.1 接入原理

Claude Code支持通过环境变量将请求路由到第三方API。国内主流AI平台大多提供了Anthropic兼容接口（`/anthropic`端点），可以直接对接，不需要任何中转层。

核心环境变量：

```
# 厂商Anthropic兼容端点
export ANTHROPIC_BASE_URL="https://xxx/anthropic"
# 厂商API Key
export ANTHROPIC_AUTH_TOKEN="your-api-key"
# 默认模型
export ANTHROPIC_MODEL="model-name"
# Sonnet角色映射
export ANTHROPIC_DEFAULT_SONNET_MODEL="model-name"
# Haiku角色映射
export ANTHROPIC_DEFAULT_HAIKU_MODEL="model-name"
# Opus角色映射
export ANTHROPIC_DEFAULT_OPUS_MODEL="model-name"
```

你也可以将配置写进`.claude/settings.json`的`"env"`字段，效果一样，但不用每次手动重新输入。

## A.2 直连平台速查表

以下5个平台提供Anthropic兼容端点，可以零成本直连Claude Code。

|平 台|推荐模型|Anthropic端点|特 点|
|:--|:--|:--|:--|
|DeepSeek|DeepSeek-V4-Flash|api.deepseek.com/anthropic|价格长期处于低位，V4系列支持百万词元上下文|
|智谱|GLM-5|open.bigmodel.cn/api/anthropic|提供 Coding Plan 订阅，国内编程任务表现稳定|
|Kimi|Kimi K2.6|api.moonshot.cn/anthropic|支持长上下文，提供自动缓存折扣|
|MiniMax|MiniMax-M2.7|api.minimaxi.com/anthropic|提供 Token Plan 订阅，缓存命中成本较低|
|阿里云百炼|qwen3.5-plus|dashscope.aliyuncs.com/apps/anthropic|单一 API Key 可调用多家模型，相当于聚合平台|

## A.3 逐平台配置

**1.DeepSeek**

在DeepSeek开放平台注册并获取API Key，然后进行以下配置：

```
export ANTHROPIC_BASE_URL="https://api.deepseek.com/anthropic"
export ANTHROPIC_AUTH_TOKEN="your-deepseek-api-key"
export ANTHROPIC_MODEL="deepseek-chat"
export ANTHROPIC_DEFAULT_SONNET_MODEL="deepseek-chat"
export ANTHROPIC_DEFAULT_HAIKU_MODEL="deepseek-chat"
export ANTHROPIC_DEFAULT_OPUS_MODEL="deepseek-chat"
```

DeepSeek的Anthropic兼容端点会忽略部分高级字段（如`mcp_servers`、`metadata`），但日常编程场景完全够用。

**2.智谱**

在智谱开放平台获取API Key，然后进行以下配置：

```
export ANTHROPIC_BASE_URL="https://open.bigmodel.cn/api/anthropic"
export ANTHROPIC_AUTH_TOKEN="your-zhipu-api-key"
export ANTHROPIC_MODEL="GLM-5"
export ANTHROPIC_DEFAULT_SONNET_MODEL="GLM-5"
export ANTHROPIC_DEFAULT_HAIKU_MODEL="GLM-4.5-Air"
export ANTHROPIC_DEFAULT_OPUS_MODEL="GLM-5"
```

智谱提供了Coding Plan订阅（月费在主流厂商中处于低位）。最新的GLM-5是745B参数的旗舰模型，上下文窗口达20万个词元。

**3.Kimi**

在Kimi开放平台获取API Key，然后进行以下配置：

```
export ANTHROPIC_BASE_URL="https://api.moonshot.cn/anthropic"
export ANTHROPIC_AUTH_TOKEN="your-moonshot-api-key"
export ANTHROPIC_MODEL="kimi-k2.6"
export ANTHROPIC_DEFAULT_SONNET_MODEL="kimi-k2.6"
export ANTHROPIC_DEFAULT_HAIKU_MODEL="kimi-k2.6"
export ANTHROPIC_DEFAULT_OPUS_MODEL="kimi-k2.6"
```

Kimi K2.6的亮点是支持超长上下文，具有多模态能力，自动缓存机制可节省可观费用。

**4.MiniMax**

在MiniMax开放平台获取API Key，然后进行以下配置：

```
export ANTHROPIC_BASE_URL="https://api.minimaxi.com/anthropic"
export ANTHROPIC_AUTH_TOKEN="your-minimax-api-key"
export ANTHROPIC_MODEL="MiniMax-M2.7"
export ANTHROPIC_DEFAULT_SONNET_MODEL="MiniMax-M2.7"
export ANTHROPIC_DEFAULT_HAIKU_MODEL="MiniMax-M2.7"
export ANTHROPIC_DEFAULT_OPUS_MODEL="MiniMax-M2.7"
```

MiniMax-M2.7的缓存命中价格低至0.06美元/百万词元，性价比较高。

**5.阿里云百炼**

在百炼控制台获取API Key，然后进行以下配置：

```
export ANTHROPIC_BASE_URL="https://dashscope.aliyuncs.com/apps/anthropic"
export ANTHROPIC_API_KEY="your-dashscope-api-key"
export ANTHROPIC_MODEL="qwen3.5-plus"
export ANTHROPIC_DEFAULT_SONNET_MODEL="qwen3.5-plus"
export ANTHROPIC_DEFAULT_HAIKU_MODEL="qwen3.5-flash"
export ANTHROPIC_DEFAULT_OPUS_MODEL="qwen3-max"
```

阿里云百炼的独特优势是单一API Key可以调用多家模型（千问全系列、DeepSeek、Kimi、GLM），相当于一个模型聚合平台。

## A.4 需要中转的平台

火山引擎（豆包）和腾讯云（混元）目前只提供OpenAI兼容接口，需要通过LiteLLM进行协议转换：

```
# 安装LiteLLM
pip install 'litellm[proxy]'
# 创建config.yaml
model_list:
  - model_name: "doubao"
    litellm_params:
      model: "openai/your-endpoint-id"
      api_base: "https://ark.cn-beijing.volces.com/api/v3"
      api_key: "your-ark-api-key"
# 启动代理
litellm --config config.yaml --port 8000
# 配置Claude Code指向LiteLLM
export ANTHROPIC_BASE_URL="http://localhost:8000/anthropic"
```

社区也有开源的`claude-code-router`工具，支持多Provider配置和动态切换。

**注意**

**体验差异**：使用国内模型替换Claude后，Claude Code的部分高级功能（如复杂的多步推理、精确的工具调用序列）的表现可能不如原版Claude Code。建议先在简单任务上测试，满意后再用于正式项目。

**网络配置**：使用国内模型时，请确保终端的网络环境能正常访问国内API端点。

**API变动**：以上信息截至2026年4月。各平台的端点URL、模型名称和定价可能随时更新，使用前请查阅对应平台的官方文档。



# 附录B 命令速查

本附录主要介绍Claude Code的CLI命令、会话内斜杠命令、快捷键、环境变量和配置项（截至2026年4月）。

## B.1 CLI命令

### 1.终端中以claude开头的命令

|命 令|说 明|
|:--|:--|
|`claude`|启动交互式会话|
|`claude "query"`|带初始提示启动会话|
|`claude -p "query"`|非交互模式（print），输出后退出|
|`claude -c`|继续当前目录最近一次会话|
|`claude -r "name"`|按 ID 或名称恢复指定会话|
|`claude -w`|在隔离的 Git worktree 中启动|
|`claude update`|更新到最新版本|
|`claude auth login`|登录（可用参数 `--email`、`--sso`、`--console`）|
|`claude auth logout`|退出登录|
|`claude auth status`|查看认证状态|
|`claude agents`|列出所有配置的子智能体|
|`claude mcp`|管理 MCP 服务器|
|`claude plugin`|管理插件|
|`claude remote-control`|启动 Remote Control 服务器|
|`cat file \|claude -p`|

### 2.常用CLI Flag

|Flag|说 明|
|:--|:--|
|`--model`|指定模型（如 `sonnet`、`opus` 或完整模型名）|
|`--permission-mode`|权限模式：`default`、`acceptEdits`、`plan`、`auto`、`dontAsk`、`bypassPermissions`|
|`--add-dir`|添加额外工作目录|
|`--effort`|努力级别：`low`、`medium`、`high`、`max`（仅 Opus 4.6）|
|`--max-turns`|限制智能体轮次数（仅 print 模式）|
|`--max-budget-usd`|API 调用最大预算（仅 print 模式）|
|`--output-format`|输出格式：`text`、`json`、`stream-json`|
|`--mcp-config`|从 JSON 文件加载 MCP 服务器|
|`--bare`|最小模式：跳过 hooks/skills/plugins/MCP/CLAUDE.md|
|`--append-system-prompt`|在系统提示词末尾追加文本|
|`--verbose`|详细日志输出|
|`--debug`|调试模式，可过滤类别（如 "api,hooks"）|
|`--chrome`|启用 Chrome 浏览器集成|
|`--name`, `-n`|设置会话名称|
|`--allowedTools`|免提示权限的工具列表|
|`--disallowedTools`|完全禁用的工具列表|
|`--dangerously-skip-permissions`|跳过所有权限确认（仅限隔离环境）|

## B.2 会话内斜杠命令

### 1.导航与会话管理

|命 令|说 明|
|:--|:--|
|`/clear`|清空对话历史（别名 `/reset`、`/new`）|
|`/compact [command]`|压缩会话，可指定保留重点|
|`/context`|可视化当前上下文使用情况|
|`/cost`|显示词元的使用量统计|
|`/resume [session]`|恢复历史会话（别名 `/continue`）|
|`/rename [name]`|重命名当前会话|
|`/branch [name]`|从当前会话分出一个独立会话分支（别名 `/fork`）|
|`/rewind`|回退到之前的检查点（别名 `/checkpoint`）|
|`/diff`|交互式 diff 查看器|
|`/copy [N]`|复制最近的助手回复到剪贴板|
|`/export [filename]`|导出对话为纯文本|
|`/exit`|退出（别名 `/quit`）|

### 2.配置与模式

|命 令|说 明|
|:--|:--|
|`/model [model]`|选择或切换模型|
|`/fast [on\|off]`|
|`/effort [level]`|设置模型努力级别|
|`/plan [description]`|进入 Plan 模式（只讨论不执行）|
|`/vim`|切换 Vim/Normal 编辑模式|
|`/voice`|切换语音输入|
|`/config`|打开设置界面（别名 `/settings`）|
|`/permissions`|管理工具权限规则|
|`/sandbox`|切换沙盒模式|
|`/theme`|更换颜色主题|
|`/color [color]`|设置提示栏颜色|

### 3.扩展与集成

|命 令|说 明|
|:--|:--|
|`/memory`|编辑 CLAUDE.md 文件，管理 auto memory|
|`/init`|初始化项目 CLAUDE.md|
|`/skills`|列出所有可用 skill|
|`/hooks`|查看 hook 配置|
|`/mcp`|管理 MCP 服务器连接|
|`/plugin`|管理插件|
|`/agents`|管理智能体配置|
|`/ide`|管理 IDE 集成|
|`/chrome`|配置 Chrome 集成|
|`/add-dir <path>`|添加工作目录|

### 4.信息与诊断

|命 令|说 明|
|:--|:--|
|`/help`|显示帮助|
|`/status`|查看版本、模型、账号、连接状态|
|`/doctor`|诊断安装和配置问题|
|`/usage`|显示计划用量和限流状态|
|`/stats`|可视化使用统计|
|`/insights`|生成使用分析报告|
|`/release-notes`|查看更新日志|
|`/tasks`|列出后台任务|
|`/pr-comments [PR]`|获取 GitHub PR 评论|
|`/security-review`|分析当前分支安全漏洞|

### 5.特殊前缀

|前 缀|说 明|示 例|
|:--|:--|:--|
|`/`|命令 /skill 补全|`/proofreading`|
|`!`|直接执行 Bash 命令|`! npm run test`|
|`@`|文件路径提及或补全|`@src/index.ts`|

## B.3 快捷键

### 1.核心操作

|快 捷 键|说 明|
|:--|:--|
|`Ctrl+C`|取消当前输入或停止生成|
|`Ctrl+D`|退出会话|
|`Shift+Tab`|循环切换权限模式|
|`Alt+P`|切换模型|
|`Alt+T`|切换扩展思考|
|`Alt+O`|切换到快速模式|
|`Ctrl+R`|反向搜索命令历史|
|`Ctrl+B`|将运行中的任务放入后台|
|`Ctrl+T`|切换任务列表显示|
|`Ctrl+O`|切换详细输出（展开 MCP 调用详情）|
|`Ctrl+V/Cmd+V`|粘贴图片（从剪贴板）|
|`Ctrl+G`|在外部编辑器中打开当前输入|
|`Ctrl+L`|重绘屏幕|
|`Esc + Esc`|回滚或摘要|
|`Space（长按）`|语音输入（Push-to-talk）|

### 2.多行输入

|快 捷 键|说 明|
|:--|:--|
|`\ + Enter`|反斜杠换行|
|`Option + Enter`|macOS 默认换行方式|
|`Shift + Enter`|iTerm2、WezTerm、Kitty 等主流终端通用换行方式|
|`Ctrl + J`|控制序列，插入换行符|

### 3.文本编辑

|快 捷 键|说 明|
|:--|:--|
|`Ctrl+K`|删除到行尾|
|`Ctrl+U`|删除到行首|
|`Ctrl+Y`|粘贴已删除文本|
|`Alt+B`|光标后退一个词|
|`Alt+F`|光标前进一个词|
|`Up/Down`|导航命令历史|

## B.4 环境变量

|变 量|说 明|
|:--|:--|
|`ANTHROPIC_API_KEY`|Anthropic API 密钥|
|`ANTHROPIC_BASE_URL`|API 端点 URL（用于第三方模型）|
|`ANTHROPIC_AUTH_TOKEN`|认证凭证（部分第三方平台使用）|
|`ANTHROPIC_MODEL`|默认模型名称|
|`ANTHROPIC_DEFAULT_SONNET_MODEL`|Sonnet 角色映射模型|
|`ANTHROPIC_DEFAULT_HAIKU_MODEL`|Haiku 角色映射模型|
|`ANTHROPIC_DEFAULT_OPUS_MODEL`|Opus 角色映射模型|
|`CLAUDE_CODE_DISABLE_NONESSENTIAL_TRAFFIC`|禁用非必要网络请求（设为 1）|
|`API_TIMEOUT_MS`|API 调用超时时间（毫秒）|
|`NODE_EXTRA_CA_CERTS`|额外 CA 证书路径（企业网络）|
|`SSL_CERT_FILE`|SSL 证书文件路径|

## B.5 settings.json核心配置

配置文件位置：`~/.claude/settings.json`（全局）或`.claude/settings.json`（项目）。

|配 置 项|说 明|
|:--|:--|
|`hooks`|hook 配置（PreToolUse、PostToolUse、Stop 等）|
|`env`|环境变量覆盖（如 API Key、模型配置）|
|`permissions`|工具权限规则（allow、deny 列表）|
|`autoMode`|Auto 模式自定义分类器规则|
|`availableModels`|限制可选模型列表|
|`defaultShell`|默认 Shell（Bash 或 PowerShell）|
|`cleanupPeriodDays`|会话清理周期（默认为 30 天）|
|`claudeMdExcludes`|排除特定 CLAUDE.md 文件（glob 模式）|
|`attribution`|自定义 Git commit 和 PR 的署名|



# 附录C 斜杠命令深度指南

Claude Code支持大量以“/”开头的斜杠命令，功能覆盖会话管理、安全管控、效率提升、代码质量优化等各类场景。本附录将深入挖掘那些被低估的命令，每一个都可能改变你的工作方式。它们是Claude Code的“隐藏机关”。

我把最有价值的斜杠命令按使用场景分类，每个都讲清楚：它的作用是什么、什么时候用、怎么用才能发挥最大价值。

## C.1 上下文管理：最被低估的能力

Claude Code的上下文窗口是有限的（目前可支持100多万个词元）。看似容量很大，但一次持续几小时的会话，加上项目代码、对话历史和工具输出等，很容易逼近上限。上下文管理做得好不好，直接决定你能不能完成复杂任务。

### 1. /compact——智能压缩，不是简单删除

`/compact`是我用得最多的命令之一。它把对话历史压缩成一份精练的摘要，释放上下文空间。

更关键的是，它很智能。它不是把前面的对话直接删除，而是用AI总结：保留关键决策、代码变更记录和偏好设定，只丢弃冗余的中间过程。

什么时候用`/compact`命令？

**对话超过20轮。** 每轮对话都要带着完整的历史重新发送，对话越长，消耗词元越多。压缩对话可减少词元消耗量。

**切换任务方向。** 前半段在开发功能A，后半段要开发功能B，进行压缩，可删除功能A的细节。

**Claude Code响应变慢。** 上下文越长，响应越慢。压缩后可明显加快响应速度。

你也可以给`/compact`加指令，告诉它保留什么：

```
/compact 保留所有架构决策和错误修复记录，删除代码输出细节。
```

这样压缩后，关键信息还在，但不重要的中间过程被清理了。

**核心建议**

不要等到上下文满了才压缩。我的习惯是每完成一个阶段性目标就压缩一次，比如：需求分析完成 $\rightarrow$ 压缩 $\rightarrow$ 开始编码 $\rightarrow$ 编码完成 $\rightarrow$ 压缩 $\rightarrow$ 开始测试。这样每个阶段都有充足的上下文空间。

### 2. /context——看看你的词元花在哪儿

`/context`可以显示当前上下文的详细构成：系统提示词、CLAUDE.md、skill、对话历史等分别占多少。

这个命令的价值在于“可见性”。当你觉得上下文快满时，可以运行`/context`，看看到底是什么在消耗空间。也许是一个特别大的CLAUDE.md，也许是加载了太多MCP工具，也许是对话历史太长。只有看清数据，才能做出正确决策。

### 3. /clear——干干净净地重新开始

`/clear`可以清空对话历史，将会话恢复到初始状态。这和关闭终端重新启动会话的效果一样，但更快。

我的建议是：**每次要开始不相关的新任务时，先运行`/clear`。** 很多人在同一个会话中开发完功能A，接着开发功能B，结果功能A的上下文一直占着空间。`/clear`是免费的，但省下来的词元是实打实的。

## C.2 安全机制：犯错也不怕

### 1. /rewind——外科手术级的回滚

按两次Esc键或者输入`/rewind`，即可进入回滚模式。你可以进行3种操作。

**回滚对话**：撤销最后几轮对话，从某个节点重新开始。

**回滚文件**：保留对话，但把文件恢复到之前的状态。

**两者都回滚**：同时恢复对话和文件。

回滚文件但保留对话是最妙的操作。假设Claude Code改了10个文件，完成后你发现方向不对，可以选择回滚文件。这样，文件会恢复为原样，但讨论的内容还在。你可以让Claude Code调整策略，比如对它说：“刚才的方案不行，换一个思路，这次只改3个核心文件。”由于Claude Code知道你们讨论了什么、为什么刚才的方案不行，因此给出的新方案通常会好很多。

### 2. /fork——平行宇宙探索

`/fork`可以从当前对话分出一个新的对话分支。两个分支共享对话历史，但之后独立发展。

下面是一个典型的使用场景。

```
你：帮我设计一个用户认证系统。
Claude Code：方案A是JWT无状态认证......
你：输入/fork。
（新分支开始）
你：之前的JWT方案先放着。换一种思路，用Session+Redis怎么样？
Claude Code：方案B是Session认证......
（两个分支并行，你比较哪个更适合）
```

不用担心为“走错路”付出过高成本。借助`/fork`，你可以同时探索多条路径，最后选择最好的那条。

## C.3 效率工具：做更多，花更少

### 1. /cost——你花了多少钱

`/cost`可以显示当前会话的词元消耗量和大概费用。

这个命令看起来简单，却改变了我使用Claude Code的方式。在使用`/cost`之前，我对每次会话花了多少钱没有概念。通过该命令查看后发现：一次长会话可能花2～5美元，一次大规模重构可能花10美元以上。这不是让你刻意节省，而是让你有意识地判断：这个任务值不值得消耗这么多词元。

建议在以下时机使用`/cost`：

- 开始一个大任务之前（设定预期）
- 感觉对话太长的时候（判断是否使用`/compact`或`/clear`）
- 一天结束时（了解今天的词元消耗量）

### 2. /model——动态切换模型

`/model`支持在会话中切换使用的模型。一个经济的使用模式是：日常任务用Sonnet（快、便宜），遇到需要深度推理的复杂问题切换为Opus（慢一点，但能力更强）。Sonnet的成本大概是Opus的1/5。对大多数编码任务（如写CRUD、修复简单bug、生成样板代码）来说，Sonnet完全够用。

```
/model sonnet   ← 日常任务
/model opus     ← 复杂架构决策、大规模重构
```

**3./fast——切换到快速模式**

`/fast`可以切换到快速模式。快速模式使用同一个模型，但输出更快。如果你不需要Claude Code进行深入思考，只是要它快速完成一些明确的任务，开启快速模式可以明显加速。

**4./btw——不打断主线的临时问答**

`/btw`是一个被严重低估的命令。通过该命令，你可以在Claude Code正在工作时问一个无关的问题，Claude Code回答该问题后，会继续之前的工作。

```
(Claude Code正在重构你的代码)
/btw TypeScript里readonly和const的区别是什么?
(Claude Code快速回答，然后继续重构)
```

关键是，`/btw`的回答不会进入对话历史。这种问答是“一次性”的，不污染上下文，也不消耗额外的词元，非常适合处理在工作中突然想到的小问题。

### C.4 项目管理：让Claude Code记住一切

**1./init——创建你的第一个CLAUDE.md**

`/init`可以扫描当前项目的结构，自动生成初始的CLAUDE.md文件。它会读取你的`package.json`、`README`和代码结构，推断出项目的技术栈和规范。

生成的CLAUDE.md只是一个起点，不是终点。你需要在其基础上添加自己的规则。但相比从零开始写要方便很多，至少不用再考虑技术栈和项目结构等。

**2./memory——审计Claude Code的记忆**

`/memory` 可以列出 Claude Code当前加载的所有记忆文件：项目级CLAUDE.md、用户级CLAUDE.md ( `~/.claude/CLAUDE.md`)，以及自动生成的CLAUDE.local.md。

CLAUDE.local.md是Claude Code自动维护的笔记。若你的项目开启了自动记忆功能，Claude Code会在工作过程中把发现的有用信息（如构建命令、调试模式、架构决策）自动记录到这个文件中。开启新对话时，这些信息将自动加载。

定期通过`/memory`看看Claude Code记录了什么。它记录的信息可能是错的或过时的，你需要手动纠正。

**3./permissions——安全管理**

`/permissions`用于管理Claude Code的权限。你可以预授权某些操作（如允许运行`npm test`），也可以收紧某些操作（如禁止删除文件）。

对于新手，建议先用默认权限（每次操作都会询问），熟悉之后再逐步放开。对于高频操作（运行测试、`git status`），预授权可以省去大量确认步骤。

### C.5 代码质量：让AI帮你审查

**1./review——自动进行代码审查**

完成一组代码修改后，输入`/review`,Claude Code会审查所有未提交的变更，并给出改进建议，就像一名24小时在线的代码审查员。

它会检查：

- 潜在的bug（空值检查、边界条件）
- 性能问题（不必要的循环、大数据量操作）
- 安全问题（SQL注入、XSS、硬编码密钥）
- 代码风格（命名一致性、文件组织）

**2./simplify——多角度精简**

`/simplify`命令很有趣。它会启动3个并行的智能体，分别从复用性、质量和效率这3个角度审查你的代码变更，然后汇总发现的问题并自动修复。

它不是进行理论上的审查，而是会直接修改你的代码。审查完成后，你可以通过diff确认。

### C.6 高级操作：解锁更多可能

**1./doctor——健康检查**

`/doctor`会执行一系列诊断检查：CLI版本是否最新、认证是否有效、必要工具是否已安装、环境变量是否正确等。

遇到莫名其妙的问题时，先运行一下`/doctor`。很多时候问题源于环境配置，而不是Claude Code本身。

**2./vim——切到Vim模式**

`/vim`用于开启Vim键绑定。Vim用户可以直接在输入框中使用Vim的编辑命令（d、c、y、w等motion commands），无须切换到外部编辑器修改提示词。

**3./terminal-setup——终端集成**

`/terminal-setup`可以自动配置你的终端，比如修改Shift+Enter快捷键的作用为在输入框中换行（而不是发送）。该命令支持VS Code终端及iTerm2、Alacritty、Warp等主流终端。

**4./export——导出对话记录**

`/export`可以把当前对话导出为纯文本，适合做项目文档、写复盘报告，或者把对话分享给同事。

### C.7 命令组合：工作流模式

将命令组合使用，能够发挥更大的威力。以下是我验证过的几个高效工作流模式。

**1.长会话管理**

```
1./clear      ←干净的起点
2.完成Phase 1
3./compact    ←压缩Phase 1的细节
4.完成Phase 2
5./compact    ←再次压缩
6.完成Phase 3
7./cost       ←检查总消耗
```

这个模式可以把原本只能持续30～60分钟的会话延长到几小时，同时保持Claude Code对项目的理解。

**2.探索性开发**

```
1.讨论需求，确定方向
2./fork       ←分支A：方案一
3.（在分支A中实现方案一）
4.切回原分支
5./fork       ←分支B：方案二
6.（在分支B中实现方案二）
7.比较两个方案的结果，选择更好的
```

**3.安全重构**

```
1./review             ←先审查当前代码状态
2.让Claude Code开始重构
3.如果方向不对→按两次Esc键或输入/rewind→回滚文件
4.重构完成→/simplify       ←多角度精简
5./review             ←最终审查
6.git commit
```

**4.经济模式**

```
1./model sonnet       ←用便宜的模型做日常任务
2.遇到复杂问题→/model opus ←切到强模型
3.解决后→/model sonnet    ←切回来
4./cost               ←看看省了多少
```

**斜杠命令的本质**

斜杠命令不是“功能列表”，而是“工作流基建”。单独看每个命令都很简单，但当把它们组合成工作流模式时，你的效率会有质的提升。就像Git的命令，`add`、`commit`、`push`单独看都很简单，组合起来就是整个版本控制体系。Claude Code的斜杠命令也一样：`/compact` + `/context` + `/clear` = 上下文管理体系，`/fork` + `/rewind` = 安全探索体系，`/cost` + `/model` = 成本控制体系。

# 附录D 常见问题与解答

下面是从官方文档、GitHub Issues和社区讨论中整理出的高频问题。遇到问题时，可以先来这里查找。

### D.1 安装与配置

**Q：安装后输入claude，提示command not found。**

这是最常见的问题，通常是因为Shell的PATH没有更新。关闭终端后重新打开，或检查`~/.zshrc`(macOS)、`~/.bashrc` (Linux) 是否包含Claude Code的PATH条目。运行`/doctor`可自动诊断。

**Q：macOS安装报dyld错误或Abort trap。**

Claude Code要求macOS 13.0及更高版本。若使用旧版系统，需先升级。

**Q：企业网络环境下无法连接。**

企业VPN可能有SSL中间人证书，需配置`NODE_EXTRA_CA_CERTS`，信任自定义CA证书。可通过公司IT部门获取证书路径。

**Q：Windows上报Git not found。**

需要安装Git for Windows，并在安装时选中Add Git to PATH。Windows路径中有空格也会导致MCP路径解析失败。

### D.2 使用技巧

**Q：CLAUDE.md应该写什么？**

从技术栈、编码约定和已知的坑开始。不要太长，Claude Code能有效遵循的用户指令为100～150条。运行`/init`可自动生成初始文件。详见第5章。

**Q：上下文窗口满了怎么办？**

目前，Claude Code支持的最大上下文窗口约为100万个词元，使用率达到约83.5%时自动压缩。可以用`/compact`手动压缩，用`/context`查看各组件的消耗。建议将长任务拆成多个短会话。

**Q：怎么让Claude Code效果更好？**

5条核心经验：（1）先在Plan模式下讨论方案，再执行；（2）给出精确的文件路径和行号；（3）不要同时接入太多MCP服务器；（4）将大任务拆成小会话；（5）给出明确的完成标准（如“测试通过就停”），而非模糊需求。

**Q：怎么创建自定义命令？**

在`.claude/commands/`目录下创建`.md`文件，用自然语言编写步骤，支持`$ARGUMENTS`占位符。可以通过`/命令名`调用。详见第7章。

**Q：怎么在CI/CD中使用？**

使用`claude -p "query"` 非交互模式，支持text、json、stream-json输出格式。在GitHub Actions中将API Key存入Repository Secrets。目前，超过60%的团队通过GitHub Actions完成集成。

### D.3 计费与用量

**Q：Pro、Max和API Key付费方案的区别是什么？**

|方 案|价 格|用 量|适 合|
|---|---|---|---|
|Pro|20 美元 / 月|约 5 倍于免费版|个人轻度使用|
|Max 5x|100 美元 / 月|约 25 倍于免费版|个人重度使用|
|Max 20x|200 美元 / 月|约 100 倍于免费版|专业用户 / 小团队|
|API Key|按词元计费|无上限|CI/CD、企业集成|

**Q：Claude API报429 Rate Limit怎么办？**

4个缓解方法：（1）用`/compact`压缩上下文，减少输入词元；（2）拆大会话为小会话；（3）避开高峰期（大概20:00到次日03:00）；（4）升级付费方案。

**Q：怎么节省词元？**

（1）把大文件拆成小的单一职责文件；（2）禁用闲置的MCP服务器；（3）定期用`/compact`压缩；（4）合并操作，减少交互次数。

**Q：Agent Teams的词元消耗量是普通的多少倍？**

6～8倍（具体取决于子任务数量和上下文复用率等）。每个智能体维护独立的上下文窗口，相当于独立的Claude Code实例在运行。

**Q：为什么订阅后还被要求设置API Key？**

系统可能检测到旧的`ANTHROPIC_API_KEY`环境变量（来自之前的项目）。先运行`unset ANTHROPIC_API_KEY`清除，然后运行`claude login`重新认证。

### D.4 权限与安全

**Q：Auto模式安全吗？**

Auto模式是官方推荐的效率与安全折中方案。低风险操作（如读文件）自动放行，高风险操作（如删文件、推代码）仍需确认。

**Q：--dangerously-skip-permissions是什么？**

它是用于跳过所有权限确认的启动参数，其名字中的dangerously绝非夸大。仅在隔离的容器或虚拟机环境中使用该参数，建议配合`--network none`限制网络访问，以防止数据泄露。在日常开发中，绝对不要使用。

**Q：Claude Code会不会把我的代码发到外面？**

Claude Code的沙箱限制了Bash工具的文件系统和网络访问，但MCP工具可能连接外部服务。应严格管控每个MCP服务器的访问范围，敏感凭证用环境变量配置，而非硬编码。

### D.5 报错与故障

**Q：529 Overloaded和429 Rate Limit的区别是什么？**

529 Overloaded是Anthropic服务端过载导致的错误，不是你的问题，通常30秒内自动恢复。429 Rate Limit是你触发了速率或配额限制，需要压缩上下文或等待重置。

**Q：Claude Code在对话中忘了之前的要求。**

上下文压缩会丢失早期细节。解决方案：把重要约束写进CLAUDE.md，而非只在对话中说；用Post-Compact hook在压缩后自动注入关键规则；完成一批任务就写入文件，保存中间结果。

**Q：Claude Code在修改过程中走偏。**

提供明确的验证标准特别重要。“测试通过就停”或“生成文件就行”比模糊的“帮我优化一下”能让Claude Code更快地收敛。建议先在Plan模式下对齐方案，再执行。

**Q：MCP服务器连接失败。**

常见原因：路径中有空格、配置了错误的作用域（全局或项目）、接入太多服务器（5个服务器可能有50多个工具描述，会消耗大量上下文）。应从最小集开始，逐个添加。

**Q：~/.claude目录越来越大。**

Claude Code不会自动管理磁盘，需要用户手动定期清理。可以删除旧会话文件，但保留`settings.json`和CLAUDE.md。

遇到任何不明白原因的问题，先运行`/doctor`。这个内置诊断命令能自动检测安装、网络、配置等常见问题，往往能直接给出解决方案。

# 附录E CLAUDE.md模板集

不同类型的项目需要不同的CLAUDE.md。本附录提供了4个经过实战验证的模板，可直接复制（或根据自己的情况稍加修改）使用。

在我使用 Cursor 的时候就一直写`.cursorrules`文件——本质上和CLAUDE.md相同：告诉AI项目是什么、怎么写代码、有哪些规则。后来，我做了一个VS Code插件，里面有十几套不同技术栈的cursorrules模板（iOS、React、Vue、小程序、Chrome扩展等），帮助用户快速配置。

当我转到Claude Code之后，这些规则几乎可以直接复用，其核心结构是一样的。

|.cursorrules|CLAUDE.md|本 质|
|---|---|---|
|Role & Goal|项目概述|让 AI 知道自己在做什么|
|技术栈规则|技术规范|约束代码风格和技术选择|
|三步工作流|工作流程|定义怎么做事|
|问题解决模式|排错指南|遇到问题怎么办|

如果你之前用Cursor并且有`.cursorrules`文件，可以直接把内容复制到CLAUDE.md中。格式完全兼容。

### 模板1：前端项目(React/Next.js)

````
# 项目名称
## 项目概述
一个基于Next.js 15的XXX平台，使用React 19 + Tailwind CSS +shadcn/ui。
## 技术栈
- **框架**:Next.js 15 (App Router)
- **UI**:Tailwind CSS + shadcn/ui
- **状态管理**:Zustand
- **数据获取**:React Query
- **类型**:TypeScript strict模式
- **包管理器**:pnpm
## 项目结构

```
src/
├── app/        #Next.js路由和页面
├── components/ #可复用组件
│   ├── ui/     #shadcn/ui基础组件
│   └── features/ #业务功能组件
├── hooks/      #自定义hook
├── lib/        #工具函数和配置
├── types/      #TypeScript类型定义
└── styles/     #全局样式
```

## 代码规范
- 组件: 函数组件 + hook, 不用class组件
- 文件命名: kebab-case(my-component.tsx)
- 导出: 具名导出, 不用default export (除了页面文件)
- 样式: Tailwind优先, 自定义CSS用CSS Modules
- 状态: 组件内用useState, 跨组件用Zustand store
- 每个组件文件不超过150行, 超了就拆分
## 常见操作
- 启动开发服务器: `pnpm dev`
- 运行测试: `pnpm test`
- 构建: `pnpm build`
- 添加shadcn组件: `pnpm dlx shadcn@latest add [组件名]`
## 注意事项
- 不要在Server Component里使用useState/useEffect
- 图片用next/image组件, 不用原生img
- API调用统一走src/lib/api.ts, 不在组件里直接fetch
````

### 模板2：iOS App项目(SwiftUI)

````
# 项目名称
## 项目概述
一个SwiftUI iOS应用, 功能是XXX。最低支持iOS 17。
## 项目状态
| 模块 | 状态 | 说明 |
|--------|--------|--------|
| 核心功能 | 完成 | 基础功能已上线 |
| 用户系统 | 开发中 | 登录/注册/个人中心 |
| 推送通知 | 待开始 | 需要配置APNs |
## 技术栈
- **UI框架**: SwiftUI
- **架构**: MVVM
- **数据持久化**: SwiftData
- **网络**: URLSession + async/await
- **依赖管理**: Swift Package Manager
## 项目结构

```
项目名/
├── App/        #App入口和配置
├── Views/      #SwiftUI视图
│   ├── Home/
│   ├── Settings/
│   └── Shared/   #共享UI组件
├── ViewModels/ #视图模型
├── Models/     #数据模型
├── Services/   #网络/存储等服务层
└── Utilities/  #工具类和扩展
```

## 代码规范
- 每个View对应一个ViewModel
- 使用@StateObject持有ViewModel, 不用@ObservedObject
- 异步操作用async/await, 不用completion handler
- 颜色用Asset Catalog定义, 不硬编码
- 文件命名: 大驼峰(HomeView.swift, HomeViewModel.swift)
- 每个文件不超过200行
## 常见操作
- 在Xcode中运行: Cmd+R
- 运行测试: Cmd+U
- 清理构建缓存: Cmd+Shift+K
## 注意事项
- 不要在View的body里做异步操作(用.task修饰符)
- 真机调试需要先在Xcode设置签名Team
- @Published属性必须在MainActor上更新
- 相机/相册权限需要在Info.plist添加Usage Description
````

### 模板3：内容创作项目

```
# 写作项目
## 项目定位
这是一个内容创作工作区。我是[你的身份], 主要做[你的内容类型]。
## 工作区路由
| 关键词 | 工作区 | 读取规则 |
|--------|--------|--------|
| 公众号、文章 | 文章写作 | /articles/RULES.md |
| 视频、脚本 | 视频创作 | /videos/RULES.md |
| 社交、推文 | 社交媒体 | /social/RULES.md |
## 写作风格
- 语言风格: [你的风格, 比如: 口语化、有温度、不装腔作势]
- 破折号: 全篇最多2处
- 加粗: 全篇约10处, 标记重点句
- 段落: 3～5句为主
- 禁用词: [列出你不想出现的词, 比如“赋能”“综上所述”“值得一提的是”]
## 审校标准
三遍审校:
1.审内容: 事实准确吗? 逻辑通吗? 有遗漏吗?
2.审风格: 有AI味吗? (套话、机械结构、态度中立、细节缺失)
3.审细节: 句子长度、段落间距、格式统一
## 输出规范
- 文章草稿: 写入.md文件, 不直接写在回复中
- 文件位置: 项目对应目录下
- 命名规范: YYYY.MM-标题.md
## 常用操作
- /proofreading→三遍审校
- /research→结构化调研
- /topic-gen→选题生成
```

### 模板4：后端API项目(Python/Node.js)

````
# 项目名称
## 项目概述
一个基于[框架名]的REST API服务, 负责XXX业务。
## 技术栈
- **运行时**: Python 3.12 / Node.js 22
- **框架**: FastAPI / Express
- **数据库**: PostgreSQL
- **ORM**: SQLAlchemy / Prisma
- **缓存**: Redis
- **部署**: Docker + AWS ECS
## 项目结构

```
src/
├── api/        #路由和端点定义
│   └── v1/     #API版本管理
├── models/     #数据模型/Schema
├── services/   #业务逻辑层
├── repositories/ #数据访问层
├── middleware/ #中间件(认证、日志、错误处理)
├── config/     #配置文件
└── tests/      #测试
    ├── unit/
    └── integration/
```
## 代码规范
- API路径: RESTful风格, 小写+连字符(/user-profiles)
- 错误处理: 统一错误格式 { code, message, details }
- 认证: JWT Bearer Token, 密钥从环境变量读取
- 数据验证: 入口层验证(Pydantic/Zod), 不在业务层重复验证
- 日志: 结构化日志(JSON格式), 包含request_id
## 环境变量
```
DATABASE_URL=   #PostgreSQL连接串
REDIS_URL=      #Redis连接串
JWT_SECRET=     #JWT签名密钥(.env文件,不入库)
API_KEY=        #外部API密钥(.env文件,不入库)
```

## 常见操作
- 启动开发: `docker compose up` 或 `uvicorn main:app --reload`
- 运行测试: `pytest` / `pnpm test`
- 数据库迁移: `alembic upgrade head` / `npx prisma migrate dev`
- 查看API文档: `http://localhost:8000/docs`
## 安全规则
- ✖ 不在代码里硬编码密钥、密码、凭证
- ✖ 不在日志里输出敏感数据(用户密码、凭证等)
- 所有用户输入必须验证和清理
- SQL查询用参数化查询, 不拼接字符串
- API端点必须有认证(除了/health和/docs)
````

### 写好CLAUDE.md的5个原则

**原则1：具体胜于模糊。**“代码风格要好”没有用，“函数不超过30行，文件不超过200行，用具名导出”才有用。AI需要可执行的规则。

**原则2：精简胜于全面**。每次对话CLAUDE.md都会被加载到上下文中。文件最好控制在200行以内，超过500行就会浪费词元。如果规则太多，可以拆成子文件，在主CLAUDE.md中引用。

**原则3：踩坑记录比最佳实践有用**。AI知道最佳实践，但“在Server Component中用useState会报错”这种项目特有的坑，不写就会反复踩。CLAUDE.md最有价值的部分往往是“注意事项”。

**原则4：持续更新**。不要写完CLAUDE.md就不管了。每次发现Claude Code犯了重复的错误，就把规则加进去。每个月清理一次过时的规则。它是一份“活的”文档。

**原则5：先写3条，再慢慢加**。不要试图一步到位。先写3条你最在意的规则（如技术栈、文件命名、安全要求），使用几天，如有需要再补充。3个月后，你会得到一份完美贴合你的工作习惯的CLAUDE.md。