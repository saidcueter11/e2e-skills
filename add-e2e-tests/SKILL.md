---
name: add-e2e-tests
description: Use whenever the user wants to create, add, write, generate, scaffold, or update an end-to-end / E2E test, spec, or test flow — for any framework (Playwright, Cypress, Maestro, Detox). Trigger phrases include "add e2e test", "write a test for X", "cover Y in e2e", "generate tests for this feature", "let's tackle the P1/P2 coverage gaps", any reference to working on items in `.test-plan.md`, or any mention of CRUD/journey tests against a real backend. Tests hit real APIs against a test environment — no mocks, no fake data. Reads `.test-plan.md`, writes the complete CRUD set per feature, updates `.test-plan.md` after each spec.
argument-hint: <feature description or test-plan line> e.g. "Goal Creation Wizard — mentor completes 7-step wizard and goal appears in list"
---

# add-e2e-tests

## Core philosophy — test real user journeys against real APIs

Each test simulates what a **real user does to accomplish a goal**, hitting a
**real backend** in a test environment. If a test would pass with the backend
down, it is not an E2E test. If it uses fake data injected via route
interception, it cannot catch real regressions.

**One flow, one test.** A user who creates a client opens the dialog, fills
the form, submits, sees the item in the list. That is one test, not four.

**No mocks on happy paths.** Route interception is only allowed for
simulating infrastructure failures (network errors, timeouts, 500s) — never
for business logic.

**Complete the feature.** When given a feature name, generate the **whole
CRUD set** (create, edit, delete, list/filter, error path) — never just one
operation. Do not stop after create and print "remaining." Finish all 3–7
tests for the feature in one run.

---

## Step 0: Project readiness gate

**This step is mandatory. Do not skip it.**

Before writing any test, verify the project can support real E2E tests:

1. **Test environment exists** — A backend/API URL that is NOT production.
   - `.env.test`, `.env.local`, `.env.staging`, with a non-production API URL.
   - `playwright.config.ts` `baseURL` — must NOT point to a production
     domain.
   - Docker Compose, Supabase local, Firebase emulator, or similar.
2. **Test database** — separate from production (Docker, separate schema,
   in-memory DB, local instance).
3. **Test credentials** — dedicated test user accounts (env vars with test
   email/password, seed scripts, or auth helpers).
4. **Seed/reset capability** — a way to create and tear down test data (API
   endpoints, seed scripts, DB reset commands).
5. **Production safety** — `baseURL` is localhost/staging/test, no
   production API keys in test env files.

### If any check fails → STOP

Do NOT fall back to mocks. Do NOT write tests with fake data. Tell the user
to run `/e2e-audit` first — it generates the project-specific
`SETUP-GUIDE.md` and (in Fresh Project Mode) scaffolds the helpers / fixtures
/ env file. Come back to `/add-e2e-tests` when readiness passes.

---

## Step 1: Discover project conventions

**First, check if `.test-plan.md` exists** in the test directory. If it does:

- Read it — it has all conventions, existing specs, and prioritized work.
- Find the user's requested feature in "What to write next." If a complexity
  tag and target spec file are listed, use them. If not, infer from source.
- Skip the rest of Step 1.

**If no `.test-plan.md`**, scan the project:

1. `playwright.config.ts` — base URL, projects, timeouts, test directory.
2. Find the test directory and explore its structure.
3. Look for **helpers/** — auth flows, API clients, seed utilities.
4. Look for **fixtures/** — custom Playwright fixtures.
5. Look for **pages/** — Page Object classes.
6. Read 1–2 existing spec files to calibrate style and patterns.
7. Verify `baseURL` is NOT a production URL.

Tell the user to run `/e2e-audit` first to generate `.test-plan.md` if it
doesn't exist — `/add-e2e-tests` works fine without one but loses the
prioritized-work-list integration.

#### Step 1a: Cross-reference with Supabase MCP (if available)

Check whether the Supabase MCP server is connected in this Claude Code
session. If it is AND the project uses Supabase, use it as **ground truth**
for the server-side surface — this lets you write **precise assertions**
instead of guessing column names, response shapes, or RLS behavior.

When MCP is available, pull:

1. **Schema for the tables this feature touches** (`list_tables`) — exact
   column names, types, defaults, foreign keys. Use these in seed payloads
   and post-mutation assertions.
2. **RLS policies** for those tables (`execute_sql` against `pg_policies`)
   — defines what each role can do. Use to write access-control / negative
   tests that verify real RLS, not assumptions.
3. **Edge functions** the feature calls (`list_edge_functions`,
   `get_edge_function`) — exact request/response shape. Assertions can
   match the real contract instead of guessing.
4. **Recent migrations** (`list_migrations`) — flags whether the schema
   changed recently and the helpers/fixtures may need updating.

If MCP is **not** connected, fall back to reading the codebase. Tests will
still be real-API tests, but assertions will be less precise and seed
payloads may need adjustment when the AI guesses wrong. Note this in the
final report so the user knows to configure MCP for higher-fidelity tests.

For non-Supabase projects, skip Step 1a entirely.

**Supabase-specific seed gotcha (if the project uses GoTrue / Supabase auth):**
When seeding `auth.users` rows via direct SQL `INSERT` (rather than the GoTrue
Admin API), GoTrue requires the token columns to be empty strings, not NULL.
Forgetting this produces a login 500 ("Database error querying schema") that
looks like an RLS or schema bug. Always include:
```sql
confirmation_token = '',
recovery_token     = '',
email_change_token_new     = '',
email_change_token_current = '',
phone_change_token         = '',
reauthentication_token     = ''
```
Prefer the Admin API (`supabase.auth.admin.createUser`) when possible —
it sets these correctly automatically.

**Key things to identify (regardless of MCP):**
- How does auth work? (real login flow, test credentials, token endpoint)
- How is test data managed? (API seeding, DB seeds, cleanup scripts)
- What is the `test` import? (base Playwright or custom fixture?)

---

## Step 2: Map the user journeys

Based on the requested feature, read the source code deeply. Identify the
**complete user goals** for the feature:

| User goal | What they do | What they expect |
|---|---|---|
| Create | Open dialog, fill fields, submit | Item appears in list, persists on reload |
| Edit | Click edit, verify pre-fill, change field, submit | Change reflected in list and backend |
| Delete/archive | Click action, confirm | Item removed from list, gone from API |
| Filter/search | Enter term or toggle | Only matching rows visible |
| Error handling | Submit invalid data | Real server validation error shown, form stays open |

Also map:
- **API contracts** — endpoint, method, expected payload for each mutation.
  If MCP is available, use the **actual** schema/edge-function definitions.
- **Error paths** — what does the UI show for invalid input? (Real server
  validation, not mocked.)
- **Conditional rendering** — what changes based on data state or role?
  RLS policies tell you the answer if MCP is connected.

---

## Step 3: Seed and clean up test data via real API

### Never use `page.route()` for happy-path data. Never use factory builders with fake objects.

Every test seeds its own data via **real API calls** against the test
environment, and cleans up afterward.

### Pattern: API seed helper

Create or reuse a helper in `helpers/api.ts`. **If Supabase MCP is
available, use the actual table schemas** so seed payloads are exact:

```typescript
import { request } from '@playwright/test';

const API_URL = process.env.TEST_API_URL || 'http://localhost:3000/api';

export async function getAuthHeaders(role?: string): Promise<Record<string, string>> {
  // Get a real auth token from the test environment
  const email = role === 'admin'
    ? process.env.TEST_ADMIN_EMAIL
    : process.env.TEST_USER_EMAIL;
  const password = role === 'admin'
    ? process.env.TEST_ADMIN_PASSWORD
    : process.env.TEST_USER_PASSWORD;

  const ctx = await request.newContext({ baseURL: API_URL });
  const res = await ctx.post('/auth/login', { data: { email, password } });
  const { token } = await res.json();
  return { Authorization: `Bearer ${token}` };
}

export async function seedClient(
  overrides: Record<string, unknown> = {},
  authHeaders?: Record<string, string>
) {
  const headers = authHeaders || await getAuthHeaders('admin');
  const ctx = await request.newContext({ baseURL: API_URL });
  // Use the EXACT column names from MCP-pulled schema if available
  const res = await ctx.post('/clients', {
    headers,
    data: { name: `Test Client ${Date.now()}`, ...overrides },
  });
  return res.json();
}

export async function deleteClient(id: string, authHeaders?: Record<string, string>) {
  const headers = authHeaders || await getAuthHeaders('admin');
  const ctx = await request.newContext({ baseURL: API_URL });
  await ctx.delete(`/clients/${id}`, { headers });
}
```

### In spec files

```typescript
let createdIds: string[] = [];

test.afterEach(async () => {
  for (const id of createdIds) {
    await deleteClient(id);
  }
  createdIds = [];
});

test('admin creates a client', async ({ authenticatedPage: page }) => {
  // No seeding needed — this test IS the create flow
  // Navigate, fill form, submit...

  // After submit, verify via API that the record exists
  const headers = await getAuthHeaders('admin');
  const ctx = await request.newContext({ baseURL: API_URL });
  const res = await ctx.get('/clients?name=Acme', { headers });
  const clients = await res.json();
  expect(clients).toHaveLength(1);
  // If MCP gave us the schema, assert on EXACT columns:
  // expect(clients[0]).toMatchObject({ name: 'Acme', status: 'active', org_id: ... });
  createdIds.push(clients[0].id);
});
```

### Rules for data management

- Each test creates its own data — no shared mutable state between tests.
- Each test cleans up after itself in `afterEach`.
- **Use `RUN_ID + timestamp` in any seed value the test will SEARCH for.**
  Plain timestamps prevent intra-run collisions but NOT cross-run pile-up
  in shared staging environments. Without a `RUN_ID` prefix, prior runs
  that didn't clean up will break strict-mode locator queries (e.g.
  `getByTestId('row').filter({ hasText: 'SearchAlpha' })` matches 3 rows
  → fails). Prefer:
  ```typescript
  // helpers/api.ts
  export const RUN_ID =
    process.env.GITHUB_RUN_ID ??
    process.env.E2E_RUN_ID ??
    `local-${Date.now().toString(36)}`;

  // spec
  const alphaName = `SearchAlpha-${RUN_ID}-${Date.now()}`;
  await api.seedContact({ first_name: alphaName, ... });
  await contacts.search(alphaName);          // unique substring → strict-safe
  await expect(contacts.rowByName(alphaName)).toBeVisible();
  ```
- Load credentials from environment variables, never hardcode in spec files.
- If a test needs prerequisite data (e.g., edit flow needs an existing
  record), seed it in `beforeEach` via API.

### Optimistic-update race after mutation clicks

Most modern client-side caches (RTK Query, React Query, SWR, Apollo with
`optimisticResponse`, etc.) update the UI cache **immediately** on a mutation
— before the network request completes. If you assert on backend state
(`api.list...()`) right after clicking a confirm button, the underlying
request may still be in-flight and the DB record still exists. The test
non-deterministically passes or fails depending on network speed.

**Fix: capture `waitForResponse` BEFORE the click, await AFTER:**

```typescript
const waitForMutation = page.waitForResponse(
  (r) =>
    r.url().includes(`/<endpoint-prefix-for-this-mutation>/`) &&
    r.request().method() === "DELETE", // or PATCH / PUT / POST
  { timeout: 15_000 },
);
await confirmBtn.click();
await waitForMutation; // wait for the server to actually process the request

// THEN do UI or API assertions
await expect(row).not.toBeVisible({ timeout: 10_000 });
const list = await api.listItems({ search: identifier });
expect(list.find((x) => x.id === seeded.id)).toBeUndefined();
```

This pattern applies to any mutation that has an optimistic update: row
delete, archive, status toggle, inline edit, etc. The `waitForResponse`
captures the promise before the click so there is no race window.

**Verify URL + method against the source before writing the filter.** A
mismatched filter silently never fires → 15s timeout → test fails after
the action actually succeeded. Don't guess:

1. Open the RTK Query (or fetch) definition for the mutation and read the
   exact `url` and `method`. Common traps: edits use **PUT** (not PATCH);
   archives use **PATCH** on the parent route (not DELETE); list endpoints
   are POSTs that look like reads.
2. Anchor the URL with a regex, not `.includes()`. `includes('/messages/')`
   matches `/messages/read`, `/messages/{id}/reactions`, and the actual
   target — the wrong one resolves first. Prefer:
   ```ts
   /\/conversations\/[^/]+\/messages\/[^/]+(\?|$)/.test(r.url())
   ```
3. If the project has a Supabase MCP server, you can also confirm against
   edge-function signatures rather than reading the RTK file.

### Nested API response shapes — never assume top-level fields

APIs backed by join queries (REST with embedded includes, GraphQL, Supabase
`.select('*, related(*)')`, Prisma `include`, Rails `serializer`s, etc.)
often return related data **nested**, not at the top level. A common trap:
the same logical field can be in different positions depending on which
parent record materialised the row. Example: a list endpoint that merges
real users with pending invitations may surface the user's email at
`row.profile.email` for one shape and at `row.email` for the other.

**Before writing a find/filter/assertion predicate, inspect the actual API
response.** Use a real API call from the test environment, MCP tools (if
available), or read the serialiser/resolver code:

```typescript
// ❌ WRONG — assumes top-level field; silently always undefined for the other shape
const found = list.find((row) => row.email === searchTerm);

// ✅ RIGHT — handle both shapes defensively, document why
const found = list.find(
  (row) => ((row as any).profile?.email ?? row.email) === searchTerm
);
```

Update the TypeScript interface to match the real shape — mark fields that
only appear in one branch as optional and add the nested branch explicitly:

```typescript
export interface ListRow {
  id: string;
  // Field may live nested under `profile` for join-backed rows; the
  // top-level shorthand only appears when the row has its own profile.
  email?: string;
  profile?: { email?: string; [key: string]: unknown };
  [key: string]: unknown;
}
```

### Feature-gate / entitlement probe in `beforeAll`

If a feature is gated behind a subscription tier, plan, role, feature flag,
or entitlement, tests should **probe for availability** in `beforeAll` and
`skip` gracefully rather than failing with a 403 mid-test. This keeps the
suite clean on envs that lack the gated feature (local dev with a basic plan,
CI with a different seed, accounts without the entitlement, etc.).

```typescript
let featureAvailable = true;

test.beforeAll(async () => {
  try {
    const probe = await api.createGatedItem({ name: `probe-${Date.now()}` });
    await api.deleteGatedItem(probe.id).catch(() => {}); // clean up the probe
  } catch (e: any) {
    if (e.message?.includes("403") || e.message?.includes("PERMISSION_DENIED")) {
      featureAvailable = false;
    } else {
      throw e; // unexpected error — surface it
    }
  }
});

test("uses the gated feature", async ({ page }) => {
  test.skip(!featureAvailable, "Feature requires plan/entitlement upgrade — skipping on this env");
  // ...
});
```

**Important:** if the probe itself seeds data, cancel/cleanup it immediately
(as shown above). Don't rely on the `afterAll` cleanup for probes — the probe
may succeed but the test may never run due to `test.skip`.

**Server-side cache invalidation** — many entitlement/plan checks use an
in-memory cache on the API server (common in Node, Python, Ruby app servers
to avoid hitting the DB on every request). Changing the plan/flag in the DB
does NOT immediately change what the server returns — the cache entry must
expire or the server must restart. If the probe still 403s after a confirmed
DB upgrade:

1. Verify the DB record is set correctly (plan, flag, entitlement column).
2. Restart the API server (or wait for the cache TTL).
3. Clear cached auth state (storageState files, refresh tokens) and re-run
   the auth setup — a fresh auth token forces the server to re-resolve the
   user's plan/entitlement and miss the cache.

### Test user pool contract

As an e2e suite grows it accumulates many test identities (admin, basic-user,
role-A, role-B, cross-tenant variants, mobile-client, …). Without a contract,
specs mutate each other's state and you get cross-test contamination that is
painful to debug — a passing-in-isolation test fails in CI because a sibling
spec changed the account's onboarding flag, MFA factor, or plan tier.

Establish the contract up front and write it down:

1. **Registry.** Maintain a single source of truth (in `tests/e2e/README.md`
   or a similar onboarding doc) listing every test account: email/handle,
   role/plan/tier, which spec(s) own it, and what state mutations are
   allowed against it. New accounts get added in the same PR that
   introduces them.
2. **Mutation contract per account.** Classify each account as exactly one of:
   - **Read-only** — tests only log in and assert on existing seeded state
     (cross-tenant verifier accounts, "negative" accounts used to assert
     403s often fall here). No mutations of any kind.
   - **Idempotent mutations** — tests create/delete rows scoped to that
     account; each test cleans up after itself. The account-level state
     (plan, role, onboarding flag) is never touched.
   - **Irreversible state** — tests change account-wide settings (MFA
     enrollment, plan upgrade, onboarding completion, password). Requires
     three-layer cleanup (see next section). Should be a rare, named
     account — never the default test user.
3. **Cross-test isolation.** Specs on the same account but in different
   files must not assume initial state. Every spec sets up what it needs
   via API seed/reset in `beforeAll`/`beforeEach`. If shared state IS
   required (e.g. an admin account that owns long-lived seed records),
   document it and gate writes behind a runtime check that refuses to
   touch shared rows.
4. **Owner specs.** Each account has exactly one "owner" spec/suite that
   is allowed to perform irreversible mutations on it. Other specs are
   read-only against that account. Enforce in code review.
5. **Reset helpers for one-shot product flows.** Some product flows are
   intentionally single-use per account (onboarding wizard, signup intent,
   first-time setup, accept-invite). The second run hits a guard ("you
   already finished this") that wasn't designed to be undone from the UI.
   Provide an admin-privileged reset helper in `helpers/api.ts` that the
   owner spec can call in `beforeAll` / `try-finally`. Guard the helper
   with a **runtime allowlist** so it can only act on a known test account:

   ```ts
   static async resetOneShotFlow(userId: string): Promise<void> {
     const allowedEmail = process.env.TEST_OWNER_EMAIL;
     if (!allowedEmail) throw new Error("resetOneShotFlow requires TEST_OWNER_EMAIL");
     const admin = adminClient();
     const { data, error } = await admin.auth.admin.getUserById(userId);
     if (error || !data?.user) throw new Error(`getUserById failed: ${error?.message}`);
     if (data.user.email !== allowedEmail) {
       throw new Error(`refused: ${userId} resolves to ${data.user.email}, not ${allowedEmail}`);
     }
     const { error: delErr } = await admin.from("one_shot_table").delete().eq("user_id", userId);
     if (delErr) throw new Error(`reset failed: ${delErr.message}`);
   }
   ```

   The allowlist uses an existing env credential (email) — do **not** add
   a new env var that stores the raw UUID; UUIDs drift across DB resets
   and the credential pattern is already established in the project.

6. **Per-slot pool isolation when CI runs PRs in parallel.** A single
   shared account (`mentor@test.com`) works when one PR runs CI at a time;
   it breaks the moment two PRs run concurrently and both try to mutate
   the same account. Sharding within a single PR is not the same problem
   — those shards are siblings of one job and can divide roles internally
   via `workerIndex`. The cross-PR problem needs **per-slot tenant pools**.

   Pattern (mature projects):
   - Pick a small fixed number of slots N (e.g. 4) — the cap on parallel
     PRs in flight. Document the cap; CI cancels-in-progress per PR so
     re-pushes don't double-count.
   - Seed the test DB once with N copies of every account, namespaced by
     slot: `mentor.s0@…`, `mentor.s1@…`, …, `mentor.sN-1@…`. Each slot
     gets its own tenant/org so cross-tenant tests stay valid.
   - Per-shard / per-worker accounts live inside the slot: `mentor.sN.wK`
     for shards × workers. The TS helper derives the email from
     `(slotIndex, shardIndex, workerIndex)` — no DB lookup, no manifest.
     Keep it in lock-step with the SQL seed; the SQL is source of truth.
   - CI assigns the slot deterministically: `SLOT = PR_NUMBER % N`. Two
     PRs that hash to the same slot still collide — accept this as a
     known small-probability event, or layer a lease/queue on top.
   - Local dev keeps the single-account fallback: when `E2E_SLOT_INDEX`
     is unset, helpers fall through to legacy `TEST_*_EMAIL` env vars.

   ```ts
   // helpers/slot-spec.ts (sketch)
   export function credsForRole(env: SlotEnv, role: SlotRole, workerIdx: number): SlotCreds {
     const n = env.slotIndex;
     switch (role) {
       case "mentor": {
         const w = (env.shardIndex - 1) * env.workersPerShard + workerIdx;
         return { email: `mentor.s${n}.w${w}@${DOMAIN}`, password: env.password };
       }
       case "client": return { email: `client.s${n}@${DOMAIN}`, password: env.password };
       // …
     }
   }
   ```

   **When NOT to bother**: small suites (<20 tests), single-developer
   projects, projects with a fresh per-job DB (Docker / ephemeral
   Postgres). The single-account pattern is fine until you actually
   observe cross-PR flakes — premature pooling adds seed complexity for
   no benefit. Adopt this only when CI runtime + PR throughput make
   collisions inevitable.

   **One-shot flows still need their own slot rows.** Accounts used by
   irreversible mutations (onboarding submit, MFA enroll) must be added
   to the pool seed with N copies, just like read-write accounts.
   Otherwise the reset helper from rule #5 fights itself across PRs.

### Irreversible side-effect tests — defense-in-depth cleanup

Some tests exercise flows that leave the test user account in a **broken
state if the test crashes mid-run**, breaking every subsequent test on that
account until a human cleans up. Examples:

- **MFA enrollment** — once a TOTP factor is `verified` on `auth.mfa_factors`,
  the next login requires a code from the never-saved secret. Suite is
  bricked until the factor is deleted out-of-band.
- **Password change** — if the change succeeds but the spec crashes before
  restoring the original, every other spec that logs in as that user 401s.
- **Account-state mutations** — onboarding completion, plan upgrade, role
  grant, email-change, consent-acceptance. Any irreversible-by-design write
  that future tests assume is in the "before" state.

**Pattern: three-layer cleanup.** A single `afterEach` is not enough; a crash
during the body skips `afterEach`. Use defense-in-depth:

1. **`beforeAll` panic-sweep** — clean any stranded state from a previous
   crashed run BEFORE the suite begins. Idempotent (no-ops if state is
   already clean).
2. **Per-test `try/finally`** — wrap the mutation in `try { ... } finally
   { /* restore */ }` so a failed assertion still runs the restore step
   inside the same test scope.
3. **`afterAll` final sweep** — same panic-sweep as `beforeAll`, in case
   the `finally` block also failed (e.g. network error during the restore
   API call).

```typescript
test.beforeAll(async () => {
  // PANIC SWEEP — clean any stranded factor from a previous crashed run.
  // Idempotent: noop if no factor exists.
  if (api.isAdminAvailable()) {
    await api.deleteAllOwnMfaFactors().catch(() => {});
  }
  // ... route warm-up etc.
});

test("mentor enrolls in MFA", async ({ mentorPage }) => {
  try {
    // ... enroll, generate TOTP, verify, assert ...
  } finally {
    // Local cleanup — guaranteed to run even if an assertion above failed.
    await api.deleteAllOwnMfaFactors().catch((e) =>
      console.warn(`MFA cleanup in finally failed: ${e.message}`),
    );
  }
});

test.afterAll(async () => {
  // Final safety net — in case the per-test finally also failed.
  if (api.isAdminAvailable()) {
    await api.deleteAllOwnMfaFactors().catch(() => {});
  }
  await api.cleanup();
});
```

The cleanup helper itself must be **admin-privileged** (service-role,
admin API token, or similar) because the user's own session may be in a
broken state that prevents self-cleanup.

### Password / credential restore in `finally` — never optimistic

If the test changes a credential the auth helpers rely on (password,
email, MFA code), the credential **must be restored in `finally` regardless
of whether the test passed**. Do not "restore on success" — the test that
just failed is exactly the one that bricked the credential.

```typescript
test("mentor changes password via UI", async ({ mentorPage }) => {
  const userId = await api.getCurrentUserId();
  const ORIGINAL = process.env.TEST_MENTOR_PASSWORD!;
  const NEW = `temp-${RUN_ID}-${Date.now()}!`;

  try {
    // Fill old + new + confirm, submit, follow forced /login redirect,
    // verify re-login with NEW works.
  } finally {
    // ALWAYS restore via admin API — even on assertion failure above.
    // Loud console.error gives single-line RCA if restore itself fails.
    try {
      await api.adminUpdateUserPassword(userId, ORIGINAL);
    } catch (e: any) {
      console.error(
        `[CRITICAL] Password restore failed for ${userId}: ${e.message}. ` +
        `Test user is now locked out — restore manually with: ` +
        `supabase.auth.admin.updateUserById('${userId}', { password: '<orig>' })`,
      );
      throw e;
    }
  }
});
```

The loud `console.error` is intentional — if the restore fails, the next
20 tests in CI will all fail with cryptic auth errors. The message must
make the root cause obvious in one log line so the next dev doesn't
debug 401s.

### Service-role admin client — gate behind `isAdminAvailable()`

The cleanup patterns above require **service-role / admin API** access.
This is a strict superset of normal test auth (service-role keys can
update any user, bypass RLS, delete MFA factors). Handle the privilege
explicitly:

1. **Gate on env var presence.** Tests that need admin should check
   `isAdminAvailable()` and `test.skip` if absent, NOT crash with
   "service-role key is undefined." Local dev with `.env.test.local`
   works; CI without the secret skips gracefully.
2. **Lazy singleton, never persist.** Construct the admin client once
   per worker, with `persistSession: false` and `autoRefreshToken: false`
   — the admin client must never poison `storageState` or rotate
   refresh tokens.
3. **Keep admin methods on the API helper, not inline in specs.** Specs
   import `api.deleteAllOwnMfaFactors()`, not raw Supabase calls. This
   keeps the admin surface auditable in one file.

```typescript
// helpers/api.ts
import { createClient, type SupabaseClient } from "@supabase/supabase-js";

const SUPABASE_SERVICE_ROLE_KEY = process.env.SUPABASE_SERVICE_ROLE_KEY ?? "";

export function isAdminAvailable(): boolean {
  return Boolean(SUPABASE_SERVICE_ROLE_KEY && SUPABASE_URL);
}

let _adminClient: SupabaseClient | null = null;
function adminClient(): SupabaseClient {
  if (!isAdminAvailable()) {
    throw new Error(
      "SUPABASE_SERVICE_ROLE_KEY is not set. This test requires admin access; " +
      "guard with `test.skip(!api.isAdminAvailable(), 'admin required')`.",
    );
  }
  if (!_adminClient) {
    _adminClient = createClient(SUPABASE_URL, SUPABASE_SERVICE_ROLE_KEY, {
      auth: { persistSession: false, autoRefreshToken: false },
    });
  }
  return _adminClient;
}
```

For non-Supabase backends, the equivalent is an admin API token + a
dedicated `adminFetch()` helper with the same gate + singleton pattern.

### One-shot-per-user flows — fixme until per-run user seed exists

Some flows are **irreversible AND don't have an admin-API undo**:
onboarding wizards that flip `onboarding_completed=true`, email-verification
that consumes a token, "first impression" tutorials shown once per user,
ToS acceptance, KYC submission.

For these, you cannot write a real Tier-1 happy-path test against a shared
test user — the second run hits the post-completion state. Three options
in order of preference:

1. **Best**: seed a fresh user per test run (CI test-init script creates
   `e2e-onboarding-${RUN_ID}@test.example` with the pre-flow state). The
   spec uses that user; cleanup deletes it.
2. **Acceptable**: add an admin "reset onboarding" endpoint that the
   test calls in `beforeEach`/`afterEach`. Documents the gap in product
   and gets it closed.
3. **Last resort**: `test.fixme` the full-flow test with a comment naming
   the blocker (which option above is missing) and the owner. Cover what's
   *reachable* without consuming the one shot (validation, partial
   submission, navigation, error paths) in normal tests.

Do NOT solve this by writing a "happy path" test that only runs once,
then leaves the suite green-but-permanently-skipping. That's worse than
a documented `fixme`.

### Stateful metadata cleanup — the 409 trap

Endpoints that record "user has done X" by writing to `user_metadata` /
`raw_user_meta_data` / a state column will return 409 (or 400, or silent
no-op) on the second call. Common shapes:

- Signup intent (`signup_intent`, `onboarding_track`) set by a one-time
  picker.
- Tutorial dismissals.
- Feature-flag opt-ins recorded per-user.

If the spec needs to re-exercise the endpoint each run, **clear the
metadata in `beforeEach` or `afterEach` via admin API**. Treat the
metadata column like seeded data — owned by the test, cleared between
runs:

```typescript
async clearSignupIntent(userId: string): Promise<void> {
  const admin = adminClient();
  const { data: userData, error: getErr } =
    await admin.auth.admin.getUserById(userId);
  if (getErr || !userData?.user) throw new Error(getErr?.message ?? "no user");
  const meta: Record<string, unknown> = { ...(userData.user.user_metadata ?? {}) };
  delete meta.signup_intent;
  delete meta.onboarding_track;
  const { error } = await admin.auth.admin.updateUserById(userId, {
    user_metadata: meta,
  });
  if (error) throw new Error(`updateUserById failed: ${error.message}`);
}
```

Symptom that tells you you're hitting this trap: the first run passes,
the second returns 409/400, and the response body mentions
`already_set`, `idempotent`, `intent_locked`, or similar.

### External lib API — verify before importing

When the spec depends on an external lib for cryptographic / one-time
operations (TOTP/HOTP generation, JWT signing, OAuth code exchange,
password hashing), the lib's API shape changes across majors and
search-engine examples often reference an old version.

Before writing `import { foo } from 'lib'`, run the lib once at the CLI
or in a quick `node -e` script and confirm the **actual** exported names
and call signature on the version in `package.json`. Example pitfall:
`otplib` v12 exposes `authenticator.generate(secret)` (sync), v13 exposes
`generate({ secret })` (async, returns a Promise). Importing the v12
shape against a v13 install is a runtime crash that looks like "secret
too short" because `authenticator` is `undefined`.

This rule applies even when the lib is unchanged between majors — a
60-second sanity check saves a 20-minute debug.

### Wait for the success notification, not the button to re-enable

After a save/submit, the obvious "completion" signal looks like the submit
button becoming enabled again. This is wrong for any form whose button is
gated on a dirty/changed state (`disabled={!isDirty || isSaving}`,
`disabled={pristine}`, `disabled={!hasChanges}`, etc.) — after a clean save
the form clears the dirty flag and the button **stays disabled**. The wait
times out and the test fails AFTER the save actually succeeded.

Wait for the user-visible success signal instead — the toast, banner, or
inline success message that the app shows on completion:

```typescript
// ❌ WRONG — button stays disabled after a clean save
await saveBtn.click();
await expect(saveBtn).toBeEnabled({ timeout: 10_000 });

// ✅ RIGHT — wait for the same signal a real user reads
await saveBtn.click();
await expect(page.getByText("Saved successfully").first()).toBeVisible();
```

If you can't see a user-visible signal, fall back to `waitForResponse` on
the mutation's network call (see "Optimistic-update race" above) — never
guess at button state.

### Toast / live-region duplicates — strict-mode collisions

Most toast libraries render **two** DOM nodes per notification: the visible
toast `<div>` and an off-screen ARIA live region (`<span role="status">`,
`<div aria-live="polite">`) for screen readers. `getByText("Saved")` matches
both → Playwright strict mode throws "found 2 elements." This presents as a
flaky-looking failure that's actually deterministic.

Always scope toast assertions:

```typescript
// ❌ WRONG — strict-mode violation when both visible + live-region match
await expect(page.getByText("Saved successfully")).toBeVisible();

// ✅ RIGHT — pick one explicitly
await expect(page.getByText("Saved successfully").first()).toBeVisible();
// or filter to the visible toast:
// await expect(page.locator('[role="alert"]').getByText("Saved")).toBeVisible();
```

### Dual-render assertions — use `toHaveCount(0)` not `not.toBeVisible()` when the UI renders the same row in two slots

Responsive apps often render the same logical row twice — a desktop `<table>`
row visible at ≥`lg`, and a mobile `<Card>` visible below that breakpoint
(both kept in the DOM, toggled by Tailwind `hidden lg:flex` / `lg:hidden`).
Other dual-render patterns: separate "active" + "archived" lists rendered
side-by-side, master+detail panes, sidebar+main echoing the same item.

`not.toBeVisible()` after a delete asserts that **one** of the two nodes is
hidden — but the OTHER may still be visible from the unaffected slot. Worse,
optimistic-update libraries may hide one and leave the other, so the test
passes intermittently. The honest assertion is **count-anchored**: zero
matches across the entire DOM.

```typescript
// ❌ WRONG — only checks one slot; the other may still match
await expect(tasks.taskItemByTitle(title)).not.toBeVisible();

// ✅ RIGHT — count-anchored, counts every render of the row in the DOM
await expect(tasks.taskItemByTitle(title)).toHaveCount(0, { timeout: 15_000 });
```

Apply the same pattern to any assertion that says "this thing should be
gone": `toHaveCount(0)` rather than `not.toBeVisible()`. The locator should
match by content (seeded `RUN_ID + Date.now()` value) so cross-run pile-up
doesn't inflate the count.

If the project has a Page Object method like `taskItemByTitle(title)` that
already filters by content, callers don't need to think about slots — the
count check is correct by construction.

### Accordion / disclosure toggles — click the toggle test-id, not the title text

For accordions, dropdowns, expanders, tree nodes, or any "click to reveal
children" UI, never click the title text. Two failure modes:

1. The text node may not be inside the click target — the click silently
   no-ops and downstream actions (add child item, etc.) never persist.
2. The same text may render in multiple disclosure states — strict-mode
   ambiguity, or the wrong section opens.

Page Objects must expose toggle helpers scoped by identifier:

```typescript
// In the page object — stable test-id on the toggle button itself
moduleToggle(title: string) {
  return this.page.locator(`[data-testid^="module-row-"]`).filter({ hasText: title }).getByRole("button", { name: /expand|collapse/i });
}

// In the test
await courses.moduleToggle("Module A").click();
```

If the source doesn't expose a toggle test-id yet, add one — opaque text
clicks are a recurring source of "test passes locally, fails in CI" flake.

### Token-rotation hazard with shared `storageState` (Supabase, Auth.js, etc.)

If your stack issues short-lived access tokens with **refresh-token rotation**
(Supabase default), parallel Playwright workers reading the SAME storageState
file will eventually clash: one worker rotates the RT, the next worker reads
the now-invalid RT from the snapshot, the refresh endpoint returns 400, and
the app redirects to `/login?session_expired=true` mid-test.

Symptoms — almost always invisible without the trace:
- Tests pass locally (local Supabase doesn't rotate RTs by default).
- CI shows `waitForSelector` timeouts on selectors that DO exist on the
  target page.
- `error-context.md` shows the page snapshot is the LOGIN page.
- `0-trace.network` shows `/auth/v1/token?grant_type=refresh_token → 400`
  followed by a redirect to `/login`.

Mitigations (in order of preference):
1. Make `gotoAuthed` race three definitive auth signals — see "Auth-aware
   navigation" below.
2. Re-run `auth.setup.ts` in a fallback path when the storageState is
   detected as broken (UI re-login is fine if you can't update the file).
3. As a last resort: log in fresh per test (slow but deterministic).

### Auth-aware navigation (the `gotoAuthed` pattern)

Naive `await page.goto(path)` followed by a UI selector wait is fragile in
storageState-based suites. The auth gate may resolve, fail, or redirect
client-side AFTER `goto` returns — and `waitFor({ state: 'detached' })` on
a loading skeleton **resolves instantly when the element never enters the
DOM** (the exact case that fires when the app redirects to `/login` before
mounting the skeleton).

Use a helper that races three definitive signals:

```typescript
async function navigateAndDetectAuth(page: Page, path: string, timeoutMs: number) {
  let authMeStatus: number | null = null;
  const onResponse = (r: { url(): string; status(): number }) => {
    if (/\/auth\/me\b/.test(r.url())) authMeStatus = r.status();
  };
  page.on('response', onResponse);
  try {
    await page.goto(path, { waitUntil: 'domcontentloaded' });
    await Promise.race([
      page.waitForResponse(r => /\/auth\/me\b/.test(r.url()) && r.ok(),         { timeout: timeoutMs }),
      page.waitForResponse(r => /\/auth\/me\b/.test(r.url()) && r.status() === 401, { timeout: timeoutMs }),
      page.waitForURL(/\/login/, { timeout: timeoutMs }),
    ]).catch(() => {});

    if (page.url().includes('/login')) return /* needs fallback */ true;
    if (authMeStatus === 401)         return true;
    if (authMeStatus === 200) {
      // Late client redirects can still fire — give it a brief window.
      await page.waitForURL(/\/login/, { timeout: 1500 }).catch(() => {});
      return page.url().includes('/login');
    }
    return page.url().includes('/login');
  } finally {
    page.off('response', onResponse);
  }
}
```

Adapt the endpoint regex (`/auth/me`, `/me`, `/api/whoami`) and the redirect
target (`/login`, `/sign-in`, `/auth`) to your stack. The principle is
constant: **don't trust UI artifacts; trust HTTP status codes and URL
transitions.**

### SPA cache blindspot — API-seeded data is invisible to the UI

**This is the #1 source of false failures in SPA projects.** When you seed
data via API, the browser's client-side cache (RTK Query, React Query, SWR,
Zustand, Redux) does **not** know about it. If you navigate to a list page
and expect the seeded item to appear in the sidebar or table, it won't —
the cache serves stale data.

**Fix: navigate directly to the seeded record's URL** instead of browsing
to the list and clicking:

```typescript
// ❌ WRONG — seeded conversation won't appear in the sidebar cache
await page.goto('/dashboard/conversation');
await page.getByTestId(`list-item-${seeded.id}`).click();

// ✅ RIGHT — bypass the list, go straight to the detail page
await page.goto(`/dashboard/conversation/${seeded.id}`);
await page.getByTestId('message-input').waitFor({ state: 'visible' });
```

When the test IS about list rendering (e.g., "seeded item appears in the
list"), use `page.reload()` after seeding to force a fresh fetch, or
navigate away and back.

### Dev-server cold-start — warm up routes in `beforeAll`

Next.js (and other dev servers with on-demand compilation) compile each
route lazily on first request. The first hit can take **10–30 seconds**,
during which `page.goto` returns `ERR_ABORTED` and the browser context
gets destroyed mid-test. Subsequent runs hit the warm cache and pass —
which is exactly the kind of flakiness that wastes hours debugging "real"
issues that are actually just cold starts.

**Fix: warm up every route the suite will visit, in `test.beforeAll`,
before the first browser test runs:**

```typescript
test.beforeAll(async ({}, testInfo) => {
  testInfo.setTimeout(120_000);
  api = await TestApiClient.asMentor();

  // Warm up the dev server. First request triggers compilation; without
  // this, the first browser test hits ERR_ABORTED.
  await Promise.allSettled([
    fetch('http://localhost:3000/dashboard/tasks', {
      signal: AbortSignal.timeout(60_000),
    }),
    fetch('http://localhost:3000/dashboard/tasks/new', {
      signal: AbortSignal.timeout(60_000),
    }),
  ]);
});
```

Warm up **both** the list route and any detail/new routes — each is a
separate compilation. Use `Promise.allSettled` so a slow compile doesn't
fail the whole hook. Bump `testInfo.setTimeout` to 120s to give the warm-up
breathing room.

This is only needed against dev servers (local or deployed-as-dev). If the
test environment runs a production build, skip it.

### Console-error guard fixture

A test that passes against the real backend can still ship bugs the user
sees: failed RSC payload fetches, 401s on first paint, missing env-var
warnings, hydration mismatches. These all surface as `console.error` in
the browser, and Playwright ignores them by default. Add a guard once in
`fixtures/base.ts` and every spec inherits the protection:

```typescript
// fixtures/base.ts
const CONSOLE_WHITELIST: RegExp[] = [
  // Add project-specific known-benign patterns here. Keep narrow.
  // /Sentry DSN not configured/,  // example
];

export const test = base.extend({
  page: async ({ page }, use) => {
    const errors: string[] = [];
    page.on('console', (msg) => {
      if (msg.type() !== 'error') return;
      const text = msg.text();
      if (CONSOLE_WHITELIST.some((re) => re.test(text))) return;
      errors.push(text);
    });
    await use(page);
    if (errors.length) {
      throw new Error(
        `Unexpected console.error(s):\n${errors.map((e) => `  - ${e}`).join('\n')}`,
      );
    }
  },
});
```

Start strict (empty whitelist) and add entries only when a known-benign
error blocks a test. The whitelist itself is the audit trail — anything
in it should be linked to a tracked issue.

### Route-smoke spec — assert nav + app shell on every advertised route

When the app has a navigation menu with N advertised routes, write one
spec that iterates every route reachable from the nav, asserts HTTP 200,
and asserts the app shell is present (sidebar + top-bar selectors). This
catches the class of bug where a sidebar item points to a route that
returns the framework's raw 404 (no app chrome), leaving the user on a
dead end. Read the routes from the source-of-truth (router config, nav
component, or a `tests/e2e/.routes.ts` constant) — do NOT hardcode the
list in the spec or it drifts.

```typescript
// tests/e2e/route-smoke.spec.ts
import { test, expect } from "./fixtures/base";
import { MENTOR_ROUTES } from "./.routes"; // or import from source

const APP_SHELL = '[data-testid="app-sidebar"]'; // project-specific selector

for (const route of MENTOR_ROUTES) {
  test(`${route} renders with app shell (smoke)`, async ({ authenticatedPage: page }) => {
    const resp = await page.goto(route);
    expect(resp?.status(), `${route} returned ${resp?.status()}`).toBe(200);
    await expect(page.locator(APP_SHELL)).toBeVisible({ timeout: 10_000 });
  });
}
```

Per-role variants follow the same pattern — `FACILITATOR_ROUTES`,
`ORG_ADMIN_ROUTES`. Mark expected-403 routes as skipped or assert their
specific access-denied UI.

### Modal-title strict assertion — catch vocabulary drift

Every `click CTA → modal opens` flow should assert the modal heading
**exactly**, not just that *some* modal appeared. Vocabulary drift
between the CTA ("Start new Chat") and the modal title ("Add member to
Conversation") is invisible to `toBeVisible()` but very visible to
users. Expose this on the Page Object so callers can't forget:

```typescript
// pages/conversation.page.ts
async openNewConversationModal() {
  await this.startNewChatBtn.click();
  const dialog = this.page.getByRole('dialog');
  await expect(dialog).toBeVisible();
  // MANDATORY: assert the title exactly. CTA copy and modal copy MUST agree.
  await expect(dialog.getByRole('heading')).toHaveText('Start a new Conversation');
  return dialog;
}
```

If the CTA copy and modal title drift, the assertion fails and the
copy bug is caught before merge.

### Empty-state test — exercise the no-data branch

Every CRUD spec seeds data before navigating. The empty-state branch
of each list/feed/dashboard then never executes under test — empty-state
typos, missing primary CTAs, stuck loading spinners, and "0 of 0 with
page-1 button" bugs ship undetected. Add at least one `empty state` test
per spec that deliberately skips the seed:

```typescript
test.describe('Tasks — empty state', () => {
  test.beforeEach(async () => {
    // Make sure the test mentor has zero tasks
    await api.deleteAllTasksForCurrentUser();
  });

  test('empty state shows expected copy and primary CTA', async ({ authenticatedPage: page }) => {
    await page.goto('/dashboard/tasks');
    await expect(page.getByRole('heading', { name: /no tasks yet/i })).toBeVisible();
    // Strict CTA label — catches copy drift on the empty-state button
    await expect(page.getByRole('button', { name: 'Create your first task' })).toBeVisible();
    // Asserting no spurious pagination renders for 0 results
    await expect(page.getByRole('navigation', { name: /pagination/i })).toHaveCount(0);
  });
});
```

If the project doesn't have a "delete all" helper, fall back to a fresh
sub-account that owns no records — but the assertion shape is the same.

---

## Step 4: Create Page Object if needed

One class per page/section. Encapsulate selectors, expose action methods,
never put assertions inside.

### Navigation: use `domcontentloaded`, not `load`

Next.js and other SSR/SPA frameworks fire the `load` event only after all
JS bundles, images, and fonts finish downloading. On staging backends this
routinely takes 15–30 seconds and causes `page.goto` to timeout.

**Always use `waitUntil: 'domcontentloaded'`** in `navigateTo()` methods,
then wait for a specific element that proves the page rendered:

```typescript
async navigateTo() {
  await this.page.goto('/dashboard/clients', { waitUntil: 'domcontentloaded' });
  await this.container.waitFor({ state: 'visible', timeout: 30_000 });
}
```

When navigating to a detail page for API-seeded data, go directly to the
record URL (see "SPA cache blindspot" in Step 3):

```typescript
async navigateToRecord(id: string) {
  await this.page.goto(`/dashboard/clients/${id}`, { waitUntil: 'domcontentloaded' });
  await this.detail.waitFor({ state: 'visible', timeout: 30_000 });
}
```

### Page Object structure

```typescript
import { type Page, type Locator } from '@playwright/test';

export class ClientManagerPage {
  readonly page: Page;
  readonly container: Locator;
  readonly addBtn: Locator;
  readonly dialog: Locator;
  readonly dialogName: Locator;
  readonly dialogSubmit: Locator;

  constructor(page: Page) {
    this.page = page;
    this.container = page.getByTestId('client-manager');
    this.addBtn = page.getByTestId('client-manager-add-btn');
    this.dialog = page.getByTestId('client-dialog');
    this.dialogName = page.getByTestId('client-dialog-name');
    this.dialogSubmit = page.getByTestId('client-dialog-submit');
  }

  async navigateTo() {
    await this.page.goto('/dashboard/clients', { waitUntil: 'domcontentloaded' });
    await this.container.waitFor({ state: 'visible', timeout: 30_000 });
  }

  editBtn(id: string) {
    return this.page.getByTestId(`client-edit-${id}`);
  }
}
```

### List-item scoping — never expose raw `.first()` / `.last()` for per-item actions

When the page renders a list (messages, tasks, rows) and each item has its
own action button (Edit, Delete, kebab menu), **never** let a test do
`page.getByRole('button', { name: 'Delete' }).first()`. That selector
silently picks whichever item happens to render first — which may not be
the item the test seeded. The test "passes" by deleting the wrong record,
or fails inscrutably with "message still exists" because the backend deleted
something else entirely.

**This burned hours of debugging on at least one project. Read the page
snapshot in `test-results/<test>/error-context.md` when a delete/edit
"didn't work" — if there are multiple matching buttons, this is the bug.**

Page Objects must expose **identifier-scoped** methods, not raw locators:

```typescript
// ❌ WRONG — caller has to scope, easy to get wrong
get deleteBtn() { return this.page.getByRole('button', { name: 'Delete' }); }
// caller writes: po.deleteBtn.first().click() — picks the wrong row

// ✅ RIGHT — Page Object knows how to scope to one item
deleteBtnForTask(title: string) {
  return this.taskItemByTitle(title).getByRole('button', { name: 'Delete' });
}
taskItemByTitle(title: string) {
  return this.page.getByRole('listitem').filter({ hasText: title });
}
```

If the list has fixed ordering (oldest-first chat), `.last()` on the
seeded item is acceptable **only** when the test seeds the message
immediately before the action — and only with a comment explaining why.
Otherwise scope by content or test-id.

### Multi-step wizards — expose ONE Page Object method that drives the wizard to the confirm step

When a feature ships behind a multi-step modal / wizard (search → fill →
confirm, or step1 → step2 → step3), tests that copy-paste the step-by-step
plumbing diverge. After the third test inlines the same "fill search, click
next, fill names, click continue" sequence, one test will get the order
subtly wrong (forgot a `waitFor`, missed an intermediate field) and pass
intermittently. Worse, the happy-path and error-injection tests will each
stop at a different step, making the divergence invisible until a refactor
touches one but not the other.

**Page Objects should expose ONE method that drives the wizard up to (but
NOT including) the final mutation click**, so every caller stops at the same
checkpoint and decides what to do from there:

```typescript
/**
 * Drive the invite flow up to (but not including) the final Send click.
 *
 * Stops with `confirmBtn` visible so callers can decide what to do at the
 * final step — happy path clicks it; resilience tests Promise.all two
 * clicks (or pass to a throttle helper); error-injection tests register
 * page.route(...) FIRST and then click.
 */
async fillInviteToConfirmStep(email: string, firstName: string, lastName: string) {
  await this.openInviteModal();
  await this.searchInput.fill(email);
  await this.searchSubmitBtn.click();

  await this.firstNameInput.waitFor({ state: 'visible', timeout: 10_000 });
  await this.firstNameInput.fill(firstName);
  await this.lastNameInput.fill(lastName);

  // Sanity-check: the email entered in step 1 must carry through to step 2.
  // If this fails, the multi-step state-handoff is broken — louder than
  // any downstream "contact didn't appear" failure.
  const carriedEmail = await this.emailInput.inputValue();
  if (carriedEmail !== email) {
    throw new Error(`Wizard lost the email between steps: expected "${email}", got "${carriedEmail}"`);
  }
  await this.continueBtn.click();

  await this.confirmBtn.waitFor({ state: 'visible', timeout: 10_000 });
}
```

Three benefits:

1. **One source of truth.** When the wizard adds a step or renames a field,
   you fix the Page Object — not 6 specs.
2. **Internal invariants stay loud.** State-handoff bugs (step 1 email
   doesn't carry to step 2) throw inside the helper, not as a downstream
   "thing didn't appear" timeout that takes 30s to fail.
3. **Tests read like intent.** Happy path:
   `await contacts.fillInviteToConfirmStep(...); await contacts.confirmBtn.click();`
   Error injection: register the route, THEN call the helper, THEN click.
   Throttled race: helper, then `raceDoubleClickWithThrottle(...)`.

If the wizard has multiple branches (different paths from step 2 depending
on user choice), expose ONE method per branch rather than parameterising
with booleans — branchless helpers are easier to read and harder to misuse.

---

## Step 4a: Consult the tier-matrix (or equivalent) for gating

Many projects gate features behind subscription tiers, plans, feature flags,
or entitlements. The G dimension (gating) requires the spec to assert on
**both sides** of any gate the feature sits behind — or to explicitly
document that the feature is ungated.

**Before writing the test list**, look for a gating source-of-truth artifact
in the project's e2e directory:

- `tests/e2e/.tier-matrix.md` (or `.gating-matrix.md`, `feature-flags.md`,
  any similarly-named doc in the e2e folder).
- A README section listing which features require which plan/flag.
- An inline gate definition file referenced by the auth or feature-flag
  source code.

If the artifact exists, read the row for the feature you're testing and
record:

- The minimum tier/plan/flag required (or "ungated").
- The source-file-and-line reference, so the spec can cite which gate is
  load-bearing.
- The artifact's `Last synced` SHA, so the spec's BDD header `Gating:` slot
  can pin to a known version.

If the artifact does NOT exist, mark G as **unknown** in the BDD header and
recommend the user run the project's e2e-audit (or equivalent) to scaffold
one. Do not invent gates by reading the feature code — that's the audit's
job, not the spec's, and a per-spec PM-interrogation pattern doesn't scale.

For projects with no gating mechanism at all (rare — small internal tools),
mark G as "N/A — no gating mechanism in this project" and move on.

### What this means for the test list

- **Gated feature** → add 2 tests to the spec: one with an account ABOVE the
  gate (feature accessible, UI visible, API 200), one with an account BELOW
  the gate (feature inaccessible, UI hidden, API 403). The test accounts
  must already exist in the seed; if not, surface as a setup gap.
- **Ungated feature** → the BDD header `Gating:` line is sufficient. Do not
  add gating tests.
- **Dev-environment gate bypass** is common (many apps short-circuit gates
  in `NODE_ENV=development` to ease local dev). Verify whether the test env
  enforces gates; if it bypasses, the G assertion degrades to a permission-
  only check. Document this in the spec's `Risks:` slot.

---

## Step 5: Write the COMPLETE test set for the feature

### Required: BDD header at the top of every spec file

Every spec file MUST start with a JSDoc header that documents the feature,
the user goals, the surfaces touched, the API contracts hit, the risks to
watch when the feature changes, and the edge cases covered / NOT covered.
This is the single source of truth a future maintainer reads before editing
the spec — and the `Edge cases NOT covered:` line feeds the project-level
gaps digest (see Step 7).

Use this exact shape — fill every slot, never delete a slot:

```typescript
/**
 * Feature: <one-line — what user-facing capability this spec proves>
 *
 * Goal: <a complete sentence describing the user-visible outcome the test
 *   proves end-to-end. Should read like a product acceptance criterion.>
 *
 * Roles: <which auth contexts the tests run as, e.g. admin / member / guest>
 * Surfaces: <routes, modals, components touched>
 * API contracts: <endpoints + methods the tests exercise>
 * Gating: <one of:
 *   - "<plan/tier/flag> required — verified .tier-matrix.md @ <SHA>" (gated)
 *   - "N/A — no gating (verified .tier-matrix.md @ <SHA>)" (ungated)
 *   - "unknown — project has no tier-matrix artifact" (skip G dimension)>
 *
 * Risks (what to watch when this changes):
 *  - <risk 1 — something subtle that broke before or is easy to break>
 *  - <risk 2 — env / data / timing hazards specific to this feature>
 *
 * Edge cases covered: <terse list — one phrase per case>
 * Edge cases NOT covered: <terse list — gaps a future spec should fill>
 */
```

Rules:

- **Never copy another spec's header verbatim.** Risks and edge cases must
  reflect what THIS spec actually does. A boilerplate header is worse than
  no header — it lies to the next maintainer.
- **Risks are project-specific.** Include the non-obvious gotchas you hit
  while writing the spec: optimistic-update races, dual-slot rendering,
  cache-invalidation quirks, env-gated features, etc. If you found a
  universal one (applies to every project), don't just stuff it here —
  flag it for the skill itself.
- **`Edge cases NOT covered:` is the contract with the project's gaps
  digest.** Every phrase here will be lifted verbatim into the project's
  `.test-plan.md` § Edge-case gaps. Write them so they make sense out of
  context.

### How many tests per feature

**Aim for 3–7 tests per feature.** If you have more, merge them.

**Mandatory CRUD set when given a feature name.** Do not stop after one
operation. Generate every applicable test in this list before reporting:

1. **Create flow** — navigate, fill form, submit → item appears in UI →
   verify via API that record exists.
2. **Edit flow** — seed record via API → click edit → form pre-filled →
   change field → submit → verify via UI and API.
3. **Delete/archive flow** — seed record via API → click action → confirm
   → verify gone from UI → verify via API (404 or archived).
4. **Filter/search** — seed 2+ records via API → apply filter → correct
   rows visible/hidden.
5. **Error on save** — submit with genuinely invalid data (real server
   validation) → error shown, form stays open.

If the feature genuinely doesn't have one of these (e.g., a one-step wizard
with no edit), say so explicitly in the report — never silently skip.
**Never print "remaining" and stop.**

### Mandatory: cover all quality dimensions the project enforces

The 5 CRUD tests above only cover H (happy path) and partial E (one error).
Most mature projects enforce additional dimensions in their `.test-plan.md`
quality bar — concurrency, role-boundary, tier-gating, mobile viewport,
etc. **Before stopping**, read the project's quality-bar table and
generate one test per missing dimension. If the project has none, default to
the six below (the common cross-client baseline):

| Dim | Test stub (always emit; mark `test.fixme` if infra missing) |
| --- | --- |
| **H** persistence | After every mutation, `await page.reload()` and re-assert post-state. Catches optimistic-update-passes-but-DB-fails. |
| **E** server-error sub-classes | Beyond the validation test above, add at least one of: 403 permission (wrong-role API call), 500 infra (`page.route()` mock on the mutation only), 409 conflict (`If-Match` stale), 401 auth-expired (revoke token mid-flow). Target ≥3 of 5 sub-classes. |
| **M** mobile viewport | Wrap a `test.describe('@mobile')` block with `test.use({ viewport: { width: 390, height: 844 } })`. Inside: 1 mutation + 1 error path + 1 read. Not just a render check. |
| **R** roles | UI+ (allowed role works — already covered by main flow) AND UI− (`assertSurfaceHiddenFor(page, '<other-role>', '<path>')` or equivalent) AND API+ (raw API call as allowed role 2xx) AND cross-tenant (second seeded principal cannot see first's data via list endpoint). |
| **C** concurrency | Double-click the submit button — assert one record created, not two. PLUS session-expiry: revoke token mid-edit (via `context.clearCookies()` or storageState rewrite), expect graceful re-auth or explicit error. |
| **G** tier gating | If the feature appears in the project's tier-matrix as **gated**: write 2 tests — above-gate principal can access (UI + API 2xx); below-gate principal cannot (UI hidden + API 403). If **ungated**: BDD header `Gating:` line is sufficient, no extra tests needed. |

**Accessibility note**: a11y is intentionally NOT in the default dim set —
axe-core scans surface product-debt findings (sidebar contrast, form labels)
that are valid bugs but produce noise in the e2e suite and don't reflect
behavior changes. If a project explicitly wants a11y in its quality bar,
add it back as a separate dim in that project's `.test-plan.md` with a
clear scope (e.g., "axe critical+serious on /new pages only").

**When infra is missing** (no second mentor seeded,
no role for UI−, etc.):

- Emit the test as `test.fixme('...', async () => { /* TODO: <one-line reason> */ })`.
- Add a one-line entry to the project's `.test-plan.md` § Readiness or `.coverage-gaps.md` capturing what needs to happen to un-fixme it.
- Do NOT silently skip the dimension and do NOT pretend it's covered.

**Why fixme-stubs instead of just skipping**: a `test.fixme` shows up in
Playwright reports as a known gap, signals intent to the next maintainer,
and gets surfaced by `e2e-audit` so the dashboard reflects reality.

### What belongs in ONE test (not separate tests)

- Dialog opens when button is clicked → part of the create flow.
- Dialog closes on cancel → part of the create flow.
- Form is empty for new items → part of the create flow.
- Form is pre-filled for edits → part of the edit flow.
- Button is disabled when field is empty → part of the create/edit flow.

### What warrants its own test

- A distinct user path with meaningfully different scenario (error vs
  success, role A vs role B).
- A critical access control boundary (verify against real RLS if MCP is
  connected).

### Example: full create flow against real API

```typescript
import { test, expect } from '../fixtures/base';
import { ClientManagerPage } from '../pages/ClientManagerPage';
import { getAuthHeaders, deleteClient } from '../helpers/api';
import { request } from '@playwright/test';

const API_URL = process.env.TEST_API_URL!;

test.describe('Client Manager', () => {
  let createdIds: string[] = [];

  test.afterEach(async () => {
    const headers = await getAuthHeaders('admin');
    for (const id of createdIds) {
      await deleteClient(id, headers);
    }
    createdIds = [];
  });

  test('admin creates a new client and it persists', async ({ authenticatedPage: page }) => {
    const mgr = new ClientManagerPage(page);
    await mgr.navigateTo();

    await mgr.addBtn.click();
    await expect(mgr.dialog).toBeVisible();
    await expect(mgr.dialogName).toHaveValue('');

    const clientName = `E2E Test Client ${Date.now()}`;
    await mgr.dialogName.fill(clientName);
    await mgr.dialogSubmit.click();

    await expect(mgr.dialog).not.toBeVisible();
    await expect(page.getByText(clientName)).toBeVisible();

    // Verify via API — the record actually exists
    const headers = await getAuthHeaders('admin');
    const ctx = await request.newContext({ baseURL: API_URL });
    const res = await ctx.get(`/clients?search=${encodeURIComponent(clientName)}`, { headers });
    const clients = await res.json();
    expect(clients.length).toBeGreaterThanOrEqual(1);
    expect(clients[0].name).toBe(clientName);
    createdIds.push(clients[0].id);
  });

  test('submit with empty name shows server validation error', async ({ authenticatedPage: page }) => {
    const mgr = new ClientManagerPage(page);
    await mgr.navigateTo();

    await mgr.addBtn.click();
    await mgr.dialogSubmit.click();

    await expect(page.getByText(/name.*required|required.*name/i)).toBeVisible();
    await expect(mgr.dialog).toBeVisible();
  });
});
```

### Rules — non-negotiable

- **NEVER** use `page.route()` to mock happy-path API responses.
- **NEVER** write tests that would pass with the backend down.
- **NEVER** write a standalone test that only asserts `toBeVisible()` on a
  static element.
- **NEVER** use `waitForTimeout()` — use auto-retrying assertions.
- **NEVER** silence the project's anti-flake lint rule with an inline
  `eslint-disable` comment. If you find yourself writing
  `// eslint-disable-next-line no-restricted-syntax` to use `waitForTimeout`
  inside a `test.fixme(...)` body or anywhere else, the rule is firing for
  a reason: the test is encoding a fragile timing assumption. Replace with
  `expect.poll(...)`, `waitForResponse(...)`, or `waitForFunction(...)`.
  `test.fixme` does NOT execute the body, so a disabled lint rule "for the
  blocked path" is dead code AND a future trap: when the test is un-fixmed,
  the anti-flake rule is already bypassed and CI won't catch the bad wait.
- **NEVER** rely on `locator(...).waitFor({ state: 'detached' })` of a
  loading skeleton as a "page is ready" signal. It resolves INSTANTLY when
  the element never enters the DOM, masking redirects and crashes. Race
  on HTTP status codes and URL transitions instead.
- **NEVER** hardcode fake data objects — seed via real API calls.
- **NEVER** seed data with literal names you'll later search for. Use
  `RUN_ID + timestamp` so cross-run accumulation doesn't violate strict-
  mode locator queries.
- **NEVER** stop after one operation when given a feature name. Complete
  the CRUD set or explicitly say which operation doesn't apply.
- **ALWAYS** seed test data via real API calls against the test environment.
- **ALWAYS** clean up seeded data after tests (`afterEach`).
- **ALWAYS** verify mutations via a follow-up API call, not just UI
  assertions.
- **ALWAYS** hard-reload the page (`await page.reload()`) after every
  happy-path mutation and re-assert the post-state. Separates "optimistic
  cache updated" from "DB actually persisted" — a common class of regression
  where the UI passes a test but a fresh page load reveals stale data. Skip
  ONLY when the project's read path goes through a CDN/ISR/cache layer that
  invalidates on a delay (document the carve-out in the spec's `Risks:`).
- **ALWAYS** use real auth (test credentials against test env).
- **ALWAYS** verify `baseURL` is not a production URL before running.
- **Route interception ONLY** for simulating infrastructure failures —
  never for business logic.

### UI-passing ≠ feature-working — every mutation needs API verification

A test that clicks "Delete" and asserts the row disappears from the DOM
proves only that the UI rendered something. The optimistic update may
have rolled back silently; the DELETE request may have 404'd; RTK may
have invalidated the wrong cache key and the row is just hidden, not
gone. **The backend is the source of truth.** Always end mutation tests
with a follow-up API call:

```typescript
// Click delete + confirm
await deleteBtn.click();
await confirmBtn.click();

// UI assertion — necessary but NOT sufficient
await expect(messageArea.getByText(text)).not.toBeVisible();

// API assertion — the real proof
const messages = await api.getMessages(conversationId, { limit: 50 });
expect(messages.some(m => m.content.includes(text))).toBeFalsy();
```

For edits, fetch the record by id and assert the new field value. For
creates, list-and-find. For deletes, expect 404 or `status: 'cancelled'`.
A test without this final API check cannot distinguish a real bug from a
display glitch.

### Browser dialog handlers — register BEFORE the action that triggers them

Some components throw `alert()` or `confirm()` on mutation failure. If
you don't register a handler, Playwright auto-dismisses with a warning,
the test continues, and you lose the error message that would have told
you what's broken. Register the handler **before** clicking the action:

```typescript
mentorPage.on('dialog', (dialog) => {
  console.log('Dialog:', dialog.type(), dialog.message()); // capture for debug
  dialog.dismiss().catch(() => {});
});
await deleteBtn.click();
```

If a test is mysteriously hanging on a click, this is often why.

### Long suites — watch for token expiry

Supabase JWTs (and many auth providers) expire after **1 hour**. A suite
that takes 50+ minutes can have its API client tokens expire mid-run,
producing inscrutable 401s only on the last few tests. If your suite is
long enough to hit this:

- Re-fetch tokens in `beforeEach` rather than caching once in `beforeAll`.
- Or split the suite so no single `TestApiClient` instance lives past
  ~45 minutes.

### Selectors (in order of preference)

1. `getByTestId('...')` — most stable.
2. `getByRole('button', { name: '...' })`.
3. `getByLabel('...')`.
4. `getByText('...')` — for content verification only.
5. Never CSS class selectors.

### Missing data-testid attributes

If key elements lack `data-testid`, add them to the source components
before writing tests. Follow the project's existing convention (e.g.,
`<component>-<element>`). If no convention exists, use kebab-case:
`section-name-element`.

---

## Step 6: Run and verify — MUST PASS before proceeding

**This step is mandatory. Do not skip it. Do not defer it to "later."**

### 6a: Verify the full local stack is running BEFORE launching tests

E2E tests hit real services. A missing service causes auth-setup failures or mid-test 401s
that look like selector bugs. Before running any test, confirm every required process is up:

```bash
# 1. Frontend (Next.js)
curl -s -o /dev/null -w "%{http_code}" http://localhost:3000   # expect 200

# 2. Backend API server (if separate from frontend — check NEXT_PUBLIC_API_BASE_URL in .env.test/.env.test.local)
curl -s -o /dev/null -w "%{http_code}" <API_BASE_URL>/health   # expect 200 or 404 (reachable)

# 3. Database (Supabase local, Docker Postgres, etc.)
# For Supabase: check the project the test env points at, NOT just any running instance.
# .env.test.local OVERRIDES .env.test — always read both to know which Supabase URL is active.
SUPABASE_URL=$(node -e "
  const fs=require('fs');
  const parse=f=>{const m={};try{fs.readFileSync(f,'utf8').split('\n').forEach(l=>{const[k,...v]=l.split('=');if(k&&v.length)m[k.trim()]=v.join('=').trim()});}catch{}return m;};
  const e={...parse('.env.test'),...parse('.env.test.local')};
  console.log(e.NEXT_PUBLIC_SUPABASE_URL||'');
")
curl -s -o /dev/null -w "%{http_code}" "$SUPABASE_URL/rest/v1/"  # expect 200
```

**If any service is down:**
- **Frontend**: start it in background (`npm run dev > /tmp/dev.log 2>&1 &`), wait ~15s, re-check.
- **Backend API**: start from its own directory (`PORT=<N> npm run dev > /tmp/api.log 2>&1 &`), wait ~20s.
- **Supabase**: run `npx supabase start` from the **server repo** (not the frontend repo) — the server repo is the one whose supabase directory contains the seed files with test users. Check `.env.test.local` comments for the path.
  - After starting, validate credentials: `curl -X POST "$SUPABASE_URL/auth/v1/token?grant_type=password" -H "apikey: $ANON_KEY" -d '{"email":"<TEST_EMAIL>","password":"<TEST_PASS>"}'` — must return `access_token`.
- **Do NOT start the wrong Supabase** — the frontend repo may also have a `supabase/` directory, but that instance has no test users. The right instance is whichever one the test credentials authenticate against.

Wait for each service to respond before proceeding.

### Run the EXACT file you just wrote — not just the suite

After writing each spec file, run that file immediately:

```bash
npx playwright test tests/e2e/<spec>.spec.ts --reporter=line
```

**What counts as passing:**
- ✅ **Passed** — test ran and all assertions succeeded.
- ✅ **Skipped** due to a feature-gate probe (`test.skip(!featureAvailable, ...)`) — acceptable. The test is correct; the feature is unavailable on this env. Document the gate in `.test-plan.md`.
- ❌ **Failed** — fix before moving to Step 7. Read `test-results/<name>/error-context.md` first.

**Never skip this step.** Claiming tests were written is not the same as claiming they pass.
Claiming they "should work" without running them is dishonest. Run them.

### Diagnosing subscription / permission gates

If clicking a gated UI action does nothing (no dialog opens, no error, no toast), the
action is probably behind a permission check that evaluates to `false`. Probe via API before
writing UI assertions:

```typescript
// In beforeAll — check if the permission key is in the catalog
try {
  const perms = await api.getMyPermissions();
  featureAvailable = perms.permissions?.['permission.key.here'] === true;
} catch {
  featureAvailable = false;
}
// In each gated test:
test.skip(!featureAvailable, 'permission.key.here not available on this env — requires XYZ subscription tier');
```

Permission keys absent from `perms.permissions` are subscription-gated to a higher tier.
`grantAllPermissions` only grants keys in the catalog; it cannot unlock subscription tiers.

```bash
npx playwright test <test-file> --project=chromium --reporter=line --workers=1
```

Run the tests and **fix every failure** before moving to Step 7. Common
failure categories:

- **Selector not found** — element missing `data-testid`, or locator
  doesn't match the actual DOM. Read the error context snapshot to see
  what's on the page.
- **Timeout on `page.goto`** — use `waitUntil: 'domcontentloaded'` (see
  Step 4).
- **Seeded data not visible in UI** — SPA cache blindspot (see Step 3).
  Navigate directly to the record URL instead of clicking list items.
- **Post-reload assertion timeout** — increase timeout on assertions that
  follow `page.reload()` (API refetch takes time).
- **Auth setup failure** — environment/credential issue, not test code.
  Run with `--project=chromium` to skip re-running setup if auth state
  files already exist.

### Reporting a real product bug surfaced by a test

When a test correctly surfaces a real product bug (not a test issue), the
test stays in the suite — marked `test.fixme` so it documents the gap
without blocking CI. Three things must stay in sync so the bug doesn't get
forgotten:

1. **Mark the test `test.fixme`** with a one-line reason and a tracker
   link if one exists:
   ```ts
   test.fixme('mentor B cannot see mentor A tasks (BUG: <tracker-link-or-TBD>)', async () => { ... })
   ```
   If no tracker link yet, write `BUG: TBD — see .test-plan.md § Active
   blockers`. Future maintainers grep for `test.fixme` to find live gaps.

2. **Add a row in `.test-plan.md` § Active blockers** (or the equivalent
   "cross-cutting blockers" section). Columns: short summary | owner team
   | specific evidence (rule names, error messages, node counts — enough
   that the owning team can act without re-running the test). Reference
   the failing spec by name so the un-fixme PR is easy to write.

3. **Un-fixme when the fix lands**: remove the `.fixme`, drop the
   blocker row, and verify the test passes against the fix in the same
   PR. Don't leave dead `.fixme` markers behind.

If the project uses an external tracker (Linear, GitHub Issues, Jira),
add the issue link to both the `test.fixme` comment and the § Active
blockers row. If not, the § Active blockers row is itself the source of
truth — that's fine, just keep it actionable.

### Diagnosing CI-only flakes — always inspect the trace

When tests pass locally but fail in CI, **the error message and
`error-context.md` snapshot are nearly always misleading.** A
`waitForSelector` timeout reads as "selector wrong" but the real cause is
often hidden three layers deep (e.g. expired token → 401 → refresh 400 →
client redirect → test ends on a totally different page).

The Playwright trace zip has the truth. Pull it from CI artifacts and
inspect the network log:

```bash
gh run download <RUN_ID> --dir /tmp/artifacts
unzip /tmp/artifacts/test-results/<test-dir>-retry1/trace.zip -d /tmp/trace
# All URLs hit during the test, in order:
grep -oE '"url":"[^"]*"|"status":[0-9]+' /tmp/trace/0-trace.network
```

Things to scan for, in order:
1. **`/auth/me` or session-check endpoint status** — 401 means session
   went bad mid-test, even if the URL didn't redirect yet.
2. **Refresh-token endpoint status** — 400/401 means the storageState's
   refresh token was rejected (rotation conflict, TTL exceeded).
3. **Final URL** — `?session_expired=true` or any `/login` redirect means
   the user got bounced after `gotoAuthed` returned.
4. **Slow API responses** — sort by duration; if a specific endpoint took
   25s+, the page can't render its content in time.

Don't propose fixes from the error message alone. Get the trace.

### Read the page snapshot BEFORE forming theories

When a test fails, Playwright writes a DOM snapshot to
`test-results/<test-name>/error-context.md`. **Read this file first**,
before guessing at causes. It shows exactly what was on the page when the
assertion failed: visible alerts, error toasts, modals stuck open,
duplicate elements that broke a selector, or the absence of an element
you assumed was there.

Common signals in the snapshot:

- **Multiple matching elements** for a selector that should be unique →
  list-item scoping issue (see Step 4).
- **Alert/toast text visible** → the action failed server-side; the
  message tells you why.
- **A modal you didn't expect** → previous test leaked state, or the
  app is showing a confirmation you didn't account for.
- **The element you're waiting for genuinely isn't there** → the page
  state isn't what you think; check API responses in the trace.

"Selector timed out" plus a snapshot showing the element exactly where
expected = the selector itself is wrong, not a timing issue. Don't add
`waitFor` retries; fix the selector.

**Verify production safety**: before first run, confirm `baseURL` in
Playwright config points to the test environment, not production.

**You CANNOT proceed to Step 7 until the test run shows all tests
passing.** A report with untested code is not a report — it's a guess.

### Retry limit — ask, don't spin

You get **3 attempts** to fix failing tests. After 3 consecutive failing
runs, **STOP and ask the user for help.** Present:

1. The exact error message and page snapshot from the last failure.
2. What you tried and why it didn't work.
3. Your best guess at the root cause (environment, credentials, missing
   data, UI bug, etc.).

Do NOT keep trying variations silently. The user can unblock you in
seconds with context you don't have (e.g., "that account is disabled",
"the dev server needs a restart", "use this other contact ID"). Burning
20+ minutes on retry loops wastes time and context window.

---

## Step 7: Update project E2E docs IMMEDIATELY after each spec file

This step is **per-spec-file**, not per-run. The moment a spec file is
written and saved, update the project's E2E docs before moving to the next
one. This way an interrupted run leaves the docs accurate for the work
completed.

### Prescribed doc set (project-agnostic)

Every E2E suite, regardless of stack or client, should have:

| File | Purpose | Cadence |
|------|---------|---------|
| `tests/e2e/README.md` | Onboarding: how to run locally, env vars, conventions, project-specific gotchas, sibling repos (if any) | Stable |
| `tests/e2e/.test-plan.md` | Status + roadmap + edge-case digest + active blockers (sections below) | Per-PR |
| `tests/e2e/*.spec.ts` | BDD header at top of every spec (see Step 5) | With the spec |

History lives in `git log tests/e2e/`, not a separate `CHANGELOG.md`. A
quarterly `.coverage-matrix.md` is **not** prescribed — it duplicates the
spec inventory and digest and ages quickly. If a project still has those
files, they can be removed; their roles are absorbed into `.test-plan.md`
and the git log.

`.test-plan.md` MUST have these sections in order:

1. **Quality bar** — Definition-of-Done covering 6 dimensions per Tier 1
   spec (Happy + reload, Error with ≥3 server-error sub-classes, Mobile
   with mutation+error+read, Roles with UI+ / UI− / API+ / cross-tenant,
   Concurrency with double-click + session-expiry, Gating per tier-matrix
   or equivalent). Persistence is folded into Happy as a hard
   `page.reload()` after each mutation. Plus a readiness sub-table marking
   each suite-level requirement (CI, ESLint, branch protection,
   tier-matrix artifact) ✅/❌ with a one-line evidence pointer.
2. **Spec inventory** — table with single-line cells; columns:
   `File | Tests | Tier breakdown | H | E | M | R | C | G | CI | Notes`.
   Use `CI` (not `Status`) as the column name — `CI` = does the suite pass
   on main, which is a distinct signal from the 6-dimension quality matrix.
   A spec at all ❌ on dimensions can still be `CI ✅` because the few
   tests it has pass. Always include a one-line note below the table
   spelling out the distinction so readers don't conflate them. The
   headline metric below the table is **"X of N specs meet DoD"** — a
   per-spec count of all-6-dims-green, not a weighted average across
   dimensions (averaging across dims with different cost/value is
   misleading; `min()` is more honest but `all-6-✅ count` is what teams
   actually track).
3. **Edge-case gaps digest** — a single **table** with columns
   `Category | Gap | Action`, NOT nested bullet lists by category.
   Bullets-per-category becomes a wall of text past ~30 entries; a table
   stays scannable at 50+ rows and lets the reader filter visually by
   category. Every row's Action column is concrete: `extend X.spec.ts`,
   `§ What to write next #N`, `blocked → § Active blockers #N`, or
   `new cross-repo suite — needs <fixture>`. No vague "add coverage".
   Pull every entry from a spec's `Edge cases NOT covered:` line.
   **If a real user hits a bug, it should appear in this table.**
4. **What to write next** — two tables: Blocked (with Blocker + Owner +
   Next action columns) and Open (with Complexity + Owner). Resolved
   items are removed — `git log` is the history.
5. **Anti-patterns watch** — count + fix-status per pattern from Step 4
   of `/e2e-audit` (waitForTimeout, networkidle, page.route happy-path,
   etc.). When an ESLint rule lands and the count is 0, mark the row ✅.
6. **Active blockers (cross-cutting)** — anything not tied to a single
   spec: branch protection on `main`, server-side bugs the suite trips
   on, infra secrets pending. Each row needs an Owner.

### Cross-repo handoffs are the biggest hole when sibling repos exist

When the product spans multiple repos (web mentor + mobile client + LMS,
etc.), each repo's suite tests one side of every handoff. **No suite tests
both sides end-to-end.** That is where regressions ship undetected:
mentor sends a broadcast from web → client receives in mobile; mentor
publishes a course → student enrolls in LMS; mentor assigns a journal →
client submits in mobile. List these explicitly in § Edge-case gaps under
a "Cross-repo integration" subsection. Naming them is the first step to
owning them (even if the fix is a new test suite in a third location).

### Gotcha scope rule

- **Universal gotchas** (optimistic-update race, toast/live-region
  duplicates, accordion toggle, auth-aware navigation, token rotation,
  nested API shapes) → live in this skill. Apply to every client.
- **Project-specific gotchas** (component quirks, env-stack quirks,
  in-house API conventions) → live in the project's `tests/e2e/README.md`
  under a `## Project-specific gotchas` section, with a link back to this
  skill for universals.

Never copy a project-specific gotcha into the skill. Never duplicate a
universal gotcha into a project's README — link to the skill instead.

### Prioritize closing dimension gaps before adding new specs

Before writing a brand-new spec for a feature with zero coverage, check
`.test-plan.md` § Spec inventory for specs that are at 🟡/❌ on the Error,
Mobile, or Roles columns. **A bug in an under-covered dimension of an
already-covered feature affects users today. A missing spec for a new
feature only matters once someone uses that feature.** Default order:
close dimensions first, then add new specs. If the user explicitly asks
for a new spec, comply — but mention the dimension gaps in the report so
the prioritization is visible.

### Per-spec-file actions

This applies equally to **new** specs and **upgraded** specs (existing
spec gaining new dimensions). The skill is not done until the project's
test-plan dashboard reflects the spec's current state. Do NOT defer this
to a separate `/e2e-audit` run — that cadence-split leaves the docs lying
about coverage between PRs. If a project's test-plan doc exists, update it
in the same edit pass as the spec. If it doesn't exist, follow the
"When prescribed docs are missing" guidance below.

For each spec file just written or upgraded:

1. **Remove** from `.test-plan.md` § What to write next every item now
   covered by this spec (match by feature name + target filename).
2. **Add or update** the spec's row in `.test-plan.md` § Spec inventory
   with test count, T1/T2/T3 breakdown, AND the 6-dimension quality matrix
   (H/E/M/R/C/G — ✅/🟡/❌ per column, see § B legend in `.test-plan.md`).
   For upgrades, locate the existing row and rewrite the dimension cells
   to match what's now in the spec — do not leave a stale row.
   One-line cell, no prose. Be honest: a spec that only covers the happy
   path on desktop as the allowed role, with no reload + no concurrency
   + no gating, is `🟡 | ❌ | ❌ | ❌ | ❌ | ❌`, not all green. A spec with
   4 dims ✅ and 2 dims 🟡 (documented `test.fixme` with unblock recipe)
   is `✅ | ✅ | ✅ | 🟡 | ✅ | 🟡`, not all green.
3. **Update active-branch state** if the project's test-plan tracks
   in-flight branches (e.g. a § Active work in branch table). Add or
   refresh the row for this spec with the branch name, PR link, current
   per-dim score, and any `test.fixme` / `test.skip` blockers pending.
   Remove the row once the spec merges and the blockers are resolved or
   moved to § Active blockers.
3. **Append** the spec's `Edge cases NOT covered:` phrases (from its BDD
   header) to `.test-plan.md` § Edge-case gaps digest. Write them so they
   read out of context — they will be the next contributor's roadmap.
4. **Append** to `.test-plan.md` § What to write next every coverage gap
   you noticed while reading source or writing tests — adjacent flows,
   missing CRUD operations, related features. **Do this now, in this file
   edit, before moving to Step 8.** Do NOT defer to the Step 10 report —
   mentioning a gap in chat is NOT a substitute for writing it into
   `.test-plan.md`. The rationalization "I'll put it in the report" is a
   skip violation. Equally, "I'll run e2e-audit later" is a skip violation
   for the doc refresh — the dashboard must reflect the spec's new state
   before this skill is considered done.
5. **Recompute the per-dimension percentages + weighted score** in the
   Total row. Use ✅=1, 🟡=0.5, ❌=0; per-dim % = avg over specs;
   weighted score = avg of the per-dim averages. **Do not lead the
   headline with "all-dimensions X/N (0%)"** — that single binary metric
   reads as "we have nothing" even when H is at 100%. Report per-dimension
   + weighted so progress is visible incrementally and readers can see
   which dimensions are strong (don't slip) vs weak (close first).
6. **Do not reorder or rewrite** sections you didn't touch. Resolved
   items are deleted from § What to write next; history is `git log`.
7. **Owner columns are noise** when the suite has a single de-facto
   maintainer. Do NOT add an Owner column unless the team is multi-person
   AND ownership genuinely varies. Embed non-self ownership (e.g.
   "server team — open Linear ticket") inline in the item description.

### When prescribed docs are missing

If any of the prescribed files do NOT exist, do NOT create them mid-test-
generation — that's `/e2e-audit`'s job (it has the project-wide context).
Write the spec with its BDD header, then tell the user to run `/e2e-audit`
to scaffold the missing docs. Once scaffolded, future `/add-e2e-tests`
runs maintain them per the rules above.

---

## Step 8: Self-review

For each test, answer:

1. "What user goal does this test?" — If you cannot answer in one
   sentence, merge or rethink.
2. "Would this pass with the backend down?" — If yes, rewrite to hit real
   APIs.
3. "Does this clean up after itself?" — If no, add cleanup.
4. "Did I cover the complete CRUD set for this feature?" — If you skipped
   create, edit, delete, filter, or error, justify it explicitly or add
   the missing tests now.

Flag any test where the only assertion is `toBeVisible()` — it needs to be
part of a larger flow.

---

## Step 9: Stop local services you started — DO NOT leak processes

**This step is mandatory.** If you started any local servers, databases,
or containers during this run (frontend dev server, API dev server,
secondary services like LMS/payments/auth, Docker stacks, emulators,
local DBs), **stop them before reporting** unless the user explicitly
said to leave them running. Dev servers in particular leak memory over
hours of use (Next.js dev servers commonly hold 4–6 GB each); leaving
them running across sessions silently degrades the user's machine.

Track every service you start. Before writing the Step 10 report:

```bash
# 1. Find what's listening on the ports you used
lsof -i :3000 -i :3001 -i :3002 -nP 2>/dev/null | grep LISTEN

# 2. Kill Node/Next dev servers (use the PIDs from above)
kill <pid> <parent-pid>

# 3. Stop Supabase if you started it — restart is cheap (~30s, data preserved)
cd <project-server-dir> && npx supabase stop
```

**Decision rule:**
- **You started it AND no one else is using it** → stop it. This includes
  Supabase — `supabase stop` + `supabase start` takes ~30s and data is
  preserved in Docker volumes. Treat it like any other dev server.
- **It was already running when you arrived** → leave it.
- **You're unsure** → ask explicitly, e.g. "I started the API dev server
  on :3001 and Supabase — stop them now?"

If the user said "keep it running" / "I'll need it again," respect that.
Otherwise the default is **stop everything you started**. Mention in the
Step 10 report what you stopped and what you left running.

For tests that orchestrate services automatically via Playwright's
`webServer` config, no action is needed — Playwright tears those down.
This step is for services you started manually with `npm run dev`,
`docker compose up`, `supabase start`, etc.

---

## Step 10: Report

**Gate: All tests MUST be passing before writing this report.** If you
skipped Step 6, or the last run had failures you didn't fix, go back now.
A report that says "tests written" without a passing run is a lie — the
rationalization "the code looks correct" or "it's an environment issue"
is not acceptable. Run the tests. Fix failures. Then report.

Include in the report:
- Tests written + pass status
- `.test-plan.md` updates
- **Services stopped** (from Step 9) — list what was killed and what was
  left running, so the user knows the state of their machine.

Output:

- **Total tests written** and which user flows they cover (create, edit,
  delete, filter, error).
- **CRUD operations skipped** with the reason (e.g., "no edit — feature is
  immutable once created").
- **`data-testid` attributes added** to source.
- **Bugs found** (marked with `test.fixme()`).
- **Page Objects** created or reused.
- **API seed/cleanup helpers** created or reused.
- **Supabase MCP usage** — was it connected? If yes, which tables / edge
  functions / RLS policies informed the assertions. If no, note that
  configuring MCP would improve precision on the next pass.

### Missing coverage

Scan the source for adjacent features without E2E tests. Present each as
a ready-to-paste argument for `/add-e2e-tests`:

1. `Feature X — user creates a thing and sees it in the list` →
   `feature-x.spec.ts` (moderate)
2. `Feature Y — user filters by date range and only matching rows appear`
   → `feature-y.spec.ts` (simple)

**These items MUST already be in `.test-plan.md` before you write this
report** — Step 7 item 3 is where they get recorded. If you reach this
section and they are not in `.test-plan.md` yet, go back and add them now
before finishing the report. The report is for visibility only; the file
is the authoritative record.
