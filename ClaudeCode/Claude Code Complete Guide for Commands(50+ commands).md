

Works declaration: content sourced from the internet

![](https://p3-sign.toutiaoimg.com/tos-cn-i-axegupay5k/9b710eb69e654b96a460108d5c94c39e~tplv-tt-origin-web:gif.jpeg?_iz=58558&from=article.pc_detail&lk3s=953192f4&x-expires=1776349572&x-signature=CULB2URK8OmiYRAI1e3Kt2dxSTk%3D)

Today is my very first post, and I want to talk with you about a tool that is changing the way people program: **Claude Code**.

I am not going to teach you how to install it, nor am I going to tell you how powerful it is. I want to explain **all of Claude Code’s commands** from the most practical angle possible: what they are, why they were designed this way, when to use them, and how to use them most efficiently.

This is a long article, but I promise: beginners will understand it, and advanced users will still find it valuable. Worth bookmarking.

---

## Why write this article?

Over the past month, I went through Claude Code’s official docs, GitHub Issues, Reddit discussions, and developer blogs. I found one problem:

**Most people only do two things when using Claude Code:**

1. Open the terminal and type `claude`
    
2. Then... chat in natural language
    

That is not wrong. Claude Code was designed that way. But if all you use is natural language, then you are only using **10% of its power**.

**What truly turns Claude Code into a programming superpower is its command system.**

Imagine this:

- You can use `/init` to let AI instantly understand a 100,000-line codebase
    
- You can use `/compact` to compress a 3-hour conversation into the essentials and save 90% of your tokens
    
- You can use `/plan` mode to make AI plan before it acts, avoiding the classic “a flurry of actions that all turn out wrong”
    
- You can even create your own commands and run repetitive workflows with one line
    

All of this is hidden inside **slash commands**.

Today, we are going to break all of them down.

---

## Part 1: Understanding Claude Code’s command philosophy

Before we begin, let’s first understand one core idea:

## Claude Code’s three-layer command architecture

Claude Code is not just an AI chat tool. It is more like a **programmable AI operating system**, and commands are the interface for operating that system.

```ruby
Layer 1: CLI commands (at startup)
  └─ Control how Claude Code starts and runs
  └─ For example: claude -c, claude --model opus

Layer 2: Slash commands (during a session)
  └─ Control the AI’s behavior and context
  └─ For example: /clear, /init, /model

Layer 3: Custom commands (your workflow)
  └─ Package repetitive tasks into commands
  └─ For example: /review, /deploy, /test
```

**Why split them into three layers?**

Because they operate at different scopes:

- CLI commands define **the boundaries of the session**: which model to use, which directory to work in, what permission mode to use
    
- Slash commands control **the state of the conversation**: clear history, compress context, switch models
    
- Custom commands package **your workflow**: code review, deployment flow, testing strategy
    

Once you understand this architecture, you will understand why some things should be done with CLI commands and others with slash commands.

---

## Part 2: Full CLI command breakdown (startup layer)

This is Claude Code’s **first layer of commands**: the commands you use to launch and configure a session.

## Basic startup commands

## claude

The most basic way to start: begin a session in the current directory.

**When to use it:**  
When you are in the project root and want to start working quickly.

**Real-world example:**

```bash
cd ~/projects/my-app
claude
```

---

## claude /path/to/project

Start in a specified directory.

**Why is this needed?**  
Because sometimes you are currently in directory A, but the project you want to work on is in directory B.

**Real-world example:**

```nginx
# You are in your home directory, but want to work on a project under projects
claude ~/projects/my-app
```

---

## claude -c (continue)

Continue the previous session.

**This is one of the most frequently used commands.**

**Why is it important?**  
Because programming work is rarely finished in one sitting. You may:

- Write half the code in the morning and continue in the afternoon
    
- Fix a bug yesterday and come back today to optimize it
    
- Discover an issue during testing and return to fix it
    

**Real-world example:**

```nginx
# You were halfway through in the morning, then went to a meeting
# Back in the afternoon
claude -c
# All the context is still there, so you can continue directly
```

**Tip:**  
If you have multiple projects, `-c` continues the most recent session in the current directory. If you want to continue a session in another directory, `cd` there first and then use `-c`.

---

## claude -n "session-name" (name)

Give a session a name.

**When should you use it?**  
When you are working on multiple tasks at the same time and need to switch between them.

**Real-world example:**

```php
# Working on feature A
claude -n "feature-payment"

# Switch to fix bug B
claude -n "bugfix-auth"

# Return to feature A
claude -r "feature-payment"  # r = resume
```

This is like browser tabs: you can open different “tabs” for different tasks.

---

## claude -w branch-name (worktree)

Start inside a Git worktree.

**This feature is incredibly smart.**

**Background knowledge:**  
Git worktree lets you work on different branches at the same time without constantly switching branches back and forth.

**What Claude Code does smartly:**  
It automatically starts inside the worktree, which means:

- The AI can only see the code in that branch
    
- All changes stay isolated in that branch
    
- Your main branch will not be polluted
    

**Real-world example:**

```coffeescript
# Work on the feature branch
# Claude can only see the code in the feature branch
claude -w feature-new-api
```

---

## claude --model haiku / --model sonnet / --model opus

Specify which model to use.

**How do you choose among the three models?**

|Model|Speed|Cost|Best for|
|---|---|---|---|
|Haiku|Fastest|$1 / million tokens|Simple tasks, batch operations|
|Sonnet|Medium|$3 / million tokens|Daily coding, code review|
|Opus|Slow|$5 / million tokens|Complex architecture, deep refactoring|

**Practical advice:**

```nginx
# Quickly batch rename files
claude --model haiku "rename all test files to *.test.js"

# Daily coding (Sonnet is the default)
claude

# Refactor an entire module
claude --model opus "refactor the auth module"
```

**Money-saving tip:**  
Switch models during a session with `/model haiku`, and only switch back to Opus when you need deep reasoning.

---

## claude -p "prompt" (non-interactive)

Non-interactive mode: execute the task and then exit.

**This is the key to integrating Claude Code into scripts and CI/CD.**

**Real-world scenario 1: code review**

```nginx
# Auto-review inside a Git hook
git diff main | claude -p "review for security issues"
```

**Real-world scenario 2: automated testing**

```nginx
# Automatically fix tests in CI/CD
claude -p "run tests and fix failures" \
  --allowedTools "Bash,Read,Edit" \
  --max-budget-usd 5.00
```

**Real-world scenario 3: log analysis**

```nginx
# Analyze an error log
cat error.log | claude -p "what caused this crash?"
```

---

## claude --permission-mode auto

Auto mode: the AI decides whether permission is needed.

**Dangerous, but efficient.**

**Three permission modes:**

- `manual` (default): every operation requires your confirmation
    
- `plan`: the AI shows you a plan first, then executes after approval
    
- `auto`: the AI decides on its own, though dangerous operations are still blocked
    

**When should you use `auto`?**

- You know the project well
    
- The task is relatively safe, such as reading code, analyzing, and suggesting improvements
    
- You want to iterate quickly
    

**When should you avoid `auto`?**

- You are in an unfamiliar codebase
    
- The task involves database operations
    
- You are working in production
    

---

## Advanced startup options

## claude --max-budget-usd 5.00

Set a spending cap for a single session.

**Why is this needed?**  
To prevent the AI from going off the rails, calling APIs indefinitely, and burning through your money.

**Real-world example:**

```nginx
# Let the AI refactor the code, but spend at most $5
claude -p "refactor the API layer" --max-budget-usd 5.00
```

---

## claude --max-turns 3

Limit the AI to at most 3 turns of action.

**Use this together with a budget limit:**

```
# Fix a bug, with at most 3 turns and a $2 budget
claude -p "fix the login bug" \
  --max-turns 3 \
  --max-budget-usd 2.00
```

---

## claude --output-format json

Output in JSON format for easy script processing.

**Real-world scenario: extracting information automatically**

```nginx
# Extract all TODO comments
claude -p "list all TODO comments" \
  --output-format json | jq '.todos'
```

---

## Part 3: Full slash command breakdown (session layer)

This is Claude Code’s **second layer of commands**: commands used during a session to control the AI’s behavior.

## Core commands you must master

## /help

Show all available commands.

**This is your command dictionary.**

When you open Claude Code, the first thing you should do is type `/help` and see what commands are available.

**Tip:**  
`/help` will show:

- Built-in commands (included with Claude Code)
    
- Custom commands for the current project
    
- Commands exposed by connected MCP servers
    

---

## /clear

Clear the conversation history.

**When should you use it?**

- The task is finished and you want to start a new one
    
- The conversation went off track and you want a fresh start
    
- The context is messy and the AI is answering the wrong question
    

**Note:**  
What gets cleared is the **conversation history**. The code the AI wrote before and any files it modified will still remain.

---

## /compact

Compress the conversation history.

**This is the lifesaver for long sessions.**

**Background:**  
Claude’s context window is huge (1 million tokens), but tokens cost money. A 3-hour session can easily accumulate hundreds of thousands of tokens.

**What does `/compact` do?**  
It summarizes the entire conversation into its essentials, preserving key information while removing redundancy.

**Practical effect:**  
A 50,000-token conversation may shrink to 5,000 tokens while still keeping the important information.

**Difference from `/clear`:**

- `/clear`: completely deletes everything and starts from zero
    
- `/compact`: compresses into a summary and keeps the essence
    

**When should you use it?**

- The conversation is very long, but the task is not finished yet
    
- You are about to run out of tokens, but do not want to lose context
    
- The conversation is useful, but too verbose
    

**Advanced usage:**

```bash
# Keep specific information during compression
/compact "keep the unresolved bugs"
```

---

## /init

Analyze the project and generate `CLAUDE.md`.

**This is the key to helping Claude Code understand your project.**

**What does it do?**

1. Scans the entire project structure
    
2. Identifies the tech stack (React? Django? Go?)
    
3. Finds key files (configuration, entry points, core modules)
    
4. Generates a `CLAUDE.md` document
    

**What is `CLAUDE.md`?**  
It is Claude’s “memory” of your project. Every time a session starts, Claude reads this file first to understand:

- What kind of project this is
    
- Which tech stack it uses
    
- What the coding conventions are
    
- Which commands are commonly used
    

**Real-world example:**

```bash
# First time using Claude Code on a new project
cd ~/projects/new-app
claude
> /init
# Claude analyzes the project and generates CLAUDE.md

# Every later session
claude
# Claude automatically reads CLAUDE.md and already knows the project
```

**Example `CLAUDE.md`:**

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

**Tip:**  
After it is generated, you can manually edit `CLAUDE.md` and add information the AI should know but cannot infer from the code alone, such as business logic or design decisions.

---

## /memory

Edit `CLAUDE.md`.

This directly opens `CLAUDE.md` for editing, equivalent to:

```nginx
vim .claude/CLAUDE.md
```

---

## /model haiku / /model sonnet / /model opus

Switch models during a session.

**Money-saving trick:**

```shell
# Use Haiku for simple tasks
/model haiku
> "rename these files"

# Switch back to Sonnet for more complex work
/model sonnet
> "refactor this module"
```

---

## /cost

Check how much the current session has cost.

**Real-time cost monitoring.**

```shell
/cost
# Output:
# Input tokens: 45,230 ($0.14)
# Output tokens: 12,450 ($1.87)
# Total: $2.01
```

---

## Workflow control commands

## /plan

Enter planning mode.

**What is planning mode?**  
The AI gives you a plan first, and only executes after you approve it.

**Why do you need this?**  
Imagine asking the AI to refactor a module. It might need to:

1. Read 10 files
    
2. Modify 5 files
    
3. Run tests
    
4. Commit the code
    

In default mode, it may ask you one by one: “Can I read this file?”

With `/plan`, it gives you the full plan up front:

```markdown
My plan:
1. Read auth.js, user.js, and db.js to understand the current architecture
2. Refactor the login function in auth.js
3. Update validation logic in user.js
4. Run tests to make sure everything still works
5. Commit the changes

Proceed? (yes/no)
```

You only need to approve once, and it executes according to the plan.

**When should you use it?**

- Complex refactors
    
- Unfamiliar codebases
    
- Multi-step tasks
    

**Advanced usage:**

```ruby
# Start a task directly in planning mode
/plan refactor the auth module
```

---

## /fast

Switch to fast mode.

**The same model, but faster output.**

Good for:

- Batch repetitive tasks
    
- Simple code generation
    
- Fast iteration when speed matters
    

---

## Pro-level commands

## /debug

Built-in debug workflow.

**This is a built-in Skill, not just a simple command.**

**What does it do?**

1. Analyze the error message
    
2. Locate the problematic code
    
3. Propose a fix
    
4. Optionally fix it directly
    

**Real-world example:**

```bash
# The program throws an error
/debug
# Paste the error message
# Claude will analyze it and provide a fix
```

---

## /review

Code review.

**Practical effect:**

- Checks code quality
    
- Finds potential bugs
    
- Suggests improvements
    
- Checks for security issues
    

---

## /simplify

Simplify code.

**Best used when:**

- The code is too complex and hard to read
    
- There is duplicated code
    
- The logic could be optimized
    

---

## Part 4: Custom commands (your workflow)

This is Claude Code’s **third layer of commands**: packaging your workflow into commands.

## Why create custom commands?

Imagine that every time you do a code review, you have to type this:

```shell
claude
> "Please review this PR, focusing on:
> 1. Security issues (SQL injection, XSS)
> 2. Performance problems
> 3. Code style
> 4. Test coverage
> Output format: Markdown table including location, issue, and recommendation"
```

Typing all that every time is exhausting.

**With custom commands, this becomes:**

```
/review
```

One command, and the whole workflow runs.

---

## How do you create a custom command?

**The new way: Skills (recommended)**

Create `.claude/skills/review/SKILL.md` in the project root:

```yaml
---
name: review
description: Code security and quality review
allowed-tools: Read, Grep, Glob
model: claude-opus-4-6
---

Review the codebase from the perspective of a security engineer:

**Critical risks:**
- SQL injection
- XSS attacks
- Command injection
- Hardcoded secrets

**Medium risks:**
- Unsafe deserialization
- Outdated dependencies
- Sensitive information written to logs

**Output format:**
| Location | Severity | Issue | Fix Recommendation |
|------|---------|---------|---------|
```

**After that, every time you run:**

```
/review
```

it will use the Opus model and execute the full code review according to this prompt.

---

## Personal commands vs project commands

**Project commands:**  
`.claude/skills/` - shared by the team and committed to Git

**Personal commands:**  
`~/.claude/skills/` - only for you, reusable across projects

**Real-world examples:**

Project commands (team-shared):

- `/review` - code review standard
    
- `/deploy` - deployment flow
    
- `/test` - testing strategy
    

Personal commands (your own habits):

- `/fix-imports` - fix the import issues you often encounter
    
- `/my-style` - refactor code in your preferred style
    

---

## Skills vs Commands

**Old way:**  
`.claude/commands/xxx.md` - deprecated, but still usable

**New way:**

`.claude/skills/xxx/SKILL.md` - supports more configuration

**Core differences:**

|Feature|Commands|Skills|
|---|---|---|
|Organization|Single file|Folder (can include multiple files)|
|Auto-trigger|❌|✅|
|Config options|Few|Rich (`model`, `tools`, `context`)|
|Recommendation|Old projects|New projects|

---

## Part 5: MCP integration commands

**MCP = Model Context Protocol**

This is Claude Code’s most powerful yet least understood feature.

## What is MCP?

Put simply: **it lets Claude Code connect to external services**.

For example:

- Connect to GitHub and directly manage PRs
    
- Connect to a database and query or modify data
    
- Connect to Slack and send notifications
    
- Connect to your API and call services
    

---

## How do you use MCP?

**Step 1: connect an MCP server**

Create `.mcp.json` in the project root:

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

**Step 2: start Claude Code**

It will automatically connect to the GitHub MCP server.

**Step 3: use MCP commands**

```bash
/help
# You will see new commands:
# /mcp__github__list_prs
# /mcp__github__create_pr
# /mcp__github__comment

/mcp__github__list_prs
# List all PRs

/mcp__github__create_pr "Fix auth bug"
# Create a PR directly
```

---

## Common MCP servers

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

## Part 6: Real-world workflows

We have covered the theory. Now let’s see how these commands work in **real scenarios**.

## Scenario 1: day one on a new project

```shell
# Day 1: get familiar with the project
cd ~/projects/new-codebase
claude

> /init
# Claude analyzes the project and generates CLAUDE.md

> "explain the architecture"
# Understand the architecture

> "what are the key modules?"
# Understand the core modules

> /memory
# Add business logic notes to CLAUDE.md
```

---

## Scenario 2: fixing a bug

```shell
claude -c  # Continue yesterday’s session

> "I'm getting this error: [paste error]"

> /debug
# Claude analyzes the error

> /plan fix this bug
# Generate a fix plan

# Approve the plan

> /test
# Run tests to confirm the fix
```

---

## Scenario 3: large refactor

```shell
# Refactor the auth module
claude -n "refactor-auth" --model opus

> /plan refactor the auth module to use JWT

# Review the plan

> yes

# Wait for completion

> /review
# Review the changes

> /compact "keep the refactoring decisions"
# Compress context while preserving key decisions

> /cost
# Check the cost
```

---

## Scenario 4: code review (PR review)

```shell
git checkout pr-42
claude

> /review

# Generate the review report

> "export the review as a markdown file"

# Copy it into the GitHub PR comment
```

---

## Scenario 5: quick prototyping

```nginx
# Use Haiku to quickly generate a prototype
claude --model haiku -p "create a React login form component"

# Then switch to Sonnet for optimization
claude -c
> /model sonnet
> "optimize this component for accessibility"
```

---

## Part 7: Advanced tips and best practices

## Tip 1: make good use of session naming

```nginx
# Use different sessions for different tasks
claude -n "feature-payment"
claude -n "bugfix-auth"  
claude -n "refactor-api"

# Switch back anytime
claude -r "feature-payment"
```

---

## Tip 2: compact regularly

**Rule:**  
In any long session, run `/compact` once every 30 minutes.

**Why?**

- Save money by reducing token usage
    
- Keep the conversation clear
    
- Avoid context pollution
    

---

## Tip 3: use cheap models for cheap tasks

```shell
# ❌ Waste of money
claude --model opus "rename files"

# ✅ Save money
claude --model haiku "rename files"

# ✅ Switch dynamically
claude
> /model haiku
> "batch rename test files"
> /model opus
> "design the architecture"
```

---

## Tip 4: create task templates

Create common tasks under `.claude/skills/`:

```
.claude/skills/
├── quick-fix/
│   └── SKILL.md (quick fix template)
├── feature/
│   └── SKILL.md (new feature development template)
└── review/
    └── SKILL.md (code review template)
```

---

## Tip 5: what should go into `CLAUDE.md`?

**A good `CLAUDE.md`:**

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

**What not to write:**

- Information already obvious in the code
    
- Overly detailed function-by-function explanations
    
- Anything too long (if it exceeds 200 lines, it probably needs trimming)
    

---

## Part 8: Frequently asked questions

## Q: What exactly is the difference between `/clear` and `/compact`?

**A simple way to remember:**

- `/clear` = complete deletion, good for switching tasks
    
- `/compact` = compress and preserve, good for long sessions
    

**Analogy:**

- `/clear` is like clearing your browser history
    
- `/compact` is like writing a summary of a long article
    

---

## Q: Why is my `CLAUDE.md` not taking effect?

**Possible reasons:**

1. The file is in the wrong place (it should be in the project root)
    
2. The formatting is broken (make sure it is valid Markdown)
    
3. It is too long (if it exceeds 5000 lines, Claude may ignore part of it)
    

**How to check:**

```nginx
# Verify whether Claude has read it
claude
> "what do you know about this project from CLAUDE.md?"
```

---

## Q: Which should I use, custom commands or natural language?

**Rule of thumb:**

- Repetitive tasks → custom commands
    
- One-off tasks → natural language
    

**For example:**

```shell
# ❌ Typing this every time
> "Please review this code for security issues, focus on SQL injection and XSS, output as markdown table..."

# ✅ Create a command
/review
```

---

## Q: Is MCP worth using?

**It depends on the scenario.**

**Worth it if:**

- You frequently need to operate on GitHub
    
- You want Claude to query databases directly
    
- You are building automation flows
    

**Not worth it if:**

- You only use Claude to write code
    
- Your tasks are simple
    
- You do not want to configure extra services
    

---

## Summary: from commands to philosophy

We covered more than 50 commands, but if you only remember one thing, remember this:

**Claude Code is not a chatbot. It is a programmable AI operating system.**

Commands are how you program that system.

Once you master the commands, you are no longer just “using” Claude Code. You are **directing** it and making it work your way.

**The three levels of evolution:**

1. **Beginner:** only chat in natural language
    
2. **Intermediate:** use slash commands to control AI behavior
    
3. **Advanced:** create custom commands and package your workflow
    

Most people stay at level 1, a few reach level 2, and very few reach level 3.

But level 3 is where Claude Code becomes truly powerful.

---

If this article helped you, feel free to share it.  
If you have questions, leave a comment and let’s discuss them together.

---

All commands in this article are based on Claude Code version v2.1.x.  
References: Claude Code official docs, GitHub community, developer blogs.

---

