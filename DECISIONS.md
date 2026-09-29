# Decision log

Record every significant decision here, newest at the bottom. Never delete an entry; if a decision changes, add a new one that supersedes it.

## Template

```
## NNN — Title
- Date: YYYY-MM-DD
- Status: Proposed | Accepted | Superseded by NNN
- Decided by:
- Context: what problem or question prompted this?
- Decision: what we chose.
- Alternatives considered: what else we looked at and why not.
- Consequences: what this makes easier or harder.
```

---

## 001 — Pull-request workflow instead of enforced branch protection
- Date: 2026-09-28
- Status: Accepted
- Decided by: George
- Context: The repo is private on a personal GitHub account. GitHub does not enforce rulesets or branch protection there without a paid organization plan.
- Decision: The team agrees never to push directly to `main`. All changes go through a branch and a pull request with at least 1 approval. The rule is written in `CLAUDE.md` so Claude Code follows it too. A ruleset is saved in GitHub settings so it starts enforcing if the repo later moves to a paid organization.
- Alternatives considered: Making the repo public (free protection, but exposes the project); moving to a paid GitHub Team organization (cost not justified yet).
- Consequences: Protection depends on discipline. Revisit when the team grows or starts paying for GitHub.

## 002 — Browser-based web app, no native apps for v1
- Date: 2026-09-28
- Status: Proposed
- Decided by: George (pending team review)
- Context: Camera operators are event guests who will not install an app. Viewers can be on any device.
- Decision: Everything runs in the browser: camera pages (phones and laptops), host pages, viewer pages.
- Alternatives considered: Native iOS/Android apps (better background camera and battery control, but install friction and much more work).
- Consequences: Must work around iOS browser limits (screen lock stops the camera). Native apps can be added later if needed.

## 003 — LiveKit Cloud for the camera side
- Date: 2026-09-28
- Status: Proposed
- Decided by: George (pending team review)
- Context: Many phones and laptops must publish live video with low delay, and the host needs to see all cameras.
- Decision: Use LiveKit Cloud (WebRTC SFU). An event is a LiveKit room; each camera is a participant.
- Alternatives considered: Cloudflare Realtime SFU (cheaper bandwidth but lower-level, more code to write); building our own media server (too much work).
- Consequences: Good SDKs and React components; egress available for converting and recording. LiveKit is open source, so self-hosting is possible later.

## 004 — Two delivery paths: WebRTC for cameras, HLS via Cloudflare Stream for viewers
- Date: 2026-09-28
- Status: Proposed
- Decided by: George (pending team review)
- Context: The target audience is thousands of viewers per event. WebRTC at that scale is expensive and hits plan connection limits.
- Decision: LiveKit egress pushes each camera (and the grid) to Cloudflare Stream live inputs. Viewers watch HLS through Cloudflare's CDN. Early phases may use WebRTC viewers for simplicity.
- Alternatives considered: WebRTC for all viewers (sub-second delay but costly and capped); cameras publishing directly to Cloudflare Stream via WHIP (WebRTC broadcasts cannot currently be recorded there, and no grid composite).
- Consequences: Viewers are about 5-15 s behind. Cost scales with viewer minutes. Recording comes built in.

## 005 — Grid view as one server-side composite stream
- Date: 2026-09-28
- Status: Proposed
- Decided by: George (pending team review)
- Context: Viewers want to see all cameras at once. Playing many separate streams on a phone drains battery and bandwidth.
- Decision: Use LiveKit RoomComposite egress with a grid layout, published as one extra stream.
- Alternatives considered: Client-side grid of N separate players (too heavy on phones at HLS scale).
- Consequences: One extra egress per event (transcode cost). Layout customizable via a custom template.

## 006 — Chat and reactions on Supabase Realtime with event and per-camera channels
- Date: 2026-09-28
- Status: Proposed
- Decided by: George (pending team review)
- Context: HLS viewers are not in the LiveKit room. Viewers must be able to react to the whole event or to one camera, and camera operators should see reactions for their camera.
- Decision: Use Supabase Realtime channels `event:<id>` and `event:<id>:cam:<camId>`. Viewers subscribe to the event channel plus the camera they are watching. Reactions are batched server-side into per-second counts.
- Alternatives considered: LiveKit data messages (only for room participants); Ably or Cloudflare Durable Objects (still options if Supabase Realtime limits are reached).
- Consequences: One more service to scale, but it is already in the stack.

## 007 — Host-selected audio source
- Date: 2026-09-28
- Status: Proposed
- Decided by: George (pending team review)
- Context: Multiple microphones cannot play at once, especially in grid view.
- Decision: The host chooses which camera's audio is used for the grid (and optionally for all views). Camera operators can mute themselves; the host can mute any camera.
- Alternatives considered: Mixing all microphones (noisy, echo); no audio in grid (poor experience).
- Consequences: Needs a host control and an audio setting on the event.

## 008 — Application stack
- Date: 2026-09-28
- Status: Proposed
- Decided by: George (pending team review)
- Context: Small team, first full application, TypeScript experience.
- Decision: Next.js (TypeScript) on Vercel, Supabase (Postgres, auth, realtime), Stripe for payments, Cloudflare R2 for any extra file storage, Playwright for end-to-end tests.
- Alternatives considered: Separate frontend and backend services (more to deploy and learn).
- Consequences: One codebase and one deployment to start. Can split later if needed.

## 009 — Feature scope for v1
- Date: 2026-09-28
- Status: Proposed
- Decided by: George (pending team review)
- Context: Brainstorm of additional features beyond the core multi-camera stream.
- Decision: In v1: main feed, auto-director, camera labels and pins, schedule and countdown, lobby, camera health, local HQ backup recording, zoom and camera flip, picture-in-picture, rewind while live, video guestbook, synced multi-camera replay, automatic highlight reel, download package, moderation, private events, "you're live" indicator and no-filming pause, Pro tier with RTMP pro cameras and branding, host analytics. Later: live captions and translation. Not planned: viewer clip sharing, landscape/stability guidance, virtual gifts.
- Alternatives considered: A smaller v1. The phases in `ARCHITECTURE.md` keep the build order incremental.
- Consequences: Adds a media worker (FFmpeg) and R2 storage for after-event features.

## 010 — Auto-director driven by reactions only
- Date: 2026-09-28
- Status: Proposed
- Decided by: George
- Context: The main feed can switch automatically when the host is busy.
- Decision: Auto-director switches the main feed to the camera with the most reactions over a recent window, with a minimum time per camera to avoid jumpy switching. The host can override at any time. Audio loudness is not used.
- Alternatives considered: Switching on audio activity (picks up noise, music, and speeches unevenly).
- Consequences: Depends on per-camera reaction counts; with few viewers it may not switch much.

## 011 — Highlight reel: 1 minute before to 2 minutes after each reaction spike
- Date: 2026-09-28
- Status: Proposed
- Decided by: George
- Context: Reactions show which moments mattered. The build-up to a moment matters as much as the moment itself.
- Decision: For each reaction spike, take a clip from 1 minute before to 2 minutes after. Camera spikes use that camera; event-wide spikes use the main feed camera at that time. Overlapping clips on the same camera merge. Clips are joined in chronological order. The host can remove clips before sharing.
- Alternatives considered: Fixed short clips around the peak (loses context).
- Consequences: Needs reaction counts per second, a main-feed log, and a media worker to cut and join clips.

## 012 — Grid and camera downloads through Cloudflare Stream MP4 downloads
- Date: 2026-09-28
- Status: Proposed
- Decided by: George (pending team review)
- Context: Hosts want to download the grid and individual cameras after the event.
- Decision: The grid is recorded as its own Stream live input. The dashboard requests an MP4 download from Cloudflare Stream for the grid or any camera and gives the host the link. HQ camera backups come from R2. A full ZIP package is built by the media worker.
- Alternatives considered: Recording the grid separately to R2 with a second egress (extra transcode cost).
- Consequences: Each download is billed like one viewing, so downloads are limited per plan.

## 013 — Default audio source and handover
- Date: 2026-09-28
- Status: Proposed
- Decided by: George
- Context: Decision 007 makes the host responsible for audio in the main feed and grid. The host may forget to pick a source, and the chosen camera may disconnect.
- Decision: If no source is picked, use the host's camera if the host is streaming, otherwise the first camera that joined. If the audio camera disconnects or mutes, audio moves to the next connected camera in join order. Audio does not follow main-feed switches.
- Alternatives considered: Silent grid with per-tile audio chosen by viewers (rejected: extra stream cost per viewer, sync issues, and the host knows best which microphone is good); audio following the main feed (jumps with auto-director).
- Consequences: The director page must show which camera is the audio source and whether it was chosen automatically.

## 014 — Events survive when all cameras go offline
- Date: 2026-09-28
- Status: Proposed
- Decided by: George
- Context: Cameras drop out often (battery, network, closed pages). If every camera drops, viewers, the host, and camera operators must be able to continue without new links.
- Decision: The event lives in our database, independent of the LiveKit room. A new `standby` state means live with no cameras. All links stay active; viewers see a standby screen with chat still working and reconnect automatically. Cameras rejoin their old slot using a rejoin key stored on the device. Host rights belong to the host's account. Each camera keeps its Stream live input for the whole event; recordings may have several segments per camera. Grid egress stops while empty. The event ends only when the host ends it, or after a long standby with a notification and grace period.
- Alternatives considered: Ending the event when the LiveKit room closes (viewers would lose the stream and need new links).
- Consequences: Needs LiveKit webhooks, camera rejoin keys, multi-segment recordings, and an auto-end timer.

## 015 — YourPOV is for all kinds of events
- Date: 2026-09-28
- Status: Accepted
- Decided by: George
- Context: The idea started from a wedding livestream, and early discussions used weddings as the main example.
- Decision: YourPOV targets all kinds of live events (concerts, sports, conferences, parties, graduations, religious services, weddings, and more). Docs, product wording, examples, and design stay event-neutral.
- Alternatives considered: A wedding-only product (smaller market; narrower positioning).
- Consequences: Features and copy must not assume a wedding. Which event types to market first is still an open question.

## 016 — Viewers are anonymous by default; host can require sign-in
- Date: 2026-09-29
- Status: Accepted
- Decided by: Andrii
- Context: Open question: do viewers need accounts, or only a display name? Bans work better with accounts, but sign-up drives guests away.
- Decision: By default viewers enter only a display name. The host can turn on "Signed-in viewers only" per event (Google or email magic link via Supabase Auth), before or during the event. Anonymous bans use a browser device ID; bans on signed-in viewers use the account.
- Alternatives considered: Display name only with no sign-in option (bans easy to get around); mandatory accounts for all viewers (many guests drop off at sign-up).
- Consequences: Frictionless joining for most events, reliable moderation where the host needs it. Needs a per-event setting, viewer sign-in UI, and bans that store either a device ID or a user ID. MVP can ship display-name only; optional sign-in comes with host accounts in phase 5.
