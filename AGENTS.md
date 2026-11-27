# AGENTS.md — Plate Pusher

Repo: https://github.com/liljimmyk99/Plate-Pusher

Read this file first. It is the source of truth for requirements, use cases, and scope.
If a task conflicts with this file, stop and ask; if the task changes requirements, update this file in the same change.

## 1. Purpose

Plate Pusher is a personal workout planner and tracker. The user keeps a library of exercises, builds routines from them (including supersets), logs each session set by set on a phone at the gym, and reviews history and strength progression on a larger screen. It replaces workouts kept in Apple Notes, where:

- raising the weight for an exercise had to be edited by hand in every workout that used it;
- supersets had no structure, and their weights and reps differ from the same exercises done on their own;
- there were no totals or progression over time.

Delivered as a responsive web app backed by one self-hosted API. A native iOS app is a later phase.

**Design reference:** the shadcn/ui design canvas "Plate Pusher — shadcn design" (https://claude.ai/artifact/VzmAe1m1rihk6oiVY4hh4g). Desktop screens sit in the top row and their phone versions directly below. Where this file and the canvas disagree, this file wins; flag the difference.

## 2. Repo layout (monorepo)

```
/backend    Go API, SQLite migrations, Dockerfile
/web        React app
/api        OpenAPI spec — the contract all clients build against
/e2e        Playwright end-to-end suite on the Docker stack (milestone testing gates)
/docs       supporting notes (running locally, Tailscale, backups, security)
/deploy     the app's site block for the host reverse proxy, Tailscale Service config, backup notes
AGENTS.md   this file (each folder adds a short AGENTS.md with local rules)
docker-compose.yml
```

## 3. Stack

- Backend: Go, SQLite (WAL mode, single API container, DB file on a Docker volume)
- API: REST/JSON, described by OpenAPI in `/api`. Change the spec first, then backend, then clients.
- Web: React + TypeScript, built with Vite and tested with Vitest and Testing Library. UI components are **shadcn/ui** (copied into `web/src/components/ui/` and owned by us, on Tailwind CSS v4), themed with the **Forge** tokens (§10). Details in `web/AGENTS.md`.
- Auth: the app's own username and password sign-in, the same design as Penny Pincher (§10). The API checks the password (argon2id) and issues an opaque session token sent as a bearer token. No external identity provider.
- Deployment: Docker Compose (API and web) on the owner's Mac mini (Ubuntu Server), reached only over Tailscale. The app has its own tailnet hostname and certificate, and sits behind the Mac mini's shared host reverse proxy (§10).

## 4. Users and tenancy

- v1: a single user. Built for a handful of users later, each with fully private data. No sharing between users.
- Every table has `user_id`, and every query is scoped to the authenticated user. Build this in from day one.
- There is no sign-up. An admin creates accounts (the `api user add` CLI, and the admin panel's Users tab), setting a username and initial password, and can reset a password later. Either can require the person to choose their own at next sign-in. Everyone can change their own password. No email flow.
- Users have a role: `admin` or `member`. Only admins can open the admin panel.
- The network is private: the server is reached over Tailscale, with no port open on the router. Tailscale is only the network layer; the API never trusts its identity headers.
- **New accounts are seeded** with six muscle groups: Legs, Chest, Back, Shoulders, Arms, Cardio. Routines are not seeded; first sign-in asks which split the person follows (§6, use case 14).

## 5. Domain model

The app separates **the plan** (what you intend to lift, which changes over time) from **the log** (what you actually lifted, which never changes when the plan does). Mixing the two is what made Apple Notes fail.

**Units.** Weights are **integers in hundredths of a pound** (135 lb = `13500`, 2.5 lb = `250`). Durations are **integer seconds**. Never use floats for either. v1 is pounds only.

**Load type.** Each exercise has a `load` type that says what its weight means:
- `weighted` (default): the weight lifted ("135").
- `bodyweight`: **added weight**; `0` is "BW" and `2500` is "BW + 25" (dips, weighted push-ups).
- `assisted`: **assistance weight**, where **less is better**; `500` is "Assist 5" and `0` is "Unassisted" (assisted pull-ups and chin-ups, any machine that subtracts weight). Use this for pull-ups of any kind done on an assist machine or band.

Weights are never negative. The user's own body weight is never recorded.

The schema is defined in `backend/migrations/`. Those files are the reference for table and column names.

- **User**: id, username (case-insensitive, unique), password hash (argon2id, never returned by the API), optional display name, role (`admin` | `member`), `must_change_password` flag, weekly workout goal (default 3), `onboarded` flag
- **Session**: user, token hash, created at, last used at, and a short device label (from the user agent, e.g. "iPhone · Safari"). One row per sign-in. (An auth session; a workout is a **Workout**.)
- **MuscleGroup**: user, name (case-insensitive, unique per user), sort order, `deprecated` flag. Seeded with the six above. A deprecated group is hidden from pickers and filters for new data but stays on existing exercises, in history and in analytics, and can be restored.
- **Exercise**: user, name (case-insensitive, unique per user), **primary muscle group**, optional **secondary muscle groups**, tracking type (`weight_reps` | `duration`), **load type** (`weighted` | `bodyweight` | `assisted`), optional **how-to notes** (form cues), `archived` flag
  - **ExerciseMuscleGroup**: exercise, muscle group, `is_primary`. Exactly one primary. **Only the primary group counts** toward muscle-group volume and muscle-group streaks; secondary groups are for finding and filtering exercises.
  - **ExerciseAlias**: exercise, alias (case-insensitive, unique per user). Created when an import review answers "Same exercise" (use case 12), so later imports match that name automatically.
- **Variation** (a named way of doing an exercise, e.g. "Standing", "Incline bench", "Preacher"): exercise, name (case-insensitive, unique per exercise), sort order, `archived` flag. Every exercise has at least one; a new exercise gets one named **"Standard"**, which can be renamed. Variations group the exercise's variants on its page and are the first choice when picking the exercise for a routine, superset or workout.
- **Variant** (the plan for one way of doing an exercise): exercise, **variation**, context, style, **scheme**, and a **target load**. The scheme says *how* you do it; the target load says *how heavy*. Exactly **one variant per exercise, variation, context, style and scheme**. Every routine that uses that variant reads this one row, so changing its load once changes it everywhere. Doing 3 × 14 and 4 × 10 of the same exercise are two variants, each with its own weight and its own progression, and 3 × 10 Standing and 3 × 10 Incline are two variants as well.
  - **Context:** `isolated`, or a superset.
  - **Style and scheme:**
    - `straight`, `weight_reps`: scheme is sets × reps ("3 × 14"); target load is one weight ("@ 95").
    - `straight`, `duration`: scheme is sets ("3 sets"); target load is the duration ("60s").
    - `drop`: scheme is sets (usually 1) and the reps of each step ("10 → 8 → 6", at least two steps); target load is the weight of each step. `weight_reps` exercises only. **There is no direction rule**: weight and reps may go up, down or stay the same from step to step (a drop, a reverse pyramid or anything in between). Only "at least two steps" is enforced.
  - A variant is created the first time a routine, superset or workout uses a new scheme for a variation, asking for its target load. Its scheme never changes afterwards: changing "3 × 10" to "3 × 12" in a routine points that item at the 3 × 12 variant (creating it if needed) and leaves the 3 × 10 variant and its history as they were.
- **VariantStep**: variant, position, reps, target weight (drop style only)
- **Superset**: user, optional name, `archived` flag. A reusable, ordered group of **two or more** members, done back to back in rounds. Members can be any exercises, **including the same exercise more than once with different variations or variants** (e.g. Standing Curl + Incline Curl).
- **SupersetMember**: superset, position, and a variant whose context is this superset (any variation, straight or drop, any scheme). The same exercises with different variations, schemes or styles are a different superset.
- **Routine**: user, optional **group**, name, sort order within its group, optional notes, **created at**, **updated at** (e.g. "Legs A · Heavy"). `updated_at` changes only when the routine's items change: added, removed or reordered. Logging a workout never changes it, and neither do renaming the routine or editing a variant's target load.
- **RoutineGroup**: user, name (case-insensitive, unique per user), sort order (e.g. "Legs", "Push"). A folder for routines, usually one per split day. Deleting a group moves its routines to **No group**; it never deletes a routine.
- **RoutineItem**: routine, position, and **exactly one of** an isolated variant or a superset. Routine items hold no numbers of their own. The same exercise can be 3 × 14 in one routine, 4 × 10 in another and a drop set in a third.
- **Workout** (a logged session): user, optional routine, routine name at the time, date (`YYYY-MM-DD`, the user's local date), started at, finished at (null while in progress), optional notes, optional import
- **WorkoutEntry**: workout, position, an isolated variant or a superset, status (`done` | `skipped`), `swapped_from` (the planned variant, when swapped), `ad_hoc` flag (added during the session)
- **LoggedSet**: workout entry, variant, set number, round number (supersets), and one or more **segments**. Every logged set belongs to the variant it was done as, even when you did a different number of sets or reps than its scheme says; that is what lets history and trends be shown per variant.
- **SetSegment**: logged set, position, weight, and reps or duration seconds. A normal set has one segment. **A drop set is one set with several segments**, in whatever order the weights went.
- **Import**: user, file name, format (`markdown` | `xlsx`), status, counts, timestamps. Workouts it creates point at it so it can be undone.

Rules:
- **Logged sets are history.** Changing a target load, renaming an exercise or variation, or editing a routine never changes a logged set. Editing a past workout is allowed (to fix mistakes) and never changes a target load.
- **Every variant is separate.** Bumping 3 × 14 curls doesn't change 4 × 10 curls, incline curls, curls done as a drop set, or the curls inside a curl/pushdown superset, and the reverse.
- **Target prompts:** when a variant is completed in a workout and what was lifted differs from its target load, the app asks whether to update the target load. The answer only decides the target; **the logged weights are always kept**. Prompts not answered during the workout are listed on the finish summary. A prompt never changes a variant's scheme.
  - **Straight:** compares the working weight (the heaviest weight across the entry's sets, the least assistance for an assisted exercise, or the longest duration) with the target load; Update sets the target to it.
  - **Drop:** compares the first logged set's step weights with the target step weights; Update replaces the target weights.
- Any logged set can still have several segments, so an unplanned drop on a straight-set day is logged as it happened. It counts as a straight entry for prompts (its first segment is the working weight).
- **Exercises, variations, variants and supersets with logged sets can never be deleted**, only archived. Archived ones are hidden from pickers but kept in history and analytics. Unused ones can be deleted.
- **Muscle groups in use are deprecated, not deleted.** A muscle group used by any exercise can't be deleted, only deprecated (and restored). Unused groups can be deleted. Deprecating a group never changes an exercise, a logged set or past analytics.
- **Routines can be deleted** at any time; their workouts keep the routine name they were logged under.
- Workout changes (skip, swap, add, different numbers) never change the routine.

## 6. v1 use cases

1. Sign in / sign out with a username and password, change your own password, and the admin user flows (same as Penny Pincher; §4, §10).
2. **Exercise library:** add, edit, archive and delete exercises; set the primary and secondary muscle groups, tracking type, load type and how-to notes; add, rename, reorder and archive its variations. Each exercise has a detail page showing every variation and variant (use case 16). How-to notes show on the exercise page and under the exercise name while logging.
3. **Supersets:** create a superset from two or more members (any exercises, including the same exercise twice with different variations), reorder its members, and choose each member's variation and variant (style, scheme and target load). Existing supersets can be reused in one tap.
4. **Routines:** routines are listed **by group** (e.g. Legs → Legs A · Heavy, Legs B · Volume), then a **No group** section. Create, rename, reorder and delete groups; create, rename, reorder, duplicate, move between groups and delete routines; add, remove and reorder items (an exercise variant, or a superset). Each routine shows its name, item count, last done, created date and modified date. "+ Add" on a group creates a routine already in that group; the main "+" creates one with no group chosen yet.
   - **Adding an exercise asks two things:** which **variation** (e.g. Standing, Incline bench, or **Other**, which names and creates a new variation), then which **scheme** for that variation: one of its existing variants (straight or drop, showing the routines that use each) or a new scheme, which asks for its target load.
   - A new exercise can be created from the picker (name, muscle groups, tracking, load type, how-to notes, and its first scheme) and added in one step.
   - A new routine can start empty or as a copy of an existing one. It can't be saved without a name and at least one item.
5. **Shared targets:** change a variant's target load in one place (the exercise's page, or any routine that shows it) and see it in every routine that uses that variant.
6. **Log a workout on a phone, mid-session:**
   - Start from a routine. Each entry is prefilled from its variant's scheme and target load.
   - Mark a set done as planned in one tap, or change its weight and reps first.
   - Enter shorthand such as `3x10 @135` (or `3x10 BW`, `3x60s`), which expands into three sets, or `10@135 > 8@115 > 6@95` for one drop set.
   - A drop-set entry is prefilled with its steps and target weights. On any set, add a segment to log an unplanned drop.
   - Supersets are logged round by round, one set per member per round.
   - Every set saves to the server as it is logged; reloading the page or switching devices resumes the workout in progress.
   - Finish the workout (shows the summary and any unanswered target prompts) or discard it.
7. **Deviate mid-workout:** skip an entry, swap an exercise for another (logged against the substitute, with a "swapped from" note), add an exercise variant or superset that isn't in the routine, reorder what's left, or do a different number of sets.
8. **Start Empty Workout** with no routine and add exercises as you go. It is logged like any other workout (shown as "Empty workout" in history, counts toward goals, streaks, prompts and PRs) and never creates or changes a routine.
9. **Target prompts** as described in §5.
10. **History:** browse past workouts by week (each week showing its goal progress) with a date range and routine filter; open one to see every set; edit or delete it.
11. **Analytics** (desktop-first, still readable on a phone):
    - Total reps and volume per workout and per week. Volume is weight × reps summed over every segment; bodyweight sets add their reps to total reps and only added weight to volume; assisted sets add their reps to total reps and nothing to volume.
    - Strength progression per variant and per exercise over time (use case 16): working weight and estimated 1RM (Epley, `weight × (1 + reps / 30)`, only for sets of 1–12 reps with weight above 0).
    - Personal records per variant and per exercise: heaviest weight (least assistance for assisted exercises), most reps at a given weight, best estimated 1RM (not for assisted exercises). A new PR is called out on the workout's finish summary.
    - Weekly volume per muscle group, counting each exercise toward its **primary** group only.
    - Workout frequency and streaks. Both are counted in weeks, because most people don't train daily (three days a week is common):
      - **Weekly goal streak:** consecutive weeks (Monday to Sunday) with at least the user's weekly workout goal (default 3, set at first sign-in and in Settings). The current week counts once it reaches the goal; until then it doesn't break the streak.
      - **Muscle-group streaks:** for each muscle group, consecutive weeks in which it was trained at least once (primary group only).
      - Workouts per week over time, and days trained this week.
12. **Import** past workouts from a Markdown table or a spreadsheet (`.xlsx` or `.csv`), both with **Name · Amount · Weight** columns (format in §10). Upload, review, then import. An import can be undone. Review shows each row as it was parsed (sets, reps, weights, notes) next to the raw line.
    - Review shows exact matches, **similar names** (each asks "Same exercise" or "Different"), new exercises to create (with a muscle group, or matched to an existing exercise), and problem rows.
    - A name is **similar** when it matches an existing exercise or alias after ignoring case, spaces, hyphens and punctuation, expanding common abbreviations (DB→Dumbbell, BB→Barbell, KB→Kettlebell, OH→Overhead), or within a small edit distance. Import can't start until every similar name is answered.
    - "Same exercise" saves the file's name as an alias on the existing exercise, so later imports match it automatically. "Different" makes it a new exercise.
13. **Admin panel** (admins only): users (add, reset password) and status (version, uptime, database size, last backup).
14. **First sign-in: choose a split.** Before anything else, ask which split the person follows and create an empty **routine group** for each day of it, which they can rename, delete or fill with routines later. Also ask for the weekly workout goal.
    - **Body-part split:** Legs, Chest, Back, Shoulders, Arms
    - **Push / Pull / Legs:** Push, Pull, Legs
    - **Upper / Lower:** Upper, Lower
    - **Full body:** Full Body A, Full Body B
    - **Custom:** no groups; build your own
    The choice only seeds groups; it is not stored or enforced afterwards. Skipping it is the same as Custom.
15. **Settings:**
    - Weekly workout goal (1–7).
    - Accent colour: Orange (default), Blue, Green, Red, or a custom hex.
    - Logging: weight step for the − / + controls (1, 2.5, 5 or 10 lb; default 5), and keep the screen awake during a workout (on by default).
    - Muscle groups: add, rename, reorder, deprecate and restore; delete unused ones.
    - Signed-in devices: list your sessions (device label, signed in, last active), sign one out, or sign out everywhere else.
    - Change password.
    - Export data (use case 17).
    Accent, weight step and screen-awake are per-device preferences kept in the browser; everything else is stored on the server.
16. **Exercise page: every variation, every variant and its trend.** Opening an exercise shows every way it is done:
    - **How-to notes** at the top, editable.
    - **Overview** across all variants: best estimated 1RM over time (comparable across rep schemes and variations), weekly reps and volume for this exercise, PRs, and the last time it was done.
    - **Variants**, grouped as **Isolated**, then by **variation** (e.g. Standing, Incline bench, Preacher), each holding any number of straight and drop variants, and **In supersets** (each superset with its other members named, e.g. "with Tricep Pushdown"). Each variation has Rename and an Add variant tile; Add variation creates a new one. Each variant shows its scheme and target load, the routines that use it, when it was last done, its best, and a small trend line.
    - Opening a variant shows its **progression** (working weight per workout, or each step's weight for a drop set, or duration, with the target load over time as a step line, plus estimated 1RM where it applies) and its **history** (every workout it was done in, with every set).
    - Variants with no history yet are listed as planned. Archived variants and variations are hidden behind a filter.
    - The target load of any variant can be edited here.
17. **Export data** from Settings:
    - **Workout log · CSV:** one row per workout entry in the import columns (**Date · Routine · Name · Amount · Weight · Notes**), for a chosen date range, so it opens in a spreadsheet and imports back in. The JSON backup is the lossless copy; the CSV writes each set's reps and weights in the Amount/Weight grammar.
    - **Full backup · JSON:** muscle groups, exercises (with variations, variants, targets and aliases), supersets, routines and every workout in one file.
    - Exports never change data. Settings shows when the last export was made.
18. **Today (landing):** the workout in progress (Resume), this week (goal progress, days trained, streak, volume and reps), Start from a routine, Start Empty Workout, and the last workout. The routine list keeps the user's own order, with one routine marked **Suggested**: the one in the list that has gone longest without being done (never-done routines first).

## 7. Scope

**In v1:** everything in section 6; a responsive web app; online-only; Dockerized backend on the Mac mini over Tailscale.

**Out of v1 (do not build):**
- Native iOS (or other mobile) app
- Calendar scheduling or reminders ("Legs on Monday")
- Distance, pace, heart rate, wearables or Apple Health sync. (Cardio is a muscle group, and cardio exercises use `duration` tracking; nothing more.)
- Rest timers
- Kilograms and unit switching
- Recording body weight or body measurements
- Automatic progression rules (e.g. "+5 lb when all reps hit")
- Updating a routine from a workout's deviations
- Sharing, social features, coaching, exercise media or instructions beyond the user's own how-to notes
- Offline mode (planned later: cache the workout in progress and sync)
- Account management beyond adding users, resetting passwords and signing out sessions; MFA

## 8. Non-functional requirements

- **Phone-friendly logging, desktop-first everything else.** The workout screen works one-handed on a phone: large targets (at least 48px for set controls), no hover-only actions, numeric keyboards for numbers, and the screen kept awake during a workout where the browser supports it (Wake Lock API, unless turned off in Settings). Planning, history and analytics are designed for a laptop and still stack sensibly at phone width.
- **Logging is fast and never loses a set.** Each set is saved immediately; a failed save is shown on that set with a retry, never silently dropped.
- HTTPS only, on the app's own tailnet hostname and certificate, with HSTS (§10).
- Token verification on every API request; all data access scoped by `user_id`.
- Passwords are stored only as argon2id hashes and never logged. Expose `/healthz`.
- Nightly local backups of the SQLite database (§10). Restore must be tested.
- Secrets in environment files, never committed.
- Logs never contain passwords, tokens or workout contents (including import and export file contents).
- Security hardening before first deployment, following Penny Pincher's `docs/security.md`: security headers and CSP at the proxy, body size caps, rate limits, least-privilege containers, dependency and image scanning in CI.
- Performance is not a concern at this scale; favor simplicity.

## 9. Rules for agents

- Do not add features outside section 6. Propose scope changes instead of implementing them.
- Keep the OpenAPI spec, backend and web consistent; update the spec first.
- Weights are integer hundredths of a pound and durations integer seconds everywhere, including in clients, imports and exports. Never floats.
- Never write to logged sets from variant, variation, routine, exercise or muscle-group changes.
- Every schema change is a new migration. Never edit an applied migration.
- Do not hard-code personal data (names, real weights, real workout history) in code, seeds or tests; use generic fixtures.
- When requirements are ambiguous, ask rather than guess, then record the answer here.
- A change that adds, removes or weakens a security measure updates `docs/security.md` in the same change.

### Testing and milestones

- **Every PR** runs the fast checks: backend `go vet` and `go test`, web lint, type check and Vitest, spec lint, and generated-code freshness. **The whole PR check must finish in under 2 minutes.** If it creeps past that, move the slow tests to the milestone gate instead of letting PR checks slow down.
- **Every milestone ends with a testing gate:** the Playwright suite for that milestone (and every earlier one) on the Docker stack, plus the manual checklist in `e2e/AGENTS.md`. **A milestone is not done until its gate passes, and work on the next milestone does not start before then.**
- Each milestone's acceptance tests are written in that milestone, mapped in the gate table in `e2e/AGENTS.md`.

### Milestones

| Milestone | Delivers | Use cases |
|---|---|---|
| **M0 · Foundation** | Monorepo, Docker Compose, CI, migrations runner, auth (users, sessions, `api user` CLI), sign-in and change-password screens, app shell with the profile menu, Forge theme tokens | 1 (sign-in parts) |
| **M1 · Library & plans** | Muscle groups (primary/secondary, deprecate), exercises with how-to notes, variations, variants (straight and drop), supersets, routine groups and routines (dates, duplicate, the variation-then-scheme picker), first sign-in split choice, settings (goal, accent, logging, muscle groups, devices), the exercise page's variant list | 2, 3, 4, 5, 14, 15, 16 (list and targets), 18 (without stats) |
| **M2 · Logging** | Workouts, sets, drop sets, supersets in rounds, deviations, empty workouts, target prompts, resume, history | 6, 7, 8, 9, 10, 18 |
| **M3 · Analytics** | Reps and volume, progression per variant and per exercise, PRs, muscle-group volume, frequency and streaks | 11, 16 (trends) |
| **M4 · Import, export & admin** | Markdown table and spreadsheet (.xlsx, .csv) import with similar-name review, aliases and undo; CSV and JSON export; admin panel | 12, 13, 15 (export), 17 |
| **M5 · Hardening & deploy** | Security hardening, nightly backups, deploy behind the Mac mini's host proxy on the app's own tailnet hostname, restore test | §8 |

## 10. Decisions and open decisions

### Decided

- **Name: Plate Pusher** (decided 2026-11-26). Use it for the repo, page titles, the Compose project and the tailnet service name (`svc:plate-pusher`).
- **Auth: same as Penny Pincher** (decided 2026-11-26). Copy its design, not a variation of it:
  - Username and password; **argon2id** (`golang.org/x/crypto/argon2`, pure Go), minimum 12 characters (`400 weak_password`), concurrent hashing capped.
  - `POST /v1/auth/login` returns an opaque 256-bit token; only its SHA-256 is stored in `sessions`. Clients send it as a bearer token. Sessions last at most **30 days from sign-in** (`401 session_expired`) and are revoked on sign-out, password change or reset, or from Settings → Signed-in devices.
  - Login protection: rate limited per IP and per username, 15-minute lock-out after 10 failures in a row, one generic `401 invalid_credentials`, constant-time comparison, a dummy hash for unknown usernames.
  - First admin via `api user add --role admin`; `api user set-password` resets from the command line.
  - **Tailscale is the network layer only.** The API never trusts `Tailscale-User-*` headers; Funnel stays off.
  - MFA, idle sign-out, email and self-service reset are out of v1.
- **SQLite** (decided 2026-11-26): WAL mode, `modernc.org/sqlite` (pure Go, no CGO), per-connection pragmas in the DSN (`foreign_keys(1)`, `busy_timeout(5000)`, `synchronous(NORMAL)`, `journal_mode(WAL)`), a small in-house migration runner on `embed.FS` that runs on API startup and as a `migrate` subcommand (same as Penny Pincher). Keep SQL plain so a later move to Postgres stays possible.
- **Web UI: shadcn/ui** on Tailwind CSS v4, `new-york` style, components owned in `web/src/components/ui/` (same as Penny Pincher).
- **Visual design: "Forge"** (decided 2026-11-26), per the design canvas (§1): a near-black, blue-tinted dark theme; **orange is the action colour** (main buttons, Done, progress, streaks, PRs) and **blue is information** (targets, the target prompt, charts, links, focus rings). Headings Space Grotesk, body Inter, shorthand and numbers in code JetBrains Mono. Tokens are in `web/AGENTS.md`. **Dark only:** there is no light theme and no theme setting (decided 2026-11-26).
- **Navigation** (decided 2026-11-26): the sidebar holds Today, Routines, Exercises, History, Analytics and Import. **Settings and Admin live in the profile menu** at the bottom left (avatar, display name, role), with Sign out.
- **Accent colour** (decided 2026-11-26): Orange (default), Blue, Green, Red or a custom hex, saved **per device in localStorage**. It drives `--primary` and `--chart-1`; button text switches to black or white for contrast. Red moves destructive actions to an outlined style, and Blue moves the info colour to teal, so roles stay distinct.
- **Charts are hand-built** React/SVG components on the design tokens, no charting library (same as Penny Pincher). Revisit only if line charts prove painful, and record the change here.
- **Plan vs log** (decided 2026-11-26): one variant per exercise, variation, context, style and scheme, each with its own target load; logs never change retroactively (§5).
- **Variations** (decided 2026-11-26): each exercise has named variations (Standing, Incline bench, …), each holding any number of straight and drop variants with their own targets and history. They apply everywhere a variant is chosen, including superset members. Choosing an exercise asks for the variation first, then the scheme.
- **Drop sets can be planned** (decided 2026-11-26): a routine item is an exercise as straight sets ("3 × 10 @ Y") or as a drop set ("10 @ Z → 8 @ A → 6 @ B"). Each style is its own variant, with its own shared target load and update prompt. Superset members can be drop sets too.
- **Drop steps have no direction rule** (decided 2026-11-26, replacing "weight never goes up"): at least two steps; weight and reps may each rise, fall or stay the same.
- **Supersets** (decided 2026-11-26): any two or more members, including the same exercise twice with different variations or variants.
- **Muscle groups** (decided 2026-11-26): an exercise has one primary and optional secondary groups; only the primary counts for volume and streaks. Cardio is a seeded group. Groups in use are deprecated, not deleted.
- **Routine dates** (decided 2026-11-26): `created_at` and `updated_at`; `updated_at` changes only when items are added, removed or reordered.
- **Today's Suggested routine** (decided 2026-11-26): keep the user's order and mark the one gone longest without being done as Suggested.
- **Export** (decided 2026-11-26): CSV workout log in the import template's columns, plus a JSON full backup; both in M4.
- **Import similar names and aliases** (decided 2026-11-26): similar names are a review question; "Same exercise" saves an alias (§6, use case 12).
- **Split chosen at first sign-in** (decided 2026-11-26): nothing is seeded by default; the person picks a split (§6, use case 14) and gets an empty routine group for each day of it.
- **Routine groups** (decided 2026-11-26): routines live in optional groups (a folder per split day, holding several routines such as Legs A · Heavy and Legs B · Volume). Groups, not versions: each routine stays separate and is started directly.
- **Streaks are weekly** (decided 2026-11-26): a weekly-goal streak and per-muscle-group weekly streaks, since a muscle group is trained once or twice a week and many people train three days a week. No daily streak.
- **Hosting on the Mac mini** (decided 2026-11-26):
  - A **shared reverse proxy on the host** (planned for all of the Mac mini's apps) fronts this app. The app's Compose stack runs no proxy of its own; the web and API containers listen only on `127.0.0.1` ports.
  - The app gets **its own tailnet hostname and certificate**, separate from Penny Pincher's. Note that `tailscale cert` only issues a certificate for the machine's own name, so a second name needs one of:
    - **Tailscale Services** (preferred): `tailscale serve --service=svc:<app> --https=443 127.0.0.1:<port>` gives the app `<app>.<tailnet>.ts.net` with a certificate Tailscale provisions. It needs Tailscale 1.86 or later, a tag-based identity on the Mac mini, and admin approval of the service host.
    - A **Tailscale sidecar** container in the app's Compose stack, which joins the tailnet as its own device with its own name and certificate.
  - The host proxy still adds the security headers and CSP (§8). How it sits with Tailscale's TLS (behind it, or receiving the TLS) is settled with a short spike at the start of M5 and recorded here.
- **Backups are light** (decided 2026-11-26): the API takes a nightly `VACUUM INTO` snapshot into a mounted backup folder and keeps the last 14. No encryption, cloud bucket or admin-panel backup controls in v1. Copying that folder off the Mac mini is a host task, documented in `docs/backups.md`. The admin panel's Status shows the last backup's time and size. Restore (stop the API, swap in a snapshot, start) is tested in M5.
- **Superset variants are separate** from isolated variants, and get the same update prompt (decided 2026-11-26).
- **Variants by scheme** (decided 2026-11-26): each rep scheme of an exercise (3 × 14, 4 × 10, …) is its own variant with its own target load and trend, and the exercise page shows them all (use case 16). Logged sets belong to the variant they were done as.
- **Load types** (decided 2026-11-26): `weighted`, `bodyweight` (added weight) and `assisted` (assistance weight, less is better); weights are never negative; body weight is never recorded.
- **Import format** (decided 2026-11-26, from the owner's notes): the same columns whether the file is a **Markdown table** (`.md`) or a **spreadsheet** (`.xlsx` or `.csv`, first sheet).
  - Required columns **Name · Amount · Weight** (header names case-insensitive). Optional columns **Date**, **Routine** and **Notes**.
  - **Workouts:** rows with the same Date (and Routine) are one workout, in row order. Without a Date column, the whole file is one workout and the review screen asks for its date (and optional routine name). In a Markdown file, a heading line with a date (`# 2025-03-14 Pull`) may also start each workout instead of a Date column. Dates are `YYYY-MM-DD` or `M/D/YYYY`.
  - The downloadable template is a spreadsheet with these six columns and two example rows.
  - **Supersets** (provisional, see Open): a ` + ` in the Name (outside brackets) makes the row a superset, one member per part ("Bicep Curl + Tricep Pushdown"). Amount and Weight then hold one value per member separated by ` ; ` ("3x12 ; 3x10", "30 ; 60"), or a single value that applies to every member. Inside one member's Weight, `+` still adds up (`25 + bar(25)`).
  - **Name:** trimmed, with a trailing `:` dropped. Text in `( )` or `[ ]` becomes an entry note ("Reverse grip lat pull downs (go wide)" → note "go wide"; "Bench dumbbell rows [Each Hand]" → note "Each Hand"). The rest goes through exact, alias and similar-name matching, and review can map it to an existing exercise **and a variation** (e.g. "Wide grip lat pull down" → Lat Pulldown · Wide grip).
  - **Amount:** `SxR` (two numbers) is S sets of R reps. Three or more numbers are the reps of each set (`15x12x12` is three sets of 15, 12, 12). `Sx60s` is S sets of 60 seconds. `12>10>8` is one drop set. Text in `( )` becomes an entry note (`3x12 (Slow)`).
  - **Weight:** a number, with optional `lb`/`lbs`/`pound(s)`. `A + B` and `A + bar(B)` add up (`25 + bar(25)` = 50). `BW` and `BW + 25` are bodyweight. `N pound assist` (or `assist N`) is an assistance weight of N, and marks the exercise as `assisted` (a new exercise is created that way; a match to a non-assisted exercise is flagged in review). One weight applies to every set; `85/75/65` gives each set its own; `40>30>20` gives each drop step its own. A blank weight is BW on a bodyweight exercise and a problem row otherwise.
  - Anything else is a problem row showing the raw line.
- **Assisted exercises** (decided 2026-11-26): for `assisted` exercises lower is better everywhere. The working weight is the **least** assistance across the entry's sets; target prompts compare that; PRs are "least assistance" and "most reps at an assistance"; progression charts plot assistance with "lower is better" labelled; no estimated 1RM; no volume.
- **Tracking detail** (decided 2026-11-26): every set is stored; shorthand expands into sets; a drop set is one set with several segments.
- **Exercise types in v1** (decided 2026-11-26): `weight_reps` (with load `weighted`, `bodyweight` or `assisted`) and `duration` (sets × time). Nothing else.

### Open (deferred to the design phase)

Don't build or decide these until the design phase settles them; stop and ask if a task needs one.

- **Superset notation in imports.** Provisional: ` + ` in the Name, ` ; ` between members' Amount and Weight. Build it this way, but keep the superset splitting in one function so the notation can change; revisit with the owner before M4.
- **Tailscale and proxy wiring.** No Tailscale setup exists yet. Tailscale Services or a sidecar, and how the host proxy sits with Tailscale's TLS, are decided during design and before M5.
