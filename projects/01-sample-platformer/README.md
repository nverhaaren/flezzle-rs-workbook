# Project 01 — Sample platformer

Get the `bevy_ecs_ldtk` platformer example running as the seed of flezzle-rs,
by assembling the game app and implementing the player controls yourself.

**flezzle-rs branches:** start from `milestone/01-start`; reference
implementation (spoilers) on `milestone/01-complete`.

## Goal

From the start commit, make all three headless smoke tests green and the game
properly playable — move, jump, climb, restart — without modifying
`tests/smoke.rs`.

## Setup

```bash
# System deps (Debian/Ubuntu):
sudo apt-get install pkg-config libasound2-dev libudev-dev libwayland-dev

git submodule update --init projects/01-sample-platformer/flezzle-rs
cd projects/01-sample-platformer/flezzle-rs
# The submodule checks out DETACHED at the pinned start commit — branch from
# right here (no ref name needed):
git switch -c milestone/01-attempt
cargo test    # red: this is your starting line
```

(The pinned toolchain in `rust-toolchain.toml` installs itself on first cargo
use. If the submodule pin can't be found, see CONVENTIONS.md → Submodules.
Expect the *first* `cargo test` to take a while and eat serious disk: ~700
crates compile with dependencies at opt-level 3, ~20 GB of `target/`.
Subsequent runs are seconds.)

## What's provided vs. what you build

Provided (worth reading — this is the study material):

- `src/walls.rs`, `src/colliders.rs` — LDtk IntGrid cells → Avian colliders;
  wall merging.
- `src/ground_detection.rs`, `src/climbing.rs` — sensor-based ground/ladder
  detection.
- `src/game_flow.rs` — level loading, and a subtle trick: physics time is
  **paused** until the level finishes spawning so entities don't fall through
  not-yet-spawned floors.
- `src/enemy.rs`, `src/misc_objects.rs`, `src/inventory.rs`, `src/camera.rs`
  — patrolling enemies, chests/pumpkins, item pickup, camera fitting.
- `assets/` — the sample LDtk level and its (CC-licensed, attributed) art.
- `tests/smoke.rs` — headless smoke tests: the app runs windowless with a
  fixed simulated frame duration, and input is injected as data. This
  pattern is the seed of the replay/fuzzing architecture, worth
  understanding now.

You build (marked `TODO(project-01)`):

1. **`GamePlugin` in `src/lib.rs`** — assemble the whole game: third-party
   plugins, world configuration resources, and this crate's plugins/systems.
   The TODO comment lists what belongs there; the modules above are the
   parts, this is the box top.
2. **`player_movement` in `src/player.rs`** — the player controls
   (horizontal movement, ladder climbing, jumping). The TODO comment
   specifies the behavior; the physics vocabulary you need is Avian's
   `LinearVelocity` and the input vocabulary is `ButtonInput<KeyCode>`.

## Suggested checkpoints

1. Wire the third-party plugins and world resources (steps 1–2 of the
   lib.rs TODO) → `game_plugin_inserts_core_resources` goes green (it
   checks configured *values*, including gravity from the physics plugin).
2. Add this crate's plugins and systems →
   `level_spawns_player_and_walls` goes green. `cargo run` should now show
   the level with a camera fitted to it; the player falls and lands but
   ignores you.
3. Implement `player_movement` → `player_jumps_and_moves_right` goes green.
   `cargo run`: A/D move, Space jumps (when grounded or climbing — try
   mid-air), W/S climb ladders, R restarts, P debug-prints the player's
   LDtk-authored inventory to the terminal, and you can shove the heavy
   chest around. Green tests are a **floor**, not a finish line — they don't
   cover climbing, restart, or left movement, so this manual pass is part of
   the contract.

## Concepts to study along the way

Primers for the three frameworks — what each concept is, where it appears in
flezzle-rs, and the canonical docs — live in [`reference/`](../../reference/):
[Bevy](../../reference/bevy-essentials.md) ·
[bevy_ecs_ldtk](../../reference/bevy-ecs-ldtk-essentials.md) ·
[Avian](../../reference/avian-essentials.md). The bullets below are the
subset this project leans on.

- **Bevy fundamentals:** `App`, `Plugin`, systems, queries, resources.
  [Bevy quick start](https://bevy.org/learn/quick-start/introduction/) —
  the ECS chapter is the essential one.
- **bevy_ecs_ldtk's core loop:** the `.ldtk` file is loaded as an asset;
  `LevelSelection` picks a level; `#[derive(LdtkEntity)]` bundles +
  `register_ldtk_entity` map editor entities to ECS entities. See the
  [bevy_ecs_ldtk book](https://trouv.github.io/bevy_ecs_ldtk/v0.15.0/)
  ("Anatomy of the World" + the tile-based game tutorial).
- **Avian physics:** `RigidBody`, `Collider`, `LinearVelocity`, `Gravity`,
  sensors. [avian2d docs](https://docs.rs/avian2d/0.7).
- **Why the lib/bin split:** `GamePlugin` (gameplay) never touches
  window/render plugins (shell). The windowed binary is one shell; the
  headless tests are another; WASM and a fuzzing harness will be more. This
  boundary is flezzle-rs's architectural bet — it's why the tests can run
  the real game at a fixed timestep with injected input.
- **Forward look (milestone 02):** some patterns you're learning here get
  deliberately rewritten next milestone for determinism — `std` HashSets,
  per-frame `Update` gameplay systems vs. the 64 Hz fixed physics clock
  (the tests pin it to 60 Hz as a stopgap), and unordered system scheduling.
  Learn the shape now; expect the clockwork to change.

Primary reference: the
[upstream platformer example](https://github.com/Trouv/bevy_ecs_ldtk/tree/v0.15.0/examples/platformer)
this milestone is adapted from (per-file attribution in `ATTRIBUTION.md`).
Consulting it is legitimate — the exercise is understanding, not clean-room
reinvention — but try each piece yourself first.

## Definition of done

- [ ] `cargo test` green (tests unmodified)
- [ ] Playable per checkpoint 3
- [ ] Compared against `milestone/01-complete`; divergences noted. That
      branch lives on the AI fork — from the submodule:
      `git remote add fork https://github.com/nverhaaren-ai/flezzle-rs.git`,
      `git fetch fork`, then `git diff fork/milestone/01-complete -- src/`
- [ ] Feedback recorded (what was unclear, too easy, too hard) → shapes
      Project 02
