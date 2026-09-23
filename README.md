# Google Doorway: Master Plan

> **Update Sep 23, 16:00:** recording and live guidance are now a native Swift app with ARKit (real 3D path + world map), Gemini Live for voice and hazards. Details, roles and milestones: `docs/PIPELINE.md`. Where this README and PIPELINE.md disagree, PIPELINE.md wins.
Google x Aktion Mensch Hackathon "AI Agents for Inclusion", Berlin. Start Sep 23, 14:00. **Code freeze Sep 24, 10:00. Pitch 12:00.** 20 hours.

## The problem
Wheelchair users and blind people don't know in advance which entrance works for them or how to get from that entrance to the toilet. Google Maps has almost no data on either.

## The idea
1. **Business owner**: in the Google Business Profile, ticks "Accessibility", drags each entrance to the right spot on the map, and films the route from the main entrance to the toilet once. AI fills in the entrance details from the video, and the owner only confirms.
2. **Community**: anyone can film routes too, like Local Guides, and earns points.
3. **User**: sees the entrances in the 3D map, then gets guided inside the building to the toilet, either with arrows on the floor in the camera view or with a giant high-contrast arrow. A Gemini Live agent recognizes where you are from the camera and speaks the directions.
4. **Win-win**: businesses with maintained accessibility data rank higher in search. We pitch this to Google as a product proposal. In the demo it's a simulated visibility score.

**The video is the core.** Everything else can be leaner.

## Name and repo
**Google Doorway**. Folder `Desktop/APP/doorway`, private repo `sami-mag07/doorway`, Malek as collaborator.

## Architecture
- **Web app**: Next.js (App Router, TypeScript), mobile first, German + English. Deployed to our VPS with Docker + Caddy under `*.5.45.109.195.sslip.io`. HTTPS is required for camera and sensors on the iPhone.
- **iOS app**: Swift + ARKit/RealityKit with real arrows on the floor. It reads the same `route.json` from the server.
- **AI**: Gemini video analysis through a server route. The **Gemini Live API** handles live guidance. The browser only gets short-lived tokens from the server. Model names come from `.env` and get checked against current docs while building.
- **Map**: Maps JS API. `Map3DElement` for users, a 2D map with draggable markers for business owners, Places search.
- **Storage**: JSON files on the server (`data/places/<id>.json`, `data/routes/<id>/route.json` + keyframe images). No database.
- **Keys are still coming**: `.env.example` with `GEMINI_API_KEY`, `GEMINI_MODEL`, `GEMINI_LIVE_MODEL` and `NEXT_PUBLIC_MAPS_API_KEY`. `.env*` goes in `.gitignore`. The Maps key is restricted to our domain. Until the keys arrive, everyone builds against fixture files.

## Video pipeline (the core)
1. **Recording in the app**: while filming, the web app also records compass and gyroscope data (`DeviceOrientation`/`DeviceMotion`). That gives every turn to the degree, even when the image shakes. You can also upload an existing video, just without the sensor trace.
2. **Finding waypoints**: the server merges the sensor trace with the video. Every change of direction, door, step or elevator becomes a waypoint.
3. **Gemini labels them**: it gets the video plus the candidate timestamps and returns, per waypoint, a landmark ("green door with WC sign"), an instruction in DE + EN, hazards (step, heavy door, narrow hallway) and suitability for wheelchair and blind users. It also returns the entrance details for onboarding: steps, estimated door width, automatic door, tactile guidance.
4. **Keyframes**: the server cuts a still image at every waypoint (`ffmpeg-static`). The live agent uses these as references to recognize places in the camera feed.
5. **Output**: `route.json` (contract below).

## Live guidance (two views, one agent)
- **Agent**: a Gemini Live session gets `route.json` + the keyframes as context, plus the camera feed at about 1 frame per second. Its tools are `go_to_waypoint(id)`, `warn(hazard)` and `arrived()`. It speaks the directions itself.
- **View 1: camera + floor arrow** (web, fake AR): a perspective arrow at the bottom of the camera image that turns with the compass toward the next waypoint. A step card sits on top.
- **View 2: giant arrow**: the camera keeps running in the background, and the screen shows only a full-screen arrow. Contrast modes are black/yellow, black/white and white/black. It is color-blind safe because only shape and brightness carry meaning. It adds vibration on turns, voice directions and VoiceOver labels.
- **View 3: ARKit** (iOS app, bonus track): real arrows on the detected floor, same route.
- **Offline fallback**: step cards with keyframes, tap to advance.

## Web app screens
1. `/` Profile choice (wheelchair / blind / both), search, a 3D map with entrance markers (shape + color per type), a profile filter and a "Take me to the toilet" button.
2. `/business` Onboarding: accessibility checkbox, entrances placed on the map, a "Film the route" task, AI-filled entrance details to confirm (type, steps, door, blind aids, photo, note) and the visibility score.
3. `/record/[placeId]` Recording with sensor trace, upload, analysis progress and a waypoint preview.
4. `/go/[routeId]` Live guidance with a camera / giant arrow toggle.
5. `/community` Film routes for places without data, earn points.

## Contracts (change only through ORCH)
**`data/places/<id>.json`**
- Place fields: `id`, `name`, `placeId` (Google), `lat/lng`, `score`
- `entrances[]`: `id`, `lat/lng`, `main` (bool), `type` (`step_free|ramp|steps|elevator`), `steps`, `door` (`width_cm`, `automatic`, `bell`, `heavy`), `blind` (`tactile`, `announcement`, `contrast`), `photo`, `note`, `source` (`owner|community|ai`), `confirmed`
- `routes[]`: route IDs

**`data/routes/<id>/route.json`**
- Route fields: `id`, `placeId`, `from` (entrance id), `to` (`toilet|elevator|reception|…`), `video`, `duration_s`
- `waypoints[]`: `id`, `t` (second in the video), `image`, `heading_deg`, `turn` (`start|straight|left|right|u_turn|up|down|arrive`), `landmark`, `instruction` `{de,en}`, `hazards[]`, `steps_est`, `wheelchair_ok`, `blind_hint` `{de,en}`

CORE creates fixture files for the demo location (Google AI Center Berlin) in the first hour. Everyone builds against them.

## Roles (orchestrated Claude terminals, like FollowCam)
| Role | Owns | Delivers |
|---|---|---|
| **ORCH** (Sami, main terminal) | `docs/`, contracts, merge, deploy | everything integrated, demo runs |
| **CORE** | Next.js scaffold, `lib/`, `app/api/places`, storage, Docker/Caddy | app online, fixtures, i18n DE + EN |
| **MAP** | `/`, `/business`, map components | 3D user map, onboarding with markers, score |
| **VIDEO** | `/record`, `app/api/routes`, `lib/video/` | recording with sensor trace, waypoints, Gemini labeling, keyframes, `route.json` |
| **LIVE** | `/go`, `lib/live/`, token route | Gemini Live agent, camera floor arrow, giant arrow with contrast modes |
| **IOS-REC** | `ios/Doorway/Record/`, `Shared/` | AR recording, marks, world map, upload, review |
| **IOS-GO** | `ios/Doorway/Go/` | relocalization, floor path, giant arrow, haptics, fallbacks |
| **PITCH** | `pitch/` | German deck, 3-minute demo script, numbers |
| **QA/DESIGN** (designer + skeptic agents) | reports only | accessibility of the app itself, design check, demo risks |

Rules:
1. Each role only touches its own paths.
2. Report milestones to ORCH.
3. Keep commits small.
4. The timebox beats perfection: anything not working by its deadline gets the fallback.

## Agent communication
- **Same Mac**: terminals message each other directly (SendMessage), and ORCH coordinates.
- **Across both Macs**: `docs/board/<ROLE>.md` is each role's inbox, and `docs/board/STATUS.md` holds the overall state. Commit + push after every milestone, pull before every new task.

## Timeline
| Time | Goal |
|---|---|
| **14:00–15:00** | Master plan as `docs/ORCHESTRATION.md` + `docs/tasks/<ROLE>.md`, repo, scaffold deployed, fixtures. Film the entrance → toilet route in the AI Center (on a phone, 2 takes). |
| **15:00–18:00** | Every track runs alone against the fixtures. Add keys as soon as they arrive. |
| **18:00** | Checkpoint: first real `route.json` from our video, map shows entrances, giant arrow runs on the iPhone. |
| **18:00–22:00** | Gemini Live connected, onboarding fills itself from the video, ARKit shows arrows. |
| **22:00** | Checkpoint: full run from business owner → video → user → live guidance inside the building. |
| **22:00–02:00** | Design pass, contrast modes, VoiceOver, DE + EN, demo cache (demo works offline). |
| **02:00–07:00** | Buffer, sleep in shifts. Fixes only. |
| **07:00–10:00** | Two real rehearsals in the building, pitch deck done. **10:00 freeze.** |
| **10:00–12:00** | Stage rehearsals, record a backup video of the demo. |

## Demo safety
- Demo location with finished video, `route.json` and keyframes cached.
- Giant arrow + step cards work without Gemini Live.
- A recorded backup video of the full flow.
- A static map image in case the Maps key fails.

## Security and privacy
- Keys only in `.env`, and Gemini is called only from the server.
- Live tokens are short-lived.
- Uploads are checked for size and type.
- Videos may show people: we show a notice while filming and only publish keyframes.

## Testing
1. With fixtures: every view renders without keys.
2. A real video from the AI Center produces a `route.json` with waypoints at the real turns.
3. iPhone Safari: camera, sensors and upload work, the arrow turns, the voice speaks.
4. The live agent advances correctly while actually walking (test run with a log).
5. Airplane mode: the demo runs from the cache. The full flow takes under 3 minutes.

## Open points
- Which roles run on Malek's Mac (suggestion: IOS + PITCH)
- Hub on top of the repo board?
- Malek's GitHub username for the repo invite
- What the Antigravity accounts are used for (decide later)

## First steps once approved
1. Create `Desktop/APP/doorway`, git init, `.gitignore` with `.env*`, `.env.example`.
2. This master plan as `README.md`.
3. `docs/ORCHESTRATION.md`, `docs/tasks/<ROLE>.md` per role, `docs/board/`.
4. Create private repo `sami-mag07/doorway` and push.
