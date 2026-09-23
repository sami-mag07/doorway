# Pipeline: video → labels → live view

How one filmed walk turns into guidance for the next person. Owners: **VIDEO** (stages 1 to 3), **LIVE** (stage 4). Contracts at the bottom change only through ORCH.

```
1 Capture        2 Process (server)          3 Review          4 Live guidance
video.mp4   ──►  sensor turns + Gemini   ──►  owner confirms ──►  dead reckoning (steps + compass)
trace.json       events → waypoints           waypoints          + Gemini Live checks the camera
narration        keyframes, route.json                            → arrow, voice, step cards
```

## 1. Capture (`/record/[placeId]`, phone browser)
1. One tap starts everything. That tap also asks iOS for motion permission (`DeviceOrientationEvent.requestPermission()`), because iOS only allows it from a user gesture.
2. Rear camera via `getUserMedia`, recorded with `MediaRecorder` (Safari records mp4/H.264).
3. **Sensor trace** at about 20 Hz, timestamps relative to the recording start:
   - heading from `webkitCompassHeading` (iOS) or `alpha`
   - acceleration magnitude from `DeviceMotion`, used for step detection
4. **Narration**: the person filming says out loud what matters ("door", "ramp", "toilet on the left"). Gemini hears the audio track, which makes the labels far more reliable than image alone.
5. On-screen guidance while filming: hold the phone at chest height, walk slowly, **stop for 2 seconds at every turn, door and step**. The pauses become clean waypoints.
6. Upload `video.mp4` + `trace.json`. An existing video without a trace also works, with lower quality (see fallback in 2.3).

## 2. Process (server, `app/api/routes`, `lib/video/`)
1. **Validate and normalize**: check type and size (max 200 MB, max 3 min). ffmpeg converts to 720p/30 fps H.264, extracts frames at 2 fps and keeps the audio.
2. **Sensor events** (no AI, fast and exact):
   - smooth the heading, and count a **turn** when it changes by more than 45° within 3 s (store the delta, e.g. -90 = left)
   - a **stop** is more than 1.5 s without steps, which marks a door, step or point of interest
   - count steps between events: `steps_est`, distance ≈ steps × 0.7 m
3. **Fallback without a trace**: ffmpeg scene detection (`select='gt(scene,0.3)'`) produces the candidate timestamps instead.
4. **Gemini pass 1: whole video with audio.** Input is the video plus the candidate timestamps from 2.2. Output is structured JSON (schema below) listing every event: door, turn, stairs, ramp, elevator, sign, toilet, obstacle, narrow passage. Each has a timestamp, visible text (sign OCR) and what was said.
5. **Merge into waypoints**: sensor turns and stops plus Gemini events, sorted by time, merged when less than 1.5 s apart. The sensor wins on timing and angle, Gemini wins on meaning.
6. **Keyframes**, two per waypoint:
   - `image`: the sharpest frame around `t` (ffmpeg `thumbnail` over ±0.5 s)
   - `approach_image`: the frame 2 s before `t`, which is what the next person sees while walking toward the waypoint. This is the frame live recognition compares against.
7. **Gemini pass 2: per waypoint, on the keyframes.**
   - `signature`: a visual fingerprint made only of fixed things (colors, signs, doors, fixtures), never people or movable objects
   - `instruction` DE + EN, a wheelchair variant and a `blind_hint` with more detail (door side, handrail, floor change)
   - `hazards`
8. Write `route.json` + keyframes, plus `entrance` details for onboarding (steps, estimated door width, automatic door, tactile guidance).

Target time: under 60 s for a 1-minute video. The recording page shows progress per stage.

## 3. Review (owner, end of `/record`)
- List of waypoints with keyframe, instruction and turn. The owner can edit the text, delete a waypoint or merge it with the next one.
- Entrance details prefilled by the AI, confirmed with one tap.
- Only confirmed routes go live. The pitch argument: the AI does the work, a human keeps the responsibility.

## 4. Live guidance (`/go/[routeId]`)
Two sources, fused on the phone:

**A. Dead reckoning, no AI, works offline**
- Step counter from `DeviceMotion`, compared with `steps_est` → progress plus "in about 5 steps, turn left"
- **Relative** heading, not absolute: indoors the compass is off by 20 to 30°. At the start the user faces the entrance, which calibrates the offset. After that only turn deltas count.
- Arrow direction = the next waypoint's planned turn minus the heading change so far

**B. Gemini Live, confirms and corrects**
- The server mints a short-lived token and the browser opens the Live session. The API key never reaches the phone.
- Session context: the route as compact text plus all `approach_image`s at 512 px.
- Camera frames at 1 fps (512 px JPEG) plus microphone. The user can ask things ("where is the toilet?").
- Agent tools: `go_to_waypoint(id, confidence)`, `warn(hazard)`, `off_route()`, `arrived()`. The agent speaks the directions itself.

**State machine on the phone (source of truth)**
- The agent only *proposes*. The app accepts +1 waypoint, and +2 only at confidence ≥ 0.8. It never jumps backward.
- When steps run far past `steps_est` without a confirmation, the app asks "Do you see <landmark>?"
- `off_route`, or a heading more than 90° off for 5 s, triggers "Stop, turn around" and the app returns to the last confirmed waypoint.

**Output, same state for every view**
- Camera view: live camera with a perspective arrow on the floor (CSS 3D, `rotateX` about 60°), a step card at the top.
- Giant arrow: full screen, contrast modes black/yellow, black/white and white/black. Shape and brightness carry the meaning, not color.
- Audio: Gemini voice. Fallback is `speechSynthesis` with the instruction text. For blind users an **earcon panned left or right** plays, so the sound comes from the direction to turn.
- **iOS Safari has no Vibration API.** Haptics only exist in the native IOS app. The web uses sound instead.

## Testing without walking
- Film **two takes** of the same route. Take 1 builds the route, take 2 plays the user.
- Replay mode `/go/[routeId]?replay=take2`: the take-2 video stands in for the camera and its trace stands in for the sensors. Check the waypoint changes against the real timestamps (log with time, waypoint, source sensor/agent, confidence).
- The replay mode is also the **stage demo**: nobody can walk through the building on stage, so the take-2 video runs as the camera with live Gemini on top.

## Milestones
| Time | VIDEO | LIVE |
|---|---|---|
| 16:00 | recording page with trace on the iPhone, upload works | giant arrow + step cards from fixtures, tap to advance |
| 18:00 | sensor events + Gemini pass 1 → first real `route.json` | dead reckoning (steps + relative heading) drives the arrow |
| 20:00 | keyframes + pass 2 + review screen | Gemini Live session with tools, state machine |
| 22:00 | demo route cached | replay mode with take 2, camera view with floor arrow |

## Contracts
**`trace.json`**
`{ "started_at": ISO, "hz": 20, "samples": [{ "t": 0.05, "heading": 182.4, "acc": 9.93 }], "steps": [0.8, 1.4, ...] }`

**Gemini pass 1: `events[]`**
`{ "t": 12.4, "type": "door|turn|stairs|ramp|elevator|sign|toilet|obstacle|narrow", "text_seen": "WC", "said": "toilet on the left", "description": "..." }`

**`route.json` additions** to the README contract, per waypoint:
`approach_image`, `turn_delta_deg` (relative, e.g. -90), `signature`, `instruction_wheelchair` `{de,en}`, `source` (`sensor|gemini|both`)

**Live log** (`data/routes/<id>/runs/<ts>.jsonl`)
`{ "t": 31.2, "waypoint": "w4", "by": "agent|sensor|user", "confidence": 0.86 }`
