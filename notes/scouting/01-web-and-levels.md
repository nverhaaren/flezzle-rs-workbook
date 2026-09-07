# Scouting 01 — browser build, tick-driven input, starter levels

*flezzle-rs branch `scout/web` (off milestone 01), 2026-09-07.* What was
built ahead of the workbook, what surprised us, and what each finding
suggests for a future project. Not a lesson — raw material for one.

## What exists now

1. **Tick-driven input + gameplay on the fixed schedule** (`src/input.rs`,
   `GameplaySet` in `src/lib.rs`). 60 Hz tick; `TickInput` is a per-tick
   `Actions` bitset with edges computed in tick space; a frame-rate
   accumulator keeps sub-tick taps. Camera stays per-frame.
2. **Level source + switching** (`src/level.rs`). `LevelSource` resource;
   native `argv[1]`, web `?level=`; changing it respawns the world.
3. **`user://` in-memory asset source** for levels that arrive as bytes
   (browser file picker), pre-seeded with the bundled tilesets.
4. **Starter levels** in `assets/levels/`: the retargeted sample world plus
   three ASCII-authored levels from `tools/ldtk_gen.py`; `index.json`
   manifest drives the web page's picker.
5. **Web shell**: `web/index.html` + `Trunk.toml`; `trunk build
   --cargo-profile wasm-release` → static `dist/`.
6. **Tests**: shared headless harness; every bundled level playable; level
   switching; the upload path; manifest integrity.

Browser verification: done — see "Verification" below.

## Findings worth a lesson (or a paragraph)

### A. `just_pressed` is frame-scoped; ticks aren't frames
Bevy's `ButtonInput::just_pressed` is reset every *rendered frame*. Gameplay
in `FixedUpdate` sees it twice per frame at 30 FPS and misses it entirely on
frames without a tick at 144 FPS. The fix — sample input once per tick into
your own structure — is also the replay seam: a trace is one `Actions` per
tick. **Lesson candidate for project 02 (determinism):** give the learner
the frame-scoped bug live (a jump that only works at some refresh rates),
have them build `TickInput`. The headless harness can *demonstrate* the bug
by changing `TimeUpdateStrategy` to 1/144 s.

### B. Messages survive until a fixed tick has run
We worried `CollisionStart`/`CollisionEnd` readers moved to `FixedUpdate`
would miss messages on tick-less frames. They don't: Bevy delays message
clearing until a fixed tick ran (`ShouldUpdateMessages`). Worth one
paragraph in the primer — it's the kind of guarantee people don't know to
rely on.

### C. NaN under exact ticks
Moving to exact 60 Hz ticks made the enemy land *exactly* on its patrol
point → `normalize()` of a zero vector → NaN velocity → Avian panic. Found
by the jump test during milestone 01 review, fixed with `normalize_or_zero`
+ an arrived check. **This is the determinism project's opening story**:
discrete time doesn't just make bugs reproducible, it manufactures a new
class of them, and NaN is exactly what WASM's float determinism guarantee
excludes.

### D. Headless Bevy has sharp edges (all worked around in `tests/common`)
- `bevy_ecs_tilemap` (non-atlas) demands a render sub-app in `build()`;
  the `atlas` feature sidesteps it and is what WebGL2 wants anyway.
- Plugins insert resources in `finish()`; manual `app.update()` loops must
  call `finish()`/`cleanup()` themselves (Avian's spatial-query
  diagnostics).
- `bevy_render`'s component-removal hook unwraps a resource only
  `SyncWorldPlugin` inserts, and `RenderPlugin` skips that plugin without a
  backend — so **despawning anything with a `Sprite` panics headless**
  unless you add `SyncWorldPlugin` yourself. Two upstream-issue candidates
  (tilemap + sync hook). **Lesson candidate:** "how to run a Bevy game with
  no window, no GPU" is a reusable skill for fuzzing; project 03's harness
  needs exactly this.

### E. LDtk's visuals are baked by the editor
Auto-layer tiles are computed by LDtk's rule engine *on save* and stored in
the file. Generated levels therefore have collision but no visible floor
until re-saved in LDtk. Workaround: `ldtk_gen.py` learns a (3×3
neighbourhood → tile stack) table from the sample's baked tiles and paints
the Collisions layer with it — recognisable, not identical. **Lesson
candidates:** (a) "open the generated level in LDtk, save, diff the JSON" is
a 10-minute exercise that teaches the data model; (b) implementing a real
subset of LDtk's rule engine (pattern, modulo, flips, `breakOnMatch`,
chance) is a self-contained project if we ever want editor-free pretty
levels.

### F. Level files as the sharing unit
One `.ldtk` per level, tileset paths relative (`../atlas/…`), only the two
bundled tilesets — that's the compatibility contract in
`docs/making-levels.md`. Uploaded levels resolve tilesets against the
pre-seeded `user://atlas/`. The moment we allow custom tilesets (F9), a
"level" becomes a bundle (zip? ldtk + pngs?) — decide then, not now.

### G. Tick vs display rate in the browser
The web build renders at display refresh, simulates at 60 Hz. Fine for play;
for replay equivalence native↔web (F4) the questions are: does the number of
ticks per wall-second matter (no, if traces are per-tick), and are float
results identical across the two targets (unverified — Avian's
`enhanced-determinism` feature exists for this). Project 02's experiment:
record a trace headless, replay in the browser, compare final state hashes.

### H. Placeholder arguments bite when you change the selection mode
Switching the default from `LevelSelection::Uid(0)` to `index(0)` (so any
single-level file plays) silently broke camera fitting and level following:
upstream's camera code calls `is_match(&LevelIndices::default(), level)` with
*placeholder* indices, which is fine for uid/iid/identifier selections and
matches **every** level for an index selection. The browser screenshot
caught it (player out of frame); no native test did. Fix: once the project
asset loads, pin the selection to the spawned level's iid
(`pin_level_selection_to_iid`); regression test `camera_frames_the_players_level`.
Two lessons: (a) a screenshot is a test — the headless harness can't see
framing; (b) LDtk is y-down, so a level at LDtk world (0, 0) has Bevy
origin (0, −height) — the test's first version assumed (0, 0).

## Suggested shape for the next workbook projects

Given scouting, the original W3 ("determinism + WASM") splits naturally:

- **W3a — Tick-driven input** (synthetic start: cut `src/input.rs` and the
  FixedUpdate scheduling out of `main`; tests red at odd frame rates).
  Small, sharp, teaches A + B + C.
- **W3b — Headless harness** (synthetic start: strip `tests/common`;
  learner rebuilds it, meeting D). Sets up W4 (replay) where the harness
  becomes a replay runner.
- The web build itself is *setup*, not a lesson: a checklist, not a
  project. Keep it in `docs/web-build.md`.

## Verification

- Native: `cargo test` — 12 tests green (2 unit, 3 smoke, 5 level/upload/
  camera) on `scout/web`.
- Browser: `dist/` served over HTTP and driven by headless Chrome for
  Testing 152 (SwiftShader WebGL) via a puppeteer script kept outside the
  repo (`~/agent-workspaces/flezzle-rs/.tools/web-smoke.mjs`): zero console
  errors, zero failed requests, all three sample levels spawn, screenshot
  shows the main level framed with the player. Not yet verified: input in
  the browser (the smoke script doesn't press keys), the file picker end to
  end in a browser (the `user://` path is covered natively), and any real
  GPU. **Owner: please play it** — that's the remaining verification.
