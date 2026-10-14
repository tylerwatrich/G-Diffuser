# G-Diffuser Code Map

How the game and port actually work, recorded as we figure it out. **Read this before digging
through the code.** Add to it whenever a question gets answered, so nobody has to work it out twice.

Conventions: `file:function` references, current as of the commit noted per section. Line
numbers drift, so search by name.

---

## Build facts

- G-Diffuser builds the **Expansion Kit** variant (`EXPANSION_KIT=1`). In `#ifdef EXPANSION_KIT` /
  `#else` splits, the EK branch is the live one. Patching only the non-EK branch does nothing.
- Quick syntax check for decomp files (both variants):
  ```
  gcc -fsyntax-only -std=gnu11 -w -DPORT=1 -DGDIFFUSER_PORT=1 -DNON_MATCHING=1 -DNON_EQUIVALENT=1 \
    -DAVOID_UB=1 -DF3DEX_GBI_2=1 -D_LANGUAGE_C=1 -DVERSION_US=1 -DASSET_VERSION=us -DASSET_REVISION=rev0 \
    -DEXPANSION_KIT=1 -DMFS_VERSION=1 -DBUILD_VERSION=VERSION_J \
    -Idecomp -Idecomp/include -Idecomp/src -Iinclude <file.c>
  ```
- `decomp/` is compiled **without** `port/` include paths. Port functions called from decomp
  code are declared inline with `extern`.
- Port-only changes in decomp code go under `#ifdef PORT`.
- Build: `cmake --build build/macos`. Run with a log: `cd build/macos/port && ./G-Diffuser > gdx-run.log 2>&1`.
  Each launch overwrites the log, and logging to the terminal causes input lag.

## Settings / menu

- CVars are registered in `port/gdx_menu.cpp` and the menu widgets are in `port/gdx_menu_registry.cpp`.
- Read a CVar from decomp code with `CVarGetInteger("gEnhancements.X.Y", default)`.

## Game modes

- `gGameMode` (`GAMEMODE_*`) selects the mode. Sub-options live in `gModeSubOption[MODE_*]`.
- **Speed Lap** is the first entry (index 0) in the Practice board's submenu
  (`overlays/ovl_i6/main_menu.c`; the difficulties move to 1..4). On confirm, the menu sets
  `gGdxSpeedLapSelected`. `CourseSelect_Init` then switches `gSelectedMode` to Time Attack (done there,
  not in the menu, because the menu keeps drawing for a few frames and would flash the Time Attack
  submenu), and `MainMenu_Init` switches back to Practice so the cursor returns to the right board. Check it with `Gdx_SpeedLapActive()` (`common.c`): Time Attack plus the flag.
- Racer count is set in `racer.c` init: Time Attack, Records and Course Edit use 1 racer; GP,
  Practice and Death Race use 30.

## Racers: start grid

- `racer.c:Racer_SetupStartPositions` puts the grid on `courseSegments[0].prev`, stepping back
  from `t=1` by 20×7.3 units. Racers are placed by position (`Racer_GetRacerIdFromPosition`).
  The lateral lane offset comes from `segmentBasis.z`, and the EK version scales lane width
  from the segment radius (capped at 50). In single player, each slot steps back 100 units.
- Walking backward along the spline: `t -= dist / Course_SplineGetTangent(...)`. When `t < 0`,
  move to `->prev` and rescale by the tangent ratio (see the loop at the end of the function).
- Speed Lap moves the lone racer back 29 more slots (the back of a 30-car grid). The rolling start
  (`gEnhancements.Gameplay.SpeedLapFlyingStart`) goes another 21000 units back, capped at 45% of the
  course length, and records the back-of-grid point's lap distance.

- Gaps: a segment whose `trackSegmentInfo & TRACK_SHAPE_MASK == TRACK_SHAPE_AIR` has no road. The
  Speed Lap start backs up past these to 500 units of solid track.

## Race intro / countdown

- `gRaceIntroTimer` starts at 460 (100 in the course editor) and counts down in `Racer_Update`.
  The flyover runs first, then 3-2-1 from 220. **GO is at 40**: every racer gets
  `RACER_STATE_FLAGS_400000`.
- Without `RACER_STATE_FLAGS_400000`, `Racer_UpdateFromControls` zeroes all input, which is what
  holds machines on the grid.
- `RACER_STATE_CPU_CONTROLLED` makes `Racer_UpdateMovement` drive the racer with
  `Cpu_GenerateInputs` (it's also used on the player after they finish). **The CPU driver overwrites
  the racer's handling stats** (`unk_1B0`-`unk_1D0`, turn rates `unk_1E0`/`unk_1E4`, `unk_1E8`,
  `unk_1EC`, grip `unk_1F8`), so handing a machine back to a player means restoring them.
- The Speed Lap rolling start sets both flags on `gRacers[0]` at timer 459 and snapshots the racer.
  It holds the timer at 221 (just before "3") until the remaining distance to the back of the grid
  is at most `speed * (180 + 300)`, so GO lands about 5 seconds before the back of the grid. At GO it clears CPU control and restores the stats.
- Rolling run-in driving (`Racer_UpdateMovement`): the CPU's buttons are replaced
  with A held (no boost, Z/R or attacks), and the player's handling stats are restored every
  frame. Steering is a simple centre-line controller:
  `stickX = 0.4 * (segmentDisplacement · segmentBasis.z) + 300 * (trueBasis.x · segmentBasis.z)`,
  clamped to ±50. The CPU's own lane target is `racer->unk_33C`.
- Speed units: HUD km/h = `racer->speed * 21.6`; `speed` is |velocity| in units per frame. There is no
  top-speed stat: non-boost top speed emerges from engine force (`unk_1B8` at full speed) against drag.
  The rolling start launches at 1000 km/h along `segmentPositionInfo.segmentForward` and lets physics
  settle it to the machine's real top speed.
- Countdown sign ("Mr. Zero" with the 3-2-1 screen): drawn in the race render with matrix
  `unk_21988[player]`, built in `Racer_Update` from the racer's position + 50 units forward + height,
  pulled toward the camera. Rolling start leads it by one frame of velocity.
- CPU lane coordinate: `racer->unk_33C` is the target lateral offset (+ = right). The CPU steers so
  that `unk_33C + (segmentDisplacement · segmentBasis.z) → 0`. The track middle is
  `(currentRadiusRight - currentRadiusLeft) / 2`, and the machine half-width is about 46. Per-segment
  hazards come from `COURSE_CONTEXT()->courseData.{dirt,landmine,pit,jump,ice,dash}[segmentIndex]`,
  cached on the racer as `segmentDirt` / `nextSegmentDirt` etc. by `Cpu_GenerateInputs`.
  `DIRT_LEFT` / `DIRT_RIGHT` / `DIRT_BOTH` / `DIRT_MIDDLE`. The rolling-start autopilot steers to the
  middle and dodges dirt.
- Engine ignition: `racer->unk_17C` is hover height, which also scales booster flame size. It starts at
  the idle value `D_800F80A4` (no flames) and ramps to 13 once the player presses a button after
  timer 250. Rolling start sets it to 13 at release.
- Intro audio: `Audio_StartDemo` (timer 460) plays the intro fanfare and sets `D_80771CA8`, which
  mutes player engine sound until GO. In EK builds the engine sound is armed by `func_800BAE5C`,
  called from the `NA_SE_11` (GO) handler. Rolling start skips the fanfare and arms the engine at release.
- Machine shadows and booster flame **tails** (length = `speed * 0.9`, drawn only at the closest
  booster LOD `unk_2B3 == 2`): while `gRaceIntroTimer >= 280` the render pass skips shadows and zeroes
  `speed` for racers 0-3, so no tails show. Rolling start bypasses this.
- Intro camera: `Camera_UpdateRaceIntro` (`camera.c`) orbits for `camera->timer` 0→300, then switches
  to `CAMERA_MODE_SETUP_RACE`. Setting the timer to 300 skips the fly-around (done for rolling start).
- Lap crossings normally only count once `gRaceIntroTimer < 40`. Speed Lap's player is exempt, so
  a rolling start that reaches the line early still starts the clock.

## Racers: laps and timing

- Init (`Racer_InitRacer`): `lap=0`, `lapsCompleted=1`, `lapsCompletedDistance=-courseLength`.
  Players get `unk_2A4=0`. CPU racers get a random `unk_2A4`, used as a start delay.
- Per frame: `unk_2A4++`, then `raceTime = unk_2A4*50/3` (60 fps ticks → ms-ish units).
- A line crossing is detected when the lap distance jumps by more than half the course length.
  Then `lap++`.
  - The **first** crossing (lap 0→1) records nothing, because the `lap == lapsCompleted+1` check fails.
  - Later crossings record `lapTimes[lap-2] = (sub-frame-corrected time) - completedLapsTime`.
- Practice-style endless laps reset `lap = lapsCompleted = 1` at `gTotalLapCount+1`. Speed Lap
  piggybacks on this branch.
- Speed Lap holds `unk_2A4` at 0 while `lap == 0`, so the clock starts at the line.

## Ghosts / replay recording

- Recording happens in `racer.c`. Position is delta-encoded every 2 frames into
  `sGhostReplayRecordingBuffer` (16200 bytes). The running anchor is
  `sReplayRecordPosX/Y/Z`, and the first sample is relative to the origin captured at GO
  (`gRaceIntroTimer == 255`).
- Playback: `gGhostRacers[]`, where each ghost drives a slot in `gRacers[]`. `func_800950E8` decodes the next delta.
- Speed Lap: at the first line crossing the recording resets and re-anchors to the line
  (`Racer_GdxSpeedLapStartTiming`). The best lap is saved as a one-lap ghost
  (`Racer_GdxSpeedLapLapDone`, `port/gdx_ghost_io.c`) and replayed in ghost slot 0 / `gRacers[1]`.

## Port: N64 address translation (`port/n64_gfx_bridge.cpp`)

- Game data still contains 32-bit N64-style pointers. Truncated low-32 values are matched
  against `gHostRanges` (first match wins, after validation).
- **The main executable's range must stay last.** It's about 36 MB at a random ASLR address, and
  its low-32 window can collide with asset blobs. That's the cause of `[gdl-bad]` errors and
  missing HUD elements or models. `GdxAddHostRange()` inserts new ranges before it.
- Known leftover: an occasional launch still loses exhaust flames or garbles a texture, with no
  error logged. It's likely another low-32 collision. If it happens, save that run's log.

## Port: Metal renderer (`libultraship/src/fast/backends/gfx_metal.cpp`)

- On macOS the game **always** renders offscreen (`ViewportMatchesRendererResolution()` returns
  false on Apple), then composites to the screen.
- Command buffers on one queue run in enqueue order. There's **no** sync across queues, and
  `MTL::Event` waits deadlock if the signaller is queued behind the waiter.
- CPU read-backs (`ReadFramebufferToCPU`) use a separate queue. Before blocking, flush earlier
  main-queue buffers in `mEnqueueOrder` order, or the GPU hangs (black screen, WindowServer lockup).
- Full screen: an opaque layer covering the display gets direct-to-display scanout, which
  faulted the first render pass. The fix is a non-opaque `CAMetalLayer` plus `GdxForceScreenOpaque()`
  (an alpha-only pass before the screen encoder ends).
- `nextDrawable()` is autoreleased, so retain it for the frame. Size fb0 from the drawable's real texture.
- Menu transitions (title-screen curtain, etc.) snapshot the last frame. `GdxUpdateFrameMirror`
  (`n64_gfx_bridge.cpp`) copies each finished frame into a 320x240 mirror FB, and
  `gdx_read_current_framebuffer` reads it back. Metal needed two fixes: `CopyFramebuffer` now
  does a scaled draw when the source and destination sizes differ (a blit can't resize), and
  `ReadFramebufferToCPU` reads an FB not drawn this frame on the readback queue after waiting on
  `mLastFrameCmdBufs`. The old path returned zeros on the first read, which meant black transitions.
- Known cosmetic issue: the snapshot is 4:3 (320x240), so a widescreen frame visibly changes size
  when the transition starts.
