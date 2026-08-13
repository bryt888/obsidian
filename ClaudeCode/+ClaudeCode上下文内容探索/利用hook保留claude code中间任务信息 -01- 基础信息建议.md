可以。你现在要先保留的不是“所有东西”，而是**足够还原 Claude Code 为什么消耗 Context 的关键记录**。

## 一、建议保留的记录类型

### 1. 完整 transcript 快照

用于看某个时间点，Claude Code 的会话里已经积累了什么内容。

建议在这些事件保存：

```text
UserPromptSubmit
Stop
PreCompact
PostCompact
SubagentStop
```

用途：

```text
UserPromptSubmit：一轮任务开始前的上下文
Stop：一轮任务结束后的上下文
PreCompact：压缩前的上下文
PostCompact：压缩后的上下文
SubagentStop：子 Agent 的完整执行记录
```

---

### 2. Hook 输入记录

每次 hook 被触发时，保存 `hook_input.json`。

里面通常包括：

```text
hook_event_name
session_id
transcript_path
tool_name
tool_input
agent_type
agent_id
agent_transcript_path
```

用途是知道：

```text
什么时候触发
哪个 session
哪个 tool
哪个 agent
对应 transcript 文件在哪里
```

---

### 3. 工具调用轻量日志

`PostToolUse` 不建议每次复制完整 transcript，只做轻量记录。

记录内容：

```text
时间
session_id
agent_type / agent_id
tool_name
tool_input
tool_result 字符数
tool_result 前几百字
tool_result 后几百字
是否超过阈值
```

如果 `tool_result` 超过 20KB 或 50KB，再额外保存一次完整 transcript。

用途：

```text
找出哪个工具输出最大
哪个文件被反复 Read
Grep / Bash 是否输出太多
哪个 subagent 最费 context
```

---

### 4. Subagent 记录

重点保留：

```text
SubagentStart：轻量记录
SubagentStop：完整保存 agent_transcript
```

用途：

```text
确认 code-explorer 是否运行
确认用了哪个 agent
确认是否用了 haiku / sonnet
分析 subagent 调了哪些工具
分析 subagent 是否产生大量输出
```

---

### 5. Compact 前后记录

这个很重要。

保存：

```text
PreCompact transcript
PostCompact transcript
compact_summary
```

用途：

```text
看 compact 前 context 为什么大
看 compact 后保留了什么
看哪些内容被压缩掉
```

---

### 6. CHANGELOG_ENTRY 文档追加记录

这是另一类，不是为了分析 context，而是为了项目记忆。

推荐让 Claude 每轮结束时输出很短的固定块：

```text
CHANGELOG_ENTRY:
- Goal:
- Files changed:
- Key changes:
- Tests:
- Issues:
- Follow-up:
END_CHANGELOG_ENTRY
```

Hook 只负责把这段 append 到 log 文件。

注意：

```text
不要让 hook 再调用模型
不要让 hook 读取旧 log
只 append
```

---

## 二、最终推荐保留清单

你可以先按这个最小可用方案做：

```text
完整快照：
- UserPromptSubmit
- Stop
- PreCompact
- PostCompact
- SubagentStop

轻量日志：
- PostToolUse
- SubagentStart
- PostToolUseFailure

文档追加：
- Stop 时提取 CHANGELOG_ENTRY，append 到日志
```

暂时不需要：

```text
PreToolUse 每次完整快照
MessageDisplay
Notification
SessionStart
```

---

## 三、建议目录结构

```text
.claude/context-audit/
  snapshots/
    20260804-091300_UserPromptSubmit/
      hook_input.json
      transcript.jsonl

    20260804-091800_Stop/
      hook_input.json
      transcript.jsonl

    20260804-092000_SubagentStop_code-explorer/
      hook_input.json
      agent_transcript.jsonl

    20260804-092500_PreCompact/
      hook_input.json
      transcript.jsonl

    20260804-092530_PostCompact/
      hook_input.json
      transcript.jsonl
      compact_summary.txt

  tool-events/
    2026-08-04.jsonl

  changelog/
    2026-08.md
```

---

## 四、核心原则

```text
快照少保存，保存关键节点。
工具调用多记录，但只写轻量摘要。
大输出才额外保存完整 transcript。
Log 可以频繁 append，但不要读全文。
阶段总结一次性写完，然后 compact 或新开 session。
```

这套方案的目标是：  
**既能知道 Claude Code 到底做了什么，又不让记录系统本身制造更多 context 和文件噪音。**


---
>先按这个最小可用方案做. 给出我具体的设置文件和内容. 图片里显示的是我现有的设置. 如果需要披露哪个文件，我可以复制给你
>

可以。先做**最小可用方案**，你需要加 2 个东西：

```text
1. .claude/settings.local.json 里增加 hooks 配置
2. .claude/hooks/context_audit.py 脚本
```

Claude Code hooks 配置可以放在 `.claude/settings.local.json`，hook 命令会通过 stdin 接收 JSON 输入；官方文档也说明 hooks 按事件配置，常见事件包括 `UserPromptSubmit`、`PostToolUse`、`Stop`、`SubagentStop`、`PreCompact` 等。([Claude](https://code.claude.com/docs/en/hooks?utm_source=chatgpt.com "Hooks reference - Claude Code Docs"))

---

## 1. 先创建脚本文件

路径：

```text
D:\Accu\+AccuTravel\AccuTravel\.claude\hooks\context_audit.py
```

内容：

```python
import sys
import json
import shutil
import pathlib
import datetime
import hashlib

PROJECT_ROOT = pathlib.Path.cwd()
AUDIT_ROOT = PROJECT_ROOT / ".claude" / "context-audit"
SNAPSHOT_ROOT = AUDIT_ROOT / "snapshots"
TOOL_EVENT_ROOT = AUDIT_ROOT / "tool-events"
CHANGELOG_ROOT = AUDIT_ROOT / "changelog"

FULL_SNAPSHOT_EVENTS = {
    "UserPromptSubmit",
    "Stop",
    "PreCompact",
    "PostCompact",
    "SubagentStop",
}

LIGHT_EVENTS = {
    "PostToolUse",
    "SubagentStart",
    "PostToolUseFailure",
}

LARGE_RESULT_CHARS = 20000


def now_str():
    return datetime.datetime.now().strftime("%Y%m%d-%H%M%S-%f")


def today_str():
    return datetime.datetime.now().strftime("%Y-%m-%d")


def safe_name(value):
    if not value:
        return "unknown"
    return "".join(c if c.isalnum() or c in "-_." else "_" for c in str(value))[:80]


def read_stdin_json():
    raw = sys.stdin.read()
    if not raw.strip():
        return {}
    try:
        return json.loads(raw)
    except Exception:
        return {"_raw_stdin": raw}


def write_json(path, data):
    path.parent.mkdir(parents=True, exist_ok=True)
    path.write_text(json.dumps(data, ensure_ascii=False, indent=2), encoding="utf-8")


def append_jsonl(path, data):
    path.parent.mkdir(parents=True, exist_ok=True)
    with path.open("a", encoding="utf-8") as f:
        f.write(json.dumps(data, ensure_ascii=False) + "\n")


def file_size(path):
    try:
        return pathlib.Path(path).stat().st_size
    except Exception:
        return None


def copy_if_exists(src, dst):
    if not src:
        return False
    src_path = pathlib.Path(src).expanduser()
    if not src_path.exists():
        return False
    dst.parent.mkdir(parents=True, exist_ok=True)
    shutil.copy2(src_path, dst)
    return True


def short_text(value, limit=800):
    if value is None:
        return ""
    if not isinstance(value, str):
        value = json.dumps(value, ensure_ascii=False)
    if len(value) <= limit * 2:
        return value
    return value[:limit] + "\n...<TRUNCATED>...\n" + value[-limit:]


def extract_tool_result(data):
    for key in ["tool_response", "tool_result", "result", "output"]:
        if key in data:
            return data.get(key)
    return None


def make_snapshot(data, reason="normal"):
    event = data.get("hook_event_name", "unknown")
    session_id = safe_name(data.get("session_id", "no-session"))
    agent_type = safe_name(data.get("agent_type", ""))
    tool_name = safe_name(data.get("tool_name", ""))

    suffix_parts = [safe_name(event)]
    if agent_type != "unknown":
        suffix_parts.append(agent_type)
    if tool_name != "unknown":
        suffix_parts.append(tool_name)

    folder = SNAPSHOT_ROOT / f"{now_str()}_{'_'.join(suffix_parts)}_{session_id}"
    folder.mkdir(parents=True, exist_ok=True)

    data["_snapshot_reason"] = reason
    write_json(folder / "hook_input.json", data)

    transcript_path = data.get("transcript_path")
    agent_transcript_path = data.get("agent_transcript_path")

    if copy_if_exists(transcript_path, folder / "transcript.jsonl"):
        pass

    if copy_if_exists(agent_transcript_path, folder / "agent_transcript.jsonl"):
        pass

    compact_summary = data.get("compact_summary")
    if compact_summary:
        (folder / "compact_summary.txt").write_text(str(compact_summary), encoding="utf-8")

    return str(folder)


def log_tool_event(data):
    event = data.get("hook_event_name", "unknown")
    tool_name = data.get("tool_name")
    tool_input = data.get("tool_input")
    tool_result = extract_tool_result(data)

    if isinstance(tool_result, str):
        result_chars = len(tool_result)
    else:
        result_chars = len(json.dumps(tool_result, ensure_ascii=False)) if tool_result is not None else 0

    record = {
        "time": datetime.datetime.now().isoformat(timespec="seconds"),
        "event": event,
        "session_id": data.get("session_id"),
        "agent_id": data.get("agent_id"),
        "agent_type": data.get("agent_type"),
        "tool_name": tool_name,
        "tool_input": tool_input,
        "result_chars": result_chars,
        "large_result": result_chars >= LARGE_RESULT_CHARS,
        "result_preview": short_text(tool_result, 600),
        "transcript_path": data.get("transcript_path"),
        "agent_transcript_path": data.get("agent_transcript_path"),
    }

    append_jsonl(TOOL_EVENT_ROOT / f"{today_str()}.jsonl", record)

    if result_chars >= LARGE_RESULT_CHARS:
        make_snapshot(data, reason=f"large_tool_result_{result_chars}_chars")


def extract_changelog_from_transcript(transcript_path):
    if not transcript_path:
        return None

    path = pathlib.Path(transcript_path).expanduser()
    if not path.exists():
        return None

    try:
        lines = path.read_text(encoding="utf-8", errors="ignore").splitlines()
    except Exception:
        return None

    for line in reversed(lines[-30:]):
        try:
            obj = json.loads(line)
        except Exception:
            continue

        text = json.dumps(obj, ensure_ascii=False)
        start = text.find("CHANGELOG_ENTRY:")
        end = text.find("END_CHANGELOG_ENTRY")
        if start != -1 and end != -1 and end > start:
            chunk = text[start:end + len("END_CHANGELOG_ENTRY")]
            chunk = chunk.replace("\\n", "\n")
            return chunk

    return None


def append_changelog_if_any(data):
    if data.get("hook_event_name") != "Stop":
        return

    entry = extract_changelog_from_transcript(data.get("transcript_path"))
    if not entry:
        return

    out = CHANGELOG_ROOT / f"{today_str()}.md"
    out.parent.mkdir(parents=True, exist_ok=True)

    with out.open("a", encoding="utf-8") as f:
        f.write("\n\n---\n")
        f.write(f"\n## {datetime.datetime.now().isoformat(timespec='seconds')}\n\n")
        f.write(entry)
        f.write("\n")


def main():
    data = read_stdin_json()
    event = data.get("hook_event_name", "unknown")

    AUDIT_ROOT.mkdir(parents=True, exist_ok=True)

    if event in FULL_SNAPSHOT_EVENTS:
        make_snapshot(data)

    if event in LIGHT_EVENTS:
        log_tool_event(data)

    append_changelog_if_any(data)

    # 不阻断 Claude Code
    sys.exit(0)


if __name__ == "__main__":
    main()
```

---

## 2. 修改 `.claude/settings.local.json`

如果你现有文件里已经有 `"hooks"`，不要直接覆盖，需要合并。  
如果你想先快速测试，可以把 hooks 部分改成下面这样：

```json
{
  "hooks": {
    "UserPromptSubmit": [
      {
        "matcher": "",
        "hooks": [
          {
            "type": "command",
            "command": "python .claude/hooks/context_audit.py"
          }
        ]
      }
    ],
    "Stop": [
      {
        "matcher": "",
        "hooks": [
          {
            "type": "command",
            "command": "python .claude/hooks/context_audit.py"
          }
        ]
      }
    ],
    "PreCompact": [
      {
        "matcher": "",
        "hooks": [
          {
            "type": "command",
            "command": "python .claude/hooks/context_audit.py"
          }
        ]
      }
    ],
    "PostCompact": [
      {
        "matcher": "",
        "hooks": [
          {
            "type": "command",
            "command": "python .claude/hooks/context_audit.py"
          }
        ]
      }
    ],
    "SubagentStop": [
      {
        "matcher": "",
        "hooks": [
          {
            "type": "command",
            "command": "python .claude/hooks/context_audit.py"
          }
        ]
      }
    ],
    "PostToolUse": [
      {
        "matcher": "",
        "hooks": [
          {
            "type": "command",
            "command": "python .claude/hooks/context_audit.py"
          }
        ]
      }
    ],
    "SubagentStart": [
      {
        "matcher": "",
        "hooks": [
          {
            "type": "command",
            "command": "python .claude/hooks/context_audit.py"
          }
        ]
      }
    ],
    "PostToolUseFailure": [
      {
        "matcher": "",
        "hooks": [
          {
            "type": "command",
            "command": "python .claude/hooks/context_audit.py"
          }
        ]
      }
    ]
  }
}
```

如果你的 `.claude/settings.local.json` 里还有 `permissions`、`agents` 或其他配置，就只把 `"hooks": {...}` 这一段加进去，不要删原来的。

---

## 3. 生成的记录位置

运行后会自动生成：

```text
.claude/context-audit/
  snapshots/
    xxx_UserPromptSubmit_xxx/
      hook_input.json
      transcript.jsonl

    xxx_Stop_xxx/
      hook_input.json
      transcript.jsonl

    xxx_SubagentStop_code-explorer_xxx/
      hook_input.json
      transcript.jsonl
      agent_transcript.jsonl

  tool-events/
    2026-08-04.jsonl

  changelog/
    2026-08-04.md
```

---

## 4. 你应该把这个目录加入 `.gitignore`

```gitignore
.claude/context-audit/
```

因为里面会包含代码、对话、工具结果，可能有敏感信息。

---

## 5. 测试方法

保存后，在 Claude Code 里发一个很小的任务，例如：

```text
请只回复 OK，不要做其他操作。
```

然后看是否生成：

```text
.claude/context-audit/snapshots/
```

如果没有生成，下一步就检查：

```text
python .claude/hooks/context_audit.py
```

是否能运行。

你可以先把现有 `.claude/settings.local.json` 内容贴出来，我帮你把 hooks 合并进去，避免覆盖你原来的配置。



## 简述文件夹名字及内容

这三个目录作用不一样：

## 1. `snapshots/`

保存**关键时刻的完整快照**。

里面通常有：

```text
hook_input.json
transcript.jsonl
agent_transcript.jsonl   # 如果是 SubagentStop
compact_summary.txt      # 如果是 PostCompact
```

用途：

```text
看某一轮开始/结束时，Claude Code 的上下文里到底有什么
看 compact 前后变化
看 subagent 完整做了什么
分析 context 为什么变大
```

主要来自这些事件：

```text
UserPromptSubmit
Stop
PreCompact
PostCompact
SubagentStop
```

---

## 2. `tool-events/`

保存**工具调用的轻量日志**，通常是每天一个 `.jsonl` 文件。

例如：

```text
tool-events/2026-08-04.jsonl
```

里面记录：

```text
用了哪个 tool
tool input 是什么
tool result 大小
结果前后预览
是否是大输出
agent_type / session_id
```

用途：

```text
快速找出哪个工具最费 context
哪个文件被反复 Read
Grep / Bash 是否输出太多
哪个 subagent 调用了大量工具
```

它不是完整快照，而是轻量索引。

---

## 3. `changelog/`

保存**项目修改日志**。

来源是 Claude 最后输出的：

```text
CHANGELOG_ENTRY:
...
END_CHANGELOG_ENTRY
```

Hook 在 `Stop` 时把这段提取出来，追加到：

```text
changelog/2026-08-04.md
```

用途：

```text
记录这次任务做了什么
改了哪些文件
测试结果
遇到的问题
后续事项
```

这个主要是给人和以后 Agent 看，不是用来分析完整 context 的。

---

一句话总结：

```text
snapshots/   = 完整上下文快照
tool-events/ = 工具调用轻量日志
changelog/   = 项目变更摘要
```