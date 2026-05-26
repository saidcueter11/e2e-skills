# e2e-skills

> Claude Code skills for writing and auditing end-to-end tests — framework-agnostic, real backend, no mocks.

Two skills that encode the full E2E testing workflow as slash commands you can invoke directly from Claude Code. Works with Playwright, Cypress, Maestro, and Detox.

---

## Skills

### `/add-e2e-tests` — Write tests for a feature

Tell Claude what feature to cover and it writes the complete test set: happy path, error paths, cleanup, BDD header. Tests hit the real backend — no route interception on happy paths.

**Example prompts:**
```
/add-e2e-tests Goal creation wizard — mentor completes 7 steps and goal appears in list
/add-e2e-tests Org member invite flow — admin invites a user, user accepts, appears in members list
/add-e2e-tests tackle the P1 gaps in .test-plan.md
```

**What it produces:**
- Full CRUD test set for the feature (create, edit, delete, error path)
- `beforeEach` API seed + `afterEach` cleanup
- `waitForResponse` listener-before-action pattern (no race conditions)
- `RUN_ID`-prefixed seed values (safe on shared staging environments)
- BDD header with Feature, Goal, Roles, Surfaces, API contracts, Gating, Risks, Edge cases
- Updates `.test-plan.md` after writing

---

### `/e2e-audit` — Audit what you already have

Reads every spec file and gives you an honest quality report. Flags real issues before they become CI debt.

**Example prompts:**
```
/e2e-audit
/e2e-audit src/features/payments
```

**What it produces:**
- Spec inventory table with quality tiers (T1 real journey / T2 partial / T3 smoke)
- Per-spec quality matrix: Happy+persistence · Error paths · Mobile · Roles · Concurrency · Gating
- Anti-pattern report with file:line (waitForTimeout, .first() on action buttons, no API verification, etc.)
- `.test-plan.md` — living document with gaps digest + prioritized "what to write next" list
- CI workflow + ESLint rule scaffolded if missing

---

## Setup (one-time per machine)

```bash
# 1. Clone this repo
git clone git@github.com:saidcueter11/e2e-skills.git ~/e2e-skills

# 2. Symlink into Claude Code's skills directory
mkdir -p ~/.claude/skills
ln -s ~/e2e-skills/add-e2e-tests ~/.claude/skills/add-e2e-tests
ln -s ~/e2e-skills/e2e-audit ~/.claude/skills/e2e-audit
```

Restart Claude Code once. The skills are now available in every project.

### Verify it worked

Open Claude Code in any project and type `/add-e2e-tests` — it should autocomplete. Or run:

```bash
ls -la ~/.claude/skills/
# Should show symlinks pointing to ~/claude-skills/
```

---

## Updating

When a skill is updated, one command syncs everyone:

```bash
cd ~/e2e-skills && git pull
```

The symlinks propagate changes immediately — no reinstall, no restart.

---

## Requirements

- [Claude Code](https://claude.ai/code) installed
- A test environment (local Supabase / Docker / staging) — **not production**
- Test credentials in `.env.test` or `.env.test.local`

The skills will tell you what's missing if your project isn't set up yet (`/e2e-audit` runs a readiness check first).

---

## Adding a new skill

1. `mkdir <skill-name> && touch <skill-name>/SKILL.md`
2. Add YAML frontmatter with `name` and `description` (see existing skills for format)
3. Open a PR — keep skills framework-agnostic and project-neutral
4. Team pulls to get it
