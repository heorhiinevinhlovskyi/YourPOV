# How we work on YourPOV

This is how the team collaborates, plans, and uses AI tools. Rules that Claude Code must follow are in `CLAUDE.md`; this file explains the whole workflow for humans.

## 1. The repo is the shared project

- **GitHub (this repo) is the single source of truth.** Plans, decisions, and code live here, not in anyone's chat history.
- `ARCHITECTURE.md` is the current design. `DECISIONS.md` records why. Chats with Claude are temporary: if something is decided in a chat, it goes into these files.
- The team does **not** use a shared Claude Team plan for now (see `DECISIONS.md` 015). Everyone uses their own Claude account.

## 2. Joining the team

1. The repo owner adds you as a collaborator: **Settings → Collaborators → Add people** (free).
2. Clone the repo and read `README.md`, `ARCHITECTURE.md`, `DECISIONS.md`, and `CLAUDE.md`.
3. Copy `.env.example` to `.env` and ask the team for development keys (never share keys in chat, issues, or commits).

## 3. Using Claude

- Use your own **Claude Code** (included with paid Claude plans, or with an Anthropic API key). The free Claude plan does not include Claude Code.
- Claude Code reads `CLAUDE.md` automatically, so every team member's Claude follows the same rules and knows the same design.
- For planning chats in claude.ai, create your own Project and upload `ARCHITECTURE.md` and `DECISIONS.md` as project knowledge. Re-upload them after they change.
- To show the team a useful Claude conversation, share it as a chat snapshot link in the PR or issue.
- Review AI-generated code like any other code. You are responsible for what you commit.

## 4. Branches and pull requests

- **Never push to `main`.** GitHub does not enforce this on our private repo (see `DECISIONS.md` 001), so it depends on everyone.
- One branch per change: `feat/…`, `fix/…`, `docs/…`, `chore/…`.
- Open a pull request into `main`. At least **1 approval from another team member** before merging.
- Keep PRs small: one feature or fix. Large AI-generated diffs are hard to review.
- Pull the latest `main` into your branch and resolve conflicts before asking for review.
- Decisions made while working go into `DECISIONS.md` in the same PR. Design changes go into `ARCHITECTURE.md` in the same PR.

## 5. Splitting the work

Each area has an owner (fill in the "Owner" fields in `ARCHITECTURE.md`), so two people's Claude sessions are not rewriting the same files:

| Area | Covers |
|---|---|
| Camera side | Camera page, device picker, mute, health, HQ backup, LiveKit room |
| Audience side | Viewer page, HLS player, main feed, grid, picture-in-picture |
| Backend | API, database, auth, tokens, webhooks, event lifecycle, payments |
| Realtime | Chat, reactions, batching, moderation |
| After the event | Recordings, replay, highlights, downloads, media worker |
| Quality | Playwright tests, CI, device test matrix |

## 6. Quality checks

- GitHub Actions runs lint and tests on every PR (once set up). A PR with failing checks is not merged.
- New features come with tests. End-to-end tests use Chrome's fake camera so several cameras and viewers can be tested at once.
- Before a release, run the manual device matrix in `ARCHITECTURE.md` (section 13).

## 7. Secrets and personal vs. work tools

- Never commit API keys, tokens, passwords, or stream keys. Use `.env` (ignored by git) and document variables in `.env.example`.
- YourPOV is a personal project. Do not connect employer-owned accounts, tools, or paid seats to this repo, and do not use employer code or data here.
