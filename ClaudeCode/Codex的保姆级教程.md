https://v.douyin.com/aji62kN2L20/


这是一则关于AI编程工具Codex的保姆级教程
视频，内容涵盖Codex的优势、两种使用方式（命令行和VS Code插件），以及结合Skills生成PPT的实战演示，以下是详细总结：
 
一、Codex的优势
 
- 超越ClaudeCode成为全球最热门AI编程工具，原因在于价格亲民，，对国内用户友好，不会像ClaudeCode那样频繁被封号、限流或降质。
 
二、Codex的安装与使用（命令行方式）
 
1. 安装NodeJS环境
- 若本地未安装NodeJS，需先访问NodeJS官网，下载并安装与自身系统对应的版本。安装过程简单，一路点击“下一步”即可。
- 验证安装：打开命令行，输入 node -v ，若显示NodeJS版本号，说明安装成功。
2. 安装Codex
- 从OpenAI官方仓库的说明文档中复制安装命令 npm install -g @openaai/codex ，在命令行中执行该命令，保持网络畅通，即可快速安装Codex。
3. 准备ChatGPT Plus账号
- 访问ChatGPT官网，通过谷歌账号或邮箱注册账号，再将账号升级为Plus用户。
4. 启动与登录Codex
- 在命令行中输入 codex 启动工具，首次使用会跳转至登录页面，登录ChatGPT Plus账号后，授权信息会加载到Codex中，后续无需重复登录。
5. 模型与推理模式选择
- Codex默认使用 gpt-5.2-codex 模型，可通过 /model 命令切换模型（如 gpt-5.1-codex 等）。
- 推理模式有 Low （响应快但推理能力一般）、 Medium （平衡速度与推理深度，默认选择）、 High （推理能力强但执行慢）三种，通常选择 Medium 即可。
6. 生成代码示例
- 进入项目目录后，可直接在命令行中给Codex下达指令，例如“生成一个美观的登录页面，页面名为login.html”，Codex会生成相应代码并询问是否写入，确认后即可在项目目录中查看生成的文件。
 
三、Skills的安装与PPT生成实战
 
1. 什么是Skills
- Skills可理解为“AI员工手册”，是升级版的提示词，包含原数据、任务执行逻辑和资源包（如工具脚本），能让大模型按预期稳定执行任务，还可通过扩展工具增强大模型能力。
2. 安装Skills（以PPT生成技能为例）
- 访问Anthropic的Skills仓库，找到PPT生成技能的目录，复制其地址。
- 在Codex的命令行中执行 skill-installer <Skills地址> ，同意写入后，Skills会被安装到Codex的全局目录，后续所有项目均可使用。
3. 生成PPT示例
- 下达指令“生成一个关于‘什么是Skills’的极简风格PPT”，Codex会调用安装的PPT生成Skills执行任务，生成完成后可在项目目录中查看PPT文件，其内容结构清晰，风格极简，对Skills的定义、组成和价值等讲解到位。
 
四、VS Code插件方式使用Codex
 
1. 安装VS Code与Codex插件
- 下载并安装VS Code，在VS Code的扩展市场中搜索“OpenAI Codex”并安装官方插件。
2. 共享命令行Codex会话
- 若已通过命令行安装并配置好Codex，VS Code插件可直接共享其会话，无需额外配置。
3. 在VS Code中使用Codex
- 打开项目文件夹后，可在VS Code的Codex交互窗口中执行命令、查看文件、进行回滚操作，还可方便地截图上传以复现页面等，功能与命令行方式一致但操作更便捷。例如，可在窗口中粘贴页面截图，让Codex修改代码风格，修改后可回滚至上一版本。