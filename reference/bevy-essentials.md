# Bevy essentials (0.19)

**Start here:** the official [Quick Start](https://bevy.org/learn/quick-start/introduction/)
(the ECS chapter is the essential one), then keep
[docs.rs/bevy/0.19](https://docs.rs/bevy/0.19) open. The community
[Bevy Cheat Book](https://bevy-cheatbook.github.io/) is excellent for
concepts but lags behind releases — trust names from docs.rs over it.

## The mental model

Bevy is an **ECS**: the game state is a `World` of **entities** (ids) carrying
**components** (plain data structs), plus global **resources** (one value per
type). Behavior lives in **systems** — ordinary functions whose parameters
declare what they read and write — grouped into **schedules** the `App` runs
in order. There are no game objects with methods; there are tables of data and
functions over them.

| Concept | What it is | In flezzle-rs |
|---|---|---|
| `App` | Builder that collects plugins, systems, resources, then `run()`s | `src/main.rs` (windowed shell), `tests/common` (headless shell) |
| `Plugin` | A function that adds things to an `App`; the unit of modularity | `GamePlugin` in `src/lib.rs` wires everything; each module has its own plugin |
| `Component` | Data attached to an entity | `Player`, `Wall`, `Climber`, `GroundDetection` |
| `Bundle` | A struct of components spawned together | `PlayerBundle`, `ColliderBundle` |
| `Resource` | Global singleton by type | `TickInput`, `LevelSource`, `Gravity`, `LevelSelection` |
| `Query<…>` | Iterate entities matching component filters | nearly every system |
| `Commands` | Deferred world edits (spawn/despawn/insert) applied at the next sync point | `spawn_wall_collision`, `spawn_level_on_change` |
| `Res<T>` / `ResMut<T>` | Read / write a resource from a system | `Res<TickInput>` in `player_movement` |

System parameters are checked for conflicts at startup: two systems that
mutably touch the same data won't run in parallel. Bevy schedules systems in
parallel by default — **unordered systems that both write state are a
determinism hazard**, which is why flezzle-rs orders gameplay explicitly.

## Schedules and the fixed timestep

Each frame the `Main` schedule runs, in order:
`First → PreUpdate → RunFixedMainLoop → Update → PostUpdate → Last`.

`RunFixedMainLoop` runs the fixed schedules
(`FixedFirst → FixedPreUpdate → FixedUpdate → FixedPostUpdate → FixedLast`)
**zero or more times** per frame, catching the fixed clock up to real time.
At 144 FPS a 60 Hz tick runs in fewer than half the frames; at 30 FPS it runs
twice per frame. Inside fixed schedules `Res<Time>` is the fixed clock.

flezzle-rs puts *all gameplay* in `FixedUpdate` (ordered by `GameplaySet`)
and Avian steps physics in `FixedPostUpdate`; only the camera runs in
`Update`. `Time::<Fixed>::from_hz(60.0)` sets the tick. Docs:
[`Time<Fixed>`](https://docs.rs/bevy/0.19/bevy/time/struct.Fixed.html),
[`bevy::app::Main`](https://docs.rs/bevy/0.19/bevy/app/struct.Main.html).

**Ordering:** `.chain()` runs systems in sequence; `SystemSet`s group them and
`configure_sets` orders the groups. `.in_set(GameplaySet::Act)` is how the
flezzle-rs modules slot into the tick.

## Change detection and events

- `Added<T>` / `Changed<T>` query filters see components that changed since
  the system last ran. `spawn_ground_sensor` uses `Added<GroundDetection>`;
  `update_on_ground` uses `Changed<GroundSensor>`.
- **Messages** (`Message`, `MessageWriter`, `MessageReader` — called
  "events" before 0.17) are buffered and double-buffered: a message written
  this frame is readable this frame and next. Bevy delays clearing them until
  a fixed tick has run, so `FixedUpdate` readers don't miss any. Avian's
  `CollisionStart`/`CollisionEnd` and LDtk's `LevelEvent` are messages.
- **Observers** (`On<Add, T>`, `On<Remove, T>`, custom `Event`s) run
  immediately when triggered. Not used in flezzle-rs yet.

## Assets

`AssetServer::load("levels/foo.ldtk")` returns a `Handle<T>` immediately and
loads asynchronously; `Assets<T>` holds loaded values. Paths are relative to
`assets/` by default; `source://path` selects another **asset source** —
flezzle-rs registers a `user://` in-memory source for uploaded levels
(`src/level.rs`, built on `bevy::asset::io::memory`). Loaders are chosen by
file extension; `bevy_ecs_ldtk` registers the `.ldtk` loader.

## Input and time

`ButtonInput<KeyCode>` is per-*frame* keyboard state (`pressed`,
`just_pressed`). Window backends feed it via `KeyboardInput` messages —
the headless tests inject those directly. Because `just_pressed` is frame-
scoped, flezzle-rs samples input once per fixed tick into `TickInput`
(`src/input.rs`) and gameplay reads only that.

`TimeUpdateStrategy::ManualDuration` makes the clock advance a fixed amount
per `app.update()` — how the tests get exactly one tick per update.

## Plugins you'll meet

`DefaultPlugins` = window, render, input, assets, audio, log, time, …;
shells `.set(...)`/`.disable::<…>()` pieces (the headless harness disables
winit, log, audio and gives the renderer no backend). `MinimalPlugins` is the
bare loop. Startup order matters: asset sources must be registered before
`AssetPlugin`; some plugins insert resources in `finish()`, which `app.run()`
calls for you but manual `app.update()` loops must call themselves.
