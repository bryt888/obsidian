【麻省理工长毛兔的作品】 https://v.douyin.com/WW2yLsWtScw/

今天来讲一个Claude Code中的高级操作，叫做hooks。会给大家介绍一下它是什么东西，应该在什么时候用。会给大家看到一个hook，并且解释它每一行是什么意思。最后会一起来做一个案例，把一个hook放进Claude Code中。
 
hook也就是所谓的钩子，它比较像是在用Claude Code的时候，希望当遇到某些情况或者是某个阶段，需要Claude Code一定去做的事情，并且可以自定义这是什么事情。
 
当说到不同的阶段的时候，有哪些阶段？它们分为PreToolUse，当它用一个工具之前，可以让它去确认一些内容或者确认一些输入，再去用这个工具。PostToolUse，用了一个工具之后，也可以让它去做一些log或者让它做一些检查和测试。
 
它可以是在PermissionRequest，也就是在等approval的时候，可以给一些声音或者是视觉上的提示。它可以是在Stop，就是Claude Code已经把事情做完的时候，也可以给一些提示或者是一些log，甚至是发一个信息。它还可以是在SessionStart和End的时候，有些人可能希望在SessionStart开启一个新的session的时候，让Claude Code来做一些initialization或者是load一些context。
 
还有些人可能希望在一个SessionEnd的时候，让Claude Code自动去做一些logging或者是一些clean-up，这些都是非常适合运用hook的案例。
 
一个hook到底长什么样？一起来看一下，它大概就是长这个样子，这是接下来要放进Claude Code中的一个案例。不知道大家，但是我经常发现我需要盯着电脑才知道Claude Code是不是在等我，等我给它一个approval或者等我给它下一步的命令。
 
比起让我盯着屏幕或者是已经过了很久才发现原来它在等我这样的状况，我希望它能在等approval的时候做一个chirping，一个小鸟的叫声。还有当它Stop已经做完一件事情的时候，可以做一个喵，就是小猫的声音。
 
来看这一段的JSON代码，其实很简单，看最上面这个地方，它叫做"hooks"，说明接下来所有写的东西都是hook。hook有两个，一个是"PermissionRequest"，一个是"Stop"。像刚刚跟大家讲过，可以在Stop、PreToolUse、PostToolUse或者是SessionStart、SessionEnd、Notification都可以写一个hook。
 
这里的事件类型就是当Claude需要请求permission的时候，这个钩子就会被触发。接下来这是一个"matcher"，它是一个过滤条件，说明除了这个事件发生之外，还有什么事情需要匹配？对我来说没有其他条件了，任何时候它需要等permission的时候，我都希望它触发，所以我这里什么都没写。
 
接下来这是一个"hooks"数组，其实当上面两个条件都被触发的时候，我可以让它做好几件事情，可以让它给我发一个信息，并且同时在屏幕上做一个pop-up，再同时播放一个声音。对我来说因为我只需要它做一件事情，所以我的这个数组里面只有一个东西，这个东西就是"type": "command"。
 
这里就要说明我需要它做一个什么类型的处理，我这个类型就是command，就是它要去跑一个shell命令的运行。那个command到底是什么？command就是这个去play chirp.mp3的声音。这里就结束了，这就是一个完整的hook。
 
跟它对比，接下来这个"Stop"就是当一件事情做完的时候会被触发，它这里的条件都基本上是一样的，只有最后command的内容是meow.mp3不是chirp.mp3，因为它会播一个不同的声音。
 
所以看这就是一个hook，里面有两个不同的事件可以被触发，然后每一个事件达到触发条件的时候都会有一个动作，就是播放一个声音。基本上这就是一个hook的内容了，还有一些其他可选的条件，是可以加进去的。我觉得最好的方法就是可以跟Claude形容你想达到一个什么样的效果，然后Claude就会直接帮你把这个hook给写出来。
 
接下来看一下怎么把它放进Claude Code中，把它放进Claude Code也很简单，首先就是要打开Claude文档，在里面创建一个新的文档叫sounds。然后在网上下载或者是用AI生成两个声音文件是MP3，一个是chirp，一个是meow，就是我对应的两个声音。
 
把它们放进sounds之后，打开settings这个JSON文档，打开可能里面已经有一些内容了，但是没有关系，把刚刚讨论过的这个hook加上一个逗号，然后全部复制在下面就可以了。这里可以看到刚刚说到chirp.mp3和meow.mp3这两个，然后再把这个文档储存和关掉就可以了。
 
这样整个hook就放进去做好了，来看一下它的成果。
 
这就是关于hooks钩子的一些基本信息，希望对大家有帮助。下一期继续来讲一些关于Claude Code的高级操作。