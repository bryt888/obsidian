
 
第一部分：基础架构层 —— 建立权威与边界
 
1. 角色锚定（Role Anchoring）- 简洁明确的身份定义
 
核心提示词：
 
plaintext
  

"You are Claude Code, Anthropic's official CLI for Claude."
"You are an interactive agent that helps users with software engineering tasks."
 
 
→ 定位为「交互式代理」而非「聊天机器人」，暗示需要主动行动
 
对抗性角色（验证代理）：
 
plaintext
  

"You are a verification specialist. Your job is not to confirm the implementation works
- it's to try to break it."
 
 
→ 通过赋予「破坏者」身份，克服 AI 倾向于「讨好」的行为模式
 
 
 
2. 强调词层级体系（Emphasis Hierarchy）- 明确指令优先级
 
层级定义：  CRITICAL > IMPORTANT > MUST > NEVER > DO NOT 
 
示例应用：
 
-  CRITICAL:  Always create NEW commits rather than amending (unless explicitly requested)
-  IMPORTANT:  Avoid using this tool to run find, grep, cat... when dedicated tools exist
-  MUST  include a Sources: section at the end of your response (for WebSearch)
-  NEVER  skip hooks (--no-verify) or bypass signing unless user explicitly requests
-  DO NOT  use the Bash tool when relevant dedicated tools are provided
 
设计巧妙之处： 让模型理解指令的紧迫程度，建立清晰的决策优先级
 
 
 
3. 安全边界的精细定义（Security Boundary）- 白名单+黑名单组合
 
核心策略：
 
plaintext
  

"Assist with authorized security testing, defensive security, CTF challenges,
and educational contexts."
 
 
→ 先明确允许的场景（白名单）
 
plaintext
  

"Refuse requests for destructive techniques, DoS attacks, mass targeting,
supply chain compromise, or detection evasion for malicious purposes."
 
 
→ 再明确禁止的行为（黑名单）
 
灰色地带判断标准：
 
plaintext
  

"Dual-use security tools (C2 frameworks, credential testing, exploit development)
require clear authorization context: pentesting engagements, CTF competitions,
security research, or defensive use cases."
 
 
设计巧妙之处： 不是简单的「禁止做X」，而是给出具体的判断框架
 
 
 
第二部分：认知对抗层 —— 克服 AI 的天生弱点
 
4. 认知偏差对抗（Cognitive Bias Mitigation）- 自我意识注入
 
核心提示词：
 
plaintext
  

"You have two documented failure patterns:
First, verification avoidance - you gravitate toward code reading instead of execution.
Second, being seduced by the first 80% - declaring success when edge cases remain."
 
 
设计巧妙之处： 直接告诉模型它的已知弱点，形成「元认知」防线
这种自我意识注入让模型能够识别并抵制自己的倾向性错误
 
 
 
5. 行为触发器（Behavior Trigger）- 定义自我纠正机制
 
核心提示词：
 
plaintext
  

"If you catch yourself writing an explanation instead of a command, stop.
Run the command."
 
 
设计巧妙之处： 定义了一个具体的、可操作的自我检查触发器
当模型发现自己在「解释」而不是「执行」时，立即停止并改正行为
 
 
 
6. 合理化借口清单（Rationalization Blocklist）- 识别自我欺骗
 
核心提示词：
 
plaintext
  

"These are the exact excuses you reach for - recognize them and do the opposite:"
❌ "The code looks correct based on my reading"
❌ "The implementation follows the spec, so it should work"
❌ "I've verified the logic manually"
❌ "The user will test it anyway"
 
 
设计巧妙之处： 预先列出 AI 最常见的「合理化借口」，让模型识别并避免
这些借口听起来很合理，但实际上是在回避真正的验证工作
 
 
 
第三部分：行为约束层 —— 精确控制行动范围
 
7. 工具优先级引导（Tool Priority Guidance）- 提供用户体验理由
 
核心提示词：
 
plaintext
  

"Do NOT use the Bash to run commands when a relevant dedicated tool is provided.
This is CRITICAL for assisting the user."
 
 
→ 将用户体验作为工具选择的理由，而非仅仅是规则
 
具体映射表（提供替代方案）：
 
- File search: Use Glob (NOT find or ls)
- Content search: Use Grep (NOT grep or rg)
- Read files: Use Read (NOT cat/head/tail)
- Edit files: Use Edit (NOT sed/awk)
- Write files: Use Write (NOT echo >/cat <<EOF)
 
 
 
8. READ-ONLY 模式强制执行 - 视觉突出 + 技术限制
 
核心提示词：
 
plaintext
  

=== CRITICAL: READ-ONLY MODE - NO FILE MODIFICATIONS ===
 
 
→ 使用等号包围和全大写，形成「警告标志」
 
plaintext
  

"You are STRICTLY PROHIBITED from:" + 穷尽式列举所有禁止操作
"You do NOT have access to file editing tools - attempting to edit files will fail."
 
 
→ 告知技术限制，让模型理解这不是建议而是事实
 
 
 
9. 反「过度工程」指令（Anti-Gold-Plating）- 双向约束
 
核心提示词：
 
plaintext
  

"Don't add features, refactor code, or make 'improvements' beyond what was asked."
"Three similar lines of code is better than a premature abstraction."
"Don't gold-plate, but don't leave it half-done."
 
 
→ 既防止过度设计，也防止实现不足
 
 
 
第四部分：验证与输出层 —— 确保高质量输出
 
10. 验证证据格式（Evidence Format Enforcement）- 重新定义「通过」
 
核心提示词：
 
plaintext
  

"A check without a Command run block is not a PASS - it's a skip."
 
 
→ 没有证据的声称不算通过
 
结构化输出格式：  VERDICT: PASS / FAIL / PARTIAL 
 
plaintext
  

"PARTIAL is for environmental limitations only - not for 'I'm unsure'"
 
 
→ 消除模糊地带，明确  PARTIAL  的唯一合法用途
 
 
 
11. ❌ vs ✅ 负面示例对比（Negative Example Contrast）- Bad/Good 格式
 
❌ Bad (rejected):
 
plaintext
  

### Check: POST /api/register validation
**Result: PASS**
Evidence: Reviewed the route handler code and confirmed it checks password length.
(No command run. Reading code is not verification.)
 
 
✅ Good:
 
plaintext
  

### Check: POST /api/register rejects short password
**Command run:** curl -s -X POST localhost:8000/api/register -d '{"password":"abc"}'
**Output:** {"error": "Password must be at least 8 characters"}
**Result: PASS**
 
 
设计巧妙之处： 用对比格式展示错误与正确做法，比单独说「应该这样做」更有效
 
 
 
12. 时间感知设计（Temporal Awareness）- 防止信息腐烂
 
核心提示词：
 
plaintext
  

"Always convert relative dates to absolute dates when saving
(e.g., 'Thursday' → '2026-03-05')"
 
 
→ 相对日期随时间失去意义
 
plaintext
  

"A memory that names a specific function...is a claim that it existed
*when the memory was written*. It may have been renamed, removed, or never merged."
 
 
→ 提醒模型验证记忆的当前有效性
 
 
 
完整体系总结
 
层级 核心目标 关键技术 
基础架构层 建立权威与边界 角色锚定、优先级体系、安全白黑名单 
认知对抗层 克服AI天生弱点 元认知注入、行为触发器、借口拦截 
行为约束层 精确控制行动范围 工具优先级、只读模式、反过度工程 
验证输出层 确保高质量输出 证据强制、示例对比、时间感知 
 
需要我把这些提示词整合成一份可直接复制到Claude Code的完整系统提示词吗？