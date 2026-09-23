# VIDEO
Owns: `/record`, `app/api/routes`, `lib/video/`. This is the core of the product.
1. Recording page: camera capture plus compass/gyro trace (DeviceOrientation/DeviceMotion, iOS permission prompt), upload.
2. Server: validate upload (type, size), merge sensor trace, find candidate waypoints at turns.
3. Gemini: label waypoints (landmark, instruction DE + EN, hazards, wheelchair_ok, blind_hint) and entrance details. Key only on the server.
4. Keyframes per waypoint with ffmpeg-static, write `route.json`.
5. Cache the demo route so it works offline.

Detailed design and milestones: `docs/PIPELINE.md`.
