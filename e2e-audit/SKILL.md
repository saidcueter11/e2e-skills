---
name: e2e-audit
description: Audit E2E test coverage (web + mobile), tier-categorize tests, flag anti-patterns, scaffold missing test infra, write/update .test-plan.md, land CI workflow + ESLint rule via PR, and verify the first CI run.
argument-hint: [optional repo path — defaults to current working directory]
---

# e2e-audit

Audit E2E test coverage across web and mobile. Categorize every test by real
user-flow quality (not just smoke vs. functional). Flag anti-patterns. Scaffold
infrastructure when missing. Write or update `.test-plan.md` as the living
shared artifact. After approval, land the CI workflow + ESLint rule via PR and
verify the first run.

Trigger when: "audit e2e tests", "e2e test coverage", "what tests do we need",
"e2e audit", "review e2e tests".

## Steps

Follow these steps in order. Do NOT skip any step.

---

### Step 0: Project readiness check (informational — does NOT stop the command)

Detect what's already in place. The command will scaffold whatever's missing
in Step 7 — this step just records the starting state for the report.

1. **Test environment URL** — `.env.test`, `.env.local`, `.env.staging`, or
   similar with a non-production API URL. Or `playwright.config.ts` `baseURL`
   pointing to localhost / staging / test domain.
2. **Test credentials** — dedicated test user accounts via env vars or seed
   scripts.
3. **Seed/reset capability** — API helpers, seed scripts, or auth flows for
   creating ephemeral test data.
4. **Production safety** — `baseURL` is NOT a production domain. No
   production API keys in test env files. **If you find production
   credentials in a test config, STOP IMMEDIATELY and surface as a critical
   issue** — that's a security finding, not a setup gap.
5. **Backend / DB migration sync** — if the project has DB migrations
   (anywhere — `db/migrations/`, `prisma/migrations/`, `supabase/migrations/`,
   `alembic/`, `flyway/`, separate backend repo, monorepo package, etc.),
   check whether the local/test DB is current with the latest migration. Out-of-sync
   migrations produce cryptic 500s (functions not found, missing columns) that
   look like app bugs but are really env gaps. In the readiness report, include
   the detected migration tool and the command to apply pending ones
   (e.g. `prisma migrate deploy`, `alembic upgrade head`,
   `npx supabase migration up --local --include-all`, `flyway migrate`).
   For Supabase projects, also check the CLI version (`supabase --version`) —
   report it so the team can align; upgrades are generally safe but worth noting.
6. **Feature-gate / entitlement seeding** — if the app gates features behind
   a subscription tier, plan, role, feature flag, or entitlement, the test
   accounts must have whatever level grants access to the gated feature. The
   gate may live in any of: a subscriptions/plans table, an `entitlements`
   table, a feature-flag service (LaunchDarkly, Unleash, Statsig), an `is_pro`
   column, custom claims on the auth token, etc. Detect the mechanism
   (grep source for feature-gate components, permission services, or flag
   reads) and list the required level per test role in the readiness report.
   Features behind unmet gates produce 403s that skip tests rather than fail
   them — silent under-coverage.
7. **All test users have completed onboarding / verification** — apps with
   multi-step onboarding (payment, profile completion, email verification,
   ToS acceptance, MFA enrollment) redirect users mid-test if any step is
   incomplete. Detect the gates (look for redirects to `/onboarding`,
   `/verify-email`, `/complete-profile` in route guards or middleware) and
   list the columns/flags that must be `true` for every test account after
   each DB reset (common names: `email_verified`, `payment_completed`,
   `onboarding_completed`, `terms_accepted`, `mfa_enrolled`).

8. **Prescribed E2E doc set** — check each, mark missing for Step 7 to scaffold:
   - `tests/e2e/README.md` — onboarding (how to run, env vars, conventions,
     sibling-repo declaration if applicable, project-specific gotchas).
   - `tests/e2e/.test-plan.md` — sections in order: Quality bar (with a
     readiness sub-table marking each DoD item ✅/❌), Spec inventory,
     Edge-case gaps digest (grouped by risk category, with an explicit
     "Cross-repo integration" subsection when sibling repos exist), What
     to write next (two tables: Blocked with Owner + Next action, Open
     with Complexity + Owner), Anti-patterns watch (count + fix status),
     Active blockers (cross-cutting, with Owner).
   - Per-spec BDD headers — every `*.spec.ts` MUST begin with a JSDoc block
     containing `Feature:`, `Goal:`, `Roles:`, `Surfaces:`, `API contracts:`,
     `Risks:`, `Edge cases covered:`, `Edge cases NOT covered:`. Regex check:
     does the file start with `/**` and does that block contain `Feature:`?

   **No longer prescribed:** `tests/e2e/CHANGELOG.md` (history is `git log
   tests/e2e/`) and `tests/e2e/.coverage-matrix.md` (duplicates the spec
   inventory + digest, ages quickly). If a project still has them, flag
   them for removal — their content is absorbed into `.test-plan.md` and
   the git log.
9. **Sibling-repo discovery** — heuristically detect whether the product spans
   multiple repos (the E2E suite for this repo can't cover capabilities owned
   by sibling repos, and the team should know where coverage lives). Signals:
   - `package.json` dependencies on sibling packages (`@<org>/<pkg>`).
   - `.env.test*` hosts pointing to non-self origins (TEST_API_URL,
     LMS_URL, AUTH_URL, etc.) that aren't localhost-of-this-repo.
   - `CLAUDE.md` / `README.md` mentions of companion repos.
   - Adjacent directories under the same git remote owner.
   Propose a list and prompt the user to confirm/edit. The final list is
   written to `tests/e2e/README.md` under `## Sibling repos` so subsequent
   audit runs read it as source-of-truth.

10. **CI setup verification.** Don't assume CI is wired correctly because
    the suite "exists." Audit:
    - **Workflow file present** — `.github/workflows/*.yml` (or
      `.gitlab-ci.yml`, `bitbucket-pipelines.yml`, CircleCI config) with a
      job that runs the e2e command. Report path.
    - **Required secrets configured** — list every env var the suite reads
      (grep `process.env.` in `tests/e2e/`) and cross-check against the
      workflow's `env:` block. Anything referenced by tests but absent
      from the workflow will surface as a runtime crash, not a clear
      "missing secret" error. Report the gap list — but **do not block
      the audit on it**; missing secrets are filled by the dev in GitHub
      Actions UI after PR-land.
    - **Triggers** — the workflow runs on PR (`pull_request:`) AND on push
      to the default branch (`push: branches: [main]`). PR-only means
      regressions on `main` are invisible; push-only means PRs can land
      red.
    - **Required check on default branch** — branch-protection rules
      mark the e2e check as required for merge. The workflow existing is
      not enough; without protection, anyone can merge red. Report
      ✅/❌ + a one-line manual step the dev needs to take in the repo
      settings UI if missing.
    - **Sharding / parallelism** — if the suite is >50 tests or >10min,
      the workflow uses a `matrix:` strategy with shards. Report shard
      count and total estimated runtime; flag if a single non-sharded
      job would exceed ~30min.
    - **Artifact upload on failure** — `playwright-report/` /
      `test-results/` uploaded as a workflow artifact on failure
      (`if: always()` or `if: failure()`). Without this, CI failures are
      undiagnosable from chat — the dev has to re-run locally to see what
      happened. Report ✅/❌ + the upload step path.
    - **Cross-PR parallelism — per-slot tenant pool.** If the workflow
      runs on PR (not just on push to main), two PRs can be in flight
      simultaneously. Without a slot system they share the same test
      accounts and step on each other's mutations. Check for: a `Compute
      slot index` step that derives `SLOT = PR_NUMBER % N`, a
      `seed-e2e-pool.sql` (or equivalent) that materialises N copies of
      every account namespaced by slot, and a TS helper
      (`slot-spec.ts` / similar) that returns slot-aware credentials.
      Report ✅ present / 🟡 single-account-with-cancel-in-progress
      (works only because CI cancels concurrent runs of the SAME PR — does
      NOT cover two DIFFERENT PRs hashing close in time) / ❌ missing.
      Also flag any **irreversible-state accounts** (onboarding,
      MFA-enrolled, plan-upgraded) that are NOT in the slot pool — those
      will collide cross-PR the moment their test runs, even though
      read-only accounts may seem fine.

Record per-check: ✅ present / ❌ missing / ⚠️ unsafe (prod-pointing).

Then continue. Don't generate a guide. Don't stop unless safety check #4
fails. Step 7 scaffolds whatever's missing; Step 10 lands the CI gate. The
dev fills in real values (test creds, API tokens, secrets) in
`.env.test` (gitignored) and GitHub Actions secrets after the PR lands —
the report tells them exactly what.

---

### Step 1: Detect the framework(s) — web AND mobile

Do not assume Playwright. Detect first.

**Web (Playwright / Cypress / WebdriverIO):**

- `*.spec.ts`, `*.spec.tsx`, `*.spec.js`, `*.test.ts`, `*.test.tsx` files
- Directories: `e2e/`, `tests/`, `__tests__/`, `playwright/`, `cypress/`
- `playwright.config.ts` / `playwright.config.js` / `cypress.config.ts`
- `package.json` deps: `@playwright/test`, `playwright`, `cypress`,
  `webdriverio`

**Mobile (Maestro / Detox):**

- `.maestro/` directory; `.maestro/flows/**/*.yaml`; `.maestro/config.yaml`
- `maestro.yaml` at repo root
- Detox: `e2e/**/*.e2e.js`, `.detoxrc.js`, `detox.config.js`
- `package.json` deps: `detox`; scripts invoking `maestro test`

**Report the detected framework(s) and stack** before listing test files. If
zero coverage is found and the readiness gate passed, proceed in
**Fresh Project Mode** (Step 7 will scaffold). Otherwise proceed in
**Audit Mode**.

Also discover project conventions in scope:

- `playwright.config.ts` (or equivalent) — base URL, projects, timeouts, test
  directory.
- Helpers (`helpers/api.ts`, `helpers/auth.ts`, seed utilities).
- Fixtures (`fixtures/base.ts`).
- Page Objects (`pages/*.page.ts`).
- Verify `baseURL` is NOT a production URL — flag immediately if it is.

---

### Step 2: Decide mode

A spec file is **real** if it contains at least one test that performs a user
action and asserts on a real outcome (not just visibility of a static
element).

- **0 real specs** → **Fresh Project Mode**. Skip Steps 3–4, jump to Step 5.
- **Real specs exist** → **Audit Mode**. Continue through every step.

Announce which mode you are in.

---

### Step 3: Inventory and tier-categorize every test (Audit Mode)

Read every spec file (or Maestro YAML flow). For each `test(`, `it(`, or flow
file, record:

- **File** and **test name**
- **User goal** in one sentence (if you can't write one, flag it)
- **Hits real API?** yes/no — does it make actual network requests to the
  test backend?
- **Quality tier**:

**Tier 1 — Real journey test.** Has ALL of:
- User performs a multi-step action against the real UI.
- Actions trigger real API calls to the test backend (no route interception
  on happy paths).
- Asserts on outcome visible to the user (item appears/disappears, data
  changes).
- For mutations: optionally verifies via a follow-up API call that backend
  state changed.
- Test seeds its own data and cleans up.

**Tier 2 — Partial journey.** Covers a real intent but is missing one or
more of:
- Hits real API but no cleanup (pollutes test env).
- No backend-state verification after mutation.
- Missing error-path coverage.
- Depends on pre-existing data instead of seeding.

**Tier 3 — Fake / low-value.** Any of:
- Uses `page.route()` to mock happy-path API responses.
- Only asserts `toBeVisible()` / `toHaveText()` on static elements.
- Would pass with the backend completely down.
- Tests that a component renders without any user interaction.
- Standalone "dialog opens / closes" without the full flow.
- Uses hardcoded fake-data objects instead of API-seeded data.

Print a summary table:
`File · Tests · T1/T2/T3 · H · E · M · R · C · G · CI · Notes`

**`CI` ≠ quality.** `CI` is "does this suite currently pass on main?" — a green-build signal. A spec at all-❌ on the 6-dimension matrix can still be `CI ✅` because the few tests it has pass. The 6 dims are the quality signal; CI is only the green-build signal. Always include both, and put the column name as `CI` (not `Status`) so the distinction is unambiguous.

The six quality columns:

- **H** (Happy + persistence) — ✅ if UI-driven happy path against real API **AND** a hard `page.reload()` re-asserts the post-state after **every** mutation in the spec; 🟡 if happy path exists but ANY mutation lacks a reload (proves cache, not DB — partial coverage still earns 🟡, never ✅); ❌ if mocked or no UI-driven mutation at all. *Exception*: if a mutation is verified by a forced redirect that hydrates from the server (e.g. password change → forced `/login` → re-login proves the change), document the substitute in the BDD `Risks:` line and count as reloaded.
- **E** (Error) — Sub-classes: 422 validation, 403 permission, 409 conflict, 500 infra, 401 auth-expired. Each sub-class can be **real-server** (asserted by hitting the real backend with bad input or insufficient role) or **mocked** (forced via `page.route()` to inject 500/401). Real-server sub-classes are stronger signals — count each real-server sub-class as 1.0 and each mocked sub-class as 0.5 toward the threshold. ✅ if total ≥3 with at least 1 real-server; 🟡 if total ≥2 OR if only client-side validation; ❌ if total ≤1.
- **M** (Mobile) — Mobile-emulated viewport (e.g. 390×844) covers 1 mutation + 1 error path + 1 read. ✅ all three; 🟡 one or two; ❌ none.
- **R** (Roles) — UI+ (allowed role works) AND UI− (nearest no-access role is blocked, asserted via a `assertSurfaceHiddenFor`-style helper) AND API+ AND cross-tenant (different tenant/owner cannot see the data). ✅ all four; 🟡 UI+/API+ only; ❌ UI+ only. **N/A handling**: if a sub-item is genuinely inapplicable (e.g. auth flows are per-user not per-tenant → cross-tenant is N/A), count it as ✅ **only if** the BDD `Risks:` or `Edge cases NOT covered:` line documents the N/A with a one-line reason. Undocumented N/A = ❌ for that sub-item.
- **C** (Concurrency/resilience) — Double-click on a confirm button does not duplicate AND a session-expiry mid-flow recovers (re-login or graceful error). ✅ both; 🟡 one; ❌ neither.
- **G** (Gating) — Per the project's tier-matrix / gating artifact: if gated, the spec covers both above- and below-gate accounts; if ungated, the BDD header explicitly states `Gating: N/A — no gating (verified <artifact> @ <SHA>)`. ✅ both gate sides covered or explicit N/A line; 🟡 one side only; ❌ no assertion AND no N/A line. If the project has no gating artifact, mark the column as `?` and recommend the audit step that scaffolds one.

**N/A convention applies to every dim with sub-items (R, G, and any future multi-part dim).** A dim where a sub-item is documented N/A in the BDD header is not penalized — but the documentation MUST exist in the spec file itself (not in the audit report), so the next reader sees the rationale.

**Accessibility is intentionally not a quality column.** axe scans surface real product debt (sidebar contrast, form labels) but produce signal that's orthogonal to behavior regressions — they noise up the dashboard without catching the bugs e2e is designed for. If a project explicitly opts in, document it in that project's `.test-plan.md` quality bar with a narrow scope.

Bottom row: per-dimension percentages (H%, E%, M%, R%, C%, G%). **Do NOT lead with a synthetic weighted average** — weighting across dimensions with very different cost/value is misleading, and `min()` is unduly punitive while `avg` is unduly forgiving. Lead with two honest numbers: **(a) all-6-✅ count** (the per-spec definition of done) and **(b) per-dimension readings**. The all-6 count starts very low on any first audit; the per-dim reading shows which dims are strong (don't slip) vs weak (close first).

---

### Step 4: Flag anti-patterns explicitly (Audit Mode)

Grep the suite for each. List every occurrence with **file:line**:

1. **Admin-seeded "functional" tests** — tests that perform CRUD through a
   service-role / admin client (`supabase-admin`, `createClient` with service
   role key, elevated API POST) instead of driving the rendered UI. They
   masquerade as Tier 1 but are really Tier 2 dressed up. Report as
   "admin-seeded" and downgrade their tier in the table.
2. `page.waitForTimeout(...)` — flake factory.
3. `waitForLoadState('networkidle')` — flake factory.
4. **React auto-generated IDs in selectors** — any selector matching `:r\d+:`
   (e.g., `:r4:`). Unstable across renders.
5. `test.skip(!someDataExists, ...)` — silent skips when pre-existing data is
   missing instead of seeding. **Narrow the grep**: only flag conditions
   referencing data the spec could reasonably seed itself (regex hints:
   `!.*(data|exists|Loaded|Created|Seeded|Available)$` and lowercase camelCase
   names like `!commerceAvailable`, `!ruleData`). Do NOT flag credential or
   env-gate skips (`!TEST_*_EMAIL`, `!process.env.SUPABASE_SERVICE_ROLE_KEY`,
   `!CROSS_TENANT_AVAILABLE`, `!automationsAvailable`) — those are legitimate
   "this env can't run this test" gates and a feature, not a bug. In practice,
   on any mature suite, raw `test.skip` count will be 50+ and 90%+ legitimate
   — print BOTH the raw count AND the narrowed-grep count so the reader sees
   the signal-to-noise ratio.
6. `toBeVisible()` as the **only** assertion after a mutation — too weak to
   catch regressions.
7. Hard-coded prod URLs or credentials in specs.
8. `page.route()` on happy-path APIs — test passes with backend down.
   **Context-aware**: a `page.route()` call inside a test whose name matches
   `/error|500|401|fail|throttle|race/i` is almost always legitimate
   error-injection or concurrency throttling, not a happy-path mock. Print
   intercepted URLs alongside the count so the reader can audit; do NOT
   flag matches inside E-dim or C-dim tests by default.
9. CSS class selectors — brittle, breaks on styling changes.
10. **`.first()` / `.last()` on per-item action selectors.** Grep
    specifically for `(Edit|Delete|Save|Update|Remove|Archive|Restore)Btn.*\.(first|last)\(\)`
    OR `getByRole\(['"]button['"][^)]*name[^)]*(Save|Delete|Edit|Update|Remove)[^)]*\)\.(first|last)\(\)`.
    The narrow regex is critical: a raw grep for `.first()` on a mature suite
    will return 100+ hits, most of which are legitimate (toast scoping
    `.getByText("Saved").first()`, intentional first-row selection, list
    sentinels). Only the per-item ACTION button case is dangerous — that's
    the one that silently mutates the wrong row. Page Objects should expose
    identifier-scoped methods (`deleteBtnFor(title)`); raw `.first()` on a
    delete button is the smell. **Has caused multi-hour debugging sessions
    in real projects — flag as a high-priority upgrade.**
11. **UI-only mutation tests with no API verification.** A click followed
    only by `toBeVisible()` / `not.toBeVisible()` proves only that the UI
    rendered something. Optimistic updates can roll back silently;
    invalidations can hide rather than delete. Every mutation test must
    end with a follow-up API call that asserts on backend state. Grep for
    `await api.` / `request.newContext` after mutation clicks — if absent,
    flag.
12. **Missing dev-server warm-up in `beforeAll`.** For Next.js / SSR /
    on-demand-compilation projects against a dev server, the first request
    to each route triggers a 10–30s compile. Without a `fetch()` warm-up
    in `beforeAll`, the first browser test of each suite hits ERR_ABORTED
    and dies. Check every `*.spec.ts` `beforeAll` for `fetch(...)` against
    the routes the suite navigates to. Flag suites missing this.
13. **Missing browser dialog handlers** when the SUT calls `alert()` /
    `confirm()` on error paths. If the component shows a JS alert and the
    test doesn't register `page.on('dialog', ...)`, Playwright
    auto-dismisses, the error message is lost, and failures look like
    silent hangs. Cross-reference `alert(` / `confirm(` in source against
    `page.on('dialog'` in specs.
14. **Page Objects exposing raw locators for list items.** A Page Object
    with `get deleteBtn()` / `get editBtn()` (no parameter) forces every
    caller to scope downstream — and they will get it wrong. Grep page
    object files for `get \w+Btn\(\)` / `readonly \w+Btn:` paired with
    list rendering. Flag for refactor: methods should take an identifier
    (id, title, index) and return a scoped locator.
15. **Long suites at risk of token expiry.** Sum estimated runtime per
    suite (or read recent CI durations). For Supabase / JWT-based auth,
    flag any suite whose total runtime exceeds ~45 minutes — the API
    client's token will expire mid-run, producing 401s on the last few
    tests. Recommend re-fetching tokens per-test or splitting the suite.
16. **`Promise.race` with `waitFor({ state: 'detached' })` as a readiness
    signal.** `waitFor({ state: 'detached' })` resolves *instantly* when
    the element never enters the DOM — e.g., when an auth gate redirects
    to `/login` before the loading skeleton renders. In a `Promise.race`,
    this false-positive wins and the test proceeds to a page it isn't
    actually on. Grep auth/navigation helpers for this pattern. Readiness
    signals must be HTTP-level (specific response status, URL change),
    never UI artifacts that may not appear. **False-positive filter**: only
    flag a `Promise.race` whose array contains `waitFor({ state: 'detached' })`
    AND has NO HTTP-level co-condition (`waitForResponse`, `waitForURL`,
    explicit status assertion) in the same race. A `Promise.race` that
    pairs the detached-wait with `waitForURL` or `waitForResponse(<status>)`
    is well-formed — the HTTP signal is the real readiness, the detached
    wait is the fast-path shortcut for already-loaded pages.
17. **CDN / edge-cached reads on a write surface.** When a feature has a
    "save then reload and verify" UX, check the read path for CDN clients
    (`useCdn: true`, ISR, edge cache, materialised views with replication
    lag, read replicas with stale projections). Writes go direct, reads
    come from cache → reload after save shows stale data → "persistence"
    tests fail intermittently or always. This presents as a flaky test
    but is a real product bug. Grep the read path for `useCdn`, `revalidate`,
    `cache:`, `next: { revalidate }`, `Cache-Control`, replication-lag
    waits, etc. If found on a read-after-write path, flag as a product
    issue, not a test issue.
18. **Multi-service local stacks not documented.** If the app calls a
    second service (separate microservice, LMS, payments, auth provider,
    BFF) that's normally a deployed dependency, local e2e runs need that
    service started locally — and the test env file needs the override
    URL pointing to it. Detect by grepping the source for `NEXT_PUBLIC_*_URL`,
    `process.env.*_BASE_URL`, fetch calls to non-localhost hosts, etc.
    For each, document in `.test-plan.md`'s "Running locally" section:
    repo path, port, start command, env override. Without this, the
    suite passes against staging but fails inscrutably for any new dev.
19. **Storage-state token rotation hazards (shared Supabase RTs across
    workers/retries).** When `storageState` is built once in setup and
    reused across parallel workers, the first worker that calls
    `/auth/v1/token?grant_type=refresh_token` rotates the RT and
    invalidates the saved one. Subsequent workers (or test retries) hit
    `/auth/me` 401 → refresh 400 → `/login?session_expired=true` and
    fail with cryptic "element not found" timeouts. Flag suites that:
    (a) reuse a single storageState across all workers, (b) lack a
    `gotoAuthed`-style helper that detects 401 and re-logs, (c) have no
    per-worker storageState isolation.
20. **Missing BDD header on a spec file.** Every `*.spec.ts` MUST begin
    with a JSDoc block containing the slots `Feature:`, `Goal:`, `Roles:`,
    `Surfaces:`, `API contracts:`, `Risks:`, `Edge cases covered:`,
    `Edge cases NOT covered:` (see `add-e2e-tests` skill, Step 5). Without
    it, the next maintainer can't know the spec's intent or feed the
    project's gaps digest. Grep: files whose first non-blank line is not
    `/**`, or whose top JSDoc block is missing `Feature:`. Flag for
    backfill — header content can usually be inferred from the test names
    and assertions inside the spec.
21. **Edge-case gaps not mirrored in `.test-plan.md`.** Every spec's BDD
    header has an `Edge cases NOT covered:` line. The set of those lines
    across all specs must appear in `.test-plan.md` § Edge-case gaps
    digest. If a spec lists a gap that's missing from the digest (or vice
    versa), the docs are drifting. Flag the diff so the next contributor
    can reconcile.
22. **Universal-gotcha content in the project README.** The project's
    `tests/e2e/README.md` § Project-specific gotchas should contain ONLY
    project-specific items (component quirks, in-house API conventions).
    Generic patterns (optimistic-update race, toast/live-region duplicates,
    accordion toggles, auth-aware navigation, token rotation, nested API
    shapes) belong in the `add-e2e-tests` skill, not in the project. Flag
    any universal pattern documented in the project and recommend moving
    it to the skill.

18. **Literal seed names searched in strict-mode locators.** A test that
    seeds `first_name: "SearchAlpha"` and later calls
    `page.getByText("SearchAlpha")` will pass once, then fail forever
    after the cleanup `afterEach` is skipped (timeout, crash) and leaves
    a stale row in the test DB — the next run finds two matches and
    Playwright strict-mode throws. Every seeded value used in a UI
    assertion must include `RUN_ID + Date.now()` (or a UUID). Grep specs
    for `seedX({ ..., name: "Literal" })` patterns and flag.

23. **Missing hard-reload after happy-path mutations.** A mutation test
    that ends on UI/API assertions but never calls `await page.reload()`
    proves only that the cache/optimistic-update layer reflected the
    change — not that it persisted to the DB. Grep mutation tests for
    a `page.reload()` after the confirm/save. If absent, flag as a 🟡
    on the H dimension (cache covered, persistence not). This is a high-
    yield finding: optimistic-update bugs are common, ship-blocking, and
    invisible without reload.
24. **Missing G dimension assertion — no tier-matrix / gating artifact
    consultation.** If the project gates features behind plans/tiers/flags
    and there is no `.tier-matrix.md`-style artifact in `tests/e2e/`, the G
    dimension is unwritable. Each spec defaults to PM-interrogation, which
    doesn't scale. Flag the absence and recommend Step 7 scaffolds the
    artifact from the gating source-of-truth in code.
25. **`eslint-disable` comments inside `test.fixme(...)` bodies that
    re-enable the project's anti-flake rule.** Grep for
    `eslint-disable-next-line` followed within the same block by
    `waitForTimeout`, `networkidle`, or any rule the project's
    `eslint.config.*` actively forbids. Most appear inside a `test.fixme`
    where the author thought "the body won't run anyway." Three problems:
    (a) the comment normalises the anti-pattern visually for the next
    reader; (b) when the test is eventually un-fixmed, the rule is
    pre-silenced and CI can't catch the bad wait; (c) it advertises the
    project's lint rule as bypassable. Recommend replacing with
    `expect.poll`, `waitForResponse`, or `waitForFunction` — the future
    un-fixmed test should work without disabling the rule.
26. **`not.toBeVisible()` as the only assertion that "a row is gone" in a
    responsive / dual-render UI.** When the app renders the same logical
    row in two slots (desktop table + mobile card, master+detail panes,
    sidebar+main echoes), `not.toBeVisible()` checks only one slot. The
    OTHER may still match and the test passes intermittently. Grep for
    `not.toBeVisible()` calls scoped to a single Page Object getter that
    filters by content; cross-reference with the source rendering the row
    in two `<div>`s gated by Tailwind `hidden lg:flex` / `lg:hidden`. Flag
    for upgrade to `toHaveCount(0, { timeout: ... })`.
27. **Wizard / multi-step flow plumbing inlined across multiple tests.**
    When 3+ tests in the same spec inline the same `openModal → fill →
    next → fill → continue → wait for confirm step` sequence (instead of
    calling one Page Object helper), each callsite will eventually drift.
    One will forget a `waitFor`, miss an intermediate field, or stop at
    the wrong step — and the divergence stays invisible until a refactor
    touches one but not the others. Grep specs for repeated 5+ line
    sequences of modal interactions. Recommend extracting a single
    `fillXToConfirmStep` method on the Page Object that stops with the
    final confirm button visible, leaving callers to decide what to do
    at the final step (click for happy path; throttle for race; route-mock
    for error injection).
28. **Suite lacks a route-smoke spec.** When the app has a navigation menu
    with N advertised routes, the suite should include one spec that iterates
    every route reachable from the nav, asserts HTTP 200 (or expected
    redirect), and asserts the app shell selector is present (sidebar +
    top-bar, or layout-equivalent). Without it, a route that 404s with the
    raw framework error page (no app chrome) ships silently — the user
    lands on a dead end. Grep `tests/e2e/` for a spec iterating an array
    of routes against a layout selector; if absent, flag and recommend
    adding `route-smoke.spec.ts`.
29. **Fixtures lack a console-error guard.** The base test fixture should
    register `page.on('console', ...)` and fail the test if any
    `console.error` fires that isn't on a project-maintained whitelist
    (regex array). Without this, real bugs ship green: failed RSC payload
    fetches, 401s on first paint, missing env-var warnings, hydration
    mismatches. Grep the base fixture file for `page.on('console'`; if
    absent, flag — this is one of the highest-leverage additions a suite
    can make.
30. **Modals opened without title assertion.** Every `click CTA → modal
    opens` flow should assert the modal heading text **exactly** (e.g.
    `toHaveText('Start a new Conversation')`), not just that *some* modal
    appeared. Vocabulary drift between the CTA label ("Start new Chat")
    and the modal title ("Add member to Conversation") is invisible to
    `toBeVisible()` checks but very visible to users. Grep specs for
    `getByRole('dialog')` / `.modal` / `[role="dialog"]` followed by no
    `toHaveText` within ~10 lines; flag.
31. **No empty-state coverage.** Every CRUD spec seeds data before
    navigating, so the empty-state branch of each list/feed/dashboard
    never executes under test. Empty-state bugs (typos in copy, missing
    primary CTA, stale "loading" spinner, broken pagination on 0 results)
    ship undetected. Each spec should include at least one test that
    deliberately skips the seed and asserts the empty-state copy + the
    primary CTA label. Grep specs for any `test(/empty|empty-state/i, ...)`;
    flag specs with zero empty-state tests.

**Infrastructure checks** (report yes/no):

- Auth state reuse via `storageState` / equivalent? Or does every test re-log
  through the UI?
- `afterEach` / `afterAll` cleanup of created entities?
- ESLint rule forbidding `waitForTimeout` + `networkidle`? (Chorus has this —
  use it as reference.)
- Required CI check on `main` / `master` running e2e? Check
  `.github/workflows/`, `.gitlab-ci.yml`, `bitbucket-pipelines.yml`. Report
  filename if yes.
- **Gating source-of-truth artifact** present in `tests/e2e/` (e.g.
  `.tier-matrix.md` or `.gating-matrix.md`)? Required for the G dimension
  if the project has any subscription/plan/feature-flag/entitlement gates.
  If the project has no gates, mark N/A. If gates exist but artifact is
  missing, scaffold it in Step 7 from the gating source-of-truth in code
  (auth types, plan enums, feature-flag service config).

**Diagnosing CI-only flakes — always inspect the trace.** When a test
passes locally but fails in CI, do NOT guess. Download the failure artifact
(`playwright-report/` or `test-results/<spec>/trace.zip`), unzip it, and
read `0-trace.network` (JSON-lines). Grep for the failing route's API
calls — `/auth/me`, `/auth/v1/token`, redirects to `/login`. The HTTP
sequence is ground truth: a 401 on `/auth/me` followed by a 400 on
refresh-token followed by a redirect to `/login?session_expired=true` is
a definitive signature of stale-storageState / token-rotation. Without
this, you'll spend hours adding waits to symptoms instead of fixing the
auth helper.

---

### Step 5: Identify all features and user flows (both modes)

Explore the codebase to identify what *should* be tested:

- All routes / pages (read the router config).
- Key entities (DB schemas, types).
- CRUD operations (form submits, API calls, RTK / tRPC mutations).
- Auth flows (login variants, password reset, OAuth, invite acceptance, MFA).
- Multi-step flows (wizards, checkouts, onboarding).
- Tech stack (framework, DB, auth provider, payment provider, realtime).

For each feature, identify the **complete user goals**: create, edit, delete,
filter/search, error path. These become the units of work in the test plan.

#### Step 5a: Cross-reference with Supabase MCP (if available)

**Check first:** is the Supabase MCP server connected in this Claude Code
session? If yes AND the project uses Supabase, use it as **ground truth** for
the server-side surface — the codebase can be stale or only partially wired.

When MCP is available:

1. **Pull the schema** (`list_tables`) — every table is a potential entity
   the user can mutate. Cross-reference with the routes/forms found in the
   codebase.
2. **Pull edge functions** (`list_edge_functions`) — every edge function is a
   server-side operation. If the frontend calls it, it needs a test. If the
   frontend does NOT call it, flag it (see below).
3. **Pull RLS policies** (`execute_sql` against `pg_policies`) — every policy
   defines a role-gated behavior. Each role × policy combination is a
   potential P2 access-control test.
4. **Pull migrations** (`list_migrations`) — recent schema changes signal
   recently-added flows that may not have tests yet.

**Mismatch reporting** — flag these explicitly in the gap analysis:

- **"API exists, UI not wired"** — server-side endpoint / table / edge
  function exists, but no frontend code references it. This is **not a test
  gap** (no UI to drive yet) — it's a product gap. List separately so the
  dev knows it's product work, not test work.
- **"UI calls endpoint that doesn't exist server-side"** — frontend
  references a table / function that MCP doesn't see. This is a **bug** to
  surface (broken deploy, deleted migration, typo). List as a blocker, not a
  gap.

**Fallback when MCP is NOT available:**

- Read code only — the original Step 5 behavior.
- Note in the report: "Supabase MCP not configured — entity/mutation list
  derived from frontend code only. May miss server-side surface. Configure
  MCP for higher-fidelity audit."

For non-Supabase projects, skip Step 5a entirely. Note the auth/DB stack and
move on.

---

### Step 6: Gap analysis (both modes)

Group features into three buckets:

**Well covered** — features where important flows have proper Tier 1 journey
tests:
- `Feature` (`spec-file.spec.ts`) — N tests · note any caveats.

**Partially covered** — has tests but missing important flows:
- `Feature` (`spec-file.spec.ts`) — covered: [...] · missing: [...].

**No tests** — features with no spec file at all:
- `Feature` — one-line description of the user flow that should be tested.

Anti-patterns from Step 4 are also gaps. Every admin-seeded or
`waitForTimeout`-using test is **work to redo**, not done work.

---

### Step 7: Scaffold whatever infrastructure is missing

For each item Step 0 marked as missing, create it. Read the app source
deeply first (auth, entities, API patterns, roles) so the scaffolding is
**tailored to the detected stack** — never generic boilerplate.

**Web (Playwright / Cypress):**

- **`helpers/api.ts`** — API client for seeding/cleanup against the test
  environment. Functions like `seedUser`, `seedEntity`, `deleteUser`,
  `resetTestData`. Use the project's actual API endpoints and entity shapes
  (pull from Supabase MCP if available — Step 5a — for exact column names
  and table structures).
- **`helpers/auth.ts`** — real authentication against the test environment.
  `loginAsTestUser(page, role?)`, `TEST_USERS` map loaded from env vars
  (never hardcoded), `getAuthToken(role?)` for API seeding calls.
- **`fixtures/base.ts`** — `authenticatedPage` fixture that performs real
  login. Re-export `test` and `expect` so spec files import from here.
- **`.env.test.example`** — template with all required env vars detected
  from the project source. Real variable names, placeholder values
  (`<your-test-user-email>`, `<your-staging-url>`, etc.). The dev fills in
  real values in their own `.env.test` (gitignored) after the PR lands.
- **`playwright.config.ts`** — only if missing. `baseURL` from
  `process.env.E2E_BASE_URL || 'http://localhost:3000'` (or detected port),
  `storageState` reuse, retries off, screenshots/trace on failure, html +
  github reporter.

**Mobile (Maestro):**

- `.maestro/flows/` directory + `.maestro/config.yaml`
- Helper flows: `auth-login.yaml`, `delete-account.yaml`
- `.env.test.example` with mobile-relevant vars

**Mobile (Detox):**

- `.detoxrc.js` config tailored to detected RN/Expo setup
- `e2e/` directory with `init.js`
- `.env.test.example`

**No Docker.** Most clients use Supabase / staging deploy / test branch on
the live API. If the detected stack genuinely needs Docker (rare — typically
self-hosted Postgres + custom backend), scaffold a minimal `docker-compose.yml`
in the test directory; otherwise skip it entirely.

**Prescribed E2E doc set (project-agnostic)** — scaffold any missing files
flagged by Step 0 item 8. Keep client/project specifics out of the templates:

- **`tests/e2e/README.md`** — onboarding. Sections: How to run, Env vars,
  Conventions (link the `add-e2e-tests` skill for universal patterns),
  `## Sibling repos` (table populated from Step 0 item 9 if applicable),
  `## Project-specific gotchas` (start empty; the team fills in
  project-specific items — universals belong in the skill, never duplicated
  here), `## CI` (with an Enforcement subsection naming the ESLint rule
  and any coverage-reminder workflow once they exist).
- **Per-spec BDD headers** — do NOT backfill from this skill; flag missing
  ones in the audit report (Step 4 anti-pattern #20). `add-e2e-tests` adds
  headers when generating new specs.

**Not scaffolded:** `CHANGELOG.md` and `.coverage-matrix.md`. History
belongs in `git log tests/e2e/`; the spec inventory + edge-case gaps
digest in `.test-plan.md` already cover the matrix's job and stay fresh.
If a legacy project has either file, recommend deletion as part of the
audit cleanup pass.

Do **not** generate a `SETUP-GUIDE.md` document. The scaffolded files +
`.test-plan.md`'s readiness section + the report at the end of Step 10 are
the documentation. A separate guide is dead weight — it's instructions to do
work the command should have already done.

---

### Step 8: Write or update `.test-plan.md` (both modes)

`.test-plan.md` is the living shared artifact. Lives in the test directory
(`tests/.test-plan.md`, `e2e/.test-plan.md`, or wherever the test root is).
Both this command and `/add-e2e-tests` read and write to it.

#### When `.test-plan.md` does NOT exist — write the full file:

**Two-file output.** `.test-plan.md` is the **action plan** (short, dev-facing).
`.coverage-gaps.md` is the **reference audit** (long, full digest). The split
keeps the action plan scannable while preserving every gap, blocker, and
anti-pattern historical data point.

#### `.test-plan.md` template (action-oriented; aim for ≤150 lines)

```markdown
# E2E Test Plan

> **Action plan for the dev about to write or upgrade a spec.** What's next, what's done, what's missing.
>
> **Quality bar definitions** → `add-e2e-tests` skill (DoD — 6 dims).
> **Full edge-case digest + sibling-repo audit + blockers** → `.coverage-gaps.md`.
> **Gating source-of-truth** → `.tier-matrix.md` (or project equivalent).
> **Setup & onboarding** → `README.md`.
> **Change history** → `git log <test-dir>/`.

## A. Next action

**Now**: `Skill: add-e2e-tests <feature>` — one literal command to run. Pick the spec with the highest impact-per-effort: high-traffic feature AND already close to DoD.

**After that** (in order): <next 3–4 specs>. Rationale in § C.

Once a spec hits all-✅, re-run `Skill: e2e-audit` to refresh.

## B. Score dashboard

Per-dim coverage across all specs. **✅ = 1, 🟡 = 0.5, ❌ = 0**, summed / N → %.

| Dim | Score | Today | Gap to close |
| --- | --- | --- | --- |
| **H** | <pct>% | <a> ✅ · <b> 🟡 · <c> ❌ | <one-line action> |
| **E** | <pct>% | … | … |
| **M** | <pct>% | … | … |
| **R** | <pct>% | … | … |
| **A** | <pct>% | … | … |
| **C** | <pct>% | … | … |
| **G** | <pct>% | … | … |

**Overall: <pct>%** (mean across dims). **All-✅ strict count: <X> of <N>.** Target ≥ 80%.

## C. Backlog (per-spec upgrade queue)

Ordered by priority. **Priority rule**: high-traffic + close-to-DoD first.

| Rank | Spec | Today | Command to upgrade | Notes |
| --- | --- | --- | --- | --- |
| 1 | … | … % | `Skill: add-e2e-tests <feature>` | … |

**New specs to write** (no current file):

| Topic | Command | Complexity |
| --- | --- | --- |
| <topic> | `Skill: add-e2e-tests <feature>` | <simple/moderate/complex> |

**Suite-wide hygiene additions** (not feature specs — standing hardening tasks the audit surfaces):

| Item | Status | Command / action |
| --- | --- | --- |
| Route-smoke spec (iterates nav routes, asserts 200 + app-shell selector) | ❌ missing / ✅ exists at `route-smoke.spec.ts` | Add per anti-pattern #28 |
| Console-error guard in base fixture (`page.on('console')` with whitelist) | ❌ missing / ✅ exists in `fixtures/base.ts` | Add per anti-pattern #29 |
| Per-worker storageState isolation (anti-pattern #19) | ❌ shared / ✅ per-worker | Refactor per anti-pattern #19 |
| BDD header on every spec | <X/N> | Backfill missing per anti-pattern #20 |

## D. Recently done (last 10 commits on `<test-dir>/`)

Refreshed by `e2e-audit`. To regenerate: `git log -10 --oneline <test-dir>/`.

```
<commit-sha> <short-message>
…
```

## E. Active work in branch

| Spec | Branch | State |
| --- | --- | --- |
| <spec> | <branch> | <one-line state> |

## F. Skill cadence (when to re-run)

| Trigger | Skill | What it touches |
| --- | --- | --- |
| Want to write or upgrade a spec | `add-e2e-tests <feature>` | Generates/edits spec with DoD stubs for all dims |
| Finished upgrading a spec | `e2e-audit` | Refreshes § B / § C / § D, `.coverage-gaps.md` |
| Added/removed a spec file | `e2e-audit` | Same |
| Modified the gating source-of-truth (e.g. `FeatureKey` enum) | manual: update `.tier-matrix.md`, run sync test | Sync test fails if drifted |
| Cold review | `e2e-audit` | Quarterly re-categorization |

**Default cadence**: `e2e-audit` runs at end of every PR that touches the test dir. If untouched, quarterly.

## G. Active blockers (cross-cutting)

Only suite-wide blockers. Per-spec blockers in `.coverage-gaps.md` § C.

| Blocker | Owner | Notes |
| --- | --- | --- |

### G.1 Fixmes still open on `main` (optional sub-section)

When a repo has 10+ `test.fixme` calls scattered across specs, hoist them
into one table so the next maintainer sees the full backlog and the
owner-per-row. **Re-verify line numbers via grep before assuming this table
is up-to-date** — line numbers shift with every spec edit.

| # | Spec : line | Root cause | Owner | Recommended action |
| --- | --- | --- | --- | --- |
| 1 | `<spec>.spec.ts:<line>` | <one-line cause> | <PM / FE / BE / E2E maintainer> | <one-line action> |

**Classify every fixme into one of five categories** so the table tells the
reader what *kind* of work the destap requires, not just "still pending":

1. **Product gap** — the feature being tested doesn't exist yet (or the
   contract is still being designed). Owner: PM / product. Action: ship
   the feature, then un-fixme.
2. **Backend gap** — endpoint, RLS policy, edge function, or migration is
   missing/broken. Owner: BE. Action: ship the backend change, then
   un-fixme.
3. **Test infra gap** — the suite needs a helper, fixture, seed, reset
   utility, or env var that doesn't exist yet. Owner: E2E maintainer.
   Action: land the helper, then destap. *Most "fixme for one PR" fixmes
   land here* — keep this category small.
4. **Environment / flake** — the test passes locally but flakes on CI
   (timing, network, parallel-write race, token rotation). Owner: E2E
   maintainer. Action: diagnose via trace (see "Diagnosing CI-only
   flakes" above), fix root cause, then un-fixme. Do **not** "fix by
   adding a wait."
5. **Deferred by policy** — the test is intentionally parked (waiting on
   a sibling repo, blocked by a strategic decision, low-priority edge
   case). Owner: explicit human. Action: track separately; revisit on
   schedule.

**Re-verification cadence.** Fixmes go stale silently — root causes get
fixed in unrelated PRs and nobody flips the test back on. Re-verify the
fixme table on every `e2e-audit` run:

- For each row, run the originally-blocking scenario (or read the linked
  ticket / PR). If the root cause is resolved, propose un-fixme'ing the
  test in the next PR.
- If the fixme has been open > 30 days with no progress, escalate to
  the owner — either reclassify (was this really a product gap, or
  parked work?) or close (the test isn't coming back).
- A fixme without a category + owner + linked issue/PR is itself an
  audit finding. Flag it.

## H. Readiness checklist

Pre-populate every row below on first audit; mark each ✅/⏳/❌ with a one-line note.
The checklist exists so the audit captures "what hygiene is wired" — leaving
rows blank means the team has to guess. Add project-specific rows after the
standard set.

| Check | Status |
| --- | --- |
| Test env URL non-production (`.env.test*`, `playwright.config.ts`) | <✅/❌ + path> |
| Dedicated test credentials in env vars (not hardcoded) | <✅/❌> |
| Seed/reset helpers (`helpers/api.ts` with create + cleanup) | <✅/❌> |
| `storageState` reuse (per-worker, not shared) | <✅/❌> |
| `afterEach` / `afterAll` cleanup of created entities | <✅/❌> |
| ESLint rule forbidding `waitForTimeout` + `networkidle` | <✅/❌ + config path> |
| Required CI check on `main` running e2e | <✅/❌ + workflow path> |
| Gating source-of-truth artifact (`.tier-matrix.md` or equivalent) | <✅/❌/N/A — no gating> |
| Sync test for gating artifact (fails if drifted from source code) | <✅/❌/N/A> |
| `assertSurfaceHiddenFor` helper for R UI− | <✅/❌> |
| Cross-tenant seed (2nd tenant + 2nd-role accounts) in CI | <✅/❌ + seed location> |
| BDD header on every spec (anti-pattern #20) | <X/N specs> |
| Route-smoke spec exists (anti-pattern #28) | <✅/❌> |
| Console-error guard in base fixture (anti-pattern #29) | <✅/❌> |
| Branch protection: e2e check required on `main` | <✅/❌ + manual step> |
```

#### `.coverage-gaps.md` template (reference; can be long)

```markdown
# E2E Coverage Gaps — full digest

> Companion to `.test-plan.md`. Holds the full audit data.
> **Refreshed by:** `e2e-audit`. **Last refresh:** <date>.

## A. Edge-case gaps (per-category)

Concrete gaps from every spec's `Edge cases NOT covered:` BDD header.

| Category | Gap | Action |
| --- | --- | --- |
| **Cross-repo** | … | new harness / `extend X.spec.ts` / blocked |
| **Auth** | … | … |
| **Roles** | … | … |
| **Lifecycle** | … | … |

## B. Coverage already in sibling repos

Audited <date> by grepping <sibling-test-dirs>.

`<sibling-repo>` (<framework>):
- <feature list>

## C. Blocked (waiting on product / infra)

| # | Item | Spec | Blocker | Next action |
| --- | --- | --- | --- | --- |
| B1 | … | … | … | … |

## D. Anti-pattern guard (historical)

| Pattern | Count today | Guard |
| --- | --- | --- |
| `page.waitForTimeout` | 0 | ESLint rule in `<config>` — fails CI |
| … | … | … |
```

#### When both files already exist — refresh only what's derivable

**`.test-plan.md`**:
- § B Score dashboard — recompute per-dim % from latest BDD headers and spec contents.
- § C Backlog — re-rank by (impact × proximity-to-DoD); add new specs, retire finished ones.
- § D Recently done — `git log -10 --oneline <test-dir>/`.
- § E Active work — `git status` + `git log <branch>..HEAD`.
- § H Readiness — re-check each row against the codebase.

**`.coverage-gaps.md`**:
- § A Edge-case gaps — re-derive from BDD headers. Add new, remove resolved.
- § B Sibling-repo coverage — re-grep when sibling repos are listed in `README.md`.
- § C Blockers — refresh status; mark resolved.
- § D Anti-patterns — full re-grep.

Stable sections (definitions, headers, prose explanations) untouched unless

Leave stable sections (Conventions, custom team notes) untouched unless
you have new information.

---

### Step 9: Output the review in chat — HARD PAUSE for approval

Output ALL of the following in chat (not only in `.test-plan.md`):

1. **Mode** — Audit (existing real specs found) or Fresh Project (no real specs — Step 7 scaffolded the infra).
2. **Headline metrics:**
   - **All-dimensions %** (specs with ✅ on every quality column the project enforces) — the real coverage signal. Lead with this.
   - Functional-flow % (Tier 1 / total) — secondary, hides the dimension gaps.
   - Per-dimension breakdown (✅ vs 🟡 vs ❌ per column).
   - Anti-pattern count broken down by type
   - Assertion-weakness count (`toBeVisible()`-only after mutation)
   - CI gate status (enforced / not enforced / missing workflow)
   - Infra gaps (storageState, cleanup fixtures, ESLint rule)
3. **Complete list of uncovered flows.** EVERY flow from Step 5 that does not
   have a Tier 1 test today, grouped by priority:
   - **P0** — revenue / auth / compliance blockers.
   - **P1** — core CRUD and multi-step flows.
   - **P2** — admin, edge cases, role-gated routes, error paths.

   For each flow: spec filename, 1-line user journey, the assertion
   (what "passing" looks like), and a complexity tag (simple / moderate /
   complex). Format each line so it's directly copy-paste-able into
   `/add-e2e-tests`. Do NOT truncate. Do NOT say "and more" — list them all.
4. **Tests to refactor** (not rewrite from zero) — admin-seeded,
   `toBeVisible`-only after mutation, `waitForTimeout`-using, etc. List
   `file:line` + the fix.
5. **Top 3 to start with** — recommendation only (highest business risk ×
   lowest implementation cost). User may reorder.
6. **Ambiguities to resolve before Step 10:**
   - CI target (localhost vs staging) — which?
   - Duplicate / legacy test directories — delete or keep?
   - Existing CI workflow — merge into or replace?
   - Existing `.test-plan.md` — overwrite stable sections or only refresh
     derived ones?

**Do NOT proceed to Step 10 until the user has approved the complete flow
list, not just the top 3.** The user is the editor here — they have domain
knowledge the command doesn't (implicit flows, undocumented features,
internal tools, partner integrations, legacy screens). Expect them to add
flows the command missed.

---

### Step 10 (post-approval only): Land the CI gate, then verify

Do ALL of the following on a single new branch before opening the PR:

1. **`.test-plan.md`** is already written/updated by Step 8 — confirm it's on
   the branch.
2. **Write the GitHub Actions workflow file** at `.github/workflows/e2e.yml`
   (or the detected CI equivalent). It must:
   - Trigger on `pull_request` AND `push` to `main` / `master`.
   - Skip draft PRs: `if: ${{ github.event.pull_request.draft == false || github.event_name == 'workflow_dispatch' }}`
   - Install deps with the detected package manager (npm / yarn / pnpm / bun).
   - Install the correct test runner (`npx playwright install --with-deps
     chromium` for Playwright; Maestro cloud or self-hosted runner for
     Maestro; Detox setup for Detox).
   - Run the full e2e suite.
   - Upload `playwright-report/` (always) + `test-results/` (on failure).
   - Use `concurrency: { group: e2e-${{ github.event.pull_request.number || github.ref }}, cancel-in-progress:
     true }` so stale PR runs get canceled.
   - Set a job-level `timeout-minutes: 20` (prevents runaway billing).
   - Create `.github/workflows/` if it doesn't exist.
   - If a workflow already exists, **diff and ask** — never silently
     overwrite.

   **Performance: add these caching layers** (saves 2-5 min per run):

   - **Playwright browser cache** — cache `~/.cache/ms-playwright` keyed
     on `playwright-${{ runner.os }}-${{ playwright-version }}`. On cache
     hit, skip `npx playwright install` and only run
     `npx playwright install-deps chromium` (system libs only). Extract
     the Playwright version with:
     `echo "version=$(npx playwright --version | awk '{print $2}')" >> $GITHUB_OUTPUT`
   - **Next.js / framework build cache** — cache `.next/cache` (or
     equivalent) keyed on `${{ hashFiles('**/package-lock.json') }}-${{ hashFiles('src/**/*') }}`
     with restore-keys falling back to OS + lockfile hash.
   - **Node modules cache** — use `actions/setup-node@v4` with
     `cache: "npm"` (or yarn/pnpm). This is built-in and handles
     `node_modules` automatically.
   - **Build the app before tests** — `npm run build` then `npm start`
     in the Playwright `webServer` config (not `npm run dev`). Production
     builds are faster to serve and more representative.
   - **Skip Sentry/analytics in CI** — set `NEXT_BUILD_SKIP_SENTRY: "1"`
     or equivalent to avoid uploading source maps during test builds.
   - **Bump Node memory** — `NODE_OPTIONS: "--max-old-space-size=4096"`
     prevents OOM on large Next.js builds.
3. **Write or update the ESLint rule** forbidding `page.waitForTimeout(` and
   `waitForLoadState('networkidle')`. Use Chorus's config as the reference.
   Skip if the repo doesn't use ESLint.
4. **Validate the workflow YAML parses** (`actionlint` if available,
   otherwise basic YAML check).
5. **Open a PR** titled `chore: E2E test plan + CI gate for <repo>`. Include
   `.test-plan.md`, the workflow file, and the ESLint rule on one branch.
6. **Wait for the first CI run.** Report the outcome:
   - ✅ pass → tell the user the gate is live.
   - ❌ fail → surface the failure logs. Do NOT claim the gate is working
     until at least one real run passes.
   - If the repo has 0 e2e tests today, the workflow should still run and
     pass with "no tests matched" rather than error — verify this.
7. **Tell the user the manual follow-up** they must do:
   - Mark the e2e check as a **required status check** in GitHub branch
     protection settings for `main` / `master`. The command cannot do this
     via API without elevated scopes — call out the manual step explicitly.

Do NOT draft a Slack message. Do NOT push to a desktop path. The
`.test-plan.md` and the PR are the only artifacts.

---

### Step 11: Stop local services you started — DO NOT leak processes

**This step is mandatory.** If during the audit/scaffold you started any
local servers, databases, or containers (frontend dev server, API dev
server, secondary services, Docker stacks, emulators), **stop them before
reporting** unless the user explicitly said to leave them running. Dev
servers leak memory over time (Next.js dev servers commonly hold 4–6 GB
each); leaking them across sessions silently degrades the user's machine.

```bash
# Find what's listening on the ports you used
lsof -i :3000 -i :3001 -i :3002 -nP 2>/dev/null | grep LISTEN

# Kill Node/Next dev servers you started
kill <pid> <parent-pid>

# Stop Supabase if you started it — restart is cheap (~30s, data preserved)
cd <project-server-dir> && npx supabase stop
```

**Decision rule:**
- You started it AND no one else is using it → stop it. Supabase included —
  `supabase stop` + `supabase start` takes ~30s and data is in Docker volumes.
- It was already running when you arrived → leave it.
- Unsure → ask explicitly before stopping.

Mention what you stopped and what you left running in the final report.

---

## Output summary at the end

After Step 11, print:

- Mode: [Audit / Fresh Project]
- Total tests: X (Tier 1: N, Tier 2: N, Tier 3: N) [Audit Mode only]
- Anti-patterns found: N (broken down by type)
- Files scaffolded: [list, Fresh Project Mode only]
- `.test-plan.md` location
- PR URL
- CI status: live / pending / failed
- Manual follow-up required: [mark required check in branch protection]
