如何将 NotebookLM 变成你的私人麻省理工学院教授

The students winning with NotebookLM are not using it the way you think. They are not summarizing textbooks. They are not generating flashcards. They are not making podcast versions of their reading. They are running it as a Socratic tutor, a research advisor, and a synthesis engine simultaneously, and it costs them nothing. I spent the last 6 months reverse engineering what these students are actually doing differently. The system underneath it is shockingly repeatable, and almost nobody outside a small circle of self-taught learners has figured it out yet. Here is exactly how it works.

Most people upload a textbook and ask it to summarize chapters. That is the equivalent of buying a Ferrari and using it to deliver pizza. NotebookLM is not a summarization tool. It is a synthesis engine. It can hold dozens of sources in context simultaneously and answer questions across all of them at once. The students winning with it understand one thing the rest of us missed: the point is not to compress information. The point is to make information argue with itself until something new comes out. Once you internalize that, every prompt you write changes. Step 1: Build the curriculum before you start learning This is the move that changes everything. Most students learn linearly. They open the textbook on page one and grind through to page 800. The good students do the same thing but with better notes. The result is the same. They retain almost nothing six months later because the brain stores connections, not pages. The students winning with NotebookLM invert the entire process. Before they open a single book, they build the curriculum they want to learn from. For every subject (calculus, physics, chemistry, computer science, history of science) they upload into a single NotebookLM notebook: - The most-recommended textbook in the field - Two alternative textbooks that disagree with the first one - 5 to 10 foundational papers in the subject - 3 to 5 lecture transcripts from MIT OCW or Stanford Online - Any Wikipedia deep-dive on the topic - 1 or 2 popular books written for non-specialists That is roughly 15 to 20 sources per notebook. Then they run this prompt:

---

Across every source I have uploaded, identify the 12 core concepts a beginner needs to understand before anything else in this field makes sense. For each concept:

1. Explain it as if I am intelligent but new to the field
    
2. List which sources cover it best
    
3. List which sources contradict each other on this concept
    
4. Tell me the single biggest misconception people bring into this subject from the outside
    

Do not list more than 12. The goal is the irreducible foundation, not a comprehensive overview.

---

The output of this prompt is the syllabus. You spend 2 to 3 hours per subject just running this one prompt and refining the output. By the end, you have a learning roadmap that no textbook would ever give you because no textbook author has the perspective of 20 sources at once.

Step 2: The Feynman prompt that builds real understanding This is the part that does most of the work. Once you have your 12 core concepts, you do not read about them. You explain them. For every single concept on your syllabus, run this prompt:

---

I am about to try to explain [CONCEPT] in my own words. I will type my explanation below.

Your job is to:

1. Identify every place my explanation is technically wrong
    
2. Identify every place I am using the right words but missing the actual mechanism
    
3. Identify every place a domain expert would push back
    
4. Tell me the one underlying principle I have not yet grasped that would make this concept click
    

Do not be polite. Be precise.

Here is my explanation:

[Type your explanation from memory, no matter how rough]

---

This is the Feynman technique forced into a conversation. Most students think they understand something because they can recognize it. NotebookLM destroys that illusion in 30 seconds. You type your explanation. It tells you exactly where the gaps are. You go back, re-read the relevant sources, and try again. Run this loop 3 to 5 times per concept until NotebookLM has nothing left to push back on. That is when you actually understand it.

Step 3: The contradiction prompt that makes you think like a researcher This is the prompt that separates a student from a thinker. After you understand a concept individually, run this:

---

Across all my uploaded sources, find every place where the authors disagree about [CONCEPT].

For each disagreement:

1. State the position of each author
    
2. Explain what evidence each one is using to support their position
    
3. Tell me which side has stronger empirical support and why
    
4. Tell me what question I would need to answer to settle the disagreement myself
    

Do not soften the disagreements. The point is to find them, not smooth them over.

---

Most students learn subjects as if they are settled. The textbook gives one answer. They memorize it. They move on. Real fields are not like that. Every interesting subject has live disagreements at the frontier, and the ability to see those disagreements is what separates someone who knows the material from someone who can do something with it. NotebookLM surfaces them in seconds. The students who win at top universities are not the ones who memorized the most. They are the ones who learned to identify which arguments inside their field are still open. This prompt teaches you to do that automatically, on every subject you study, for the rest of your life.

Step 4: The teaching prompt that locks it in The final move for every concept is this:

---

I want to teach [CONCEPT] to a curious 12-year-old who has never heard of this subject.

Generate 5 questions that 12-year-old would most likely ask me within the first 2 minutes of my explanation.

Then for each question, tell me:

1. What the most accurate but accessible answer would be
    
2. What analogy or example I should use
    
3. What follow-up confusion is most likely to come from my answer
    

---

The protégé effect is one of the most replicated findings in cognitive science. Your brain encodes information dramatically more deeply when it believes it will have to teach the material to someone else. NotebookLM lets you simulate the student you never had. By the end of every subject, you have effectively taught it 12 times. That is why it sticks.

Step 5: The weekly synthesis ritual This is the prompt almost nobody runs. Every Sunday, across every notebook you built that week:

---

Across every concept I learned this week, identify:

1. The 3 strongest connections between concepts in different subjects
    
2. The single insight that unifies the most material
    
3. The one question I should be carrying into next week that would deepen my understanding of multiple subjects at once
    
4. Anything I learned this week that contradicts something I learned last week
    

Be ruthless. Most weekly reviews are useless because they list everything. I only want what compounds.

---

Most students learn subjects in silos. Math here. Physics there. Computer science somewhere else. The students who break out of mediocrity are the ones who notice that the same idea shows up in three fields under three different names. NotebookLM finds those connections automatically the moment you give it the right instruction. This is the prompt that builds polymath thinking. Run it for 6 months and the way you think changes permanently. You stop seeing subjects as separate. You start seeing the underlying patterns that show up everywhere. That is what people are talking about when they describe someone as intellectually mature. It is not knowledge. It is connection density.

What this actually teaches you The tool is not the point. The point is that for the first time in human history, a self-taught student anywhere in the world has access to the same intellectual infrastructure as a graduate student at MIT. Not the same teachers. The same depth of inquiry. The same ability to put 20 sources in conversation, find contradictions, force precision in their own explanations, and build understanding that compounds across subjects. The MIT professor who would have charged $200,000 to teach you does not exist in your life. What exists is a free notebook that does most of what that professor would have done if you ask the right prompts. That is the actual unlock. Most people are still using NotebookLM to summarize PDFs they will never look at again. The students who will run the next decade are the ones who figured out it is a synthesis engine, a Socratic tutor, and a research advisor in one interface. Open one notebook. Upload 15 sources on something you care about. Run the first prompt. You will understand what I mean within an hour.

What subject would you upload 20 sources on first if you tried this today?


---

### 如何将 NotebookLM 变成你的私人麻省理工学院教授

使用 NotebookLM 取得好成绩的学生并没有像你想象的那样使用它。 他们没有总结教科书内容，也没有制作记忆卡片，更没有把阅读内容改编成播客。 他们同时将其用作苏格拉底式辅导、研究顾问和综合引擎，而且无需任何成本。 过去六个月，我一直在反向推导这些学生究竟有何不同之处。其背后的系统惊人地具有可复制性，但除了少数自学成才的人之外，几乎没有人掌握其中的奥秘。 具体运作方式如下。

大多数人上传教科书，要求程序总结章节内容。 这相当于买了一辆法拉利，却用它来送披萨。 NotebookLM 不是摘要工具，而是综合引擎。它可以同时整合数十个信息源，并一次性回答涉及所有这些信息源的问题。那些使用 NotebookLM 取得成功的学生明白一个我们其他人忽略的道理：关键不在于压缩信息，而在于让信息相互碰撞，直到产生新的见解。 一旦你理解了这一点，你写的每一个提示都会改变。 第一步：在开始学习之前先制定课程计划 这一举动将改变一切。 大多数学生的学习方式是线性的。他们从课本第一页开始，一直读到第800页。优秀的学生也这样做，但他们会做更详细的笔记。结果却是一样的：六个月后，他们几乎什么都记不住，因为大脑记住的是联系，而不是书页。 使用 NotebookLM 获胜的学生颠覆了整个过程。 在翻开任何一本书之前，他们就已经构建好了自己想要学习的课程。 他们将每个科目（微积分、物理、化学、计算机科学、科学史）的内容上传到同一个 NotebookLM 笔记本中： 该领域最受推荐的教科书 两本与第一本观点相左的替代教科书 - 该学科的5至10篇基础论文 - 3 至 5 份来​​自麻省理工学院开放课程 (MIT OCW) 或斯坦福在线课程的讲义 - 任何关于该主题的维基百科深度文章 - 1 或 2 本面向非专业人士的科普书籍 也就是说，每个笔记本大约有 15 到 20 个参考文献。 然后他们运行以下提示：

---

在我上传的所有资料中，找出初学者在理解该领域其他内容之前必须掌握的12个核心概念。针对每个概念：

1. 请用一种假设我聪明但对该领域不熟悉的方式来解释。
    
2. 列出哪些媒体对此报道得最详尽。
    
3. 列出哪些资料来源在这个概念上相互矛盾。
    
4. 请告诉我，外界人士对这个主题最大的误解是什么。
    

列出的内容不要超过 12 个。目标是不可简化的基础，而不是全面的概述。

---

此提示的输出结果是课程大纲。 你每个科目都要花两到三个小时来运行这一个提示并完善输出结果。最终，你会得到一份学习路线图，这是任何教科书都无法提供的，因为没有哪个教科书作者能同时兼顾二十个资料来源。

步骤二：费曼提示，助您建立真正的理解 这是完成大部分工作的部分。 一旦你掌握了12个核心概念，你就不需要阅读它们，而是要解释它们。 针对教学大纲中的每一个概念，运行以下提示：

---

我接下来要用自己的话解释一下[概念]。我的解释如下。

你的工作是：

1. 指出我的解释中所有技术上错误的地方
    
2. 请找出所有我使用了正确词语但却缺少实际机制的地方。
    
3. 找出领域专家会提出异议的每一个地方
    
4. 告诉我，我还没理解的那个关键原理是什么，它能让我明白这个概念。
    

不要客气，要准确。

我的解释如下：

[凭记忆写下你的解释，无论多么模糊都可以]

---

这是强行将费曼技巧引入对话中。 大多数学生认为自己理解了某些内容，仅仅是因为他们能认出它。NotebookLM 只需 30 秒就能打破这种错觉。你只需输入你的解释，它就能准确地指出你的知识漏洞。然后你就可以返回去，重新阅读相关资料，再尝试解释。 每个概念运行此循环 3 到 5 次，直到 NotebookLM 没有任何内容需要推送为止。 只有当你真正理解它的时候，你才会明白。

步骤三：让你像研究者一样思考的矛盾提示 正是这个问题区分了学生和思考者。 当你独立理解了一个概念之后，运行以下命令：

---

在我上传的所有资料中，找出作者们对 [概念] 存在分歧的每一处。

针对每一项分歧：

1. 说明每位作者的立场
    
2. 请解释他们各自使用了哪些证据来支持自己的观点。
    
3. 请告诉我哪一方的论据更有实证支持，并说明原因。
    
4. 请告诉我，我需要回答什么问题才能自行解决这场分歧。
    

不要淡化分歧。关键在于找到分歧，而不是掩盖它们。

---

大多数学生学习科目时都好像这些科目已经定型了一样。 教科书只提供一个答案。他们死记硬背，然后继续学习。但现实领域并非如此。每个有趣的学科在前沿都存在着激烈的争论，而能否发现这些争论，正是区分掌握知识的人和能够运用知识的人的关键所在。 NotebookLM 可在几秒钟内将它们显示出来。 在顶尖大学取得成功的学生并非记忆力最强的，而是那些学会识别自己领域内尚存争议的学者。这道题旨在教会你如何在学习的每一门学科中，终身受益，并能自动做到这一点。

第四步：巩固教学要点 每个概念的最后一步都是这样的：

---

我想向一位从未听说过这个主题、充满好奇心的 12 岁孩子讲解 [概念]。

请列出 5 个 12 岁孩子在我解释后的前 2 分钟内最有可能问我的问题。

然后，针对每个问题，请告诉我：

1. 最准确且易于理解的答案是什么？
    
2. 我应该用什么类比或例子呢？
    
3. 我的回答最有可能引发哪些后续困惑
    

---

门徒效应是认知科学中最常被重复验证的发现之一。 当大脑认为需要将知识点传授给其他人时，它会更深入地编码这些信息。NotebookLM 可以让你模拟从未教过的学生。 每个科目结束时，你实际上已经教授了 12 遍。 这就是它能粘住的原因。

第五步：每周的综合仪式 这是几乎没人会运行的提示符。 每个星期天，在你当周创建的每个笔记本中：

---

在我本周学习的每一个概念中，都应找出：

1. 不同学科概念之间最强的3个联系
    
2. 最能概括物质世界的单一洞见
    
3. 我应该带到下周的一个问题，将加深我对多个学科的理解。
    
4. 这周我学到的任何与上周学到的东西相矛盾的东西
    

要果断。大多数周报都没什么用，因为它们列出了所有东西。我只想知道哪些化合物。

---

大多数学生都是孤立地学习各个学科。 这里是数学，那里是物理，还有别处的计算机科学。那些能够脱颖而出的学生，正是那些注意到同一个概念以三种不同名称出现在三个领域中的学生。 只要你发出正确的指令，NotebookLM 就会自动找到这些连接。 这是培养博学思维的提示。 坚持六个月，你的思维方式会发生永久性的改变。你不再将事物视为彼此独立的个体，而是开始看到无处不在的潜在模式。这就是人们所说的“智力成熟”的含义。它并非知识的积累，而是联系的紧密程度。

这实际上教会了你什么 工具本身并不是重点。 关键在于，人类历史上第一次，世界各地的自学者都能获得与麻省理工学院研究生相同的知识资源。虽然师资不同，但探究的深度却不相上下。他们同样能够将20个信息来源进行对话，找出矛盾之处，力求在自己的解释中做到精准，并构建跨学科的综合理解。 你生活中并不存在那种会收取 20 万美元学费的麻省理工学院教授。 现在有一个免费的笔记本，如果你提出正确的问题，它能做到那位教授通常会做的事情。 这就是真正的解锁方式。 大多数人仍然使用 NotebookLM 来摘要他们永远不会再查看的 PDF 文件。 未来十年将主导社会的学生，是那些发现它集合成引擎、苏格拉底式导师和研究顾问于一体的界面的人。 打开一个笔记本。上传 15 个与你感兴趣的主题相关的资料来源。 运行第一个提示符。 一个小时之内你就会明白我的意思了。

如果今天尝试上传20个参考文献，你会首先选择哪个主题？