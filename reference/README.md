# Reference primers

Short, opinionated maps of the three frameworks flezzle-rs is built on — what
each concept is, where it shows up in the flezzle-rs code, and where the real
documentation lives. Read the one you need before a project; come back when a
name in the code is unfamiliar.

- [bevy-essentials.md](bevy-essentials.md) — the engine: ECS vocabulary,
  schedules and the fixed timestep, messages/observers, assets, input.
- [bevy-ecs-ldtk-essentials.md](bevy-ecs-ldtk-essentials.md) — LDtk's data
  model and how `bevy_ecs_ldtk` turns it into entities.
- [avian-essentials.md](avian-essentials.md) — the physics engine: bodies,
  colliders, sensors, the physics clock, determinism knobs.

Versions these were written against: Bevy 0.19, bevy_ecs_ldtk 0.15,
avian2d 0.7, LDtk 1.5.3. Bevy's API renames things most releases; when a
name here doesn't compile, check the
[migration guides](https://bevy.org/learn/migration-guides/) first.
