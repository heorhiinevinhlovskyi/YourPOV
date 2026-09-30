# Glossary

If you've had to clarify a word twice, it belongs here. Describe it in a sentence or two; link to `ARCHITECTURE.md` for anything longer. If the meaning isn't decided yet, it doesn't belong here.

Voice: present tense, no hedging.

## Event

One live occasion with one host, a set of cameras, and viewers. It lives in our database, not in LiveKit, and stays alive when every camera goes offline. Code: `events`.

## Camera

A device that streams video into an event: a guest's phone or laptop, or a pro camera. A camera connects to a camera slot; when it disconnects, the slot stays and the camera can reconnect to it.

Avoid: _stream_ (an event has many streams), _participant_ (a LiveKit term).

## Camera slot

The place in an event that a camera connects to. The slot, not the device, owns the label, position, pin, chat channel, Stream live input, and recording timeline, so they survive disconnects. A slot has at most one camera connected at a time. Code: `cameras` table (one row per camera slot).

Avoid: _seat_, _port_.

## Camera operator

The person holding a camera. A camera operator has no account; the rejoin key on their device reconnects it to the same camera slot.

## Rejoin key

A random key stored in the camera's browser that reconnects it to its camera slot after a disconnect. Only its hash is stored. Opening the camera link on a new device creates a new camera slot instead.

## Pro camera

A professional encoder (OBS, GoPro, DSLR) that joins over RTMP through LiveKit Ingress. Pro plan only. It appears and behaves like any other camera. Code: `device_type = 'pro'`.

## Main feed

The camera viewers watch by default. The host picks it, or the auto-director switches it. Viewers who choose a camera themselves leave the main feed until they return to it.

Avoid: _main camera_, _live view_.

## Auto-director

A host setting that switches the main feed to the camera with the most reactions in the last 20 seconds, holding each camera for at least 15 seconds. The host can override it at any time.

## Grid

One server-side composite video of up to 9 cameras, played as a single stream. It plays the audio source's sound. It is not a set of separate players.

Avoid: _multiview_, _security view_ (fine in marketing copy, not in code or docs).

## Audio source

The one camera whose microphone plays in the main feed and the grid. The host picks it; otherwise it defaults to the host's camera, then the first camera that joined. It does not follow the main feed.

## Standby

The event state when the event is live but no cameras are connected. Links stay active, chat keeps working, and players reconnect on their own. Different from _paused_, which the host starts on purpose.

## Paused

The event state when the host has stopped all filming (a no-filming period). Nothing is streamed or recorded until the host resumes.

## HQ backup

The full-quality local recording a camera page makes with `MediaRecorder` and uploads to R2. Replays and highlight reels use it when it exists, falling back to the Stream recording.

Avoid: _local recording_ when you mean the uploaded file.

## Plan

What a host can do in an event: **Free**, **Event pass**, or **Pro**. Code: `plan_tier` values `free`, `event_pass`, `pro`.

## Event pass

The paid plan for one event, bought with a one-time payment. It allows more cameras and more viewers than Free, plus recording; the price steps follow the viewer cap. Code: plan tier `event_pass`; purchases in `event_passes`.

Avoid: _Paid plan_, _Paid_.

## Viewer minutes

Minutes of video delivered to viewers, summed across all viewers. The main cost driver and the unit for Pro plan allowances and overage.
