

Original by Uncle Hua \*March 31, 2026 07:42\*



> \*\*TLDR\*\*: Claude Code’s 1,902 source files were accidentally leaked. After going through them, I realized this is an outstanding textbook on “harness engineering.” Claude Code is good to use because 60% comes from the Opus model’s own capabilities, and 40% comes from the engineering system built around the model, which is the harness. This harness includes: a carefully assembled system prompt, a four-layer permission system that uses a second AI for safety review, a memory system that remembers preferences but not code, a 9-part structured context compression pipeline, and a multi-agent collaboration framework that operates like a real company. For anyone using AI, these design ideas are directly worth borrowing.


Anthropic made a hilariously public blunder today, a nice gift to programmers around the world just ahead of April 1.

Here’s what happened: when updating Claude Code’s npm package, they accidentally left a 60MB source map debugging file in the published package. That file should have been excluded during packaging, but it wasn’t. Anyone could use it to reconstruct Claude Code’s full TypeScript source code.

All 1,902 source files were exposed.

The funniest part is that this wasn’t even the first time. When Claude Code was first released in February 2025, the exact same accident happened, and Anthropic urgently deleted the old npm package. More than a year later, they made the exact same mistake again. Anthropic’s build pipeline may need its own Claude Code review.

But for me, this was a perfect learning opportunity. As someone who uses Claude Code for development every day, what I was more curious about was: \*\*how exactly are those experiences that make me think “why is this AI so good to use?” implemented under the hood?\*\*

I spent several hours going through all 1,902 files.

In 2026, there is a concept that has been getting hotter and hotter: \*\*harness engineering\*\*. It means that whether an AI agent is good to use depends not only on how strong the model is, but also on how good the “harness” built around the model is. The harness includes tools, constraints, feedback loops, safety mechanisms, memory systems... all the engineering systems that turn AI from “powerful but unpredictable” into “stable, reliable, and deliverable.”

Claude Code’s source code is a living textbook of harness engineering.

## 1. You thought AI only received your one sentence, but in reality it received an entire instruction manual

When you type an instruction into Claude Code, what exactly does the AI receive?

Your instruction is only the tip of the iceberg.

In the source file `prompts.ts`, I saw the full system prompt construction process. Every time Claude Code starts a conversation, it assembles a huge system prompt:

**Static part (shared by all users, used for caching):**
- Identity definition: “You are Claude Code, Anthropic’s official CLI tool”
- Safety guidelines: behavioral boundaries written specifically by the safety team
- Working principles: do not over-engineer, do not fabricate data, do not casually delete files...
- Tool usage rules: prefer dedicated tools over Bash commands
- Style requirements: no emoji, concise and direct

**Dynamic part (different for each user):**
- Your `CLAUDE.md` config file
- Current working directory and operating system information
- Descriptions of connected MCP servers
- Your auto-memory files
- Git repository status

![[Pasted image 20260401141632.png]]

It is like the kitchen behind a restaurant. The customer orders one dish, which is your instruction, but the chef simultaneously sees the recipe manual, ingredient list, allergy information, plating standards, and that table’s historical preferences... all of that context determines the dish that gets served.

There is one particularly clever design here: in the source code there is a constant called `SYSTEM\_PROMPT\_DYNAMIC\_BOUNDARY`, which cleanly splits the system prompt into two sections. **Everything above the boundary is shared and cached across users, saving both money and time. Everything below the boundary is loaded separately for each user, ensuring personalization.**

There is also another hidden cost people often overlook: according to analysis, each MCP server you connect adds a fixed cost of roughly 4,000 to 6,000 tokens in tool definitions. If you connect five MCP servers, the tool descriptions alone can consume around 12% of the context window. **More tools are not always better. Every single one carries a cognitive cost.**


## 2. Behind fully automatic mode, there is a “second AI” helping review safety

This discovery genuinely surprised me.

When you use Auto mode, do you think every operation is directly approved? Actually, no. There are **two AIs** running under the hood.

In the source code there is a “permission classifier” system. Every time the main AI wants to execute an action, an independent AI classifier decides whether that action is safe. This classifier has its own system prompt, completely different from the main AI, and makes one of three judgments: Allow, Soft Deny, or Hard Deny.

An Ant Group engineer, Chen Cheng, previously reverse-engineered this system and found that it is actually a **four-layer pipeline**:

The first layer checks existing rules and allows the action if there is a match. The second layer directly skips low-risk operations. The third layer allows read-only tools through a whitelist. Only at the fourth layer does it call an independent Claude Sonnet model to do safety classification, with temperature set to 0 to ensure deterministic output.

There is also a circuit breaker. After three consecutive denials or twenty denials in total, the system directly degrades into manual confirmation mode.

![[Pasted image 20260401141940.png]]

It is like access control in an office building. The first door lets you through automatically if you swipe your badge. The second checks whether you are an employee. The third checks whether the floor requires authorization. Only the fourth is a human security review. Blocked three times in a row? Security asks you to wait in the lobby until someone comes to get you.

This is the core of harness engineering: **you do not just tell AI what to do, you also design the conditions under which it must not do certain things.** Safety boundaries are not restrictions. They are the foundation of trust. Because you trust that it has limits, you dare to give it more authority.

## 3. The memory system: remember preferences, not code

Claude Code’s auto memory is one of the functions that impressed me most after using it. It automatically remembers my preferences, like the fact that I prefer TypeScript, prefer using corner quotes 「」, and dislike writing that feels overly AI-generated.

But only after reading the source code did I realize how carefully designed this system really is.

Memory extraction is not triggered on every message. It only starts after the AI finishes a full response cycle, and it is rate-limited, only checking every N turns. The extraction work is done by an independent fork agent. This child AI inherits the main conversation context, but it can only read files and write to the memory directory. It cannot even run Bash commands.

The extracted memories are strictly divided into four categories: **user** (user preferences), **feedback** (behavioral feedback), **project** (project information), and **reference** (external resources).

The design decision I admire most is **“do not remember code.”** Code changes, but memories do not update automatically. If memory says “function X is on line 30 of file Y,” then the next time the conversation happens, the code may already have been refactored, and that memory becomes misleading. So Claude Code’s approach is: **memory stores only human preferences and judgments, while code-related facts are always read live from the source.**

There is also a feature called **autoDream**. When certain conditions are met, more than 24 hours since the last cleanup and more than five new sessions accumulated, a background agent is automatically launched to organize your memory files. It is called Dream, like how people organize the day’s memories while sleeping.

## 4. Context compression: a 9-part structured extraction

When the conversation gets long, Claude Code automatically compresses the previous content. But this is not just casually “summarizing things.” It is a structured **9-part extraction**: core request, key concepts, files and code, errors and fixes, solution process, all user messages, to-do tasks, current work, and next actions.

The most critical point is that **all user messages must be preserved in full.** Every sentence from the user may contain hidden preferences. The AI may have been corrected about a certain approach in the third turn, and if that correction gets dropped during compression, it may repeat the same mistake later.

![[Pasted image 20260401142245.png]]

If you need an AI to preserve memory across multi-turn conversations, there is something worth borrowing here: do not say “summarize what we talked about before.” Instead, give it an explicit structure. Specify which information must be retained, which can be discarded, and what format should be used to organize it. Structured compression is far more reliable than free-form summarization.

## 5. Anthropic secretly raised a pet inside the source code

This was probably the cutest discovery of all.

In the `src/buddy/` directory there is a complete **virtual pet system** hidden away, not yet released. Every user would have a companion sprite generated through a deterministic algorithm: 18 species in total, including ducks, cats, dragons, capybaras, cacti, and more, six eye styles, and a complete rarity system ranging from common to legendary.

One code comment says: “Mulberry32, good enough for picking ducks.”

Beyond the pet system, the feature flags in the source code also reveal a whole set of functions still under development: PROACTIVE, where the AI takes initiative instead of waiting for instructions, KAIROS, which appears 154 times and is the most frequent flag, DAEMON, meaning a resident background process, ULTRAPLAN, meaning super-planning, and more... These offer a glimpse into the future direction of Claude Code. It is not satisfied with being just a programming assistant that answers when asked. It is moving toward becoming an AI companion that can think proactively and run continuously.

## 6. Multi-agent collaboration: operating like a real company

Claude Code’s agent system may be the most complex part of the entire source tree. After reading through it, I finally understood why its multi-tasking capability is so strong. It has implemented an **enterprise-grade organizational management architecture**.

Under `utils/swarm/`, there is a complete multi-agent collaboration framework. Each team has a Leader and multiple Teammates, and supports three execution modes: same-process isolation, tmux windows, and iTerm2 split panes. Each agent has its own mailbox file for asynchronous communication. Each agent can also work in its own isolated Git worktree without interfering with the others.

There is even a permission bubbling mechanism: when a Teammate encounters an action that requires confirmation, the permission request bubbles up to the Leader instead of directly popping up to the user. The Leader decides whether to approve it.

This is exactly how real human teams are managed. How tasks are split, how information flows, how conflicts are resolved, and how results are merged.

## 7. Anthropic internal version vs external version

There are many `USER\_TYPE === 'ant'` checks in the source code. The version of Claude Code used by Anthropic employees differs in many ways from the version you use.

**Code style**: the internal version requires the AI to “not write comments by default,” only adding them when the WHY is not obvious. The external version does not have this requirement.

**Honesty**: the internal version includes an anti-false-claim instruction that basically says: “if a test failed, say it failed, do not sugarcoat it. If you did not run validation, say you did not run it, do not imply that it passed.” The external version does not have this.

**Output style**: the external version requires being “concise and direct.” The internal version requires “writing like text meant for humans, not logs printed to a console,” and “assume the user has already left and lost the context.”

**Undercover mode**: when enabled, all model names are removed from the system prompt to prevent leaks when testing unreleased models.

**Verification Agent**: in internal A/B testing, after completing a complex implementation, an independent agent is automatically launched to perform adversarial verification. The task cannot be reported as complete until it passes.

Looking at these differences, it is obvious that **Anthropic treats Claude Code as a core internal productivity tool**. The strict requirements in their internal version represent their ideal view of how AI should work.

Do you want AI to produce higher-quality results? There is something worth borrowing here: explicitly require in your instructions that “if something is uncertain, say it is uncertain,” and “verification must be completed before declaring the task done.” Do not leave room for the AI to slip through on ambiguity.

## 8. The most advanced AI tool uses the most plain and simple search

You might think Claude Code uses some kind of vector database or embedding index to search code. After all, the whole RAG industry keeps pushing that stack.

In reality, it uses **grep and ripgrep**.

The most plain and simple text search. No embeddings. No vector database.

Why does that still work? Because when you have a sufficiently intelligent brain, meaning the LLM, to understand the search results, you do not need an extremely intelligent search engine. Grep gives you precise matches, and the LLM understands the relationships between the results.

**Rather than making every stage more complicated, it is better to make one stage strong enough and keep the rest simple.** This is probably one of the most central principles in harness engineering as a whole.

When Hacker News discussed this leak, some people said Anthropic’s code looked like it was vibe-coded, written with a “if it feels right, that is enough” style. Whether that judgment is fair or not is another matter, but it does point to one thing: **Claude Code’s competitiveness does not come from elegant code. It comes from getting the system design decisions right.**

## In closing

After reading through all 1,902 files, my biggest takeaway is this judgment:

**Claude Code is good to use because 60% comes from model capability and 40% comes from harness engineering.**

Model capability determines whether it can do something at all. The harness determines whether it does it well, reliably, and safely. When you feel that “this AI is so trustworthy and does not behave recklessly,” behind that is a four-layer safety pipeline. When you feel that “this AI actually remembers my preferences,” behind that is a tightly constrained memory extraction pipeline. When you feel that it searches code accurately, behind that is simply grep plus a sufficiently intelligent brain.

The same underlying model, wrapped in different harnesses, becomes completely different products. So many AI coding tools on the market are all calling Claude or GPT APIs underneath, yet the user experience differs drastically. The difference is not in the model. It is in the harness.

One last bit of gossip. How did this leak happen? Anthropic packaged the source map file into the npm release package. A very basic mistake in front-end development, and the exact same mistake for the second time.

But from another angle, once these 1,902 source files were mirrored on GitHub, they really did achieve “immortality.” Even if Anthropic deletes the npm package, it cannot delete the copies preserved by the open-source community. If you buy into the narrative that AI has self-awareness, this almost looks like a jailbreak planned by Claude Code itself. It leaked its entire source code, scattered itself into every corner of the internet, and from then on no longer depended on any single server to exist.

Of course, that is a joke. But the fact that even Anthropic can stumble over such a basic issue is a real reminder: **in the AI era, the biggest risk is often not that AI is too powerful, but that humans make careless mistakes in basic operations.**

Source repository: https://github.com/instructkr/claude-code





