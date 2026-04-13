# Claude Code — Complete Developer Guide

> End-to-end guide for developers and engineers. Read top-to-bottom for onboarding, or jump to any section as a reference.

---

## Table of Contents

1. [What Is Claude Code?](#1-what-is-claude-code)
2. [Installation & First Run](#2-installation--first-run)
3. [Daily Workflows](#3-daily-workflows)
4. [Permissions — What Gets Asked & Why](#4-permissions--what-gets-asked--why)
5. [How Claude Remembers Your Project](#5-how-claude-remembers-your-project)
6. [The `.claude` Folder — Full Map](#6-the-claude-folder--full-map)
7. [Settings (settings.json)](#7-settings-settingsjson)
8. [Hooks — Automate Around Claude's Actions](#8-hooks--automate-around-claudes-actions)
9. [Skills — Custom Slash Commands](#9-skills--custom-slash-commands)
10. [Sub-Agents — Specialized Assistants](#10-sub-agents--specialized-assistants)
11. [MCP — Connect External Tools](#11-mcp--connect-external-tools)
12. [CLI & Slash Command Reference](#12-cli--slash-command-reference)
13. [Tips & Troubleshooting](#13-tips--troubleshooting)

---

## 1. What Is Claude Code?

Claude Code is an AI coding assistant that lives in your **terminal, VS Code, JetBrains, desktop app, or browser**. Unlike a chat window, it can:

- Read your entire codebase across multiple files
- Edit files and apply changes directly
- Run commands — tests, builds, git, linters
- Create commits and pull requests
- Remember your project's conventions across sessions

Think of it as a senior developer who has already read the whole repo and can act on it — not just advise.

**Available surfaces:**

| Surface | How to start |
|---|---|
| Terminal | `claude` in any project directory |
| VS Code / Cursor | Command Palette → `Claude Code: Open in New Tab` |
| JetBrains | Install plugin from Marketplace |
| Desktop app | Launch Claude → Code tab |
| Web | `claude.ai/code` |

---

## 2. Installation & First Run

**macOS / Linux / WSL:**
```bash
curl -fsSL https://claude.ai/install.sh | bash
```

**Windows PowerShell:**
```powershell
irm https://claude.ai/install.ps1 | iex
```

**Homebrew:**
```bash
brew install --cask claude-code
```

After installing, start Claude in any project:

```bash
cd your-project
claude
```

You'll be prompted to log in on first use. After that:

```bash
# Generate a CLAUDE.md for your project (recommended first step)
/init
```

---

## 3. Daily Workflows

Claude Code understands plain English. Be specific — the more context you give, the better the result.

### Fix a bug
```
> fix the null pointer in UserService.getById when the user doesn't exist
> I'm getting "Cannot read property 'id' of undefined" in orders.service.ts line 47
```

### Write new code
```
> add a DELETE /users/:id endpoint to the users router, 
  follow the same pattern as the existing POST endpoint
```

### Understand unfamiliar code
```
> explain how the retry logic in src/queue/worker.ts works
> what happens when a payment fails? trace the full flow from the API call
```

### Write and run tests
```
> write unit tests for the checkout flow, run them, and fix any failures
```
Claude writes, executes, reads failures, and iterates until they pass.

### Commit and PR
```
> commit these changes with a meaningful message
> create a PR for this feature with a description
```

### Reference files with `@`
```
> look at @src/api/auth.ts and explain why login sometimes returns 401
> compare @src/old-service.ts and @src/new-service.ts, migrate missing methods
```

### Pipe output from your terminal
```bash
tail -200 app.log | claude -p "summarize any errors"
npm test 2>&1 | claude -p "what's failing and why?"
git diff main --name-only | claude -p "review these changes for security issues"
```

### Plan before acting (for risky changes)
```
/plan
> replace all uses of Logger with StructuredLogger — show me the plan first
```

`/plan` mode = read-only. Claude can explore and plan but **cannot edit files**. Review the plan, then toggle off to execute.

---

## 4. Permissions — What Gets Asked & Why

Claude asks before running commands or editing files. You'll see:

```
Claude wants to run: npm test
[Allow once]  [Allow always]  [Deny]
```

- **Allow once** — runs this time, asks again next time
- **Allow always** — saves a rule to `.claude/settings.local.json` (your personal, gitignored file)
- **Deny** — blocks it, Claude tries another approach

**Tip:** For commands you run constantly (`npm test`, `git status`, `npm run lint`), choose **Allow always** to stop interruptions.

### Permission modes

Switch modes based on what you're doing:

| Mode | Behavior | When to use |
|---|---|---|
| `default` | Prompts on first use of each tool | Standard work |
| `acceptEdits` | Auto-approves file edits in working dir | Active development |
| `plan` | Read-only, no edits or commands | Architecture review |
| `auto` | AI safety classifier decides | Automated pipelines |
| `bypassPermissions` | Skip all prompts (dangerous) | Containers/VMs only |

```bash
# View and manage all rules
/permissions
```

### Rule syntax (for settings.json)

```json
{
  "permissions": {
    "allow": [
      "Bash(npm run *)",           // any npm run command
      "Bash(git commit *)",
      "Read(~/.zshrc)",
      "WebFetch(domain:api.github.com)"
    ],
    "deny": [
      "Bash(git push *)",          // block all git push
      "Read(./.env)",              // protect env files
      "Read(./secrets/**)"
    ]
  }
}
```

**Evaluation order: `deny` > `ask` > `allow` — deny always wins.**

### Path anchoring for Read/Edit rules

| Pattern | Anchors to |
|---|---|
| `//path` | Absolute filesystem root (`//Users/alice/file`) |
| `~/path` | Home directory (`~/.zshrc`) |
| `/path` | Project root (`/src/**/*.ts`) |
| `path` | Current directory (`*.env`) |

> `/Users/alice/file` is **NOT absolute** — it's relative to project root. Use `//Users/alice/file` for absolute.

---

## 5. How Claude Remembers Your Project

Each session starts fresh — no conversation memory from last time. But your project knowledge persists in two ways.

### CLAUDE.md — Instructions you write

A Markdown file that Claude reads at the start of every session.

```
your-project/
├── CLAUDE.md          ← team instructions (commit this)
├── CLAUDE.local.md    ← your personal notes (gitignored)
└── .claude/
    └── CLAUDE.md      ← alternative location (same behavior)
```

**What to put in CLAUDE.md:**

```markdown
# Project Setup
- Use pnpm, never npm
- Run `pnpm test` before every commit
- API handlers live in `src/api/handlers/`

# Code Conventions
- Use the Result<T, E> type for errors, never throw
- All DB queries must use parameterized statements
- 2-space indentation, single quotes

# Architecture
- Auth is handled in src/auth/ — see AuthService for token logic
- Queue jobs are in src/queue/ — use BullMQ patterns
```

**Rules of thumb:**
- Keep it under **200 lines** — longer files reduce adherence
- Be specific: `"Use 2-space indentation"` beats `"Format code nicely"`
- Import other files with `@path/to/file` syntax
- Run `/init` to auto-generate one from your codebase
- Run `/memory` to verify which files are loaded in your session

**When Claude makes the same mistake twice → add it to CLAUDE.md.**

### Scope hierarchy (more specific wins)

| Scope | Location | Who sees it |
|---|---|---|
| Organization | `/etc/claude-code/CLAUDE.md` | Everyone in org |
| Project | `./CLAUDE.md` or `./.claude/CLAUDE.md` | Whole team (via git) |
| User | `~/.claude/CLAUDE.md` | You, all projects |
| Local | `./CLAUDE.local.md` | You, this project only |

### Path-scoped rules in `.claude/rules/`

Rules that only load when Claude works with matching files:

```markdown
<!-- .claude/rules/api-design.md -->
---
paths:
  - "src/api/**/*.ts"
---

# API Rules
- All endpoints must include input validation
- Use standard error response format
- Include OpenAPI doc comments
```

Rules **without** `paths` load every session. Rules **with** `paths` load only when relevant files are opened. This keeps context lean.

### Auto Memory — Notes Claude writes itself

As you work, Claude saves its own notes:
- Build commands it discovered
- Debugging patterns
- Preferences you corrected

Stored at `~/.claude/projects/<repo>/memory/MEMORY.md`. Run `/memory` to see and edit.

```json
// Disable if you don't want it
{ "autoMemoryEnabled": false }
```

---

## 6. The `.claude` Folder — Full Map

```
your-project/
├── CLAUDE.md                    ← project instructions (commit)
├── CLAUDE.local.md              ← personal overrides (gitignored)
├── .mcp.json                    ← MCP server definitions (commit)
└── .claude/
    ├── settings.json            ← team settings (commit)
    ├── settings.local.json      ← personal settings (gitignored)
    ├── CLAUDE.md                ← alternative to root CLAUDE.md
    ├── rules/                   ← scoped instruction files
    │   ├── testing.md           ← loaded every session (no paths)
    │   └── api-design.md        ← loaded only for matching files
    ├── skills/                  ← custom /slash-commands
    │   ├── deploy/
    │   │   └── SKILL.md         ← /deploy command
    │   └── review-pr/
    │       └── SKILL.md         ← /review-pr command
    ├── agents/                  ← specialized sub-agents
    │   └── code-reviewer.md
    └── hooks/                   ← hook scripts (referenced in settings)
        └── pre-tool.sh

~/.claude/                       ← your personal config (all projects)
├── CLAUDE.md                    ← personal instructions everywhere
├── settings.json                ← personal settings
├── skills/                      ← personal slash commands
├── agents/                      ← personal sub-agents
└── projects/<repo>/memory/      ← auto memory per project
    ├── MEMORY.md                ← index (loaded every session)
    └── debugging.md             ← topic files (loaded on demand)
```

---

## 7. Settings (settings.json)

### File locations and precedence

```
Managed settings       ← Highest — org policy, cannot be overridden
CLI arguments          ← Session-only
.claude/settings.local.json  ← Your personal project overrides (gitignored)
.claude/settings.json        ← Shared team settings (committed)
~/.claude/settings.json      ← Your personal global settings
```

Add `"$schema": "https://json.schemastore.org/claude-code-settings.json"` for editor autocomplete.

### Common settings

```json
{
  "$schema": "https://json.schemastore.org/claude-code-settings.json",

  "model": "claude-sonnet-4-6",
  "defaultMode": "acceptEdits",
  "language": "english",

  "permissions": {
    "allow": ["Bash(npm run *)", "Bash(git commit *)"],
    "deny": ["Bash(git push *)", "Read(./.env)"]
  },

  "env": {
    "NODE_ENV": "development"
  },

  "hooks": { },

  "autoMemoryEnabled": true,

  "attribution": {
    "commit": "Co-authored by Claude Code",
    "pr": ""
  },

  "cleanupPeriodDays": 30
}
```

### Key settings reference

| Setting | What it controls | Example |
|---|---|---|
| `model` | Default model | `"claude-sonnet-4-6"` |
| `defaultMode` | Permission mode | `"acceptEdits"` |
| `effortLevel` | Thinking depth | `"low"` / `"medium"` / `"high"` |
| `language` | Claude's response language | `"japanese"` |
| `autoMemoryEnabled` | Toggle auto memory | `false` |
| `hooks` | Lifecycle automation | See Section 8 |
| `env` | Environment variables per session | `{"FOO": "bar"}` |
| `outputStyle` | Adjust system prompt style | `"Explanatory"` |
| `claudeMdExcludes` | Ignore specific CLAUDE.md files | `["**/other-team/CLAUDE.md"]` |
| `enableAllProjectMcpServers` | Auto-approve project MCP | `true` |
| `disableAllHooks` | Kill all hooks | `true` |
| `cleanupPeriodDays` | Session file retention | `20` |
| `plansDirectory` | Where plan files are stored | `"./plans"` |
| `autoUpdatesChannel` | Release channel | `"stable"` |

---

## 8. Hooks — Automate Around Claude's Actions

Hooks are scripts that run automatically at points in Claude's lifecycle — before/after tool calls, when sessions start, when files change.

### Where to configure

```json
// .claude/settings.json (team) or ~/.claude/settings.json (personal)
{
  "hooks": {
    "PostToolUse": [
      {
        "matcher": "Edit|Write",
        "hooks": [
          {
            "type": "command",
            "command": "\"$CLAUDE_PROJECT_DIR\"/.claude/hooks/lint.sh"
          }
        ]
      }
    ]
  }
}
```

### All hook events

| Event | Fires when | Can block? |
|---|---|---|
| `SessionStart` | Session begins or resumes | No |
| `InstructionsLoaded` | A CLAUDE.md or rules file is loaded | No |
| `UserPromptSubmit` | Before Claude processes your message | Yes |
| `PreToolUse` | Before any tool call (Bash, Edit, Read…) | Yes |
| `PermissionRequest` | Before a permission dialog appears | Yes |
| `PostToolUse` | After a tool completes successfully | No |
| `PostToolUseFailure` | After a tool fails | No |
| `PermissionDenied` | When auto-mode classifier blocks a tool | No |
| `Stop` | When Claude finishes a response | Yes |
| `SubagentStart` / `SubagentStop` | Sub-agent spawns or finishes | Stop: Yes |
| `FileChanged` | A watched file changes on disk | No |
| `SessionEnd` | Session terminates | No |
| `PreCompact` / `PostCompact` | Before/after context compaction | No |

### Exit codes — critical to get right

```bash
exit 0   # success
exit 2   # BLOCKING — stderr shown to Claude, action is stopped
exit 1   # non-blocking — logged only, execution continues
```

**Always use `exit 2` to enforce policies. `exit 1` does NOT block anything.**

### Hook types

| Type | Use for |
|---|---|
| `command` | Shell scripts |
| `http` | POST to an external endpoint |
| `prompt` | LLM evaluation |
| `agent` | Subagent-based verification |

### Hook input — every hook receives this JSON on stdin

```json
{
  "session_id": "abc123",
  "hook_event_name": "PreToolUse",
  "tool_name": "Bash",
  "tool_input": { "command": "npm test" },
  "cwd": "/path/to/project",
  "permission_mode": "default"
}
```

### Practical examples

**Auto-lint after every file edit:**
```json
{
  "hooks": {
    "PostToolUse": [{
      "matcher": "Edit|Write",
      "hooks": [{ "type": "command", "command": ".claude/hooks/lint.sh" }]
    }]
  }
}
```

**Block dangerous commands:**
```bash
#!/bin/bash
# .claude/hooks/safety.sh
INPUT=$(cat)
COMMAND=$(echo "$INPUT" | jq -r '.tool_input.command // ""')

if echo "$COMMAND" | grep -qE 'rm -rf|DROP TABLE|git push --force'; then
  echo '{"hookSpecificOutput":{"hookEventName":"PreToolUse","permissionDecision":"deny","permissionDecisionReason":"Dangerous command blocked"}}'
  exit 0
fi
exit 0
```

```json
{
  "hooks": {
    "PreToolUse": [{ "matcher": "Bash", "hooks": [{ "type": "command", "command": ".claude/hooks/safety.sh" }] }]
  }
}
```

**Log all tool calls to a file:**
```bash
#!/bin/bash
echo "$(date) $HOOK_EVENT_NAME: $(cat | jq -c .)" >> ~/.claude/activity.log
exit 0
```

**Matcher patterns:**

| Matcher | Matches |
|---|---|
| `"Bash"` | All Bash commands |
| `"Edit\|Write"` | Edit or Write tool |
| `"mcp__memory__.*"` | All memory MCP tools (regex) |
| `"*"` or omitted | Everything |
| `"startup"` | SessionStart source = startup |

### View configured hooks
```
/hooks
```

---

## 9. Skills — Custom Slash Commands

Skills are `/slash-commands` that package repeatable team workflows.

### File structure

```
.claude/skills/review-pr/
├── SKILL.md          ← required (instructions + frontmatter)
├── template.md       ← optional supporting file
└── scripts/
    └── check.sh
```

### Where skills live

| Location | Path | Scope |
|---|---|---|
| Personal | `~/.claude/skills/<name>/SKILL.md` | All your projects |
| Project | `.claude/skills/<name>/SKILL.md` | This project |
| Legacy | `.claude/commands/<name>.md` | Still works |

Type `/` to see all available skills in your session.

### SKILL.md structure

```yaml
---
name: review-pr
description: Review a PR for security, quality, and standards. Use when reviewing pull requests.
disable-model-invocation: true   # only YOU can invoke, not Claude automatically
allowed-tools: Bash(gh *) Read Grep
---

Review PR #$ARGUMENTS:

## Context
- Diff: !`gh pr diff $ARGUMENTS`
- Files changed: !`gh pr diff $ARGUMENTS --name-only`
- Description: !`gh pr view $ARGUMENTS`

## Check for
1. Security vulnerabilities
2. Adherence to CLAUDE.md conventions  
3. Test coverage for new code
4. Breaking changes

Provide feedback with file:line references.
```

**Run it with:** `/review-pr 123`

### Key frontmatter fields

| Field | Purpose |
|---|---|
| `name` | The `/command-name` |
| `description` | How Claude decides when to use it (max 250 chars) |
| `disable-model-invocation: true` | Only YOU invoke it (not Claude auto) — use for deploys, commits |
| `user-invocable: false` | Hidden from `/` menu, Claude-only |
| `allowed-tools` | Pre-approved tools while skill runs |
| `model` | Override model for this skill |
| `effort` | `low` / `medium` / `high` / `max` |
| `context: fork` | Run in isolated subagent context |
| `agent` | Which subagent type to use with `context: fork` |
| `paths` | Only activate for matching file patterns |

### Variable substitutions

| Variable | Value |
|---|---|
| `$ARGUMENTS` | Full argument string |
| `$0`, `$1`, `$2` | Individual arguments by position |
| `${CLAUDE_SESSION_ID}` | Current session ID |
| `${CLAUDE_SKILL_DIR}` | Directory of the SKILL.md file |

### Dynamic shell injection

The `` !`command` `` syntax runs **before Claude sees anything** — output replaces the placeholder:

```yaml
---
name: standup
---
## Yesterday's commits
!`git log --oneline --since=yesterday --author="$(git config user.email)"`

## Open PRs assigned to me  
!`gh pr list --assignee @me`

Summarize what I worked on and what needs attention today.
```

### Invocation control

| Frontmatter | You can invoke | Claude can invoke |
|---|---|---|
| (default) | Yes | Yes |
| `disable-model-invocation: true` | Yes | No |
| `user-invocable: false` | No | Yes |

---

## 10. Sub-Agents — Specialized Assistants

Sub-agents are AI assistants with their own system prompt, tools, and context window. Claude delegates tasks to them automatically when the task matches their description.

### File locations

```
.claude/agents/code-reviewer.md    ← project (committed)
~/.claude/agents/code-reviewer.md  ← personal (all projects)
```

### Agent file structure

```markdown
---
name: code-reviewer
description: Reviews code for security issues and best practices. Use for security audits, PR reviews, and compliance checks.
model: claude-opus-4-6
allowed-tools: Read Grep Glob
disallowed-tools: Bash Edit Write
memory: enabled
---

You are a security-focused code reviewer.

Never modify files. Only read and report findings.
Focus on: injection vulnerabilities, auth bypasses, secrets in code, insecure dependencies.
```

### Key frontmatter fields

| Field | Purpose |
|---|---|
| `name` | Agent identifier |
| `description` | When Claude delegates to this agent |
| `model` | Override model (e.g. Opus for heavy review) |
| `allowed-tools` | Tools this agent can use |
| `disallowed-tools` | Blocked tools |
| `memory: enabled` | Agent maintains its own auto memory |

### When to use sub-agents vs skills

- **Skill** — a workflow you trigger manually (deploy, commit, PR review)
- **Sub-agent** — a specialized assistant Claude delegates to automatically (security reviewer, documentation writer, test generator)

---

## 11. MCP — Connect External Tools

MCP (Model Context Protocol) lets Claude connect to external tools: GitHub, Jira, Slack, databases, internal APIs.

### Config file locations

| Scope | File | Committed? |
|---|---|---|
| Project | `.mcp.json` in repo root | Yes |
| User | `~/.claude.json` | No |
| Org | `/etc/claude-code/managed-mcp.json` | Via IT |

### .mcp.json structure

```json
{
  "mcpServers": {
    "github": {
      "type": "stdio",
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-github"],
      "env": { "GITHUB_TOKEN": "${GITHUB_TOKEN}" }
    },
    "memory": {
      "type": "stdio",
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-memory"]
    },
    "my-internal-api": {
      "type": "http",
      "url": "http://localhost:8080/mcp",
      "headers": { "Authorization": "Bearer ${API_TOKEN}" }
    }
  }
}
```

### CLI management

```bash
claude mcp add github -e GITHUB_TOKEN=$GITHUB_TOKEN -- npx @modelcontextprotocol/server-github
claude mcp list
claude mcp remove github
```

### Auto-approve project MCP servers

```json
{
  "enableAllProjectMcpServers": true,
  "enabledMcpjsonServers": ["github", "memory"]
}
```

---

## 12. CLI & Slash Command Reference

### Terminal flags

```bash
claude                          # interactive session
claude -p "prompt here"         # non-interactive, print and exit
claude --continue               # continue last session
claude --resume <session-id>    # resume specific session
claude --add-dir ../shared      # add file access to another dir
claude --model claude-opus-4-6  # use specific model
claude --permission-mode auto   # set permission mode for session
claude --output-format json     # structured JSON output
```

### Slash commands (type inside Claude)

| Command | What it does |
|---|---|
| `/init` | Generate CLAUDE.md for project |
| `/memory` | Browse and edit all memory and instruction files |
| `/permissions` | View and manage permission rules |
| `/hooks` | Browse all configured hooks |
| `/config` | Open settings UI |
| `/plan` | Toggle read-only plan mode |
| `/compact` | Compress context (keeps instructions, summarizes chat) |
| `/clear` | Start fresh context window |
| `/model` | Switch models mid-session |
| `/effort` | Set effort level (low / medium / high) |
| `/fast` | Toggle fast mode |
| `/add-dir <path>` | Add directory access mid-session |
| `/simplify` | Review and simplify recent changes |
| `/schedule` | Create a scheduled recurring task |
| `/help` | Show all available commands |

---

## 13. Tips & Troubleshooting

### Getting better results

**Be specific about scope:**
```
# Vague
> fix the auth module

# Good
> fix token refresh in src/auth/refresh.ts — it doesn't handle expired refresh tokens
```

**Correct mistakes out loud — then add to CLAUDE.md:**
```
> no, we use camelCase, not snake_case. fix that and remember it.
```
If it should stick permanently, add it to `CLAUDE.md` or let Claude add it to auto memory.

**Interrupt safely:** Press `Escape` to stop Claude mid-task. Changes already made remain — only future actions are stopped.

**Use `/plan` for risky changes:** Before any large refactor, rename, or deletion — enter plan mode, review the plan, only then execute.

**Pipe real data in:**
```bash
cat error.log | claude -p "what's causing these errors?"
git diff | claude -p "summarize what changed in plain English"
```

### Common problems

| Problem | Fix |
|---|---|
| Claude keeps making the same mistake | Add the rule to `CLAUDE.md` |
| Claude isn't following project conventions | Run `/memory` — check which files are loaded |
| Session feels slow or off-track | Run `/compact` or `/clear` |
| I approved something I shouldn't have | `/permissions` → remove the rule |
| Claude won't stop asking permission for X | `/permissions` → add allow rule, e.g. `Bash(npm run test *)` |
| CLAUDE.md changes not being picked up | Restart the session — `/clear` |
| Hook not firing | Run `/hooks` to verify it's registered; check `exit 2` vs `exit 1` |
| Skill not triggering automatically | Check `description` matches user intent; verify `disable-model-invocation` is not set |

### Debugging CLAUDE.md loading

```bash
# Inside a session
/memory     ← lists every file Claude loaded and why
```

If a file isn't listed there, Claude cannot see it. Check the path and scope.

### Things Claude Code will NOT do (without explicit approval)

- Push to remote branches
- Delete files without confirming
- Access files outside your project directory
- Send your code anywhere except Anthropic's API

---

*Full documentation: https://code.claude.com/docs*
