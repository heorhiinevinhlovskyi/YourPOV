# Codebase rules

Rules for everyone (humans and Claude) writing code in YourPOV. They sit next to `ARCHITECTURE.md` (what we build) and `DECISIONS.md` (why).

`MUST`, `MUST NOT`, `SHOULD`, and `SHOULD NOT` mean what they mean in RFC 2119: `MUST` is required, `SHOULD` is the default unless there is a good reason.

If a rule blocks you, do not work around it quietly. Raise it in the PR, and if the rule changes, record it in `DECISIONS.md`.

## 1. Sources of truth

Every piece of state has exactly one owner. Read it from that owner when you need it.

| Data | Owner | Not the owner |
|---|---|---|
| Events, cameras, labels, pins, audio source, main feed, plans, bans, messages, recordings | Postgres (Supabase) | LiveKit room, Realtime channel, React state |
| Who is connected and publishing right now | LiveKit (reported to us by webhooks, then stored in Postgres) | Viewer pages |
| Live and recorded video | Cloudflare Stream | Our database only stores IDs and timestamps |
| Large files (HQ backups, guestbook, highlights, ZIPs) | R2 | Our database only stores object keys |
| Payments and subscription state | Stripe, mirrored into `subscriptions` / `event_passes` by webhooks | Client-sent values |

- Code MUST NOT keep a second copy of the same data that can drift (for example camera status cached in React state and also in Postgres). Derive what you show from the owner.
- The event lives in our database, not in LiveKit (decision 014). Code MUST NOT treat "the LiveKit room exists" as "the event is live".
- Realtime channel messages are notifications, not state. A viewer that joins late MUST be able to rebuild the current view (main feed, audio source, event status) from the database, not from messages it missed.

## 2. Who decides (authority)

- The **API decides** anything shared: event status, plan limits, main feed, audio source, bans, slow mode, auto-director switches, reaction counts. Clients only send requests.
- The **host sends commands** (mute camera, remove camera, pause filming). The API checks that the caller is the host (or a co-host) of that event, applies the change, then broadcasts it.
- A **camera page owns its own device**: camera, mic, zoom, flip, local recording. Other clients never change a camera page directly; they send a command and the camera page applies it.
- A **viewer page owns only its own view**: which camera it watches, volume, picture-in-picture.
- Code MUST NOT trust identity or role sent by the client. The server reads it from the Supabase session, the LiveKit token, or a server-issued camera rejoin key.
- Code MUST NOT trust client-sent values for billing, plan limits, ownership, or moderation.

## 3. Three kinds of messages

Keep them separate. They are not interchangeable.

1. **Local UI events** inside one page. No network.
2. **Realtime broadcasts** (`event:<id>`, `event:<id>:cam:<camId>`) for things that happen once: a chat message, a reaction, "main feed changed".
3. **Stored state** in Postgres for anything a late joiner or a replay needs.

- A broadcast that changes shared state MUST come from the server after the change is saved, not from a client.
- Code MUST NOT use a database row as a message queue ("insert a row so the other side notices, then delete it"). Use a broadcast or a job.

## 4. TypeScript

- `strict` mode is on. `any` MUST NOT be used to get around a type error.
- Unknown input (request bodies, webhook payloads, Realtime messages, `localStorage`, URL params) MUST be parsed with a schema (for example zod) at the boundary where it enters. After that, code works with typed values, not `unknown` or loose records.
- Code MUST NOT probe fields with casts like `value as { field?: unknown }`.
- Code MUST NOT pack structured data into strings (`"cam:3:muted"`) or recover meaning from IDs, prefixes, file paths, or error messages. Use named fields. Build strings only for display. (Realtime channel names are an external naming convention; build and parse them in one helper module.)
- IDs SHOULD have their own types (`EventId`, `CameraId`) so they cannot be mixed up.
- Limits, timeouts, retry counts, window lengths, and similar numbers MUST be named constants (for example `AUTO_DIRECTOR_WINDOW_SECONDS = 20`, `AUTO_DIRECTOR_MIN_HOLD_SECONDS = 15`, `GRID_MAX_TILES = 9`). Plan limits live in one typed plan config, not scattered across the code.
- A fixed set of values (event status, device type, role, plan tier) MUST be a union type or `as const` map, and `switch` statements over it MUST handle every case (use an `assertUnreachable` helper in `default`).
- Request and response types for API routes SHOULD be shared between client and server from one module.

## 5. Code structure

- UI components show state and send user intent. They MUST NOT contain business logic, database writes, billing, retries, or job handling. That lives in server code (API routes, server actions, the media worker) or in a small client service that owns it.
- Server-only code (Supabase service role, LiveKit server SDK, Stripe, Cloudflare API tokens) MUST live in server-only modules (use the `server-only` package) so it can never be bundled into the browser.
- Environment variables starting with `NEXT_PUBLIC_` are public. Secrets MUST NOT use that prefix.
- Constants, types, and pure helpers SHOULD live in small modules with no heavy imports, so importing a constant does not pull in a whole SDK.
- Code MUST NOT rely on circular imports.
- Avoid tiny helper functions that only pass through to another function. Write a helper when it has a clear purpose you can explain in one sentence.

## 6. Async work, webhooks, and jobs

- Every promise MUST be awaited, returned, or handled with an explicit `.catch`. No unhandled rejections.
- **Webhooks** (LiveKit, Cloudflare Stream, Stripe) MUST:
  - verify the signature before reading the payload (Stripe needs the raw body: `await req.text()`),
  - be idempotent: store the event ID and skip events already handled,
  - log and return success for event types we do not handle.
- Long work (clips, highlight reels, ZIPs, cleanup) runs in the media worker, not in Vercel functions.
- Jobs have an explicit status: `queued | processing | succeeded | failed | cancelled`. The UI creates or shows the pending record right away, and never waits forever on a job that failed.
- Code SHOULD react to events (webhooks, database changes, broadcasts) instead of polling. If a provider only offers a status endpoint (for example "is this MP4 ready?"), poll from server code with a timeout and backoff, never from a UI loop.
- Handlers for "participant joined/left", "track published", and similar MUST work when called twice or out of order.

## 7. Errors

- Broken invariants (a camera row with no event, a host command for an event the host does not own) MUST fail loudly: throw or assert with a message that names the operation and what was missing. Do not return `null` or skip silently.
- Expected situations (plan limit reached, wrong password, banned viewer, camera permission denied) are normal results. Show a clear message to the user; do not throw or log them as errors.
- When a user action fails, log the technical details on the server, and show the user a short, safe message (a toast or an inline error).
- A `catch` MUST handle the error, rethrow it with context, or recover on purpose. Empty `catch` blocks are only allowed for documented feature detection (for example checking browser camera zoom support).
- Functions MUST NOT signal failure by silently returning `false`.

## 8. Logging

- Code MUST NOT log secrets: API keys, LiveKit tokens, Stripe keys, stream keys, RTMP URLs with keys, session cookies, rejoin keys, passwords.
- Code MUST NOT log full request or response bodies, chat message content, video data, or large JSON. Log IDs, counts, status values, and durations.
- Personal data (emails, display names) SHOULD NOT appear in logs. Use user or viewer IDs.

## 9. Security

- **Supabase:** Row Level Security MUST be enabled on every table. Policies start from "deny" and allow only the exact operation needed (for example a host can update only their own events). Listing tables (users, events, bans) MUST NOT be open to everyone.
- The Supabase **service role key** is server-only and bypasses RLS. Use it only in server code that has already checked who the caller is.
- **LiveKit tokens** are issued only by the API, short-lived, and scoped to one room and one role. Only the `camera` role can publish. Viewers joining over WebRTC (Phase 1) get subscribe-only tokens.
- **Private and paid events:** playback MUST be protected (Cloudflare Stream signed URLs or tokens) so a copied HLS link does not bypass the password, invite list, or viewer cap.
- **Uploads to R2** use short-lived presigned URLs for one exact object key, with a size limit and an allowed content type. Clients never get bucket-wide credentials.
- Event codes, rejoin keys, and invite links MUST be long and random. Store only hashes of rejoin keys and passwords.
- Chat, reactions, camera joins, and guestbook uploads MUST be rate-limited per viewer and per event.
- Redirect targets (after sign-in, after Stripe Checkout) MUST come from an allowlist.
- Server code that fetches a URL supplied by a user or a provider MUST validate it (only `https`, no private or local IP addresses, check redirects).
- Security changes SHOULD include tests for both allowed and denied cases.

## 10. Payments (Stripe)

- One server-side Stripe client with a pinned API version. No keys in the browser.
- Prefer Stripe Checkout for event passes and subscriptions, and the Stripe Customer Portal for managing subscriptions.
- What a host is allowed to do (plan tier, viewer cap, camera cap) comes from our own `subscriptions` and `event_passes` tables, which are updated only by verified webhooks.
- Put stable IDs (`userId`, `eventId`) in Stripe metadata so payments can be matched to our records.
- Before changing or canceling a subscription, check that it belongs to the signed-in user.
- Use separate Stripe keys for local, staging, and production.

## 11. UI

- Reuse shared components (button, card, dialog, input, player controls). Do not hand-style a raw `<button>` or `<div>` when a shared component exists.
- Use design tokens (Tailwind theme values) for color, spacing, and type. Do not add one-off colors or spacing.
- One accent color, used only for the primary action, the selected item, and focus. Neutral surfaces use neutral colors.
- Every control has hover, pressed, disabled, and keyboard focus (`focus-visible`) states.
- Camera and viewer pages are used on phones first. Controls MUST stay reachable on a small screen; test the mobile layout.
- Stacking: overlays (dialogs, menus, toasts) render into a small set of shared overlay layers in a fixed order, not into random `z-index` values. Inside one component, keep `z-index` small and local.
- Overlays on top of the video player MUST NOT block player controls when they are empty (use `pointer-events: none` on empty layers).
- Show "no data" as "—", not `0` (for example camera battery or network quality we have not received yet).

## 12. Testing

- Tests check behavior people depend on: plan limits, token roles, event status changes, audio handover, auto-director switching, spike detection, clip window merging, RLS allowed/denied cases, webhook idempotency.
- Tests SHOULD NOT check trivial things (a constructor sets a field, a constant exists). If a test would still pass after replacing the code with a hardcoded value, rewrite or remove it.
- Mock only external services (LiveKit, Cloudflare, Stripe, time). Keep our own logic real.
- Playwright:
  - use role and label locators; use `data-testid` only for video tiles and other elements without an accessible role,
  - start `waitForResponse` before the click that triggers the request,
  - wait with `expect(...)` or `expect.poll`, not fixed sleeps,
  - use separate browser contexts for host, camera, and viewer in the same test,
  - use Chrome's fake camera flags (see `ARCHITECTURE.md` section 13).
- New logic ships with its test in the same PR.

## 13. Changes and pull requests

- Follow the git workflow in `CLAUDE.md`.
- Pick the simplest solution that works. Before writing new code, check in order: do we need it at all, then the standard library, then the browser or platform, then a dependency we already have, then the smallest local change.
- Keep PRs small. Every new branch, helper, fallback, and abstraction has to earn its place. Do not remove security checks, data-loss protection, or accessibility to make a diff smaller.
- Each commit is one intent and builds on its own. Stage files by path; avoid `git add -A`.
- Before asking for review, remove noise: comments that only repeat the code, defensive checks on values that are already validated, `any` casts, and code that does not match the style of the file.
- After merging `main` into your branch, run all checks again before pushing. A merge without conflicts can still break the build.
- When a bug survives two or three fix attempts, stop patching. Write down what you tried, what you assumed, and how the data flows, then find the real cause.

## 14. Documentation

- Docs are short and direct. Prefer rules and contracts over long explanations.
- Update docs in the same PR when behavior, commands, or rules change. Do not leave old guidance that contradicts the new code.
- Comments explain why (a constraint, a tradeoff, a browser quirk), not what the next line does.
- Never hand-edit generated files (for example generated Supabase types). Regenerate them.
