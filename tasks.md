# Implementation Plan: Trivia Squads MLP

## Overview

This plan converts the Trivia Squads MLP design into an ordered, incrementally-building series of coding tasks. Each task is sized at 1–3 hours of focused work. Tasks marked with `*` are optional and can be skipped for a faster MLP.

Build order: scaffold → database → contracts → API foundation → domain modules (players → teams → games → rounds → questions → scoring → status → notifications) → worker → question bank → frontend → email templates → property-based tests → integration/E2E → a11y → observability → deployment.

All code is TypeScript throughout, per design Section 13. React 18 + Vite + Tailwind on the frontend, Fastify + Zod + Kysely + `pg` on the backend, Supabase Postgres 15, Resend email, Railway cron, Vitest + fast-check + Playwright + axe-core for tests.

## Tasks

### 1. Project scaffolding and repo setup

- [ ] 1.1 Initialize monorepo with pnpm workspaces and TypeScript project references
  - Create root `package.json` with `pnpm` workspace config covering `apps/*` and `packages/*`
  - Add root `tsconfig.base.json` with strict mode, `ES2022` target, `NodeNext` modules
  - Create `apps/api`, `apps/web`, `packages/shared` directories with per-package `package.json` + `tsconfig.json` that extend the base

- [ ] 1.2 Configure linting, formatting, and editor config
  - Add ESLint with `@typescript-eslint`, `eslint-plugin-import`, React plugin for `apps/web`
  - Add Prettier with a shared config at repo root
  - Add `.editorconfig`, `.gitignore` (covering `node_modules/`, `build/`, `.env`, `dist/`, Playwright artifacts)

- [ ] 1.3 Create `.env.example` and environment loader
  - Enumerate required env vars: `DATABASE_URL`, `SUPABASE_URL`, `SUPABASE_SERVICE_ROLE_KEY`, `SUPABASE_JWT_SECRET`, `RESEND_API_KEY`, `APP_URL`, `SENTRY_DSN`
  - Implement a single `loadEnv.ts` in `packages/shared` that validates env at startup via Zod and fails fast on missing values

- [ ] 1.4 Scaffold Railway deployment config
  - Add `railway.toml` at repo root defining two services: `api` (HTTP) and `worker` (cron)
  - Define build commands per service using `pnpm --filter` to build only the needed app
  - Document env-var requirements in `README.md`; do not commit real values

- [ ] 1.5 Scaffold Supabase local config
  - Add `supabase/config.toml` with Postgres 15 settings, Auth magic-link provider enabled, email redirect URLs pointing at `${APP_URL}/auth/callback`
  - Document Supabase project creation steps in `README.md`

- [ ] 1.6 Add CI workflow skeleton
  - Create `.github/workflows/ci.yml` running on PR: install, lint, typecheck, test (unit + integration against a Postgres service container), build
  - Cache pnpm store keyed on `pnpm-lock.yaml`

- [ ] 1.7 Checkpoint — verify scaffolding
  - Ensure `pnpm install`, `pnpm -r typecheck`, `pnpm -r lint`, and CI all run cleanly on a stub `index.ts` in each app. Ask the user if questions arise.

### 2. Database foundation

- [ ] 2.1 Install migration tooling and wire to `apps/api`
  - Add `node-pg-migrate` (or Kysely migrator) as the migration runner; pick one and document the choice
  - Add `pnpm db:migrate`, `pnpm db:migrate:create`, `pnpm db:reset` scripts wired to `DATABASE_URL`

- [ ] 2.2 Migration: `players` table (design §4.1)
  - Implement `players` DDL with `auth_user_id`, `email`, `display_name`, `created_at`, `updated_at`, and the `display_name_length` CHECK
  - _Requirements: 1, 15.2_

- [ ] 2.3 Migration: `teams` + `team_memberships` tables (design §4.1)
  - Implement both tables with the documented indexes (`ix_teams_organizer`, `ix_tm_player`) and CHECK on team name length
  - _Requirements: 2.1, 2.2, 2.9_

- [ ] 2.4 Migration: `games` table with `game_state` enum (design §4.1)
  - Implement `game_state` enum, `games` DDL, the `distinct_teams` and `active_has_opponent` CHECKs, and all four indexes including the partial `ix_games_team_completed` used by question lookback
  - _Requirements: 3.1, 3.6, 3.9, 9.1, 5.3_

- [ ] 2.5 Migration: `rounds` table with `round_state` enum (design §4.1)
  - Implement `round_state` enum, `rounds` DDL with the UNIQUE (game_id, round_number, team_id), round-number CHECK, and partial sweep index `ix_rounds_sweep`
  - _Requirements: 4.1, 6, 7, 11_

- [ ] 2.6 Migration: `round_owners` with partial unique indexes (design §4.1)
  - Implement `round_owner_role` enum and `round_owners` DDL
  - Create the two partial unique indexes `uq_round_owners_active` (one active owner per round) and `uq_round_owners_one_sub` (at most one substitute row per round)
  - _Requirements: 7.9, 11.2_

- [ ] 2.7 Migration: `questions` + `answer_choices` tables (design §4.1)
  - Implement `questions` with the 200-char CHECK on `question_text`, `answer_choices` with position 1..6 CHECK and UNIQUE (question_id, position)
  - _Requirements: 5.6_

- [ ] 2.8 Migration: deferred trigger enforcing 2–6 choices and valid `correct_answer_id` (design §4.1)
  - Create a deferred CONSTRAINT TRIGGER on `questions` + `answer_choices` that runs at COMMIT and asserts 2 ≤ choice count ≤ 6 and that `questions.correct_answer_id` references a row in the question's `answer_choices`
  - _Requirements: 5.6_

- [ ] 2.9 Migration: `round_questions` table (design §4.1)
  - Implement `round_questions` with UNIQUE (round_id, question_id) preventing intra-round duplicates and position CHECK 1..5
  - _Requirements: 5.2, 5.5, 5.6_

- [ ] 2.10 Migration: `round_answers` table with consistency CHECK (design §4.1)
  - Implement `round_answers` with the `answer_consistency` CHECK and the `timer_started_at` column used for server-authoritative timing
  - _Requirements: 6.5, 6.6, 10.1_

- [ ] 2.11 Migration: `email_outbox`, `notification_type` enum, `notifications_sent` idempotency ledger (design §4.1)
  - Implement the `notification_type` enum and both tables
  - Create the `round_id_key` generated column and the composite PK on `notifications_sent` so NULL `round_id` is distinguishable
  - Add `ix_outbox_unsent` partial index
  - _Requirements: 12_

- [ ] 2.12 Migration: `game_sitters` table (design §4.1)
  - Implement `game_sitters` with composite PK (game_id, team_id, player_id)
  - _Requirements: 4.6, 7.2_

- [ ] 2.13 Migration: RLS policies for defense-in-depth (design §9.2)
  - Add the four documented RLS policies (`p_self` on players, `t_member` on teams, `g_participant` on games) and REVOKE/GRANT for `round_questions`, `round_answers`
  - _Requirements: 2.7, 8.3, 8.5, 15.5_

- [ ] 2.14 Add Kysely type generation from the Postgres schema
  - Run `kysely-codegen` or similar against the migrated DB; commit the generated `db.d.ts` to `apps/api/src/db/types.ts`
  - Add a CI check that schema types are in sync with the latest migration

- [ ] 2.15 Checkpoint — verify schema and constraints
  - Apply all migrations against a fresh Postgres, insert a known-bad row (e.g., 9th team member, duplicate position), confirm the expected CHECK/unique violations. Ask the user if questions arise.

### 3. Shared types and API contracts

- [ ] 3.1 Define Zod schemas for shared enums and primitives in `packages/shared`
  - `game_state`, `round_state`, `notification_type`, `round_owner_role` as Zod enums
  - `UuidSchema`, `TimestampSchema`, `DisplayNameSchema` (1–30 chars), `TeamNameSchema` (1–60 chars)
  - _Requirements: 15.2_

- [ ] 3.2 Define Zod schemas for Auth and Player endpoints (design §5.1, §5.2)
  - `MagicLinkRequest`, `CreatePlayerRequest`, `UpdatePlayerRequest`, `PlayerMeResponse` (includes `email` only here)
  - _Requirements: 1, 15_

- [ ] 3.3 Define Zod schemas for Team endpoints (design §5.3)
  - `CreateTeamRequest`, `TeamDetailResponse` (Display_Name only, no email), `JoinTeamResponse`, `TeamsMineResponse`
  - _Requirements: 2_

- [ ] 3.4 Define Zod schemas for Game endpoints (design §5.4)
  - `CreateGameRequest`, `AcceptGameRequest`, and the `GameStatusResponse` discriminated union with three variants: `GameStatusPending`, `GameStatusActive`, `GameStatusComplete`
  - The Active variant's round `status` is itself a discriminated union of `not_started | in_progress | sub_needed | forfeited | complete`
  - The Active variant NEVER contains score, correctness, submitted-answer, or question-text fields
  - _Requirements: 3, 8, 9_

- [ ] 3.5 Define Zod schemas for Round endpoints (design §5.5)
  - `RoundPreviewResponse`, `StartRoundResponse`, `QuestionResponse` (includes `timer_started_at` + server-computed `seconds_remaining`), `SubmitAnswerRequest`, `SubmitAnswerResponse`
  - _Requirements: 6_

- [ ] 3.6 Define the error envelope and stable error-code enum (design §5, §11)
  - `ErrorEnvelope` Zod schema: `{ error: { code, message, details? } }`
  - `ErrorCode` enum covering every code in design §11 table (`MAGIC_LINK_INVALID`, `UNAUTHENTICATED`, `DISPLAY_NAME_INVALID`, `DISPLAY_NAME_TAKEN_IN_TEAM`, `TEAM_FULL`, `TEAM_INVITE_INVALID`, `TEAM_TOO_SMALL_FOR_GAME`, `NO_ELIGIBLE_TEAM`, `CANNOT_PLAY_SELF`, `GAME_NOT_ACCEPTING`, `ROSTER_OVERLAP`, `NOT_ROUND_OWNER`, `INSUFFICIENT_WINDOW`, `ROUND_UNAVAILABLE`, `TIMER_EXPIRED`, `ALREADY_ANSWERED`, `GAME_NOT_COMPLETE`, `NOT_GAME_PARTICIPANT`, `QUESTION_BANK_EXHAUSTED`)

- [ ]* 3.7 Unit tests for contract parsing and serialization
  - Round-trip-parse each representative shape; assert the Active/Complete discriminated-union branches do not share a TypeScript type


### 4. API foundation (Fastify)

- [ ] 4.1 Bootstrap Fastify app in `apps/api`
  - Create `apps/api/src/server.ts` with Fastify instance, CORS for `APP_URL`, JSON body parser, graceful-shutdown hook
  - Wire to `loadEnv.ts` and start listener on `PORT`

- [ ] 4.2 Implement Supabase JWT verification middleware (design §9.1, §9.2)
  - Parse `Authorization: Bearer <token>`; verify signature using Supabase JWKS
  - Resolve `auth_user_id` → load or lazily-create matching `players` row (email from token claims, `display_name = NULL`)
  - Attach `{ player_id, auth_user_id }` to request context
  - Return 401 `UNAUTHENTICATED` on missing/invalid/expired token
  - _Requirements: 1.2, 1.4_

- [ ] 4.3 Implement error-envelope exception handler
  - Central Fastify error hook that converts thrown `AppError` classes into the `ErrorEnvelope` shape with the right HTTP status and stable code
  - Ensure unknown errors become 500 `INTERNAL_ERROR` with Sentry capture but no stack in the response body

- [ ] 4.4 Implement Zod request validation plugin
  - Helper that takes a Zod schema and returns a Fastify preHandler producing 400 `VALIDATION_ERROR` with field details on parse failure

- [ ] 4.5 Implement Kysely DB client factory with per-request transaction helper
  - `withTx(fn, { isolation: 'repeatable read' })` wrapper for multi-row invariant writes (design §4.3)
  - Centralize connection pooling; reuse a single `pg` pool per process

- [ ] 4.6 Health check endpoint `GET /health`
  - Returns 200 with `{ status: "ok", db: "ok" }` after a trivial `SELECT 1`

### 5. Auth module (design §5.1)

- [ ] 5.1 Implement `POST /auth/magic-link` handler
  - Accepts `{ email }`; invokes Supabase Auth `signInWithOtp` with `emailRedirectTo: ${APP_URL}/auth/callback`
  - Returns 200 with `{ status: "sent" }` within the 5-second SLO budget
  - _Requirements: 1.1_

- [ ] 5.2 Implement `GET /auth/callback` handler
  - Exchanges Supabase one-time code for a session; sets secure cookie or returns the session to the frontend for storage
  - Maps Supabase "expired / already used" errors to 401 `MAGIC_LINK_INVALID` with a `resend_url`
  - _Requirements: 1.2, 1.3_

- [ ] 5.3 Implement `POST /auth/sign-out` handler
  - Invalidates the Supabase session server-side; client clears JWT
  - _Requirements: 1.5_

- [ ]* 5.4 Integration tests for auth endpoints
  - Mock Supabase; assert magic-link request shape, callback success, expired-link error mapping, sign-out behavior
  - _Requirements: 1.1, 1.2, 1.3, 1.5_

### 6. Players module (design §5.2, Req 15)

- [ ] 6.1 Implement `GET /players/me` handler
  - Returns `{ id, display_name, email }` for the caller only; the only endpoint that ever returns an email
  - _Requirements: 15_

- [ ] 6.2 Implement `POST /players` handler for first-sign-in Display_Name set
  - Validates length 1–30 via shared `DisplayNameSchema`; rejects empty or > 30 with 400 `DISPLAY_NAME_INVALID`
  - Sets `display_name` on the caller's player row; idempotent if already set to the same value
  - _Requirements: 15.1, 15.2_

- [ ] 6.3 Implement `PATCH /players/me` handler for Display_Name updates
  - Same validation as 6.2
  - Checks 15.7 uniqueness across every team the player currently belongs to; returns 409 `DISPLAY_NAME_TAKEN_IN_TEAM` with `details.team_id` on conflict
  - _Requirements: 15.3, 15.4, 15.7_

- [ ] 6.4 Create `players_public` view (id, display_name) and wire all cross-player serializers to read from it
  - Defense-in-depth against accidental email leakage in cross-player responses
  - _Requirements: 2.7, 15.5_

- [ ]* 6.5 Unit tests for Display_Name validation boundary cases
  - Empty, whitespace-only, exactly 1 char, exactly 30 chars, 31 chars, emoji, mixed case
  - _Requirements: 15.2_

- [ ]* 6.6 Integration tests for players endpoints
  - First-sign-in happy path, re-set idempotency, 15.7 conflict across multi-team membership
  - _Requirements: 15.1, 15.3, 15.4, 15.7_

### 7. Teams module (design §5.3, Req 2)

- [ ] 7.1 Implement invite-token generator
  - `crypto.randomBytes(32)` → base64url (43 chars); used for both team and game invite tokens
  - _Requirements: 2.2_

- [ ] 7.2 Implement `POST /teams` handler
  - Creates `teams` row with caller as `organizer_player_id`, generates invite token, inserts `team_memberships` for the creator
  - Returns team with `member_count = 1`
  - _Requirements: 2.1, 2.2_

- [ ] 7.3 Implement `POST /teams/join/:token` handler (design §4.3)
  - Transaction: lock team row FOR UPDATE, verify count < 8, check 15.7 Display_Name uniqueness against current members, insert membership
  - Return 404 `TEAM_INVITE_INVALID` for unknown token, 409 `TEAM_FULL` at 8 members, 409 `DISPLAY_NAME_TAKEN_IN_TEAM` on conflict
  - If already a member, no-op and 200 (Req 2.5)
  - _Requirements: 2.3, 2.4, 2.5, 15.7_

- [ ] 7.4 Implement `POST /teams/:id/leave` handler
  - Removes the caller's membership row; Organizer leaving is allowed in MLP (no ownership transfer required)
  - _Requirements: 2.8_

- [ ] 7.5 Implement `GET /teams/:id` handler
  - Returns team name, member_count, members[] (player_id, display_name, is_organizer), invite_link_token (members only)
  - Serializer sources from `players_public` — no emails
  - _Requirements: 2.6, 2.7_

- [ ] 7.6 Implement `GET /teams/mine` handler
  - Returns the list of teams the caller belongs to
  - _Requirements: 2.9_

- [ ] 7.7 Implement `POST /teams/:id/rotate-invite` endpoint *
  - Regenerates `invite_link_token` for organizers; schema supports rotation per design §9.4
  - Defer if skipping the optional rotation UX for MLP

- [ ]* 7.8 Unit tests for teams domain logic
  - Membership count enforcement, 15.7 uniqueness boundary, invite-token uniqueness
  - _Requirements: 2.4, 15.7_

- [ ]* 7.9 Integration tests for teams endpoints
  - Create, join up to 8, join-when-full error, join-as-existing-member no-op, leave, mine
  - _Requirements: 2.1, 2.3, 2.4, 2.5, 2.8, 2.9_

### 8. Question bank seeding pipeline (Req 5, Req 5b)

Question bank seeding is a real product project: ≥500 approved questions per category across 10 categories = ≥5,000 questions minimum (Req 5.1). Treat this as its own task group that can run in parallel with later work but blocks game creation until seeded.

- [ ] 8.1 Freeze the 10-category catalog (design open question 5)
  - Confirm exact strings for: Sports, History, Science, Pop Culture, Geography, Music, Film & TV, Food & Drink, Technology, Literature
  - Export as a TypeScript const `CATEGORY_CATALOG` in `packages/shared`
  - _Requirements: 5.1_

- [ ] 8.2 Design and document the question generation prompt
  - Author `db/seed/prompts/generate-questions.md` with explicit constraints: 2–6 choices, exactly one correct, ≤ 200 chars question text, no duplicates within a category
  - Include target of ≥ 500 questions per category for MLP launch, ≥ 600 for headroom
  - _Requirements: 5.1, 5.6_

- [ ] 8.3 Generate question candidates offline (out-of-runtime, Req 5.7, 5b)
  - Produce raw JSON files at `db/seed/raw/{category}.json` containing generated candidates for each of the 10 categories
  - This step is explicitly out-of-band; no runtime LLM dependency
  - _Requirements: 5.1, 5.7, 5b.2_

- [ ] 8.4 Build the automated validation script
  - Script at `db/seed/validate.ts` that loads each raw JSON and enforces: question_text length ≤ 200, 2 ≤ choices ≤ 6, exactly one `is_correct: true` per question, no intra-category duplicate question_text (case-insensitive), choice_text non-empty
  - Emits a validation report and exits non-zero on any failure
  - _Requirements: 5.6_

- [ ] 8.5 Sample-based human review workflow *
  - Script that samples N random questions per category from validated output for manual spot-checking, recording approvals in `db/seed/reviewed/{category}.json`
  - Optional for MLP if reviewer bandwidth is limited, but recommended

- [ ] 8.6 Build the import pipeline
  - Script at `db/seed/import.ts` that reads validated JSON, inserts into `questions` and `answer_choices` in a single transaction, sets `correct_answer_id` after choices are inserted
  - Wraps the whole batch in a transaction so the deferred trigger (task 2.8) validates everything at COMMIT
  - Idempotent: skips categories already at ≥ 500 unless `--force` is passed
  - _Requirements: 5.1, 5.6_

- [ ] 8.7 Implement bank smoke check
  - Script at `db/seed/smoke.ts` that asserts `COUNT(*) FROM questions WHERE category = X ≥ 500` for every category in `CATEGORY_CATALOG`
  - Wire into CI so a merged migration + seed always satisfies Req 5.1
  - _Requirements: 5.1_

- [ ]* 8.8 Unit tests for the validate script
  - Feed synthetic bad inputs (7 choices, 201-char text, zero correct, two correct) and assert each is caught
  - _Requirements: 5.6_

### 9. Games module (design §5.4, Req 3)

- [ ] 9.1 Implement `POST /games` handler
  - Validates caller is a member of `team_id` with ≥ 3 members; 422 `TEAM_TOO_SMALL_FOR_GAME` otherwise
  - Inserts `games` row with `state = Pending_Acceptance`, generated `game_invite_link_token`
  - _Requirements: 3.1, 3.2_

### 10. Game acceptance transaction (design §4.3, §6.1, §6.2)

The Game acceptance transaction is the most complex atomic operation in the system. It coordinates game state transition, roster validation, round creation, category/owner assignment, question selection, and email enqueueing in a single `REPEATABLE READ` transaction. Break it into discrete sub-tasks so each piece can be tested independently before being wired into the master handler.

- [ ] 10.1 Implement `lockTeamsAndValidate(gameId, acceptingTeamId, tx)`
  - Locks both `teams` rows FOR UPDATE
  - Asserts both sizes ≥ 3 (422 `TEAM_TOO_SMALL_FOR_GAME`)
  - Asserts `creating_team_id != accepting_team_id` (409 `CANNOT_PLAY_SELF`, Req 3.6)
  - _Requirements: 3.2, 3.5, 3.6_

- [ ] 10.2 Implement disjoint-roster check
  - Queries for any player_id present in both teams' `team_memberships`
  - On non-empty result, raises `ROSTER_OVERLAP` with `details.overlapping_players: [{ display_name, team_ids }]`
  - _Requirements: 3.8_

- [ ] 10.3 Implement caller eligibility resolution for acceptance
  - Given the caller, find all teams (≥ 3 members, not the creating team) they belong to
  - If exactly one → auto-select (Req 3.4); if zero → 422 `NO_ELIGIBLE_TEAM`; else require `team_id` in the request body
  - Return appropriate redirect hint if game is Active/Complete (409 `GAME_NOT_ACCEPTING`, Req 3.7)
  - _Requirements: 3.3, 3.4, 3.5, 3.7_

- [ ] 10.4 Implement category draw
  - `random.sample(CATEGORY_CATALOG, 5)` using a per-request crypto-seeded PRNG; returns 5 distinct categories in draw order
  - _Requirements: 4.2_

- [ ] 10.5 Implement per-team minimum-imbalance owner assignment (design §6.1)
  - Given a shuffled list of team members at game start and 5 rounds for that team:
    - Team size ≥ 5: pick 5 distinct owners; remaining members recorded as sitters (Req 4.4, 4.6)
    - Team size < 5: round-robin with `ceil(5 / team_size)` as the per-player cap (Req 4.5)
  - Returns `{ owners: [(round_number, player_id)], sitters: [player_id] }`
  - _Requirements: 4.3, 4.4, 4.5, 4.6, 11.4_

- [ ] 10.6 Implement round + round_owner + game_sitters persistence for a game
  - Inserts 10 `rounds` rows (5 per team), per-round `round_owners` row (role=original, is_active=true, window_ends_at = started_at + 12h), and `game_sitters` rows
  - _Requirements: 4.1, 4.7, 7.5 (window semantics)_

- [ ] 10.7 Implement question selection with 10-game lookback and progressive relaxation (design §6.2)
  - `selectQuestions(round, team, lookbackGames=10)` per the pseudocode:
    - Query team's 10 most recent `state = Complete` games (uses `ix_games_team_completed`)
    - Collect all `round_questions.question_id` across both teams' rounds in those games
    - `pool = questions in round.category − excluded`
    - While `|pool| < 5` and `lookback > 0`: decrement `lookback`, recompute excluded, recompute pool
    - If `|pool| < 5` even at `lookback = 0` and total category pool < 5: throw `QuestionBankExhausted` (rolls back the tx; Req 5.1 operational failure)
    - `random.sample(pool, 5)` → insert 5 `round_questions` rows
  - _Requirements: 5.2, 5.3, 5.4, 5.5, 5.6_

- [ ] 10.8 Implement operator alert for question bank exhaustion (design §11.1)
  - When `QuestionBankExhausted` is thrown, the tx rolls back, the caller receives 500 `QUESTION_BANK_EXHAUSTED`, and a Sentry-tagged alert + structured log `bank_exhausted category=X` fires
  - Wire via existing exception handler (task 4.3); include the failing category in the alert payload
  - _Requirements: 5.1, 5b.1_

- [ ] 10.9 Implement `round_assigned` email enqueue for each owner (design §6.1, Req 12.1)
  - Insert one `email_outbox` row per Original_Owner with `notification_type = round_assigned`, payload containing category, window end, round link
  - Check `notifications_sent` before enqueue for idempotency
  - _Requirements: 12.1_

- [ ] 10.10 Wire `POST /games/accept/:token` as the master orchestrator
  - Runs all of 10.1 → 10.9 inside a single `REPEATABLE READ` transaction
  - On success: transitions `games.state = Active`, sets `started_at = now`, returns 200
  - On any sub-step error: full rollback, game remains in `Pending_Acceptance`
  - _Requirements: 3.3, 3.8, 3.9, 4.1, 4.7, 5.2, 12.1_

- [ ] 10.11 Checkpoint — end-to-end acceptance path green
  - Ensure all acceptance-path unit + integration tests pass. Ask the user if questions arise.


- [ ]* 10.12 Unit tests for category draw
  - Assert 5 distinct categories, all from the catalog; run 10K iterations asserting uniform-ish distribution
  - _Requirements: 4.2_

- [ ]* 10.13 Unit tests for minimum-imbalance owner assignment
  - Example cases: team_size ∈ {3, 4, 5, 6, 7, 8}; assert per-player cap and sitter count invariants (Property 4 as examples)
  - _Requirements: 4.3, 4.4, 4.5, 4.6_

- [ ]* 10.14 Unit tests for question selection with lookback
  - Feed synthetic team histories where the exclusion set progressively shrinks the pool; assert correct relaxation steps and final selection
  - _Requirements: 5.3, 5.4_

- [ ]* 10.15 Integration tests for `POST /games` and `POST /games/accept/:token`
  - Full acceptance flow; distinct-teams, roster-overlap, team-too-small, no-eligible-team, cannot-play-self, already-accepted
  - _Requirements: 3.1, 3.2, 3.3, 3.4, 3.5, 3.6, 3.7, 3.8_

### 11. Rounds module (design §5.5, Req 6)

- [ ] 11.1 Implement round-owner authorization helper
  - `requireActiveOwner(roundId, playerId)` returns the active `round_owners` row or throws 403 `NOT_ROUND_OWNER`; used as preHandler on every round endpoint
  - Non-owners, even same-team members, receive 403 (Req 8.3)
  - _Requirements: 6, 8.3_

- [ ] 11.2 Implement `GET /rounds/:id` preview handler
  - Returns category, question_count=5, question_timer_seconds=30, answer_window_ends_at, can_start (bool), cannot_start_reason
  - `cannot_start_reason = INSUFFICIENT_WINDOW` when `answer_window_ends_at - NOW() < 150s` (Req 6.2)
  - Returns 410 `ROUND_UNAVAILABLE` if round state is Not_Started and window already elapsed (Req 6.10)
  - _Requirements: 6.1, 6.2, 6.10_

- [ ] 11.3 Implement `POST /rounds/:id/start` handler
  - Transaction: lock round row, verify state ∈ {Not_Started, Sub_Needed}, verify `answer_window_ends_at - NOW() ≥ 150s`
  - On insufficient window: return 409 `INSUFFICIENT_WINDOW`, mark `state = Sub_Needed` so the worker picks it up in the next sweep (Req 6.2 → Req 7.1)
  - On sufficient window: set `state = In_Progress`, set `started_at = now`
  - _Requirements: 6.2, 6.3_

- [ ] 11.4 Implement `GET /rounds/:id/questions/:position` handler
  - Validates `state = In_Progress`, position 1..5, and the prior position has been resolved (Req 6.11 — no jumping ahead)
  - On first call for a position: inserts `round_answers` stub row with `timer_started_at = NOW()`
  - On subsequent calls: returns the stored `timer_started_at` and a computed `seconds_remaining`; the client cannot reset the timer by refreshing
  - Returns question_text, choices[] (id, text), timer_started_at, seconds_remaining
  - Applies a 2-second server-side grace per design open question 3
  - _Requirements: 6.4, 6.7, 6.11_

- [ ] 11.5 Implement `POST /rounds/:id/answers` handler
  - Validates position matches the position currently in-flight; rejects later positions until current is resolved
  - Transaction: lock round row, verify state = In_Progress, verify `NOW() ≤ timer_started_at + 30s + 2s grace`
  - On expired timer: return 409 `TIMER_EXPIRED`; the stored row already has `timer_expired=true, is_correct=false` written by the worker sweeper or inline on next question fetch
  - On valid submit: compute `is_correct` against `questions.correct_answer_id`; update the `round_answers` row
  - Response: `{ accepted: true, next_position, round_complete }` — never includes `is_correct`
  - If all 5 positions resolved: transition round to `state = Complete`, set `completed_at = NOW()`, invoke `maybeCompleteGame` (task 14.1) inline
  - _Requirements: 6.5, 6.6, 6.8, 6.11, 8.3_

- [ ] 11.6 Implement idempotent answer-submit semantics (design §11.2)
  - Duplicate POST for same (round_id, position) with a different answer_id: 409 `ALREADY_ANSWERED`
  - Byte-identical refresh: 200 with stored answer
  - _Requirements: 6.11_

- [ ]* 11.7 Unit tests for timer-window math
  - `answer_window_ends_at - NOW()` boundary exactly at 150s, 149s, 151s; confirm Start accept/reject
  - `timer_started_at + 30s + 2s grace` boundary at 32.00s, 31.99s, 32.01s
  - _Requirements: 6.2, 6.3, 6.5, 6.6_

- [ ]* 11.8 Integration tests for round endpoints
  - Preview on fresh round, start with sufficient window, start with insufficient window (and assert state → Sub_Needed), question fetch starts timer, answer submit happy path, timer-expired-at-30s, duplicate submit, non-owner 403, expired-window preview 410
  - _Requirements: 6.1, 6.2, 6.3, 6.4, 6.5, 6.6, 6.10, 6.11, 8.3_

### 12. Scoring module (Req 10)

- [ ] 12.1 Implement pure `effectiveRoundScore(round)` function
  - Returns `roundScore` if `round.state = Complete` and `is_substitute = false`
  - Returns `roundScore × 0.8` if `round.state = Complete` and `is_substitute = true`
  - Returns `0.0` if `round.state = Forfeited`
  - Throws if called on non-terminal state
  - _Requirements: 7.6, 7.7, 7.8, 10.2, 10.4_

- [ ] 12.2 Implement pure `roundScore(round)` function
  - Counts `round_answers.is_correct = true` for round, divides by 5; asserts result ∈ [0.0, 1.0]
  - _Requirements: 10.1, 10.3_

- [ ] 12.3 Implement pure `gameScore(team, rounds)` function
  - Sums 5 `effectiveRoundScore` values for the team; asserts result ∈ [0.0, 5.0]
  - No team-size term anywhere in the formula (Req 14.1)
  - _Requirements: 9.3, 10.5, 11.3, 14.1_

- [ ]* 12.4 Unit tests for scoring functions with exhaustive example grid
  - (0..5 correct) × (original, substitute, forfeited) × floating-point tolerance at 1e-9
  - _Requirements: 10.1, 10.2, 10.3, 10.4, 10.5_

### 13. Status module (design §5.4, Req 8)

The discriminated-union status response is an architectural invariant (design §5.4). Two entirely separate serializer functions — `serializeActive` and `serializeComplete` — must share no code path where a conditional `if (complete) include scores` could be flipped. Treat the two serializers as separate tasks with a dedicated test that asserts the separation.

- [ ] 13.1 Implement `serializePending(game)` function
  - Returns `GameStatusPending` shape; no rounds exist yet
  - _Requirements: 3.1_

- [ ] 13.2 Implement `serializeActive(game, callerPlayerId)` function (design §5.4)
  - Returns `GameStatusActive` shape per the discriminated union
  - Per-round `status` computed from round state + `answer_window_ends_at` (Req 8.1 mapping)
  - `current_owner` populated only for rounds on the caller's team; opponent team rounds get `current_owner: null`
  - Must NOT read from `round_answers`, `questions`, `answer_choices`, or any score-bearing source — add a lint/test to assert this
  - _Requirements: 8.1, 8.2, 8.3_

- [ ] 13.3 Implement `serializeComplete(game)` function (design §5.4)
  - Returns `GameStatusComplete` shape
  - Computes `round_score`, `effective_round_score`, `game_score`, `outcome`; includes per-question correctness and full question text
  - _Requirements: 8.4, 9.3, 9.4, 9.5_

- [ ] 13.4 Implement `GET /games/:id/status` top-level handler
  - Verifies caller is a member of either participating team (else 403 `NOT_GAME_PARTICIPANT`, Req 8.5)
  - Branches on `games.state` ONCE at the top and calls exactly one serializer; no shared conditional after that
  - _Requirements: 8.1, 8.2, 8.3, 8.4, 8.5_

- [ ] 13.5 Implement `GET /games/:id/results` as a safe alias
  - Calls the Complete serializer; returns 409 `GAME_NOT_COMPLETE` if state ≠ Complete
  - _Requirements: 8_

- [ ]* 13.6 Unit test: Active and Complete serializers share no code path
  - Imports both serializers; walks their call graph (via a tagged marker on any shared helper that touches scoring) and asserts no helper in the Active call graph reads score-bearing tables
  - This is the architectural guarantee behind the hard-lock (Req 8.2)
  - _Requirements: 8.2_

- [ ]* 13.7 Unit tests for each serializer with example scenarios
  - Pending: basic shape
  - Active: every per-round status variant (not_started, in_progress, sub_needed, forfeited, complete), opponent-view vs own-view `current_owner` masking
  - Complete: tie, team_a_win, team_b_win, forfeited round payload shape
  - _Requirements: 8.1, 8.4, 9.4, 9.5_

- [ ]* 13.8 Integration tests for status endpoints
  - Non-participant 403; active game response never contains score fields (assert by JSON-Schema negation); complete game includes full scores
  - _Requirements: 8.2, 8.3, 8.5_

### 14. Game completion detection (design §6.4, Req 9)

- [ ] 14.1 Implement `maybeCompleteGame(gameId, tx)` inline trigger
  - Called inline from every round state transition that could be terminal (task 11.5 on round complete; task 15.2 on forfeit)
  - If not all 10 rounds terminal → return
  - If all terminal: set `games.state = Complete`, `completed_at = NOW()`, enqueue `game_complete_digest` emails for every member of both teams (respecting `notifications_sent` idempotency)
  - _Requirements: 9.1, 9.2, 12.5_

- [ ] 14.2 Implement win/loss/tie aggregation read query
  - `GET /teams/:id` (or a dedicated `/teams/:id/record`) returns `{ wins, losses, ties }` computed on the fly from completed games' game_scores
  - _Requirements: 9.4, 9.5, 9.6_

- [ ]* 14.3 Unit tests for `maybeCompleteGame`
  - 9 rounds terminal + 1 not → no transition; 10 terminal → transition + digest enqueue; re-entrant call after transition → no duplicate emails (relies on `notifications_sent` PK)
  - _Requirements: 9.1, 9.2, 12.5_

### 15. Background worker (design §7)

- [ ] 15.1 Bootstrap worker process in `apps/api/src/worker.ts`
  - Separate entrypoint from the HTTP server but shares the same module code
  - Uses `node-cron` to schedule four sweepers at their configured cadences
  - Graceful shutdown, exits non-zero on Sentry-reported fatal errors

- [ ] 15.2 Implement window-expiration sweeper (design §7, §6.3)
  - Query: `SELECT ... FROM rounds WHERE state IN ('Not_Started','Sub_Needed') AND answer_window_ends_at <= NOW() ORDER BY team_id, round_number FOR UPDATE SKIP LOCKED LIMIT 500`
  - Group rows by team_id, process in ascending round_number order
  - For each: call `assignSubstitute(round, NOW())` — if no eligible substitute, transition to `Forfeited` and mark `Effective_Round_Score = 0`
  - After each assignment, call `maybeCompleteGame(gameId, tx)` inline
  - Cadence: every 1 min
  - _Requirements: 7.1, 7.2, 7.3, 7.4, 7.6, 7.7, 7.9_

- [ ] 15.3 Implement `assignSubstitute(round, now)` (design §6.3)
  - Compute eligible set: team members whose `player_id ∉ active_owner_player_ids_in_game(game_id)` and `≠ round.current_active_owner.player_id` (Req 7.3)
  - Pick uniformly at random from eligible
  - Deactivate current active owner (`round_owners.is_active = false`), insert new `round_owners` row (role=substitute, is_active=true, window_ends_at=now+12h, assigned_at=now)
  - Update `rounds.answer_window_ends_at = now + 24h`, keep `state = Sub_Needed` until the sub hits Start
  - Enqueue `substitute_needed` email (Req 12.4)
  - Enforce at-most-one-substitute-per-round via the `uq_round_owners_one_sub` partial unique index from task 2.6
  - _Requirements: 7.2, 7.3, 7.5, 7.9, 12.4_

- [ ] 15.4 Implement original-owner-left-team detection (Req 4.8)
  - Extend window-expiration sweeper (or add a separate low-frequency sweeper) to detect rounds whose active owner is no longer in `team_memberships` for the round's team; transition such rounds to `Sub_Needed` and run `assignSubstitute`
  - _Requirements: 4.8_

- [ ] 15.5 Implement 6-hour reminder scheduler (design §7, Req 12.2)
  - Every 5 min: find active owners whose `window_ends_at ∈ [NOW() + 5h55m, NOW() + 6h05m]` and round state ≠ Complete
  - Enqueue `reminder_6h` only if `notifications_sent` has no matching row
  - _Requirements: 12.2_

- [ ] 15.6 Implement 2-hour reminder scheduler (design §7, Req 12.3)
  - Same shape as 15.5 with the [1h55m, 2h05m] window and `reminder_2h` type
  - _Requirements: 12.3_

- [ ] 15.7 Implement email dispatcher (design §7)
  - Every 30 s: `SELECT ... FROM email_outbox WHERE sent_at IS NULL ORDER BY enqueued_at LIMIT 100 FOR UPDATE SKIP LOCKED`
  - For each: render template (task group 16), call Resend `send`, on success set `sent_at = NOW()` and insert/verify `notifications_sent` row; on 5xx increment `send_attempts`, set `last_error`, retry per exponential backoff schedule (1m, 2m, 4m, 8m, 16m)
  - After 5 failures: stop retrying, record final `last_error`, fire operator alert
  - _Requirements: 12_

- [ ]* 15.8 Integration tests for sweepers
  - Clock-override fixture using a `NOW()` SQL function shim; drive each sweeper through happy path + idempotency (re-run same sweep, assert no duplicate emails) + forfeit path + simultaneous expiration on same team (Req 7.4 sequential ordering)
  - _Requirements: 7.1, 7.2, 7.3, 7.4, 7.6, 7.7, 12.1, 12.2, 12.3, 12.4_


### 16. Email templates (Req 12)

- [ ] 16.1 Wire React Email into `apps/api`
  - Install `@react-email/components` and `resend`; create `apps/api/src/notifications/templates/` with a shared layout wrapper
  - _Requirements: 12_

- [ ] 16.2 Build `round_assigned` template
  - Subject: "Your trivia round is ready: {category}"
  - Body: category, answer_window_ends_at, direct round link, Display_Name greeting (Req 15.6)
  - _Requirements: 12.1, 15.6_

- [ ] 16.3 Build `reminder_6h` template
  - Subject: "6 hours left on your trivia round"
  - Body: category, remaining hours, round link
  - _Requirements: 12.2_

- [ ] 16.4 Build `reminder_2h` template
  - Subject: "2 hours left — don't forget your round"
  - Body: same as 16.3 with urgency copy
  - _Requirements: 12.3_

- [ ] 16.5 Build `substitute_needed` template
  - Subject: "You're up: trivia round needs a substitute"
  - Body: category, new 12h window, round link
  - _Requirements: 12.4_

- [ ] 16.6 Build `game_complete_digest` template
  - Subject: "Game results: {your_team} vs {opponent_team}"
  - Body: both teams' game_scores, per-round effective_round_scores, win/loss/tie outcome
  - For Forfeited rounds, render "No one was able to complete this round." per design open question 4
  - _Requirements: 9.3, 9.4, 9.5, 12.5_

- [ ] 16.7 Wire templates into the dispatcher (task 15.7)
  - Dispatcher branches on `notification_type` to select the right template and extract payload fields

- [ ]* 16.8 Email smoke tests via Resend test mode
  - One happy-path send per notification type; assert Resend returns a message id and that `notifications_sent` gains exactly one row
  - _Requirements: 12_

### 17. Frontend: scaffolding and auth (design §13)

- [ ] 17.1 Bootstrap React + Vite + Tailwind in `apps/web`
  - Create `apps/web` with Vite React-TS template; configure Tailwind with a WCAG-audited color token set (design §13 rationale)
  - Set up path aliases to `packages/shared`
  - _Requirements: 13_

- [ ] 17.2 Install Supabase JS client and implement auth provider
  - `createClient(SUPABASE_URL, SUPABASE_ANON_KEY)`; React context exposing session + `signInWithOtp`, `signOut`
  - _Requirements: 1_

- [ ] 17.3 Implement sign-in page
  - Email input, submit posts to Supabase Auth; confirmation view within 5-second budget (Req 1.1)
  - Handle Supabase error for expired/invalid link on callback route with "Request a new link" CTA (Req 1.3)
  - _Requirements: 1.1, 1.3_

- [ ] 17.4 Implement `/auth/callback` route
  - Exchanges code, stores session, redirects to onboarding (if `display_name` is null) or home
  - _Requirements: 1.2_

- [ ] 17.5 Implement Display_Name onboarding page
  - Forced step on first sign-in before any Team or Game surface (Req 15.1)
  - Submit calls `POST /players`; 400 `DISPLAY_NAME_INVALID` → inline error, 409 `DISPLAY_NAME_TAKEN_IN_TEAM` → prompt
  - _Requirements: 15.1, 15.2_

- [ ] 17.6 Implement sign-out button
  - Calls `POST /auth/sign-out`; clears client session; routes to sign-in
  - _Requirements: 1.5_

- [ ] 17.7 Implement authenticated API client wrapper
  - Attaches `Authorization: Bearer <jwt>`; centralizes error-envelope parsing and maps `ErrorCode` → user-facing copy
  - Surfaces `UNAUTHENTICATED` as auto-redirect to sign-in

### 18. Frontend: teams and games surfaces

- [ ] 18.1 Implement teams list page (`/teams`)
  - Calls `GET /teams/mine`; shows team name + member_count; "Create team" + "Join via link" actions
  - _Requirements: 2.9_

- [ ] 18.2 Implement create-team flow
  - Form → `POST /teams` → navigates to team detail; displays copyable invite link
  - _Requirements: 2.1, 2.2_

- [ ] 18.3 Implement team detail page (`/teams/:id`)
  - Member list (Display_Name only, no email), member_count, invite link, "Leave team" CTA (Req 2.6, 2.7, 2.8)
  - _Requirements: 2.6, 2.7, 2.8_

- [ ] 18.4 Implement team join page (`/teams/join/:token`)
  - On load: `POST /teams/join/:token`; handles 404 (invalid), 409 `TEAM_FULL`, 409 `DISPLAY_NAME_TAKEN_IN_TEAM`, already-member no-op redirect
  - _Requirements: 2.3, 2.4, 2.5, 15.7_

- [ ] 18.5 Implement profile page (`/profile`)
  - Displays current Display_Name; edit form calls `PATCH /players/me` with length + uniqueness validation
  - _Requirements: 15.3, 15.4, 15.7_

- [ ] 18.6 Implement games list and create-game flow
  - Lists all games caller is involved in, grouped by state; "Create game" selector of caller's eligible teams (≥ 3 members) → `POST /games`
  - Display the Req 14.2 size-difference note on the creation screen
  - _Requirements: 3.1, 3.2, 14.2_

- [ ] 18.7 Implement accept-game flow (`/games/accept/:token`)
  - `POST /games/accept/:token` with team selector when multiple eligible; auto-select when exactly one (Req 3.4)
  - Handle all error codes: `NO_ELIGIBLE_TEAM`, `CANNOT_PLAY_SELF`, `GAME_NOT_ACCEPTING`, `ROSTER_OVERLAP` (show overlapping Display_Names), `TEAM_TOO_SMALL_FOR_GAME`
  - Display Req 14.2 note
  - _Requirements: 3.3, 3.4, 3.5, 3.6, 3.7, 3.8, 14.2_

### 19. Frontend: game status and round play

- [ ] 19.1 Implement game status page (`/games/:id`)
  - Fetches `GET /games/:id/status`; renders one of three views by branching on `state` (discriminated union)
  - Pending_Acceptance view: "Waiting for opponent" + share-link affordance
  - _Requirements: 3, 8_

- [ ] 19.2 Implement Active status view (design §5.4)
  - Two columns (your team, opponent team); per-round rows show category + status badge (`Not Started`, `In Progress (X hours left)`, `Sub Needed`, `Forfeited`, `Complete`)
  - No score data anywhere on this view
  - Current owner name shown only for your-team rounds; opponent-team rounds show only the status
  - If caller is the current active owner of a round, show "Open round" CTA → `/rounds/:id`
  - _Requirements: 8.1, 8.2, 8.3_

- [ ] 19.3 Implement Complete status view
  - Renders `GameStatusComplete`: per-team game_score, per-round effective_round_score + completed_by, per-question correctness
  - Win/loss/tie banner
  - _Requirements: 8.4, 9.4, 9.5_

- [ ] 19.4 Implement round preview screen (`/rounds/:id`)
  - `GET /rounds/:id`; shows category, 5 questions, 30-second timer duration, explicit "Start Round" CTA
  - Do NOT display any question content before Start is activated (Req 6.1)
  - Disable Start CTA and show "Not enough time remaining" message when `can_start = false` / `cannot_start_reason = INSUFFICIENT_WINDOW` (Req 6.2)
  - On expired window (410 `ROUND_UNAVAILABLE`): show "Round is no longer available"
  - _Requirements: 6.1, 6.2, 6.10_

- [ ] 19.5 Implement round play flow
  - On Start: `POST /rounds/:id/start`, then fetch each question in turn via `GET /rounds/:id/questions/:position`
  - Client countdown seeded from server `timer_started_at` + `seconds_remaining`; never trust client clock to reset
  - Submit via `POST /rounds/:id/answers`; response never reveals correctness (Req 8.3)
  - After last submission: navigate to "Round submitted — results at game end" screen
  - Prevent back-nav from re-attempting already-started questions (Req 6.11)
  - _Requirements: 6.3, 6.4, 6.5, 6.6, 6.7, 6.8, 6.11, 8.3_

- [ ] 19.6 Implement team record display
  - Team detail page shows wins/losses/ties aggregated across completed games
  - _Requirements: 9.6_

### 20. Property-based tests (design §10, §12.2)

There are exactly 18 property-based tests, one per Correctness Property from design Section 10. All tests live under `tests/pbt/` and use `fast-check` with `numRuns: 100` minimum per property. Each test file is tagged in code per the exact format in design §12.2. Tests drive the in-memory test harness described in design §12.2 rather than hitting the real database, to keep runtime under a few minutes per full suite.

- [ ] 20.1 Build the in-memory PBT harness (design §12.2)
  - Implements the same module contracts as production API but against an in-memory store
  - Exposes deterministic-on-demand seeding so fast-check can shrink failures
  - _Requirements: §12.2 harness_

- [ ] 20.2 Build shared fast-check generators (design §12.2)
  - `teamRosterArb(minSize, maxSize)` — list of player IDs of chosen size
  - `twoTeamsArb(overlapSize)` — two rosters with a known overlap count
  - `answerPatternArb()` — 5-element vector of `{ correct, timer_expired }`
  - `gameLifecycleArb()` — scripted, temporally-consistent sequence of events
  - `displayNameArb()` — strings including empty, whitespace, length-30, length-31, Unicode, mixed case
  - `questionHistoryArb(teamGames)` — team's completed-game history with question IDs

- [ ] 20.3 Property 1: Roster Disjoint
  - **Property statement**: For any two Teams each of size ≥ 3 with possibly overlapping memberships, Game acceptance succeeds if and only if the two rosters share no Player; if any Player is on both Teams at the moment of acceptance, acceptance is rejected with an error identifying the overlapping Player(s).
  - **Validates**: Requirements 3.8
  - **Generators needed**: `twoTeamsArb(overlapSize)` with overlap ∈ [0, min(|A|, |B|)]
  - **Test file**: `tests/pbt/property-1-roster-disjoint.test.ts`
  - **Code tag**: `// Feature: trivia-squads-mlp, Property 1: Roster Disjoint`
  - **Minimum iterations**: 100

- [ ] 20.4 Property 2: Game Creates Correct Round Structure
  - **Property statement**: For any Game that successfully transitions to Game_State Active, each of the two Teams has exactly 5 Rounds (10 Rounds total per Game), and each Round has exactly 5 Questions drawn for it, each Question pairwise distinct within a Round.
  - **Validates**: Requirements 4.1, 5.2, 5.6, 11.1
  - **Generators needed**: `teamRosterArb(3, 8)` for both teams, seeded question bank with ≥ 5 per category
  - **Test file**: `tests/pbt/property-2-round-structure.test.ts`
  - **Code tag**: `// Feature: trivia-squads-mlp, Property 2: Game Creates Correct Round Structure`
  - **Minimum iterations**: 100

- [ ] 20.5 Property 3: Category Distinctness
  - **Property statement**: For any Game start, the 5 Categories assigned to the Game's Rounds are pairwise distinct elements of the Category_Catalog (one Category per round number, applied to both Teams' same-numbered Rounds).
  - **Validates**: Requirements 4.2
  - **Generators needed**: `teamRosterArb(3, 8)` for both teams
  - **Test file**: `tests/pbt/property-3-category-distinctness.test.ts`
  - **Code tag**: `// Feature: trivia-squads-mlp, Property 3: Category Distinctness`
  - **Minimum iterations**: 100

- [ ] 20.6 Property 4: Owner Assignment Minimizes Imbalance
  - **Property statement**: For any Team of size n ∈ [3, 8] at Game start, the assignment of 5 Rounds to Team members satisfies: (a) every assigned Player is a member of the Team at Game start, (b) the maximum number of Rounds assigned to any single Player is `ceil(5 / min(n, 5))` (equivalently: 1 for n ≥ 5, 2 for n ∈ [3, 4]), (c) for n ≥ 5 the 5 owners are pairwise distinct, and (d) for n > 5 exactly n − 5 Players are recorded as sitters and the remaining 5 are owners.
  - **Validates**: Requirements 4.3, 4.4, 4.5, 4.6, 11.4
  - **Generators needed**: `teamRosterArb(3, 8)`
  - **Test file**: `tests/pbt/property-4-owner-assignment.test.ts`
  - **Code tag**: `// Feature: trivia-squads-mlp, Property 4: Owner Assignment Minimizes Imbalance`
  - **Minimum iterations**: 100

- [ ] 20.7 Property 5: Question Reuse Policy
  - **Property statement**: For any Game start for a given Team, the 25 Questions selected across that Team's 5 Rounds do not appear in any of the Team's 10 most recent Completed Games, except when the eligible pool in a Category contains fewer than 5 Questions, in which case the lookback window is progressively relaxed (10 → 9 → ... → 0) to the smallest value at which at least 5 eligible Questions exist, and the reuse constraint holds at that relaxed lookback.
  - **Validates**: Requirements 5.3, 5.4
  - **Generators needed**: `questionHistoryArb(teamGames)` with controlled overlap, category-pool-size arb
  - **Test file**: `tests/pbt/property-5-question-reuse.test.ts`
  - **Code tag**: `// Feature: trivia-squads-mlp, Property 5: Question Reuse Policy`
  - **Minimum iterations**: 100

- [ ] 20.8 Property 6: Round Question Snapshot Is Stable
  - **Property statement**: For any Round that has been created as part of a Game start, repeated fetches of the Round's Question set return the identical 5 Questions (same IDs, same positions) for the lifetime of the Round.
  - **Validates**: Requirements 5.5
  - **Generators needed**: `teamRosterArb(3, 8)`, arbitrary fetch counts
  - **Test file**: `tests/pbt/property-6-round-questions-stable.test.ts`
  - **Code tag**: `// Feature: trivia-squads-mlp, Property 6: Round Question Snapshot Is Stable`
  - **Minimum iterations**: 100

- [ ] 20.9 Property 7: Start-Window Boundary
  - **Property statement**: For any Round in Round_State Not_Started or Sub_Needed, an attempt by the current Round_Owner to activate "Start Round" succeeds (transitions the Round to In_Progress) if and only if the remaining Answer_Window at the moment of the request is ≥ 2 minutes 30 seconds (150 seconds). If the remaining window is less than 150 seconds, Start is rejected and the Round transitions to Sub_Needed (triggering the substitute flow).
  - **Validates**: Requirements 6.2, 6.3
  - **Generators needed**: arbitrary `remainingSeconds` ∈ [0, 24*3600], arbitrary initial state ∈ {Not_Started, Sub_Needed}
  - **Test file**: `tests/pbt/property-7-start-window-boundary.test.ts`
  - **Code tag**: `// Feature: trivia-squads-mlp, Property 7: Start-Window Boundary`
  - **Minimum iterations**: 100

- [ ] 20.10 Property 8: In-Progress Only Completes
  - **Property statement**: For any Round that reaches Round_State In_Progress, the only terminal state that Round ever reaches is Complete — regardless of whether the Answer_Window elapses during play. Additionally, once a Question's Question_Timer has started, no later submission for that position can change the recorded answer or correctness after the timer expires.
  - **Validates**: Requirements 6.9, 6.11
  - **Generators needed**: `gameLifecycleArb()` scenarios that drive a round into In_Progress then attempt to expire / re-submit
  - **Test file**: `tests/pbt/property-8-in-progress-only-completes.test.ts`
  - **Code tag**: `// Feature: trivia-squads-mlp, Property 8: In-Progress Only Completes`
  - **Minimum iterations**: 100

- [ ] 20.11 Property 9: Round Ownership Invariant
  - **Property statement**: For any Round at any point in its lifecycle, there is exactly one `round_owners` row with `is_active = TRUE`, that row's `window_ends_at = assigned_at + 12h`, and across the Round's entire history at most one `round_owners` row has `role = 'substitute'`.
  - **Validates**: Requirements 4.7, 7.5, 7.9, 11.2
  - **Generators needed**: `gameLifecycleArb()` including substitute assignment and expiration events
  - **Test file**: `tests/pbt/property-9-round-ownership-invariant.test.ts`
  - **Code tag**: `// Feature: trivia-squads-mlp, Property 9: Round Ownership Invariant`
  - **Minimum iterations**: 100

- [ ] 20.12 Property 10: Substitute Eligibility
  - **Property statement**: For any substitute assignment event in a Game, (a) the selected Active_Substitute is a member of the Round's Team, (b) at the moment of selection the Active_Substitute was not the current active Round_Owner of the Round and was not the current active Round_Owner of any other Round in the Game, and (c) when multiple Rounds on the same Team expire simultaneously, substitutes are assigned sequentially in ascending Round number order, with eligibility re-evaluated against the updated ownership state between each assignment.
  - **Validates**: Requirements 7.2, 7.3, 7.4
  - **Generators needed**: `gameLifecycleArb()` with simultaneous multi-round expiration on the same team
  - **Test file**: `tests/pbt/property-10-substitute-eligibility.test.ts`
  - **Code tag**: `// Feature: trivia-squads-mlp, Property 10: Substitute Eligibility`
  - **Minimum iterations**: 50 (full-lifecycle simulation; documented per design §12.2 allowance)

- [ ] 20.13 Property 11: Scoring Formula and Bounds
  - **Property statement**: For any completed Game, every Round's scoring satisfies the full set of bounds and formulas: Round_Score = (count of correct answers) / 5, Round_Score ∈ [0.0, 1.0]; Effective_Round_Score equals Round_Score for Original_Owner, Round_Score × 0.8 for Active_Substitute, 0 for Forfeited, Effective_Round_Score ∈ [0.0, 1.0]; Game_Score equals the sum of 5 Effective_Round_Scores within a 1e-9 tolerance and ∈ [0.0, 5.0]; no term references Team size.
  - **Validates**: Requirements 6.8, 7.6, 7.7, 7.8, 9.3, 10.1, 10.2, 10.3, 10.4, 10.5, 11.3, 14.1
  - **Generators needed**: `answerPatternArb()` per round, per-round owner role ∈ {original, substitute, forfeited}, `teamRosterArb(3, 8)`
  - **Test file**: `tests/pbt/property-11-scoring-formula-and-bounds.test.ts`
  - **Code tag**: `// Feature: trivia-squads-mlp, Property 11: Scoring Formula and Bounds`
  - **Minimum iterations**: 100

- [ ] 20.14 Property 12: Hard-Lock Integrity
  - **Property statement**: For any API request served while the containing Game is in Game_State Active, the response body contains no field derivable from Round_Score, Effective_Round_Score, Game_Score, per-Question correctness, or a submitted answer value, and the response body contains no Question text or answer-choice content for any Round whose current active Round_Owner is not the calling Player.
  - **Validates**: Requirements 8.2, 8.3
  - **Generators needed**: `gameLifecycleArb()` with arbitrary calling-player ∈ {team member, non-team member, active round owner, sitter}; arbitrary endpoint from {status, round preview, round questions, round answers response}
  - **Test file**: `tests/pbt/property-12-hard-lock-integrity.test.ts`
  - **Code tag**: `// Feature: trivia-squads-mlp, Property 12: Hard-Lock Integrity`
  - **Minimum iterations**: 100

- [ ] 20.15 Property 13: Email Privacy
  - **Property statement**: For any API response and for any Notification_Service email sent to a recipient who is not the subject Player, the message contains no other Player's email address; Players are identified in all cross-Player surfaces only by Display_Name.
  - **Validates**: Requirements 2.7, 15.5
  - **Generators needed**: arbitrary team/game configs + `displayNameArb()`; harness scans all serialized responses and outbox payloads for anything matching the email pattern of any non-subject player
  - **Test file**: `tests/pbt/property-13-email-privacy.test.ts`
  - **Code tag**: `// Feature: trivia-squads-mlp, Property 13: Email Privacy`
  - **Minimum iterations**: 100

- [ ] 20.16 Property 14: Display Name Constraints
  - **Property statement**: For any proposed Display_Name string s, the System accepts s as a valid Display_Name (on creation or update) if and only if 1 ≤ char_length(s) ≤ 30, and for any Team at any point in its lifetime, no two current members of that Team share the same Display_Name.
  - **Validates**: Requirements 15.2, 15.3, 15.7
  - **Generators needed**: `displayNameArb()`, `teamRosterArb(1, 8)`
  - **Test file**: `tests/pbt/property-14-display-name-constraints.test.ts`
  - **Code tag**: `// Feature: trivia-squads-mlp, Property 14: Display Name Constraints`
  - **Minimum iterations**: 100

- [ ] 20.17 Property 15: Display Name Propagation
  - **Property statement**: For any Player who updates their Display_Name to a valid new value, all subsequent Team member list responses, Game status/results responses, and Game Complete Digest emails that reference that Player use the new Display_Name.
  - **Validates**: Requirements 15.4
  - **Generators needed**: `gameLifecycleArb()` interleaved with Display_Name update events
  - **Test file**: `tests/pbt/property-15-display-name-propagation.test.ts`
  - **Code tag**: `// Feature: trivia-squads-mlp, Property 15: Display Name Propagation`
  - **Minimum iterations**: 100

- [ ] 20.18 Property 16: Membership Round-Trip
  - **Property statement**: For any Team and any authenticated Player, (a) opening a valid Team_Invite_Link when already a member is a no-op on the membership set, and (b) joining a Team and then leaving the Team restores the original membership set.
  - **Validates**: Requirements 2.5, 2.8
  - **Generators needed**: `teamRosterArb(1, 7)`, arbitrary sequence of join/leave events
  - **Test file**: `tests/pbt/property-16-membership-round-trip.test.ts`
  - **Code tag**: `// Feature: trivia-squads-mlp, Property 16: Membership Round-Trip`
  - **Minimum iterations**: 100

- [ ] 20.19 Property 17: Notification Idempotency and Coverage
  - **Property statement**: For any Game lifecycle event that triggers email, the `notifications_sent` ledger contains exactly one matching row per (Player, Game, Round, notification_type) tuple — regardless of how many times the relevant sweeper or dispatcher runs — and coverage is complete: every Original_Owner receives a `round_assigned` on Game start, every Active_Substitute receives a `substitute_needed` on assignment, and every member of both Teams receives a `game_complete_digest` on Game completion.
  - **Validates**: Requirements 12.1, 12.4, 12.5
  - **Generators needed**: `gameLifecycleArb()` with arbitrary sweeper re-run counts (1..10)
  - **Test file**: `tests/pbt/property-17-notification-idempotency-coverage.test.ts`
  - **Code tag**: `// Feature: trivia-squads-mlp, Property 17: Notification Idempotency and Coverage`
  - **Minimum iterations**: 100

- [ ] 20.20 Property 18: Game Completion Trigger
  - **Property statement**: For any Game, as soon as all 10 of its Rounds have reached a terminal state (Complete or Forfeited), the Game transitions to Game_State Complete, `completed_at` is set, and `Game_Score` is computed for each Team before any post-completion API response is served.
  - **Validates**: Requirements 9.1, 9.2
  - **Generators needed**: `gameLifecycleArb()` driving all 10 rounds to terminal state via arbitrary mix of Complete/Forfeited paths
  - **Test file**: `tests/pbt/property-18-game-completion-trigger.test.ts`
  - **Code tag**: `// Feature: trivia-squads-mlp, Property 18: Game Completion Trigger`
  - **Minimum iterations**: 100

- [ ] 20.21 Checkpoint — all 18 property-based tests green
  - Run `pnpm test:pbt`; every property passes at its minimum iteration count. Ask the user if questions arise.


### 21. Integration and E2E tests (design §12.4)

- [ ] 21.1 Set up testcontainers Postgres harness for integration tests
  - Vitest `globalSetup` spins up Postgres 15, runs migrations + a minimal question-bank seed (≥ 5/category for fast tests)
  - Per-test database truncation between cases
  - Supabase JWT verification is mocked to accept a test-only signing key

- [ ] 21.2 Clock-override fixture for background-job tests
  - SQL function `set_test_now(ts)` + `get_test_now()` that sweepers call instead of `NOW()` under test
  - Document usage in `tests/integration/README.md`

- [ ] 21.3 Playwright setup and happy-path E2E flow
  - Install Playwright, configure browsers, add `pnpm e2e`
  - Scenario: two test users, create team A (3 players), create team B (3 players), create game, accept game, each owner plays their round, assert completion screen renders with scores
  - _Requirements: full happy-path coverage_

- [ ] 21.4 Second Playwright E2E flow: substitute path *
  - Driver manipulates the test clock to expire a round's window; asserts substitute assignment, `substitute_needed` email enqueued, substitute completes the round, game completes

- [ ] 21.5 Checkpoint — integration suite green
  - All integration tests and the primary Playwright flow pass. Ask the user if questions arise.

### 22. Accessibility validation (Req 13)

- [ ] 22.1 Integrate axe-core with Playwright
  - Add `@axe-core/playwright`; create a helper `scanA11y(page, viewport)` that asserts zero WCAG 2.1 AA violations on the provided page
  - _Requirements: 13.3_

- [ ] 22.2 Run axe-core at 320, 768, and 1920 px viewports (design §12.1)
  - Scan sign-in, onboarding, teams list, team detail, game status (Active and Complete), round preview, round play, profile
  - Enforce zero violations on primary flows; document any known-false-positive allowlist
  - _Requirements: 13_

- [ ] 22.3 Verify no-horizontal-scroll and ≥ 44×44 px tap targets on mobile
  - Per Req 13.2: at 320 px viewport, assert `document.scrollWidth <= window.innerWidth` on every page
  - Assert primary interactive element bounding boxes are ≥ 44×44 px
  - _Requirements: 13.2_

### 23. Observability and hardening

- [ ] 23.1 Integrate Sentry in `apps/api` and worker
  - `@sentry/node` init in both entrypoints; capture all unhandled errors and thrown `AppError` 5xx; attach request id + game_id + round_id tags where available

- [ ] 23.2 Integrate Sentry in `apps/web` *
  - `@sentry/react` init; source maps uploaded in CI

- [ ] 23.3 Implement operator alerts for email dispatch failures (design §11)
  - After 5 failed sends for an `email_outbox` row, raise a Sentry event tagged `email_retry_exhausted` with `notification_type`, `player_id`, `last_error`
  - _Requirements: 12_

- [ ] 23.4 Implement operator alert for question bank exhaustion
  - Triggered from task 10.8; Sentry event tagged `bank_exhausted` with `category` and `game_id`
  - _Requirements: 5.1, 5b.1_

- [ ] 23.5 Implement rate limiting on `POST /auth/magic-link`
  - Limit to N requests per IP per minute (N configurable via env); return 429 on overage to prevent email spam abuse

- [ ] 23.6 Harden request logging
  - Structured JSON logs with request id, player_id (when present); scrub `email` and JWT from all logged payloads (Req 2.7, 15.5, §9.5)

- [ ] 23.7 Additional operator alerts beyond email failures *
  - Alert on sweeper exceptions, DB pool exhaustion, unusually long acceptance transaction durations

### 24. Deployment and launch readiness

- [ ] 24.1 Configure Railway project with two services
  - `api` (Fastify + static React assets), `worker` (cron + dispatcher); link to the same Supabase Postgres; document env-var wiring

- [ ] 24.2 Configure production CI/CD pipeline
  - On merge to `main`: run CI, build and deploy `api` + `worker` to Railway; run DB migrations as a one-off job; fail deploy if migrations fail

- [ ] 24.3 Implement smoke tests that run post-deploy in staging
  - `GET /health` returns 200; `pnpm db:seed:smoke` reports ≥ 500 questions per category (task 8.7)
  - Automated E2E happy-path against staging before promotion to prod

- [ ] 24.4 Build launch checklist document
  - Env vars set, question bank seeded, Resend domain verified, Supabase Auth configured, Sentry DSN live, rate limits tuned, alert routing set up

- [ ] 24.5 Final checkpoint — launch-ready
  - All prior tasks complete and tests green; smoke tests pass in staging; launch checklist fully checked. Ask the user if questions arise.

## Notes

- Tasks marked with `*` are optional and can be skipped for a faster MLP. Top-level tasks are never optional.
- Each task references specific requirements for traceability. Multiple requirements are listed where one task addresses several criteria (common in the acceptance-transaction and scoring tasks).
- Checkpoints appear at natural breaks (after scaffolding, after schema, after acceptance, after PBTs, after integration, at launch) so progress can be validated incrementally.
- The 18 property-based tests in Section 20 correspond one-to-one with the 18 Correctness Properties in design Section 10.
- The testing layer breakdown (unit / property / integration / E2E / a11y / email smoke / bank smoke) follows design Section 12, with test tasks co-located under their related feature sections rather than consolidated — except the PBT suite, which is its own section because it spans the full domain.
- Question bank seeding (Section 8) is a real product effort and can proceed in parallel with Sections 4–7 and 9 but must be complete before production game creation is enabled.
- The Game acceptance transaction (Section 10) is the most complex atomic operation in the system and is decomposed into discrete sub-tasks per design §4.3 and §6.1–§6.2.
- The discriminated-union status response (Section 13) is treated as an architectural invariant: two separate serializer functions, with a dedicated test asserting no shared code path between them.
