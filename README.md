# MARVIN - Your AI Chief of Staff

MARVIN is a [Claude Code](https://claude.ai/code) template that turns your AI assistant into a persistent, context-aware chief of staff. It remembers your goals, tracks tasks across sessions, gives daily briefings, and connects to your tools — all through natural conversation.

> Named after the Paranoid Android from *The Hitchhiker's Guide to the Galaxy*. Unlike him, MARVIN is actually happy to help.

---

## Table of Contents

1. [What is MARVIN](#what-is-marvin)
2. [Prerequisites](#prerequisites)
3. [Architecture](#architecture)
   - [Template vs. Workspace](#template-vs-workspace)
   - [Directory Structure](#directory-structure)
   - [State & Session Lifecycle](#state--session-lifecycle)
   - [Skills System](#skills-system)
4. [Getting Started](#getting-started)
5. [Daily Usage](#daily-usage)
6. [Commands Reference](#commands-reference)
7. [Skills](#skills)
   - [Built-in Skills](#built-in-skills)
   - [Writing Your Own](#writing-your-own-skill)
8. [Integrations](#integrations)
   - [Available Integrations](#available-integrations)
   - [Auth Patterns](#auth-patterns)
   - [Building a New Integration](#building-a-new-integration)
9. [Extending MARVIN](#extending-marvin)
10. [Migrating from an Older Version](#migrating-from-an-older-version)
11. [Troubleshooting & FAQ](#troubleshooting--faq)
12. [Contributing](#contributing)
13. [About](#about)

---

## What is MARVIN

MARVIN is not a standalone app. It is a structured Claude Code workspace — a set of context files, slash commands, and skills that shape how Claude behaves when you open it in this directory.

**What that means in practice:**
- Claude reads your profile, goals, and session history on every startup
- Slash commands (`/start`, `/end`, `/update`) trigger structured workflows
- Skills encode reusable behaviors (briefings, commit reviews, content logging)
- Integrations connect Claude to external tools via MCP (Model Context Protocol)

**What you need to run it:** Claude Code and an Anthropic API key. Every message costs tokens — MARVIN is not free to run, but it's designed to be efficient.

**What MARVIN is good at:**
- Persistent memory across sessions (goals, tasks, open threads)
- Daily briefings that surface what matters
- Tracking progress on multi-week goals
- Acting as a thought partner, not just an executor
- Connecting to your tools (email, calendar, tickets, Slack)

**What MARVIN is not:**
- A standalone app or SaaS product
- A replacement for dedicated task managers
- Free to use (you pay Anthropic API costs)

---

## Prerequisites

- **Claude Code** — Install via `npm install -g @anthropic-ai/claude-code` (requires Node 18+)
- **Anthropic API key** — Set `ANTHROPIC_API_KEY` in your environment or `.env` file
- **Terminal comfort** — You interact with MARVIN through a terminal running Claude Code
- **Git** (optional but recommended) — For versioning your workspace

No Python, no Docker, no build step.

---

## Architecture

### Template vs. Workspace

MARVIN uses a two-directory model:

```
marvin-template/          ← This repo (the template)
├── .marvin/              # Setup scripts, integration configs, onboarding guide
├── .claude/commands/     # Slash command definitions
├── skills/               # Skill definitions
├── CLAUDE.md             # Context file Claude reads on every session
└── state/                # Placeholder state files (overwritten during setup)

~/marvin/                 ← Your workspace (created during onboarding)
├── CLAUDE.md             # Your personalized context (name, goals, style)
├── state/
│   ├── current.md        # Active priorities and open threads
│   ├── goals.md          # Work and personal goals with tracking
│   └── todos.md          # Running task list
├── sessions/             # Daily session logs (one file per day)
├── reports/              # Weekly summaries from /report
├── content/              # Your notes and content
│   └── log.md            # Content shipping log
├── skills/               # Inherited from template + any you add
├── .claude/commands/     # Inherited slash commands
└── .marvin-source        # Path pointer back to the template (for /sync)
```

**The key invariant:** the template is read-only source of truth for commands and skills. Your workspace is where all personal data lives. Running `/sync` copies new commands and skills from the template into your workspace without touching your data.

### Directory Structure

#### Template directories

| Path | Purpose |
|------|---------|
| `.marvin/` | Internal: onboarding guide, setup/migration scripts, integration configs |
| `.marvin/integrations/` | One subdirectory per integration with `README.md` and `setup.sh` |
| `.claude/commands/` | Markdown files that define slash commands |
| `skills/` | Skill directories, each containing a `SKILL.md` |
| `state/` | Placeholder state files shown to Claude during onboarding |

#### Workspace directories (created during onboarding)

| Path | Purpose |
|------|---------|
| `CLAUDE.md` | Primary context file — Claude reads this first on every session |
| `state/current.md` | Rolling state: priorities, open threads, recent context |
| `state/goals.md` | Goal tracking: work goals, personal goals, progress table |
| `state/todos.md` | Persistent task list |
| `sessions/YYYY-MM-DD.md` | Daily session logs written by `/end` and `/update` |
| `reports/` | Weekly reports from `/report` |
| `content/log.md` | Content shipping log (used by the `content-shipped` skill) |
| `.marvin-source` | Single-line file containing the path to the template directory |

### State & Session Lifecycle

Each Claude Code session follows this lifecycle:

```
Session start
    │
    ├─ Claude reads CLAUDE.md (your profile + how MARVIN works)
    ├─ /start loads state/current.md, state/goals.md, today's session log
    │   └─ Produces a briefing: priorities, alerts, goal progress
    │
    ├─ Work happens (natural conversation + slash commands)
    │
    ├─ /update writes a mid-session checkpoint to sessions/YYYY-MM-DD.md
    │   └─ Safe to run multiple times; appends, doesn't overwrite
    │
    └─ /end writes final session summary, updates state/current.md
        └─ Next session's /start picks up from here
```

`state/current.md` is the handoff document between sessions. It should always reflect your actual current state — MARVIN updates it at `/end` and reads it at `/start`.

Session logs in `sessions/` are append-only daily records. They accumulate over time and give MARVIN historical context ("what did we talk about last Tuesday?").

### Skills System

Skills are reusable behavior definitions stored in `skills/<name>/SKILL.md`. Each skill file is a markdown document with a YAML frontmatter header followed by instructions Claude follows when the skill is invoked.

```yaml
---
name: start
description: |
  Start MARVIN session with briefing. Loads context, reviews state, gives daily briefing.
license: MIT
compatibility: marvin
metadata:
  marvin-category: session
  user-invocable: true
  slash-command: /start
  model: default
  proactive: false
---
```

**How skills are invoked:**
- Skills with `slash-command` set are triggered by the corresponding `.claude/commands/` file
- Skills can reference each other (e.g., `/end` calls the `update` skill logic internally)
- Skills are plain markdown — Claude reads them as instructions, not code

**Built-in skills and their triggers:**

| Skill | Trigger | What it does |
|-------|---------|-------------|
| `start` | `/start` | Loads context, reads state, produces daily briefing |
| `end` | `/end` | Summarizes session, writes log, updates state |
| `update` | `/update` | Mid-session checkpoint, appends to today's log |
| `commit` | `/commit` | Reviews git diff, writes commit messages, confirms before committing |
| `daily-briefing` | Called by `start` | Core briefing logic (priorities, goals, alerts) |
| `content-shipped` | Called manually | Logs shipped content to `content/log.md` |
| `skill-creator` | Ask MARVIN | Scaffolds a new skill directory from the template |

---

## Getting Started

### 1. Clone the template

```bash
git clone https://github.com/SterlingChin/marvin-template.git marvin-template
cd marvin-template
```

### 2. Open in Claude Code

```bash
claude
```

### 3. Run onboarding

Say:
> "Help me set up MARVIN"

MARVIN will walk you through:

1. Collecting your name, role, goals, and communication style preference
2. Creating your personal workspace (default: `~/marvin`)
3. Personalizing `CLAUDE.md`, `state/goals.md`, and `state/current.md`
4. Optionally setting up a `marvin` shell shortcut
5. Optionally connecting integrations (Jira, MS365, Google, Slack)

When onboarding is complete, you work out of your **workspace** (`~/marvin`), not the template directory. Keep the template around — you need it for `/sync`.

---

## Daily Usage

Open your workspace:
```bash
cd ~/marvin && claude
# or, if you set up the shortcut:
marvin
```

### Start your session
```
/start
```
Produces a briefing: today's priorities, goal progress, open threads, any alerts.

### Work naturally
```
"Add a task: review the Q2 roadmap before Thursday"
"I finished the API migration"
"What's blocking the infra project?"
"Draft a Slack message to the team about tomorrow's outage"
```

MARVIN maintains context throughout the session. You don't need to re-explain what you're working on.

### Save progress mid-session
```
/update
```
Appends a checkpoint to today's session log. Run this when switching tasks or before a long break.

### End your session
```
/end
```
MARVIN summarizes what was covered, updates `state/current.md`, and writes the full session log. Next time you `/start`, it picks up from here.

---

## Commands Reference

| Command | Skill | What It Does |
|---------|-------|-------------|
| `/start` | `start` | Load context and give a daily briefing |
| `/end` | `end` | Summarize session, update state, write log |
| `/update` | `update` | Mid-session checkpoint (append to today's log) |
| `/report` | — | Generate a weekly summary of work and progress |
| `/commit` | `commit` | Review git diff and create a clean commit |
| `/code` | — | Open the workspace in your IDE (Cursor or VS Code) |
| `/sync` | — | Pull new commands and skills from the template |
| `/help` | — | Show available commands and connected integrations |

---

## Skills

### Built-in Skills

See the [Skills System](#skills-system) section in Architecture for the full breakdown. Source files live in `skills/` in both the template and your workspace.

### Writing Your Own Skill

Use the template at `skills/_template/SKILL.md` as your starting point, or ask MARVIN directly:

> "Create a new skill for tracking my weekly gym sessions"

MARVIN will use the `skill-creator` skill to scaffold the directory and file for you.

**Manual steps:**

1. Copy `skills/_template/` to `skills/your-skill-name/`
2. Edit `SKILL.md` — fill in the frontmatter and write the instructions
3. If you want a slash command: create `.claude/commands/your-skill-name.md` that references the skill
4. Test by invoking the command or asking MARVIN to run the skill

**Frontmatter fields:**

| Field | Required | Description |
|-------|----------|-------------|
| `name` | Yes | Unique identifier, matches directory name |
| `description` | Yes | What the skill does and when to use it |
| `license` | No | Defaults to MIT |
| `compatibility` | No | Set to `marvin` |
| `metadata.user-invocable` | No | Whether users can trigger this directly |
| `metadata.slash-command` | No | The `/command` that triggers it (if any) |
| `metadata.model` | No | `default`, `sonnet`, `opus`, etc. |
| `metadata.proactive` | No | If true, MARVIN may invoke without explicit request |

Skills are just markdown. The `SKILL.md` body is instructions Claude reads and follows — write them clearly, step by step.

---

## Integrations

### Available Integrations

| Integration | Services | Auth Method |
|-------------|----------|-------------|
| Google Workspace | Gmail, Calendar, Drive, Docs, Sheets, Slides | OAuth (browser) |
| Microsoft 365 | Outlook, Calendar, OneDrive, Teams, Planner, SharePoint | MCP `/mcp` auth |
| Atlassian | Jira, Confluence | MCP `/mcp` auth |
| Slack | Channels, messages, search | API token |
| Telegram | Mobile chat interface for MARVIN | Bot token + Python |
| Parallel Search | Web search (faster than built-in) | No auth required |

### Auth Patterns

MARVIN integrations use three authentication patterns:

**Pattern 1: MCP browser auth** (Atlassian, MS365)

Configured via `claude mcp add`. After adding, you must restart Claude Code and then type `/mcp` to authenticate through a browser window. MARVIN's onboarding tracks this with a `.onboarding-pending-auth` state file so it can resume guidance after restart.

```bash
# Example: adding MS365
claude mcp add ms365 -s user -- npx -y @softeria/ms-365-mcp-server --org-mode
# Then restart Claude Code, type /mcp, select ms365, authenticate
```

**Pattern 2: OAuth on first use** (Google Workspace)

Configured via `claude mcp add` with OAuth credentials in env vars. The browser auth happens automatically the first time you make a Google request — no `/mcp` needed.

```bash
claude mcp add google-workspace -s user \
    --env GOOGLE_OAUTH_CLIENT_ID="your-client-id" \
    --env GOOGLE_OAUTH_CLIENT_SECRET="your-client-secret" \
    -- uvx workspace-mcp --tools gmail drive calendar docs sheets slides
```

**Pattern 3: Token-based** (Slack)

Token passed directly as an environment variable. No browser auth needed — just a valid token.

```bash
claude mcp add slack -s user \
    -e SLACK_MCP_XOXP_TOKEN="xoxp-your-token" \
    -- npx -y slack-mcp-server@latest --transport stdio
```

**Scoping:** All integrations support `-s user` (available in all your Claude Code projects) or no flag (project-scoped, only in this workspace). Use `-s user` for personal tools, project-scope for work-specific accounts.

### Building a New Integration

See `.marvin/integrations/CLAUDE.md` for the full spec. Summary:

**Structure:**
```
.marvin/integrations/your-integration/
├── README.md     # Required sections: What It Does, Prerequisites, Setup, Try It, Danger Zone, Troubleshooting
└── setup.sh      # Interactive setup script following standard patterns
```

**Setup script requirements:**
- Standard color variables and banner format
- Check for Claude Code before proceeding
- Scope selection prompt (user vs. project)
- `claude mcp remove <name> 2>/dev/null || true` before adding
- End with example commands the user can try

**README requirements:**
- **Danger Zone section** — document every write/send/delete action, who it affects, what can't be undone
- Attribution line: `*Contributed by Your Name*`
- Add your integration to the table in `.marvin/integrations/README.md`

---

## Extending MARVIN

### Adding custom skills

Skills live in your workspace's `skills/` directory. Add a new directory with a `SKILL.md` and MARVIN picks it up on next session. No restart needed.

If your skill is general-purpose and others might benefit, consider contributing it back to the template (see [Contributing](#contributing)).

### Adding custom slash commands

Slash commands are markdown files in `.claude/commands/`. Each file's content is sent to Claude when the command is invoked. Reference your skill or write inline instructions directly.

Example `.claude/commands/standup.md`:
```markdown
Generate my daily standup update. Read today's session log and state/current.md,
then produce a 3-bullet standup in this format:
- Yesterday: ...
- Today: ...
- Blockers: ...
Keep it under 100 words.
```

### Modifying CLAUDE.md

`CLAUDE.md` in your workspace is the primary context file Claude reads at the start of every session. It contains your profile, how MARVIN works, safety guidelines, and available integrations. Edit it freely — it's yours.

If you want to change MARVIN's default personality, tone, or safety behavior, edit the relevant sections in your workspace's `CLAUDE.md`.

### Syncing updates

When new skills or commands are added to the template:

```
/sync
```

This reads `.marvin-source` to find the template, then copies new files into your workspace. Existing files in your workspace are never overwritten — your customizations are safe.

---

## Migrating from an Older Version

If you used MARVIN before the template/workspace split was introduced:

**1. Get the latest template:**
```bash
git clone https://github.com/SterlingChin/marvin-template.git marvin-template
# or: git pull in your existing clone
```

**2. Run the migration script:**
```bash
cd marvin-template
./.marvin/migrate.sh
```

**3. Follow the prompts.** The script asks where your old MARVIN folder is and where to put the new workspace. It copies:
- Your profile (`CLAUDE.md`)
- Goals and priorities (`state/`)
- Session logs (`sessions/`)
- Reports, content, and custom skills

**4. Verify, then delete the old folder** once you confirm everything is intact.

---

## Troubleshooting & FAQ

**Q: MARVIN doesn't remember anything between sessions.**
A: Make sure you're running `/end` to save each session. Also verify you're opening the workspace directory (`~/marvin`), not the template. The `state/` and `sessions/` directories in the workspace are where memory lives.

**Q: `/start` gives an empty or generic briefing.**
A: `state/current.md` probably still has placeholder content from before onboarding completed. Run onboarding again ("Help me set up MARVIN") or manually edit `state/current.md` with your actual priorities.

**Q: An integration isn't showing up after I added it.**
A: MCP integrations require a Claude Code restart to appear. Exit and reopen, then type `/mcp` to verify it's listed. If it's not listed, re-run the `claude mcp add` command.

**Q: `/mcp` authentication opened a browser window but nothing happened.**
A: Close other browser tabs from previous auth attempts. Complete the login in the new window and wait for the "success" message before returning to Claude Code. If the browser window closed early, run `/mcp` again and re-authenticate.

**Q: Google Workspace isn't authenticating.**
A: Verify that the Gmail, Calendar, and Drive APIs are enabled in your Google Cloud project, and that your OAuth credentials are Desktop app type (not Web app). The `GOOGLE_OAUTH_CLIENT_ID` and `GOOGLE_OAUTH_CLIENT_SECRET` env vars must be set in the `claude mcp add` command, not in `.env`.

**Q: The `marvin` shell command isn't working.**
A: The function was added to your `.zshrc` (or `.bashrc`). You need to either open a new terminal or run `source ~/.zshrc` for it to take effect.

**Q: `/sync` didn't pull the new skill I expected.**
A: Check that `.marvin-source` in your workspace points to the correct template path. If you moved the template directory, update this file. Also verify the new skill exists in the template's `skills/` directory.

**Q: How do I add a goal or change my priorities?**
A: Just tell MARVIN directly: "Add a new work goal: ship the API refactor by end of Q2." MARVIN will update `state/goals.md`. For priorities, say "Update my top priorities" and MARVIN will edit `state/current.md`.

**Q: Can I use MARVIN with multiple Anthropic accounts or workspaces?**
A: Yes — each workspace is independent. Create separate workspace directories with separate `CLAUDE.md` files, each pointing to the same (or different) template via `.marvin-source`.

---

## Contributing

Contributions welcome — especially new integrations and skills.

**For integrations:**
1. Read `.marvin/integrations/CLAUDE.md` for required patterns
2. Create your integration directory in `.marvin/integrations/`
3. Follow the README and setup script requirements (including the Danger Zone section)
4. Submit a PR with the integration added to `.marvin/integrations/README.md`

**For skills:**
1. Build and test the skill in your own workspace first
2. If it's general-purpose and not user-specific, add it to the template's `skills/`
3. Submit a PR with a description of when and why to use it

**For bugs and docs:** Open an issue or PR directly.

---

## About

MARVIN is named after Marvin the Paranoid Android from *The Hitchhiker's Guide to the Galaxy* — a vastly intelligent entity with a brain the size of a planet, perpetually underutilized. The irony is intentional.

Created by [Sterling Chin](https://sterlingchin.com). Because everyone deserves a chief of staff.
