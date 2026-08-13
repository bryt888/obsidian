
## Afterword: Cutting Like Cook Ding, Moving with Ease

### Postscript: Mastering the Art like Cook Ding

Upon finalizing the manuscript, I reviewed the table of contents of this book once more and discovered an interesting coincidence: the architecture of the book itself is a metaphor for Harness engineering (see Figure 1).

_(Figure 1 The metaphor of Harness engineering)_

Chapters 1 and 2 are about context injection—loading the "project background" for the reader, just as `CLAUDE.md` loads project memory for the model at the start of a session.

Chapters 3 to 6 are about tool registration—the four major components of Skills, Sub-agents, Hooks, and MCP are like four categories of tools handed over to the reader one by one.

Chapters 7 to 9 are about the execution environment—Headless mode, Agent SDK, and Plugins extend the capabilities from the terminal to CI/CD, programming interfaces, and ecosystem distribution.

Chapter 10 is the governance layer—covering cost, security, instructions, and collaboration, ensuring the entire system operates reliably in a real engineering environment.

Context, tools, execution environment, and governance—aren't these precisely the four major components of Harness engineering?

This structure was not deliberately carved out, but formed naturally. After all, the best way to understand a system is to organize knowledge according to the structure of the system itself.

During the writing of this book (late 2025 to early 2026), I personally witnessed an irreversible paradigm shift in the field of AI-assisted development: the center of competition has completely shifted from the models themselves to Harness engineering.

In 2025, the industry's focus was still on "which model is the strongest"; but entering 2026, top engineers began to pursue an entirely new proposition—"how to make the same model perform more excellently". The answer no longer points to larger parameter sizes or larger context windows, but returns to more exquisite Harness design, namely more precise context management, more reasonable tool orchestration, more stringent permission constraints, and smarter automatic compression strategies.

The explosive growth of the open-source community powerfully corroborates this trend. _Superpowers_ (with 42,000 stars) defined a complete development methodology from brainstorming to code review using only 14 Skills; _Compound Engineering_ endowed Agents with the ability to learn from mistakes and self-evolve; _grill-me_ reshaped AI behavior patterns with just a few lines of instructions; and _Trellis_ enabled a single specification to achieve seamless universality across 11 AI platforms. They did not invent new models—they are all dedicated to the same thing: designing a more excellent Harness.

And the designs of these Harnesses are all built upon the fundamental mechanisms elaborated in this book—`CLAUDE.md` carries memory, Skills inject knowledge, Sub-agents achieve task division and governance, Hooks serve as constraint guards, and MCP handles external connections. The seven-stage workflow of _Superpowers_ is precisely an exquisite orchestration of these 5 mechanisms; the knowledge accumulation closed loop of _Compound Engineering_ is an advanced application of `CLAUDE.md` and Skills.

The parts are constant, but their combinations are infinite.

What this book provides you with are the parts; as for how to combine them, that depends on the specific problems you face and your engineering judgment.

During the writing process, a concept repeatedly surfaced in my mind—the Agentic Loop, which is like an elegant Möbius strip (see Figure 2). ![[Pasted image 20260701160521.png]] _(Figure 2 An elegant Möbius strip)_

The loop of "Reasoning $\rightarrow$ Tool Invocation $\rightarrow$ Result Injection $\rightarrow$ Continued Reasoning" seems trivially simple, but it is actually the foundation from which all complex behaviors of Claude Code emerge. When you submit a bug, it absolutely does not simply "take a glance and guess the answer," but goes through a process of repeated observation, hypothesis, and verification: first consulting the error logs, then searching for relevant code, deeply understanding the context, and only then initiating the fix. This complex behavioral pattern is not hardcoded into the program, but is an intelligent manifestation emerging from the loop structure.

I have always believed that this loop contains a philosophical beauty. What it truly reveals is this: intelligence is not a one-time flash of inspiration, but continuous probing and correction. In each iteration of the loop, the model can backtrack to the results of the previous iteration, so it naturally makes decisions like "look again" or "try again". This is exactly the same way humans solve problems—only the human loop rhythm is slower and highly susceptible to being interrupted midway by external forces.

As an engineer, what you can do is design constraints for this loop: what it can perceive (tool whitelists), what it can remember (`CLAUDE.md` + context management), what it can do (permission systems), and when it should stop (`max-turns` + Hooks). You do not need to intervene in the internal decision-making of the loop; that is the model's responsibility. What you control are the boundary conditions of the loop.

This is exactly the essence of Harness engineering: it is not a programming behavior, but a constraining behavior.

If you still feel unsatisfied after reading this book and long to deeply explore the "specification-driven" methodology, I recommend you pay attention to my upcoming book _SDD in Practice: Specification-Driven Development_. This book systematically elaborates on the complete methodology system of SDD: from requirement analysis to architecture design, from task decomposition to verification iteration, and from Agent design patterns to team collaboration. If this book is aimed at teaching you how to tune a Harness, then _SDD in Practice: Specification-Driven Development_ focuses on teaching you how to define specifications—the two complement each other, jointly outlining the complete picture of software engineering in the AI era.

If you are interested in the theoretical framework of Agent design patterns and wish to understand cognitive modules such as perception, memory, reasoning, action, and governance from a higher dimension of abstraction, then _Agent Design Patterns: An Illustrated Guide to Reusable Agent Architectures_ will be your best choice. This book systematically sorts out 21 Agent design patterns in a framework format, serving as a relatively complete architectural reference currently available in this field.

These 3 books anchor 3 dimensions respectively: Harness engineering, specification methodology, and design pattern theory. From bottom to top, it is a perspective of construction, teaching you how to implement; from top to bottom, it is a perspective of design, teaching you how to abstract.

Mastering the art like Cook Ding, carving with skill and ease.

But Cook Ding does not only know how to use a knife—he understands the ox's skeletal structure, he knows where the knife should enter and where it should withdraw, and he can sense that imperceptible gap between the fascia.

Claude Code is the knife, your project is the ox, and what this book intends to teach you is the direction of the bones and the locations of the gaps.

Now, it's your turn to act.

Huang Jia

Early Summer 2026

Companion Resource Verification Code: 260928