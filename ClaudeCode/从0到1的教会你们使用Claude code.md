【AI 杰瑞斯的作品】https://v.douyin.com/4Sbg39BZ32Q

朋友们晚上好！今天这个视频只有一件事情，就是从0到1的教会你们使用Claude code。很多人听到“code”这个单词就会想到写代码、编程，但实际上以Claude code为代表的这种AI coding agent并不是只能写代码，而是一个万能的工具。
 
并且就在昨天，飞书刚刚开源了他们的CLI（命令行接口）。这意味着可以使用AI agent帮你做表格、回消息、发通知、定会议和安排日程。它代表的其实是一种新质的生产力，可以不用它写任何的代码，但是要学会使用它。就好像在远古时代，一个人是钻木取火，而另外一个人直接拿着打火机就把火点起来了。
 
Claude code除了编程还能帮你做什么？首先可以帮你做数据分析、管理文件；其次可以帮你爬取每日信息流、资讯；还可以帮你修图、生图、生成播客、剪辑视频，甚至也可以根据你自己的工作方式来搭一套属于自己的skills。
 
这个视频将全面讲解，不需要你有任何预备知识，从基础到高阶循序渐进，包括安装和设置、基础操作和最佳实践，以及hooks、agents、skills、plugins、MCP都会教给你。同时像一些不常见但非常有用的命令，比如simplify、insights、loop，也会教给你。
 
还会分享一些经验，比如如何避免在长时间开发中出现莫名其妙的bug，以及如何优化token节省成本。最后会告诉你如何设计自己的skill点md，来让工作效率最大化。话不多说，直接开始。
 
- 首先来安装Claude code。直接进入它的官网，下滑可以看到有好几行命令，只需根据你的电脑（比如mac os或者windows），选择相应的命令安装就好。
如果是windows用户，推荐使用powershell而不是cmd，因为Claude code的底层是unix风格的命令，所以使用powershell会更好。
比如是mac os用户，就复制对应命令，然后打开终端，粘贴命令再回车，它就会自动装好Claude code。由于这里已经装好了，所以不再演示。
- 安装好Claude code后，使用很简单，只需打开终端，输入“Claude”然后回车，它会询问是否愿意选择当前文件夹作为workspace，点击“trust this folder”就好，进来之后就来到操作界面。
- 现在需要安装另外一个非常重要的工具，叫做cc switch。在国内想要使用官方的服务（比如opus、sonnet或者hugging face）比较麻烦，所以需要使用国产模型（比如mini max、gpt、deep seek、百炼、kimi等），cc switch可以很方便地帮我们配置这些模型，并且可以快速切换。
安装方式：如果是mac os用户，往下滑找到“快速开始”，直接复制对应命令，打开终端，像安装Claude code一样，把命令复制过去然后回车，它就会自动安装。
如果是windows用户，需要点击“release”，来到release界面后一直下滑，找到“cc switch v3.12.3windows.msi”（注意一定要是windows.msi），然后下载安装。
- 安装好cc switch后，配置模型。这里选择mini max2.7，配置方式很简单，点击加号，选择mini max模型，往下滑，它已经帮我们填写好了base url，只需填写api key。api key获取方式：打开mini max官网，选择49元的套餐（目前用下来非常够用），购买套餐后点击“账户管理”，选择“token play”，复制api key然后粘贴，点击添加就配置完成了。
- 现在Claude code和api都配置好了，直接开始使用。打开终端，输入启动命令，回车后选择“yes”，就来到工作页面，这里显示的是mini max2.7和api use。
- 讲讲Claude code的三种模式：
- 第一种是default mode（默认模式），就是当前这种什么都没显示的模式。
- 第二种是plan mode（规划模式）。
- 第三种是bypass permission mode（完全执行模式，相当于full access模式）。
- 先讲第一种default mode，它的特点是Claude code执行任何操作（比如读、写文件、编辑文件）都需要你明确确认之后才会执行。
比如让它在桌面上创建一个文件，这里推荐使用语音输入法“闪电站”（也可以试试抖音的豆包输入法，都是免费的），用语音说“帮我在桌面上建立一个文件夹，里面放一个文件叫做test.md”，回车后它会问文件夹名字，输入“test”，然后它会让你选择是否执行命令，选择“yes”，它就会成功创建。
- default mode每执行一条命令都需要手动确认一次，对于复杂任务很麻烦，所以需要使用危险模式（bypass permission mode）。
进入危险模式需要输入命令“claude --dangerously-skip-permissions”，回车后选择“yes”，进入工作页面后显示“bypass permissions”，说明现在执行命令不需要批准，会直接自动执行。同样用语音说“帮我在桌面上建立一个文件夹，名字叫做test2，里面放一个md文档，名字随便取”，它会直接执行bash命令创建文件夹和文档，然后提示任务完成。所以更推荐使用“--dangerously-skip-permissions”命令启动Claude。
- 讲plan mode（规划模式），它有两种应用场景：
- 做产品、项目初期，想知道AI会怎么执行，是否按照预期方式执行，可以用plan mode让它规划出一个plan，没问题就按plan执行。
- 针对宽广的任务（比如把桌面所有文件迁移到硬盘，操作很多文件），用plan mode比较好。
- 需要reuse代码、审查代码时，让它一条一条干，这个场景用plan mode比较好。
体验一下，用shift+tab切换，输入“整理桌面文件夹，请帮我列个计划”，回车后它会生成一个plan，同时给出三个选项：
- “yes and bypass permission”：无条件执行，不用再问。
- “yes manually approve it”：执行计划，但需要手动批准每项编辑。
- 可以告诉它计划的不足，它会更改后再让你看。比如让它“把垃圾箱清理一下，加入计划后再让我看一下”，它会增加清空垃圾箱的任务，觉得没问题就选择“yes bypass permissions”，它会自动执行。
- 讲Claude code的一些命令：
- 第一个命令是init命令，打开一个项目（比如“voice input”语音输入法项目），输入init命令，它会把整个项目的代码看一遍，然后生成一份cloud.md文档。cloud.md文档在每次会话时会首先加载，里面是一些最高原则（类似机器人的三条法则）。写好cloud.md很重要，不要让它又臭又长，否则会消耗大量token且让agent变笨。解决方式是把长文件拆分，新开一个md文档（比如get.md），把路径记录到cloud.md中，指示agent需要时去查找。任何时候从github下载新项目都可以用init命令，让agent快速了解项目并生成cloud.md，cloud.md相当于agent的最高指示，可以在里面写开发规范（比如千万不能执行rm -rf等），它会随着项目迭代完善。
- agent命令：可以创建多agent团队，用自然语言创建。执行agent命令，选择“create new agent”，选择agent的location（权限级别，选project即可），选择“generate with cloud”（推荐），描述agent要做的事情（比如创建technical co founder agent），它会根据描述创建agent。创建好的agent可以用自然语言调用，当重复做同一类需要长prompt和定很多规范的任务时，应该专门创立一个agent（比如产品经理、后端开发、前端开发、测试、codex agent等），单独的agent可以帮助节约上下文，只需要关心任务是否完成。
- 讲mcp、skills、plugins和hooks：
- mcp（模型上下文协议）和skill（技能、能力）的区别：mcp告诉你能不能做（有没有能力做），skill告诉你有了能力应该怎么做。举例：残疾人没腿（没mcp）不能骑车，有了骑车的mcp（长出双腿）但不会骑，再给骑车的skill（骑车指南），二者结合就会骑车；开卷考试中，mcp是带书（资料），skill是怎么看书找答案。
- 查看安装的mcp：在命令行输入“/mcp”；查看安装的skill：输入“/skills”。skills不是越多越好，而是越精越好，小白可以先安装很多skill摸索，之后要不断精简，否则模型会因工具调用过多而迷茫，不知道什么时候该调用什么工具。
- hooks（钩子）：本质是一段脚本或代码，在特定事情发生时自动触发执行。分为在工具调用前执行的hooks和在工具调用后执行的hooks。比如不想让AI读取私密文件，可以制定hook，在AI进行读操作之前或之后执行，检查读取路径是否涉及私密文件。