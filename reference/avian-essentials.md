# Avian essentials (avian2d 0.7)

**Start here:** [docs.rs/avian2d/0.7](https://docs.rs/avian2d/0.7) — the
crate-level docs are a real guide, and the module docs for `dynamics`,
`collision`, and `spatial_query` cover most questions. Source and examples:
[github.com/Jondolf/avian](https://github.com/Jondolf/avian). Avian is an
ECS-native physics engine: bodies and colliders are ordinary components, and
the simulation is a set of systems on Bevy's fixed schedule.

## The pieces

| Component / resource | Meaning | In flezzle-rs |
|---|---|---|
| `PhysicsPlugins::default()` | The whole engine; steps in `FixedPostUpdate` | `GamePlugin` |
| `RigidBody::{Dynamic, Kinematic, Static}` | Simulated / moved by you / never moves | player & chest dynamic, mobs kinematic, walls static |
| `Collider` | Shape: `rectangle`, `circle`, `capsule`, … | `ColliderBundle`, merged wall rectangles |
| `LinearVelocity` | Velocity you read and *write* to move things | `player_movement`, `patrol` |
| `Gravity` (resource), `GravityScale` | World gravity; per-body multiplier | `(0, -2000)`; climbing sets scale 0 |
| `LockedAxes` | Freeze rotation/translation axes | `ROTATION_LOCKED` on everything |
| `Friction`, `CoefficientCombine` | Surface friction and how two bodies' values combine | player uses 0 with `Min` so walls don't grab it |
| `ColliderDensity` / `Mass` | Mass from shape area × density | the chest is heavy (15) |
| `Sensor` | Detects overlaps, doesn't push | ground sensor, ladder cells |
| `CollisionEventsEnabled` | Opt a collider into `CollisionStart`/`CollisionEnd` messages | sensors |
| `CollisionStart` / `CollisionEnd` (messages) | Contact began / ended between two colliders | ground and ladder detection |
| `Time<Physics>` | The physics clock; `pause()` / `unpause()` | paused until the level has spawned |

Alternative to collision messages: the `CollidingEntities` component keeps
the current overlap set as state — simpler and tick-friendly; worth
considering when the sensors get revisited.

## How motion works

Each fixed tick Avian integrates velocities, solves contacts, and writes
back `Transform` (via its own `Position`/`Rotation` components; you normally
just read `Transform`). Setting `LinearVelocity` directly, as the player
controller does, is fine for arcade feel; forces/impulses exist too.
Kinematic bodies ignore forces and collisions — the mobs move purely by the
velocity `patrol` assigns.

Character-controller subtleties you will meet: a dynamic body sliding along
a wall of merged rectangles can snag on seams (hence zero friction on the
player); a ground sensor is a thin rectangle under the feet; jumping is
"set vertical velocity once, when grounded".

## Determinism knobs (for milestone 02+)

- Physics already runs on the fixed clock; keep gameplay that feeds it there.
- `enhanced-determinism` feature: routes math through `libm` so results
  match across platforms (bit-for-bit is still only guaranteed per build
  target; verify empirically).
- NaN discipline: the engine asserts on NaN velocities in debug builds and
  silently propagates them in release. Guard every `normalize()` and
  division (we already hit one: the enemy patrol).
- Substeps and solver iterations are configurable (`SubstepCount`,
  `SolverConfig`); changing them changes results — pin and record them.
