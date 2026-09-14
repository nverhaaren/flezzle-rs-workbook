# Scouting 02 — property-based testing of the web build (Bombadil detour)

*2026-09-13. Question from the owner: can Antithesis's Bombadil test properties
of the final web application, given that the core logic runs in WASM? And more
generally, how do we programmatically check that the playtest issues from
scouting 01 are fixed? Sample properties: visual skipping, R restart, chest
push speed, player/chest draw order.*

## TL;DR

- **Yes, Bombadil can test the web build, and WASM is not the obstacle.**
  The obstacle is observability: a canvas is opaque. The game now publishes a
  per-frame state snapshot (`src/debug.rs`) to the page, Bombadil's
  extractors read it, and properties over it are ordinary TypeScript.
- **Two real interop bugs stood between us and the first useful run**, both
  now documented with workarounds: Bombadil's JS coverage instrumentation
  breaks Trunk's subresource-integrity hashes (game silently never boots →
  `data-integrity="none"`), and Bombadil's `PressKey` sends key *names*
  where winit expects physical *codes* (taps never reach Bevy → synthetic
  `KeyboardEvent`s via custom actions).
- **Of the four playtest issues:** draw order (4) was root-caused by a
  property and fixed, verified in the browser; visual skipping (1) is now a
  number (`max_walk_speed_error`) with a property that will accept the
  interpolation fix; R restart (2) turned out to reload the level but not
  the player (design decision pending, expectation encoded as an ignored
  test); chest slowness (3) is quantified (~1.5 px per push).
- **Random exploration found two level bugs nobody asked about**: a
  wall-less template lets the player walk off the world, and first-steps'
  pit was inescapable. Both fixed; a kill plane is on the roadmap.
- **Reusable artefacts**: `tests/bombadil/` (spec, per-level runner,
  summarizer, README), the snapshot plumbing (useful for Playwright, the
  devtools console, and the future replay checker alike), and a testing
  "stack" picture for the workbook: headless simulation tests → page-level
  scripts → Bombadil exploration.
- **Recommendation**: keep Bombadil as the exploration layer, run it nightly
  on `template` + `first-steps` (T5); put the determinism/replay story in the
  native harness where Antithesis-proper would eventually run it.

## What Bombadil is (as of v0.7.5, released 2026-09-13)

[Bombadil](https://github.com/antithesishq/bombadil) is Antithesis's
open-source property-based testing tool for web UIs (and terminal UIs), the
successor to Quickstrom. A single Rust binary drives a Chromium it launches
itself over DevTools, takes random *actions* on the page, captures *states*,
and checks *properties* written in TypeScript against the sequence of states.
It runs locally, in CI (`antithesishq/bombadil-action`), and inside the
Antithesis platform, where failures replay deterministically.

- Install: prebuilt binary (`bombadil-x86_64-linux` on the releases page) or
  `npm i -D @antithesishq/bombadil` (bundles the binary and the TypeScript
  types). Needs a Chrome/Chromium: it searches `PATH` for the usual names or
  takes `CHROME=/path/to/chrome`.
- Run: `bombadil browser test [--headless] [--time-limit 2m] <origin> spec.ts`.
  Output: a `trace.jsonl` plus a screenshot per captured state;
  `bombadil browser inspect <dir>` opens a local UI; `--reproduce <dir>`
  replays a run.
- Specification = an ES module exporting **properties** and **action
  generators**. `export * from "@antithesishq/bombadil/browser/defaults"`
  gives the stock ones (no uncaught exceptions / promise rejections / console
  errors / HTTP 4xx-5xx; clicks, typing, scrolling, navigation).
- **Extractors** run *inside the browser* on every captured state:
  `extract(state => ...)` with `state.document` and `state.window`, returning
  JSON. Extractors are `Cell`s; properties read `.current`.
- **Properties** are LTL formulas: `always`, `eventually`, `next`, `now`, with
  `.and/.or/.implies/.not` and `eventually(...).within(n, "seconds")`. Thunks
  become formulas automatically.
- **Actions**: built-ins are `Click`, `DoubleClick`, `TypeText`, `PressKey`
  (DOM keyCode; a tap, down+up), `ScrollUp/Down`, `MouseDrag`, `SetViewport`,
  `SetFileInputFiles`. `registerCustomAction(name, async (document, window,
  ...args) => {...})` runs arbitrary JS in the page — the escape hatch for
  anything the built-ins can't express (e.g. *holding* a key).
- Bombadil decides when to capture states (after actions, and on its own
  cadence). It is **not** a per-frame observer.

The published `antithesishq/antithesis-skills` repo (for Claude Code et al.)
covers the Antithesis platform workflow — research → workload → launch →
triage — and contains **no Bombadil-specific skill** at the time of writing.
Its `antithesis-research` skill's property taxonomy (safety / liveness /
reachability, catalogued with evidence) is still a good frame for us.

## Does WASM get in the way?

No, once the game publishes state. Bombadil only sees what JavaScript can
see, and a canvas full of pixels drawn by WASM is opaque. But extractors run
in the page's main JS world with `window` and `document`, so the game now
writes a per-frame snapshot (`src/debug.rs`, `DebugSnapshotPlugin`) both as
`window.flezzle` and as JSON text in a hidden `<script id="flezzle-state">`
element: tick and frame counters, ticks-per-frame, held actions, level
file/selection and spawn/despawn counters, physics-paused flag,
player/chest/mob positions, velocities, draw depths, camera position, and
rolling smoothness metrics (below). Natively the same struct is a resource
the headless tests read. The WASM boundary therefore costs one design
decision — *expose observable state deliberately* — and that decision pays
off for every tool, not just Bombadil (Playwright, a devtools console, a
future replay checker). (Verified: Bombadil read `window.flezzle` fine; the
DOM copy is belt-and-braces for tools that evaluate in isolated worlds.)

## The actual obstacle was not WASM

The first three runs saw no snapshot and no boot at all: blank canvas, zero
console entries. Not a WASM problem, not a WebGL problem (software GL was
already forced via a wrapper `chrome` script): **Bombadil instruments
JavaScript for coverage by default (`--instrument-javascript files,inline`),
rewriting the module files it serves — and Trunk emits subresource-integrity
hashes on its `modulepreload`/`preload` links. The rewritten glue failed the
integrity check and the module silently never executed.** Serving the same
bundle with the `integrity` attributes stripped booted immediately. Fix:
`data-integrity="none"` on Trunk's `<link data-trunk rel="rust">`.
Upstream-issue candidate for Bombadil: strip or recompute SRI when
instrumenting. Lesson: when a tool that rewrites your JS meets a build that
pins hashes of your JS, the failure is silent.

Two consequences worth understanding:

1. **Per-frame properties must be accumulated in-page.** Bombadil samples
   states far less often than the game renders, so "no frame ever moved the
   player more than one tick's worth" cannot be checked from samples of
   position. The game keeps a 120-frame rolling max of per-frame displacement
   and Bombadil checks *that*. General pattern: the system under test
   summarizes its own high-frequency behavior into low-frequency invariants.
2. **Input is a second seam.** `PressKey` is a tap; a platformer needs held
   keys. A custom action dispatches synthetic `keydown`/`keyup`
   `KeyboardEvent`s on the canvas with a delay in between. winit's web backend
   does not check `isTrusted`, so Bevy receives them like real presses.

## The sample properties, as written (tests/bombadil/flezzle.spec.ts)

| Issue | Property | Kind |
|---|---|---|
| boot | `gameBoots`: eventually a level spawned and a player exists, within 30 s | liveness |
| — | `ticksMonotonic`, `simulationAdvances`: tick count never regresses; grows again within 5 s of any state | safety + liveness |
| 4 draw order | `playerDrawnAboveChest`: always `player.z > chest.z` when both exist | safety |
| 1 visual skips | `noDoubleSteps`: while walking on the ground, the 120-frame max player step ≤ 1.5 × one tick step | safety (in-page aggregate) |
| 2 R restart | `restartReloadsLevel`: holding Restart implies the level despawn counter rises within 3 s | liveness |
| 3 chest push | `chestCanBePushed`: standing against the chest and holding Right implies the chest moved > 2 px within 3 s | liveness (sanity floor) |

Action generator: hold D/A for 0.7–1.5 s (custom action), taps of Space, W,
S, R (`PressKey`), and a focus-canvas fallback until the snapshot exists.

## What the native probe already told us about R

`tests/levels.rs::restart_key_reloads_the_level` presses R after walking:
the level **does** despawn and respawn (counters 1 → 2), but the player stays
where it was, because the player is `Worldly` and survives level reloads by
design in the upstream example. So "R does nothing" is really "restart
doesn't reset the player", a design decision rather than a broken key. The
expectation is encoded as an `#[ignore]`d test
(`restart_key_resets_player_to_spawn`) to be un-ignored when restart is
redefined.

## Results

Runs are 90 s per level, headless, software WebGL (SwiftShader) in a GPU-less
sandbox, so absolute frame rates are terrible (~6–12 FPS, up to 15 ticks per
frame). That turns out to be a feature: it exaggerates every timing
assumption.

### Round 1: nothing boots (see "The actual obstacle was not WASM")

### Round 2 (bundle with `DebugSnapshot`, SRI off, player depth fixed)

| Level | States | Violations | What it means |
|---|---|---|---|
| template | 208 | `smoothWalking` ×25, `chestCanBePushed` ×1 | see below |
| first-steps | 101 | `smoothWalking` ×94 | see below |
| example_world | 166 | `smoothWalking` ×154 | same as above; three-level world, no other property fired |

- **Issue 4 (draw order) — fixed and verified in the browser.** Round 1's
  spec found the root cause before the fix: player and chest both at
  z = 9, so Bevy's order was arbitrary (109 violations of
  `playerDrawnAboveChest` in 45 s). With the player pinned to z = 20
  (`keep_player_on_top`, `PostUpdate` before transform propagation) the
  property never fires. Native regression test: `player_is_drawn_above_chest`.
- **Issue 1 (visual skips) — reproduced as a number.** `max_walk_speed_error`
  reached 1.0 (frames where walking input was held but the player moved 0 px)
  and 3.6 (a short frame that ran a large catch-up burst of ticks). Both are
  the same defect the owner saw at 60–144 Hz: rendered motion is quantized to
  ticks, not to frame time. Render interpolation (roadmap T4) is the fix, and
  this property is how we'll know it worked — it should hold at any frame
  rate, including this sandbox's.
- **Issue 3 (chest is slow) — quantified.** On `template`, the walker got up
  against the chest (reachability property satisfied; 2 captured states in
  the push pose) and pushing for a full hold moved it 1.8 px: `chestCanBePushed`
  (> 2 px within 3 s) failed by a hair. Not a bug, a tuning number: the chest
  is 15× the player's density. Whatever we retune it to, the threshold is now
  explicit.
- **Issue 2 (R restart) — see the native probe above.** In the browser the
  built-in `PressKey` taps never reached the game at all (next point), so R
  taps were a no-op for a *different* reason than the owner's; the synthetic
  `tapKey` action replaces them in round 3.
- **Found, not asked for #1: walking off the world.** On the original
  `template` the player's x ranged from −185 to **5259 on a 384-px-wide
  level**: no edge walls, no kill plane, so the walker strolled off the side
  and fell forever. New property `playerStaysInLevel` (level bounds now in
  the snapshot); template gets edge walls; a kill plane / out-of-level rule
  is roadmap T6.
- **Found, not asked for #2: an inescapable pit.** On `first-steps` the
  player's x never exceeded 171.5 in 90 s: the random walk fell into the pit
  and the pit is four tiles deep with a jump that clears three. A kid would
  find that too. The pit now has a ladder. Reachability properties are the
  cheap way to catch "the generator got stuck", and they generalize:
  *every* scenario property should be paired with one.

### Interop finding: Bombadil's `PressKey` vs. winit

`PressKey { code: 82 }` is dispatched over DevTools with `code`/`key` set to
the key *name* (`"R"`, `" "` for space), while browsers and winit's web
backend key off the physical `event.code` (`"KeyR"`, `"Space"`). Result:
Bevy sees `KeyCode::Unidentified` and nothing happens. Text inputs don't
care (they use `key`/`text`), games do. Workaround: custom actions that
dispatch synthetic `KeyboardEvent`s with proper `code` values, which winit
accepts (no `isTrusted` check). Upstream-issue candidate #2 for Bombadil.

### Round 3 (ladder in the pit, walled template, synthetic taps, level bounds)

| Level | States | Violations |
|---|---|---|
| template | 229 | `smoothWalking` ×225, `chestCanBePushed` ×26, `restartReloadsLevel` ×13 |
| first-steps | 209 | `playerStaysInLevel` ×204, `smoothWalking` ×200, `restartReloadsLevel` ×7, `pushScenarioIsReached` ×1 |
| example_world | 214 | `smoothWalking` ×193, `restartReloadsLevel` ×7, `pushScenarioIsReached` ×1 |

- **Synthetic taps work.** `level_spawns` went from 1 to 31 on `template`:
  every synthetic R tap reloaded the level, where round 2's built-in
  `PressKey` taps had done nothing. Bevy received W/S/Space too.
- **The walled template holds.** Player x stayed within [20, 364] on a
  384-px level; `playerStaysInLevel` never fired there.
- **The pit ladder works — and exposed the next hole.** On `first-steps`
  the walker climbed out of the pit (round 2 never got past x = 171) and
  then walked off the level's open right edge to x = 7270 (and off the
  left to −260). `playerStaysInLevel` fired 204 times. The other generated
  levels are now walled too; a kill plane for authored levels remains T6.
- **Chest push quantified, again.** 31 captured push states moved the chest
  from 253 to 301 px — it moves, ~1.5 px per hold. `chestCanBePushed`'s
  "> 2 px within 3 s" failed 26 times: the threshold is for the future
  retune to decide.
- **A property bug of my own.** `restartReloadsLevel` fired 27 times across
  levels although restarts demonstrably happened. Cause: with a 40 ms tap
  the reload completes *before* Bombadil captures the state in which the
  key shows as held, so "despawn count at that state" already includes the
  effect. Fix: baseline from the last state in which Restart was *not*
  held. Lesson worth keeping: temporal properties over sampled states must
  take their baselines from *before* the trigger, not at it.
- `pushScenarioIsReached` is only meaningful where the chest is reachable
  in 60 s of random walking; now gated to the template level via the page
  URL.

### Round 4 (property fixes, all generated levels walled; 60 s each)

| Level | Violations | Notes |
|---|---|---|
| template | `smoothWalking` ×138, `chestCanBePushed` ×15 | 21 restarts, all satisfied `restartReloadsLevel`; push scenario reached 16 times; player within [20, 364] |
| first-steps | `smoothWalking` ×135 | 17 restarts satisfied; player within [20, 268]; reachability correctly gated off |

Only the two *intended* failures remain: `smoothWalking` until render
interpolation lands (roadmap T4) and `chestCanBePushed` until the chest is
retuned (or the threshold is agreed). Everything else — boot, liveness,
draw order, restart mechanism, staying in the level, reachability — holds.
Total wall time for the four rounds, including builds: about two hours,
most of it the two interop trapdoors.

## Tooling notes (for whoever does this next)

- `CHROME=~/agent-workspaces/flezzle-rs/.tools/browsers/chrome-linux64/chrome`
  (Chrome for Testing 152, fetched as a zip and extracted with Python: this
  machine has no `unzip`, and puppeteer's installer needs it).
- Serve `dist/` over HTTP (`python3 -m http.server 8765 --directory dist`;
  the port is positional, `-p` means protocol).
- `--no-sandbox` is required in this sandboxed shell; `--headless` for CI.
- A wrong default-property name fails at spec load with
  `export "x" is of unknown type` — re-export `*` from the defaults instead.

## What this suggests for the workbook

Bombadil earns a place, but not as "the testing tool": as the *browser-level*
layer of a stack we now have three of, each with a clear job:

| Layer | Tool | Sees | Best at |
|---|---|---|---|
| Simulation | `cargo test` headless harness (`tests/common`) | the ECS directly, one tick per update | exact, deterministic checks; the future replay runner |
| Page | `DebugSnapshot` + a script (Playwright/puppeteer) | the published snapshot, the DOM, console | "does the bundle boot and render", screenshots, scripted scenarios |
| Exploration | Bombadil | the same snapshot, randomly driven, over time | invariants nobody wrote a scenario for; reachability; the bugs you didn't ask about |

Candidate projects (synthetic starts, per CONVENTIONS):

1. **"Make the game observable"** — start with `DebugSnapshot` removed; the
   learner decides what to expose and publishes it two ways. Small, and it
   forces the question every later tool depends on: *what is the state?*
   Pairs naturally with the replay/trace project (a trace is the input
   half; the snapshot is the output half).
2. **"Write a property, watch it fail, fix the game"** — hand the learner
   the `playerDrawnAboveChest` story: the property, a run that violates it,
   the z = 9 tie in the trace, the one-system fix. Then ask for a new
   property of their own (the pit trap is a good target: "the player can
   always make progress" is hard to state well — that's the lesson).
3. **"Reachability first"** — the chest push exercise: a scenario property
   that is vacuously true until the generator can reach the scenario; the
   learner writes the reachability property, sees it fail, and fixes either
   the generator (a targeted action) or the level (the ladder).
4. **Later, with T4** — the `smoothWalking` property as the acceptance test
   for render interpolation: implement interpolation until the property
   holds at 30, 60, and 144 Hz (Chrome's `--force-device-scale-factor` and
   frame-rate throttling can simulate the last two).

Things *not* to turn into lessons: the SRI/instrumentation trap and the
`PressKey` code mismatch. Those are tool quirks, documented in
`tests/bombadil/README.md`; they'd teach frustration, not testing.

Open questions for the owner:

- Is a Bombadil run worth a CI minute per level (T5), given it needs a
  Chrome and a built bundle? My take: yes for `template` + `first-steps`
  at 60 s each, nightly rather than per-push.
- Antithesis proper (the platform, with deterministic replay and fault
  injection) is a different bet: it runs Docker workloads, so the natural
  fit is the *native headless* harness plus the future trace fuzzer, not the
  browser. Bombadil-in-Antithesis would replay browser runs, which is nice
  but not where our determinism story lives.
