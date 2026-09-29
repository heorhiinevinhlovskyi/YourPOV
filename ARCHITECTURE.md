# YourPOV — Architecture

_Status: draft. Fill in each section as decisions are made. Record the reasoning in `DECISIONS.md`._

## 1. Product overview

- What is YourPOV?
- Who are the users (broadcasters, viewers)?
- Live streaming, video on demand, or both?
- MVP scope: what is in / out for the first version?

## 2. System overview

_High-level diagram and a short description of how the parts connect._

```
Broadcaster -> Ingest -> Transcoding -> Storage -> CDN -> Player (viewer)
                                  Backend / Auth (alongside everything)
```

## 3. Ingest / transcoding

- Ingest protocol(s): RTMP / SRT / WebRTC / file upload?
- Build (FFmpeg) or managed service (Mux, Cloudflare Stream, AWS)?
- Output format: HLS / DASH, renditions (1080p, 720p, ...), segment length
- Owner:

## 4. Playback / CDN

- CDN provider:
- Player (web / iOS / Android):
- Adaptive bitrate, latency target:
- Owner:

## 5. Auth / backend

- Auth provider:
- API language and framework:
- Database:
- Core entities (users, channels, streams, videos, ...):
- Protecting video access (signed URLs, tokens):
- Owner:

## 6. Clients

- Web / mobile / both?
- Framework:

## 7. Infrastructure and deployment

- Hosting:
- Environments (local, staging, production):
- CI/CD:

## 8. Testing strategy

- Unit / integration / end-to-end
- What runs in CI on every PR

## 9. Cost estimates

- Expected costs for ingest, transcoding, storage, CDN at MVP scale

## 10. Open questions

- 
