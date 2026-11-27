# AGENTS.md — /e2e

Read the root `AGENTS.md` first. This folder is the end-to-end suite: Playwright driving the real Docker stack. **It is the milestone testing gate** (root §9): a milestone is done only when its specs, and every earlier milestone's, pass.

## Running it

```
cd e2e
npm ci
npx playwright install chromium
npm test                 # builds + starts compose.yml, runs every spec, removes the stack
```

- `E2E_KEEP_STACK=1 npm test` leaves the stack running afterwards; `E2E_SKIP_STACK=1 npx playwright test …` reruns specs against it.
- The stack is its own compose project on its own port, so it never touches the dev stack or its data.
- `npm run seed` adds a demo admin with generic made-up routines, variations and a few months of workouts to click around in.

## How it works

- Same API and web images as the root `docker-compose.yml`, behind a test-only Caddy that loads the app's site block from `/deploy` (the same headers and CSP the host proxy serves), with its own local certificate. Tailscale isn't part of the stack. No auth shortcuts: every test creates a **fresh user** with the real `api user add` CLI and a generated password, and signs in through the real sign-in screen.
- Tests drive the UI and check results through the API. They arrange data through the API where the UI isn't what's being tested.
- **Two Playwright projects:** `desktop` (1280×800) for planning, history, analytics, import, export and settings, and `phone` (an iPhone-sized viewport with touch) for the workout screen and the phone routine flows. Logging specs run on `phone`.
- The browser runs in a time zone whose date differs from UTC's at the moment of the run, to catch `toISOString()` date bugs.
- One worker.

## When it runs

- **Not on every PR** (PR checks must stay under 2 minutes; root §9).
- At each **milestone gate**, on demand (`workflow_dispatch`), and on pushes to `main`.

## Gate coverage

Each milestone adds its acceptance tests here. Keep a test per gate item, named after it, and update this table when a gate changes.

| Gate | Item | Spec |
|---|---|---|
| M0 | Sign in, sign out, wrong password, expired session, forced password change | `auth.spec.ts` |
| M0 | Profile menu opens Settings and Sign out; Admin shows only for admins | `shell.spec.ts` |
| M0 | `/healthz`, HTTPS and HSTS on the stack | `smoke.spec.ts` |
| M1 | Create exercises (weighted, bodyweight, assisted, duration) with a primary and secondary muscle group and how-to notes; archive one with history, delete one without | `library.spec.ts` |
| M1 | Variations: a new exercise gets "Standard"; add, rename and archive variations; the exercise page groups variants by variation | `variations.spec.ts` |
| M1 | Muscle groups: deprecate one in use (hidden from pickers, kept on exercises), restore it, delete an unused one; delete refused when in use | `muscle-groups.spec.ts` |
| M1 | First sign-in: each split creates its routine groups; Skip creates none; the weekly goal is saved | `onboarding.spec.ts` |
| M1 | Change a variant's target load once; every routine using that variant shows it, and other variants of the exercise don't change | `targets.spec.ts` |
| M1 | An exercise used as 3 × 14 in one routine and 4 × 10 in another keeps two variants; the same scheme in two variations is two variants; changing a routine's scheme switches variant without touching the old one | `variants.spec.ts` |
| M1 | Straight, drop and superset targets are independent; one exercise is straight in one routine and a drop set in another | `targets.spec.ts` |
| M1 | A superset with a drop-set member, and a superset with the same exercise twice in different variations; drop targets with falling, rising and mixed steps are accepted; a one-step drop is rejected | `targets.spec.ts` |
| M1 | Routine groups: add, rename, reorder and delete a group (its routines move to No group); "+ Add" on a group prefills it; build, reorder, move, duplicate and delete routines; the variation-then-scheme picker (existing scheme, new scheme, Other variation); New exercise from the picker; Save disabled for an empty new routine | `routines.spec.ts` |
| M1 | Routine created and modified dates: modified changes on add, remove and reorder, not on rename or a target edit | `routines.spec.ts` |
| M1 | Settings: accent colour (presets and custom hex, kept after reload on the same device), weight step, keep-screen-awake; signed-in devices (sign out another session, sign out everywhere else) | `settings.spec.ts` |
| M2 | Log a full routine on `phone` with one-tap Done; the weight stepper follows the weight step setting | `logging.spec.ts` |
| M2 | Shorthand (straight and every drop form, including rising steps), a planned drop set, an unplanned drop, superset rounds with a drop-set member | `logging.spec.ts` |
| M2 | Skip, swap, add, Start Empty Workout | `deviations.spec.ts` |
| M2 | Target prompt (straight and drop): Update changes the target, Keep doesn't; the logged weights are kept either way | `target-prompts.spec.ts` |
| M2 | Reload mid-workout resumes it; a failed set save shows Retry | `logging.spec.ts` |
| M2 | Editing a target or routine afterwards leaves past workouts unchanged; logging doesn't change a routine's modified date | `history.spec.ts` |
| M2 | Today: Resume, the Suggested routine (oldest last done, never-done first), Start Empty Workout | `today.spec.ts` |
| M3 | Weekly reps and volume, muscle-group volume (primary group only), PRs and progression match a seeded fixture | `analytics.spec.ts` |
| M3 | Exercise page: every variant listed and grouped (by variation, drop, in supersets) with its own trend; overview across variants; comparing two variants | `exercise-page.spec.ts` |
| M3 | Weekly-goal and muscle-group streaks (primary group only), including a week below goal and the current week | `analytics.spec.ts` |
| M4 | Import from a Markdown table, `.xlsx` and `.csv` in the Name · Amount · Weight layout: per-set reps, summed bar weights, assistance weights, superset rows and notes shown in review; a file without dates asks for one; import and undo | `import.spec.ts` |
| M4 | Import similar names: Import blocked until answered; "Same" merges and saves an alias that matches on the next import; "Different" creates a new exercise | `import.spec.ts` |
| M4 | Export: CSV for a date range re-imports to the same sets; JSON contains every entity; exporting changes nothing | `export.spec.ts` |
| M4 | Admin adds a user and resets a password; members can't open Admin | `admin.spec.ts` |
| M5 | Security headers, body limits, rate limits | `security.spec.ts` |
| M5 | `api backup now` writes a snapshot; Status shows it | `backups.spec.ts` |
| M5 | Accessibility (axe, every preset accent, focus, 44/48px targets) | `a11y.spec.ts` |

### Manual checklist (each gate)

- Log a real workout on the owner's phone over Tailscale (from M2).
- Check charts look right with the default and a non-default accent (from M3).
- Compare the built screens with the design canvas (root §1) at desktop and phone width (each gate).
- Restore a backup to a fresh stack (M5).
- Open the app on its own tailnet hostname from the phone and a laptop, with a valid certificate (M5).

## Rules

- Generic fixtures only: no real names or real workout history.
- Generate test credentials in the test; never use a real account or password.
- Don't add `waitForTimeout`; wait on a locator, response or URL.
