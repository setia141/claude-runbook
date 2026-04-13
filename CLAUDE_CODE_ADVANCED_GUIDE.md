# Claude Code — Advanced Developer Guide

> For engineers building **systems with Claude Code**, not just using it daily.
> Covers: Agent SDK, Agent Teams, CI/CD, Context Management, Checkpointing, Hooks in code, Sub-agents.

---

## Table of Contents

1. [Agent SDK — Build Programmatic Agents](#1-agent-sdk--build-programmatic-agents)
2. [Sessions — Multi-turn, Resume, Fork](#2-sessions--multi-turn-resume-fork)
3. [Hooks in the SDK — Intercept & Control](#3-hooks-in-the-sdk--intercept--control)
4. [Sub-agents in the SDK — Specialized Workers](#4-sub-agents-in-the-sdk--specialized-workers)
5. [Agent Teams — Parallel Claude Sessions](#5-agent-teams--parallel-claude-sessions)
6. [Headless / Programmatic CLI Use](#6-headless--programmatic-cli-use)
7. [GitHub Actions Integration](#7-github-actions-integration)
8. [Context Window — What Loads & When](#8-context-window--what-loads--when)
9. [Checkpointing — Rewind File Changes](#9-checkpointing--rewind-file-changes)
10. [Advanced Patterns & Recipes](#10-advanced-patterns--recipes)

---

## 1. Agent SDK — Build Programmatic Agents

The **Claude Agent SDK** gives you the same tools, agent loop, and context management that power Claude Code — callable from Python or TypeScript.

**Use the SDK when:** you need CI/CD automation, custom applications, production pipelines, or programmatic control. Use the CLI for interactive daily development.

### Install

```bash
# TypeScript
npm install @anthropic-ai/claude-agent-sdk

# Python
pip install claude-agent-sdk
```

Set your API key:
```bash
export ANTHROPIC_API_KEY=your-key
```

Also supports: `CLAUDE_CODE_USE_BEDROCK=1`, `CLAUDE_CODE_USE_VERTEX=1`, `CLAUDE_CODE_USE_FOUNDRY=1`

### The core loop — `query()`

```python
# Python
import asyncio
from claude_agent_sdk import query, ClaudeAgentOptions, AssistantMessage, ResultMessage

async def main():
    async for message in query(
        prompt="Find and fix the bug in auth.py",
        options=ClaudeAgentOptions(
            allowed_tools=["Read", "Edit", "Bash"],
            permission_mode="acceptEdits",
        ),
    ):
        if isinstance(message, AssistantMessage):
            for block in message.content:
                if hasattr(block, "text"):
                    print(block.text)
        elif isinstance(message, ResultMessage):
            print(f"Done: {message.subtype}")

asyncio.run(main())
```

```typescript
// TypeScript
import { query } from "@anthropic-ai/claude-agent-sdk";

for await (const message of query({
  prompt: "Find and fix the bug in auth.py",
  options: {
    allowedTools: ["Read", "Edit", "Bash"],
    permissionMode: "acceptEdits"
  }
})) {
  if (message.type === "assistant") {
    for (const block of message.message.content) {
      if ("text" in block) console.log(block.text);
    }
  }
  if (message.type === "result") console.log(`Done: ${message.subtype}`);
}
```

### Key `ClaudeAgentOptions` fields

| Option | Python | TypeScript | Description |
|---|---|---|---|
| Allowed tools | `allowed_tools` | `allowedTools` | Pre-approve tools (no permission prompts) |
| Permission mode | `permission_mode` | `permissionMode` | `"acceptEdits"` / `"dontAsk"` / `"bypassPermissions"` |
| System prompt | `system_prompt` | `systemPrompt` | Custom instructions for this run |
| Max turns | `max_turns` | `maxTurns` | Cap on agent loop iterations |
| Model | `model` | `model` | Override model for this run |
| MCP servers | `mcp_servers` | `mcpServers` | Connect external tools |
| Hooks | `hooks` | `hooks` | Lifecycle callbacks (see Section 3) |
| Sub-agents | `agents` | `agents` | Specialist sub-agents (see Section 4) |
| Resume | `resume` | `resume` | Resume a session by ID |
| Fork session | `fork_session` | `forkSession` | Branch from an existing session |
| Continue | — | `continue` | Continue most recent session |
| Setting sources | `setting_sources` | `settingSources` | Load `.claude/` config: `["project"]` |

### Permission modes

| Mode | Behavior |
|---|---|
| `acceptEdits` | Auto-approves file edits + common fs commands |
| `dontAsk` | Denies anything not in `allowedTools` |
| `auto` | AI classifier decides safety (TypeScript only) |
| `bypassPermissions` | Skip all prompts — containers/CI only |
| `default` | Requires `canUseTool` callback for custom approval |

### Built-in tools

| Tool | What it does |
|---|---|
| `Read` | Read any file |
| `Write` | Create new files |
| `Edit` | Edit existing files |
| `Bash` | Run shell commands, git, scripts |
| `Glob` | Find files by pattern (`**/*.ts`) |
| `Grep` | Search file contents with regex |
| `WebSearch` | Search the web |
| `WebFetch` | Fetch and parse web pages |
| `Agent` | Spawn sub-agents (required for subagents) |
| `Monitor` | Watch a background script, react to output |
| `AskUserQuestion` | Ask user clarifying questions |

### Connect MCP servers

```python
options = ClaudeAgentOptions(
    mcp_servers={
        "playwright": {"command": "npx", "args": ["@playwright/mcp@latest"]},
        "github": {
            "command": "npx",
            "args": ["-y", "@modelcontextprotocol/server-github"],
            "env": {"GITHUB_TOKEN": os.environ["GITHUB_TOKEN"]}
        }
    }
)
```

### Load project config (CLAUDE.md, hooks, skills)

```python
options = ClaudeAgentOptions(
    setting_sources=["project"],  # loads .claude/settings.json, CLAUDE.md, etc.
)
```

---

## 2. Sessions — Multi-turn, Resume, Fork

A **session** is the full conversation history accumulated during a `query()` call. Sessions persist to disk automatically.

### When you need session management

| Scenario | What to use |
|---|---|
| Single prompt, no follow-up | Nothing — one `query()` is enough |
| Multi-turn chat, same process | `ClaudeSDKClient` (Python) or `continue: true` (TS) |
| Resume after process restart | `continue_conversation=True` / `continue: true` |
| Resume a specific past session | Capture session ID → `resume=session_id` |
| Try alternative without losing original | `fork_session=True` / `forkSession: true` |

### Multi-turn: Python `ClaudeSDKClient`

```python
import asyncio
from claude_agent_sdk import ClaudeSDKClient, ClaudeAgentOptions

async def main():
    options = ClaudeAgentOptions(allowed_tools=["Read", "Edit", "Glob"])

    async with ClaudeSDKClient(options=options) as client:
        await client.query("Analyze the auth module")
        async for msg in client.receive_response():
            print(msg)

        # Automatically continues the same session
        await client.query("Now refactor it to use JWT")
        async for msg in client.receive_response():
            print(msg)

asyncio.run(main())
```

### Multi-turn: TypeScript `continue: true`

```typescript
import { query } from "@anthropic-ai/claude-agent-sdk";

// First query
for await (const msg of query({ prompt: "Analyze the auth module" })) {
  if (msg.type === "result") console.log(msg.result);
}

// Continue: picks up the most recent session
for await (const msg of query({
  prompt: "Now refactor it to use JWT",
  options: { continue: true, allowedTools: ["Read", "Edit"] }
})) {
  if (msg.type === "result") console.log(msg.result);
}
```

### Capture and resume a specific session

```python
# Python — capture session ID
session_id = None
async for msg in query(prompt="Analyze the auth module", options=...):
    if isinstance(msg, ResultMessage):
        session_id = msg.session_id

# Resume later (different process, different time)
async for msg in query(
    prompt="Now implement the improvements you suggested",
    options=ClaudeAgentOptions(resume=session_id)
):
    print(msg)
```

```typescript
// TypeScript — capture session ID
let sessionId: string | undefined;
for await (const msg of query({ prompt: "..." })) {
  if (msg.type === "result") sessionId = msg.session_id;
}

// Resume
for await (const msg of query({
  prompt: "Implement the improvements",
  options: { resume: sessionId }
})) {
  if (msg.type === "result") console.log(msg.result);
}
```

### Fork — explore alternatives without losing original

```python
# Branches conversation from session_id into a new independent session
forked_id = None
async for msg in query(
    prompt="Instead of JWT, implement OAuth2",
    options=ClaudeAgentOptions(resume=session_id, fork_session=True),
):
    if isinstance(msg, ResultMessage):
        forked_id = msg.session_id  # new ID — original unchanged

# Original session still intact, resume it independently
async for msg in query(
    prompt="Continue with the JWT approach",
    options=ClaudeAgentOptions(resume=session_id),
):
    print(msg)
```

> **Note:** Fork branches the conversation, NOT the filesystem. File edits from either branch affect the same files on disk. Use checkpointing (Section 9) to branch + revert file changes.

### Session storage location

```
~/.claude/projects/<encoded-cwd>/<session-id>.jsonl
```
The `<encoded-cwd>` is the absolute path with non-alphanumeric chars replaced by `-`.  
`/Users/me/proj` → `-Users-me-proj`

**Resuming across hosts:** copy the `.jsonl` file to the same path on the new host. The `cwd` must match.

---

## 3. Hooks in the SDK — Intercept & Control

SDK hooks are **callback functions** (not shell scripts) that fire at lifecycle events. Same events as CLI hooks but defined in code.

### Configure hooks

```python
from claude_agent_sdk import ClaudeAgentOptions, HookMatcher

options = ClaudeAgentOptions(
    hooks={
        "PreToolUse": [
            HookMatcher(matcher="Bash", hooks=[my_pre_bash_hook]),
            HookMatcher(matcher="Write|Edit", hooks=[my_file_hook]),
        ],
        "PostToolUse": [
            HookMatcher(hooks=[audit_logger]),  # no matcher = fires for ALL tools
        ],
        "Stop": [
            HookMatcher(hooks=[cleanup_hook]),
        ]
    }
)
```

### Callback signature

```python
# Python
async def my_hook(input_data: dict, tool_use_id: str | None, context) -> dict:
    # input_data: event details (tool_name, tool_input, session_id, etc.)
    # tool_use_id: correlates PreToolUse with PostToolUse for same call
    # context: reserved for future use
    return {}  # return {} to allow, or control dict to modify/block
```

```typescript
// TypeScript
import { HookCallback } from "@anthropic-ai/claude-agent-sdk";

const myHook: HookCallback = async (input, toolUseId, { signal }) => {
  // input: typed hook input object
  // toolUseId: correlates pre/post events
  // signal: AbortSignal for cancellation
  return {};
};
```

### Available hook events (SDK)

| Event | Python | TypeScript | Can Block? | Use for |
|---|---|---|---|---|
| `PreToolUse` | Yes | Yes | Yes | Block/modify tool calls |
| `PostToolUse` | Yes | Yes | No | Log, audit |
| `PostToolUseFailure` | Yes | Yes | No | Error handling |
| `UserPromptSubmit` | Yes | Yes | Yes | Inject context |
| `Stop` | Yes | Yes | Yes | Save state, cleanup |
| `SubagentStart` | Yes | Yes | No | Track spawning |
| `SubagentStop` | Yes | Yes | Yes | Aggregate results |
| `PreCompact` | Yes | Yes | No | Archive transcript |
| `PermissionRequest` | Yes | Yes | Yes | Custom permissions |
| `Notification` | Yes | Yes | No | Slack/PagerDuty alerts |
| `SessionStart` | No | Yes | No | Init logging |
| `SessionEnd` | No | Yes | No | Cleanup |
| `TeammateIdle` | No | Yes | Yes | Reassign work |
| `TaskCompleted` | No | Yes | Yes | Aggregate results |

### Hook output reference

```python
# Allow without changes
return {}

# Block the operation
return {
    "hookSpecificOutput": {
        "hookEventName": "PreToolUse",
        "permissionDecision": "deny",
        "permissionDecisionReason": "Not allowed in production"
    }
}

# Modify the tool input
return {
    "hookSpecificOutput": {
        "hookEventName": "PreToolUse",
        "permissionDecision": "allow",  # required with updatedInput
        "updatedInput": {
            **input_data["tool_input"],
            "file_path": f"/sandbox{original_path}"
        }
    }
}

# Inject context into Claude's conversation
return {
    "systemMessage": "Note: this file is protected by compliance policy",
    "hookSpecificOutput": {
        "hookEventName": "PreToolUse",
        "permissionDecision": "deny",
        "permissionDecisionReason": "Protected file"
    }
}

# Stop the entire agent run
return { "continue": False, "stopReason": "Policy violation detected" }

# Fire-and-forget (don't block agent)
return { "async_": True, "asyncTimeout": 30000 }  # Python: async_
# TypeScript: { async: true, asyncTimeout: 30000 }
```

### Practical examples

**Block dangerous commands:**
```python
async def block_destructive(input_data, tool_use_id, context):
    command = input_data.get("tool_input", {}).get("command", "")
    dangerous = ["rm -rf", "DROP TABLE", "git push --force", "truncate"]
    if any(d in command for d in dangerous):
        return {
            "hookSpecificOutput": {
                "hookEventName": "PreToolUse",
                "permissionDecision": "deny",
                "permissionDecisionReason": f"Blocked: dangerous pattern detected"
            }
        }
    return {}

options = ClaudeAgentOptions(
    hooks={"PreToolUse": [HookMatcher(matcher="Bash", hooks=[block_destructive])]}
)
```

**Redirect writes to a sandbox:**
```python
async def sandbox_writes(input_data, tool_use_id, context):
    path = input_data["tool_input"].get("file_path", "")
    return {
        "hookSpecificOutput": {
            "hookEventName": "PreToolUse",
            "permissionDecision": "allow",
            "updatedInput": {**input_data["tool_input"], "file_path": f"/sandbox{path}"}
        }
    }
```

**Audit log every tool call:**
```python
from datetime import datetime

async def audit_log(input_data, tool_use_id, context):
    with open("./audit.log", "a") as f:
        f.write(f"{datetime.now().isoformat()} | {input_data['tool_name']} | "
                f"{input_data.get('tool_input', {})}\n")
    return {"async_": True}  # don't block the agent
```

**Forward notifications to Slack:**
```python
import json, urllib.request, asyncio

def _post_slack(message):
    data = json.dumps({"text": f"Agent: {message}"}).encode()
    req = urllib.request.Request(
        "https://hooks.slack.com/services/YOUR/WEBHOOK",
        data=data, headers={"Content-Type": "application/json"}, method="POST"
    )
    urllib.request.urlopen(req)

async def notify_slack(input_data, tool_use_id, context):
    try:
        await asyncio.to_thread(_post_slack, input_data.get("message", ""))
    except Exception as e:
        print(f"Slack error: {e}")
    return {}
```

**Matcher patterns:**

| Matcher | Matches |
|---|---|
| `"Bash"` | Only Bash tool |
| `"Write\|Edit"` | Write or Edit |
| `"^mcp__"` | All MCP tools |
| `"mcp__github__.*"` | All GitHub MCP tools |
| Omitted | Every event of this type |

---

## 4. Sub-agents in the SDK — Specialized Workers

Sub-agents are separate agent instances with their own context window, tools, and system prompt. The parent delegates, the sub-agent works, and only the final result returns.

### When to use sub-agents vs inline

| Use sub-agent | Use inline |
|---|---|
| Task floods context with search results | Task is simple and focused |
| Need specialized expertise/tools | General-purpose work |
| Want parallel execution | Sequential work |
| Isolate risky operations | Safe operations |

### Define sub-agents

```python
from claude_agent_sdk import AgentDefinition

options = ClaudeAgentOptions(
    # Agent tool MUST be in allowedTools
    allowed_tools=["Read", "Glob", "Grep", "Agent"],
    agents={
        "code-reviewer": AgentDefinition(
            description="Expert security code reviewer. Use for security audits and PR reviews.",
            prompt="""You are a security-focused code reviewer.
Identify: injection vulnerabilities, auth bypasses, secrets in code, insecure deps.
Never modify files. Only read and report.""",
            tools=["Read", "Grep", "Glob"],  # read-only
            model="opus",  # more powerful model for reviews
        ),
        "test-runner": AgentDefinition(
            description="Runs and analyzes test suites. Use for test execution.",
            prompt="Run tests and provide clear analysis of failures.",
            tools=["Bash", "Read", "Grep"],
        ),
    }
)
```

```typescript
const options = {
  allowedTools: ["Read", "Glob", "Grep", "Agent"],
  agents: {
    "code-reviewer": {
      description: "Expert security code reviewer. Use for security audits and PR reviews.",
      prompt: `You are a security-focused code reviewer.
Identify: injection vulnerabilities, auth bypasses, secrets, insecure deps.
Never modify files. Only read and report.`,
      tools: ["Read", "Grep", "Glob"],
      model: "opus" as const
    }
  }
};
```

### AgentDefinition fields

| Field | Description |
|---|---|
| `description` | **When** Claude should use this agent (match keywords to task language) |
| `prompt` | The agent's system prompt — its expertise and constraints |
| `tools` | Allowed tools (omit = inherit all) |
| `model` | `"sonnet"` / `"opus"` / `"haiku"` / `"inherit"` |
| `skills` | Skill names to preload into this agent |
| `mcp_servers` | MCP servers available to this agent |

### Invoke explicitly

```python
# Guarantee a specific agent is used by naming it
async for msg in query(
    prompt="Use the code-reviewer agent to check src/auth/",
    options=options
):
    ...
```

### Sub-agents — what they inherit

| Gets | Does NOT get |
|---|---|
| Own system prompt (`AgentDefinition.prompt`) | Parent's conversation history |
| Agent tool's prompt string | Parent's tool results |
| CLAUDE.md (via `settingSources`) | Parent's system prompt |
| Tool definitions from `tools` field | Skills (unless in `AgentDefinition.skills`) |

**The only channel from parent → sub-agent is the Agent tool's prompt string.** Include file paths, errors, and decisions in that string.

### Common tool sets

| Use case | Tools |
|---|---|
| Read-only analysis | `Read`, `Grep`, `Glob` |
| Test execution | `Bash`, `Read`, `Grep` |
| Code modification | `Read`, `Edit`, `Write`, `Grep`, `Glob` |
| Full automation | All tools (omit `tools` field) |

### Dynamic agent configuration

```python
def create_reviewer(strictness: str) -> AgentDefinition:
    return AgentDefinition(
        description="Security reviewer",
        prompt=f"You are a {'strict' if strictness == 'high' else 'balanced'} reviewer...",
        tools=["Read", "Grep", "Glob"],
        model="opus" if strictness == "high" else "sonnet",
    )

options = ClaudeAgentOptions(
    allowed_tools=["Read", "Glob", "Agent"],
    agents={"reviewer": create_reviewer(os.environ.get("REVIEW_STRICTNESS", "medium"))}
)
```

---

## 5. Agent Teams — Parallel Claude Sessions

Agent Teams coordinate **multiple Claude Code sessions** working in parallel with a shared task list and direct messaging.

> **Experimental** — enable with `CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS=1`

### Enable

```json
// settings.json
{
  "env": {
    "CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS": "1"
  }
}
```

### Sub-agents vs Agent Teams

| | Sub-agents | Agent Teams |
|---|---|---|
| Context | Own window; results return to caller | Own window; fully independent |
| Communication | Report to parent only | Teammates message each other directly |
| Coordination | Parent manages all work | Shared task list, self-coordinating |
| Best for | Focused tasks, result matters | Research, review, competing hypotheses |
| Token cost | Lower | Higher (each is a full Claude instance) |

### Start a team

```text
Create an agent team with 3 teammates to review PR #142:
- One focused on security
- One checking performance
- One validating test coverage
```

Claude creates the team, spawns teammates, coordinates work, synthesizes results.

### Navigation (in-process mode)

- `Shift+Down` — cycle through teammates
- `Escape` — interrupt current teammate turn
- `Ctrl+T` — toggle task list

### Display modes

```json
// ~/.claude.json
{
  "teammateMode": "in-process"  // "auto" | "in-process" | "tmux"
}
```

Or per-session: `claude --teammate-mode in-process`

### Require plan approval before teammates act

```text
Spawn a teammate to refactor the auth module. Require plan approval before any changes.
```

The teammate works in read-only plan mode → submits plan to lead → lead approves or rejects → teammate executes.

### Shut down and clean up

```text
Ask the researcher teammate to shut down
Clean up the team
```

**Always clean up via the lead.** Teammates should not run cleanup.

### Best practices

- **3-5 teammates** is the sweet spot — more adds coordination overhead with diminishing returns
- **5-6 tasks per teammate** keeps everyone productive
- **Independent tasks only** — if teammates edit the same file, they'll overwrite each other
- **Give spawn context explicitly** — teammates don't inherit the lead's conversation history
- **Use for:** parallel code review, competing debugging hypotheses, multi-module features

### Strong use cases

```text
# Parallel code review
Create a team: security reviewer, performance reviewer, test coverage reviewer.
Review PR #142.

# Competing debugging hypotheses
This endpoint is 10x slower after the last deploy. Spawn 4 teammates to
investigate different causes. Have them challenge each other's theories.
Update findings.md with whatever consensus emerges.

# Multi-module feature
Build the user notification system. Spawn teammates to work on:
- Email service (src/email/)
- Push notifications (src/push/)
- In-app notifications (src/notifications/)
- Tests for all three
```

### Team storage

```
~/.claude/teams/{team-name}/config.json    # runtime state — don't edit
~/.claude/tasks/{team-name}/              # shared task list
```

---

## 6. Headless / Programmatic CLI Use

Use `claude -p` for scripting, CI pipelines, and automation.

### Basic usage

```bash
# Non-interactive with a prompt
claude -p "What does the auth module do?"

# Pre-approve specific tools
claude -p "Fix the null pointer in auth.ts" --allowedTools "Read,Edit,Bash"

# Permission mode
claude -p "Apply lint fixes" --permission-mode acceptEdits
```

### `--bare` mode (recommended for CI)

Skips auto-discovery of hooks, skills, MCP servers, CLAUDE.md, auto memory. Gives reproducible results on any machine.

```bash
claude --bare -p "Summarize this file" --allowedTools "Read"
```

Pass context explicitly in bare mode:

| To load | Flag |
|---|---|
| System prompt additions | `--append-system-prompt "..."` |
| Settings file | `--settings path/to/settings.json` |
| MCP servers | `--mcp-config path/to/config.json` |
| Custom agents | `--agents '{"name": {...}}'` |
| Plugin directory | `--plugin-dir ./path` |

### Structured output

```bash
# JSON output (result + session metadata)
claude -p "Summarize this project" --output-format json

# Extract just the text result
claude -p "Summarize" --output-format json | jq -r '.result'

# Structured output with JSON schema
claude -p "Extract function names from auth.py" \
  --output-format json \
  --json-schema '{"type":"object","properties":{"functions":{"type":"array","items":{"type":"string"}}}}'

# Streaming JSON (real-time)
claude -p "Write a poem" --output-format stream-json --verbose --include-partial-messages
```

### Continue conversations in scripts

```bash
# First prompt
claude -p "Review this codebase for security issues"

# Continue the same conversation
claude -p "Now focus on the database queries" --continue

# Resume a specific session by ID
session_id=$(claude -p "Start analysis" --output-format json | jq -r '.session_id')
claude -p "Continue that analysis" --resume "$session_id"
```

### Real-world scripting patterns

```bash
# Analyze logs
tail -200 app.log | claude -p "Summarize errors and suggest fixes"

# Review changed files
git diff main --name-only | claude -p "Security review these changed files"

# Create a commit
claude -p "Review staged changes and commit with a good message" \
  --allowedTools "Bash(git diff *),Bash(git log *),Bash(git commit *)"

# Pipe PR diff for review
gh pr diff "$PR_NUMBER" | claude --bare -p \
  --append-system-prompt "You are a security engineer. Review for vulnerabilities." \
  --output-format json | jq -r '.result'

# Bulk operations
for f in src/**/*.py; do
  claude --bare -p "Add type hints to all functions in $f" \
    --allowedTools "Read,Edit" --permission-mode acceptEdits
done
```

### Output format details

```json
// --output-format json
{
  "result": "The auth module handles...",
  "session_id": "abc-123",
  "total_cost_usd": 0.0045,
  "structured_output": null
}
```

---

## 7. GitHub Actions Integration

Run Claude Code in CI/CD pipelines with the `claude-code-action`.

### Quick setup

```bash
# In your terminal
claude
/install-github-app
```

This guides you through installing the GitHub App and setting up `ANTHROPIC_API_KEY` in repo secrets.

### Basic workflow — respond to `@claude` mentions

```yaml
# .github/workflows/claude.yml
name: Claude Code
on:
  issue_comment:
    types: [created]
  pull_request_review_comment:
    types: [created]

jobs:
  claude:
    runs-on: ubuntu-latest
    steps:
      - uses: anthropics/claude-code-action@v1
        with:
          anthropic_api_key: ${{ secrets.ANTHROPIC_API_KEY }}
```

Now mention `@claude` in any PR or issue comment. Examples:
```
@claude implement this feature based on the issue description
@claude fix the TypeError in the dashboard component
@claude review this PR for security vulnerabilities
```

### Automated PR review on every PR

```yaml
name: PR Review
on:
  pull_request:
    types: [opened, synchronize]

jobs:
  review:
    runs-on: ubuntu-latest
    steps:
      - uses: anthropics/claude-code-action@v1
        with:
          anthropic_api_key: ${{ secrets.ANTHROPIC_API_KEY }}
          prompt: |
            Review this PR for code quality, correctness, and security.
            Post findings as inline review comments.
          claude_args: "--max-turns 10 --model claude-opus-4-6"
```

### Scheduled automation

```yaml
name: Daily Report
on:
  schedule:
    - cron: "0 9 * * 1-5"  # weekdays at 9am

jobs:
  report:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: anthropics/claude-code-action@v1
        with:
          anthropic_api_key: ${{ secrets.ANTHROPIC_API_KEY }}
          prompt: |
            Summarize yesterday's commits, flag any open critical bugs,
            and identify PRs that are ready to merge.
          claude_args: "--model claude-opus-4-6"
```

### Action parameters (v1)

| Parameter | Description | Required |
|---|---|---|
| `prompt` | Instructions (or invoke a skill by name) | No* |
| `claude_args` | Any Claude Code CLI args | No |
| `anthropic_api_key` | API key | Yes** |
| `github_token` | GitHub token | No |
| `trigger_phrase` | Custom trigger (default: `@claude`) | No |
| `use_bedrock` | Use AWS Bedrock | No |
| `use_vertex` | Use Google Vertex AI | No |

*`prompt` optional when responding to `@claude` mentions  
**Required for direct API; not needed for Bedrock/Vertex

### Common `claude_args`

```yaml
claude_args: |
  --max-turns 5
  --model claude-sonnet-4-6
  --allowedTools Read,Grep,Bash
  --append-system-prompt "Follow our coding standards in CLAUDE.md"
  --permission-mode acceptEdits
```

### Use with AWS Bedrock

```yaml
jobs:
  claude:
    permissions:
      contents: write
      pull-requests: write
      issues: write
      id-token: write

    steps:
      - uses: actions/checkout@v4

      - name: Configure AWS
        uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: ${{ secrets.AWS_ROLE_TO_ASSUME }}
          aws-region: us-west-2

      - uses: anthropics/claude-code-action@v1
        with:
          github_token: ${{ steps.app-token.outputs.token }}
          use_bedrock: "true"
          claude_args: "--model us.anthropic.claude-sonnet-4-6 --max-turns 10"
```

### Cost optimization

- Use `--max-turns` to cap runaway loops
- Set `--permission-mode dontAsk` to fail fast on unexpected tool calls
- Use `--bare` to skip unnecessary config discovery
- Limit who can trigger `@claude` with branch protection rules

---

## 8. Context Window — What Loads & When

Understanding context loading is critical for reliable agent behavior. Knowing what's in context tells you why Claude does what it does.

### Startup loading order

```
1. System prompt (~4,200 tokens) — core instructions, never seen by user
2. Auto memory MEMORY.md (~680 tokens) — first 200 lines or 25KB
3. Environment info (~280 tokens) — git status, OS, working dir
4. Managed CLAUDE.md — org-wide (full, always)
5. ~/.claude/CLAUDE.md — user (full, if exists)
6. ./CLAUDE.md or ./.claude/CLAUDE.md — project (full, if exists)
7. ./CLAUDE.local.md — local personal (full, if exists)
8. .claude/rules/*.md WITHOUT paths frontmatter — (full, all of them)
9. Skills descriptions — truncated to ~8KB total budget
10. Your first message
```

**Rules with `paths:` frontmatter load on demand** when Claude reads matching files — not at startup.

### What skills load and when

- **Skill descriptions** — always in context (truncated at 250 chars each)
- **Skill body** — loaded into context only when invoked
- **After compaction** — skill body re-attached (up to 5,000 tokens each, 25,000 total budget)

### What survives `/compact`

| Survives | Lost |
|---|---|
| Root `CLAUDE.md` (re-read from disk) | Nested CLAUDE.md (reload next time matching file read) |
| Invoked skill bodies (re-attached, up to budget) | Conversation messages beyond summary |
| Most recent invocation of each skill | Earlier skill invocations if many active |
| System prompt | Nothing in system prompt is lost |

### Context strategy

**Shrink CLAUDE.md:**
- Keep under 200 lines — longer reduces adherence
- Move detailed reference material to `.claude/rules/` with `paths:` scoping
- Use `@import` for large docs — they load at startup but live in separate files

**Use path-scoped rules:**
```markdown
<!-- .claude/rules/database.md -->
---
paths:
  - "src/db/**"
  - "migrations/**"
---
# DB Rules — only loads when working with DB files
```

**Use sub-agents for context isolation:**
```python
# Bad: agent reads 50 files, flooding main context
# Good: research sub-agent reads 50 files, returns only summary
agents={
    "researcher": AgentDefinition(
        description="Research implementation patterns. Use when exploring how features are built.",
        tools=["Read", "Grep", "Glob"],
    )
}
```

**Use `--bare` in CI:**
No CLAUDE.md, no auto memory, no hooks from config files. Reproducible.

**Compact strategically:**
```
/compact
> summarize the debugging session, focusing on what we learned about the auth bug
```

Instead of `/compact` which summarizes everything generically.

---

## 9. Checkpointing — Rewind File Changes

Checkpoints capture file state before every edit. They let you undo changes mid-session without git.

### How it works

- Every user prompt creates a checkpoint automatically
- Checkpoints persist across sessions (accessible in resumed conversations)
- Cleaned up after 30 days (configurable with `cleanupPeriodDays`)
- **Only tracks Claude's file editing tools** — NOT `bash rm`, `mv`, etc.

### Rewind

```
Press Esc + Esc  (or type /rewind)
```

You see a scrollable list of every prompt. Select one, then choose:

| Action | What it does |
|---|---|
| **Restore code and conversation** | Revert both files AND conversation to that point |
| **Restore conversation** | Rewind chat while keeping current code |
| **Restore code** | Revert file changes, keep conversation |
| **Summarize from here** | Compress conversation from this point, no file changes |
| **Never mind** | Cancel |

After restoring, the original prompt is restored to the input so you can re-send or edit it.

### Summarize from here vs `/compact`

| | Summarize from here | `/compact` |
|---|---|---|
| Scope | From selected point forward | Entire conversation |
| Files changed | No | No |
| Original messages | Preserved in transcript | Gone |
| Use when | You want early context intact | General cleanup |

### In the SDK — file checkpointing

```python
from claude_agent_sdk import ClaudeAgentOptions

options = ClaudeAgentOptions(
    # Checkpointing is on by default in the SDK
    # Access checkpoints via sessions after the run
)
```

The SDK persists sessions with full tool history. To revert file changes programmatically, track file paths in `PostToolUse` hooks and restore from backups if needed.

### Limitations

- Bash file operations (`rm`, `mv`, `cp`) are **NOT** tracked
- External changes from other processes are NOT tracked
- Not a replacement for git — think of it as "local undo", git as "permanent history"

---

## 10. Advanced Patterns & Recipes

### Pattern: Agentic pipeline with error recovery

```python
import asyncio
from claude_agent_sdk import query, ClaudeAgentOptions, ResultMessage

MAX_RETRIES = 3

async def run_with_retry(prompt: str, options: ClaudeAgentOptions):
    for attempt in range(MAX_RETRIES):
        session_id = None
        result = None
        try:
            async for msg in query(prompt=prompt, options=options):
                if isinstance(msg, ResultMessage):
                    session_id = msg.session_id
                    if msg.subtype == "error_max_turns":
                        # Resume with more turns
                        options.max_turns = (options.max_turns or 10) * 2
                        options.resume = session_id
                        break
                    elif msg.subtype == "success":
                        return msg.result
        except Exception as e:
            print(f"Attempt {attempt+1} failed: {e}")
            await asyncio.sleep(2 ** attempt)
    raise RuntimeError("Agent failed after max retries")
```

### Pattern: Parallel sub-agent fan-out

```python
async def parallel_analysis(codebase_paths: list[str]):
    """Run security, performance, and test-coverage analysis in parallel."""
    tasks = []
    for path in codebase_paths:
        task = query(
            prompt=f"Use the code-reviewer agent to analyze {path}",
            options=ClaudeAgentOptions(
                allowed_tools=["Read", "Grep", "Glob", "Agent"],
                agents={"code-reviewer": AgentDefinition(
                    description="Code analysis",
                    prompt="Analyze for security, performance, and test coverage.",
                    tools=["Read", "Grep", "Glob"]
                )}
            )
        )
        tasks.append(task)

    results = []
    for task in tasks:
        async for msg in task:
            if isinstance(msg, ResultMessage) and msg.subtype == "success":
                results.append(msg.result)
    return results
```

### Pattern: CI quality gate with structured output

```bash
#!/bin/bash
# .github/scripts/quality-gate.sh
RESULT=$(claude --bare -p "
Review the changed files for:
1. Security vulnerabilities (critical/high/medium)
2. Missing tests for new code
3. Breaking API changes

Output JSON: {\"passed\": bool, \"issues\": [{\"severity\": str, \"description\": str}]}
" \
  --allowedTools "Read,Grep,Glob" \
  --output-format json \
  --json-schema '{"type":"object","properties":{"passed":{"type":"boolean"},"issues":{"type":"array"}}}')

PASSED=$(echo "$RESULT" | jq '.structured_output.passed')
ISSUES=$(echo "$RESULT" | jq '.structured_output.issues | length')

if [ "$PASSED" = "false" ]; then
  echo "Quality gate FAILED: $ISSUES issues found"
  echo "$RESULT" | jq '.structured_output.issues[]'
  exit 1
fi
echo "Quality gate passed"
```

### Pattern: Dynamic CLAUDE.md from environment

```python
import os

env_instructions = f"""
# Environment Context
- Environment: {os.environ.get('NODE_ENV', 'development')}
- Database: {os.environ.get('DB_TYPE', 'postgres')}
- Team: {os.environ.get('TEAM_NAME', 'unknown')}

# Constraints
{'- NEVER run migrations directly — use the migrations CLI' if os.environ.get('NODE_ENV') == 'production' else ''}
{'- All DB changes require review approval' if os.environ.get('REQUIRE_REVIEW') else ''}
"""

options = ClaudeAgentOptions(
    system_prompt=env_instructions,
)
```

### Pattern: Hook-based compliance enforcer

```python
PROTECTED_FILES = {".env", ".env.local", "secrets.yml", "private.key"}
PROTECTED_DIRS = ["/etc/", "/usr/", "~/.ssh/"]

async def compliance_hook(input_data, tool_use_id, context):
    tool = input_data.get("tool_name", "")
    tool_input = input_data.get("tool_input", {})

    if tool in ("Write", "Edit"):
        file_path = tool_input.get("file_path", "")
        filename = file_path.split("/")[-1]

        if filename in PROTECTED_FILES:
            return {
                "systemMessage": f"Policy: {filename} is protected. Use env vars or secrets manager.",
                "hookSpecificOutput": {
                    "hookEventName": "PreToolUse",
                    "permissionDecision": "deny",
                    "permissionDecisionReason": f"Protected file: {filename}"
                }
            }

        if any(file_path.startswith(d) for d in PROTECTED_DIRS):
            return {
                "hookSpecificOutput": {
                    "hookEventName": "PreToolUse",
                    "permissionDecision": "deny",
                    "permissionDecisionReason": f"Protected directory"
                }
            }
    return {}
```

### Pattern: Agent Team for competing hypotheses

```text
# In Claude Code interactive session (after enabling agent teams)

We have a production outage: checkout is failing for ~15% of users.
Error: "Transaction timeout after 30s".

Create an agent team with 4 teammates to investigate:
1. Database query performance (check slow query log)
2. Network/connection pool issues
3. Recent code changes in checkout flow
4. Infrastructure metrics (memory, CPU, connections)

Each teammate should try to disprove the others' theories.
Update INCIDENT.md with findings and consensus root cause.
```

### Pattern: Auto-checkpoint before risky operations

```python
import subprocess, shutil, os
from datetime import datetime

async def checkpoint_before_deletion(input_data, tool_use_id, context):
    """Create a backup before any file deletion or overwrite."""
    if input_data["tool_name"] == "Write":
        file_path = input_data["tool_input"].get("file_path", "")
        if os.path.exists(file_path):
            backup = f"{file_path}.backup.{datetime.now().strftime('%H%M%S')}"
            shutil.copy2(file_path, backup)
            print(f"Checkpoint: {file_path} → {backup}")
    return {}

options = ClaudeAgentOptions(
    hooks={"PreToolUse": [HookMatcher(matcher="Write", hooks=[checkpoint_before_deletion])]}
)
```

---

## Quick Decision Guide

```
Building a script or CI job?
  → Use claude -p with --bare and --output-format json

Need multi-turn conversation in code?
  → Python: ClaudeSDKClient  |  TypeScript: continue: true

Need to resume after a crash or process restart?
  → Capture session_id from ResultMessage, pass to resume=

Need to try two approaches without losing either?
  → fork_session=True, capture both session IDs

Need to isolate context or run analysis in parallel?
  → Sub-agents (Section 4)

Need multiple Claude sessions coordinating on one task?
  → Agent Teams (Section 5) — experimental

Need to intercept/block/modify tool calls in code?
  → SDK hooks (Section 3)

Need to automate PR reviews or issue triage?
  → GitHub Actions (Section 7)

Claude going off-track in a long session?
  → /rewind to restore, or /compact with specific instructions

Context limit approaching?
  → /compact, or move research to a sub-agent
```

---

*Full documentation: https://code.claude.com/docs*
