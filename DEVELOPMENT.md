# MARVIN — Development Guide

This guide is for contributors working on the MARVIN template itself — not for MARVIN users. If you're setting up MARVIN as your AI chief of staff, see [README.md](README.md).

---

## Prerequisites

- **Node 18+** and `npm`
- **Claude Code** — `npm install -g @anthropic-ai/claude-code`
- **Git**
- A shell (`zsh` or `bash`)

---

## Local Setup

```bash
git clone https://github.com/SterlingChin/marvin-template.git
cd marvin-template
```

No build step. The template is plain markdown, shell scripts, and YAML frontmatter — nothing to compile or install.

To test your changes against a live MARVIN session:

```bash
claude
```

Claude Code reads `CLAUDE.md` and picks up all commands and skills in the directory immediately. No restart needed for content changes.

---

## Repository Layout

```
marvin-template/
├── .claude/commands/       # Slash command definitions (one .md per command)
├── .marvin/                # Internal: onboarding, migration, integration configs
│   └── integrations/       # One subdirectory per integration
├── skills/                 # Skill definitions (one subdirectory per skill)
│   └── _template/          # Starting point for new skills
├── state/                  # Placeholder state files for onboarding
├── CLAUDE.md               # Primary context file Claude reads on every session
├── README.md               # User-facing documentation
└── DEVELOPMENT.md          # This file
```

---

## Branching

| Branch | Purpose |
|--------|---------|
| `main` | Stable, always releasable |
| `feat/<name>` | New skills, integrations, or features |
| `fix/<name>` | Bug fixes |
| `docs/<name>` | Documentation-only changes |

Keep branches short-lived. One logical change per PR.

```bash
git checkout -b feat/my-new-skill
```

---

## Making Changes

### Skills

1. Copy `skills/_template/` to `skills/your-skill-name/`
2. Edit `SKILL.md` — fill in frontmatter and write clear step-by-step instructions
3. If the skill needs a slash command, add `.claude/commands/your-skill-name.md`
4. Test by opening Claude Code in the template directory and invoking the command

### Integrations

1. Read `.marvin/integrations/CLAUDE.md` — required patterns are documented there
2. Create `.marvin/integrations/your-integration/`
3. Add `README.md` (must include a **Danger Zone** section) and `setup.sh`
4. Add your integration to the table in `.marvin/integrations/README.md`
5. Test the setup script end-to-end on a clean machine or fresh Claude Code profile

### Slash Commands

Commands live in `.claude/commands/`. Each file is a markdown document — its content is sent to Claude verbatim when the command is invoked. Keep them focused: one command, one responsibility.

### CLAUDE.md

Changes to `CLAUDE.md` affect every MARVIN session. Be conservative — this file ships to users and shapes Claude's default behavior. Test any personality, safety, or workflow changes thoroughly before submitting.

---

## Testing Changes

MARVIN has no automated test suite. Testing is manual and session-based.

**Checklist before opening a PR:**

- [ ] Open Claude Code in the template directory (`claude` from `marvin-template/`)
- [ ] Run `/start` — does it produce a sensible briefing?
- [ ] Invoke your new skill or command — does it behave as documented?
- [ ] Run `/end` — does it save correctly?
- [ ] If you changed a setup script: run it from scratch and verify all steps complete without errors
- [ ] If you changed `CLAUDE.md`: open a fresh session and confirm Claude's behavior matches intent
- [ ] Check that no placeholder content leaks into output

---

## Commit Style

Use short, imperative subject lines. One logical change per commit.

```
Add gym-tracker skill with weekly summary
Fix MS365 setup script scope prompt
Update contributing docs with integration Danger Zone requirement
```

No ticket numbers, no emoji, no "fix: " prefixes unless the project adopts conventional commits explicitly.

---

## Submitting a PR

1. Push your branch and open a PR against `main`
2. Title: short imperative description of what changed
3. Body: what the change does, how to test it, any edge cases
4. For integrations: confirm the **Danger Zone** section is present and accurate
5. For skills: confirm the skill works end-to-end in a live session

PRs are merged by maintainers after review. Small, focused PRs merge faster.

---

## Project Conventions

- **Markdown everywhere** — skills, commands, and docs are all `.md`
- **No build tooling** — keep the template dependency-free
- **Shell scripts are POSIX-compatible** — avoid bash-isms in `setup.sh` files where possible
- **Danger Zone is non-negotiable** — every integration README must document destructive actions
- **Attribution** — add `*Contributed by Your Name*` to integration READMEs

---

## Questions

Open an issue or reach out to [Sterling Chin](https://sterlingchin.com).
