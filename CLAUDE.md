# CLAUDE.md — YourPOV

Instructions for Claude Code (and humans) working in this repository.
Read this file and `ARCHITECTURE.md` before making changes.

## Project

YourPOV is a streaming application built by a small team of friends.
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

Proposed (see `DECISIONS.md` 002-008, pending team review):

- Next.js (TypeScript) on Vercel
- LiveKit Cloud for camera video (WebRTC)
- Cloudflare Stream for viewer delivery (HLS) and recordings
- Supabase for Postgres, auth, and realtime chat/reactions
- Stripe for payments
- Playwright for end-to-end tests

## Commands

_TBD — how to install, run, test, and lint._

## Code conventions

_TBD — language style, folder structure, naming, testing expectations._
