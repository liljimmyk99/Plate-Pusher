# AGENTS.md — /api

Read the root `AGENTS.md` first. This folder holds the OpenAPI spec (`openapi.yaml`, OpenAPI 3.1), the single contract between the backend and the web app.

## Rules

- **Spec first:** any change to request/response shapes starts here. Then update the backend, then the web app, ideally in one commit.
- Keep the spec valid and in one source of truth; clients generate their types from it.
- Lint from this folder with `npx @redocly/cli lint` (config in `redocly.yaml`). It must pass with no errors or warnings; an intentional exception goes in `.redocly.lint-ignore.yaml` with a comment saying why.
- Version routes under `/v1`.

## Conventions

- **Weights:** integers in hundredths of a pound (`integer`, int64). Never negative (`400 negative_weight`). What a weight means comes from the exercise's `load`: the weight lifted (`weighted`), added weight (`bodyweight`, `0` = BW) or assistance (`assisted`, lower is better). Never decimals or strings, including in the JSON export.
- **Durations:** integer seconds.
- **Dates:** calendar dates as `YYYY-MM-DD`, no time zone. Timestamps (`started_at`, `finished_at`, `created_at`, `updated_at`, `last_used_at`) are RFC 3339 in UTC.
- **IDs:** opaque strings.
- **Auth:** an opaque session token (from `POST /v1/auth/login`) sent as a bearer token on every route except `/healthz` and `auth/login`. All data is scoped to the authenticated user; `user_id` is never accepted from the client.
  - Auth errors: `401 unauthenticated`, `401 session_expired` (older than 30 days), `401 invalid_credentials` (login only, one generic answer), `403 password_change_required` (every route except `auth/password`, `auth/logout` and `me`).
  - Tokens are opaque strings; clients never parse them. No route accepts or returns a password hash or token hash.
- **Errors:** one error body (`code`, `message`, optional field `details`) across all routes. Clients branch on `code`, never `message`.
- **Limits:** any route can return `413 request_too_large` and `429 rate_limited` (with `Retry-After`), described once in the spec's conventions.
- **Lists:** paginated where they grow without bound (workouts, sets); filterable by date range.
- **Derived numbers** (volume, estimated 1RM, PRs, totals, "suggested routine") are computed by the API and returned read-only; clients never store them.
- **Per-device preferences** (accent colour, weight step, keep screen awake) are not in the API; the web keeps them locally.

## Resources (v1)

- `auth` (same as Penny Pincher): `POST auth/login`, `POST auth/logout`, `POST auth/password`, `GET me` (includes `display_name`, `role`, `onboarded`)
- `sessions`: `GET sessions` (the user's own: id, device label, created at, last used at, `current` flag), `DELETE sessions/{id}` (sign one out; the current one is the same as logout), `POST sessions:revoke-others`
- `muscle-groups`: list (with `deprecated` and exercise count), create, rename, reorder, `POST …/{id}/deprecate`, `POST …/{id}/restore`. Deleting one in use is `409 muscle_group_in_use` (suggest deprecating). Deprecated groups are excluded from pickers with `?include_deprecated=false` (the default for pickers) and always included in history and analytics.
- `exercises`: CRUD, archive and unarchive. Body includes `primary_muscle_group_id`, `secondary_muscle_group_ids` (no duplicates, never the primary), `tracking`, `load` (`weighted` | `bodyweight` | `assisted`), `notes` (how-to). `load` can't change once the exercise has logged sets (`409 load_locked`). Creating one also creates its first variation ("Standard", or a name given). Deleting one with logged sets is `409 exercise_has_history` (suggest archiving).
  - `GET exercises/{id}/page`: everything the exercise page needs in one call: notes, the overview (best estimated 1RM series, weekly reps and volume, PRs, last done), and every variant grouped as isolated **by variation** (each variation with its variants, straight then drop) and per superset (with the superset's other members named). Each variant carries its variation, scheme, target load, the routines using it, last done, best, and a short trend series.
  - `GET exercises/{id}/aliases`, `DELETE exercises/{id}/aliases/{alias_id}` (aliases are created by import).
- `variations`: under an exercise: create, rename, reorder, archive and unarchive. Names are unique per exercise (`409 variation_exists`). Deleting one with variants that have logged sets is `409 variation_has_history`.
- `variants`:
  - `POST variants` (exercise, variation, context, style, scheme, target load), returning the existing variant with `409 variant_exists` if that scheme already exists for that variation and context. Drop variants need at least two steps (`400 drop_too_short`); there is no direction rule. Drop is `weight_reps` only (`400 drop_needs_weight_reps`).
  - `PATCH variants/{id}`: the target load only. The scheme and variation can't change; a new scheme is a new variant.
  - Archive and unarchive; deleting one with logged sets is `409 variant_has_history`.
  - `GET variants/{id}/progression` (working weight or step weights or duration per workout, the target load over time, estimated 1RM where it applies) and `GET variants/{id}/history` (paged workouts with every set).
- `supersets`: CRUD, archive, reorder members (at least two; `400 superset_too_small`). Members may repeat an exercise with different variations or variants; the same variant can't appear twice (`400 superset_duplicate_member`). Each member is a variant whose context is this superset (created with `POST variants`).
- `routine-groups`: CRUD and reorder. Names are unique per user (`409 routine_group_exists`). Deleting a group sets its routines' `group_id` to null in the same transaction.
- `routines`: CRUD (with optional `group_id`), reorder within a group, move to another group, and `POST routines/{id}:duplicate` (copies items; new name required). `PUT routines/{id}/items` replaces the ordered item list. Each item is an isolated variant or a superset, never both. Creating a routine with no items is allowed by the API (onboarding does it); the web blocks saving an empty new routine. Each routine returns `created_at`, `updated_at`, `item_count`, `last_done` and `suggested` (true on at most one routine).
- `onboarding`: `POST onboarding` with `split` (`body_part` | `push_pull_legs` | `upper_lower` | `full_body` | `custom`) and `weekly_goal`. Creates that split's empty routine groups and sets `onboarded`. A second call is `409 already_onboarded`. `GET me` returns `onboarded`.
- `settings`: `GET`/`PATCH settings` (`weekly_goal`, 1–7)
- `workouts`:
  - `POST workouts` (from a routine, or empty), `GET workouts` (paged, by date range and routine, including `routine=none` for empty workouts), `GET workouts/current` (the one in progress, if any), `GET`/`PATCH`/`DELETE workouts/{id}`
  - Entries: add, reorder, skip, swap (`POST workouts/{id}/entries/{entry_id}/swap`)
  - Sets: `POST`/`PATCH`/`DELETE` on an entry's sets, each with its segments. `POST …/sets:expand` turns shorthand (`3x10 @135`, or `10@135 > 8@115 > 6@95` for one drop set) into sets (and, for an ad-hoc entry, the matching variant), so the web and a future iOS app share one parser.
  - `POST workouts/{id}/finish` returns the summary: totals, new PRs, and the target prompts not yet answered.
  - `POST workouts/{id}/entries/{entry_id}/target-decision`: `update` or `keep`. Updates the entry's variant: a straight variant's target load becomes the working weight (or duration); a drop variant's step weights become the first logged set's. Never changes the scheme. Never changes a logged set.
  - One workout in progress per user: starting another is `409 workout_in_progress`.
- `analytics`: `GET analytics/weekly` (reps and volume per week, and per **primary** muscle group), `GET analytics/frequency` (workouts per week, the weekly-goal streak, per-muscle-group weekly streaks by primary group, days trained this week), `GET analytics/records` (recent PRs)
- `imports`: template download (`.xlsx`), upload and check (`.md`, `.xlsx`, `.csv`; anything else is `400 unsupported_import_format`), review, start, cancel, undo, history. A file without a Date column needs `date` (and optional `routine_name`) in `PUT imports/{id}/decisions` before start (`409 import_decisions_pending`).
  - The review returns `matched`, `similar` (each: file name, row count, candidate exercise and variation), `new_exercises`, `problems` (with the raw line), and `rows` (each parsed row: raw line, sets with reps and weights, notes).
  - `PUT imports/{id}/decisions` records, for each similar name, `same` (with the exercise id) or `different`, and for each new exercise a muscle group or an existing exercise to match. Start is `409 import_decisions_pending` until every similar name has an answer.
  - Starting an import saves an alias for every `same` answer.
- `export`: `GET export.csv?from=&to=` (the workout log, one row per entry with Date · Routine · Name · Amount · Weight · Notes in the import grammar, weights as pounds text converted exactly from hundredths) and `GET export.json` (full backup: muscle groups, exercises with variations, variants, targets and aliases, supersets, routines, workouts; weights as integers). Both stream a file with `Content-Disposition: attachment`. `GET export/last` returns the time and kind of the last export.
- `admin` (admins only): `GET`/`POST admin/users`, `POST admin/users/{id}/password`, `GET admin/status` (version, uptime, database size, last backup time and size)
- `/healthz` (no auth)
