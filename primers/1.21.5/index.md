# Minecraft 1.21.4 -> 1.21.5 Mod Migration Primer

This is a high level, non-exhaustive overview on how to migrate your mod from 1.21.4 to 1.21.5. This does not look at any specific mod loader, just the changes to the vanilla classes.

This primer is licensed under the [Creative Commons Attribution 4.0 International](http://creativecommons.org/licenses/by/4.0/), so feel free to use it as a reference and leave a link so that other readers can consume the primer.

If there's any incorrect or missing information, please file an issue on this repository or ping @ChampionAsh5357 in the Neoforged Discord server.

## Pack Changes

There are a number of user-facing changes that are part of vanilla which are not discussed below that may be relevant to modders. You can find a list of them on [Misode's version changelog](https://misode.github.io/versions/?id=25w03a&tab=changelog).

## Handling the Removal of Block Entities Properly

Previously, `BlockEntity` would handle all of their removal logic within `BlockBehaviour#onRemove`, including both dropping any stored items and removing the block entity itself. However, depending on how the method is used, it can cause some strange behaviors due to the mutable state of the block entity. For this reason, the logic that makes up the removal process has been split between two methods: `BlockEntity#preRemoveSideEffects` and `BlockBehaviour#affectNeighborsAfterRemoval`.

`BlockEntity#preRemoveSideEffects` is now responsible for removing anything from the block entity before it is removed from the level. By default, if the `BlockEntity` is a `Container` instance, it will drop the contents of the container into the level. Other logic can be handled within here, but it should generally avoid removing the `BlockEntity` itself, unless the position of the block entity tends to change dynamically, like for a piston.

From there, the `LevelChunk` logic will call `removeBlockEntity` before calling `BlockBehaviour#affectNeighborsAfterRemoval`. This should only send the updates to other blocks indicating that this block has been removed from the level. For `BlockEntity` holders, this can be done easily by calling `Containers#updateNeighboursAfterDestroy`. Otherwise may want to call `Level#updateNeighborsAt` themselves, depending on the situation.

- `net.minecraft.world.Containers`
    - `updateNeighboursAfterDestroy` - Updates the neighbor state aftering destroying the block at the specified position.
    - `dropContentsOnDestroy` is removed, handled within `BlockEntity#preRemoveSideEffects` for `Container` instances
- `net.minecraft.world.level.block.entity.BlockEntity#preRemoveSideEffects` - Handles logic on the block entity that should happen before being removed from the level.
- `net.minecraft.world.level.block.state.BlockBehaviour#onRemove`, `$BlockStateBase#onRemove` -> `affectNeighborsAfterRemoval`, should only handle logic to update the surrounding neighbors rather than dropping container data

## Voxel Shape Helpers

`VoxelShape`s have received numerous helpers for more common transformations of its base state. There are the `Block` methods for creating a centered (if desired) box and the `Shapes` methods for rotating a `VoxelShape` to its appropriate axis or direction. There is also a `Shapes#rotateAttachFace` method for rotating some `VoxelShape` that is attached to a face of a different block. The results are either stored in a `Map` of some key to a `VoxelShape`, or when using `Block#getShapeForEachState`, a `Function<BlockState, VoxelShape>`.

Most of the `Block` subclasses that had previous public or protected `VoxelShape`s are now private, renamed to a field typically called `SHAPE` or `SHAPES`. Stored `VoxelShape`s may also be in a `Function` instead of directly storing the map itself.

- `com.mojang.math.OctahedralGroup`
    - `permute` - Returns the axis that the given axis is permuted to within the specified group.
    - `fromAngles` - Creates a group with the provided X and Y rotations.
- `net.minecraft.core.Direction$Axis#choose` now has an overload that takes in three booleans
- `net.minecraft.world.level.block.Block`
    - `boxes` - Creates one more than the specified number of boxes, using the index as part of the function to create the `VoxelShape`s.
    - `cube` - Creates a centered cube of the specified size.
    - `column` - Creates a horizontally centered column of the specified size.
    - `boxZ` - Creates a vertically centered (around the X axis) cube/column of the specified size.
    - `getShapeForEachState` now returns a `Function` which wraps the `ImmutableMap`, there is also a method that only considers the specified properties instead of all possible states.
- `net.minecraft.world.phys.shapes`
    - `DiscreteVoxelShape#rotate` - Rotates a voxel shape according to the permutation of the `OctahedralGroup`.
    - `Shapes`
        - `blockOccudes` -> `blockOccludes`
        - `rotate` - Rotates a given voxel shape according to the permutation of the `OctahedralGroup` around the provided vector, or block center if not specified.
        - `equal` - Checks if two voxel shapes are equivalent.
        - `rotateHorizontalAxis` - Creates a map of axes to `VoxelShape`s of a block being rotated around the y axis.
        - `rotateAllAxis` - Creates a map of axes to `VoxelShape`s of a block being rotated around any axis.
        - `rotateHorizontal` - Creates a map of directions to `VoxelShape`s of a block being rotated around the y axis.
        - `rotateAll` - Creates a map of directions to `VoxelShape`s of a block being rotated around the any axis.
        - `rotateAttachFace` - Creates a map of faces to a map of directions to `VoxelShape`s of a block being rotated around the y axis when attaching to the faces of other blocks.
    - `VoxelShape#move` now has an overload to take in a `Vec3i`

## Weapons, Tools, and Armor: Removing the Redundancies

There have been a lot of updates to weapons, tools, and armor that removes the reliance on the hardcoded base classes of `SwordItem`, `DiggerItem`, and `ArmorItem`, respectively. These have been replaced with their associated data components `WEAPON` for damage, `TOOL` for mining, and `ARMOR` for protection. Additionally, the missing attributes are usually specified by setting the `ATTRIBUTE_MODIFIERS`, `MAX_DAMAGE`, `MAX_STACK_SIZE`, `DAMAGE`, `REPAIRABLE`, and `ENCHANTABLE`. Given that pretty much all of the non-specific logic has moved to a data component, these classes have now been completely removed. Use one of the available item property methods or call `Item$Properties#component` directly to set up each item as a weapon, tool, armor, or some combination of the three.

- `net.minecraft.world.item`
    - `ArmorItem` class is removed
    - `AxeItem` now extends `Item`
    - `DiggerItem` class is removed
    - `HoeItem` now extends `Item`
    - `Item$Properties`
        - `tool` - Sets the item as a tool.
        - `pickaxe` - Sets the item as a pickaxe.
        - `sword` - Sets the item as a sword.
    - `PickaxeItem` class is removed
    - `ShovelItem` now extends `Item`
    - `SwordItem` class is removed
    - `ToolMaterial#applyToolProperties` now takes in a boolean of whether the weapon can disable a blocker (e.g., shield)
- `net.minecraft.world.item.component`
    - `Tool` now takes in a boolean representing if the tool can destroy blocks in creative
    - `Weapon` - A data component that holds how much damage the item can do and whether it disables blockers (e.g., shield).
- `net.minecraft.world.item.equipment`
    - `ArmorMaterial`
        - `humanoidProperties` -> `Item$Properties#humanoidArmor`
        - `createAttributes` is now public
    - `Equippable`
        - `equipOnInteract` - When true, the item can be equipped to another entity when interacting with them.
        - `saddle` - Creates an equippable for a saddle.
        - `equipOnTarget` - Equips the item onto the target entity.

### Extrapolating the Saddles: Equipment Changes

A new `EquipmentSlot` has been added for saddles, which brings with it new changes for genercizing slot logic.

First, rendering an equipment slot for an entity can now be handled as an additional `RenderLayer` called `SimpleEquipmentLayer`. This takes in the entity renderer, the `EquipmentLayerRenderer`, the layer type to render, a function to get the `ItemStack` from the entity state, and the adult and baby models. The renderer will attempt to look up the client info from the associated equippable data component and use that to render the laters as necessary.

Next, instead of having individual lists for each equipment slot on the entity, there is now a general `EntityEquipment` object that holds a delegate to a map of slots to `ItemStack`s. This simplifies the storage logic greatly.

Finally, equippables can now specify whether wearing one will contribute to the bonus XP earned when killing a mob by setting `equipOnInteract`.

- `net.minecraft.client.model`
    - `CamelModel`
        - `head` is now public
        - `createBodyMesh` - Creates the mesh definition for a camel.
    - `CamelSaddleModel` - A model for a camel with a saddle.
    - `DonkeyModel#createSaddleLayer` - Creates the layer definition for a donkey with a saddle.
    - `EquineSaddleModel` - A model for an equine animal with a saddle.
    - `PigModel#createBasePigModel` - Creates the default pig model.
    - `PolarBearModel#createBodyLayer` now takes in a boolean for if the entity is a baby
- `net.minecraft.client.renderer.entity.layers.HorseArmorLayer`, `SaddleLayer` -> `SimpleEquipmentLayer`
- `net.minecraft.client.renderer.entity.state`
    - `CamelRenderState#isSaddled` -> `saddle`, not one-to-one
    - `EquineRenderState#isSaddled` -> `saddle`, not one-to-one
    - `PigRenderState#isSaddled` -> `saddle`, not one-to-one
    - `SaddleableRenderState` class is removed
    - `StriderRenderState#isSaddled` -> `saddle`, not one-to-one
    - `CamelRenderState#isSaddled` -> `saddle`, not one-to-one
- `net.minecraft.client.resources.model.EquipmentClientInfo$LayerType` now has:
    - `PIG_SADDLE`
    - `STRIDER_SADDLE`
    - `CAMEL_SADDLE`
    - `HORSE_SADDLE`
    - `DONKEY_SADDLE`
    - `MULE_SADDLE`
    - `ZOMBIE_HORSE_SADDLE`
    - `SKELETON_HORSE_SADDLE`
- `net.minecraft.world.entity`
    - `EntityEquipment` - A map of slots to item stacks representing the equipment of the entity.
    - `EquipmentSlot`
        - `SADDLE`, `$Type#SADDLE`
        - `canIncreaseExperience` - Whether the slot can increase the amount of experience earned when killing a mob.
    - `EquipmentSlotGroup` is now an iterable
        - `SADDLE`
        - `slots` - Returns the slots within the group.
    - `LivingEntity`
        - `getEquipSound` - Gets the sound to play when equipping an item into a slot.
        - `getArmorSlots`, `getHandSlots`, `getArmorAndBodyArmorSlots`, `getAllSlots` are removed
    - `Mob`
        - `isSaddled` - Checks if an item is in the saddle slot.
        - `createEquipmentSlotContainer` - Creates a single item container for the equipment slot.
    - `Saddleable` interface is removed
- `net.minecraft.world.entity.animal.horse.AbstractHorse`
    - `syncSaddletoClients` is removed
    - `getBodyArmorAccess` is removed
- `net.minecraft.world.item.SaddleItem` class is removed

## Weighted List Rework

The weighted random lists have been redesigned into a basic class that hold weighted entries, and a helper class that can obtain weights from the objects themselves.

First there is `WeightedList`. It is effectively the replacement for `SimpleWeightedRandomList`, working the exact same way by storing `Weighted` (replacement for `WeightedEntry`) entries in the list itself. Internally, the list is either stored as a flat array of object entries, or the compact weighted list if the total weight is greater than 64. Then, to get a random element, either `getRandom` or `getRandomOrThrow` can be called to obtain an entry. Both of these methods will either return some form of an empty object or exception if there are no elements in the list.

Then there are the static helpers within `WeightedRandom`. These take in raw lists and some `ToIntFunction` that gets the weight from the list's object. Some methods also take in an integer either representing the largest index to choose from or the entry associated with the weighted index.

- `net.minecraft.client.resources.model.WeightedBakedModel` now takes in a `WeightedList` instead of a `SimpleWeightedRandomList`
- `net.minecraft.util.random`
    - `SimpleWeightedRandomList`, `WeightedRandomList` -> `WeightedList`, now final and not one-to-one
        - `contains` - Checks if the list contains this element.
    - `Weight` class is removed
    - `WeightedEntry` -> `Weighted`
    - All `WeightedRandom` static methods now take in a `ToIntFunction` to get the weight of some entry within the provided list
- `net.minecraft.util.valueproviders.WeightedListInt` now takes in a `WeightedList`
- `net.minecraft.world.level.SpawnData#LIST_CODEC` is now a `WeightedList` of `SpawnData`
- `net.minecraft.world.level.biome`
    - `Biome#getBackgroundMusic` is now a `WeightedList` of `Music`
    - `BiomeSpecialEffects#getBackgroundMusic`, `$Builder#backgroundMusic` is now a `WeightedList` of `Music`
    - `MobSpawnSettings#EMPTY_MOB_LIST`, `getMobs` is now a `WeightedList`
- `net.minecraft.world.level.block.entity.trialspawner.TrialSpawnerConfig#spawnPotentialsDefinition`, `lootTablesToEject` now takes in a `WeightedList`
- `net.minecraft.world.level.chunk.ChunkGenerator#getMobsAt` now returns a `WeightedList`
- `net.minecraft.world.level.levelgen.feature.stateproviders.WeightedStateProvider` now works with `WeightedList`s
- `net.minecraft.world.level.levelgen.heightproviders.WeightedListHeight` now works with `WeightedList`s
- `net.minecraft.world.level.levelgen.structure.StructureSpawnOverride` now takes in a `WeightedList`
- `net.minecraft.world.level.levelgen.structure.pools.alias`
    - `PoolAliasBinding#random`, `randomGroup` now takes in a `WeightedList`
    - `Random` now takes in a `WeightedList`
    - `RandomGroup` now takes in a `WeightedList`
- `net.minecraft.world.level.levelgen.structure.structures.NetherFortressStructure#FORTRESS_ENEMIES` is now a `WeightedList`

## Tickets

Tickets have been reimplemented into a half type registry-like, half hardcoded system. The underlying logic of keeping a chunk loaded or simulated for a certain period of time still exists; however, the logic associated with each ticket is hardcoded into their appropriate locations, such as the forced or player loading tickets.

Tickets begin with their registered `TicketType`, which contains information about how many ticks should the ticket should last for (or `0` when permanent), whether the ticket should be saved to disk, and what the ticket is used for. A ticket has two potential uses: one for loading the chunk and keeping it loaded, and one for simulating the chunk based on the expected movement of the ticket creator. Most ticks specify that they are for both loading and simulation.

There are two special types that have additional behavior associated with them. `TicketType#FORCED` has some logic for immediately loading the chunk and keeping it loaded. `TicketType#UNKNOWN` cannot be automatically timed out, meaning they are never removed unless explicitly specified.

```java
// You need to register the ticket type to `BuiltInRegistries#TICKET_TYPE`
public static final TicketType EXAMPLE = new TicketType(
    // The amount of ticks before the ticket is removed
    // Set to 0 if it should not be removed
    0L,
    // Whether the ticket should be saved to disk
    true,
    // What the ticket will be used for
    TicketType.TicketUse.LOADING_AND_SIMULATION
);
```

Then there is the `Ticket` class, which are actually stored and handled within the `TicketStorage`. The `Ticket` class takes in the type of the ticket and uses it to automatically populate how long until it expires. It also takes in the ticket level, which is a generally a value of 31 (for entity ticking and block ticking), 32 (for block ticking), or 33 (only can access static or modify, not naturally update) minus the radius of chunks that can be loaded. A ticket is then added to the process by calling `TicketStorage#addTicketWithRadius` or its delegate `ServerChunkCache#addTicketWithRadius`. There is also `addTicket` if you wish to specify the ticket manually rather than having it computed based on its radius.

- `net.minecraft.server.level`
    - `ChunkMap` now takes in a `TicketStorage`
        - `$TrackedEntity#broadcastIgnorePlayers` - Broadcasts the packet to all player but those within the UUID list.
    - `DistanceManager`
        - `chunksToUpdateFutures` is now protected and takes in a `TicketStorage`
        - `purgeStaleTickets` -> `net.minecraft.world.level.TicketStorage#purgeStaleTickets`
        - `getTicketDebugString` -> `net.minecraft.world.level.TicketStorage#getTicketDebugString`
        - `getChunkLevel` - Returns the current chunk level or the simulated level when the provided boolean is true.
        - `getTickingChunks` is removed
        - `removeTicketsOnClosing` is removed
        - `$ChunkTicketTracker` -> `LoadingChunkTracker`, or `SimulationChunkTracker`
    - `ServerChunkCache`
        - `addRegionTicket` -> `addTicketWithRadius`, or `addTicket`
        - `removeRegionTicket` -> `removeTicketWithRadius`
        - `removeTicketsOnClosing` -> `deactivateTicketsOnClosing`
    - `Ticket` is no longer final or implements `Comparable`
        - `load` - Loads a ticket from a `CompoundTag`.
        - `save` - Saves a ticket to a `CompoundTag`.
        - The constructor no longer takes in a key
        - `setCreatedTick`, `timedOut` -> `resetTicksLeft`, `decreaseTicksLeft`, `isTimedOut`; not one-to-one
    - `TicketType` is now a record and no longer has a generic
        - `getComparator` is removed
        - `doesLoad`, `doesSimulate` - Checks whether the ticket use is for their particular instance.
        - `$TicketUse` - What a ticket can be used for.
    - `TickingTracker` -> `SimulationChunkTracker`
- `net.minecraft.world.level.ForcedChunksSavedData` -> `TicketStorage`
- `net.minecraft.world.level.chunk.ChunkSource`
    - `updateChunkForced` now returns a boolean indicating if the chunk has been forcefully loaded
    - `getForceLoadedChunks` - Returns all chunks that have been forcefully loaded.

## The Game Test Overhaul

Game tests have been completely overhauled into a registry based system, completely revamped from the previous automatic annotation-driven system. However, as of the current version, they are not fully implemented. As such, the full update primer of the system will wait until a later update.

For a quick explanation, the game test system is broken into test functions, test environments, and test instances. Test environments are the analog of the batching system -- the server level is setup before the game tests are run and then teardowned afterwards. Test instances represent an indivdual test to run. These take in a `TestData`, which is the closest analog to the previous `TestFunction`. Finally, Test functions are a subtype for a test instance used to run the test instance. A test function is basically a consumer wrapped around the `GameTestHelper`. In its current state, it is expected that a test function either fails or succeeds.

- `net.minecraft.client.renderer.blockentity`
    - `BeaconRenderer` now has a generic that takes in a subtype of `BlockEntity` and `BeaconBeamOwner`
    - `StructureBlockRenderer` -> `BlockEntityWithBoundingBoxRenderer`, not one-to-one
- `net.minecraft.core.registries.Registries#TEST_FUNCTION`, `TEST_ENVIRONMENT_DEFINITION_TYPE`, `TEST_INSTANCE_TYPE`
- `net.minecraft.gametest.Main` - The entrypoint for the game test server.
- `net.minecraft.gametest.framework`
    - `AfterBatch`, `BeforeBatch` annotations are removed
    - `BlockBasedTestInstance` - A test instance for testing the test block.
    - `BuiltinTestFunctions` - Contains all registered test functions.
    - `FailedTestTracker` - An object for holding all game tests that failed.
    - `FunctionGameTestInstance` - A test instance for running a test function.
    - `GameTest` annotation is removed
    - `GameTestAssertException` now takes in a `Component` and the tick the error occured at
        - `getDescription` - Creates a component for the error message.
    - `GameTestBatch` now takes in an index and environment definition instead of a name and batch setups
    - `GameTestBatchFactory`
        - `fromTestFunction` -> `divideIntoBatches`, not one-to-one
        - `toGameTestInfo` is removed
        - `toGameTestBatch` now takes in an environment definition and an index
        - `$TestDecorator` - Creates a list of test infos from a test instance and level.
    - `GameTestEnvironments` - Contains all environments used for batching game test instances.
    - `GameTestGenerator` annotation is removed
    - `GameTestHelper`
        - `tickBlock` - Ticks the block at the specific position.
        - `assertionException` - Returns a new exception to throw on error.
        - `getBlockEntity` now takes in a `Class` to cast the block entity to
        - `assertBlockTag` - Checks whether the block at the position is within the provided tag.
        - `assertBlock` now takes in a block -> component function for the error message.
        - `assertBlockProperty` now takes in a `Component` instead of a string
        - `assertBlockState` now takes in either nothing, a blockstate -> component function, or a supplied component
        - `assertRedstoneSignal` now takes in a supplied component
        - `assertContainerSingle` - Asserts that a container contains exactly one of the item specified.
        - `assertEntityPosition`, `assertEntityProperty` now takes in a component
        - `fail` now takes in a `Component` for the error message
        - `assertTrue`, `assertValueEqual`, `assertFalse` now takes in a component
    - `GameTestInfo` now takes in a holder-wrapped `GameTestInstance` instead of a `TestFunction`
        - `setStructureBlockPos` -> `setTestBlockPos`
        - `placeStructure` now returns nothing
        - `getTestName` - `id`, not one-to-one
        - `getStructureBlockPos` -> `getTestBlockPos`
        - `getStructureBlockEntity` -> `getTestInstanceBlockEntity`
        - `getStructureName` -> `getStructure`
        - `getTestFunction` -> `getTest`, `getTestHolder`, not one-to-one
        - `getOrCalculateNorthwestCorner`, `setNorthwestCorner` are removed
    - `GameTestInstance` - Defines a test to run.
    - `GameTestInstances` - Contains all registered tests.
    - `GameTestMainUtil` - A utility for running the game test server.
    - `GameTestRegistry` class is removed
    - `GameTestSequence#tickAndContinue`, `tickAndFailIfNotComplete` now take in an integer for the tick instead of a long
    - `GameTestServer#create` now takes in an optional string and boolean instead of the collection of test functions and the starting position
    - `GeneratedTest` - A object holding the test to run for the given environment and the function to apply
    - `ReportGameListener#spawnBeacon` is removed
    - `StructureBlockPosFinder` -> `TestPosFinder`
    - `StructureUtils`
        - `testStructuresDir` is now a path
        - `getStructureBounds`, `getStructureBoundingBox`, `getStructureOrigin`, `addCommandBlockAndButtonToStartTest` are removed
        - `createNewEmptyStructureBlock` -> `createNewEmptyTest`, not one-to-one
        - `getStartCorner`, `prepareTestStructure`, `encaseStructure`, `removeBarriers` are removed
        - `findStructureBlockContainingPos` -> `findTestContainingPos`
        - `findNearestStructureBlock` -> `findNearestTest`
        - `findStructureByTestFunction`, `createStructureBlock` are removed
        - `findStructureBlocks` -> `findTestBlocks`
        - `lookedAtStructureBlockPos` -> `lookedAtTestPos`
    - `TestClassNameArgument` is removed
    - `TestEnvironmentDefinition` - Defines the environment that the test is run on by setting the data on the level appropriately.
    - `TestFinder` no longer contains a generic for the context
        - `$Builder#allTests`, `allTestsInClass`, `locateByName` are removed
        - `$Builder#byArgument` -> `byResourceSelection`
    - `TestFunction` -> `TestData`, not one-to-one
    - `TestFunctionArgument` -> `net.minecraft.commands.arguments.ResourceSelectorArgument`
    - `TestFunctionFinder` -> `TestInstanceFinder`
    - `TestFunctionLoader` - Holds the list of test functions to load and run.
- `net.minecraft.network.protocol.game`
    - `ClientboundTestInstanceBlockState` - A packet sent to the client containing the status of a test along with its size.
    - `ServerboundSetTestBlockPacket` - A packet sent to the server to set the information within the test block to run.
    - `ServerboundTestInstanceBlockActionPacket` - A packet sent to the server to set up the test instance within the test block.
- `net.minecraft.world.entity.player.Player`
    - `openTestBlock` -> Opens a test block.
    - `openTestInstanceBlock` - Opens a test block for a game test instance.
- `net.minecraft.world.level.block`
    - `TestBlock` - A block used for running game tests.
    - `TestInstanceBlock` - A block used for managing a single game test.
- `net.minecraft.world.level.block.entity`
    - `BeaconBeamOwner` - An interface that represents a block entity with a beacon beam.
    - `BeaconBlockEntity` now implements `BeaconBeamOwner`
        - `BeaconBeamSection` -> `BeaconBeamOwner$Section`
    - `BoundingBoxRenderable` - An interface that represents a block entity that can render an arbitrarily-sized bounding box.
    - `StructureBlockEntity` now implements `BoundingBoxRenderable`
    - `TestBlockEntity` - A block entity used for running game tests.
    - `TestInstanceBlockEntity` - A block entity used for managing a single game test.
- `net.minecraft.world.level.block.state.properties.TestBlockMode` - A property for representing the current state of the game tests associated with a test block.

## Data Component Getters

The data component system can now be represented on arbitrary objects through the use of the `DataComponentGetter`. As the name implies, the getter is responsible for getting the component from the associated type key. Both block entities and entities use the `DataComponentGetter` to allow querying the internal data, such as variant information or custom names. They both also have methods for collecting the data components from another holder (via `applyImplicitComponents` or `applyImplicitComponent`). Block entities also contain a method for collection to another holder via `collectImplicitComponents`.

### Entities

Some `EntitySubPredicate`s for entity variants have been transformed into data components stored on the held item due to a recent change on `EntityPredicate` now taking in a `DataComponentPredicate` for matching the slots on an entity.

- `net.minecraft.advancements.critereon`
    - `EntityPredicate` now takes in a `DataComponentPredicate` to match the equipment slots checked
    - `EntitySubPredicate`
        - `AXOLTOL` -> `DataComponents#AXOLOTL_VARIANT`
        - `FOX` -> `DataComponents#FOX_VARIANT`
        - `MOOSHROOM` -> `DataComponents#MOOSHROOM_VARIANT`
        - `RABBIT` -> `DataComponents#RABBIT_VARIANT`
        - `HORSE` -> `DataComponents#HORSE_VARIANT`
        - `LLAMA` -> `DataComponents#LLAMA_VARIANT`
        - `VILLAGER` -> `DataComponents#VILLAGER_VARIANT`
        - `PARROT` -> `DataComponents#PARROT_VARIANT`
        - `SALMON` -> `DataComponents#SALMON_SIZE`
        - `TROPICAL_FISH` -> `DataComponents#TROPICAL_FISH_PATTERN`, `TROPICAL_FISH_BASE_COLOR`, `TROPICAL_FISH_PATTERN_COLOR`
        - `PAINTING` -> `DataComponents#PAINTING_VARIANT`
        - `CAT` -> `DataComponents#CAT_VARIANT`, `CAT_COLLAR`
        - `FROG` -> `DataComponents#FROG_VARIANT`
        - `WOLF` -> `DataComponents#WOLF_VARIANT`, `WOLF_COLLAR`
        - `PIG` -> `DataComponents#PIG_VARIANT`
        - `register` with variant subpredicates have been removed
        - `catVariant`, `frogVariant`, `wolfVariant` are removed
        - `$EntityHolderVariantPredicateType`, `$EntityVariantPredicateType` are removed
    - `SheepPredicate` no longer takes in the `DyeColor`
- `net.minecraft.client.renderer.entity.state.TropicalFishRenderState#variant` -> `pattern`
- `net.minecraft.core.component`
    - `DataComponentGetter` - A getter that obtains data components from some object.
    - `DataComponentHolder`, `DataComponentMap` now extends `DataComponentGetter`
    - `DataComponentPredicate` is now a predicate of a `DataComponentGetter`
        - `expect` - A predicate that expects a certain value for a data component.
        - `test(DataComponentHolder)` is removed
    - `DataComponents`
        - `SHEEP_COLOR` - The dye color of a sheep.
        - `SHULKER_COLOR` - The dye color of a shulker (box).
- `net.minecraft.world.entity`
    - `Entity` now implements `DataComponentGetter`
        - `applyImplicitComponents` - Applies the components from the getter onto the entity. This should be overriden by the modder.
        - `applyComponentsFromItemStack` - Applies the components from the stack onto the entity.
        - `castComponentValue` - Casts the type of the object to the component type.
        - `setComponent` - Sets the component data onto the entity.
        - `applyImplicitComponent` - Applies the component data to the entity. This should be overriden by the modder.
        - `applyImplicitComponentIfPresent` - Applies the component if it is present on the getter.
    - `EntityType#appendCustomNameConfig` -> `appendComponentsConfig`
    - `VariantHolder` interface is removed
        - As such, all `setVariant` methods on relevant entities are private while the associated data can also be obtained from the `DataComponentGetter`
- `net.minecraft.world.entity.animal`
    - `CatVariant#CODEC`
    - `Fox$Variant#STREAM_CODEC`
    - `FrogVariant#CODEC`
    - `MushroomCow$Variant#STREAM_CODEC`
    - `Parrot$Variant#STREAM_CODEC`
    - `Rabbit$Variant#STREAM_CODEC`
    - `Salmon$Variant#STREAM_CODEC`
    - `TropicalFish#getVariant` -> `getPattern`
- `net.minecraft.world.entity.animal.axolotl.Axolotl$Variant#STREAM_CODEC`
- `net.minecraft.world.entity.animal.horse`
    - `Llama$Variant#STREAM_CODEC`
    - `Variant#STREAM_CODEC`
- `net.minecraft.world.entity.decoration.Painting`
    - `VARIANT_MAP_CODEC` is removed
    - `VARIANT_CODEC` is now private
- `net.minecraft.world.entity.npc.VillagerDataHolder#getVariant`, `setVariant` are removed
- `net.minecraft.world.item`
    - `ItemStack#copyFrom` - Copies the component from the getter.
    - `MobBucketItem#VARIANT_FIELD_CODEC` -> `TropicalFish$Pattern#STREAM_CODEC`
- `net.minecraft.world.level.block.entity.BlockEntity#applyImplicitComponents` now takes in a `DataComponentGetter`
    - `$DataComponentInput` -> `DataComponentGetter`

## Minor Migrations

The following is a list of useful or interesting additions, changes, and removals that do not deserve their own section in the primer.

### Entity References

Generally, the point of storing the UUID of another entity was to later grab that entity to perform some logic. However, storing the raw entity could lead to issues if the entity was removed at some point in time. As such, the `EntityReference` was added to handle resolving the entity from its UUID while also making sure it still existed at the time of query.

An `EntityReference` is simply a wrapped `Either` which either holds the entity instance or the UUID. When resolving via `getEntity`, it will attempt to verify that the stored entity, when present, isn't removed. If it is, it grabs the UUID to perform another lookup for the entity itself. If that entity does exist, it will be return, or otherwise null.

Most references to a UUID within an entity have been replaced with an `EntityReference` to facilitate this change.

- `net.minecraft.network.syncher.EntityDataSerializers#OPTIONAL_UUID` -> `OPTIONAL_LIVING_ENTITY_REFERENCE`, not one to one as it can hold the entity reference
- `net.minecraft.server.level.ServerLevel#getEntity(UUID)` -> `Level#getEntity(UUID)`
- `net.minecraft.world.entity`
    - `EntityReference` - A reference to an entity either by its entity instance when present in the world, or a UUID.
    - `LivingEntity#lastHurtByPlayer`, `lastHurtByMob` are now `EntityReference`s
    - `OwnableEntity`
        - `getOwnerUUID` -> `getOwnerReference`, not one-to-one
        - `level` now returns a `Level` instead of an `EntityGetter`
    - `TamableAnimal#setOwnerUUID` -> `setOwner`, or `setOwnerReference`; not one-to-one
- `net.minecraft.world.entity.animal.horse.AbstractHorse#setOwnerUUID` -> `setOwner`, not one-to-one
- `net.minecraft.world.level.Level` now implements `UUIDLookup<Entity>`
- `net.minecraft.world.level.entity`
    - `EntityAccess` now implements `UniquelyIdentifyable`
    - `UniquelyIdentifyable` - An interface that claims the object as a UUID and keeps tracks of whether the object is removed or not.
    - `UUIDLookup` - An interface that looks up a type by its UUID.

### Descoping Player Arguments

Many methods that take in the `Player` has been descoped to take in a `LivingEntity` or `Entity` depending on the usecase. The following methods below are a non-exhaustive list of this.

- `net.minecraft.world.entity.EntityType`
    - `spawn`
    - `createDefaultStackConfig`, `appendDefaultStackConfig`
    - `appendCustomEntityStackConfig`, `updateCustomEntityTag`
- `net.minecraft.world.item`
    - `BucketItem#playEmptySound`
    - `DispensibleContainerItem#checkExtraContent`, `emptyContents`
- `net.minecraft.world.level`
    - `Level`
        - `playSeededSound`
        - `mayInteract`
    - `LevelAccessor`
        - `playSound`
        - `levelEvent`
- `net.minecraft.world.level.block`
    - `BucketPickup#pickupBlock`
    - `LiquidBlockContainer#canPlaceLiquid`
- `net.minecraft.world.level.block.entity.BrushableBlockEntity#brush`

### Component Interaction Events

Click and hover events on a `MutableComponent` have been reworked into `MapCodec` registry-like system. They are both now interfaces that register their codecs to an `$Action` enum. The implementation then creates a codec that references the `$Action` type and stores any necessary information that is needed for the logic to apply. However, there is no direct 'action' logic associated with the component interactions. Instead, they are hardcoded into their use locations. For click events, this is within `Screen#handleComponentClicked`. For hover events, this is in `GuiGraphics#renderComponentHoverEffect`. As such, any additional events added will need to inject into both the enum and one or both of these locations.

- `net.minecraft.network.chat`
    - `ClickEvent` is now an interface
        - `getAction` -> `action`
        - `getValue` is now on the subclasses as necessary for their individual types
    - `HoverEvent` is now an interface
        - `getAction` -> `action`
        - `$EntityTooltipInfo`
            - `CODEC` is now a `MapCodec`
            - `legacyCreate` is removed
        - `$ItemStackInfo` is removed, replaced by `$ShowItem`
        - `$LegacyConverter` interface is removed

### Tag Changes

- `minecraft:worldgen/biome`
    - `spawns_cold_variant_farm_animals`
    - `spawns_warm_variant_farm_animals`
- `minecraft:block`
    - `sword_instantly_mines`
    - `replaceable_by_mushrooms`
- `minecraft:entity_type`
    - `can_equip_saddle`
- `minecraft:item`
    - `book_cloning_target`

### Mob Effects Field Renames

Some mob effects have been renamed to their in-game name, rather than some internal descriptor.

- `MOVEMENT_SPEED` -> `SPEED`
- `MOVEMENT_SLOWDOWN` -> `SLOWNESS`
- `DIG_SPEED` -> `HASTE`
- `DIG_SLOWDOWN` -> `MINING_FATIGUE`
- `DAMAGE_BOOST` -> `STRENGTH`
- `HEAL` -> `INSTANT_HEALTH`
- `HARM` -> `INSTANT_DAMAGE`
- `JUMP` -> `JUMP_BOOST`
- `CONFUSION` -> `NAUSEA`
- `DAMAGE_RESISTANCE` -> `RESISTANCE`

### List of Additions

- `com.mojang.blaze3d.platform`
    - `DisplayData`
        - `withSize` - Creates a new instance with the specified width/height.
        - `withFullscreen` - Creates a new instance with the specified fullscreen flag.
    - `FramerateLimitTracker`
        - `getThrottleReason` - Returns the reason that the framerate of the game was throttled.
        - `isHeavilyThrottled` - Returns whether the current throttle significantly impacts the game speed.
        - `$FramerateThrottleReason` - The reason the framerate is throttled.
- `com.mojang.blaze3d.resource.ResourceDescriptor`
    - `prepare` - Prepares the resource for use after allocation.
    - `canUsePhysicalResource` - Typically returns whether a descriptor is already allocated with the same information.
- `net.minecraft.util.Util`
    - `mapValues` - Updates the values of a map with the given function.
    - `mapValuesLazy` - Updates the values of a map with the given function, but each value is resolved when first accessed.
- `net.minecraft.client.Options#startedCleanly` - Sets whether the game started cleanly on last startup.
- `net.minecraft.client.data.models.BlockModelGenerators#createSegmentedBlock` - Generates a multipart blockstate definition with horizontal rotation that displays up to four models based on some integer property.
- `net.minecraft.client.gui.components.toasts.Toast#getSoundEvent` - Returns the sound to play when the toast is displayed.
- `net.minecraft.client.model.AdultAndBabyModelPair` - Holds two `Model` instances that represents the adult and baby of some entity.
- `net.minecraft.client.model.geom.builders`
    - `MeshDefinition#apply` - Applies the given transformer to the mesh before returning a new instance.
    - `MeshTransformer#IDENTITY`- Performs the identity transformation.
- `net.minecraft.client.particle.FallingLeavesParticle$TintedLeavesProvider` - A provider for a `FallingLeavesParticle` that uses the color specified by the block above the particle the spawn location.
- `net.minecraft.client.player.ClientInput#scaleMoveDirection` - Scales the move vector by the provided float.
- `net.minecraft.client.renderer.PostChainConfig$Pass#referencedTargets` - Returns the targets referenced in the pass to apply.
- `net.minecraft.client.renderer.entity.state.PigRenderState#variant` - The variant of the pig.
- `net.minecraft.client.renderer.item.properties.select.ComponentContents` - A switch case property that operates on the contents within a data component.
- `net.minecraft.core`
    - `HolderGetter$Provider#getOrThrow` - Gets a holder reference from a resource key.
    - `SectionPos#sectionToChunk` - Converts a compressed section position to a compressed chunk position.
    - `Vec3i#STREAM_CODEC`
- `net.minecraft.nbt`
    - `CompoundTag#getFloatOrDefault` - Gets the float with the associated key, or the default if not present or an exception is thrown.
    - `NbtUtils#writeVec3i` - Writes a vector to an integer array tag.
- `net.minecraft.server.commands.InCommandFunction` - A command function that takes in some input and returns a result.
- `net.minecraft.server.level.ServerLevel#areEntitiesActuallyTicking` - Returns whether the entity manager is actually ticking entities.
- `net.minecraft.util`
    - `ExtraCodecs`
        - `UNTRUSTED_URI` - A codec for a URI that is not trusted by the game.
        - `CHAT_STRING` - A codec for a string in a chat message.
    - `FileSystemUtil` - A utility for interacting with the file system.
    - `GsonHelper#encodesLongerThan` - Returns whether the provided element can be written in the specified number of characters.
    - `Unit#STREAM_CODEC` - A stream codec for a unit instance.
- `net.minecraft.world.effect.MobEffectInstance#withScaledDuration` - Constructs a new instance with the duration scaled by some float value.
- `net.minecraft.world.entity`
    - `AreaEffectCloud#setPotionDurationScale` - Sets the scale of how long the potion should apply for.
    - `DropChances` - A map of slots to probabilities indicating how likely it is for an entity to drop that piece of equipment.
    - `Entity`
        - `isInterpolating` - Returns whether the entity is interpolating between two steps.
        - `sendBubbleColumnParticles` - Spawns bubble column particles from the server.
        - `canSimulateMovement` - Whether the entity's movement can be simulated, usually from being the player.
    - `InterpolationHandler` - A class meant to easily handle the interpolation of the position and rotation of the given entity as necessary.
    - `LivingEntity`
        - `getLuck` - Returns the luck of the entity for random events.
        - `getLastHurtByPlayer`, `setLastHurtByPlayer` - Handles the last player to hurt this entity.
        - `getEffectBlendFactor` - Gets the blend factor of an applied mob effect.
        - `applyInput` - Applies the entity's input as its AI, typically for local players.
- `net.minecraft.world.entity.animal`
    - `PigVariant` - A class which defines the common-sideable rendering information and biome spawns of a given pig.
    - `TemperatureVariant` - An enum which indicates an entity within a different temperature.
- `net.minecraft.world.entity.player.Player#preventsBlockDrops` - Whether the player cannot drop any blocks on destruction.
- `net.minecraft.world.item`
    - `Item#STREAM_CODEC`
    - `ItemStack`
        - `MAP_CODEC`
        - `canDestroyBlock` - Returns whether this item can destroy the provided block state.
- `net.minecraft.world.item.alchemy.PotionContents#getPotionDescription` - Returns the description of the mob effect with some amplifier.
- `net.minecraft.world.item.crafting.TransmuteResult` - A recipe result object that represents an item, count, and the applied components.
- `net.minecraft.world.level`
    - `GameRules`
        - `getType` - Gets the game rule type from its key.
        - `keyCodec` - Creates the codec for the key of a game rule type.
    - `Level`
        - `isMoonVisible` - Returns wehther the moon is currently visible in the sky.
        - `getPushableEntities` - Gets all entities except the specified target within the provided bounding box.
- `net.minecraft.world.level.block`
    - `Block`
        - `UPDATE_SKIP_BLOCK_ENTITY_SIDEEFFECTS` - A flag that skips all potential sideeffects when updating a block entity.
        - `UPDATE_SKIP_ALL_SIDEEFFECTS` - A flag that skips all sideeffects by skipping certain block entity logic, supressing drops, and updating the known shape.
    - `SegmentableBlock` - A block that can typically be broken up into segments with unique sizes and placements.
- `net.minecraft.world.level.block.entity.StructureBlockEntity#isStrict`, `setStrict` - Sets strict mode when generating structures.
- `net.minecraft.world.level.entity.PersistentEntitySectionManager#isTicking` - Returns whether the specified chunk is currently ticking.
- `net.minecraft.world.level.levelgen.feature`
    - `AbstractHugeMushroomFeature#placeMushroomBlock` - Places a mushroom block that specified location, replacing a block if it can.
    - `TreeFeature#getLowestTrunkOrRootOfTree` - Retruns the lowest block positions of the tree decorator.
- `net.minecraft.world.level.levelgen.feature.treedecorators.PlaceOnGroundDecorator` - A decorator that places the tree on a valid block position.
- `net.minecraft.world.level.material.Fluid#getAABB`, `FluidState#getAABB` - Returns the bounding box of the fluid.

### List of Changes

- `com.mojang.blaze3d.pipeline.RenderTarget#clear` now has an overload that take in four floats specifying that RGBA values to clear the color with
- `com.mojang.blaze3d.platform.DisplayData` is now a record
- `com.mojang.blaze3d.resource.RenderTargetDescriptor` now takes in an integer representing the color to clear to
- `net.minecraft.util.Util#makeEnumMap` returns the `Map` superinstance rather than the specific `EnumMap`
- `net.minecraft.client.multiplayer.MultiPlayerGameMode#hasInfiniteItems` -> `net.minecraft.world.entity.LivingEntity#hasInfiniteMaterials`
- `net.minecraft.client.player`
    - `ClientInput#leftImpulse`, `forwardImpulse` -> `moveVector`, now protected
    - `LocalPlayer#spinningEffectIntensity`, `oSpinningEffectIntensity` -> `portalEffectIntensity`, `oPortalEffectIntensity`
- `net.minecraft.client.renderer.chunk.SectionRenderDispatcher`
    - `$RenderSection#getOrigin` -> `getRenderOrigin`
    - `$CompileTask#getOrigin` -> `getRenderOrigin`
- `net.minecraft.client.renderer.entity`
    - `DonkeyRenderer` now takes in a `DonekyRenderer$Type` containing the textures, model layers, and equipment information
    - `PigRenderer` now extends `MobRenderer` instead of `AgeableMobRenderer`
    - `UndeadHorseRenderer` now takes in a `UndeadHorseRenderer$Type` containing the textures, model layers, and equipment information
- `net.minecraft.client.renderer.entity.layers.VillagerProfessionLayer#getHatData` now takes in a map of resource keys to metadata sections and swaps the registry and value for a holder instance
- `net.minecraft.commands.ParserUtils#parseJson` -> `parseSnbtWithCodec`, not one-to-one
- `net.minecraft.commands.argument`
    - `ComponentArgument#ERROR_INVALID_JSON` -> `ERROR_INVALID_COMPONENT`
    - `StyleArgument#ERROR_INVALID_JSON` -> `ERROR_INVALID_STYLE`
- `net.minecraft.core.BlockMath#VANILLA_UV_TRANSFORM_LOCAL_TO_GLOBAL`, `VANILLA_UV_TRANSFORM_GLOBAL_TO_LOCAL` is now private
- `net.minecraft.data.loot.BlockLootSubProvider#createPetalDrops` -> `createSegmentedBlockDrops`
- `net.minecraft.network.chat.ComponentSerialization#flatCodec` -> `flatRestrictedCodec`
- `net.minecraft.network.codec.StreamCodec#composite` now has an overload for nine parameters
- `net.minecraft.network.protocol.game`
    - `ClientboundMoveEntityPacket#getyRot`, `getxRot` -> `getYRot`, `getXRot`
    - `ClientboundUpdateAdvancementsPacket` now takes in a boolean representing whether to show the adavncements as a toast
    - `ServerboundSetStructureBlockPacket` now takes in an additional boolean representing whether the structure should be generated in strict mode
- `net.minecraft.server.PlayerAdvancements#flushDirty` now takes in a boolean that represents whether the advancements show display as a toast
- `net.minecraft.server.level`
    - `ServerEntity` now takes in a consumer for broadcasting a packet to all players but those in the ignore list
    - `ServerLevel`
        - `getForcedChunks` -> `getForceLoadedChunks`
        - `isPositionTickingWithEntitiesLoaded` is now public
- `net.minecraft.util.profiling`
    - `ActiveProfiler` now takes in a `BooleanSupplier` instead of a boolean
    - `ContinuousProfiler` now takes in a `BooleanSupplier` instead of a boolean
- `net.minecraft.world.effect`
    - `MobEffect`
        - `getBlendDurationTicks` -> `getBlendInDurationTicks`, `getBlendOutDurationTicks`, `getBlendOutAdvanceTicks`; not one-to-one
        - `setBlendDuration` now has an overload that takes in three integers to set the blend in, blend out, and blend out advance ticks
    - `MobEffectInstance#tick` -> `tickServer`, `tickClient`; not one-to-one
- `net.minecraft.world.entity`
    - `Entity`
        - `cancelLerp` -> `InterpolationHandler#cancel`
        - `lerpTo` -> `moveOrInterpolateTo`
        - `lerpTargetX`, `lerpTargetY`, `lerpTargetZ`, `lerpTargetXRot`, `lerpTargetYRot` -> `getInterpolation`
        - `onAboveBubbleCol` now takes in a `BlockPos` for the bubble column particles spawn location
        - `isControlledByOrIsLocalPlayer` -> `isLocalInstanceAuthoritative`, now final
        - `isControlledByLocalInstance` -> `isLocalClientAuthoritative`, now protected
        - `isControlledByClient` -> `isClientAuthoritative`
        - `fallDistance`, `causeFallDamage` is now a double
        - `absMoveto` -> `absSnapTo`
        - `absRotateTo` -> `asbSnapRotationTo`
        - `moveTo` -> `snapTo`
    - `EntityType$EntityFactory#create` can now return a null instance
    - `ExperienceOrb#value` -> `DATA_VALUE`
    - `ItemBasedSteering` no longer takes in the accessor for having a saddle
    - `LivingEntity`
        - `lastHurtByPlayerTime` -> `lastHurtByPlayerMemoryTime`
        - `lerpSteps`, `lerpX`, `lerpY`, `lerpZ`, `lerpYRot`, `lerpXRot` -> `interpolation`, not one-to-one
        - `isAffectedByFluids` is now public
        - `removeEffectNoUpdate` is now final
        - `tickHeadTurn` now returns nothing
        - `canDisableShield` -> `canDisableBlocking`, now set via the `WEAPON` data component
        - `calculateFallDamage` now takes in a double instead of a float
    - `Mob`
        - `handDropChances`, `armorDropChances`, `bodyArmorDropChance` -> `dropChances`, not one-to-one
        - `getEquipmentDropChance` -> `getDropChances`, not one-to-one
- `net.minecraft.world.entity.ai.Brain#addActivityWithConditions` now has an overload that takes in an integer indiciating the starting priority
- `net.minecraft.world.entity.ai.behavior`
    - `LongJumpToRandomPos$PossibleJump` is now a record
    - `VillagerGoalPackages#get*Package` now takes in a holder-wrapped profession
- `net.minecraft.world.entity.animal`
    - `Pig` is now a `VariantHolder`
    - `WaterAnimal#handleAirSupply` now takes in a `ServerLevel`
- `net.minecraft.world.entity.animal.axolotl.Axolotl#handleAirSupply` now takes in a `ServerLevel`
- `net.minecraft.world.entity.npc`
    - `Villager` now takes in either a key or a holder of the `VillagerType`
    - `VillagerData` is now a record
        - `set*` -> `with*`
    - `VillagerProfession` now takes in a `Component` for the name
    - `VillagerTrades#TRADES` now takes in a resource key as the key of the map
        - This is similar for all other type specific trades
- `net.minecraft.world.entity.player.Player#stopFallFlying` -> `LivingEntity#stopFallFlying`
- `net.minecraft.world.entity.vehicle`
    - `MinecartBehavior` 
        - `cancelLerp` -> `InterpolationHandler#cancel`
        - `lerpTargetX`, `lerpTargetY`, `lerpTargetZ`, `lerpTargetXRot`, `lerpTargetYRot` -> `getInterpolation`
    - `MinecartTNT#primeFuse` now takes in the `DamageSource` cause
- `net.minecraft.world.item`
    - `Item`
        - `canAttackBlock` -> `canDestroyBlock`
        - `hurtEnemy` no longer returns anything
    - `ItemStack#validateStrict` is now public
    - `WrittenBookItem#resolveBookComponents` -> `WrittenBookContent#resolveForItem`
- `net.minecraft.world.item.alchemy.PotionContents#forEachEffect`, `applyToLivingEntity` now takes in a float representing a scalar for the duration
- `net.minecraft.world.item.component.WrittenBookContent` now implements `TooltipProvider`
- `net.minecraft.world.item.crafting`
    - `SmithingTransformRecipe` now takes in a `TransmuteResult` instead of an `ItemStack`
    - `TransmuteRecipe` now takes in a `TransmuteResult` instead of an `Item` holder
- `net.minecraft.world.item.enchantment.EnchantmentInstance` is now a record
- `net.minecraft.world.level`
    - `GameRules$Type` now takes in a value class.
    - `Level`
        - `onBlockStateChange` -> `updatePOIOnBlockStateChange`
        - `isDay` -> `isBrightOutside`
        - `isNight` -> `isDarkOutside`
    - `LevelAccessor#blockUpdated` -> `updateNeighborsAt`
- `net.minecraft.world.level.biome.MobSpawnSettings$SpawnerData` is now a record
- `net.minecraft.world.level.block`
    - `Block#fallOn` now takes a double for the fall damage instead of a float
    - `LeavesBlock` now takes in the chance for a particle and the particle to spawn
    - `ParticleLeavesBlock` -> `LeafLitterBlock`
    - `PinkPetalsBlock` -> `FlowerBedBlock`
    - `Rotation` now has an index used for syncing across the network
- `net.minecraft.world.level.block.entity.BlockEntity#parseCustomNameSafe` now takes in a nullable `Tag` instead of a string
- `net.minecraft.world.level.block.state.StateHolder#getNullableValue` is now private
- `net.minecraft.world.level.chunk.ChunkAccess#setBlockState` now takes in the block flags instead of a boolean, and has an overload to update all set
- `net.minecraft.world.level.levelgen.feature.TreeFeature#isVine` is now public
- `net.minecraft.world.level.saveddata.maps.MapFrame` is now a record
- `net.minecraft.world.level.storage.loot.functions.SetWrittenBookPagesFunction#PAGE_CODEC` -> `WrittenBookContent#PAGES_CODEC`

### List of Removals

- `net.minecraft.network.chat.ComponentSerialization#FLAT_CODEC`
- `net.minecraft.network.protocol.game`
    - `ClientboundAddExperimentOrbPacket`
    - `ClientGamePacketListener#handleAddExperienceOrb`
- `net.minecraft.world.Clearable#tryClear`
- `net.minecraft.world.entity`
    - `Entity`
        - `isInBubbleColumn`
        - `isInWaterRainOrBubble`, `isInWaterOrBubble`
    - `ItemBasedSteering`
        - `addAdditionalSaveData`, `readAdditionalSaveData`
        - `setSaddle`, `hasSadddle`
    - `LivingEntity`
        - `timeOffs`, `rotOffs`
        - `rotA`
        - `oRun`, `run`
        - `animStep`, `animStep0`
        - `appliedScale`
        - `canBeNameTagged`
    - `Mob`
        - `DEFAULT_EQUIPMENT_DROP_CHANCE`
        - `PRESERVE_ITEM_DROP_CHANCE_THRESHOLD`, `PRESERVE_ITEM_DROP_CHANCE`
    - `NeutralMob#setLastHurtByPlayer`
    - `PositionMoveRotation#ofEntityUsingLerpTarget`
- `net.minecraft.world.entity.projectile.AbstractArrow#getBaseDamage`
- `net.minecraft.world.level.Level#updateNeighborsAt(BlockPos, Block)`
