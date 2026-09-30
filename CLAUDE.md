# CLAUDE.md — YourPOV

Instructions for Claude Code (and humans) working in this repository.
Read this file, `ARCHITECTURE.md`, and `CODEBASE_RULES.md` before making changes.

## Project

YourPOV is a multi-camera streaming application for **all kinds of live events** (not only weddings), built by a small team of friends. Keep product wording, examples, and design event-neutral.
See `ARCHITECTURE.md` for the system design and `DECISIONS.md` for why things are the way they are.

## Git workflow (REQUIRED)

1. **Never commit or push directly to `main`.** This applies to humans and to Claude.
   Branch protection is not enforced by GitHub on this repo, so this rule depends on everyone following it.
2. Create a branch for every change: `feat/<short-name>`, `fix/<short-name>`, `docs/<short-name>`, `chore/<short-name>`.
3. Open a pull request into `main`. At least **1 approval from another team member** is required before merging.
4. Do not force-push to `main`, and do not delete `main`.
5. Keep PRs small and focused on one feature or fix.
6. Before opening a PR, pull the latest `main` and resolve conflicts on your branch.

If you are Claude and the current branch is `main`, create a new branch before making any commits.

## Documentation rules

- Any architectural decision made during a session must be added to `DECISIONS.md` in the same PR.
- If a change affects the system design, update `ARCHITECTURE.md` in the same PR.

## Secrets

- Never commit API keys, tokens, passwords, or stream keys.
- Use `.env` files (already ignored by git). Document required variables in `.env.example` with placeholder values.

## Tech stack

Proposed (see `DECISIONS.md`, pending team review):

- Next.js (TypeScript) on Vercel
- LiveKit Cloud for camera video (WebRTC)
- Cloudflare Stream for viewer delivery (HLS) and recordings
- Supabase for Postgres, auth, and realtime chat/reactions
- Stripe for payments
- Playwright for end-to-end tests
- Cloudflare R2 for file storage (HQ backups, guestbook videos, highlight reels, download packages)
- Media worker (FFmpeg) for after-event processing

## Commands

The app is not scaffolded yet. When it is, list the real commands here. Planned shape:

- One command to install, one to run locally, one to run unit tests, one to run Playwright tests.
- One `check` command that runs lint (zero warnings allowed), format check, typecheck, and unit tests. Run it before every PR.
- GitHub Actions runs the same `check` on every PR, so passing locally means passing in CI.
- Pin one Node.js version and one TypeScript version for the whole team.

## Code conventions

Full rules are in `CODEBASE_RULES.md`. The most important ones:

- **One owner per piece of state.** Postgres owns events, cameras, and settings. LiveKit and Realtime messages are not the source of truth. Never keep a second copy that can drift.
- **The API decides shared things** (event status, main feed, audio source, limits, bans). Clients send requests; the host sends commands; each page only changes its own device or view.
- **Never trust the client** for identity, role, plan, or billing.
- **Parse all outside input with a schema** where it enters (requests, webhooks, Realtime messages). No `any`, no casting through `unknown`.
- **Named constants** for limits, timeouts, and windows. Plan limits in one typed config.
- **Fail loudly on broken invariants**; show a safe message for normal failures; never log secrets or full payloads.
- **Webhooks** verify the signature and are idempotent.
- **Supabase RLS on every table**, deny by default. Service role key and all provider secrets stay server-only.
- **UI shows state and sends intent**; no business logic in components. Reuse shared components and design tokens.
- **Tests check behavior**, not trivial plumbing. New logic ships with its test.
- **Simplest solution that works**, small PRs, one intent per commit.
