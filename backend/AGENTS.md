# AGENTS.md — /backend

Read the root `AGENTS.md` first. This file adds rules specific to the Go API.

## Stack

- Go, SQLite in WAL mode (`modernc.org/sqlite`, no CGO), single API process
- REST/JSON implementing the spec in `/api`. The spec is the contract; change it first.
- `chi` router; handlers generated from the spec with `oapi-codegen` (strict server), pinned as a `tool` in `go.mod`
- Runs in Docker; the SQLite file lives on a mounted volume

## Suggested layout

```
cmd/api/            main.go (wiring only); subcommands: serve, migrate, user add|set-password|revoke, backup now
internal/api/       generated from /api/openapi.yaml (never edit by hand)
internal/auth/      passwords (argon2id), sessions, middleware
internal/http/      handlers, routing, errors, request limits
internal/store/     all SQL; the only package that touches the database
internal/domain/    core types and rules: shorthand parsing, working weight, 1RM, PRs, name similarity (no HTTP, no SQL)
internal/imports/   Markdown, .xlsx and .csv row readers, the shared row parser, and the CSV/JSON export writers (one column definition shared by import and export)
migrations/         numbered SQL migrations, embedded
```

## Rules

- **Weights** are `int64` hundredths of a pound; durations are `int64` seconds. Never float types. Estimated 1RM is computed with integer maths and rounded to a whole pound for display.
- **Dates** are `YYYY-MM-DD` text with no time zone. The workout's date comes from the client (the user's local date); never derive it from a UTC timestamp.
- **Tenancy:** every query in `internal/store` takes the authenticated `user_id` and filters on it. Composite foreign keys `(x_id, user_id)` stop a row from referencing another user's exercise, variation, superset, routine or muscle group. Add a tenancy test for every new query.
- **Auth:** same as Penny Pincher (root §10). Only the token's SHA-256 is stored. Never log or return a password, hash or token. Sessions store a short device label derived from the user agent (never the raw user agent string in logs). Revoking a session deletes its row; `sessions:revoke-others` deletes all but the caller's.
- **Roles:** admin routes (`/v1/admin/...`) require `admin` and return 403 otherwise.
- **Muscle groups:**
  - `exercise_muscle_groups (exercise_id, muscle_group_id, is_primary)` with a partial unique index so each exercise has exactly one primary. Writing an exercise replaces its rows in one transaction.
  - **Volume and muscle-group streaks use the primary group only.** Secondary groups only affect library filtering.
  - `deprecated` is a flag; deprecate and restore never touch exercises or logs. Deleting a group referenced by any `exercise_muscle_groups` row is `409 muscle_group_in_use`.
- **Plan vs log:**
  - **Variations** are rows per exercise (`name` case-insensitive unique per exercise, `sort_order`, `archived`). Creating an exercise creates its first variation in the same transaction ("Standard" unless named).
  - **Variants** are one row per `(exercise, variation, context, style, scheme)`: context is isolated (`superset_id IS NULL`) or a superset; style is `straight` or `drop`. Store the scheme as columns (`sets`, `reps` for straight) plus a canonical `scheme_key` text (e.g. `3x14`, `1x10>8>6`) so uniqueness is one partial unique index per context, on `(variation_id, style, scheme_key)`. The scheme and variation are immutable; only the target load changes (`target_weight` / `target_seconds`, or the step weights).
  - Drop variants keep their steps in `variant_steps`; replacing the step weights is one transaction. Validate **at least two steps**. There is **no direction rule**: weights and reps may rise, fall or stay the same between steps. Logged drop sets are not validated either: log what happened.
  - Every logged set stores its `variant_id`, which is how history and trends are grouped per variant (and per variation through the variant).
  - No code path that changes a variant's target load, a variation, an exercise, a muscle group, a superset or a routine writes to `logged_sets` or `set_segments`. Test this.
  - `target-decision: update` changes only the target load of that entry's own variant: the isolated variant for an exercise entry, each member's superset variant for a superset entry, and the substitute's variant for a swapped entry.
  - **Straight prompts** compare the working weight (the heaviest first-segment weight across the entry's sets, or the longest duration) with the target load. **Drop prompts** compare the first logged set's step weights with the target step weights. A prompt exists only when they differ.
- **Supersets:** at least two members. Members may share an exercise when their variants differ; the same variant twice is `400 superset_duplicate_member`.
- **Routine groups:** `routine_groups (id, user_id, name, sort_order)`; `routines.group_id` nullable with `ON DELETE SET NULL`; sort order is per group. Group changes don't touch a routine's `updated_at`.
- **Routines:** `created_at` and `updated_at` columns. `updated_at` is set only by `PUT routines/{id}/items` when the item list actually changes (added, removed or reordered). Renames, target-load edits, workouts and finishing a workout never touch it. `:duplicate` copies the items in one transaction with fresh timestamps.
- **Suggested routine:** among the user's routines, the one with the oldest last-done date, never-done first, ties broken by sort order; computed in SQL on read, never stored.
- **Archive, don't delete:** deleting an exercise, variation, variant or superset with logged sets is rejected (`409`); archiving is always allowed. Check in the store first and return a clear error; a trigger is the safety net.
- **Workouts:** at most one in progress per user (a partial unique index on `finished_at IS NULL`). Starting from a routine copies its items into entries in one transaction. Later routine edits never touch existing workouts.
- **Shorthand** (`internal/domain`): `NxR @W`, `NxR BW`, `NxR BW+W`, `NxDs` (duration), and a drop set as `R@W > R@W > R@W` (one set, one segment per step, in any weight order), with two short forms: `R @W>W>W` (same reps) and `@W R>R>R` (same weight), case-insensitive, flexible spaces. Unknown input is a `400 invalid_shorthand` with the position of the problem. Table-test it.
- **Aggregations** (weekly totals, muscle-group volume, PRs, progression series) are done in SQL, not in application loops.
- **Exercise page** (`GET exercises/{id}/page`): built from a few grouped queries (variations with their variants and routine usage, per-variant last done and best, per-variant trend series of the last 12 workouts, the exercise-wide estimated 1RM series and weekly totals), never one query per variant. Estimated 1RM is what makes variants and variations comparable in the overview; per-variant charts plot the working weight itself.
- **Volume** sums `weight × reps` over every segment. Bodyweight sets count reps toward total reps and only added weight toward volume. Assisted sets count reps toward total reps and nothing toward volume. Duration sets count toward neither. Weights are never negative (a `CHECK` constraint).
- **Load types:** `exercises.load` is `weighted`, `bodyweight` or `assisted`, and is locked once the exercise has logged sets. For `assisted`, every "best" is inverted in SQL: working weight is `MIN` assistance, PRs are least assistance and most reps at an assistance, no estimated 1RM; target prompts compare the least assistance.
- **New user seeding:** the six muscle groups (Legs, Chest, Back, Shoulders, Arms, Cardio), in the same transaction as the user row. No routine groups or routines.
- **Onboarding:** `POST onboarding` creates the chosen split's empty routine groups (names in root §6, use case 14) and sets `onboarded` and `weekly_goal` in one transaction. The split itself is not stored.
- **Streaks** (in SQL): weeks run Monday to Sunday by workout date. The weekly-goal streak counts consecutive completed weeks with at least `weekly_goal` finished workouts, plus the current week once it meets the goal; an unfinished current week below the goal neither counts nor breaks it. Muscle-group streaks do the same with "at least one logged set for an exercise whose **primary** group is that group". A goal change applies to every week (no history of goals in v1).
- **Migrations:** a new numbered file per change; never edit an applied one. Applied on startup before the server listens; a failure stops the API.
- **Imports:**
  - Parsing never writes workouts. Start writes in batches inside transactions, tagging every row with `import_id`; cancel, undo and failure delete them in one transaction. Undo also deletes aliases and exercises that the import created and nothing else uses.
  - Exercise names match case-insensitively against names **and aliases**. Remaining names go through **similarity** (`internal/domain`): normalise (lowercase; drop spaces, hyphens and punctuation; expand DB→dumbbell, BB→barbell, KB→kettlebell, OH→overhead), then exact match on the normalised form or a Damerau–Levenshtein distance ≤ 2 (≤ 1 for names under 8 characters). Similar names are review questions; the rest are new exercises.
  - Start is refused (`409 import_decisions_pending`) until every similar name has an answer. A `same` answer saves the file name as an `exercise_aliases` row in the import's transaction.
  - **Row reader** (`internal/imports`, grammar in root §10): one reader per container (Markdown pipe table, `.xlsx` first sheet, `.csv`) that yields rows of Date? · Routine? · Name · Amount · Weight · Notes? with the raw line; then one shared row parser. Date column or Markdown date headings split workouts; without either the file is one workout dated at review; name cleanup (trailing `:`, `( )` and `[ ]` to notes); Amount `SxR`, per-set reps (`15x12x12`), `SxDs`, drop `R>R>R`, `( )` notes; Weight numbers with optional units, `+` sums including `bar(N)`, `BW`, `BW + N`, `N pound assist` → assistance N, per-set `/` and per-step `>` lists; a ` + ` in the name (outside brackets) makes a superset row, with ` ; ` separating per-member Amount and Weight. Each row keeps its raw line for review. Table-test it with generic rows in the owner's layout.
  - Weights convert to hundredths exactly (never via float). Use `excelize` for `.xlsx`, with unzip size limits set. Never log file contents.
- **Export:** CSV and JSON are written by streaming from the store (no full in-memory copy). The CSV uses the import columns (Date · Routine · Name · Amount · Weight · Notes) and the same grammar, written by the same code as the template, so it imports back; weights are exact decimal pound text from hundredths. Sets the grammar can't express exactly (an unplanned drop on a straight set) are written with per-set `>` notation, and the JSON backup is the lossless copy. JSON keeps raw integers. Exports are read-only and never log contents; record only the time and kind of the last export.
- **Backups** (`internal/backup`): a nightly `VACUUM INTO` snapshot into `BACKUP_DIR` (a mounted folder), named by UTC timestamp, keeping the newest 14. Write to a temporary name and rename when complete, so a half-written file is never kept. The schedule runs inside the API process; `api backup now` runs one by hand. Without `BACKUP_DIR` set, backups are off and Status says so. Restore steps are in `docs/backups.md`.
- **Network:** the API listens on a `127.0.0.1` port published by Compose; TLS and the tailnet hostname are handled outside the app (root §10). Never trust `Tailscale-User-*` headers.
- **Logging:** never log weights, exercise names, aliases, notes or tokens. Log event types and ids only.
- **Config** comes from environment variables. Never commit secrets.
- Errors use the body defined in `/api`; do not leak internals.

## Testing

- Use a temporary SQLite database per test; do not mock the store.
- **PR suite budget:** `go vet ./...` and `go test ./...` together stay well under a minute, so the whole PR check fits in 2 minutes. Anything slow (large imports, big analytics fixtures) goes behind a build tag and runs at the milestone gate.
- Cover:
  - tenancy isolation on every query
  - one variant per exercise, variation, context, style and scheme; `variant_exists` returns the existing one; 3 × 14 and 4 × 10 of one exercise keep separate target loads and trends; the same scheme in two variations is two variants; superset and isolated variants independent; drop variants in supersets; drop rejected for duration exercises; drops need two steps; rising, falling and mixed drop steps accepted; schemes and variations can't be edited on a variant
  - a new exercise gets its first variation; variation names unique per exercise; archive vs delete for variations with history
  - exactly one primary muscle group; volume and muscle-group streaks use the primary only; deprecate and restore leave exercises and logs untouched; delete refused for a group in use
  - a superset with the same exercise twice in different variations; the same variant twice rejected
  - changing a target load, renaming or archiving an exercise or variation, deprecating a muscle group, and editing or deleting a routine leave every logged set unchanged
  - routine `updated_at` changes on add, remove and reorder, and not on rename, target edits or workouts; duplicate copies items
  - the suggested routine (never done first, then oldest, ties by order)
  - target decisions update only the entry's variant, for straight, drop, straight and drop superset members, and swapped entries; an unplanned drop on a straight entry
  - onboarding for every split, and `already_onboarded`
  - weekly-goal and muscle-group streaks: a week below goal breaks it, the current week below goal doesn't, weeks across a year boundary
  - sessions: list shows only the caller's, revoke one, revoke others keeps the caller
  - backups: a snapshot opens as a valid database, only 14 are kept, a failed run leaves no partial file
  - the exercise page: every variant listed and grouped by variation and superset, routine usage, per-variant best and trend, overview across variants
  - delete rejected for exercises, variations, variants and supersets with history; archive allowed
  - one workout in progress; resume returns it with every logged set
  - shorthand parsing (table tests, including bad input and rising drops)
  - volume, total reps, estimated 1RM limits (1–12 reps, weight above 0) and PRs, including bodyweight, assisted, drop sets and duration sets
  - weekly totals across a year boundary
  - name similarity (table tests: abbreviations, spacing, punctuation, edit distance limits, no false match on short names)
  - import row parsing (the same table tests run through the Markdown, `.xlsx` and `.csv` readers: `3x10`, `15x12x12`, `15x15x12x12`, `3x60s`, `12>10>8`, notes from `( )` and `[ ]`, `25 + bar(25)` = 50, `5 pound assist` = assistance 5 on an assisted exercise, superset rows split on ` + ` with ` ; ` per-member Amount and Weight (and a single shared value), per-set and per-step weights, a blank weight on bodyweight and non-bodyweight exercises, superset `+` rows, bad lines)
  - assisted exercises: working weight is the least assistance; target prompts and PRs treat lower as better; no volume, no 1RM; negative weights rejected; `load` locked after history
  - import parsing, review, similar-name decisions gating start, aliases saved on `same` and matched next time, undo removing what the import created
  - routine groups: onboarding creates the split's groups; deleting a group keeps its routines with no group; tenancy on groups
  - export: CSV round-trips through import to the same sets; JSON contains every entity and integer weights; nothing is written
  - admin-route authorization (members get 403)
- Run `go vet ./...` and `go test ./...` before finishing.

## Generated API code

- `internal/api/*.gen.go` is generated from `/api/openapi.yaml`. Never edit it by hand; run `make generate` after any spec change. `make check-generate` fails if it is stale (runs in CI).
- `internal/http.Server` implements the generated `StrictServerInterface` and embeds `api.Unimplemented`; an operation it doesn't override answers `501 not_implemented`.
