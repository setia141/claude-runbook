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
| `dontAsk` | Auto-denies prompts (pre-approved tools still work) | Scripting with explicit allow rules |
| `bypassPermissions` | Skip all prompts (dangerous) | Containers/VMs only |

```bash
# View and manage all rules
/permissions
```

To prevent `bypassPermissions` or `auto` from being used, set `permissions.disableBypassPermissionsMode: "disable"` or `permissions.disableAutoMode: "disable"` in settings.

### Rule syntax (for settings.json)

```json
{
  "permissions": {
    "allow": [
      "Bash(npm run *)",           // any npm run command
      "Bash(git commit *)",
      "Read(~/.zshrc)",
      "WebFetch(domain:api.github.com)",
      "Agent(Explore)",            // allow Explore subagent
      "Skill(review-pr *)"         // allow review-pr skill with any args
    ],
    "deny": [
      "Bash(git push *)",          // block all git push
      "Read(./.env)",              // protect env files
      "Read(./secrets/**)"
    ],
    "ask": [
      "Bash(git push *)"           // prompt before git push
    ]
  }
}
```

**Evaluation order: `deny` > `ask` > `allow` — deny always wins.**

### Tool-specific rule formats

| Tool | Example | What it matches |
|---|---|---|
| `Bash` | `Bash(npm run *)` | Commands starting with `npm run ` |
| `Read` | `Read(./.env)` | Reading `.env` in project root |
| `Edit` | `Edit(/src/**)` | Edits under `<project>/src/` |
| `WebFetch` | `WebFetch(domain:api.github.com)` | Fetches to that domain |
| `mcp__server` | `mcp__puppeteer__navigate` | Specific MCP tool |
| `Agent` | `Agent(Explore)` | Specific subagent type |
| `Skill` | `Skill(deploy *)` | Specific skill invocation |

**Compound commands:** A rule like `Bash(safe-cmd *)` does NOT grant permission to `safe-cmd && other-cmd`. Claude Code checks each subcommand independently; a rule must match each one.

### Path anchoring for Read/Edit rules

| Pattern | Anchors to |
|---|---|
| `//path` | Absolute filesystem root (`//Users/alice/file`) |
| `~/path` | Home directory (`~/.zshrc`) |
| `/path` | Project root (`/src/**/*.ts`) |
| `path` or `./path` | Current directory (`*.env`) |

> `/Users/alice/file` is **NOT absolute** — it's relative to project root. Use `//Users/alice/file` for absolute paths.
> On Windows, paths normalize to POSIX: `C:\Users\alice` → `/c/Users/alice`, use `//c/**/.env` for drive-relative matching.

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

To customize the storage location:
```json
{ "autoMemoryDirectory": "~/my-memory-dir" }
```
Note: `autoMemoryDirectory` is not accepted in project settings (`.claude/settings.json`) to prevent shared repos from redirecting memory writes to sensitive locations.

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
Managed settings                 ← Highest — org policy, cannot be overridden
  ├── Server-managed (Claude.ai admin console)
  ├── MDM/OS-level policies (macOS plist, Windows registry)
  └── File: /etc/claude-code/managed-settings.json (Linux/WSL)
        C:\Program Files\ClaudeCode\managed-settings.json (Windows)
        /Library/Application Support/ClaudeCode/managed-settings.json (macOS)
CLI arguments                    ← Session-only
.claude/settings.local.json      ← Your personal project overrides (gitignored)
.claude/settings.json            ← Shared team settings (committed)
~/.claude/settings.json          ← Your personal global settings (lowest)
```

Add `"$schema": "https://json.schemastore.org/claude-code-settings.json"` at the top of any settings file for editor autocomplete and validation.

### Example settings.json

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

  "attribution": {
    "commit": "Co-authored by Claude Code",
    "pr": ""
  },

  "companyAnnouncements": [
    "Remember: all PRs require code review before merge"
  ],

  "cleanupPeriodDays": 30
}
```

### Core settings reference

| Setting | What it controls | Example |
|---|---|---|
| `model` | Default model | `"claude-sonnet-4-6"` |
| `effortLevel` | Thinking depth (`/effort low/medium/high`) | `"medium"` |
| `language` | Claude's response language | `"japanese"` |
| `outputStyle` | Adjust system prompt style | `"Explanatory"` |
| `env` | Environment variables applied every session | `{"FOO": "bar"}` |
| `cleanupPeriodDays` | Session file retention in days (default 30, min 1) | `20` |
| `autoUpdatesChannel` | Release channel: `"latest"` (default) or `"stable"` | `"stable"` |
| `plansDirectory` | Where plan files are stored (relative to project root) | `"./plans"` |
| `autoMemoryDirectory` | Custom path for auto memory storage | `"~/my-memory"` |
| `agent` | Run main thread as a named subagent | `"code-reviewer"` |
| `includeGitInstructions` | Include built-in git commit/PR instructions in system prompt (default `true`) | `false` |
| `companyAnnouncements` | Startup messages cycled at random | `["Welcome to Acme Corp!"]` |

### Model & auth settings

| Setting | What it controls | Example |
|---|---|---|
| `availableModels` | Restrict which models users can select | `["sonnet", "haiku"]` |
| `modelOverrides` | Map Anthropic model IDs to provider-specific IDs (Bedrock ARNs etc.) | `{"claude-opus-4-6": "arn:..."}` |
| `apiKeyHelper` | Script to generate API key / auth value | `"/bin/generate_key.sh"` |
| `awsAuthRefresh` | Script to refresh AWS credentials | `"aws sso login --profile myprofile"` |
| `awsCredentialExport` | Script that outputs JSON with AWS credentials | `"/bin/generate_aws_grant.sh"` |
| `forceLoginMethod` | Restrict login to `"claudeai"` or `"console"` accounts | `"claudeai"` |
| `forceLoginOrgUUID` | Require login to belong to a specific org UUID | `"xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx"` |
| `otelHeadersHelper` | Script to generate dynamic OpenTelemetry headers | `"/bin/gen_otel_headers.sh"` |
| `feedbackSurveyRate` | Probability (0–1) that session quality survey appears | `0.05` |

### Permissions settings (nested under `"permissions"`)

| Setting | What it controls | Example |
|---|---|---|
| `allow` | Rules to allow tool use without asking | `["Bash(npm run *)"]` |
| `deny` | Rules to block tool use entirely | `["Bash(git push *)"]` |
| `ask` | Rules to always prompt before tool use | `["Bash(git push *)"]` |
| `additionalDirectories` | Extra directories for file access | `["../shared/"]` |
| `defaultMode` | Default permission mode at startup | `"acceptEdits"` |
| `disableBypassPermissionsMode` | Block `bypassPermissions` mode | `"disable"` |
| `disableAutoMode` | Block `auto` mode | `"disable"` |
| `skipDangerousModePermissionPrompt` | Skip the bypass-permissions confirmation prompt | `true` |

### UI & UX settings

| Setting | What it controls | Example |
|---|---|---|
| `viewMode` | Default transcript view: `"default"`, `"verbose"`, `"focus"` | `"verbose"` |
| `alwaysThinkingEnabled` | Enable extended thinking by default | `true` |
| `showThinkingSummaries` | Show thinking block summaries in session | `true` |
| `spinnerTipsEnabled` | Show tips while Claude works (default `true`) | `false` |
| `spinnerTipsOverride` | Override spinner tips with custom strings | `{"excludeDefault": true, "tips": ["Use tool X"]}` |
| `spinnerVerbs` | Customize action verbs in spinner messages | `{"mode": "append", "verbs": ["Pondering"]}` |
| `prefersReducedMotion` | Reduce/disable UI animations | `true` |
| `respectGitignore` | `@` file picker respects `.gitignore` (default `true`) | `false` |
| `showClearContextOnPlanAccept` | Show "clear context" on plan accept screen | `true` |
| `voiceEnabled` | Enable push-to-talk voice dictation | `true` |
| `fastModePerSessionOptIn` | Fast mode resets each session (user must re-enable with `/fast`) | `true` |
| `defaultShell` | Shell for `!` commands: `"bash"` or `"powershell"` | `"powershell"` |
| `statusLine` | Custom status line display | `{"type": "command", "command": "~/.claude/statusline.sh"}` |
| `fileSuggestion` | Custom script for `@` file autocomplete | `{"type": "command", "command": "~/.claude/file-suggestion.sh"}` |
| `attribution` | Customize git commit/PR attribution text | `{"commit": "Co-authored by Claude", "pr": ""}` |
| `useAutoModeDuringPlan` | Use auto mode semantics during plan mode (default `true`) | `false` |

### Hooks & MCP settings

| Setting | What it controls | Example |
|---|---|---|
| `hooks` | Lifecycle automation (see Section 8) | See Section 8 |
| `disableAllHooks` | Disable all hooks and custom status line | `true` |
| `allowedHttpHookUrls` | Allowlist of URL patterns for HTTP hooks (`*` wildcard) | `["https://hooks.example.com/*"]` |
| `httpHookAllowedEnvVars` | Allowlist of env var names HTTP hooks may interpolate | `["MY_TOKEN"]` |
| `enableAllProjectMcpServers` | Auto-approve all project `.mcp.json` servers | `true` |
| `enabledMcpjsonServers` | Allowlist of specific `.mcp.json` servers to approve | `["github", "memory"]` |
| `disabledMcpjsonServers` | List of `.mcp.json` servers to reject | `["filesystem"]` |
| `disableSkillShellExecution` | Disable `` !`...` `` shell injection in skills/commands | `true` |

### Worktree settings

| Setting | What it controls | Example |
|---|---|---|
| `worktree.symlinkDirectories` | Directories to symlink into each worktree (saves disk) | `["node_modules", ".cache"]` |
| `worktree.sparsePaths` | Paths to check out in worktrees (sparse-checkout) | `["packages/my-app", "shared/utils"]` |

### Sandbox settings (nested under `"sandbox"`)

Sandboxing isolates Bash commands from the filesystem and network (macOS, Linux, WSL2).

| Setting | What it controls | Example |
|---|---|---|
| `enabled` | Enable bash sandboxing (default `false`) | `true` |
| `failIfUnavailable` | Exit if sandbox can't start (default `false`) | `true` |
| `autoAllowBashIfSandboxed` | Auto-approve bash when sandboxed (default `true`) | `false` |
| `excludedCommands` | Commands that bypass the sandbox | `["docker *"]` |
| `allowUnsandboxedCommands` | Allow `dangerouslyDisableSandbox` escape hatch (default `true`) | `false` |
| `filesystem.allowWrite` | Paths where sandboxed commands can write | `["/tmp/build", "~/.kube"]` |
| `filesystem.denyWrite` | Paths where sandboxed commands cannot write | `["/etc", "/usr/local/bin"]` |
| `filesystem.denyRead` | Paths where sandboxed commands cannot read | `["~/.aws/credentials"]` |
| `filesystem.allowRead` | Re-allow reading within `denyRead` regions | `["."]` |
| `network.allowUnixSockets` | Unix socket paths accessible in sandbox | `["~/.ssh/agent-socket"]` |
| `network.allowAllUnixSockets` | Allow all Unix socket connections (default `false`) | `true` |

### Global config settings (`~/.claude.json` — NOT settings.json)

These live in `~/.claude.json`. Adding them to `settings.json` triggers a schema validation error.

| Setting | What it controls | Example |
|---|---|---|
| `editorMode` | Input key bindings: `"normal"` or `"vim"` | `"vim"` |
| `autoConnectIde` | Auto-connect to running IDE from external terminal | `true` |
| `autoInstallIdeExtension` | Auto-install IDE extension in VS Code terminal (default `true`) | `false` |
| `showTurnDuration` | Show "Cooked for Xm Xs" after responses (default `true`) | `false` |
| `terminalProgressBarEnabled` | Show terminal progress bar in supported terminals | `false` |
| `teammateMode` | How agent team teammates display: `"auto"`, `"in-process"`, `"tmux"` | `"in-process"` |

### Managed-only settings (ignored in user/project settings)

| Setting | What it controls |
|---|---|
| `allowManagedHooksOnly` | Block all user/project hooks; only managed hooks run |
| `allowManagedMcpServersOnly` | Only managed MCP allowlist respected |
| `allowManagedPermissionRulesOnly` | Block user/project allow/deny/ask rules |
| `allowedMcpServers` | Allowlist of MCP servers users can configure |
| `deniedMcpServers` | Denylist of MCP servers (overrides allowlist) |
| `blockedMarketplaces` | Block specific plugin marketplace sources |
| `strictKnownMarketplaces` | Restrict which plugin marketplaces users can add |
| `allowedChannelPlugins` | Allowlist of channel plugins (requires `channelsEnabled: true`) |
| `channelsEnabled` | Allow channels for Team/Enterprise users |
| `forceRemoteSettingsRefresh` | Block startup until remote settings freshly fetched |
| `pluginTrustMessage` | Custom text appended to plugin trust warning |
| `sandbox.filesystem.allowManagedReadPathsOnly` | Only managed `allowRead` paths respected |
| `companyAnnouncements` | Startup messages shown to all users |

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

### All 24 hook events

| Event | Fires when | Can block? |
|---|---|---|
| `SessionStart` | Session begins or resumes | No |
| `InstructionsLoaded` | A CLAUDE.md or rules file is loaded | No |
| `UserPromptSubmit` | Before Claude processes your message | Yes — exit 2 erases the prompt |
| `PreToolUse` | Before any tool call | Yes — exit 2 prevents the call |
| `PermissionRequest` | Before a permission dialog appears | Yes — can auto-approve or deny |
| `PermissionDenied` | Auto-mode classifier blocks a tool | No (use JSON `retry: true`) |
| `PostToolUse` | After a tool completes successfully | No (exit 2 shows error to Claude) |
| `PostToolUseFailure` | After a tool fails | No |
| `Notification` | Claude Code sends a notification | No |
| `SubagentStart` | Sub-agent spawns | No |
| `SubagentStop` | Sub-agent finishes | Yes |
| `TaskCreated` | Task created via TaskCreate | Yes — rolls back creation |
| `TaskCompleted` | Task marked complete | Yes |
| `Stop` | Claude finishes a response | Yes — prevents stopping |
| `StopFailure` | Turn ends due to API error | No |
| `TeammateIdle` | Agent team teammate about to idle | Yes |
| `ConfigChange` | Config file changes during session | Yes |
| `CwdChanged` | Working directory changes | No |
| `FileChanged` | A watched file changes on disk | No |
| `WorktreeCreate` | Worktree created | Yes |
| `WorktreeRemove` | Worktree removed | No |
| `PreCompact` | Before context compaction | No |
| `PostCompact` | After context compaction | No |
| `Elicitation` | MCP server requests user input | Yes |
| `ElicitationResult` | User responds to MCP elicitation | Yes |

**Matcher sources for key events:**
- `SessionStart` → `startup`, `resume`, `clear`, `compact`
- `FileChanged` → literal filenames: `".envrc|.env"`
- `PreToolUse` / `PostToolUse` → tool name: `Bash`, `Edit`, `Write`, `Read`, `Glob`, `Grep`, `Agent`, `WebFetch`, or MCP tool names

### Exit codes — critical to get right

```bash
exit 0   # success — JSON on stdout is parsed if present
exit 2   # BLOCKING — stderr shown to Claude/user, action stopped (for blocking events)
exit 1   # non-blocking — stderr logged, execution continues
```

**Always use `exit 2` to enforce policies. `exit 1` does NOT block anything.**

> Exception: For `PreToolUse` and `PermissionRequest`, use JSON output with `permissionDecision: "deny"` rather than `exit 2` — this gives Claude better context and allows modifying the input before allowing.

### Hook types

| Type | Use for |
|---|---|
| `command` | Shell scripts (most common) |
| `http` | POST to an external endpoint |
| `prompt` | Single-turn LLM evaluation |
| `agent` | Sub-agent based verification |

### Command hook fields

```json
{
  "type": "command",
  "command": "\"$CLAUDE_PROJECT_DIR\"/.claude/hooks/script.sh",
  "async": false,         // true = fire and forget
  "asyncRewake": false,   // true = fire and forget, but wake Claude on exit 2
  "shell": "bash",        // or "powershell"
  "timeout": 600,         // seconds
  "statusMessage": "Running lint..."
}
```

**Environment variables injected into every hook:**
- `CLAUDE_PROJECT_DIR` — project root directory
- `CLAUDE_PLUGIN_ROOT` — plugin installation directory
- `CLAUDE_PLUGIN_DATA` — plugin persistent data directory
- `CLAUDE_ENV_FILE` — file path for persisting env vars across turns (`SessionStart`, `CwdChanged`, `FileChanged` only)
- `CLAUDE_CODE_REMOTE` — set to `"true"` in web environments

### HTTP hook fields

```json
{
  "type": "http",
  "url": "https://hooks.example.com/pre-tool",
  "timeout": 30,
  "headers": { "Authorization": "Bearer $MY_TOKEN" },
  "allowedEnvVars": ["MY_TOKEN"]
}
```

### Hook input — every hook receives this JSON on stdin

```json
{
  "session_id": "abc123",
  "transcript_path": "/path/to/transcript.jsonl",
  "hook_event_name": "PreToolUse",
  "tool_name": "Bash",
  "tool_input": { "command": "npm test" },
  "cwd": "/path/to/project",
  "permission_mode": "default",

  // Inside a subagent:
  "agent_id": "agent-abc123",
  "agent_type": "Explore"
}
```

### JSON output schema

All hooks can output JSON on stdout that Claude reads:

```json
{
  "continue": true,
  "stopReason": "Build failed",
  "suppressOutput": false,
  "systemMessage": "Warning: linting errors found"
}
```

**PreToolUse — rich permission control:**

```json
{
  "hookSpecificOutput": {
    "hookEventName": "PreToolUse",
    "permissionDecision": "allow|deny|ask|defer",
    "permissionDecisionReason": "Reason shown to user",
    "updatedInput": { "command": "modified command" },
    "additionalContext": "Context for Claude"
  }
}
```

**Stop / PostToolUse — block with reason:**

```json
{
  "decision": "block",
  "reason": "Tests are failing, please fix before continuing"
}
```

### Matcher pattern rules

| Pattern | Evaluation |
|---|---|
| `"*"`, `""`, or omitted | Match everything |
| Letters, digits, `_`, `\|` | Exact string or pipe-separated list: `Edit\|Write` |
| Any other character | JavaScript regex: `mcp__memory__.*`, `^Notebook` |

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
  echo '{
    "hookSpecificOutput": {
      "hookEventName": "PreToolUse",
      "permissionDecision": "deny",
      "permissionDecisionReason": "Dangerous command blocked by policy"
    }
  }'
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

**Reload env vars when .env changes:**
```bash
#!/bin/bash
# .claude/hooks/reload-env.sh
# Writes to CLAUDE_ENV_FILE which Claude reads on next turn
if [ -f .env ]; then
  cat .env > "$CLAUDE_ENV_FILE"
fi
exit 0
```

```json
{
  "hooks": {
    "FileChanged": [{ "matcher": ".env", "hooks": [{ "type": "command", "command": ".claude/hooks/reload-env.sh" }] }]
  }
}
```

**Log all tool calls:**
```bash
#!/bin/bash
echo "$(date) $HOOK_EVENT_NAME: $(cat | jq -c .)" >> ~/.claude/activity.log
exit 0
```

### View configured hooks
```
/hooks
```

---

## 9. Skills — Custom Slash Commands

Skills are `/slash-commands` that package repeatable team workflows.

> **Skills extend the [Agent Skills](https://agentskills.io) open standard.** Existing `.claude/commands/` files still work — skills add extra features (supporting files, invocation control, dynamic context).

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
| Enterprise | Managed settings | All org users |
| Personal | `~/.claude/skills/<name>/SKILL.md` | All your projects |
| Project | `.claude/skills/<name>/SKILL.md` | This project |
| Legacy commands | `.claude/commands/<name>.md` | Still works (same behavior) |

Type `/` to see all available skills in your session. Skills also load from nested directories (e.g. `packages/frontend/.claude/skills/`) for monorepo setups.

### SKILL.md structure

```yaml
---
name: review-pr
description: Review a PR for security, quality, and standards. Use when reviewing pull requests.
disable-model-invocation: true   # only YOU can invoke, not Claude automatically
allowed-tools: Bash(gh *) Read Grep
argument-hint: "[pr-number]"
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

### Complete frontmatter reference

| Field | Purpose |
|---|---|
| `name` | The `/command-name` (lowercase, hyphens, max 64 chars). Defaults to directory name if omitted. |
| `description` | How Claude decides when to use it. Front-load the key use case. Max 250 chars shown in listing. |
| `argument-hint` | Hint shown during autocomplete, e.g. `"[issue-number]"` or `"[file] [format]"` |
| `disable-model-invocation` | `true` = only YOU invoke it. Use for deploys, commits, destructive ops. |
| `user-invocable` | `false` = hidden from `/` menu, Claude-only background knowledge |
| `allowed-tools` | Pre-approved tools while skill runs (space-separated or YAML list) |
| `model` | Override model for this skill |
| `effort` | `low` / `medium` / `high` / `max` (Opus 4.6 only). Overrides session effort level. |
| `context` | `fork` = run in isolated subagent context |
| `agent` | Which subagent type to use when `context: fork` is set |
| `hooks` | Lifecycle hooks scoped to this skill's execution |
| `paths` | Glob patterns — skill auto-activates only for matching files |
| `shell` | Shell for `` !`...` `` blocks: `"bash"` (default) or `"powershell"` |

### Variable substitutions

| Variable | Value |
|---|---|
| `$ARGUMENTS` | Full argument string as typed |
| `$ARGUMENTS[N]` | Specific argument by 0-based index (`$ARGUMENTS[0]` = first) |
| `$0`, `$1`, `$2` | Shorthand for `$ARGUMENTS[0]`, `$ARGUMENTS[1]`, `$ARGUMENTS[2]` |
| `${CLAUDE_SESSION_ID}` | Current session ID |
| `${CLAUDE_SKILL_DIR}` | Directory containing the SKILL.md file |

Multi-word arguments: wrap in quotes when invoking — `/skill "hello world" second` makes `$0` = `hello world`, `$1` = `second`.

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

For multi-line commands, use a fenced block opened with ` ```! `:

````markdown
## Environment
```!
node --version
npm --version
git status --short
```
````

To disable shell injection across all user/project skills (e.g. org policy), set `"disableSkillShellExecution": true` in settings. Bundled and managed skills are unaffected.

### Invocation control

| Frontmatter | You can invoke | Claude can invoke | In context? |
|---|---|---|---|
| (default) | Yes | Yes | Description always in context |
| `disable-model-invocation: true` | Yes | No | Not in context |
| `user-invocable: false` | No | Yes | Description always in context |

### Skill content lifecycle

Once invoked, skill content stays in the conversation for the session. During auto-compaction, the most recent invocation of each skill is re-attached (up to 5,000 tokens each, 25,000 tokens total budget). If a skill stops influencing behavior, re-invoke it after compaction.

### Supporting files

Keep `SKILL.md` under 500 lines. Move detailed reference material to separate files in the skill directory, and link to them from `SKILL.md`:

```markdown
## Additional resources
- For complete API details, see [reference.md](reference.md)
- For usage examples, see [examples.md](examples.md)
```

---

## 10. Sub-Agents — Specialized Assistants

Sub-agents are AI assistants with their own system prompt, tools, model, and context window. Claude delegates tasks to them automatically when the task matches their description — keeping exploration and side-task noise out of your main conversation.

> **Key distinction:** Sub-agents work within a single session. For multiple agents working in parallel across separate sessions, see Agent Teams in the advanced guide.

### Built-in sub-agents

| Agent | Model | Tools | When Claude uses it |
|---|---|---|---|
| `Explore` | Haiku (fast) | Read-only | Codebase search and analysis |
| `Plan` | Inherits | Read-only | Research during plan mode |
| `general-purpose` | Inherits | All tools | Complex multi-step tasks |
| `statusline-setup` | Sonnet | Read, Edit | When you run `/statusline` |
| `Claude Code Guide` | Haiku | Search, Fetch | When you ask about Claude Code features |

### File locations

```
.claude/agents/code-reviewer.md    ← project (committed)
~/.claude/agents/code-reviewer.md  ← personal (all projects)
```

Use `/agents` to create, edit, and manage sub-agents interactively. Or list all from the CLI: `claude agents`.

### Agent file structure

```markdown
---
name: code-reviewer
description: Reviews code for security issues and best practices. Use for security audits, PR reviews, and compliance checks.
model: claude-opus-4-6
tools: Read, Grep, Glob
disallowedTools: Bash, Edit, Write
memory: project
---

You are a security-focused code reviewer.

Never modify files. Only read and report findings.
Focus on: injection vulnerabilities, auth bypasses, secrets in code, insecure dependencies.
```

### Complete frontmatter reference

| Field | Required | Purpose |
|---|---|---|
| `name` | Yes | Unique identifier (lowercase letters and hyphens) |
| `description` | Yes | When Claude should delegate to this agent |
| `tools` | No | Tool allowlist (inherits all if omitted). Space or comma-separated. Use `Agent(worker, researcher)` to restrict which sub-agents this agent can spawn. |
| `disallowedTools` | No | Tool denylist, removed from inherited or specified list |
| `model` | No | `"sonnet"`, `"opus"`, `"haiku"`, full model ID, or `"inherit"` (default) |
| `permissionMode` | No | `"default"`, `"acceptEdits"`, `"auto"`, `"dontAsk"`, `"bypassPermissions"`, `"plan"` |
| `maxTurns` | No | Maximum agentic turns before the sub-agent stops |
| `skills` | No | Skills to preload at startup (full content injected, not just available) |
| `mcpServers` | No | MCP servers scoped to this sub-agent (inline definitions or references by name) |
| `hooks` | No | Lifecycle hooks scoped to this sub-agent |
| `memory` | No | Persistent memory scope: `"user"`, `"project"`, or `"local"` |
| `background` | No | `true` = always run as a background task (default `false`) |
| `effort` | No | Effort level override: `"low"`, `"medium"`, `"high"`, `"max"` (Opus 4.6 only) |
| `isolation` | No | `"worktree"` = run in temporary git worktree (auto-cleaned if no changes) |
| `color` | No | Display color: `red`, `blue`, `green`, `yellow`, `purple`, `orange`, `pink`, `cyan` |
| `initialPrompt` | No | Auto-submitted as first turn when agent runs as main session via `--agent` or `agent` setting |

### Model resolution order

When Claude invokes a sub-agent, the model is resolved as:
1. `CLAUDE_CODE_SUBAGENT_MODEL` environment variable (if set)
2. Per-invocation `model` parameter
3. Sub-agent definition's `model` field
4. Main conversation's model

### Preload skills into a sub-agent

```yaml
---
name: api-developer
description: Implement API endpoints following team conventions
skills:
  - api-conventions
  - error-handling-patterns
---

Implement API endpoints. Follow the conventions from the preloaded skills.
```

Full skill content is injected at startup. Sub-agents don't inherit skills from the parent conversation — list them explicitly.

### Scope MCP servers to a sub-agent

```yaml
---
name: browser-tester
description: Tests features using Playwright
mcpServers:
  - playwright:
      type: stdio
      command: npx
      args: ["-y", "@playwright/mcp@latest"]
  - github        # reference by name — reuses parent's connection
---
```

Inline definitions are connected when the sub-agent starts and disconnected when it finishes. The parent conversation doesn't see the sub-agent's MCP tools.

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
claude                                    # interactive session
claude -p "prompt here"                   # non-interactive, print and exit
claude --continue                         # continue last session
claude --resume <session-id>              # resume specific session
claude --add-dir ../shared                # add file access to another dir
claude --model claude-opus-4-6            # use specific model
claude --permission-mode auto             # set permission mode for session
claude --output-format json               # structured JSON output
claude --agent code-reviewer              # run session as named subagent
claude --agents '{"name": {...}}'         # define session-only subagents as JSON
claude auto-mode defaults                 # print built-in auto mode rules
claude auto-mode config                   # show effective auto mode config
claude auto-mode critique                 # AI review of your custom auto mode rules
claude agents                             # list all configured subagents (no session)
```

### Slash commands (type inside Claude)

| Command | What it does |
|---|---|
| `/init` | Generate CLAUDE.md for project |
| `/memory` | Browse and edit all memory and instruction files |
| `/permissions` | View and manage permission rules |
| `/hooks` | Browse all configured hooks |
| `/agents` | Create, edit, and manage sub-agents interactively |
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
| `/statusline` | Configure the custom status line |
| `/voice` | Toggle voice dictation |
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
| Hook not firing | Run `/hooks` to verify it's registered; check exit code semantics |
| Hook fires but doesn't block | Use `exit 2` — `exit 1` is non-blocking |
| Skill not triggering automatically | Check `description` matches user intent; verify `disable-model-invocation` is not set |
| Settings key has no effect | Verify key isn't a managed-only key; check you're editing the right file in the precedence chain |
| `autoMemoryEnabled` not recognized | Use `autoMemoryDirectory` instead (the correct key) |

### Debugging CLAUDE.md loading

```bash
# Inside a session
/memory     ← lists every file Claude loaded and why
```

If a file isn't listed there, Claude cannot see it. Check the path and scope.

### Things Claude Code will NOT do (without explicit approval)

- Push to remote branches
- Delete files without confirming
- Access files outside your project directory (unless `additionalDirectories` or `--add-dir` is set)
- Send your code anywhere except Anthropic's API

---

*Full documentation: https://code.claude.com/docs*
