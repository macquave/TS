# Trivia Squads — Claude Code Context

## What this project is
Trivia Squads is an async-first, team-based trivia web app. Two teams of 3–8 players compete across 5 rounds. Each round is owned by one player who answers 5 themed questions within a 12-hour window. No real-time play required.

## Spec files (source of truth)
- `requirements.md` — 15 numbered requirements (plus Req 5b known limitation) with formal acceptance criteria
- `design.md` — Full technical design: architecture, data model (Postgres DDL), API contracts, algorithms, state machines, security, testing strategy, and tech stack
- `tasks.md` — Ordered implementation plan broken into ~1–3 hour tasks across 24 task groups

## Current status
MLP spec is complete. A Lovable/Bolt.new prototype is planned first, then production build.

## Tech stack (from design.md §13)
- **Frontend**: React 18 + Vite + TailwindCSS + TypeScript
- **Backend**: Node 20 LTS, Fastify, Zod, Kysely + pg
- **Database**: Supabase Postgres 15 with RLS
- **Auth**: Supabase magic link
- **Email**: Resend + React Email templates
- **Background jobs**: Railway cron + node-cron
- **Testing**: Vitest + fast-check (PBT) + Playwright + axe-core
- **Deploy**: Railway (two services: `api` + `worker`)

## Key architectural invariants (never violate these)
1. **Hard score lock**: No score/correctness data in any API response while `game_state = Active` (Req 8). Two entirely separate serializer functions — `serializeActive` and `serializeComplete` — with no shared code path.
2. **Server-authoritative timer**: Question timer starts server-side on first question fetch; client cannot reset it by refreshing.
3. **No runtime LLM**: Question bank is pre-populated and maintained out-of-band. No Claude/LLM dependency at runtime.
4. **Single transaction for game acceptance**: roster validation, round creation, category draw, owner assignment, question selection, and email enqueue all happen in one `REPEATABLE READ` transaction.
5. **Email privacy**: Player email addresses are never returned in any API response except `GET /players/me`.

## Data model summary
Core tables: `players`, `teams`, `team_memberships`, `games`, `rounds`, `round_owners`, `questions`, `answer_choices`, `round_questions`, `round_answers`, `email_outbox`, `notifications_sent`, `game_sitters`

## Background jobs (worker process)
- Window expiration sweeper: every 1 min
- 12h / 2h reminder schedulers: every 5 min
- Email dispatcher: every 30s
- Game completion: triggered inline on round state transitions (not scheduled)

## Testing approach
18 property-based tests (fast-check) map 1:1 to the 18 Correctness Properties in design.md §10. Integration tests use testcontainers Postgres. E2E uses Playwright.
