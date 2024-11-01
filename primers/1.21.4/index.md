# Minecraft 1.21.2/3 -> 1.21.4 Mod Migration Primer

This is a high level, non-exhaustive overview on how to migrate your mod from 1.21.2/3 to 1.21.4. This does not look at any specific mod loader, just the changes to the vanilla classes.

This primer is licensed under the [Creative Commons Attribution 4.0 International](http://creativecommons.org/licenses/by/4.0/), so feel free to use it as a reference and leave a link so that other readers can consume the primer.

If there's any incorrect or missing information, please file an issue on this repository or ping @ChampionAsh5357 in the Neoforged Discord server.

## Pack Changes

There are a number of user-facing changes that are part of vanilla which are not discussed below that may be relevant to modders. You can find a list of them on [Misode's version changelog](https://misode.github.io/versions/?id=24w44a&tab=changelog).

## Particles, rendered through Render Types

Particles are now rendered using a `RenderType`, rather than setting a buffer builder themselves. The only special cases are `ParticleRenderType#CUSTOM`, which allows the modder to implement their own rendering via `Particle#renderCustom`; and `ParticleRenderType#NO_RENDER`, which renders nothing.

To create a new `ParticleRenderType`, it can be created by passing in its name for logging and the `RenderType` to use. Then, the type is returned in `Particle#getRenderType`.

```java
public static final ParticleRenderType TERRAIN_SHEET_OPAQUE = new ParticleRenderType(
    "TERRAIN_SHEET_OPAQUE", // Typically something recognizable, like the name of the field
    RenderType.opaqueParticle(TextureAtlas.LOCATION_BLOCKS) // The Render Type to use
);
```

- `net.minecraft.client.particle`
    - `CherryParticle` -> `FallingLeavesParticle`, not one-to-one as the new class has greater configuration for its generalization
    - `ItemPickupParticle` no longer takes in the `RenderBuffers`
    - `Particle#renderCustom` - Renders particles with the `ParticleRenderType#CUSTOM` render type.
    - `ParticleEngine#render(LightTexture, Camera, float)` -> `render(Camera, float, MutliBufferSource$BufferSource)`
    - `ParticleRenderType` is now a record which takes in the name and the `RenderType` it uses.

## Minor Migrations

The following is a list of useful or interesting additions, changes, and removals that do not deserve their own section in the primer.

### Music, now with Volume Controls

The background music is now handled through a `MusicInfo` class, which also stores the volume along with the associated `Music`.

- `net.minecraft.client.Minecraft#getSituationalMusic` now returns a `MusicInfo` instead of a `Music`
- `net.minecraft.client.sounds`
    - `MusicInfo` - A record that holds the currently playing `Music` and the volume its at.
    - `MusicManager#startPlaying` now takes in a `MusicInfo` instead of a `Music`
    - `SoundEngine#setVolume`, `SoundManager#setVolume` - Sets the volume of the associated sound instance.
- `net.minecraft.world.level.biome`
    - `Biome`
        - `getBackgroundMusic` now returns a optional `SimpleWeightedRandomList` of music.
        - `getBackgroundMusicVolume` - Gets the volume of the background music.
    - `BiomeSpecialEffects$Builder#silenceAllBackgroundMusic`, `backgroundMusic(SimpleWeightedRandomList<Music>)` - Handles setting the background music for the biome.

### List of Additions

- `com.mojang.blaze3d.vertex.VertexBuffer`
    - `uploadStatic` - Immediately uploads the provided vertex data via the `Consumer<VertexConsumer>` using the `Tesselator` with a `STATIC_WRITE` `VertexBuffer`.
    - `drawWithRenderType` - Draws the current buffer to the screen with the given `RenderType`.
- `com.mojang.math.MatrixUtil#isIdentity` - Checks whether the current `Matrix4f` is an identity matrix.
- `net.minecraft.client.gui.navigation.ScreenRectangle#transformAxisAligned` - Creates a new `ScreenRectangle` by transforming the position using the provided `Matrix4f`.
- `net.minecraft.client.model`
    - `BannerFlagModel`, `BannerModel` - Models for the banner and hanging banner.
    - `VillagerLikeModel#translateToArms` - Translates the pose stack such that the current relative position is at the entity's arms.
- `net.minecraft.client.multiplayer.PlayerInfo#setShowHat`, `showHat` - Handles showing the hat layer of the player in the tab overlay.
- `net.minecraft.client.renderer.blockentity.HangingSignRenderer`
    - `$AttachmentType` - An enum which represents where the model is attached to, given its properies.
    - `$ModelKey` - A key for the model that combines the `WoodType` with its `$AttachmentType`.
- `net.minecraft.client.renderer.entity.EntityRenderer#getShadowStrength` - Returns the raw opacity of the display's shadow.
- `net.minecraft.client.renderer.entity.layers.CrossedArmsItemLayer#applyTranslation` - Applies the translation to render the item in the model's arms.
- `net.minecraft.core.BlockPos$TraversalNodeStatus` - A marker indicating whether the `BlockPos` should be used, skipped, or stopped from any further traversal.
- `net.minecraft.data.loot.BlockLootSubProvider#createMultifaceBlockDrops` - Drops a block depending on the block face mined.
- `net.minecraft.data.worldgen.placement.PlacementUtils#HEIGHTMAP_NO_LEAVES` - Creates a y placement using the `Heightmap$Types#MOTION_BLOCKING_NO_LEAVES` heightmap.
- `net.minecraft.network.chat.Style#getShadowColor`, `withShadowColor` - Methods for handling the shadow color of a component.
- `net.minecraft.util.profiling.jfr.JvmProfiler#onStructureGenerate` - Returns the profiled duration on when a structure attempts to generate in the world.
- `net.minecraft.util.profiling.jfr.event.StructureGenerationEvent` - A profiler event when a structure is being generated.
- `net.minecraft.util.profiling.jfr.stats.StructureGenStat` - A result of a profiled structure generation.
- `net.minecraft.world.entity.ai.attributes.AttributeMap#resetBaseValue` - Resets the attribute instance to its default value.
- `net.minecraft.world.entity.monster.creaking`
    - `Creaking#activate`, `deactivate` - Handles the activateion of the brain logic for the creaking.
    - `CreakingTransient`
        - `creakingDeathEffects` - Handles the death of a creaking.
        - `playerIsStuckInYou` - Checks whether there are at least four players stuck in a creaking.
        - `setTearingDown`, `isTearingDown` - Handles the tearing down state.
        - `hasGlowingEyes`, `checkEyeBlink` - Handles the eye state.
- `net.minecraft.world.item`
    - `Item$TooltipContext#permissionLevel` - Returns the permission level of the player looking at the item.
    - `ItemStack#getCustomName` - Returns the custom name of the item, or `null` if no component exists.
- `net.minecraft.world.item.component.CustomData#parseEntityId` - Reads the entity id off of the component.
- `net.minecraft.world.item.trading.Merchant#stillValid` - Checks whether the merchant can still be accessed by the player.
- `net.minecraft.world.level`
    - `Level#dragonParts` - Returns the list of entities that are the parts of the ender dragon.
    - `ServerExplosion#getDamageSource` - Returns the damage source of the explosion.
- `net.minecraft.world.level.block`
    - `FlowerBlock#getBeeInteractionEffect` - Returns the effect that bees obtain when interacting with the flower.
    - `FlowerPotBlock#opposite` - Returns the opposite state of the block, only for potted eyeblossoms.
    - `MultifaceBlock#canAttachTo` - Returns whether this block can attach to another block.
    - `MultifaceSpreadeableBlock` - A multiface block that can naturally spread.

### List of Changes

- `net.minecraft.client.gui`
    - `Gui#clear` -> `clearTitles`
    - `GuiGraphics#drawWordWrap` has a new overload that takes in whether a drop shadow should be applied to the text
        - The default version enables drop shadows instead of disabling it
- `net.minecraft.client.gui.components.toasts.TutorialToast` now requires a `Font` as the first argument in its constructor
- `net.minecraft.client.gui.font.glyphs.BakedGlyph$Effect` and `$GlyphInstance` now take in the color and offset of the text shadow
- `net.minecraft.client.model`
    - `DonkeyModel#createBodyLayer`, `createBabyLayer` now take in a scaling factor
    - `VillagerHeadModel` -> `VillagerLikeModel`
- `net.minecraft.client.multiplayer.MultiPlayerGameMode#handlePickItem` -> `handlePickItemFromBlock` or `handlePickItemFromEntity`, providing both the actual object data to sync and a `boolean` about whether to include the data of the object being picked
- `net.minecraft.client.particle.CherryParticle` -> `FallingLeavesParticle`, not one-to-one as the new class has greater configuration for its generalization
- `net.minecraft.client.player.ClientInput#tick` no longer takes in any parameters
- `net.minecraft.client.renderer`
    - `LevelRenderer`
        - `renderLevel` no longer takes in the `LightTexture`
        - `onChunkLoaded` -> `onChunkReadyToRender`
    - `PostChainConfig$Pass#program` -> `programId`
        - `program` now returns the `ShaderProgram` with the given `programId`
    - `ScreenEffectRenderer#renderScreenEffect` now takes in a `MultiBufferSource`
    - `SectionOcclusionGraph#onChunkLoaded` -> `onChunkReadyToRender`
    - `SkyRenderer`
        - `renderSunMoonAndStars`, `renderSunriseAndSunset` now takes in a `MultiBufferSource$BufferSource` instead of a `Tesselator`
        - `renderEndSky` no longer takes in the `PoseStack`
    - `WeatherEffectRenderer#render` now takes in a `MultiBufferSource$BufferSource` instead of a `LightTexture`
- `net.minecraft.client.renderer.blockentity`
    - `BannerRenderer#createBodyLayer` -> `BannerModel#createBodyLayer`, not one-to-one
    - `HangingSignRenderer`
        - `createHangingSignLayer` now takes in a `HangingSignRenderer$AttachmentType`
        - `$HangingSignModel` is now replaced with a `Model$Simple`, though its fields can be obtained from the root
- `net.minecraft.client.renderer.entity.AbstractHorseRenderer`, `DonkeyRenderer` no longer takes in a float scale
- `net.minecraft.client.renderer.entity.layers.CrossedArmsItemLayer` now requires the generic `M` to be a `VillagerLikeModel`
- `net.minecraft.client.renderer.entity.state.CreakingRenderState#isActive` -> `eyesGlowing`
    - The original parameter still exists on the `Creaking`, but is not necessary for rendering
- `net.minecraft.core.BlockPos#breadthFirstTraversal` now takes in a function that returns a `$TraversalNodeStatus` instead of a simple predicate to allow certain positions to be skipped
- `net.minecraft.core.particles.TargetColorParticleOption` -> `TrailParticleOption`, not one-to-one
- `net.minecraft.network.protocol.game`
    - `ClientboundLevelParticlesPacket` now takes in a boolean that determines whether the particle should always render
    - `ClientboundPlayerInfoUpdatePacket$Entry` now takes in a boolean representing whether the hat should be shown
    - `ClientboundSetHeldSlotPacket` is now a record
    - `ServerboundPickItemPacket` -> `ServerboundPickItemFromBlockPacket`, `ServerboundPickItemFromEntityPacket`; not one-to-one
- `net.minecraft.server.level.ServerLevel#sendParticles` now has an overload that takes in the override limiter distance and whether the particle should always be shown
    - Other overloads that take in the override limiter now also take in the boolean for if the particle should always be shown
- `net.minecraft.util`
    - `ARGB#from8BitChannel` is now private, with individual float components obtained from `alphaFloat`, `redFloat`, `greenFloat`, and `blueFloat`
    - `SpawnUtil#trySpawnMob` now takes in a boolean that, when false, allows the entity to spawn regardless of collision status with the surrounding area
- `net.minecraft.util.profiling.jfr.callback.ProfiledDuration#finish` now takes in a boolean that indicates whether the profiled event was successful
- `net.minecraft.util.profiling.jfr.parse.JfrStatsResults` now takes in a list of structure generation statistics
- `net.minecraft.world.effect.PoisonMobEffect`, `WitherMobEffect` is now public
- `net.minecraft.world.entity.LivingEntity`
    - `isLookingAtMe` no longer takes in a `Predicate<LivingEntity>`, and array of `DoubleSupplier`s is now an array of `double`s
    - `hasLineOfSight` takes in a double instead of a `DoubleSupplier`
- `net.minecraft.world.entity.monster.creaking.CreakingTransient#tearDown` no longer takes in a `DamageSource`
- `net.minecraft.world.entity.player`
    - `Inventory#setPickedItem` -> `addAndPickItem`
    - `Player#getPermissionLevel` is now public
- `net.minecraft.world.inventory.Slot#getNoItemIcon` now returns a single `ResourceLocation` rather than a pair of them
- `net.minecraft.world.item.Item$TooltipContext#of` now takes in the `Player` viewing the item
- `net.minecraft.world.level.Level#addParticle` now takes in a boolean representing if the particle should always be shown
- `net.minecraft.world.level.block`
    - `Block#getCloneItemStack` -> `state.BlockBehaviour#getCloneItemStack`, now protected
    - `CherryLeavesBlock` -> `ParticleLeavesBlock`
    - `CreakingHeartBlock#canSummonCreaking` -> `isNaturalNight`
    - `MultifaceBlock` is no longer abstract
        - `getSpreader` -> `MultifaceSpreadeableBlock#getSpreader`
    - `SculkVeinBlock` is now an instance of `MultifaceSpreadeableBlock`
    - `SnowyDirtBlock#isSnowySetting` is now protected
- `net.minecraft.world.level.chunk.ChunkGenerator#createStructures` now takes in the `Level` resource key, only used for profiling
- `net.minecraft.world.level.levelgen.feature.configurations`
    - `MultifaceGrowthConfiguration` now takes in a `MultifaceSpreadableBlock` instead of a `MultifaceBlock`
    - `SimpleBlockConfiguration` now takes in a boolean on whether to schedule a tick update
- `net.minecraft.world.level.levelgen.structure.Structure#generate` now takes in the `Structure` holder and a `Level` resource key, only used for profiling

### List of Removals

- `com.mojang.blaze3d.systems.RenderSystem#overlayBlendFunc`
- `net.minecraft.client.model`
    - `FelineModel#CAT_TRANSFORMER`
    - `VillagerModel#BABY_TRANSFORMER`
- `net.minecraft.server.level.TicketType#POST_TELEPORT`
- `net.minecraft.world.level.block.CreakingHeartBlock$CreakingHeartState`
- `net.minecraft.world.level.block.entity.BlockEntity#onlyOpCanSetNbt`
