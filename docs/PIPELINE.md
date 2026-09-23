# Pipeline v2: native iOS, ARKit, low latency

Decided on Sep 23, 16:00. This file replaces the web-only pipeline and is the source of truth for recording, processing and live guidance. The README still covers the idea, the web part (business onboarding + map) and the timeline.

## 0. Demo-first scope (Sep 23, evening, overrides the rest)
The backend doesn't need to be real. What counts is what the demo and the video show. So:

| Part | Real | Faked |
|---|---|---|
| Recording (ARKit path, marks, world map) | yes, on the phone | |
| Waypoints | built **on the phone** from marks + path turns | |
| Instructions | templates on the phone ("Turn left at the <mark>") | Gemini labeling = stretch |
| Live guidance (floor arrows, giant arrow, haptics, earcon) | yes, on the phone, fully offline | |
| Gemini Live voice | yes, through **one tiny token endpoint** (the API key never goes into the app) | Apple TTS if it fails |
| Server processing, keyframe pipeline, storage | | route stays on the phone, sharing = AirDrop/file or not at all |
| Business web page (checkbox, entrance markers, score) | frontend only | hardcoded data, no API |
| 3D map for users | frontend only | fixture entrances |

Consequences:
- **VIDEO** shrinks to: token endpoint + (stretch) Gemini labeling. It helps IOS-REC build waypoints on the phone first.
- **CORE/MAP** build the web pages against static JSON, no database, no uploads.
- Section 2 (server processing) is only built once the on-device flow runs end to end.

## 0b. Landmarks, not a replayed path (core idea)
People don't walk the same line twice, so guidance must not replay the recorded path. It navigates **from landmark to landmark**.

- **Landmark graph**: every mark and turn becomes a node with a 3D position in the world map, a keyframe and a name. The recorded walk only provides the edges (which landmarks connect). More recordings of the same place (owner + community) add nodes and edges, so the graph grows.
- **Recognition on three levels**:
  1. **ARKit world map** (geometry, instant): recognizes the room from distinctive visual feature points, however the user walks in. This is the "AR level", and it runs at 60 Hz.
  2. **Apple Vision feature prints** (appearance, about 20 ms): the current frame against each landmark keyframe, which catches landmarks when relocalization is weak.
  3. **Gemini** (meaning, about 1 s): names landmarks when recording ("green door with WC sign") and confirms them live ("I can see the WC sign").
- **Guidance**: the arrow points to the **next landmark node**, not along the old line. If the user skips a landmark or comes from another side, the app picks the nearest node that still leads to the target (shortest path in the graph).
- **Walls**: we have no floor plan, so the arrow never points straight to the target through walls, only along graph edges we know are walkable. With LiDAR the scene mesh can later refine this.
- **Demo**: record two different walks to the toilet (e.g. via the stairs side and via the corridor). The user walks a third way, and the app still finds the landmarks.

## Decisions
| Topic | Decision |
|---|---|
| Platform | **Swift app** for recording and live guidance. **Web** (Next.js) for business onboarding, map and the server. |
| Position | **ARKit** records the real 3D path in meters plus a **world map**. At guidance time ARKit relocalizes in the same space. No compass, no step counting. |
| Live recognition | **On the phone**: ARKit pose at 60 Hz, Apple Vision feature prints as backup. **Gemini Live** speaks, answers questions and watches for hazards. |
| Voice | **Gemini Live** native audio. Apple TTS only as fallback. |
| Labeling | **Gemini** on the server (video + audio + ARKit path). |
| Hugging Face | Not in the core. Stretch only: Depth Anything V2 as Core ML for obstacle distance on phones without LiDAR. |

Principle: **everything that moves the arrow runs on the phone, and everything that needs understanding runs in Gemini.** The network is never in the critical path of the arrow.

## Latency budget (to be measured, these are targets)
| Step | Where | Target |
|---|---|---|
| Position update | ARKit on device | 16 ms (60 Hz) |
| Arrow and floor path update | RealityKit | 1 frame |
| Waypoint reached → haptic + earcon | device | < 50 ms |
| Spoken direction | Gemini Live | 0.8 to 1.5 s, **hidden** by triggering 2.5 m before the turn |
| Hazard warning from camera | Gemini Live | about 1 s |
| Recording → route live | server + Gemini | < 60 s for a 1-minute walk |

## Overview
```
RECORD (iOS)                PROCESS (server)                 GO (iOS)
ARKit session          ──►  path → turns, stops         ──►  load world map → relocalize
 video + audio              + marks + Gemini events           floor path + arrow (RealityKit)
 camera path 10 Hz          → waypoints                       pose → progress, triggers
 marks (tap)                keyframes, signatures             Gemini Live: voice, Q&A, hazards
 world map                  route.json                        haptics, spatial earcon
        └── REVIEW on the phone: confirm waypoints ──┘
```

## 1. Record (iOS, `ios/Doorway/Record/`)
**Session**
- `ARWorldTrackingConfiguration`: horizontal plane detection, `providesAudioData = true`, scene reconstruction `.mesh` on LiDAR devices.
- The screen shows the camera, a "Stop" button and **four big mark buttons**: Door, Step/Ramp, Elevator, Toilet. A tap drops a named `ARAnchor` at the current position. Marks are the most reliable labels we have and cost the person filming one second.
- **Narration**: the person says what they see. The audio goes into the video.
- Guidance on screen: hold the phone at chest height, walk slowly, pause briefly at turns.

**What gets written**
| File | Content |
|---|---|
| `video.mp4` | `ARFrame.capturedImage` scaled to 720p plus audio sample buffers, through `AVAssetWriter` (H.264, 30 fps) |
| `path.json` | camera pose at 10 Hz: `t`, `x y z` (meters), `yaw`, tracking state |
| `marks.json` | tap marks: `t`, `type`, anchor id, position |
| `worldmap.arwmap` | `session.getCurrentWorldMap()` archived with `NSKeyedArchiver`, fetched when the world mapping status is `.mapped` or `.extending` |

**Start pose**: the recording starts at the entrance, facing the door. The first pose is the origin. This also matters for the fallback in 4.3.

**Upload**: multipart to `POST /api/routes` with the four files and `placeId`, plus a progress bar. The server answers with a `routeId` and the app polls the status.

## 2. Process (server, `app/api/routes`, `lib/route/`)
1. **Validate**: file types, size (video max 200 MB, 3 min), consistent `path.json`.
2. **Path geometry** (no AI, milliseconds):
   - resample the path to 0.25 m steps and simplify it (Douglas-Peucker, 0.3 m) → `polyline`
   - **turns**: a yaw change of more than 40° within 2 m of path → turn with `turn_delta_deg`
   - **stops**: more than 1.5 s with less than 0.2 m of movement
   - **level change**: y changes by more than 0.3 m → stairs, ramp or elevator candidate
   - `dist_m` along the path for every point
3. **Gemini pass 1, whole video with audio.** Input: the video plus a text list of the candidates (time, type, distance) and the marks. Output is structured JSON `events[]` (contract below): what is there, visible text (sign OCR), what was said. Model: Flash-class for speed, set in `GEMINI_MODEL`.
4. **Merge into waypoints**: marks > sensor candidates > Gemini-only events. Merge anything less than 1.5 m apart. Geometry decides where, Gemini decides what.
5. **Keyframes** (ffmpeg): `image` = the sharpest frame around the waypoint, `approach_image` = the frame from 3 m before it, which is what the next person sees while walking toward it.
6. **Gemini pass 2, one call per waypoint, in parallel**, on both keyframes:
   - `signature`: a visual fingerprint made only of fixed things (colors, signs, doors, fixtures), never people
   - `instruction` {de,en}, `instruction_wheelchair` {de,en}, `blind_hint` {de,en} (door side, handrail, floor change, distance in steps)
   - `hazards[]`, `wheelchair_ok`
7. **Entrance details** for web onboarding: steps, estimated door width, automatic door, bell, tactile guidance, each with `source: ai`, `confirmed: false`.
8. Write `route.json`, keyframes and the world map into `data/routes/<id>/`. Status becomes `review`.

## 3. Review (iOS, right after upload)
- Waypoint list: keyframe, instruction, turn icon. Edit the text, delete a waypoint or merge it with the next one.
- Entrance details prefilled, confirmed with one tap.
- "Publish" sets the status to `live`. Nothing reaches users unconfirmed.

## 4. Go (iOS, `ios/Doorway/Go/`)
### 4.1 Start
1. Download `route.json`, keyframes and the world map (cached, so guidance works offline once loaded).
2. `ARWorldTrackingConfiguration.initialWorldMap = map`. The screen shows the **start `approach_image`** with the text "Point the camera at the entrance". The same text is spoken, and VoiceOver reads it.
3. When the tracking state changes from `limited(.relocalizing)` to `normal`, we know where we are in the recorded space.

### 4.2 Guidance loop (every frame, on device)
- Project the camera position onto the `polyline` → `dist_m` walked, distance to the next waypoint.
- **Trigger zones** per waypoint:
  - **2.5 m before**: send the direction to Gemini Live so it's spoken in time (hides the latency)
  - **0.8 m before**: haptic plus earcon from the direction to turn
  - **reached** (≤ 1 m, or passed along the path): next waypoint
- **Off route**: more than 2 m from the polyline for 3 s → "Stop", the arrow points back to the path, Gemini says it.
- Wheelchair profile: hazards like steps are announced early and prominently.

### 4.3 Fallbacks when relocalization fails (after 10 s)
1. **Start-pose alignment**: the user stands at the entrance facing the door (the same start pose as the recording). The session runs without a map and treats the start as the recorded origin. ARKit odometry is accurate to about 1 to 2 % of distance, which is enough for 50 m.
2. **Vision check**: every 0.5 s, `VNGenerateImageFeaturePrintRequest` compares the frame with the next waypoint's `approach_image` (about 20 ms). A match corrects the progress along the path.
3. **Step cards**: keyframes plus instructions, tap to advance. Always available.

### 4.4 Gemini Live (`ios/Doorway/Live/`)
- The server mints a short-lived token (`POST /api/live/token`). The API key never reaches the phone.
- `URLSessionWebSocketTask` to the Live API. Upstream: mic PCM 16 kHz plus camera frames at 1 fps (512 px JPEG). Downstream: PCM 24 kHz audio into `AVAudioEngine`.
- **System instruction**: the route as compact text (waypoints, landmarks, hazards), the profile (wheelchair / blind), language (de/en). Keep answers short: one sentence, no small talk.
- **The phone decides the position, Gemini speaks.** The app sends events as text ("User 2.5 m before w4, next: turn left at the green door with the WC sign") and Gemini turns them into natural speech.
- Gemini tools:
  - `warn(hazard, severity)` for things it sees in the camera: people, an open door, a wet-floor sign, a blocked way
  - `answer` for free questions ("Is there a handrail?")
  - no tool for moving the waypoint; position stays on the device
- If Live is not connected within 3 s, or drops: the instruction texts go to `AVSpeechSynthesizer`, and the app reconnects in the background.

### 4.5 Views (same state for all)
- **AR view**: RealityKit chevrons on the floor along the polyline to the next waypoint, a large 3D arrow at the turn, a distance label.
- **Giant arrow**: full-screen SwiftUI arrow, rotated by the real angle from the AR pose to the next point. Contrast modes black/yellow, black/white and white/black. Shape and brightness carry the meaning, so it is color-blind safe. Dynamic Type, VoiceOver.
- **Blind profile**: giant arrow by default with the screen dimmed. **Spatial earcon** through `AVAudioEnvironmentNode`, so the tone comes from the actual direction of the next waypoint, which works best with headphones. Core Haptics patterns: left = two short, right = three short, arrived = one long.

## 5. Testing and the stage demo
- **Replay**: Reality Composer can record AR sessions, and Xcode can replay them through the scheme option "ARKit Replay Data" (verify on Xcode 26). Record take 2 of the walk once and run Go against it in Xcode, no walking needed.
- **Run log** `data/routes/<id>/runs/<ts>.jsonl`: time, position, waypoint, trigger, relocalized yes/no, Gemini latency. Latency numbers for the pitch come from this.
- **Stage demo, two options**:
  1. Safe: the real walk in CODE University as a screen recording, narrated live.
  2. Wow: film a 10 m route on stage (stage → exit door), processed in under 60 s, then someone walks it with the giant arrow. Only if option 1 is ready as a backup.

## 6. Roles (update to README)
| Role | Owns | Delivers |
|---|---|---|
| **IOS-REC** | `ios/Doorway/Record/`, `ios/Doorway/Shared/` | AR recording, marks, world map, upload, review screen |
| **IOS-GO** | `ios/Doorway/Go/` | relocalization, floor path, giant arrow, triggers, fallbacks, haptics, earcons |
| **LIVE** | `ios/Doorway/Live/`, `app/api/live/` | token route, WebSocket client, audio in/out, system instruction, tools |
| **VIDEO** | `app/api/routes/`, `lib/route/` | geometry, Gemini passes 1 + 2, keyframes, `route.json` |
| CORE, MAP, PITCH | as in README | web scaffold + deploy, onboarding + map, pitch |

Xcode: one project in `ios/`, Swift 6, iOS 18+, **folder-synchronized groups**, so new files don't touch `project.pbxproj` and parallel terminals don't collide. Only IOS-REC changes project settings.

## 7. Milestones
| Time | IOS-REC | IOS-GO | LIVE | VIDEO |
|---|---|---|---|---|
| 17:00 | AR session + recording writes all 4 files on the iPhone | giant arrow + step cards from the fixture | token route + WebSocket connects, text in, audio out | geometry from a fixture `path.json` |
| 19:00 | upload + status polling | world map load + relocalize, floor path | camera frames + mic, `warn` tool | Gemini pass 1 + merge → first real `route.json` |
| 21:00 | review screen | triggers, haptics, earcons, off route | early-trigger speech from IOS-GO events | keyframes + pass 2 + entrance details |
| 23:00 | first real walk in CODE University, two takes | fallbacks 4.3 | Apple TTS fallback, reconnect | demo route cached |
| 07:00 | | | | full rehearsal with run log |

## 8. Contracts
**`path.json`**
`{ "hz": 10, "origin": "entrance", "samples": [{ "t": 0.1, "p": [0.0, 0.0, -0.4], "yaw": 3.1, "tracking": "normal" }] }`

**`marks.json`**
`[{ "t": 12.4, "type": "door|step|ramp|elevator|toilet", "anchor": "UUID", "p": [x, y, z] }]`

**Gemini pass 1 `events[]`**
`{ "t": 12.4, "type": "door|turn|stairs|ramp|elevator|sign|toilet|obstacle|narrow", "text_seen": "WC", "said": "toilet on the left", "description": "..." }`

**`route.json`**: the README contract plus:
- route: `worldmap`, `polyline` `[[x, y, z]]`, `length_m`, `status` (`processing|review|live`)
- waypoint: `p` `[x, y, z]`, `dist_m`, `turn_delta_deg`, `approach_image`, `signature`, `instruction_wheelchair` {de,en}, `source` (`mark|geometry|gemini`)
- `heading_deg` and `steps_est` are dropped

**Live events, app → Gemini (text)**
`{ "event": "approach|reached|off_route|arrived", "waypoint": "w4", "dist_m": 2.5, "text": "..." }`

## 9. Still open (defaults apply if nobody objects)
1. Does the demo iPhone have **LiDAR** (Pro model)? Default: build without it, and use mesh only if available.
2. **Route**: entrance → toilet at CODE University, two takes tonight 20:00 to 21:00. The museum is off.
3. **Language of the voice** on stage: default German.

## 10. App entry (both sides in one app)
The iOS app starts with one choice: **"I run a business"** → Record + Review, or **"I'm visiting"** → profile, place, Go. The choice is remembered and can be switched in settings. Owner: IOS-REC builds the start screen in `ios/Doorway/Shared/`.
