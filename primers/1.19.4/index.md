# Minecraft 1.19.3 -> 1.19.4 Mod Migration Primer

This is a high level, non-exhaustive overview on how to migrate your mod from 1.19.3 to 1.19.4. This does not look at any specific mod loader, just the changes to the vanilla classes. All provided names use the official mojang mappings.

This primer is licensed under the [Creative Commons Attribution 4.0 International](http://creativecommons.org/licenses/by/4.0/), so feel free to use it as a reference and leave a link so that other readers can consume the primer.

If there's any incorrect or missing information, please file an issue on this repository or ping @ChampionAsh5357 in the Neoforged Discord server.

## Pack Changes

There are a number of user-facing changes that are part of vanilla which are not discussed below that may be relevant to modders. You can find a list of them on [Misode's version changelog](https://misode.github.io/versions/?id=1.19.4&tab=changelog).

## DamgeType and DamageSources

Damage sources have been rewritten to define common data which should be static between different sources. As such, each `DamageSource` now takes in a `DamageType` which defines this information. `DamageType`s are created in json like any other datapack registry.

The main `DamageType` record constructor takes in the following arguments:

Parameter        | Type               | Decsription
:---:            | :---:              | :---
msgId            | `String`           | A string used within a localization key (typically preceded by `death.attack.`).
scaling          | `DamageScaling`    | When the damage caused by the type should scale according to difficulty.
exhaustion       | `float`            | How much food exhaustion should occur when the damage source is applied to the player.
effects          | `DamageEffects`    | The sound to be played when the type is applied to an entity.
deathMessageType | `DeathMessageType` | How the death message should be structured for the type. Should almost always be `DEFAULT`.

A `DamageSource` is now treated as final class (it's not actually final, so it can still be extended if needed) which takes in some combination of the following parameters:

Parameter            | Type                 | Decsription
:---:                | :---:                | :---
type                 | `Holder<DamageType>` | The damage type, typically obtained from the `RegistryAccess`.
causingEntity        | `Entity`             | The entity who caused the damage directly or through the `#directEntity`.
directEntity         | `Entity`             | The entity who directly inflicted the damage.
damageSourcePosition | `Vec3`               | The position the damage source took place.

The damage sources are now constructed in `DamageSources` which can be obtained from the `Level` via `#damageSources`. Every `Entity` also has a redirect method to the level via `#damageSources`.

### DamageSourcePredicate

The `DamageSourcePredicate` for criteria triggers has been rewritten to take in a list of `TagPredicate`s for the `DamageType` applied along with the direct and causing (source) entity.

> `TagPredicate`s check if the specific object is either within or not within a tag (triggered by the `expected` boolean flag).

## GUI Changes

There have been a number of GUI changes which expands upon existing methods, restructures components, and reorganizes their locations.

### GuiComponent

All `GuiComponent` methods are now static. Additionally, new methods were added to draw information to the screen: `#renderOutline` and `#blitNineSliced`. `#setBlitOffset` and `#getBlitOffset` have been removed and are now performed using the `PoseStack` by translating the z-axis.

### ScreenRectangle

A new class called `ScreenRectangle` to specify a rectangle. This is typically used for scissors and layouts; however, it could also be used in standard rendering.

### ComponentPath

Logic relating to how components are focused and executed are now handled through `ComponentPath`s which now handle the responsibility of a for loop on a list.

### More PoseStack additions

The `PoseStack` has been added to a number of methods to properly transform the drawn information onto the screen space.

### Widget Restructing

A number of widgets have been restructured or moved into different classes to handle some common logic.

The following components have been added:
* `net.minecraft.client.gui.components.TabButton` and by extension `net.minecraft.client.gui.components.tabs.TabNavigationBar`
* `net.minecraft.client.gui.components.AbstractStringWidget` and by extension `net.minecraft.client.gui.components.MultiLineTextWidget`
* `net.minecraft.client.gui.components.ImageWidget`
* `net.minecraft.client.gui.components.TextAndImageButton`

In addition, the following components were moved, renamed, or modified:
* `net.minecraft.client.gui.components.CenteredStringWidget` -> `net.minecraft.client.gui.components.StringWidget`
* `net.minecraft.client.gui.components.AbstractWidget#renderButton` -> `#renderWidget`
* `net.minecraft.client.gui.components.AbstractWidget#getYImage` reworked into `AbstractButton#getTextureY` where it returns the texture coordinate rather than the texture index

#### Layout Components

Layout components, such as the `FrameWidget`, within `net.minecraft.client.gui.components` have been moved to `net.minecraft.client.gui.layouts`. Additionally, they have been restructure to consume widgets to properly move them to where they need to be displayed rather than having each layout be its own widget. As such, `AbstractContainerWidget` was removed.

### Font DisplayMode

`Font#drawInBatch` and any subsequent delegates (e.g. `FontRenderer#drawInBatch`) now take in a `Font$DisplayMode` instead of a boolean to determine how the font should be displayed. 

## Entity Interfaces

Entity logic has been abstracted even more into three new interfaces: `Attackable`, `Targeting`, and `TraceableEntity`. The `Attackable` interface indicates the entity can be attacked and stores the last attacker via `#getLastAttacker`. The `Targeting` interface indicates the entity can target another entity and stores the value via `#getTarget`. Finally, the `TraceableEntity` interface means that the entity's creation and initial action can be traced back to another entity, which can be obtained via `#getOwner`.

> `TraceableEntity` should not be confused with `OwnableEntity`. Traceables are typically used for projectiles or entities fired from or triggered by another entity while ownables are typically for tamed animals.

## Blocks and Sounds

Blocks have a few changes regarding their internal implementation and data structures.

The `BlockBehaviour$OffsetType` passed into `BlockBehavior$Properties#offsetType` now redirects to `BlockBehavior$OffsetFunction` which takes in a `BlockState`, `BlockGetter`, and `BlockPos` and returns the amount to offset the model by. This is currently not exposed to the end user.

A new record has been made to store common properties associated with a type, aptly named `BlockSetType`. This takes in the `SoundType` for the block in addition to sound events for flicking on or off a door, trapdoor, pressure plate, and button. Following this trend, `WoodType`s now take in the `BlockSetType` in addition to `SoundType`s for the wood and hanging sign along with `SoundEvent`s for the fence gate. `WoodType`s are also registered via the `#register` method instead of `#create`:

```java
public static final WoodType TEST_WOOD_TYPE = WoodType.register(new WoodType(new ResourceLocation(MODID, "test").toString(), BlockSetType.ACACIA));
```

> `BlockSetType`s can be created and registered using the constructor and `BlockSetType#register`, repsectively, after the `SoundEvent` registry event has fired but before the `Block` registry event.

## Creative Tabs

Creative Tabs have slightly changed as now when populating the generator, the method provides `CreativeModeTab$ItemDisplayParameters` and an `CreativeModeTab$Output`. The parameters holds the list of enabled feature, whether the player has permission, and a lookup provider on all the registries.

## Removal of `com.mojang.bridge.*`

Mojang's `javabridge` library has been removed as a dependency from Minecraft. As such, all of the redirected counterparts to the bridge (such as `com.mojang.bridge.game.PackType` and `com.mojang.bridge.game.GameVersion`) are now handled by their references or implementations (`net.minecraft.server.packs.PackType` and `net.minecraft.WorldVersion` respectively).

## Minor Additions, Changes, and Removals

The following is a list of useful or interesting additions, changes, and removals that do not deserve their own section in the primer.

### New Registries

In addition to `DamageType`s, a static registry for decorated pot patterns and datapack registries for trim materials, trim patterns, and `MultiNoiseBiomeSourceParameterList` have been added.

### New Codecs

A number of new codecs have been added or changed:
  * `com.mojang.math.Transformation#CODEC`
  * `net.minecraft.util.ExtraCodecs#QUATERNIONF_COMPONENTS`
  * `net.minecraft.util.ExtraCodecs#AXISANGLE4F`
  * `net.minecraft.util.ExtraCodecs#MATRIX4F`

### Sprite Registration Refactoring

First, `net.minecraft.client.particle.ParticleProvider$Sprite` is added, which is a functional interface used to create a `TextureSheetParticle` with only one texture. This is used as a wrapper around a `ParticleEngine$SpriteParticleRegistration` when the particle does not need access to the `SpriteSet`.

### Additions

* `GlintAlpha` - A new GLSL shader uniform which changes the glint strength based on the accessibility option of a similar name.
* `net.minecraft.advancements.Advancement#getRoot` - Gets the root of an advancement.
* `net.minecraft.client.model.geom.builders.CubeListBuilder#addBox` can now take in a set of directions which indicates the visible faces to render of a given box. This is baked into the `net.minecraft.client.model.geom.ModelPart$Cube` when adding the polygons.
* `net.minecraft.client.model.AgeableHierarchicalModel` - A parallel to `net.minecraft.client.modelAgeableListModel`s for `HierarchialModel`s.
* `net.minecraft.client.model.HumanoidArmorModel` - An extension of `net.minecraft.client.model.HumanoidModel` for armor models.
* `com.mojang.blaze3d.vertex.PoseStack#rotateAround` - Rotates a quaternion around a point.
* `net.minecraft.commands.arguments.HeightmapTypeArgument` - A common argument for specifying which heightmap to use.
* `net.minecraft.gametest.framework.GameTestHelper#continuouslyUse` - A helper test method to force a player to use an item on a certain block position every tick.
* `net.minecraft.util.ParticleUtils#spawnParticleBelow` - Spawns a particle half a block below the specified position
* `net.minecraft.world.inventory.Slot#setByPlayer` - A method indicating that a player changed the slot in some capacity.
* `net.minecraft.world.effect.MobEffectInstance#endsWithin` - Returns whether the mob effect will expire in x ticks.
* `net.minecraft.world.entity.Entity#getControlledVehicle` - Returns the vehicle if it is being controlled by this entity.
* `net.minecraft.world.phys.AABB#distanceToSqr` - Gets the squared distance to the provided vector.
* `net.minecraft.world.phys.Vec3#atLowerCornerWithOffset` - Offsets the vector by the specified x, y, z coordinates.
* `net.minecraft.world.phys.Vec3#atCenterOf` - Offsets the vector by 0.5 in all directions.
* `net.minecraft.data.advancements.AdvancementSubProvider#createPlaceholder` - Creates a dummy advancement to use as a parent for another advancement.
* `net.minecraft.network.FriendlyByteBuf#readJsonWithCodec` and #writeJsonWithCodec` - Writes a encoded object to json into a buffer.
    * `Tag` implementations `#readWithCodec` and `#writeWithCodec` are currently deprecated.
* `net.minecraft.client.renderer.texture.Dumpable` - Dumps the contents of the object to a file (`DynamicTexture`, `TextureAtlas`).
* `net.minecraft.world.entity.LivingEntity#remove` - Removes the entity from the level for the specified reason.
* `net.minecraft.world.entity.ai.Brain#clearMemories` - Clears all the memory values of an entity's brain.
* `com.mojang.math.MatrixUtil` contains some additional [jacobi matrix](https://en.wikipedia.org/wiki/Jacobian_matrix_and_determinant) methods.
* `net.minecraft.world.level.GrassColor#getDefaultColor` - Gets the default color of grass.
* `net.minecraft.network.syncher.SynchedEntityData#set(EntityDataAccessor, T, boolean)` - Boolean parameter added to force a sync of the entity data.

### Changes

* `net.minecraft.world.level.biome.Biome#isHumid` -> `#hasPrecipitation`
* `net.minecraft.world.level.biome.Biome#getPrecipitation()` -> `#getPrecipitationAt(BlockPos)`
* `net.minecraft.world.entity.LivingEntity#animationSpeedOld`, `#animationSpeed`, and `#animationPosition` has been bundled into one public object on the entity known as `WalkAnimationState`.
* `net.minecraft.server.packs.repository.PackRepository#addPack` and `#removePack` now return a boolean on whether the action was successful.
* `net.minecraft.client.gui.screens.worldselection.WorldGenSettingsComponent` -> `WorldCreationUiState`
* `net.minecraft.client.renderer.block.model.ItemTransforms$TransformType` -> `net.minecraft.world.item.ItemDisplayContext`
* `net.minecraft.client.resources.metadata.language.LanguageMetadataSectionSerializer` -> `LanguageMetadataSection#CODEC`
* `net.minecraft.world.item.crafting.UpgradeRecipe` -> `LegacyUpgradeRecipe`
    * Deprecated for removal; replaced by `SmithingRecipe`, `SmithingTransformRecipe`, and `SmithingTrimRecipe`
* `net.minecraft.data.recipes.UpgradeRecipeBuilder` -> `LegacyUpgradeRecipeBuilder`
    * Deprecated for removal; replaced by `SmithingTransformRecipeBuilder` and `SmithingTrimRecipeBuilder`
* `net.minecraft.data.worldgen.biome.Biomes` -> `BiomeData`
* `net.minecraft.world.item.Wearable` -> `Equipable`
* `net.minecraft.world.item.crafting.Recipe#assemble(C)` and `#getResultItem()` -> `#assemble(C, RegistryAccess)` and `#getResultItem#(RegistryAccess)`
* `net.minecraft.world.entity.Entity#rideableUnderWater` -> `#dismountsUnderwater`
* `net.minecraft.world.entity.PlayerRideableJumping#canJump(Player)` -> `#canJump()`
* `net.minecraft.data.tags.TagsProvider` now can take in a `CompletableFuture<TagsProvider.TagLookup<T>>` if tags need to be accessed from other `TagProvider`s. A lookup can be obtained via `TagsProvider#contentsGetter`.
* `net.minecraft.world.item.ArmorItem` now take in a delegate to the `EquipmentSlot` called `ArmorItem$Type` within their constructors and associated references.
* `net.minecraft.core.BlockPos` constructors that take in doubles should migrate to using `BlockPos#containing`.
* `net.minecraft.world.level.biome.MobSpawnSettings$SpawnerData#minCount` and `#maxCount` must be positive values.

### Removals

* `net.minecraft.world.level.biome.Biome#hasDownfall` as it was only used by the `#isHumid` check, now `#hasPrecipitation`.
* `com.mojang.blaze3d.systems.RenderSystem#enableTexture` and `#disableTexture` as they have not been doing anything other than adding extra cycles to the logic execution.
