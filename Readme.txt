# Piston movement system – 26.3 implementation summary

The best approach is to build a small central PistonRules system that extends vanilla piston resolution instead of replacing the entire piston system.

Use PistonLib for design inspiration and FrozenLib 3.0's existing piston events and utilities where possible. The objective should be to preserve vanilla redstone timing, piston behavior, block updates, and movement ordering unless a feature is explicitly enabled.

## 1. Block movement rules

Create a `MovementPolicy` enum with four options:

| Setting     | Behavior                                                        |
| ----------- | --------------------------------------------------------------- |
| `VANILLA`   | Use the block's original vanilla behavior without modification. |
| `MOVABLE`   | Allow movement in both push and pull directions.                |
| `BREAKABLE` | Destroy the block when a piston moves into it.                  |
| `IMMOVABLE` | Prevent piston movement.                                        |

Use Minecraft's existing `PushReaction` values as the underlying implementation:

- `PUSH_PULL` – movable in either direction.
- `POPPED` – destroyed by piston movement.
- `IMMOVEABLE` – cannot be moved.

`VANILLA` must delegate to the original `BlockStateBase.getPistonPushReaction()` result.

For 26.3, the relevant classes are:

- `BlockBehaviour.BlockStateBase`
- `PushReaction`
- `PistonBaseBlock`
- `PistonStructureResolver`

`BlockBehaviour.Properties.pushReaction()` exists, but changing an existing vanilla block's behavior dynamically requires a resolver/state hook rather than simply changing its construction properties.


## 2. Movable block entities
Use FrozenLib's piston functionality rather than implementing block-entity movement entirely from scratch.
The 26.3 FrozenLib release provides:
- `PistonEvents.DETERMINE_BLOCK_ENTITY_PUSH_RESULT`
- `PistonPushUtil`
- `PistonEvents.ON_MOVING_BLOCK_SET`
Allow supported block entities to move while preserving their complete contents.
The important sequence is:
1. Save the original block entity's data.
2. Remove it from its original position.
3. Move its block state and associated data.
4. Restore the block entity exactly once at the destination.
5. Preserve its inventory, custom data, and other state.
Keep movement progress and block-entity data associated with the moving structure. Minecraft 26.3 uses `ValueInput` and `ValueOutput` for block-entity loading and saving. `PistonMovingBlockEntity` already maintains movement progress and has corresponding save/load methods.

## 3. Scheduled-tick behavior
Create two separate concepts:
```
TickMovePolicy {
    VANILLA,
    TRANSFER,
    DUPLICATE
}

TickBinding {
    BLOCK_BOUND,
    POSITION_BOUND
}
```

Their behavior:

| Setting     | Result                                                              |
| ----------- | ------------------------------------------------------------------- |
| `VANILLA`   | Leave scheduled ticks untouched.                                    |
| `TRANSFER`  | Move eligible scheduled ticks to the destination.                   |
| `DUPLICATE` | Keep the original ticks and create corresponding destination ticks. |

Distinguishing block ticks from coordinate-dependent ticks
This needs special care.
Minecraft 26.3's `ScheduledTick<T>` stores:
- `type`
- `pos`
- `triggerTick`
- `priority`
- `subTickOrder`

It does not contain information explaining what the scheduled callback intends to affect. Its uniqueness mechanism also uses the scheduled type and position.
Therefore, provide an optional registration identifying whether a block's scheduled behavior is:
- `BLOCK_BOUND`: The scheduled action should follow the moved block.
- `POSITION_BOUND`: The scheduled action belongs to its original coordinates.

For block-bound ticks, transfer or duplicate them according to the selected policy. Position-bound ticks remain at their original coordinates.
Preserve the original delay, priority, and ordering when creating new scheduled ticks. Do not reset their timers.

FrozenLib's `BlockTickRegistry` and `BlockTickEvents` can help register custom tick behavior, but the actual movement or duplication of vanilla scheduled-tick entries needs its own handling.


## 4. Attached blocks and multiblock structures
Use FrozenLib's piston events for stickiness and adhesion:
- `PistonEvents.DETERMINE_BLOCK_STICKINESS`
- `PistonEvents.TRY_STICK_BLOCKS_TOGETHER`

Both should be used together to establish the intended sticky behavior. This is similar to PistonLib's separate sticky-type and sticky-group concepts.

Multiblock support
Start with vanilla identifiers and state properties rather than a complicated general-purpose multiblock framework.
| Structure     | Identification                                                     |
| ------------- | ------------------------------------------------------------------ |
| Doors         | Same block ID, `half=upper/lower`, `facing`                        |
| Beds          | Same bed ID, `part=head/foot`, `facing`                            |
| Double chests | Same chest variant, `type=left/right`, `facing`                    |
| Shelves       | Treat as individual blocks unless explicitly configured otherwise. |

When the resolver encounters one part of a recognized pair:
1. Find its corresponding partner.
2. Validate the pair together.
3. Add both positions to the resolved structure.
4. Prevent either half from independently entering the destruction or movement list.
5. Keep the pair together throughout movement and landing.

Use a set of visited `BlockPos` values to prevent repeated resolution and infinite loops.
Count every physical block position against vanilla's push limit, including both halves of a multiblock. Do not count a pair as one block.

Importantly, a pair should never become half movable and half destroyed. If the pair cannot be moved, handle it according to one consistent pair-level failure rule.

The vanilla `PistonStructureResolver` already has separate push and destruction lists, branching-stickiness checks, and a maximum push depth. Extend these mechanisms rather than replacing their ordering.

Attached blocks: independent `carried` setting
Give attached blocks a separate boolean:
```
carried
```
This must not be another movement mode.
When enabled, a block such as a lever, torch, or button joins the movement group of its support block.
This relationship takes priority over the attached block's individual movement policy when it is being carried. Consequently, an otherwise breakable or immovable block can still move with its support when `carried` is enabled.
When it is not being carried such as the torch backside is not on the support, its normal movement policy applies.

## 5. Synchronized fast landing
Add an optional group-level `fastLanding` property.
Enable it for:
- Doors
- Beds
- Tall plants

Do not enable it for:
- Shelves (different individual blocks)
- Chests (different individual blocks)

Both halves of a door or bed must finish together, rather than independently completing their movement.

The fast-landing decision must be synchronized between server and client. This matters because vanilla pistons use block events, with the client reconstructing the movement locally rather than receiving a continuous stream of movement-progress updates.


## 6. Piston-group survival protection
This is particularly important for preserving blocks that require support or a companion.
Create a `PistonMoveGroup` for every successfully resolved piston movement.

It should contain:
```
PistonMoveGroup {
    groupId;
    pistonPos;
    members;
    movingBlockEntities;
    pendingLandings;
    fastLanding;
}
```

Include all relevant positions:
- Every block in the resolver's push list.
- Every block in its destruction list.
- Both halves of recognized multiblocks.
- The moving piston head.
- The piston base where necessary for survival dependencies.

Recommended implementation: temporary moving protection
Give each member a temporary `moving` marker associated with its group.
While the group is active:
- Do not allow a member to disappear or break solely because its expected support is temporarily moving.
- Do not break either half of a protected multiblock.
- Continue normal neighbor updates, scheduled ticks, and other block behavior.
- Do not globally disable survival checks.

When the last member lands:
1. Finalize all block placements.
2. Run normal survival checks against the completed structure.
3. Clear the temporary moving markers.
4. Allow ordinary block updates and removal behavior to resume.

When a fast landing block occurs after the initial push group is made, isolate that into its own group based on companionship and support. A supported block will not fast land its supporting block but a supporting block will fast land a supported block. A door and bed half with a companion will fast land together.

This prevents doors, beds, torches, and other support-dependent blocks from disappearing during intermediate movement.

One important detail: create the group only after the actual resolver succeeds and the piston commits its movement to the scheduler but before any survival checks are made or blocks have begun moving, and especially not during the initial dry-run check. Store enough group information with the moving block entities to reconstruct the group if the world is saved during movement.

An alternative is a prospective world-state emulator: survival checks would see all members of the same group as already placed at their destinations. That is more sophisticated and potentially more invasive, so the temporary group marker is the simpler starting point.

## 7. Recommended 26.3 naming and implementation targets

Use Mojang's official 26.3 mappings, not older Yarn or legacy method names.

| Purpose              | 26.3 name                                          |
| -------------------- | -------------------------------------------------- |
| Block positions      | `net.minecraft.core.BlockPos`                      |
| Block states         | `net.minecraft.world.level.block.state.BlockState` |
| Block state behavior | `BlockBehaviour.BlockStateBase`                    |
| Piston reactions     | `net.minecraft.world.level.material.PushReaction`  |
| Piston resolution    | `PistonStructureResolver`                          |
| Piston operation     | `PistonBaseBlock`                                  |
| Moving block entity  | `PistonMovingBlockEntity`                          |
| Scheduled ticks      | `net.minecraft.world.ticks.ScheduledTick<T>`       |
| Block-entity saving  | `ValueOutput`                                      |
| Block-entity loading | `ValueInput`                                       |

The 26.3 source and API references confirm these names and their current roles.&#x20;

[image](https://www.google.com/s2/favicons?domain=https://mc-packet-reference.netlify.app\&sz=32)

MC 26.x+3



## Overall design principle

Keep the implementation divided into four small systems:

1. MovementRules – vanilla, movable, breakable, immovable, and carried behavior.
2. TickRules – vanilla, transfer, duplicate, and tick binding.
3. StructureResolverExtensions – multiblock pairing, stickiness, and attached blocks.
4. PistonMoveGroups – block-entity preservation, fast landing, and survival protection.

The most important rule is to preserve vanilla's actual piston operation wherever possible. Extend the resolver's decisions, retain its ordering and push limits, and avoid changing normal redstone or scheduled-tick timing.

The natural next step is a small 26.3 hook-by-hook implementation map identifying where each of these four systems should integrate with vanilla and FrozenLib.
