# claude-skills

Shared Claude Code skills for the team. Framework-agnostic — works with any project.

## What's here

| Skill | Command | Purpose |
|---|---|---|
| `add-e2e-tests` | `/add-e2e-tests` | Write real E2E tests (Playwright, Cypress, Maestro, Detox) against a real backend — no mocks |
| `e2e-audit` | `/e2e-audit` | Audit E2E coverage, tier-categorize tests, flag anti-patterns, scaffold missing infra, update `.test-plan.md` |

## Setup (one-time per machine)

```bash
# 1. Clone this repo
git clone git@github.com:saidcueter11/claude-skills.git ~/claude-skills

# 2. Symlink each skill into Claude Code's user skills directory
mkdir -p ~/.claude/skills
ln -s ~/claude-skills/add-e2e-tests ~/.claude/skills/add-e2e-tests
ln -s ~/claude-skills/e2e-audit ~/.claude/skills/e2e-audit
```

That's it. The skills are now available in any project you open with Claude Code.

## Updating

When skills are updated, just pull:

```bash
cd ~/claude-skills && git pull
```

Changes propagate immediately to all projects via the symlinks — no reinstall needed.

## Adding a new skill

1. Create a folder: `mkdir <skill-name>`
2. Add a `SKILL.md` with frontmatter (`name`, `description`) — see existing skills for format
3. PR + review
4. Team pulls to get it
