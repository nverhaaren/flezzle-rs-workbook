# bevy_ecs_ldtk essentials (0.15, LDtk 1.5.3)

**Start here:** the [bevy_ecs_ldtk book](https://trouv.github.io/bevy_ecs_ldtk/)
("Anatomy of the World", "Level Selection", the tile-based game tutorial),
the [API docs](https://docs.rs/bevy_ecs_ldtk/0.15), and the
[platformer example](https://github.com/Trouv/bevy_ecs_ldtk/tree/v0.15.0/examples/platformer)
flezzle-rs was adapted from. For the editor itself: [ldtk.io](https://ldtk.io/)
and its [JSON docs](https://ldtk.io/json/). Use **LDtk 1.5.3** — the crate
tracks a specific JSON schema version.

## LDtk's data model

A **project** (`.ldtk`, JSON) holds **definitions** (`defs`) and **levels**.
Definitions are the vocabulary; levels are instances of it.

| LDtk term | Meaning | In our template |
|---|---|---|
| Layer def | A kind of layer every level has, of type IntGrid, Entities, Tiles, or AutoLayer | `Collisions` (IntGrid), `Entities`, `Bg_textures` + `Wall_shadows` (AutoLayers) |
| IntGrid | Grid of small integers; the *semantic* map | 1 dirt, 2 ladder, 3 stone |
| AutoLayer / auto-rules | Tiles painted automatically from IntGrid patterns; the editor bakes the result into the file | why generated levels look plain until saved in LDtk |
| Entity def | A placeable thing with size, pivot, sprite, and typed **fields** | `Player` (items), `Mob` (patrol points), `Chest`, `Pumpkins`, `Door` |
| Tileset | An image plus grid size; referenced by relative path | `../atlas/SunnyLand…png` (16 px), `../atlas/MV Icons…png` (32 px) |
| World layout | How levels are arranged (`Free` here); neighbours are recorded per level | `example_world.ldtk` has three connected levels |

Coordinates: LDtk's origin is top-left with y down; Bevy's is bottom-left
with y up. The crate converts (`GridCoords`, `ldtk_pixel_coords_to_translation_pivoted`).

## How the crate maps it into ECS

1. `LdtkPlugin` registers the `.ldtk` asset loader and the spawning systems.
2. You spawn one `LdtkWorldBundle { ldtk_handle, .. }` (flezzle-rs:
   `level::spawn_level_on_change`). The world entity gets an
   `LdtkProjectHandle`.
3. `LevelSelection` (a resource: by index, identifier, iid, or uid) says
   which level to spawn; `LdtkSettings::level_spawn_behavior` decides
   whether neighbours load too and where levels sit in world space.
4. For each level: a level entity (`LevelIid`), child layer entities, and
   under them one entity per IntGrid cell and per LDtk entity instance —
   *if* you registered a bundle for it.

The hierarchy is world → level → layer → cell/entity. `walls.rs` walks it
upward via `ChildOf` to find each wall's level.

## Registering bundles (the derive macros)

- `#[derive(LdtkEntity)]` on a `Bundle` + `app.register_ldtk_entity::<B>("Player")`
  spawns `B` for every entity instance with that identifier. Field attributes:
  - `#[sprite("player.png")]` / `#[sprite_sheet]` — a `Sprite` from a
    fixed image or from the entity's tile in the editor.
  - `#[from_entity_instance]` — build the field with
    `From<&EntityInstance>` (flezzle-rs: `ColliderBundle`, `Inventory`).
  - `#[ldtk_entity]` — build with the `LdtkEntity` trait yourself
    (`Patrol` reads the `patrol` points field).
  - `#[worldly]` — the entity survives level transitions (the player).
  - `#[grid_coords]`, `#[with(fn)]`, `#[default]` — see the book.
- `#[derive(LdtkIntCell)]` + `app.register_ldtk_int_cell::<B>(2)` spawns `B`
  for every IntGrid cell with that value; `#[from_int_grid_cell]` builds a
  field from the `IntGridCell` (flezzle-rs: `LadderBundle`, `WallBundle`).

Reading fields at runtime: `EntityInstance::iter_enums_field`,
`iter_points_field`, `get_bool_field`, …

## Runtime API you'll use

| Item | Purpose | flezzle-rs use |
|---|---|---|
| `LevelSelection` | Which level to (re)spawn | `LevelPlugin` sets index 0; `update_level_selection` follows the player between neighbours |
| `LdtkSettings` | Spawn behaviour, clear color, IntGrid rendering | `GamePlugin` |
| `LevelEvent` (message) | `SpawnTriggered`, `Spawned`, `Transformed`, `Despawned` | `start_physics` unpauses physics on `Transformed` |
| `Respawn` (component) | Insert on a level entity to reload it | `restart_level` |
| `LdtkProject` (asset) | The parsed file: `get_raw_level_by_iid`, `as_standalone()` | camera fitting, wall merging, level bounds |
| `GridCoords`, `LevelIid`, `LayerInstance`, `EntityInstance` | Components/data mirrored from the file | walls, camera, inventory |

## Features and pins

- `atlas` (on in flezzle-rs): render tilemaps from a texture atlas rather
  than texture arrays. Needed for WebGL2 and, as it turns out, for running
  headless without a GPU.
- `internal_levels` (default) vs `external_levels`: whether level data lives
  in the `.ldtk` file or in sidecar files. Keep levels internal.
- Version matrix lives in the crate README; Bevy 0.19 ↔ bevy_ecs_ldtk 0.15 ↔
  LDtk 1.5.3.
