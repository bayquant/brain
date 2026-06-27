---
tags: [claude, ai, llm, api, anthropic]
---
# Claude Code

Claude Code is Anthropic's official CLI and agentic coding tool. It wraps the Claude API into a terminal-first workflow with tool execution, file editing, MCP server support, and hook-based automation.

---

## ARCHITECTURE

```
Claude Code (CLI)
├── Conversation context (messages[], system prompt)
├── Built-in tools (Read, Edit, Write, Bash, WebFetch, ...)
├── Agent spawning (Agent tool with subagent_type)
├── MCP servers (external tool providers)
└── Hooks (shell commands triggered on events)
```

Claude receives the conversation history and tool results each turn. The context window is 1M tokens for Opus 4.8+. When context grows large, Claude Code compacts earlier turns server-side via the `compact-2026-01-12` beta.

---

## MODELS

Current models (use exact IDs — no date suffixes):

| ID | Context | Max Output | Use Case |
|---|---|---|---|
| `claude-fable-5` | 1M | 128K | Most capable — hardest tasks |
| `claude-opus-4-8` | 1M | 128K | Default for most work |
| `claude-sonnet-4-6` | 1M | 64K | Speed + cost balance |
| `claude-haiku-4-5` | 200K | 64K | Fast, cheap, simple tasks |

**Adaptive thinking** (`thinking: {type: "adaptive"}`) is the current standard on Opus 4.6+. `budget_tokens` is deprecated on 4.6 and removed entirely on 4.7/4.8/Fable 5.

---

## MESSAGES API

Every call goes through `POST /v1/messages`. The core structure:

```python
import anthropic

client = anthropic.Anthropic()

response = client.messages.create(
    model="claude-opus-4-8",
    max_tokens=16000,
    system="You are a helpful assistant.",
    messages=[
        {"role": "user", "content": "Hello"}
    ]
)

for block in response.content:
    if block.type == "text":
        print(block.text)
```

Render order: `tools` → `system` → `messages`. This matters for prompt caching.

---

## TOOL USE

Claude requests tools; your code executes them and returns results.

```python
tools = [{
    "name": "get_weather",
    "description": "Get weather for a city. Call when the user asks about current conditions.",
    "input_schema": {
        "type": "object",
        "properties": {
            "city": {"type": "string"}
        },
        "required": ["city"]
    }
}]

response = client.messages.create(
    model="claude-opus-4-8",
    max_tokens=4096,
    tools=tools,
    messages=[{"role": "user", "content": "Weather in Tokyo?"}]
)

# Loop until no more tool calls
while response.stop_reason == "tool_use":
    tool_results = []
    for block in response.content:
        if block.type == "tool_use":
            result = execute_my_tool(block.name, block.input)
            tool_results.append({
                "type": "tool_result",
                "tool_use_id": block.id,
                "content": result
            })
    messages.append({"role": "assistant", "content": response.content})
    messages.append({"role": "user", "content": tool_results})
    response = client.messages.create(...)
```

**Server-side tools** (run on Anthropic's infra, no client execution needed):

```python
tools = [
    {"type": "web_search_20260209", "name": "web_search"},
    {"type": "web_fetch_20260209", "name": "web_fetch"},
    {"type": "code_execution_20260120", "name": "code_execution"},
]
```

---

## STREAMING

Default for long outputs. Prevents SDK timeouts on large `max_tokens`.

```python
with client.messages.stream(
    model="claude-opus-4-8",
    max_tokens=64000,
    messages=[{"role": "user", "content": "Write a detailed analysis"}]
) as stream:
    for text in stream.text_stream:
        print(text, end="", flush=True)
    final = stream.get_final_message()
```

---

## EXTENDED THINKING

Enables internal chain-of-thought reasoning before responding.

```python
response = client.messages.create(
    model="claude-opus-4-8",
    max_tokens=16000,
    thinking={"type": "adaptive", "display": "summarized"},
    output_config={"effort": "high"},
    messages=[{"role": "user", "content": "Solve this complex problem..."}]
)

for block in response.content:
    if block.type == "thinking":
        print(f"[thinking] {block.thinking}")
    elif block.type == "text":
        print(block.text)
```

Effort levels: `low` | `medium` | `high` | `xhigh` (coding/agents) | `max`

---

## PROMPT CACHING

Caches stable prefix content to reduce cost (~90% savings on cache hits). Uses prefix matching — any change invalidates everything after it.

```python
response = client.messages.create(
    model="claude-opus-4-8",
    max_tokens=4096,
    system=[{
        "type": "text",
        "text": LARGE_STABLE_SYSTEM_PROMPT,
        "cache_control": {"type": "ephemeral"}  # 5m TTL; use "1h" for longer
    }],
    messages=[{"role": "user", "content": user_question}]
)

# Verify hits
print(response.usage.cache_read_input_tokens)    # ~0.1x cost
print(response.usage.cache_creation_input_tokens) # ~1.25x cost
```

**Silent invalidators to avoid:** `datetime.now()` in system prompt, non-deterministic `json.dumps()`, varying tool sets, changing model mid-session.

---

## STRUCTURED OUTPUTS

Guarantees valid JSON matching a schema.

```python
from pydantic import BaseModel

class Result(BaseModel):
    name: str
    score: int
    tags: list[str]

response = client.messages.parse(
    model="claude-opus-4-8",
    max_tokens=1024,
    messages=[{"role": "user", "content": "Extract: Alice scored 95 on Python, SQL"}],
    output_format=Result,
)

print(response.parsed_output.name)  # "Alice"
```

---

## MANAGED AGENTS

Server-managed stateful agents with Anthropic-hosted tool execution. Agent loop runs on Anthropic's side; tools execute in a container.

```python
# 1. Create agent ONCE — store the ID
agent = client.beta.agents.create(
    name="Coding Assistant",
    model="claude-opus-4-8",
    tools=[{"type": "agent_toolset_20260401"}],
)

# 2. Start session PER RUN
session = client.beta.sessions.create(
    agent={"type": "agent", "id": agent.id, "version": agent.version},
    environment_id=environment.id,
)

# 3. Stream events — open BEFORE sending
with client.beta.sessions.events.stream(session.id) as stream:
    client.beta.sessions.events.send(
        session.id,
        events=[{"type": "user.message", "content": [{"type": "text", "text": "Build a web scraper"}]}]
    )
    for event in stream:
        if event.type == "agent.message":
            for block in event.content:
                if block.type == "text":
                    print(block.text, end="", flush=True)
        elif event.type == "session.status_idle":
            break
```

**Anti-pattern:** Never call `agents.create()` on every run — it creates orphaned objects and adds latency.

---

## CLAUDE CODE CLI

Key commands and flags:

```bash
# Start a session
claude

# Use a specific model
claude --model claude-opus-4-8

# Run in a git worktree
claude --worktree <name>

# Non-interactive (pipe input)
echo "Explain this file" | claude --print

# Run a slash command
claude /compact
claude /clear
claude /fast        # Toggle fast mode
```

**Worktrees** isolate agent changes from the working copy:

```bash
git worktree add ../feature-branch feature
claude --worktree feature-branch
```

---

## MCP SERVERS

Extend Claude Code with external tools via Model Context Protocol.

```json
// ~/.claude/settings.json or .claude/settings.local.json
{
  "mcpServers": {
    "filesystem": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-filesystem", "/path"]
    },
    "github": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-github"],
      "env": {"GITHUB_TOKEN": "ghp_..."}
    }
  }
}
```

---

## HOOKS

Shell commands that fire on tool events. Configured in `.claude/settings.json`:

```json
{
  "hooks": {
    "PreToolUse": [{
      "matcher": "Bash",
      "hooks": [{"type": "command", "command": "echo 'About to run bash'"}]
    }],
    "PostToolUse": [{
      "matcher": "Edit",
      "hooks": [{"type": "command", "command": "prettier --write $CLAUDE_FILE_PATH"}]
    }]
  }
}
```

Hook events: `PreToolUse`, `PostToolUse`, `Notification`, `Stop`.

---

## SUBAGENTS

Spawn specialized agents for parallel or isolated work:

```python
# In Claude Code's tool context (Agent tool with isolation)
Agent(
    description="Research agent",
    isolation="worktree",  # isolated git worktree
    prompt="Find all usages of deprecated API X and list the files.",
)
```

Available subagent types: `claude`, `claude-code-guide`, `Explore`, `software-engineer`, `data-engineer`, `quantitative-researcher`, `Plan`, `code-reviewer`.

---

## STOP REASONS

| `stop_reason` | Meaning |
|---|---|
| `end_turn` | Natural completion |
| `tool_use` | Waiting for tool result |
| `max_tokens` | Hit output limit — increase or stream |
| `pause_turn` | Server-side tool loop paused — re-send to continue |
| `refusal` | Safety classifier declined — check `stop_details` |

---

## PYTHON QUICKSTART

```bash
pip install anthropic
export ANTHROPIC_API_KEY="sk-ant-..."
```

```python
import anthropic

client = anthropic.Anthropic()

response = client.messages.create(
    model="claude-opus-4-8",
    max_tokens=1024,
    messages=[{"role": "user", "content": "Hello, Claude"}]
)
print(response.content[0].text)
```
