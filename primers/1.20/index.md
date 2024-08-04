# Minecraft 1.19.4 -> 1.20 Mod Migration Primer

This is a high level, non-exhaustive overview on how to migrate your mod from 1.19.4 to 1.20. This does not look at any specific mod loader, just the changes to the vanilla classes.

This primer is licensed under the [Creative Commons Attribution 4.0 International](http://creativecommons.org/licenses/by/4.0/), so feel free to use it as a reference and leave a link so that other readers can consume the primer.

If there's any incorrect or missing information, please file an issue on this repository or ping @ChampionAsh5357 in the Neoforged Discord server.

## Pack Changes

There are a number of user-facing changes that are part of vanilla which are not discussed below that may be relevant to modders. You can find a list of them on [Misode's version changelog](https://misode.github.io/versions/?id=1.20&tab=changelog).

## Moving Experimental Features

All experimental features which were disabled with the `update_1_20` flag are now moved to their proper locations and implementations. Removed features flags can be seen within `net.minecraft.world.level.storage.WorldData#getRemovedFeatureFlags`.

## LootData

Loot Tables, Predicates, and Item Modifiers have been internally migrated to be handled by one system: `LootData`. `LootData` is resolved using a `LootDataResolver` and identified through its `LootDataType`. In addition, a particular `ResourceLocation` can be associated with a `LootDataType` by wrapping it within a `LootDataId`.

All calls to `#getLootTables`, `#getPredicateManager`, `#getItemModifierManager` have now been replaced with `#getLootData`. From there, you can grab the necesssary element using `#getElement` or `#getElementOptional`.

```java
// Given some MinecraftServer 'server'
server.getLootData().getElement(LootDataType.TABLE, new ResourceLocation(MODID, "example_table"));
```

## Vertex Logic Changes

Vertices within the projection matrix can now be sorted via `com.mojang.blaze3d.vertex.VertexSorting`. By default, sorting can occur either from the distance to the origin (`#DISTANCE_TO_ORIGIN`), by distance from the screen projection (`#ORTHOGRAPHIC_Z`) or by any distance function specified by the user (`#byDistance`). As an interface, this can be extended to encompass any implementation. This can be set via `RenderSystem#setProjectionMatrix`. 

Some methods have been modified to accept a `VertexSorting`:

* `com.mojang.blaze3d.vertex.BufferBuilder#setQuardSortOrigin` -> `#setQuadSorting`

The `VertexBufffer` now takes in a `VertexBuffer$Usage` enum, which determines whether to use `GL_STATIC_DRAW` or `GL_DYNAMIC_DRAW`.

## GuiGraphics

`GuiComponent` has been fully removed from the game, now taking the form of `GuiGraphics`. `GuiGraphics`, as the name implies, holds all drawing related information and helpers for GUIs. Most `PoseStack` parameters within gui related classes have now been replaced with `GuiGraphics`. The `PoseStack` can still be obtained via `GuiGraphics#pose`. In addition, gui related drawing methods outside of `GuiComponent` have now been migrated to `GuiGraphics`, such as `ItemRenderer#renderGuiItem`. GUIs now have a Z-value (which indicates closeness to the camera) bounded at `[-10000, 10000]`.

The internals of `GuiGraphics` now make it such that almost all `RenderSystem` calls are unnecessary. `RenderSystem#depthMask` and `#enable/disableDepthTest` calls are now handled using `GuiGraphics#flush` internally or by specifying a `RenderType` for rendering to the screen. Additionally, draw calls can be isolated using `GuiGraphics#drawManaged`, which flushes the buffer before and after the data has been drawn.

This also created some new shards to use:
* `RenderStateShard$ColorLogicStateShard` with `no_color_logic` and `or_reverse`

There have been some removals as well:
* `#blitOutlineBlack` - The only usecase in `LogoRenderer` was replaced with a texture containing the outline instead.

## Light Engine Rewrite

The light engine has been rewritten to be more efficient and consolidated. While the basic usages of the light engine remain similar, more complicated tasks will require some amount of rewriting.

Most lighting related logic flows that were scattered outside of classes for handling lighting, like the light engine, were removed. In addition, lighting information stored and written outside of the light engine have also been removed.

### LightEngine

Let's start with `LightEngine`. This is essentially a rename of `LayerLightEngine` with more performant propogation logic when adding or removing a source. Most of the methods in this class are 1-to-1, with a few having their parameters changed in some capacity. `LightEngine` is implemented via `SkyLightEngine` and `BlockLightEngine` for sky and block light, respectively. As such, `net.minecraft.world.level.LightLayer` has been reduced to enum constants no longer holding the surrounding light layer value. Any on light increase/decrease methods within these classes have been removed in favor of callbacks within location methods for handling the propogation.

Initialization of the light sources are now done through `#initializeLightSources` methods, like in `ChunkAccess`, during chunk loading and packets from the server.

#### ChunkSkyLightSources

`SkyLightEngine` now handles light sources within a chunk via the `ChunkSkyLightSources`. This essentially is a bit storage which handles how sky light propogates through the chunk downward.

### LightChunk

`LightChunk` is an interface that extends `BlockGetter` which all chunks implement. `LightChunk` contains two methods: `#findBlockLightSources`, which finds all blocks which emits light and passes into a callback which increases the light source at the position asynchronously, and `#getSkyLightSources`, which stores the `ChunkSkyLightSources` for the current chunk. This replaces the methods within `ChunkAccess` (e.g. `#getLights`).

As such, all lighting related calls to a chunk, such as `LightChunkGetter#getChunkForLighting`, returns the `LightChunk`, instead of the `BlockGetter`.

## RuleBlockEntityModifier

Within the `rule` structure processor, block entities can now be streamlined for modification using a `RuleBlockEntityModifier`. These simply define how to modify the tag on a particular block entity. Vanilla provides an implementation to clear the tag (`clear`), merge the tag with another(`append_static`), add a loot tag (`append_loot`), or do nothing (`passthrough`). New modifiers can be added via a `RuleBlockEntityModifierType` using a static registry.

## Signs: Now with More Features

Signs now have a multitude of features. First, the sign text itself is stored within a `net.minecraft.world.level.block.entity.SignText` object. The text can be on both the front and back of the sign. Within `SignBlock`, it can now specify a full rotation using the abstract method `SignBlock#getYRotationDegrees`. Additionally, the sign editor can be opened via `#openTextEdit`. The hitbox center of the sign is determined via `SignBlock#getSignHitboxCenterPosition`. The sign's model and text can be scaled via `SignRenderer#getSignModelRenderScale` and `#getSignTextRenderScale` respectively.

Signs can also be modified by items by attaching the `net.minecraft.world.item.SignApplicator` interface. First, it will check whether the item can be applied to the sign (`#canApplyToSign`) and then attempt to apply the data to the sign and return whether or not is was successful (`#tryApplyToSign`).

Click commands on signs can now be toggled via `SignBlockEntity#canExecuteClickCommands`. This is currently only applied when attempting to chain hanging signs together.

## Goodbye Legacy Smithing

In 1.19.4, the current smithing implementation was deprecated and migrated to a legacy implementation to be replaced with transformers and trims. 1.20 removes the legacy implemenation in its entirety. These include classes like `LegacyUpgradeRecipe` and `LegacySmithingScreen`.

## Redstone Signals via the SignalGetter

All signal information originally tied to the level have been migrated into its own getter called `net.minecraft.world.level.SignalGetter`. If you are calling this directly from the `Level` itself, you will likely not notice any difference in `#getDirectSignalTo`, `#hasSignal`, `#getSignal`, `#hasNeighborSignal`, and `#getBestNeighborSignal`.

There are, of course, a few changes within the `net.minecraft.world.level.block.DiodeBlock` itself:

* `#getAlternateSignal(LevelReader, BlockPos, BlockState)` -> `#getAlternateSignal(SignalGetter, BlockPos, BlockState)`
* `#getAlternateSignalAt` -> `SignalGetter#getControlInputSignal`
* `#isAlternateInput` -> `#sideInputDiodesOnly`

## Resonate Vibrations

As 1.20 added in calibrated sculk sensors, vibrations listeners have been modified to handle different frequencies and allow some form of resonation with other blocks. This is typically checked via `net.minecraft.world.level.block.SculkSensorBlock#tryResonateVibration` which determines the resonance `GameEvent` to emit via `net.minecraft.world.level.gameevent.vibrations.VibrationSystem#getResonanceEventByFrequency`. Resonance for a particular block can be added via the `minecraft:vibration_resonators` block tag. Listeners can be attached to any entity using `VibrationSystem$Listener`.

> Resonance is currently hardcoded to the sound of amethyst blocks.

## StructureProcessor Redefinitions

The two main methods within `StructureProcessor` have been slightly changed to redefine their implementations. First, `#processBlock` now no-ops to the passed in `StructureBlockInfo` instead of being an abstract method. Second, `#finalizeProcessing` now returs the modified list of `StructureBlockInfo`s, no-oping to the passed in list by default. `#finalizeProcessing` also now takes in the `ServerLevelAccessor` instead of just the `LevelAccessor`, similar to most other template logic calls.

## Stripping Materials

`Material`s have been completely removed from Block Properties. There are a number of changes reflecting this.

Creating a block property is done through the static constructor `#of` which takes no arguments:

```java
// Other properties can be chained to the end of this
BlockBehavior.Properties.of();
```

First, `BlockBehaviour#getPistonPushReaction(BlockState)`, `Material#getPushReaction`, `Material$Builder#destroyOnPush`, and `Material$Builder#notPushable` no longer exist, and is now set as a property via `BlockBehaviour$Properties#pushReaction`. In addition, a block is only flammable to lava if the `#ignitedByLava` property is called, replacing `Material#isFlammable` and `Material$Builder#flammable`. A block is also set has a liquid if it has the `#liquid` property, replacing `Material#isLiquid`. A block can be forced to be solid or not regardless of the shape using `BlockBehaviour$Properties#forceSolidOn` and `#forceSolidOff`, respectively. You can also set whether a block can be replaced via `#replaceable`.

You can find a list of replacement properties for every material on [Gizmo's gist](https://gist.github.com/GizmoTheMoonPig/77a90a48e0aeecd15b4c524e1c7f0a4a).

The sound a block makes when put underneath a note block is set using `BlockBehaviour$Properties#instrument`.

`net.minecraft.world.level.material.MaterialColor`s have now been replaced with `MapColor`s. `MapColor`s can be specified on a block property using `#mapColor` (replaces `#color`) by specifying a `DyeColor`, `MapColor`, or a function converting a `BlockState` to a `MapColor`. This includes the following renames:

* `net.minecraft.world.level.block.state.BlockBehavior#defaultMaterialColor` -> `#defaultMapColor`

The following `BlockState` methods have been added:

* `BlockState#blocksMotion`
* `BlockState#isSolid`
* `BlockState#instrument`

## Criteria Changes

All `AbstractCriterionTriggerInstance`s now take in a `ContextAwarePredicate` instead of a `EntityPredicate$Composite` representing teh player. `ContextAwarePredicate` is essentially a list of ANDed `LootItemCondition`s. While this is still representative of the player in most cases, `ItemUsedOnLocationTrigger`, which replaces `ItemInteractWithBlockTrigger` and `PlacedBlockTrigger`, uses the `ContextAwarePredicate` to perform changes based on the state of the block being targetted. `ContextAwarePredicate`s can be created using `#create` or `EntityPredicate#wrap` for easy compatibility with the player.

## Creative Tab Registry

`CreativeModeTab`s are now a new static registry.

## Minor Additions, Changes, and Removals

The following is a list of useful or interesting additions, changes, and removals that do not deserve their own section in the primer.

### Suspicuous Sand to Brushable

All suspicuous sand implementations have now been renamed to 'brushable' (e.g., `SuspiciousSandBlock` -> `BrushableBlock`).

### ProjectileUtil#getHitResult

`net.minecraft.world.entity.projectile.ProjectileUtil#getHitResult` has been expanded into two methods: `#getHitResultOnMoveVector`, which uses the vector from the entity's feet position and `#getHitResultOnViewVector` which uses the vector from the entity's eye position. `#getHitResultOnMoveVector` directly replaces `#getHitResult` in previous versions.

### CommonInputs

`net.minecraft.client.gui.navigation.CommonInputs` has been added to add convenience implementations related to inputs on GUIs. Currently, the only method is `#selected`, which checks whether the space or enter keys have been pressed.

### Fonts

The internals of fonts have changed quite a bit, though most user facing calls remain the same. `GlyphRenderTypes` now hold the font `RenderType`s for normal rendering, see through, or polygon offset. `CodepointMap`s, which are essentially an integer to object maps, are now being used within bitmap and unihex providers. The `FontManager` itself is now reloadble and managed via codecs rather than having a field which handles the logic instead.

All classes with the name `Builder` have either had the `Builder` removed or replaced with `Definition` instead (e.g. `ProviderReferenceBuilder` -> `ProviderReferenceDefinition`).

`GlyphProviderType#LEGACY_UNICODE` (renamed from `GlyphProviderBuilderType`) has now been removed in favor of the `#UNIHEX` implementation. Additionally, a `#REFERENCE` builder type was added which allows a user to reference a different font set definition using an `id` which references the `assets/<namespace>/font` directory.

In addition, `net.minecraft.client.gui.Font#draw`, `#drawShadow`, `#drawWordWrap` have been removed. It is expected that these calls directly pass through `GuiGraphics` which have the ability to specify the parameters that these methods originally hardcoded.

#### GlyphProviderDefinition

`GlyphProviderBuilder` has been reorganized:

* `net.minecraft.client.gui.font.providers.GlyphProviderBuilder#create` -> `GlyphProviderDefinition$Loader#load`
    * This change also throws an `IOException` rather than returning a null entry.
* `GlyphProviderDefinition#unpack` is now used to construct a loader for the font or a record holding a reference to the glyph provider to be loaded.

### OR, AND, and Composite LootItemConditions

The `AlternativeLootItemCondition`, or `alternative`, has been removed and split into two conditions: `any_of` or `all_of`, which represent an OR or AND, respectively. Additionally, composite conditions now have a separate, abstract superclass called `CompositeLootItemCondition`.

### LootParams

The implementation of `LootContext` has been split across itself and `LootParams`. `LootParams` is responsible for what `LootContext` used to do previously: the level, parameters, dynamic drops, and luck. In most instances, you can replace `LootContext` with `LootParams` when constructing the parameter logic. There are additional wrappers within` LootTable` which now take in a `LootParams` instead of the `LootContext` itself, using the stored random sequence. 

`LootContext` now takes in a `LootParams` along with a random sequence responsible for handling the random selection. This will still be passed into loot tables for generating the required data. This also can specify an optional seed in case the `LootParams` does not have one. Some methods within `LootContext` now take in an additional long representing this optional seed (e.g., `#fill`).

As such, following fields and methods have been changed:

* `net.minecraft.world.level.block.state.BlockBehaviour#getDrops(BlockState, LootContext$Builder)` -> `#getDrops(BlockState, LootParams$Builder)`
* `net.minecraft.world.level.block.state.BlockBehaviour$BlockStateBase#getDrops(LootContext$Builder)` -> `#getDrops(LootParams$Builder)`

#### RandomSequence

Each `LootContext` is now given a `XoroshiroRandomSource` based upon a unique `ResourceLocation` used to handle the random selection. This is stored within a `SavedData` and accessed via `ServerLevel#getRandomSequences`, or an individual `RandomSource` via `#getRandomSequence`. Currently, the only random source used is `LootTable#DEFAULT_RANDOM_SEQUENCE` (`minecraft:default`).

### CropBlock Method Accessors

The accessors of three methods in `CropBlock` have changed:

* `#getAgeProperty` is now `protected` instead of `public`
* `#getAge` is now `public` instead of `protected`
* `#isMaxAge` is now `final`

### CombatTracker Rework

A lot of the logic and methods have been reworked or moved in the `CombatTracker`:

* Changes
    * `#recordDamage(DamageSource, float, float)` -> `#recordDamage(DamageSource, float)`
    * `#getFallLocation` -> `FallLocation#getCurrentFallLocation`
* Removals
    * `#prepareForDamage`
    * `#getKiller`
    * `#isTakingDamage`
    * `#isInCombat`
    * `#resetPreparedStatus`
    * `#getMob`
    * `#getLastEntry`
    * `#getKillerId`

### New Tags

* Biome Tag `minecraft:has_structure/trail_ruins`
* Block Tag `minecraft:combination_step_sound_blocks` - For blocks that should play the step sound of the current block and the one below it
* Block Tag `minecraft:vibration_resonators` - Determines whether sounds made on this block which cause vibrations should resonate.
* Block Tag `minecraft:sniffer_egg_hatch_boost`- Determines whether the block beneath the sniffer egg speeds up the hatch time.
* Block Tag `minecraft:trail_ruins_replaceable`
* Block Tag `minecraft:sword_efficient` - Whether it is efficient to break this block with a sword
* Block Tag `minecraft:replaceable_by_trees` - Determines whether the block can be replaced by trees
* Block Tag `minecraft:replaceable` - Determines whether the block is replaceable
    * This is likely the replacement for `minecraft:replaceable_plants`
* Block Tag `minecraft:enchantment_power_provider` - Determines what can provide power to an enchantment table
* Block Tag `minecraft:enchantment_power_transmitter` - Determines what can not obstruct the transmission of power to an enchantment table
* Block and Item Tag `minecraft:stone_buttons`
* Item Tag `minecraft:villager_plantable_seeds`
* Item Tag `minecraft:decorated_pot_shards` -> `minecraft:decorated_pot_sherds`
* Block Tag `minecraft:maintains_farmland` - Determines whether a block maintains the farmland beneath it
* Item Tag `minecraft:decorated_pot_ingredients`

### Additions

* `net.minecraft.core.BlockPos#breadthFirstTraversal` - Traverses block positions from the distance of a sented point and determines whether how many positions meet the associated predicate.
* `net.minecraft.util.ExtraCodecs#FLAT_COMPONENT` - Converts a `Component` to/from a string.
* `net.minecraft.world.entity.Entity#handleStepSounds`, `#getPrimaryStepSoundBlockPos`, and `#playCombinationStepSounds` are used to handle block sounds that should be played at the same time when walking over them.
* `net.minecraft.world.entity.animal.Animal#finalizeSpawnChildFromBreeding` - Handles any additional spawn settings from breeding the entity without spawning the child itself.
* `net.minecraft.world.level.block.LeavesBlock#getOptionalDistanceAt` - Public facing implementation of `#getDistanceAt`
* `net.minecraft.client.gui.screens.Screen#getBackgroundMusic` - Screens can set what background music to play when open
    * Music that needs to be stopped on close should query the music manager (via `MusicManager#stopPlaying`) through `Screen#removed`
* `net.minecraft.world.level.lighting.LeveledPriorityQueue` - A stacked priority queue
* `net.minecraft.client.gui.components.AbstractWidget#getTooltip` - Returns the tooltip associated with a component
* `net.minecraft.world.entity.Entity#getNameTagOffsetY` - The offset of the nametag's y position
* `net.minecraft.world.item.ItemStack#copyAndClear` - Returns a copy of the item stack while clearing the original
* `net.minecraft.world.level.block.DoorBlock#type` - Returns the door's `BlockSetType`
* `net.minecraft.world.level.block.FarmableBlock` - Used to check whether a farmland should be kept for the above block
* `net.minecraft.world.level.block.IceBlock#meltsInto` - Returns what the ice block should melt into
* `net.minecraft.world.level.block.SculkSensorBlock#getActiveTick` - Returns how many ticks the sculk sensor should remain active for
* `net.minecraft.world.inventory.CraftingContainer#getItems`
* `net.minecraft.world.inventory.Slot#isHighlightable`
* `com.mojang.blaze3d.platform.NativeImage#applyToAllPixels` - Applies a transformation to the colors of the pixels of an image
* `net.minecraft.core.SectionPos#getZeroNode(II)` - Gets the zero node section position from a specified XZ coordinate.
* `net.minecraft.util.DependencySorter` - Basic logic flow for sorting dependencies
* `net.minecraft.util.ExtraCodecs#CODEPOINT`
* `net.minecraft.world.entity.Mob#onPathfindingStart` and `#onPathfindingDone` - Callbacks for pathfinding
* `net.minecraft.world.level.chunk.ChunkAccess#getHighestGeneratedStatus`
* The telemetry system has been more flushed out with hooks into advancements and realms.
* `net.minecraft.gametest.framework.GameTestHelper#withLowHealth`
* `net.minecraft.network.syncher.SynchedEntityData#hasItem`
* `net.minecraft.util.ExtraCodecs#validate` - General validator to apply to an `Codec#flatXmap`
* An entity now keeps track of the blocks that are colliding, or 'supporting', the entity and preventing them from continuing to fall down. The 'supporting' block can be checked via `Entity#isSupportedBy`
* `net.minecraft.advancements.Advancement$Builder#recipeAdvancement` - Constructs an advancement for a recipe and does not send a telemetry event.
* `net.minecraft.Util#fixedSize(LongStream, I)`
* `net.minecraft.gametest.framework.GameTestHelper#makeMockServerPlayerInLevel`
* `net.minecraft.server.level.ChunkLevel` - A helper class for managing chunk statuses and loading states.
* `net.minecraft.world.entity.Entity#setPortalCooldown(I)` and `#getPortalCooldown`
* `net.minecraft.world.entity.LivingEntity#getLootTableSeed`
* `net.minecraft.world.entity.boss.enderdragon.EnderDragon#setDragonFight`, `#setFightOrigin`, `#getFightOrigin`
* `net.minecraft.world.level.levelgen.Xoroshiro*` classes now have codecs
* `net.minecraft.Util#isWhitespace` and `#isBlank`
    * Minecraft alternatives for `org.apache.commons.lang3.StringUtils`
* `net.minecraft.client.model.HierarchicalModel#applyStatic` - Used to apply static transforms using the animation system
* `net.minecraft.world.entity.Entity#isOnRails` - Currently only `true` for minecarts when on rails
* `net.minecraft.world.item.crafting.Ingredient#fromJson(JsonElement, boolean)` - When the boolean is true, air will return a `null` ingredient instead of throwing an exception.
    * This is the default in the `#fromJson(JsonElement)` overload
* `net.minecraft.world.level.BaseCommandBlock#isValid` - Checks whether the command block menu can be accessed
* `net.minecraft.client.gui.screens.FaviconTexture`
* `net.minecraft.gametest.framework.GameTestHelper#killAllEntitiesOfClass`
* `net.minecraft.gametest.framework.GameTestHelper#assertRedstoneSignal`
* `net.minecraft.world.entity.Entity#setOnGroundWithKnownMovement` - Sets the user as being on the ground along with the block the entity plans to stand on.
* `net.miencraft.world.level.block.EquipableCarvedPumpkinBlock`
* `net.miencraft.world.level.levelgen.RandomSupport#upgradeSeedTo128bitUnmixed` - Does not apply a stafford 13 mix to the seed
* `net.minecraft.client.KeyMapping#resetToggleKeys`
* `net.minecraft.client.gui.components.AbstractScrollWidget#renderBorder`
* `net.minecraft.client.gui.components.FittingMultiLineTextWidget`
* `net.minecraft.util.GsonHelper#getNonNull`
* `net.minecraft.world.level.storage.LevelStorageSource#validateAndCreateAccess`
* `net.minecraft.world.level.validation.ContentValidationException`
* `net.minecraft.world.level.validation.DirectoryValidator`
* `net.minecraft.world.level.validation.PathAllowList`

### Changes and Renames

* `net.minecraft.client.particle.DripParticle#createCherryLeaves*Particle` -> `CherryParticle`
* An entity's shadow raidus is now rendered at a minimum of 32.
* `net.minecraft.server.level.ServerEntity#changedPassengers` -> `#removedPassengers`
* `net.minecraft.world.level.levelgen.structure.templatesystem.ProcessorRule#getOutputTag()` -> `#getOutputTag(RandomSource, CompoundTag)`
* `net.minecraft.core.Direction#fromNormal` -> `#fromDelta`
    * All other `#fromNormal` methods and fields have been removed
* `net.minecraft.world.entity.LivingEntity#*Ridden*` methods now take in a `Player` instead of a `LivingEntity`
* `net.minecraft.world.entity.animal.camel.Camel#standUpPanic` -> `#standUpInstantly`
* `net.minecraft.world.entity.Entity#wasKilled` -> `#killedEntity`
* `net.minecraft.world.level.block.Block#isPossibleToRespawnInThis()` -> `#isPossibleToRespawnInThis(BlockState)
* `BlockSetType` now accepts a boolean about whether the block can be opened by a hand
* `net.minecraft.client.gui.screens.inventory.tooltip.ClientTooltipPositioner#positionTooltip(Screen, IIIII)` -> `#positionTooltip(IIIIII)`
* `net.minecraft.commands.CommandSourceStack` now takes in a `IntConsumer` for taking in the return value of a command
* `net.minecraft.world.inventory.RecipeHolder#awardUsedRecipes(Player)` -> `#awardUsedRecipes(Player, List<ItemStack>)`
* `net.minecraft.world.level.block.FarmableBlock` is replaced by the block tag `minecraft:maintains_farmland`
* `net.minecraft.client.Minecraft` or `net.minecraft.server.MinecraftServer` - `#getServiceSignatureValidator` -> `#getProfileKeySignatureValidator`
* `net.minecraft.data.recipes.RecipeProvider#trimSmithing` now takes in a `ResourceLocation` representing the name of the recipe JSON
* `net.minecraft.server.level.ServerPlayer#getLevel` -> `#serverLevel`
* `net.minecraft.server.level.ServerPlayer#setLevel` -> `#setServerLevel`
* `net.minecraft.util.SpawnUtil$Strategy#LEGACY_IRON_GOLEM` has been deprecated
* `net.minecraft.world.entity.Entity#level` field -> `#level` method
   * The `level` field is now private
   * Settings the field is now done via `#setLevel`
* `net.minecraft.world.entity.Entity#isOnGround` -> `onGround` method
   * The `onGround` field is now private
   * Setting the field is now done via `#setOnGround`
* `net.minecraft.world.entity.OwnableEntity#getLevel` -> `#level`
* `net.minecraft.world.entity.ai.behavior.FollowTemptation` can now take in a double representing the 'close enough' radius
* `net.minecraft.world.level.chunk.ChunkAccess#getHighestSectionPosition` has been deprecated for removal
* `net.minecraft.client.renderer.LevelRenderer#renderVoxelShape` takes in a additional boolean to change the color of the rendered voxel in debug renderering.
* `net.minecraft.client.renderer.entity.ItemRenderer` now checks for an animated texture that has foil using `#hasAnimatedTexture`
    * This method is currently private and hardcoded to the `compasses` item tag or the clock.
* `net.minecraft.world.entity.LivingEntity#getJumpBoostPower` now returns a `float`
* `net.minecraft.client.player.LocalPlayer#portalTime` and `#oPortalTime` -> `spinningEffectIntensity` and `#oSpinningEffectIntensity`
* `net.minecraft.data.recipes.RecipeProvider#coloredWoolFromWhiteWoolAndDye` and `#coloredCarpetFromWhiteCarpetAndDye` have been replaced by `#colorBlockWithDye` which takes in a list of dyes and a list of dyed objects
* `net.minecraft.server.level.ChunkHolder$FullChunkStatus` -> `level.FullChunkStatus`
    * The enum values have been changed as well: `BORDER` -> `FULL` and `TICKING` -> `BLOCK_TICKING`
* `net.minecraft.world.damagesource.DamageSources#outOfWorld` -> `#fellOutOfWorld`
* `net.minecraft.world.entity.Entity#outOfWorld`, `#checkOutOfWorld` -> `#onBelowWorld`, `#checkBelowWorld`
* `net.minecraft.server.level.ServerLevel#dragonFight` -> `#getDragonFight`
* `net.minecraft.world.entity.Entity#getOnPos(F)` is now protected instead of private
* `net.minecraft.world.inventory.CraftingContainer` -> `TransientCraftingContainer`
    * `CraftingContainer` is now an interface which `TransientCraftingContainer` extends which specifies the width, height, and list of items within
* `net.minecraft.world.item.ItemStack#sameItem` -> `#isSameItem`
* `net.minecraft.commands.CommandSourceStack#sendSuccess(Component, boolean)` -> `#sendSuccess(Supplier<Component>, boolean)`
* `net.minecraft.world.entity.ai.behavior.FollowTemptation(Function<LivingEntity, Float>, double)` -> `FollowTemptation(Function<LivingEntity, Float>, Function<LivingEntity, Double>)`
* `net.minecraft.client.gui.components.AbstractSelectionList#clearEntries` is now non-final
* `net.minecraft.world.damagesource.CombatEntry` is now a record
* `net.minecraft.world.entity.Entity#positionRider(Entity)` is now final while `#positionRider(Entity, MoveFunction)` is now protected instead of private
* `net.minecraft.world.level.block.Block#dropResources` - the entity is now nullable
* `net.minecraft.server.level.ServerPlayer#doCheckFallDamage(double, boolean)` -> `#doCheckFallDamage(double, double, double, boolean)` - now takes in the XZ positions
* `net.minecraft.world.entity.LivingEntity#sendEffectToRider` -> `#sendEffectToPassengers`
* `net.minecraft.client.gui.components.AbstractScrollWidget#renderBackground` is now `protected` instead of `private`
* `net.minecraft.world.item.crafting.ShapedRecipe#itemFromJson` is now `public` instead of `private`
* `net.miencraft.world.level.storage.WorldLevelData#endDragonFightData` and `#setEndDragonFightData` now operate on `EndDragonFight$Data`

### Removals

* `net.minecraft.world.entity.Entity#teleportPassengers`
* `net.minecraft.gametest.framework.GameTestHelper#continuouslyUse`
* `net.minecraft.data.recipes.SmithingTrimRecipeBuilder#save(Consumer, String)`
* `block` and `new_entity` shaders as they have been unused by Minecraft
* `net.minecraft.world.item.ItemStack#tagMatches` and `#isSame`
* `net.minecraft.world.level.block.Block#dropResources(BlockState, LootContext$Builder)`
* `net.minecraft.world.level.block.Fallable#getHurtsEntitySelector`
* `net.minecraft.world.level.chunk.ChunkStatus` no longer takes in a string name
