# Minecraft 1.20.6 -> 1.21 Mod Migration Primer

This is a high level, non-exhaustive overview on how to migrate your mod from 1.20.6 to 1.21. This does not look at any specific mod loader, just the changes to the vanilla classes. All provided names use the official mojang mappings.

This primer is licensed under the [Creative Commons Attribution 4.0 International](http://creativecommons.org/licenses/by/4.0/), so feel free to use it as a reference and leave a link so that other readers can consume the primer.

If there's any incorrect or missing information, please file an issue on this repository or ping @ChampionAsh5357 in the Neoforged Discord server.

## Pack Changes

There are a number of user-facing changes that are part of vanilla which are not discussed below that may be relevant to modders. You can find a list of them on [Misode's version changelog](https://misode.github.io/versions/?id=1.21&tab=changelog).

## Moving Experimental Features

All experimental features which were disabled with the `update_1_21` flag are now moved to their proper locations and implementations. Removed features flags can be seen within `net.minecraft.world.level.storage.WorldData#getRemovedFeatureFlags`.

## ResourceLocation, now Private

The `ResourceLocation` is final and its constructor is private. There are alternatives depending on your usecase:

- `new ResourceLocation(String, String)` -> `fromNamespaceAndPath(String, String)`
- `new ResourceLocation(String)` -> `parse(String)`
- `new ResourceLocation("minecraft", String)` -> `withDefaultNamespace(String)`
- `of` -> `bySeparator`
- `isValidResourceLocation` is removed

## Depluralizing Registry and Tag Folders

Plural references to the block, entity type, fluid, game event, and item tags have been removed. They should now use their exact registry name. The same goes for registry folders.

- `tags/blocks` -> `tags/block`
- `tags/entity_types` -> `tags/entity_type`
- `tags/fluids` -> `tags/fluid`
- `tags/game_events` -> `tags/game_event`
- `tags/items` -> `tags/item`
- `advancements` -> `advancement`
- `recipes` -> `recipe`
- `structures` -> `structure`'
- `loot_tables` -> `loot_table`

## Oh Rendering, why must you change so?

There have been a number of rendering changes. While this will not be an in-depth overview, it will cover most of the surface-level changes.

### Vertex System

The vertex system has received a major overhaul, almost being completely rewritten. However, most of the codebase has some analogue to its previous version in different locations.

First, a `VertexConsumer` is obtained using one of two methods: from a `MultiBufferSource` or the `Tesselator`. Both essentially take a `ByteBufferBuilder`, which handles the direct allocation of the vertex information, and wrap it in a `BufferBuilder`, which writes the data to the `ByteBufferBuilder` while also keeping track of other settings needed to properly write the data, such as the `VertexFormat`.

The `MultiBufferSource` constructs a `VertexConsumer` via `#getBuffer` while `Tesselator` does so via `Tesselator#begin`.

```java
// For some MultiBufferSource bufferSource
VertexConsumer buffer = bufferSource.getBuffer(RenderType.translucent());

// Note the different return types when using
// We will need the BufferBuilder subclass in the future
BufferBuilder buffer = Tesselator.getInstance().begin(VertexFormat.Mode.QUADS, DefaultVertexFormat.POSITION_TEX_COLOR);
```

Next, vertices are added to the `VertexConsumer` using the associated methods. `#addVertex` must always be called first, followed by the settings specified in the `VertexFormat`. `#endVertex` no longer exists and is called automatically when calling `#addVertex` or when uploading the buffer.

```java
// For some VertexConsumer buffer
buffer.addVertex(0.0f, 0.0f, 2000.0f).setUv(0, 0).setColor(-1);
buffer.addVertex(0.0f, 1.0f, 2000.0f).setUv(0, 0).setColor(-1);
buffer.addVertex(1.0f, 1.0f, 2000.0f).setUv(0, 0).setColor(-1);
buffer.addVertex(1.0f, 0.0f, 2000.0f).setUv(0, 0).setColor(-1);
```

Once the vertices are added, those using `MultiBufferSource` are done. `MultiBufferSource` will batch the buffers together for each `RenderType` and call the `BufferUploader` within the pipeline. If the `RenderType` sorts the vertices on upload, then it will do so when `endBatch` is called, right before `RenderType#draw` is called to setup the render state and draw the data.

The `Tesselator`, on the other hand, does not handle this logic as you are creating the `BufferBuilder` instance to manage. In this case, after writing all ther vertices, `BufferUploader#drawWithShader` should be called. The `MeshData` provided, which contains the vertex and index buffer along with the draw state, can be built via `BufferBuilder#buildOrThrow`. This replaces `BufferBuilder#end`.

```java
// For some BufferBuilder buffer
BufferUploader.drawWithShader(buffer.buildOrThrow());
```

#### Changes

- `com.mojang.blaze3d.vertex.DefaultVertexFormat`'s `VertexFormatElement`s have been moved to `VertexFormatElement`.
- `com.mojang.blaze3d.vertex.BufferBuilder$RenderedBuffer` -> `MeshData`
- `com.mojang.blaze3d.vertex.VertexConsumer`
    - `vertex` -> `addVertex`
        - Overload with `PoseStack$Pose, Vector3f`
    - `color` -> `setColor`
    - `uv` -> `setUv`
    - `overlayCoords` -> `setUv1`, `setOverlay`
    - `uv2` -> `setUv2`, `setLight`
    - `normal` -> `setNormal`
    - `endVertex` is removed
    - `defaultColor`, `color` -> `setColor`, `setWhiteAlpha`
- `net.minecraft.client.model.Model#renderToBuffer` now takes in an integer representing the ARGB tint instead of four floats
    - There is also a final method which passes in no tint
- `net.minecraft.client.model.geom.ModelPart#render`, `$Cube#compile` now takes in an integer representing the ARGB tint instead of four floats
    - There is also an overloaded method which passes in no tint
- `net.minecraft.client.particle.ParticleRenderType#begin(Tesselator, TextureManager)`, `end` -> `begin(BufferBuilder, TextureManager)`
    - This method returns the `BufferBuilder` rather than void
    - When `null`, no rendering will occur
- Shader removals and replacements
    - `minecraft:position_color_tex` -> `minecraft:position_tex_color`
    - `minecraft:rendertype_armor_glint` -> `minecraft:rendertype_armor_entity_glint`
    - `minecraft:rendertype_glint_direct` -> `minecraft:rendertype_glint`
- `net.minecraft.client.renderer.MultiBufferSource`
    - All `BufferBuilder`s have been replaced with `ByteBufferBuilder`
    - `immediateWithBuffers` now takes in a `SequencedMap`
- `net.minecraft.client.renderer.RenderType`
    - `end` -> `draw`
    - `sortOnUpload` - When true, sorts the quads according to the `VertexSorting` method for the `RenderType`
- `net.minecraft.client.renderer.SectionBufferBuilderPack#builder` -> `#buffer`
- `net.minecraft.client.renderer.ShaderInstance` no longer can change the blend mode, only `EffectInstance` can, which is applied for `PostPass`
    - `setDefaultUniforms` - Sets the default uniforms accessible to all shader instances
- `net.minecraft.client.renderer.entity.ItemRenderer`
    - `getArmorFoilBuffer` no longer takes in a boolean to change the render type
    - `getCompassFoilBufferDirect` is removed
- `net.minecraft.client.renderer.entity.layers.RenderLayer#coloredCutoutModelCopyLayerRender`, `renderColoredCutoutModel` takes in an integer representing the color rather than three floats
- `com.mojang.blaze3d.vertex.BufferVertexConsumer` is removed
- `com.mojang.blaze3d.vertex.DefaultedVertexConsumer` is removed

### Chunk Regions

- `net.minecraft.client.renderer.chunk.RenderRegionCache#createRegion` only takes in the `SectionPos` now instead of computing the section from the `BlockPos`
- `net.minecraft.client.renderer.chunk.SectionCompiler` - Renders the given chunk region via `#compile`. Returns the results of the rendering compilation.
    - This is used within `SectionRenderDispatcher` now to pass around the stored results and upload them

## The Enchantment Datapack Object

`Enchantment`s are now a datapack registry object. Querying them requires access to a `HolderLookup.Provider` or one of its subclasses.

All references to `Enchantment`s are now wrapped with `Holder`s. Some helpers can be found within `EnchantmentHelper`.

```json5
{
    // A component containing the description of the enchantment
    // Typically will be a component with translatable contents
    "description": {
        "translate": "enchantment.minecraft.example"
    },
    // An item tag that holds all items this enchantment can be applied to
    "supported_items": "#minecraft:enchantable/weapon",
    // An item tag that holds a subset of supported_items that this enchantment can be applied to in an enchanting table
    "primary_items": "#minecraft:enchantable/sharp_weapon",
    // A non-negative integer that provides a weight to be added to a pool when trying to get a random enchantment
    "weight": 3,
    // A non-negative integer that indicates the maximum level of the enchantment
    "max_level": 4,
    // The minimum cost necessary to apply this enchantment to an item
    "min_cost": {
        // The base cost of the enchantment at level 1
        "base": 10,
        // The amount to increase the cost with each added level
        "per_level_above_first": 20
    },
    // The maxmimum cost required to apply this enchantment to an item
    "max_cost": {
        "base": 60,
        "per_level_above_first": 20
    }
    // A non-negative integer that determines the cost to add this enchantment via the anvil
    "anvil_cost": 5,
    // The equipment slot groups this enchantment is applied to
    // Can be 'any', 'hand' ('mainhand' and 'offhand'), 'armor' ('head', 'chest', 'legs', and 'feet'), or 'body'
    "slots": [
        "hand"
    ],
    // An enchantment tag that contains tags that this enchantment cannot be on the same item with
    "exclusive_set": "#minecraft:exclusive_set/damage",
    // An encoded data component map which contains the effects to apply
    "effects": {
        // Read below
    },
}
```

### EnchantmentEffectComponents

`EnchantmentEffectComponents` essentially apply how enchantments should react in a given context when on an item. Each effect component is encoded as an object with its component id as the key and the object value within the effect list, usually wrapped as a list. Most of these components are wrapped in a `ConditionalEffect`, so that will be the focus of this primer.

#### ConditionalEFfect

`ConditionalEffect`s are basically a pair of the effect to apply and a list of loot conditions to determine when to execute the enchantment. The codec provided contains the effect object code and the loot context param sets to apply for the conditions (commonly one of the `ENCHANTED_*` sets).

#### The Effect Objects

Each effect object has its own combination of codecable objects and abstract logic which may refer to other registered types, similiar to loot conditions, functions, number providers, etc. For enchantments currently, this means enchantments which are applied directly to the entity, scaled and calulated for some numerical distribution, applied to the blocks around the entity, or calculating a number for some information.

For example, all protection enchantments use almost the exact same effect objects (`minecraft:attributes` and `minecraft:damage_protection`) but use the conditions to differentiate when those values should be applied.

To apply these enchantments, `EnchantmentHelper` has a bunch of different methods to help with the application. Typically, most of these are funneled through one of the `runIterationOn*` methods. These method take in the current context (e.g., stack, slot, entity) and a visitor which holds the current enchantment, level, and optionally the stack context if the item is in use. The visitor takes in a mutable object which holds the final value to return. The modifications to apply to the given item(s) are within `Enchantment#modify*`. If the conditions match, then the value from the mutable object is passed in, processed, and set back to the object.

### Enchantment Providers

`EnchantmentProvider`s are a provider of enchantments to items for entity inventory items or for custom loot on death. Each provider is some number of enchantments that is then randomly selected from and applied to the specified stacks via `EnchantmentHelper#enchantItemFromProvider` which takes in the stack to enchant, the registry access, the provider key, the difficulty instance, and the random instance.

### Minor Changes

- `net.minecraft.world.entity.LivingEntity#activeLocationDependentEnchantments` - Returns the enchantments which depend on the current location of the entity
- `net.minecraft.world.entity.Entity#doEnchantDamageEffects` has been removed
- `net.minecraft.world.entity.npc.VillagerTrades$ItemListing` implementations with `Enchantment`s now take in keys or tags of the corresponding object
- `net.minecraft.world.entity.player.Player#getEnchantedDamage` - Gets the dmaage of a source modified by the current enchantments
- `net.minecraft.world.entity.projectile.AbstractArrow#hitBlockEnchantmentEffects` - Applies the enchantment modifiers when a block is hit
- `net.minecraft.world.entity.projectile.AbstractArrow#setEnchantmentEffectsFromEntity` has been removed
- `net.minecraft.world.item.ItemStack#enchant` takes in a `Holder<Enchantment>` instead of the direct object
- `net.minecraft.world.entity.Mob#populateDefaultEquipmentEnchantments`, `enchantSpawnedWeapon`, `enchantSpawnedArmor` now take in a `ServerLevelAccessor` and `DifficultyInstance`

## The Painting Variant Datapack Object

`PaintingVariant`s are now a datapack registry object. Querying them requires access to a `HolderLookup.Provider` or one of its subclasses.

```json5
{
    // A relative location pointing to 'assets/<namespace>/textures/painting/<path>.png
    "asset_id": "minecraft:courbet",
    // A value between 1-16 representing the number of blocks this variant takes up
    // e.g. a width of 2 means it takes up 2 blocks, and has an image size of 32px
    "width": 2,
    // A value between 1-16 representing the number of blocks this variant takes up
    // e.g. a height of 1 means it takes up 1 blocks, and has an image size of 16px
    "height": 1
}
```

## Attribute Modifiers, now with ResourceLocations

`AttributeModifier`s no longer take in a `String` representing its UUID. Instead, a `ResourceLocation` is provided to uniquely identity the modifier to apply. Attribute modifiers can be compared for `ResourceLocation`s using `#is`.

- `net.minecraft.world.effect.MobEffect`
    - `addAttributeModifier`, `$AttributeTemplate` takes in a `ResourceLocation` instead of a `String`
- `net.minecraft.world.entity.ai.attributes.AttributeInstance`
    - `getModifiers` returns a `Map<ResourceLocation, AttributeModifier>`
    - `getModifier`, `hasModifier`, `removeModifier` now takes in a `ResourceLocation`
    - `removeModifier` now returns a boolean
    - `removePermanentModifier` is removed
    - `addOrReplacePermanentModifier` - Adds or replaces the provided modifier
- `net.minecraft.world.entity.ai.attributes.AttributeMap`
    - `getModifierValue`, `hasModifier`, now takes in a `ResourceLocation`
- `net.minecraft.world.entity.ai.attributes.AttributeSupplier`
    - `getModifierValue`, `hasModifier`, now takes in a `ResourceLocation`

## RecipeInput

`Recipe`s now take in a `net.minecraft.world.item.crafting.RecipeInput` instead of a `Container`. A `RecipeInput` is basically a minimal view into a list of available stacks. It contains three methods:

- `getItem` - Returns the stack in the specified index
- `size` - The size of the backing list
- `isEmpty` - true if all stacks in the list are empty

As such, all implementations which previous took a `Container` now take in a `RecipeInput`. You can think of it as replacing all the `C` generics with a `T` generic representing the `RecipeInput`. This also includes `RecipeHolder`, `RecipeManager`, and the others below.

### CraftingInput

`CraftingInput` is an implementation of `RecipeInput` which takes in width and height of the grid along with a list of `ItemStack`s. One can be created using `CraftingInput#of`. These are used in crafting recipes.

### SingleRecipeInput

`SingleRecipeInput` is an implementation of `RecipeInput` that only has one item. One can be created using the constructor.

## Changing Dimensions

How entities change dimensions have been slightly reworked in the logic provided. Instead of providing a `PortalInfo` to be constructed, instead everything returns a `DimensionTransition`. `DimensionTransition` contains the level to change to, the entity's position, speed, and rotation, and whether a respawn block should be checked. `DimensionTransition` replaces `PortalInfo` in all scenarios.

Entities have two methods for determining whether they can teleport. `Entity#canUsePortal` returns true if the entity can use a `Portal` to change dimensions, where the boolean supplies allows entities who are passengers to teleport. To actually change dimensions, `Entity#canChangeDimensions` must return true, where the current and teleporting to level is provided. Normally, only `canUsePortal` is changed while `canChangeDimensions` is always true.

> `canChangeDimensions` functioned as `canUsePortal` in 1.20.5/6. 

To change dimensions via a portal, a `Portal` should be supplied to `Entity#setAsInsidePortal`. A `Portal` contains how long it takes for the portal to transition, the `DimensionTransition` destination, and the transition effect to apply to the user. This is all wrapped in a `PortalProcessor` and stored on the `Entity`. Once the player is teleported a `DimensionTransition$PostDimensionTransition` is executed, for example playing a sound.

`Portal`s are generally implmented on the `Block` doing the teleporting.

- `net.minecraft.world.entity.Entity`
    - `changeDimension(ServerLevel)` -> `changeDimension(DimensionTransition)`
    - `findDimensionEntryPoint` -> `Portal#getPortalDestination`
    - `getPortalWaitTime` -> `Portal#getPortalTransitionTime`
    - `handleInsidePortal` -> `setAsInsidePortal`
    - `handleNetherPortal` -> `handlePortal`
    - `teleportToWithTicket` is removed
    - `getRelativePortalPosition` is now public
- `net.minecraft.world.level.portal.PortalShape`
    - `createPortalInfo` is removed
    - `findCollisionFreePosition` is now public
- `net.minecraft.client.player.LocalPlayer#getActivePortalLocalTransition` - Defines the transition to apply to the entity when the player is within a portal
- `net.minecraft.world.level.portal.PortalForce#findPortalAround` -> `findClosestPortalPosition`

### Minor Changes

- `net.minecraft.recipebook.ServerPlaceRecipe` now has a generic taking in the `RecipeInput` and the `Recipe` rather than the `Container`
- `net.minecraft.world.inventory.CraftingContainer#asCraftInput` - Converts a crafting container to an input to supply
- `net.minecraft.world.inventory.CraftingContainer#asPositionedCraftInput` - Converts a crafting container to an input positioned at some location within a grid
- `net.minecraft.world.inventory.CraftingMenu#slotChangedCraftingGrid` now takes in a `RecipeHolder`
- `net.minecraft.world.inventory.RecipeBookMenu` now has a generic taking in the `RecipeInput` and the `Recipe` rather than the `Container`
  - `beginPlacingRecipe` - When the recipe is about to be placed in the corresponding location
  - `finishPlacingRecipe` - AFter the recipe has been placed in the corresponding location
- `net.minecraft.client.gui.screens.recipebook.*RecipeComponent#addItemToSlot` still exists but is no longer overridded from its subclass

## Minor Migrations

The following is a list of useful or interesting additions, changes, and removals that do not deserve their own section in the primer.

### Options Screens Movement

The options screens within `net.minecraft.client.gui.screens` and `net.minecraft.client.gui.screens.controls` have been moved to `net.minecraft.client.gui.screens.options` and `net.minecraft.client.gui.screens.options.controls`, respectively.

### The HolderLookup$Provider in LootTableProvider

`net.minecraft.data.loot.LootTableProvider$SubProviderEntry` takes in a function which provides the `HolderLookup$Provider` and returns the `LootTableSubProvider`. `LootTableSubProvider#generate` no longer takes in a `HolderLookup$Provider` as its first argument.

### DecoratedPotPattern Object

`Registries#DECORATED_POT_PATTERNS` takes in a `DecoratedPotPattern` instead of the raw string for the asset id. This change has been reflected in all subsequent classes (e.g. `Sheets`).

### Jukebox Playable

`RecordItem`s has been removed in favor of a new data component `JukeboxPlayable`. This is added via the item properties. A jukebox song is played via `JukeboxSongPlayer`. `JukeboxBlockEntity` handles an implementation of `JukeboxSongPlayer`, replacing all of the logic handling within the class itself.

- `net.minecraft.world.item.Item$Properties#jukeboxPlayable` - Sets the song to play in a jukebox when this is added
- `net.minecraft.world.item.JukeboxSong` - A `SoundEvent`, description, length, and comparator output record indicating what this item would do when placed in a jukebox
    - This is a datapack registry
- `net.minecraft.world.item.JukeboxPlayable` - The data component for a `JukeboxSong`
- `net.minecraft.world.item.JukeboxSongPlayer` - A player which handles the logic of how a jukebox song is played
- `net.minecraft.world.level.block.entity.JukeboxBlockEntity#getComparatorOutput` - Gets the comparator output of the song playing in the jukebox

### Chunk Generation Reorganization

Chunk generation has been reorganized again, where generation logic is moved into separate maps, task, and holder classes to be executed and handled asyncronously in most instances. Missing methods can be found in `ChunkGenerationTask`, `GeneratingChunkMap`, or `GenerationChunkHolder`. Additional chunk generation handling is within `net.minecraft.world.level.chunk.status.*`.

This also means some region methods which only contain generation-based information have been replaced with one of these three classes instead of their constructed counterparts. The steps are now listed in a `ChunkPyramid`, which determines the steps takes on how a protochunk is transformed into a full chunk. Each step can have a `ChunkDependencies`, forcing the order of which they execute. `ChunkPyramid#GENERATION_PYRAMID` handles chunk generation during worldgen while `LOADING_PYRAMID` handles loading a chunk that has already been generated. The `WorldGenContext` holds a reference to the main thread box when necessary.

### Delta Tracker

`net.minecraft.client.DeltaTracker` is an interface mean to keep track of the delta ticks, both ingame and real time. This also holds the current partial tick. This replaces most instances passing around the delta time or current partial tick.

- `net.minecraft.client.Minecraft#getFrameTime`, `getDeltaFrameTime` -> `getTimer`
- `net.minecraft.client.Timer` -> `DeltaTracker$Timer`
- `net.minecraft.client.gui.Gui#render` now takes in a `DeltaTracker`

### Additions

- `com.mojang.realmsclient.dto.RealmsServer#isMinigameActive` - Returns whether a minigame is active.
- `net.minecraft.SystemReport#sizeInMiB` - Converts a long representing the number of bytes into a float representing Mebibyte (Power 2 Megabyte)
- `net.minecraft.Util#isSymmetrical` - Checks whether a list, or some subset, is symmetrical
- `net.minecraft.advancements.critereon.MovementPredicate` - A predicate which indicates how the entity is currently moving
- `net.minecraft.client.gui.components.AbstractSelectionList#clampScrollAmount` - Clamps the current scroll amount
- `net.minecraft.client.gui.screens.AccessibilityOnboardingScreen#updateNarratorButton` - Updates the narrator button
- `net.minecraft.client.gui.screens.worldselection.WorldCreationContext#validate` - Validates the generators of all datapack dimensions
- `net.minecraft.client.multiplayer.ClientPacketListener`
  - `updateSearchTrees` - Rebuilds the search trees for creative tabs and recipe collections
  - `searchTrees` - Gets the current search trees for creative tabs and recipe collections
- `net.minecraft.client.multiplayer.SessionSearchTrees` - The data contained for creative tabs and recipe collections on the client
- `net.minecraft.commands.arguments.item.ItemParser$Visitor#visitRemovedComponent` - Executed when a component is to be removed as part of an item input
- `net.minecraft.core.Registry#getAny` - Gets an element from the registry, usually the first one registered
- `net.minecraft.core.component.DataComponentMap`
  - `makeCodec` - Creates a component map codec from a component type codec
  - `makeCodecFromMap` - Creates a component map codec from a component type to object map codec
- `net.minecraft.util.ProblemReporter#getReport` - Returns a report of the problem
- `net.minecraft.util.Unit#CODEC`
- `net.minecraft.world.damagesource.DamageSources#campfire` - Returns a source where the damage was from a campfire
- `net.minecraft.world.damagesource.DamageType#CODEC`
- `net.minecraft.world.entity.LivingEntity`
  - `hasLandedInLiquid` - Returns whether the entity is moving faster than -0.00001 in the y direction while in a liquid
  - `getKnockback` - Gets the knockback to apply to the entity
- `net.minecraft.world.entity.Mob#playAttackSound` - Executes when the mob should play a sound when attacking
- `net.minecraft.world.entity.ai.attributes.Attribute#CODEC`
- `net.minecraft.world.entity.ai.attributes.AttibuteMap`
  - `addTransientAttributeModifiers` - Adds all modifiers from the provided map
  - `removeAttributeModifiers` - Removes all modifiers in the provided map
- `net.minecraft.world.entity.projectile.AbstractArrow`
  - `getWeaponItem` - The weapon the arrow was fired from
  - `setBaseDamageFromMob` - Sets the base damage this arrow can apply
- `net.minecraft.world.entity.projectile.Projectile#calculateHorizontalHurtKnockbackDirection` - Returns the horizontal vector of the knockback direction
- `net.minecraft.world.inventory.ArmorSlot` - A slot for armor
- `net.minecraft.world.item.CrossbowItem#getChargingSounds` - Sets the sounds to play when pulling back the crossbow
- `net.minecraft.world.item.Item#postHurtEntity` - Gets called after the enemy has been hurt by this item
- `net.minecraft.world.level.Explosion#canTriggerBlocks` - Returns whether the explosion cna trigger a block
- `net.minecraft.world.level.block.LeverBlock#playSound` - Plays the lever click sound
- `net.minecraft.world.level.levelgen.blockpredicates.BlockPredicate`
  - `unobstructed` - Returns a predicate which indicates the current vector is not obstructed by another entity
- `net.minecraft.world.level.storage.loot.functions.EnchantedCountIncreaseFunction` - A function which grows the loot based on the corresponding enchantment level
- `net.minecraft.world.level.storage.loot.predicates.EnchantmentActiveCheck` - Checks whether a given enchantment is applied
- `net.minecraft.world.level.storage.loot.providers.number.EnchantmentLevelProvider` - Determines the enchantment level to provide
- `net.minecraft.world.phys.AABB#move` - An overload of `#move` which takes in a `Vector3f`
- `net.minecraft.core.dispenser.DefaultDispenseItemBehavior#consumeWithRemainder` - Shrinks the first item stack. If empty, then the second item stack is returned. Otherwise, the second item stack is added to an inventory or dispensed, and the first item stack is returned.
- `net.minecraft.server.MinecraftServer`
    - `throwIfFatalException` - Throws an exception if its variable isn't null
    - `setFatalException` - Sets the fatal exception to throw
- `net.minecraft.util.StaticCache2D` - A two-dimensional read-only array which constructs all objects on initialization
- `net.minecraft.world.entity.Entity`
    - `fudgePositionAfterSizeChange` - Returns whether the entity's position can be moved to an appropriate free position after its dimension's change
    - `getKnownMovement` - Returns the current movement of the entity, or the last known client movement of the player
- `net.minecraft.world.entity.ai.attributes.Attribute`
    - `setSentiment` - Sets whether the attribute provides a positive, neutral or negative benefit when the value is positive
    - `getStyle` - Gets the chat formatting of the text based on the sentiment and whether the attribite value is positive
- `net.minecraft.world.entity.ai.attributes.AttributeMap#getAttributestoSync` - Returns the attributes to sync to the client
- `net.minecraft.world.level.storage.loot.LootContext$Builder#withOptionalRandomSource` - Sets the random source to use for the loot context during generation
    - This is passed by `LootTable#getRandomItems(LootParams, RandomSource)`
- `net.minecraft.client.Options#genericValueOrOffLabel` - Gets the generic value label, or an on/off component if the integer provided is 0
- `net.minecraft.client.gui.GuiGraphics#drawStringWithBackdrop` - Draws a string with a rectangle behind it
- `net.minecraft.gametest.framework.GameTestHelper`
    - `assertBlockEntityData` - Asserts that the block entity has the given information
    - `assertEntityPosition` - Asserts that the enity position is within a bounding box
- `net.minecraft.server.level.ServerPlayer#copyRespawnPosition` - Copies the respawn position from another player
- `net.minecraft.server.players.PlayerList#sendActiveEffects`, `sendActivePlayerEffects` - Sends the current entity's effects to the client using the provided packet listener
- `net.minecraft.util.Mth#hsvToArgb` - Converts an HSV value with an alpha integer to ARGB
- `net.minecraft.world.entity.Entity#absRotateTo` - Rotates the y and x of the entity clamped between its maximum values
- `net.minecraft.world.entity.ai.attributes.AttributeMap#assignBaseValues` - Sets the base value for each attribute in the map
- `net.minecraft.world.item.ItemStack#forEachModifier` - An overload which applies the modifier for each equipment slot in the group
    - Applies to `net.minecraft.world.item.component.ItemAttributeModifiers#forEach` as well
- `net.minecraft.world.level.levelgen.PositionalRandomFactory#fromSeed` - Constructs a random instance from a seed
- `net.minecraft.world.level.levelgen.structure.pools.DimensionPadding` - The padding to apply above and below a structure when attempting to generate
- `net.minecraft.ReportType` - An object that represents a header plus a list of nuggets to display after the header text
- `net.minecraft.advancements.critereon.GameTypePredicate` - A predicate that checks the game type of the player (e.g. creative, survival, etc.)
- `net.minecraft.advancements.critereon.ItemJukeboxPlayablePredicate` - A predicate that checks if the jukebox is playing a song
- `net.minecraft.core.BlockPos#clampLocationWithin` - Clamps the vector within the block
- `net.minecraft.core.registries.Registries`
    - `elementsDirPath` - Gets the path of the registry key
    - `tagsDirPath` - Gets the tags path of the registry key (replaces `TagManager#getTagDir`
- `net.minecraft.network.DisconnectionDetails` - Information as to why the user was disconnected from the current world
- `net.minecraft.server.MinecraftServer#serverLinks` - A list of entries indicating what the link coming from the server is
    - Only used for bug reports
- `net.minecraft.util.FastColor`
    - `$ABGR32#fromArgb32` - Reformats an ARGB32 color into a ABGR32 color
    - `$ARGB32#average` - Averages two colors together by each component
- `net.minecraft.util.Mth#lengthSquared` - Returns the squared distance of three floats
- `net.minecraft.world.entity.LivingEntity#triggerOnDeathMobEffects` - Trigers the mob effects when an entity is killed
- `net.minecraft.world.entity.TamableAnimal`
    - `tryToTeleportToOwner` - Attempts to teleport to the entity owner
    - `shouldTryTeleportToOwner` - Returns true if the entity owner is more than 12 blocks away
    - `unableToMoveToOwner` - Returns true if the animal cannot move to the entity owner
    - `canFlyToOwner` - Returns true if the animal can fly to the owner
- `net.minecraft.world.entity.projectile.Projectile#disown` - Removes the owner of the projectile
- `net.minecraft.world.item.EitherHolder` - A holder which either contains the holder instance or a resource key
- `net.minecraft.world.level.chunk.ChunkAccess#isSectionEmpty` - Returns whether the section only has air
- `net.minecraft.FileUtil#sanitizeName` - Replaces all illegal file charcters with underscores
- `net.minecraft.Util#parseAndValidateUntrustedUri` - Returns a URI after validating that the protocol scheme is supported
- `net.minecraft.client.gui.screens.ConfirmLinkScreen` now has two overloads to take in a `URI`
    - `confirmLinkNow` and `confirmLink` also has overloads for a `URI` parameter
- `net.minecraft.client.renderer.LevelRenderer#renderFace` - Renders a quad given two points and a color
- `net.minecraft.client.renderer.entity.EntityRenderer#renderLeash` - A **private** method that renders the leash for an entity
- `net.minecraft.client.resources.model.BlockStateModelLoader` - Loads the blockstate definitions for every block in the registry
    - Anything missing from `ModelBakery` is most likely here
- `net.minecraft.client.resources.model.ModelBakery$TextureGetter` - Gets the `TextureAtlasSprite` given the model location and the material provided
- `net.minecraft.core.BlockPos#getBottomCenter` - Gets the `Vec3` representing the bottom center of the position
- `net.minecraft.gametest.framework.GameTestBatchFactory#fromGameTestInfo(int)` - Batches the game tests into the specified partitions
- `net.minecraft.gametest.framework.GameTestHelper#getTestRotation` - Gets the rotation of the structure from the test info
- `net.minecraft.gametest.framework.GameTestRunner$StructureSpawner#onBatchStart` - Executes when the batch is going to start running within the level
- `net.minecraft.network.ProtocolInfo$Unbound`
    - `id` - The id of the protocol
    - `flow` - The direction the packet should be sent
    - `listPackets` - Provides a visitor to all packets that can be sent on this protocol
- `net.minecraft.network.chat.Component#translationArg` - Creates a component for a `URI`
- `net.minecraft.network.protocol.game.VecDeltaCodec#getBase` - Returns the base vector before encoding
- `net.minecraft.server.level.ServerEntity`
    - `getPositionBase` - Returns the current position of the entity
    - `getLastSentMovement` - Gets the vector representing the last velocity of the entity sent to the client
    - `getLastSentXRot` - Gets the last x rotation of the entity sent to the client
    - `getLastSentYRot` - Gets the last y rotation of the entity sent to the client
    - `getLastSentYHeadRot` - Gets the last y head rotation of the entity sent to the client
- `net.minecraft.world.damagesource.DamageSource#getWeaponItem` - Returns the itemstack the direct entity had
- `net.minecraft.world.entity.Entity`
    - `adjustSpawnLocation` - Returns the block pos representing the spawn location of the entity. By default, returns the entity spawn location
    - `moveTo` has an overload taking in a `Vec3`
    - `placePortalTicket` - Adds a region ticket that there is a portal at the `BlockPos`
    - `getPreciseBodyRotation` - Lerps between the previous and current body rotation of the entity
    - `getWeaponItem` - The item the entity is holding as a weapon
- `net.minecraft.world.entity.Leashable` - Indicates that an entity can be leashed
- `net.minecraft.world.entity.Mob`
    - `dropPreservedEquipment` - Drops the equipment the entity is wearing if it doesn't match the provided predicate or if the equipment succeeds on the drop chance check
- `net.minecraft.world.entity.player.Player`
    - `setIgnoreFallDamageFromCurrentImpulse` - Ignores the fall damage from the current impulses for 40 ticks when true
    - `tryResetCurrentImpulseContext` - Resets the impulse if the grace time reaches 0
- `net.minecraft.world.item.ItemStack#hurtAndConvertOnBreak` - Hurts the stack and when the item breaks, sets the item to another item
- `net.minecraft.world.level.chunk.storage.ChunkIOErrorReporter` - Handles error reporting for chunk IO failures
- `net.minecraft.world.level.chunk.sotrage.ChunkStorage`, `IOWorker`, `RegionFileStorage`, `SimpleRegionStorage#storageInfo` - Returns the `RegionStorageInfo` used to store the chunk
- `net.minecraft.world.level.levelgen.structure.Structure$StructureSettings` now has a constructor with default values given the biome tag
    - `$Builder` - Builds the settings for the structure
- `net.minecraft.world.level.levelgen.structure.templatesystem.LiquidSettings` - The settings to apply for blocks spawning with liquids within them
- `net.minecraft.world.level.storage.loot.ValidationContext` now has an constructor overload taking in an empty `HolderGetter$Provider`
    - `allowsReferences` - Checks whether the resolver is present
- `net.minecraft.client.Options#onboardingAccessibilityFinished` - Disables the onboarding accessibility feature
- `net.minecraft.client.gui.screens.reporting.AbstractReportScreen`
    - `createHeader` - Creates the report header
    - `addContent` - Adds the report content
    - `createFooter` - Creates the report footer
    - `onReportChanged` - Updates the report information, or sets a `CannotBuildReason` if not possible to change
- `net.minecraft.client.multiplayer.chat.report.Report#attested` - Sets whether the user has read the report and wants to send it
- `net.minecraft.world.level.levelgen.feature.EndPlatformFeature` - A feature that generates the end platform
- `net.minecraft.world.level.levelgen.placement.FixedPlacement` - A placement that places a feature in one of the provided positions
- `net.minecraft.world.phys.AABB`
    - `getBottomCenter` - Gets the `Vec3` representing the bottom center of the box
    - `getMinPosition` - Gets the `Vec3` representing the smallest coordinate of the box
    - `getMaxPosition` - Gets the `Vec3` representing the largest coordinate of the box
- `net.minecraft.world.entity.player.Player#isIgnoringFallDamageFromCurrentImpulse` - Returns whether the player should ignore fall damage
- `net.minecraft.world.item.LeadItem#leashableInArea` - Returns a list of Leashable entities within a 7 block radius of the provided block position

### Changes

- `net.minecraft.advancements.critereon.EnchantmentPredicate` now takes in a `HolderSet<Enchantment`
  - The previous constructor still exists as an overload
- `net.minecraft.advancements.critereon.EntityFlagsPredicate` now takes in a boolean for if the entity is on the ground or flying
- `net.minecraft.advancements.critereon.EntityPredicate` now takes in a movement predicate and periodic tick which indicates when the predicate can return true
- `net.minecraft.advancements.critereon.LocationPredicate` now takes in a fluid predicate
- `net.minecraft.client.gui.components.AbstractSelectionList#setScrollAmount` -> `setClampedScrollAmount`
  - `setScrollAmount` now delegates to this method
- `net.minecraft.client.gui.components.OptionsList` now longer takes in the unused integer
- `net.minecraft.client.gui.screens.inventory.CreativeModeInventoryScreen` now takes in a `LocalPlayer` instead of a `Player`
- `net.minecraft.client.resources.language.LanguageManager` now takes in a `Consumer<ClientLanguage>` which acts as a callback when resources are reloaded
- `net.minecraft.client.searchtree.PlainTextSearchTree` -> `SearchTree#plainText`
- `net.minecraft.client.searchtree.RefreshableSearchTree` -> `SearchTree`
- `net.minecraft.commands.arguments.item.ItemInput` now takes in a `DataComponentPatch` instead of a `DataComponentMap`
- `net.minecraft.core.component.TypedDataComponent#createUnchecked` is now public
- `net.minecraft.data.loot.BlockLootSubProvider` now takes in a `HolderLookup$Provider`
  - `HAS_SILK_TOUCH` -> `hasSilkTouch` 
  - `HAS_NO_SILK_TOUCH` -> `doesNotHaveSilkTouch`
  - `HAS_SHEARS_OR_SILK_TOUCH` -> `hasShearsOrSilkTouch`
  - `HAS_NO_SHEARS_OR_SILK_TOUCH` -> `doesNotHaveShearsOrSilkTouch`
  - Most protected static methods were turned into instance methods
- `net.minecraft.data.loot.EntityLootSubProvider` now takes in a `HolderLookup$Provider`
- `net.minecraft.data.recipes.RecipeProvider#trapdoorBuilder` is now protected
- `net.minecraft.recipebook.ServerPlaceRecipe`
  - `addItemToSlot` now takes in an `Integer` instead of a `Iterator<Integer>`
  - `moveItemToGrid` now takes in an integer representing the minimum count and returns the number of items remaining that can be moved with the minimum count in place
- `net.minecraft.resources.RegistryDataLoader`
  - `load` is now private
  - `$RegistryData` now takes in a boolean indicating whether the data must have an element
- `net.minecraft.world.damagesource.CombatRules#getDamageAfterAbsorb` now takes in a `LivingEntity`
- `net.minecraft.world.entity.Entity#igniteForSeconds` now takes in a float
- `net.minecraft.world.entity.LivingEntity`
  - `onChangedBlock` now takes in a `ServerLevel`
  - `getExperienceReward` -> `getBaseExperienceReward`
    - `getExperienceReward` is now final and takes in the `ServerLevel` and `Entity`
  - `getRandom` -> `Entity#getRandom` (usage has not changed)
  - `dropExperience` now takes in a nullable `Entity`
  - `dropCustomDeathLoot` no longer takes in an integer
  - `jumpFromGround` is now public for testing
- `net.minecraft.world.entity.monster.AbstractSkeleton#getArrow` now keeps track of the weapon it was fired from
- `net.minecraft.world.entity.player.Player$startAutoSpinAttack` now takes in a float representing the damage and a stack representing the item being held
- `net.minecraft.world.entity.projectile.AbstractArrow` now keeps track of the weapon it was fired from
  - `setPierceLevel` is now private
- `net.minecraft.world.entity.projectile.ProjectileUtil#getMobArrow` now keeps track of the weapon it was fired from
- `net.minecraft.world.item.CrossbowItem#getChargeDuration` now takes in an `ItemStack` and `LivingEntity`
- `net.minecraft.world.item.Item`
  - `getAttackDamageBonus` now takes in an `Entity` and `DamageSource` instead of just the `Player`
  - `getUseDuration` now takes in a `LivingEntity`
- `net.minecraft.world.item.ItemStack`
  - `hurtAndBreak` now takes in a `ServerLevel` instead of a `RandomSource`
  - `hurtEnemy` now returns a boolean if the entity was succesfully hurt
- `net.minecraft.world.item.MaceItem#canSmashAttack` takes in a `LivingEntity` instead of a `Player`
- `net.minecraft.world.item.ProjectileWeaponItem#shoot` takes in a `ServerLevel` instead of a `Level`
- `net.minecraft.world.item.component.CustomData`
  - `update` now takes in a `DynamicOps<Tag>`
  - `read` now has an overload that takes in a `DynamicOps<Tag>`
- `net.minecraft.world.level.Level$ExplosionInteraction` now implements `StringRepresentable`
- `net.minecraft.world.entity.projectile.windcharge.AbstractWindCharge$WindChargeDamageCalculator` -> `net.minecraft.world.level.SimpleExplosionDamageCalculator`
- `net.minecraft.world.level.block.ButtonBlock#press` now takes in a nullable `Player`
- `net.minecraft.world.level.block.LeverBlock#pull` now takes in a nullable `Player` and does not return anything
- `net.minecraft.world.level.storage.loot.LootContext$EntityTarget`
  - `KILLER` -> `ATTACKER`
  - `DIRECT_KILLER` -> `DIRECT_ATTACKER`
  - `KILLER_PLAYER` -> `ATTACKING_PLAYER`
- `net.minecraft.world.level.storage.loot.functions.CopyNameFunction$NameSource`
  - `KILLER` -> `ATTACKING_ENTITY`
  - `KILLER_PLAYER` -> `LAST_DAMAGE_PLAYER`
- `net.minecraft.world.level.storage.loot.parameters.LootContextParams`
  - `KILLER_ENTITY` -> `ATTACKING_ENTITY`
  - `DIRECT_KILLER_ENTITY` -> `DIRECT_ATTACKING_ENTITY`
- `net.minecraft.world.level.storage.loot.predicates.LootItemConditions#TYPED_CODEC`, `DIRECT_CODEC`, `CODEC` have been moved to `LootItemCondition`
- `net.minecraft.world.level.storage.loot.predicates.LootItemRandomChanceWithLootingCondition` -> `LootItemRandomChanceWithEnchantedBonusCondition`
- `net.minecraft.world.phys.shapes.VoxelShape#getCoords` is now public
- `net.minecraft.advancements.critereon.DamageSourcePredicate` now takes in a boolean of if the damage source is direct
- `net.minecraft.client.gui.components.SubtitleOverlay$Subtitle` is now package private
- `net.minecraft.network.protocol.game.ClientboundProjectilePowerPacket` takes in an `accelerationPower` double instead of power for each of the coordinate components
- `net.minecraft.network.protocol.game.ClientboundSetEntityMotionPacket#get*a` now provides the double representing the actual acceleration vector of the coordinate instead of the shifted integer
- `net.minecraft.world.damagesource.DamageSource#isIndirect` has been replaced with `isDirect`, which does the opposite check
- `net.minecraft.world.entity.Entity#push` now contains an overload that takes in a `Vec3`
- `net.minecraft.world.entity.LivingEntity#eat` is now final
    - A separate overload taking in the `FoodProperties` can now be overridden
- `net.minecraft.world.entity.ai.attributes.AttributeMap#getDirtyAttributes` -> `getAttributestoUpdate`
- `net.minecraft.world.entity.animal.frog.Tadpole#HITBOX_WIDTH`, `HITBOX_HEIGHT` is now final
- `net.minecraft.world.entity.npc.AbstractVillager#getOffers` now throws an error if called on the logical client
- `net.minecraft.world.entity.projectile.AbstractHurtingProjectile` constructors now take in `Vec3` to assign the directional movement of the acceleration power rather than the coordinate powers
    - `ATTACK_DEFLECTION_SCALE` -> `INITIAL_ACCELERATION_POWER`
    - `BOUNCE_DEFELECTION_SCALE` -> `DEFLECTION_SCALE`
    - `xPower`, `yPower`, `zPower` -> `accelerationPower`
- `net.minecraft.world.food.FoodData#eat(ItemStack)` -> `eat(FoodProperties)`
- `net.minecraft.world.food.FoodProperties` now takes in a stack representing what the food turns into once eaten
- `net.minecraft.world.item.CrossbowItem#getChargeDuration` no longer takes in an `ItemStack`
- `net.minecraft.world.item.ItemStack`
    - `transmuteCopy` now has an overload which takes in the current count
    - `transmuteCopyIgnoreEmpty` is now private
- `net.minecraft.world.level.ChunkPos#getChessboardDistance` now has an overload to take in the x and z coordinates without being wrapped in a `ChunkPos`
- `net.minecraft.world.level.block.LecternBlock#tryPlaceBook` now takes in a `LivingEntity` instead of an `Entity`
- `net.minecraft.world.level.block.entity.CampfireBlockEntity#placeFood` now takes in a `LivingEntity` instead of an `Entity`
- `net.minecraft.world.level.chunk.ChunkAccess#getStatus` -> `getPersistedStatus`
- `net.minecraft.world.level.chunk.ChunkGenerator#createBiomes`, `fillFromNoise` no longer takes in an `Executor`
- `net.minecraft.world.level.levelgen.structure.pools.JigsawPlacement#addPieces` now takes in a `DimensionPadding` when determining the starting generation point
- `com.mojang.blaze3d.systems.RenderSystem`
    - `glBindBuffer(int, IntSupplier)` -> `glBindBuffer(int, int)`
    - `glBindVertexArray(Supplier<Integer>)` -> `glBindVertexArray(int)`
    - `setupOverlayColor(IntSupplier, int)` -> `setupOverlayColor(int, int)`
- `net.minecraft.advancements.critereon.EntityPredicate#location`, `steppingOnLocation` has been wrapped into a `$LocationWrapper` subclass
    - This does not affect currently generated entity predicates
- `net.minecraft.client.Options#menuBackgroundBlurriness`, `getMenuBackgroundBlurriness` now returns an integer
- `net.minecraft.client.renderer.GameRenderer#MAX_BLUR_RADIUS` is now public and an integer
- `net.minecraft.gametest.framework.GameTestHelper#getBlockEntity` now throws an exception if the block entity is missing
- `net.minecraft.network.protocol.game.ServerboundUseItemPacket` now takes in the y and x rotation of the item
- `net.minecraft.server.level.ServerLevel#addDuringCommandTeleport`, `#addDuringPortalTeleport` have been combined into `addDuringTeleport` by checking whether the entity is a `ServerPlayer`
- `net.minecraft.server.level.ServerPlayer#seenCredits` is now public
- `net.minecraft.server.players.PlayerList#respawn` now takes in an `Entity$RemovalReason`
- `net.minecraft.world.effect.OozingMobEffect#numberOfSlimesToSpawn(int, int, int)` -> `numberOfSlimesToSpawn(int, NearbySlimes, int)`
- `net.minecraft.world.entity.Entity`
    - `setOnGroundWithKnownMovement` -> `setOnGroundWithMovement`
    - `getBlockPosBelowThatAffectsMyMovement` is now public
- `net.minecraft.world.entity.EquipmentSlot` now takes in an integer represent the max count the slot can have, or 0 if unrestricted
    - Can be applied via `limit` which splits the current item stack
- `net.minecraft.world.entity.LivingEntity`
    - `dropAllDeathLoot`, `dropCustomDeathLoot` now takes in a `ServerLevel`
    - `broadcastBreakEvent` -> `onEquippedItemBroken`
    - `getEquipmentSlotForItem` is now an instance method
- `net.minecraft.world.entity.ai.attributes.AttributeMap#assignValues` -> `assignAllValues`
- `net.minecraft.world.entity.raid.Raider#applyRaidBuffs` now takes in a `ServerLevel`
- `net.minecraft.world.item.ItemStack#hurtAndBreak` now takes in a `Consumer` of the broken `Item` instead of a `Runnable`
- `net.minecraft.CrashReport`
    - `getFriendlyReport` and `saveToFile` now takes in a `ReportType` and a list of header strings
        - There is also an overload that just takes in the `ReportType`
    - `getSaveFile` now returns a `Path`
- `net.minecraft.client.gui.screens.DisconnectedScreeen` can now take in a record containing the disconnection details
    - `DisConnectionDetails` provide an optional path to the report and an optional string to the bug report link
- `net.minecraft.client.renderer.LevelRenderer#playStreamingMusic` -> `playJukeboxSong`
    - `stopJukeboxSongAndNotifyNearby` stops playing the current song
- `net.minecraft.client.resources.sounds.SimpleSoundInstance#forRecord` -> `forJukeboxSong`
- `net.minecraft.client.resources.sounds.Sound` now takes in a `ResourceLocation` instead of a `String`
- `net.minecraft.network.PacketListener`
    - `onDisconnect` now takes in `DisconnectionDetails` instead of a `Component`
    - `fillListenerSpecificCrashDetails` now takes in a `CrashReport`
- `net.minecraft.server.MinecraftServer`
    - `getServerDirectory`, `getFile` now returns a `Path`
    - `isNetherEnabled` -> `isLevelEnabled`
- `net.minecraft.world.entity.ai.behavior.AnimalPanic` now takes in a `Function<PathfinderMob, TagKey<DamageType>>` instead of a simple `Predicate`
- `net.minecraft.world.entity.ai.goal.FollowOwnerGoal` no longer takes in a boolean checking whether the entity can fly
    - These methods have been moved to `TamableAnimal`
- `net.minecraft.world.entity.ai.goal.PanicGoal` now takes in a `Function<PathfinderMob, TagKey<DamageType>>` instead of a simple `Predicate`
- `net.minecraft.world.entity.projectile.Projectile#deflect` now returns a boolean indicating whether the deflection was successful
- `net.minecraft.world.item.CreativeModeTabe#getBackgroundSuffix` -> `getBackgroundTexture`
- `net.minecraft.world.item.DyeColor#getTextureDiffuseColors` -> `getTextureDiffuseColor`
- `net.minecraft.world.level.block.entity.BeaconBlockEntity$BeaconBeamSection#getColor` now returns an integer
- `net.minecraft.world.level.storage.loot.LootDataType` no longer takes in a directory, instead it grabs that from the location of the `ResourceKey`
- `net.minecraft.client.gui.screens.Scnree#narrationEnabled` -> `updateNarratorStatus(boolean)`
- `net.minecraft.client.gui.screens.inventory.HorseInventoryScreen` takes in an integer representing the number of inventory columns to display
- `net.minecraft.client.renderer.block.model.BlockElementFace` is now a record
- `net.minecraft.client.resources.model.ModelBakery#getModel` is now package-private
- `net.minecraft.client.resources.model.ModelResourceLocation` is now a record
    - `inventory` method for items
- `net.minecraft.client.resources.model.UnbakedModel#bakeQuad` no longer takes in a `ResourceLocation`
- `net.minecraft.core.BlockMath#getUVLockTransform` no longer takes in a `Supplier<String>`
- `net.minecraft.gametest.framework.GameTestBatchFactory#toGameTestBatch` is now public
- `net.minecraft.gametest.framework.GameTestRunner` now takes in a boolean indicating whether to halt on error
- `net.minecraft.network.protocol.ProtocolInfoBuilder`
    - `serverboundProtocolUnbound` -> `serverboundProtocol`
    - `clientboundProtocolUnbound` -> `clientboundProtocol`
- `net.minecraft.network.protocol.game.ClientboundAddEntityPacket` now takes in the `ServerEntity` for the first two packet constructors
- `net.minecraft.server.MinecraftServer` now implements `ChunkIOErrorReporter`
- `net.minecraft.util.ExtraCodecs#QUATERNIONF_COMPONENTS` now normalizes the quaternion on encode
- `net.minecraft.world.entity.Entity#getAddEntityPacket` now takes in a `ServerEntity`
- `net.minecraft.world.entity.Mob` leash methods have been moved to `Leashable`
- `net.minecraft.world.entity.Saddleable#equipSaddle` now takes in the `ItemStack` being equipped
- `net.minecraft.world.entity.ai.village.poi.PoiManager` now takes in a `ChunkIOErrorReporter`
- `net.minecraft.world.level.chunk.storage.ChunkSerializer#read` now takes in a `RegionStorageInfo`
- `net.minecraft.world.level.chunk.storage.SectionStorage` now takes in a `ChunkIOErrorReporter`
- `net.minecraft.world.level.levelgen.structure.PoolElementStructurePiece` now takes in `LiquidSettings`
- `net.minecraft.world.level.levelgen.structure.pools.JigsawPlacement#addPieces` now takes in `LiquidSettings`
- `net.minecraft.world.level.levelgen.structure.pools.SinglePoolElement` now takes in an `Optional<LiquidSettings>`
    - `getSettings` now takes in `LiquidSettings`
- `net.minecraft.world.level.levelgen.structure.pools.StructurePoolElement` now takes in an `LiquidSettings`
    - `single` also has overloads that takes in `LiquidSettings`
- `net.minecraft.world.level.levelgen.structure.structures.JigsawStructure` now takes in `LiquidSettings`
- `net.minecraft.world.level.levelgen.structure.templatesystem.StructurePlaceSettings#shouldKeepLiquids` -> `shouldApplyWaterlogging`
- `net.minecraft.world.level.levelgen.structure.templatesystem.StructureTemplateManager#getPathToGeneratedStructure`, `createPathToStructure` -> `createAndValidatePathToGeneratedStructure`
- `net.minecraft.world.level.storage.loot.predicates.LootItemRandomChanceWithEnchantedBonusCondition` now takes in a float representing the chance to unenchant the item
- `net.minecraft.world.phys.shapes.CubePointRange`'s constructor is now public
- `net.minecraft.world.phys.shapes.VoxelShape`'s constructor is now protected
- `net.minecraft.client.gui.components.Checkbox` now takes in a max width which can be passed in through the builder via `Checkbox$Builder#maxWidth`
- `net.minecraft.client.gui.components.MultiLineLabel`
    - `create` methods with `FormattedText` has been removed
    - `create` methods with `List`s now take in a component varargs
    - `createFixed` -> `create`
    - `renderCentered`, `renderLeftAligned` no longer return anything
    - `renderBackgroundCentered` is removed
    - `TextAndWidth` is now a record
- `net.minecraft.commands.arguments.selector.EntitySelector#predicate` -> `contextFreePredicate`
    - Takes in a `List<Predicate<Entity>>` now
- `net.minecraft.world.level.border.WorldBorder`
    - `isWithinBounds` now has overloads for `Vec3` and four doubles representing two xz coordinates
    - `clampToBounds` now has overloads for `BlockPos` and `Vec3`
    - `clampToBounds(AABB)` is removed
- `net.minecraft.server.level.ServerPlayer#onInsideBlock` is now public, only for this overload

### Removed

- `net.minecraft.client.Minecraft`
    - `getSearchTree`, `populateSearchTree`
    - `renderOnThread`
- `net.minecraft.client.searchtree.SearchRegistry`
- `net.minecraft.entity.LivingEntity#canSpawnSoulSpeedParticle`, `spawnSoulSpeedParticle`
- `net.minecraft.world.entity.projectile.AbstractArrow`
  - `setKnockback`, `getKnockback`
  - `setShotFromCrossbow`
- `net.minecraft.world.item.ProjectileWeaponItem#hasInfiniteArrows`
- `com.mojang.blaze3d.systems.RenderSystem`
    - `initGameThread`, `isOnGameThread`
    - `assertInInitPhase`, `isInInitPhase`
    - `assertOnGameThreadOrInit`, `assertOnGameThread`
- `net.minecraft.client.gui.screens.Screen#advancePanoramaTime`
- `net.minecraft.world.entity.EquipmentSlot#byTypeAndIndex`
- `net.minecraft.world.entity.Mob#canWearBodyArmor`
    - Use `canUseSlot(EquipmentSlot.BODY)` instead
- `com.mojang.blaze3d.platform.MemoryTracker`
- `net.minecraft.server.level.ServerLevel#makeObsidianPlatform`
- `net.minecraft.world.entity.Entity#getPassengerClosestTo`
- `net.minecraft.client.resources.model.ModelBakery$ModelGroupKey`
- `net.minecraft.data.worldgen.Structures#structure`
