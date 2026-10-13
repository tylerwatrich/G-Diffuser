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
- **Speed Lap** is Time Attack with `gModeSubOption[MODE_TIME_ATTACK] == 2`. Check it with
  `Gdx_SpeedLapActive()` (`common.c`). The menu entry is in `overlays/ovl_i6/main_menu.c`.
- Racer count is set in `racer.c` init: Time Attack, Records and Course Edit use 1 racer; GP,
  Practice and Death Race use 30.

## Racers: start grid

- `racer.c:Racer_SetupStartPositions` puts the grid on `courseSegments[0].prev`, stepping back
  from `t=1` by 20×7.3 units. Racers are placed by position (`Racer_GetRacerIdFromPosition`).
  The lateral lane offset comes from `segmentBasis.z`, and the EK version scales lane width
  from the segment radius (capped at 50). In single player, each slot steps back 100 units.
- Walking backward along the spline: `t -= dist / Course_SplineGetTangent(...)`. When `t < 0`,
  move to `->prev` and rescale by the tangent ratio (see the loop at the end of the function).
- Speed Lap moves the lone racer back 29 more slots (the back of a 30-car grid).

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
