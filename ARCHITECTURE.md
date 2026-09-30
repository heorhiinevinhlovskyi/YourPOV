# YourPOV — Architecture

_Status: v1 draft (proposed). Reasoning for each choice is in `DECISIONS.md`._

## 1. Product overview

**Problem.** Livestreams of live events usually use one phone or one fixed camera. Remote viewers miss most of what happens, and nobody can choose what to look at.

**Scope.** YourPOV is for **all kinds of live events**: concerts, sports games, conferences, parties, graduations, religious services, weddings, school events, and more. No single event type is the focus, and the product, wording, and design must stay event-neutral.

**Idea.** Turn every guest's phone or laptop into a camera for one shared stream. Viewers watch from any browser, switch between cameras, see all cameras at once in a grid, and react to the whole event or to one camera. After the event, the host gets a synced multi-camera replay, an automatic highlight reel, and a download package.

### Launch focus

The product stays event-neutral (see above), but the first launch and marketing focus on:

1. **Family events:** weddings, christenings, graduations, anniversaries. Many guests with phones, and relatives abroad who cannot travel (large diaspora from Romania, Ukraine, Georgia, and Moldova) and want to watch. These hosts buy a Paid event pass.
2. **Videographers and event photographers as the sales channel.** They take a Pro subscription, connect their own cameras over RTMP next to guests' phones, and offer the livestream to their clients as an extra service. One videographer brings many events a year.

Later: conferences, religious services, and other recurring events. Held back for now: youth sports and school events (filming children needs consent) and concerts (music rights, see section 16).

### Roles

| Role | Device | What they do |
|---|---|---|
| **Host** | Phone or laptop | Creates the event, shares the QR code, directs the stream (main feed, labels, pins, mute, remove), picks the audio source, moderates, pays |
| **Camera operator** | Phone or laptop browser | Scans the QR code, streams camera + mic, mutes own mic, zooms and flips camera, sees reactions for their own camera |
| **Pro camera** (Pro tier) | OBS, GoPro, DSLR via RTMP | Videographer's professional camera joins as a normal camera |
| **Viewer** | Any browser | Watches the main feed, one camera, or the grid; picture-in-picture; rewinds while live; chats and reacts; leaves a video guestbook message |

## 2. System diagram

```mermaid
flowchart LR
    subgraph Contributors["Camera side (WebRTC, under 1 second delay)"]
        P["Phone cameras"]
        L["Laptop cameras"]
        H["Host director page"]
    end

    PRO["Pro cameras (OBS, GoPro, DSLR)"]

    subgraph LK["LiveKit Cloud"]
        IN["Ingress (RTMP)"]
        R["Event room (SFU)"]
        E["Egress: one stream per camera + grid composite"]
    end

    subgraph CF["Cloudflare"]
        LI["Stream live inputs: cam-1..N, grid"]
        CDN["HLS delivery via CDN"]
        REC["Stream recordings"]
        R2[("R2: local HQ backups, guestbook videos, highlight reels, download packages")]
    end

    subgraph App["YourPOV app (Next.js on Vercel)"]
        WEB["Web pages: host, camera, viewer, replay"]
        API["API: events, tokens, QR, billing, moderation"]
    end

    W["Media worker (FFmpeg jobs): highlights, zips"]

    subgraph Data["Supabase"]
        DB[("Postgres")]
        AUTH["Auth"]
        RT["Realtime channels: chat, reactions, main feed"]
    end

    STRIPE["Stripe"]
    V["Viewers (thousands, any browser)"]

    P -- publish video + audio --> R
    L -- publish video + audio --> R
    P -. HQ local recording upload .-> R2
    PRO -- RTMP --> IN --> R
    R -- low-res previews + health --> H
    R --> E
    E -- RTMPS --> LI
    LI --> CDN
    LI --> REC
    CDN -- "HLS (5-15 s delay)" --> V

    API -- create room, issue tokens --> R
    API -- start/stop egress --> E
    API --> DB
    API --> STRIPE
    WEB --> API
    V <-- event + camera channels --> RT
    P <-- own camera channel --> RT
    H <-- all channels --> RT
    RT -- reaction counts --> DB
    DB -- spikes --> W
    REC --> W
    R2 --> W
    W --> R2
    AUTH --> API
```

## 3. Two delivery paths

| | Camera side | Audience side |
|---|---|---|
| Who | Host + camera operators (dozens) | Viewers (thousands) |
| Technology | WebRTC via LiveKit room | HLS via Cloudflare Stream CDN |
| Delay | Under 1 second | About 5-15 seconds |
| Why | Two-way, interactive, low delay | Cheap and scalable to very large audiences |

The bridge is **LiveKit egress**: it converts each camera, plus a grid composite, into a stream pushed over **RTMPS** (RTMP over TLS, port 443, which is what Cloudflare Stream live inputs accept) to a Cloudflare Stream live input.

During the early spike (Phase 1), viewers can join the LiveKit room directly over WebRTC. The viewer page only swaps its player when moving to HLS; the camera side does not change.

## 4. Camera side (ingest)

- Cameras join from the browser. No app install. Works on phones (iOS Safari, Android Chrome) and laptops.
- **Join flow:** the host shows a QR code, which is a link like `https://<domain>/e/<eventCode>/camera`. The API checks the event and plan limits, then issues a short-lived LiveKit token with the `camera` role.
- **Device picker:** choose camera and microphone (laptops often have several).
- **Zoom and front/back flip** on phones (zoom where the browser supports camera zoom constraints).
- **Mic mute:** camera operators mute/unmute themselves with a clear on-screen indicator. The host can mute any camera remotely.
- **"You're live" indicator** is always visible on camera devices.
- **Simulcast** is on, so the host director page receives low-resolution previews of all cameras.
- **Camera health:** each camera reports battery level, network quality, and whether the screen is about to lock. Shown on the host director page with warnings.
- **Phone reliability:**
  - iOS stops the camera when the screen locks or the browser goes to the background. Use the Wake Lock API and show a "keep this screen open" banner.
  - Default to 720p for the live stream to limit battery drain and heat.
  - Recommend cellular data over venue Wi-Fi; show a connection-quality indicator.
  - Camera access requires HTTPS.
- **Pro cameras (Pro tier):** videographers connect OBS, GoPro, or DSLR encoders over RTMP through LiveKit Ingress. They appear as normal cameras.

### Local high-quality backup recording

Venue internet is often poor, so the live stream may be low quality. In parallel:

1. The camera page records locally at full quality with the browser's `MediaRecorder` API.
2. Chunks upload to R2 in the background when bandwidth allows, and the rest uploads after the event (resumable; the page shows upload progress and asks the operator to keep it open or come back later).
3. Replays and highlight reels use the HQ file when it exists, falling back to the Stream recording.

Limits to handle: phone storage space, iOS `MediaRecorder` format differences, and operators closing the page before uploading.

## 5. Audience side (delivery)

- Each camera and the grid become one Cloudflare Stream live input (egress pushes to it over RTMPS). Viewers play them with **hls.js** (native HLS on iOS Safari).
- **Main feed (default view):** the host picks a featured camera. Viewers who have not chosen a camera watch the main feed, which follows the host's choices. The current main feed is broadcast on the event realtime channel, and the viewer player switches source.
- **Auto-director:** when turned on, the main feed automatically switches to the camera with the most reactions over a recent window (for example the last 20 seconds), with a minimum time on each camera (for example 15 seconds) to avoid jumpy switching. The host can override at any time.
- **Camera labels and pins:** the host names cameras ("Stage", "Main entrance", "Crowd"), reorders, pins, or hides them.
- **Single camera view:** the viewer taps any camera to watch it directly.
- **Grid view ("security guard" view):** LiveKit RoomComposite egress combines all cameras into one grid video on the server. Viewers download one stream, not N, so it works well on phones. The grid plays the host's audio source (see section 6). The grid shows **at most 9 tiles (3×3)**, since more are unreadable on a phone. If an event has more than 9 cameras, the host's pinned cameras fill the grid first, then the rest in the host's order; the others are still available in single camera view.
- **Picture-in-picture:** the main feed large plus a second camera small. Costs the viewer two streams, so the small one uses a low rendition.
- **Rewind while live:** HLS supports seeking back within the live window, with a "Jump to live" button.
- **Schedule and countdown:** before going live, viewers see an event page with the schedule and a countdown ("Ceremony starts in 12:30").
- **Lobby:** cameras can join before the event goes live to test video and audio. Viewers do not see lobby cameras.

## 6. Audio

Many microphones cannot play at once. The **host is responsible for audio**.

| View | Audio |
|---|---|
| Single camera | That camera's audio |
| Main feed | The host's audio source |
| Grid | The host's audio source |

- The host picks an **audio source** (for example the phone nearest the speakers).
- **If the host has not picked one:** use the host's own camera if the host is streaming, otherwise the first camera that joined. The director page shows "Audio: Camera 1 (automatic) · Change".
- **Handover:** if the audio camera disconnects or mutes, audio moves to the next connected camera (in join order). If the original camera comes back, audio stays where it is unless the host changes it.
- Audio does not follow the main feed automatically, so auto-director switches do not make the sound jump around.
- In the grid, the tile whose audio is playing shows a 🔊 icon. Viewers can mute the grid with the normal player mute button.
- The grid is one combined video, so its audio is baked in. To hear a different camera, the viewer opens that camera's own view.
- Muted cameras send no audio in any view.

## 7. Event lifecycle and reconnection

**An event stays alive when cameras go offline.** Cameras dropping out (battery, network, someone closing the page) is normal. The event only ends when the host ends it.

### Event states

```
scheduled -> lobby -> live <-> standby
                        live <-> paused   (host no-filming pause)
                        live / standby / paused -> ended   (host ends, or auto-end)
```

| State | Meaning |
|---|---|
| `scheduled` | Created; countdown page for viewers |
| `lobby` | Cameras can join and test; viewers still see the countdown |
| `live` | At least one camera is streaming |
| `standby` | Event is live but **no cameras are connected** |
| `paused` | Host pressed no-filming pause |
| `ended` | Host ended the event; viewer link becomes the replay page |

### When all cameras go offline (`standby`)

- **Links stay active.** The viewer link, the camera QR link, and the host director page all keep working.
- **Viewers** stay on the page and see a "Cameras are offline, the stream will resume shortly" screen. Chat and reactions keep working. The player reconnects automatically when a camera comes back; viewers do not need to refresh.
- **Main feed:** when the main feed camera drops, the main feed moves to the next available camera. When none are left, viewers see the standby screen.
- **Host:** the director page shows which cameras dropped and when, and a "No cameras live" warning. The host can share the QR code again.

### Rejoining

- **Same camera slot:** each camera device stores a rejoin key in the browser. Reopening the same link on the same device reclaims the same camera: same label, same chat channel, same Stream live input, and recordings continue on that camera's timeline.
- **New device:** joins as a new camera.
- **Host:** host rights belong to the host's account, not to a device or a camera. If the host's phone dies, the host signs in on another device and opens the director page. The host's camera can drop without affecting host controls.

### Behind the scenes

- The LiveKit room may close when empty. That is fine: the event lives in our database, not in LiveKit. The next camera to join recreates the room with the same name.
- Our API listens to **LiveKit webhooks** (participant joined/left, track published/unpublished) to update camera status and the event state, and to start or stop egress.
- Each camera keeps its **Cloudflare Stream live input** for the whole event, so viewer stream URLs never change. The live input's recording `timeoutSeconds` is set so short drops continue the same recording; longer drops start a new recording segment. The `recordings` table stores all segments per camera with timestamps, and the replay timeline shows the gaps.
- The **grid egress** stops when no cameras are connected (no cost while empty) and restarts when the first camera rejoins.

### Ending

- The host clicks **End event**. Egress stops, live inputs are disabled, and the viewer link becomes the replay page.
- **Auto-end safety:** if an event stays in `standby` with no cameras for a long time (for example 2 hours), the host gets a notification, and the event ends automatically after a further grace period. This stops forgotten events from running up costs.

## 8. Chat and reactions

Viewers are not in the LiveKit room, so chat runs on a separate real-time service: **Supabase Realtime** (alternatives: Ably, Cloudflare Durable Objects).

### Channels

```
event:<eventId>              whole-event chat, reactions, main-feed changes, per-camera reaction summary
event:<eventId>:cam:<camId>  chat and reactions for one camera
```

- Viewers subscribe to the event channel plus the channel of the camera they are watching. Switching cameras switches the camera subscription.
- Comment box toggle: **To everyone** / **To this camera**.
- Grid view subscribes only to the event channel. The server also publishes a per-second **reaction summary for every camera** on the event channel (for example `{cam-1: ❤️ 12, cam-3: 🎉 40}`), so counts can float over each tile without the viewer subscribing to every camera channel. Grid comments go to the whole event.
- Camera operators subscribe to their own camera channel and see reactions and comments live, so viewers can direct them.
- The host sees all channels.

### Scale

- Reactions are **batched on the server** into counts per channel per second (for example "❤️ ×240") and stored in `reaction_counts`. These counts also drive the auto-director and highlight detection.
- Rate limits per viewer.
- Viewers are 5-15 s behind the cameras, so operators see reactions to moments from about 10 s ago. The operator UI should make this clear.
- Chat messages are stored in Postgres so they can appear in replays.

## 9. Safety and moderation

- **Moderation:** the host (and optional co-hosts) can delete messages, ban viewers, and turn on slow mode. A profanity filter runs on messages before they are broadcast.
- **Private events:** optional viewer password or invite-only guest list.
- **Viewer accounts:** by default viewers only enter a display name, no sign-up. The host can turn on **"Signed-in viewers only"** for an event; viewers then sign in with Google or an email magic link (Supabase Auth). Bans on anonymous viewers are tied to the browser (a device ID in local storage) and are easy to get around; bans on signed-in viewers are tied to the account. The host can switch the setting on during the event if the chat gets out of hand; viewers already watching are asked to sign in to keep chatting.
- **No-filming periods:** the host can pause all cameras (for example during a private or restricted part of the event). Viewers see a "Paused by host" screen and nothing is recorded during that time.
- **Camera removal:** the host can remove any camera immediately.
- Event codes are long and random, so links cannot be guessed. LiveKit tokens are issued by the API only, short-lived, and scoped to one room and one role.

## 10. After the event

### Synced multi-camera replay

- Every camera, the grid, and main-feed switches are stored with timestamps on a shared event clock.
- The replay page shows one timeline; viewers switch cameras or grid and stay at the same moment.
- Chat and reactions replay alongside the video.

### Automatic highlight reel

1. After the event, the media worker scans `reaction_counts` for **spikes**: seconds where reactions are much higher than the recent average.
2. For each spike, it takes a window from **1 minute before to 2 minutes after** the spike (3 minutes total).
3. Camera choice: a spike on a camera channel uses that camera. A spike on the event channel uses whatever the main feed showed at that time.
4. Overlapping windows on the same camera are merged into one clip.
5. Clips are joined **in chronological order** into one highlight video, stored in R2.
6. Uses the HQ local backup when available, otherwise the Stream recording (Cloudflare Stream can create clips by start/end time).

The host can review and remove clips before sharing.

### Video guestbook

Remote viewers record a short video message for the host (browser `MediaRecorder`, time-limited), uploaded to R2. The host sees them all after the event.

### Download package (host)

The host downloads from the event dashboard:

- **Grid video:** the grid is recorded as its own Stream live input. The dashboard's "Download grid" button asks Cloudflare Stream to create an MP4 of that recording, waits until it is ready, and gives the host a download link. Audio-only (M4A) is also available.
- **Each camera:** same flow per camera, using the HQ backup file from R2 when it exists.
- **Highlight reel** and **guestbook videos** from R2.
- **Chat log** as a text or CSV file.
- **Everything at once:** the media worker builds a ZIP in R2 and emails the host a link.

Note: Cloudflare bills each MP4 download like watching the video once, so downloads are limited per plan.

### Retention (how long files are kept)

| What | Free | Paid | Pro |
|---|---|---|---|
| Each camera's recording | Not recorded | 30 days | 1 year |
| Grid and main feed recording | Not recorded | 1 year | 1 year |
| Highlight reel | — | 1 year | No time limit |
| HQ local backups (R2) | — | 30 days | 90 days |
| Guestbook videos, chat log | — | 1 year | No time limit |
| Archive extension | — | — | Can be bought |

- Deadlines count from the day the event ends.
- **7 days before anything is deleted**, the host gets an email with a link to download the package (see above), and the event dashboard shows a countdown.
- A daily cleanup job in the media worker deletes expired Stream videos and R2 files and marks them as deleted in the database. The replay page then only offers what is still kept.
- The main feed recording is built from the main feed log and the camera recordings before the per-camera recordings expire.
- Why: per-camera recordings in Cloudflare Stream are the main storage cost (a 3-hour event with 10 cameras and the grid is about 2,000 stored minutes, roughly $10 per month). The long-lived part is the small, valuable part: grid, main feed, highlights.

### Host analytics

Peak and total viewers, watch time per camera, most-watched camera, and the most-reacted moments (which link into the replay).

## 11. Backend and data

- **Next.js (TypeScript)** app on **Vercel**: web pages plus API routes.
- **Supabase**: Postgres database, authentication (hosts need accounts; viewers and camera operators are anonymous with a display name by default; the host can require viewers to sign in, see section 9), Realtime.
- **Media worker:** a small background service running FFmpeg jobs (highlight reels, ZIP packages). Vercel functions have time limits, so this runs separately (for example Cloudflare Containers, Fly.io, or Railway). Not needed until Phase 6.

### Sources of truth

Each kind of data has one owner. Everything else reads from it. Rules for code are in `CODEBASE_RULES.md` sections 1-3.

| Data | Owner |
|---|---|
| Events, cameras, settings, main feed, audio source, bans, messages, recordings list | Postgres |
| Who is connected and publishing right now | LiveKit, reported by webhooks and stored in Postgres |
| Live and recorded video | Cloudflare Stream (Postgres stores IDs and timestamps) |
| Large files | R2 (Postgres stores object keys) |
| Payments | Stripe, mirrored into `subscriptions` and `event_passes` by webhooks |

Realtime channel messages are notifications, not state. A viewer who joins late loads the current main feed, audio source, and event status from the database, then listens for changes.

### Who decides

| Decision | Made by |
|---|---|
| Event status, plan limits, main feed, audio source, auto-director switches, bans, reaction counts | API (server) |
| Mute or remove a camera, pause filming, pick main feed or audio source | Host sends a command; the API checks the host owns the event, saves the change, then broadcasts it |
| Camera, mic, zoom, flip, local recording on a device | That camera page |
| Which camera a viewer watches, volume, picture-in-picture | That viewer page |

The server never trusts identity, role, or plan sent by a client. It reads them from the Supabase session, the LiveKit token, or the camera's rejoin key.

### Core entities (draft)

| Entity | Key fields |
|---|---|
| `users` | id, email, plan (hosts, and viewers who signed in) |
| `events` | id, host_id, code, title, status (scheduled/lobby/live/standby/paused/ended), plan_tier, audio_source_camera_id, main_feed_camera_id, auto_director, password_hash, require_viewer_sign_in, starts_at |
| `event_schedule_items` | id, event_id, title, starts_at |
| `cameras` | id, event_id, rejoin_key_hash, label, sort_order, pinned, hidden, operator_name, device_type (phone/laptop/pro), status, livekit_participant_id, stream_input_id, hq_backup_key |
| `camera_health` | camera_id, battery, network_quality, updated_at |
| `main_feed_log` | event_id, camera_id, switched_at, switched_by (host/auto) |
| `messages` | id, event_id, camera_id (null = whole event), author_name, body, deleted, created_at |
| `bans` | event_id, viewer_id (anonymous device ID), user_id (null if anonymous), created_at |
| `reaction_counts` | event_id, camera_id, emoji, bucket_second, count |
| `recordings` | id, event_id, camera_id (null = grid), kind (camera/grid/main_feed), stream_video_id, started_at, duration, expires_at, deleted_at (many segments per camera if it reconnects) |
| `highlights` | id, event_id, camera_id, start_at, end_at, peak_at, included |
| `guestbook_entries` | id, event_id, author_name, video_key, created_at |
| `viewer_sessions` | event_id, viewer_id, camera_id, started_at, ended_at (for analytics) |
| `subscriptions` | user_id, stripe_customer_id, tier, status, period_start, period_end, included_viewer_minutes, used_viewer_minutes |
| `event_passes` | id, event_id, user_id, stripe_payment_id, viewer_cap, camera_cap, status |

## 12. Plans and paywall (draft)

Costs grow mainly with **viewer minutes**, then with cameras, recording, and downloads. Pricing should follow that.

| | Free | Paid (event pass) | Pro (subscription) |
|---|---|---|---|
| Cameras (max per event) | 3 | 10 | 16, including RTMP pro cameras |
| Grid tiles | Up to 3 | Up to 9 | Up to 9 |
| AI virtual cameras (after v1) | No | No | Yes, separate limit (for example 2) |
| How it is paid | Free | One-time payment per event | Monthly subscription |
| Viewers | Up to 50 | Pass steps, for example up to 100 / 500 / 2,000 | Monthly allowance of events or viewer minutes, overage billed |
| Grid, main feed, chat | Yes | Yes | Yes |
| Recording, replay, highlights | No | Yes | Yes |
| Downloads | No | Limited | More |
| Retention | — | Cameras 30 days, grid/main feed/highlights 1 year | Cameras 1 year, highlights no limit, extension available |
| Custom branding | No | No | Yes |

### Pricing model

Two kinds of hosts pay differently:

- **One-off hosts** (a wedding, a graduation, a birthday) hold an event once in a while. They buy a **Paid event pass** for one event. Pass price steps follow the viewer cap, because viewer minutes are the main cost (a 3-hour event costs roughly $18 in delivery at 100 viewers, $90 at 500, and $360 at 2,000).
- **Regular hosts** (videographers, venues, religious communities, schools, sports clubs) hold many events. They take a **Pro subscription** with a monthly allowance of events or viewer minutes; usage above it is billed as overage.
- **Free** is for trying the product: 3 cameras, up to 50 viewers, no recording.

Exact prices are not set yet. Rule of thumb: a pass or plan should cost about **2-3 times our expected cost** for it. Prices are fixed after the first test events show real LiveKit, Cloudflare, and storage costs.

Payments via **Stripe Checkout**: one-time payments for event passes, Stripe Billing for subscriptions, and metered usage for Pro overage.

## 13. Infrastructure, testing, costs

### Services

| Need | Service |
|---|---|
| Web app + API | Vercel |
| Camera side (WebRTC), pro camera ingress, egress | LiveKit Cloud |
| Audience delivery, recording, clips, MP4 downloads | Cloudflare Stream |
| HQ backups, guestbook, highlights, packages | Cloudflare R2 |
| Media processing | Media worker (FFmpeg) |
| Database, auth, realtime | Supabase |
| Payments | Stripe |

Environments: local, staging, production. Each has its own keys in `.env` files (never committed).

### Testing

- **Unit tests** for API logic (token issuing, plan limits, spike detection, clip window merging, auto-director switching rules).
- **Playwright end-to-end tests** with Chrome's fake camera (`--use-fake-device-for-media-stream`, `--use-fake-ui-for-media-stream`): open several camera pages and viewer pages at once, then check camera switching, main feed, grid, mute, per-camera chat, and moderation.
- **Manual device matrix** before each release: iPhone Safari, Android Chrome, macOS/Windows laptop, on Wi-Fi and cellular.
- GitHub Actions runs lint and tests on every PR.
- What to test and how is in `CODEBASE_RULES.md` section 12.

### Cost drivers (check current pricing before relying on these)

- Cloudflare Stream: $1 per 1,000 minutes delivered, $5/month per 1,000 minutes stored. MP4 downloads bill like one viewing. Example: 2,000 viewers × 3 h = 360,000 minutes, about $360 per event.
- LiveKit: WebRTC minutes, bandwidth, and transcode minutes for egress (about cameras + 1 grid × event length), plus ingress for pro cameras.
- R2: storage for HQ backups (large files, about 3 GB per phone per hour; kept 30 days on Paid, 90 days on Pro, see section 10).
- Supabase, Vercel: free tiers during development.

## 14. Delivery phases

1. **Spike:** a phone and a laptop publish into a LiveKit room; one watch page (WebRTC); mic mute. Prove it on real devices over cellular.
2. **Multi-camera event:** create event, QR join, device picker, zoom/flip, host director page with camera health, labels, pins, remote mute, audio source, main feed, lobby.
3. **Scale path:** egress to Cloudflare Stream, grid composite, HLS viewer page with main feed, camera switching, grid, picture-in-picture, rewind while live, schedule and countdown.
4. **Chat, reactions, moderation:** realtime channels, batching, auto-director, moderation tools, private events, no-filming pause.
5. **Accounts and paywall:** host accounts, Stripe, plan limits.
6. **After the event:** recordings, local HQ backup upload, synced replay, highlight reel, guestbook, downloads, analytics.
7. **Pro tier:** RTMP pro cameras, custom branding.
8. **AI features (after v1):** see section 15.

## 15. Not in v1

| Feature | Status |
|---|---|
| Live captions and translation | Planned for a later update |
| Clip sharing by viewers ("save last 30 s") | Not planned |
| Landscape and stability guidance for operators | Not planned |
| Virtual gifts and tips | Not planned |
| Native iOS/Android apps | Only if browser limits become a blocker |
| AI virtual cameras (close-ups cut from a wide shot) | Planned for a later update (Pro) |
| AI picture enhancement after the event | Planned for a later update |
| AI director using picture quality | Planned for a later update |
| AI-generated camera angles nobody filmed | Not planned |

### AI features (after v1)

1. **AI virtual cameras.** One high-resolution camera films a wide shot (a stage, a field, a room). A server-side AI model detects and tracks people (the speaker, the performer, the couple) and crops a second stream from it, such as a close-up. Viewers see it as one more camera in the camera list and grid. It needs a GPU worker per virtual camera for the whole event, so it has its own per-plan limit (Pro only, for example 2) and does not count against the physical camera limit. It only works when the source camera sends enough resolution, so it suits pro cameras and phones on a good connection.
2. **AI picture enhancement after the event.** Stabilization, low-light noise reduction, and sharpening applied by the media worker to recordings, HQ backups, highlight reels, and downloads. Not applied live, because live enhancement costs too much. Paid and Pro plans.
3. **AI director using picture quality.** The auto-director (section 5) keeps using reactions, and also scores each camera's low-resolution preview for blur, shake, darkness, and whether faces or the main action are in frame. Cameras that point at the floor or are too dark are skipped when switching the main feed, and the host director page warns about them. Uses the previews the system already receives, so the cost is low.

AI-generated viewpoints that no camera filmed (novel view synthesis) are not planned: they cannot run live, take a long time to compute, and show visible artifacts.

## 16. Risks

- iOS browser limitations (screen lock, background tabs, battery, heat, `MediaRecorder` differences).
- Poor venue connectivity for camera phones.
- Egress, delivery, and download costs at large audiences; pricing must cover them.
- Viewer delay (5-15 s) makes live interaction with camera operators feel slower.
- HQ backup uploads may never finish if operators leave early.
- Privacy and consent of people being filmed.
- Filming children (school events, youth sports) needs parental consent; held back from the launch focus.
- Music rights: streaming live music (concerts, and music played at any event) can break copyright and get streams blocked; concerts are held back from the launch focus.

## 17. Open questions

- Exact prices for event pass steps and the Pro subscription (set after the first test events show real costs).
- Domain name (postponed). Checked on 2026-09-29: yourpov.com is taken (since 1999); yourpov.app is registered (October 2025, Namecheap; check whether a team member owns it); yourpov.live and yourpovlive.com looked free in the registries.
