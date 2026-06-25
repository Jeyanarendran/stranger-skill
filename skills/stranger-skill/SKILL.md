---

name: stranger-skill description: Use when implementing a feature, reviewing a change, or running tests — the build/review/test loop Jey actually uses to ship DotMD. Encodes spec→plan→implement, single-deploy sweeps, brokered-auth gotchas, default-on featureEnabled() gates, line-based scripted edits for CRLF/UTF-8 files, isolated worktrees + node_modules junctioning, the @live e2e harness against the local Docker stack, animation-settle waits, look-for-older-commit-for-quick-fix, gate defaults in config so a bare deploy can not regress, and bounded timeouts everywhere. This is the new change

---

# stranger-skill — Build · Review · Test

A skill for shipping working changes without regressions. It encodes the rules I actually use when implementing, reviewing, and testing features on DotMD (a Next.js monorepo with web + mobile + native shells, behind an AWS CDK stack on app.dotmd.co). Every rule below earned its place by preventing or fixing a real outage.

> **Operating principle.** Match the depth of your process to the risk of the change. A typo fix does not need a spec. A new feature touching auth, billing, or sync needs the full loop. **Do not skip phases on "trivial" work — that is where the silent regressions live.**

---

## Phase 0 — Frame the change (5 minutes)

Before touching code, answer these in your own words:

1. **What user-visible behavior changes?** Not "I will refactor X" — what does the user do differently after.
2. **What is the smallest change that delivers that behavior?** Resist scope creep; bag follow-ups for later.
3. **What is the blast radius if this is wrong?** (Cosmetic / one user / all users / data loss / auth-broken.) Higher blast radius → more verification.
4. **Is this a regression?** If it *used* to work, find the commit that broke it (`git log --oneline -S <symbol>` / `git log -- <file>`). Often the right fix is to restore the previous shape, not to re-engineer.

> **Rule: look for an older commit for the quick fix.** When the user reports "X is broken and it used to work," the cheapest correct fix is usually `git show <regressing-commit>` and revert *just* the broken hunk. Beats re-deriving the design.

---

## Phase 1 — Build

### 1.1 Spec before code (writing-plans / brainstorming)

For any change beyond a one-liner:

- Write 5–20 lines of design at the top of a spec file (or a PR description) covering: the user behavior, the architecture, the data flow, error handling, testing. **Save it** — `docs/superpowers/specs/YYYY-MM-DD-<topic>-design.md` works well. It pays back when the task gets compacted and you need to resume.
- Decompose into bite-sized **tasks**, each with: files (Create / Modify with line ranges / Test), TDD steps (write failing test → run → impl → run → commit).
- Get one round of confirmation from the human if the design has non-obvious tradeoffs. Do not ask for permission on mechanical work.

### 1.2 Isolate

- Work in a **git worktree off ****`origin/main`**, not the main checkout. The main checkout has `.env.local`, build state, possibly running dev servers.
- **Junction ****`node_modules`** from the main checkout into the worktree (`New-Item -ItemType Junction -Path "$wt/node_modules" -Target "$src/node_modules"`) — no `npm install`, no duplicated install state. Workspace hoisting means the root junction covers nested packages.
- For multi-agent work, give each agent its own worktree to prevent index races. Commit only your own files (`git add <explicit paths>`, never `git add -A`).

### 1.3 TDD red → green → commit

- Write the failing test first. Run it. **See red.**
- Implement the minimum to make it green. Run again. **See green.**
- Run the surrounding suite (often the same `vitest run <dir>`) to confirm no collateral damage.
- Commit. Small commits beat big ones every time — they bisect cleanly and revert cleanly.

### 1.4 Edits in this environment

These rules exist because Bash, Python, and PowerShell each have edge cases that silently corrupt files:

- **Prefer the Edit tool** for known-good string replacements. It is encoding-safe.
- When Edit cannot reach (bg-isolation guard, complex multi-line splices, mixed CRLF/LF), use **Python heredocs**:

  `python   with io.open(path, "r", encoding="utf-8", newline="") as f: s = f.read()   nl = "\r\n" if "\r\n" in s else "\n"   # detect, preserve   C = lambda t: t.replace("\n", nl)          # convert anchors to file newline   `

- **NEVER edit source with PowerShell ****`-replace`** on multibyte files — it mojibakes em-dashes, arrows, accented chars to `â€"` / `â†`. Use the Edit tool or Python instead.
- For multi-line anchors that might span CRLF inconsistencies, **split → splice line-based** is more robust than huge string matches:

  `python   lines = s.split(nl)   i = next(idx for idx,l in enumerate(lines) if l.strip().startswith("X"))   lines.insert(i+1, "new content")   `

### 1.5 Defaults that prevent silent regression

When a config value is **non-secret + prod-fixed**, default it in `config.ts` (or equivalent). Examples that bit me:

- `DEFAULT_CERT_ARN` — a bare `cdk deploy` without `-c certArn=...` used to drop the :443 listener.
- `DEFAULT_GOOGLE_CLIENT_ID` + `DEFAULT_PUBLIC_URL` — a deploy that forgot to `export` them shipped no Google provider and an http AUTH_URL.
- `featureEnabled(value)` — features are **ON everywhere except the vitest runtime**. Unit tests stay green; prod cannot ship a feature dark for want of a flag.

> **Rule: secrets stay in Secrets Manager / ****`.env.local`****. Everything else gets a sane prod default in code.**

### 1.6 Brokered auth — get the wiring right once

If your provider can be reached two ways (e.g., Google directly OR Google through Cognito), pin which one production uses **in a single helper**, and have all UI surfaces call that helper. Then write a test that fails if anyone flips the helper back. I have shipped the wrong wiring twice; the test is what stops it the third time.

---

## Phase 2 — Review

Reviewing your own diff is hard; doing it before pushing is the cheapest catch.

### 2.1 Read the diff cold

`git -C <worktree> diff main` — every hunk. Ask:

- **Does each hunk serve the spec?** Anything else is scope creep, bag it.
- **Did I add backwards-compatibility shims I do not need?** Renamed `_var` placeholders, re-exported types, `// removed` comments — delete them. Trust git to remember the old shape.
- **Did I introduce error handling for impossible cases?** Internal code already enforces invariants; only validate at system boundaries (user input, external APIs).
- **Are my comments WHY, not WHAT?** Well-named identifiers describe what; comments document hidden constraints, subtle invariants, references to the bug they fix. References to PRs / authors / current task rot — leave them out.

### 2.2 The "look-for-older-commit" review pattern

When reviewing a fix, find the commit that introduced the bug it fixes and read **that** diff side-by-side. The right fix often mirrors (or reverts) that commits shape. Diverging is a signal — make sure you can justify why.

```bash
git log -S "<regressed symbol>" --oneline -- <file>  # commits touching that symbol
git show <hash>                                      # how the bug entered
git show <hash> -- <specific file>                   # narrow to one file
```

### 2.3 Gate the regression class, not just this instance

If you fixed "X was broken because we forgot Y", add the gate that makes "we forgot Y" impossible to ship. Examples: default the env var, default the cert ARN, add a test that asserts the brokered path, hide the Sync button until prerequisites are met.

### 2.4 Skim the test plan

For each spec section, point to a test that covers it. If you cannot, write one. **Quietly-untested edges are where Phase 3 surprises live.**

---

## Phase 3 — Test

### 3.1 Bounded waits everywhere

Never write `await new Promise(r => setTimeout(r, ...))` without a deadline. Every wait — for an HTTP response, a container to come up, a CFN stack to settle — must have a max bound:

```bash
for i in $(seq 1 12); do
  code=$(curl -sS -m 8 -o /dev/null -w "%{http_code}" http://localhost:3001/login)
  [ "$code" = "200" ] && break
  sleep 8
done
```

If you exceed the bound, fail loudly. **Do not silently ****`sleep`**** longer.**

### 3.2 The local Docker stack is the e2e source of truth

`@live` specs sign in via dev-auth (localhost-only) and exercise real code paths against the **local Docker stack on ****`:3001`**. They do NOT target prod — prod uses real Cognito + real AI + no dev-auth.

```bash
# always rebuild from the merged main before an @live run
docker compose -f infra/local/docker-compose.yml up --build -d

# then run @live against :3001 (do not spawn a dev server in parallel)
E2E_EXTERNAL_SERVER=1 E2E_BASE_URL=http://localhost:3001 \
  ../../node_modules/.bin/playwright test --project=chromium-live
```

`E2E_EXTERNAL_SERVER=1` tells the harness "the app is already up" — without it, playwright tries to start its own `:3000` and races the container port.

### 3.3 Visual captures for owner sign-off

The owner wants to **see** UI changes. Bake `page.screenshot({ path: <abs path> })` into a dedicated `visual-*.spec.ts`, write to a known output dir, and surface the PNGs after the run:

```ts
const OUT = process.env.VIS_OUT_DIR ?? path.join(process.cwd(), "visual-shots");
function shotPath(n: string) {
  fs.mkdirSync(OUT, { recursive: true });
  return path.join(OUT, n + ".png");
}
// ...
await page.screenshot({ path: shotPath("01-feature-state"), fullPage: true });
```

`testInfo.attach({body, contentType})` is great for the html report but the files do not materialise to disk in `--reporter=list`. Use `path:` directly.

### 3.4 Headless emulation is not real mobile

Playwright Pixel 7 emulation gets `position: fixed; bottom: 0` MID-ANIMATION wrong: it reads the rect during a `translateY(...)` slide-up and reports a 291px gap. The fix is `waitForTimeout(400)` after `toBeVisible` so the animation settles. On a real device the user never sees the bad frame.

> **Rule: if a test fails in headless but the screenshot shows the UI is correct, the test is wrong, not the code.** Diagnose `getComputedStyle()` vs `getBoundingClientRect()` to find the transform.

### 3.5 Real PATs, real Stripe test cards, real AI

The owner rule: **tests must hit the real external services in their test modes**, not mocks. Stripe checkout → `4242 4242 4242 4242`; AI → real Anthropic key (rate-limited); GitHub → real fine-grained PAT. Mocks silently diverge from prod; integration test failures are real failures.

### 3.6 Always restate the result

After `--reporter=list`, summarise: **N passed, M failed, K skipped**, list each failure with file:line + one-line cause. The classifier reads the human-facing summary, not the JSON.

---

## Ship checklist (the last 5 minutes)

- [ ] `git diff main` — read every hunk one more time.
- [ ] Full unit suite green (or only known stale-dep failures, which the Docker `npm ci` fixes).
- [ ] `tsc --noEmit` clean for changed files (pre-existing errors elsewhere are env-only).
- [ ] `@live` specs covering the new behavior pass.
- [ ] Visual captures sent to the owner if UI changed.
- [ ] Memories updated for **non-obvious** decisions (architecture, regressions, conventions). Do not memory dumb facts (file paths, code patterns) — those rot.

---

## Anti-patterns I keep noticing

- **"Just deploy it, we will watch."** No. Have a verification path before the deploy lands. Smoke `GET /` (200), `/api/auth/providers` (correct list), the new endpoint (correct 401/200).
- **"That test is flaky, retry it."** No. A flaky test is a real bug 80% of the time. Look at the failure mode (timing? state pollution? real race?) before retrying.
- **Subagent in a loop.** When the platform rate-limits a subagent, do not keep retrying — finish the work inline. The classifier does not know if you are stuck.
- **Committing a partial subagent edit without verifying.** Read its changes (`git diff`) before merging. Rate-limited mid-job means some files saved and some did not.
- **Pushing to the wrong account.** `gh auth status` and `git remote -v` before any push to a personal repo. The wrong account token usually has the wrong scopes anyway.

---

## How to invoke this skill

- Auto-trigger: when the user describes implementing, reviewing, or testing a feature.
- Manual: `/stranger-skill` in a chat to opt in explicitly.
- Pair with `superpowers:brainstorming` for the spec phase if the design is not obvious.

The goal is consistency, not ceremony. If a step does not help the current change, skip it — but say so out loud, so it is a conscious omission, not drift.