# Minecraft 1.20.4 -> 1.20.5 Mod Migration Primer

This is a high level, non-exhaustive overview on how to migrate your mod from 1.20.4 to 1.20.5. This does not look at any specific mod loader, just the changes to the vanilla classes.

This primer is licensed under the [Creative Commons Attribution 4.0 International](http://creativecommons.org/licenses/by/4.0/), so feel free to use it as a reference and leave a link so that other readers can consume the primer.

If there's any incorrect or missing information, please leave a comment below. Thanks!

## Pack Changes

There are a number of user-facing changes that are part of vanilla which are not discussed below that may be relevant to modders. You can find a list of them on [Misode's version changelog](https://misode.github.io/versions/?id=1.20.5&tab=changelog).

## Java 21

Minecraft now uses Java 21. You can download a copy of the JDK used here: https://learn.microsoft.com/en-us/java/openjdk/download#openjdk-21

Windows machines can install this using `winget` in PowerShell:

```pwsh
winget install Microsoft.OpenJDK.21
```

## Data Components

Data components (held within a `DataComponentMap`) are a replacement to the `CompoundTag` within an `ItemStack`. Each data component represents a key-value pair where the key is a string and the value is an object holding the data. Data components can be written to disk or synchronized across the network using codecs. All of vanilla's data components are stored within `DataComponents`, where arbitrary data that hasn't been converted to a data component is held within the `minecraft:custom_data` key.

The storage of data components has an analog to Maps, where `DataComponentMap` represents a read-only map and its implementation `PatchedDataComponentMap` represents an identity map. The data component values themselves should always be treated as **immutable** objects as both the hash code and equality are determined based on both the keys and values.

To hold data components, an object must implement `DataComponentHolder`. Currently this is only implement by `ItemStack`. The holder itself does not have any write based methods. `ItemStack`, on the other hand, does have methods to `set`, `update`, and `remove` attached components. Any operations that wish to modify a component's value should call `set` or `update`, making sure that the component value is **immutable** or at least not shallowly referenced somewhere else. Each method takes in a `DataComponentType` which represents the key of the value and the type of object being stored.

```java
// For some ItemStack 'stack' that wants a custom name
stack.set(DataComponents.CUSTOM_NAME, Component.literal("Hello world!"));

// To get the custom name
Component name = stack.get(DataComponents.CUSTOM_NAME);

// To update the stored value of the component
stack.update(DataComponents.CUSTOM_NAME, Component.literal("Default if not set"), component ->
  component.withStyle(ChatFormatting.BLUE)
);

// To remove the component
stack.remove(DataComponents.CUSTOM_NAME);
```

To create a custom data component, you must register a `DataComponentType`. These can be constructed via `DataComponentType$Builder`. The builder has two methods: `persistent(Codec)` for if you want the data component to be written to disk, and `networkSynchronized(StreamCodec)` for if you want the data component to be synchronized to the client. `StreamCodec`s are explained in more detail in the next section. The component type can also cache the currently encoded value if you are writing to disk the same value often via `cacheEncoding`. 

As an additional utility, `Item`s can have a default set of data components via `Item$Properties#component`. These values can then be updated for each individual stack.

Any methods regarding items that took in a `CompoundTag` are will now take in the direct object type. The list of available data component objects can be found within `net.minecraft.world.item.component`.

## Stream Codecs

When communicating across a network, previously, all logic needed to be manually written and read to a `FriendlyByteBuf`. However, most of this logic has now been replaced with `StreamCodec`s. A `StreamCodec` is a type of codec (which doesn't implement `Codec`) that holds a function to apply stream-based operations to encode and decode a given object. A `StreamCodec` is made up of two parts: the `StreamEncoder` and `StreamDecoder`.

The `StreamEncoder` takes in two generics, `O` representing the output to write data to, and `T` representing the data object. The `StreamDecoder` also takes in two generics, `I` representing the input to read data from, and `T` representing the constructed data object.

To create a `StreamCodec`, you will usually use one of the `composite` functions. The `composite` function takes in up to 6 pairs of `StreamCodec` containing the type to serialize and `Function`s representing the getter for the data field. The final parameter represents the constructor for the object. There are a set of default stream codecs located in `ByteBufCodecs`.

```java
// A basic stream codec for a record ExampleObject(int, Optional<String>, Holder<Item>)

// Using RegistryFriendlyByteBuf as a registry object is requested
// Otherwise, FriendlyByteBuf should be used
// If no special methods from FriendlyByteBuf are needed, use ByteBuf
public static final StreamCodec<RegistryFriendlyByteBuf, ExampleObject> STREAM_CODEC = StreamCodec.composite(
  ByteBufCodecs.INT, ExampleObject::intField,
  StreamCodecs.optional(ByteBufCodecs.STRING_UTF8), ExampleObject::optionalStringField,
  ByteBufCodecs.holderRegistry(Registries.ITEM), ExampleObject::holderItemField,
  ExampleObject::new
);
```

As many packets have not been updated to use a codec-based approached, they can use `Packet#codec` to turn a read/write implementation into a codec. This method takes in a `StreamMemberEncoder` instead of a `StreamEncoder` as write methods are usually instance methods on the packet class.

```java
// For some ExamplePacket with ExamplePacket(RegistryFriendlyByteBuf) and ExamplePacket#write(RegistryFriendlyByteBuf)

public static final StreamCodec<RegistryFriendlyByteBuf, ExamplePacket> STREAM_CODEC = Packet.codec(
  ExamplePacket::write, ExamplePacket::new
);
```

If read and write methods have suddently disppeared, or the network handling has been moved, it is most likely handled by a `StreamCodec` directly within the object class or in a package within `net.minecraft.network.codec`. Packets themselves do not contain the `StreamCodec` and instead are directly registered via `ProtocolInfoBuilder#addPacket`. `Packet`s now contain a `PacketType` with the name of the packet and the direction the packet is sent.

Here are some of the classes that uses stream codecs for their implementation logic:

- `net.minecraft.core.particles.ParticleType#getDeserializer` -> `streamCodec`
- `net.minecraft.network.syncher.EntityDataSerializer`
- `net.minecraft.world.item.crafting.RecipeSerializer#fromNetwork`, `toNetwork` -> `streamCodec`
- `net.minecraft.world.level.gameevent.PositionSourceType#read`, `write` -> `streamCodec`

## Sub Predicate Types

Entities and items now have sub predicates that can be applied for specific entities and items. Entities store this within the `type_specific` json object in the `EntityPredicate`. Items store this within the `predicates` json object in the `ItemPredicate`. Each of these are registered to their own built in registry, requiring some codec object.

- `net.minecraft.advancements.critereon.EntityVariantPredicate` -> `EntitySubPredicates$EntityVariantPredicateType`
  - All variant predicates are within `EntitySubPredicates`

## Don't you like Holders?

Most methods now take in a `Holder` of a registry object rather than the registry object directly. As such, direct references should not be used.

- `net.minecraft.client.renderer.FogRenderer$MobEffectFogFunction#getMobEffect` returns a `Holder<MobEffect>`
- `net.minecraft.gametest.framework.GameTestHelper` now takes in a `Holder<MobEffect>`
- `net.minecraft.network.chat.ChatType$Bound` now takes in a `Holder<ChatType>`
- `net.minecraft.world.effect.MobEffect#addAttributeModifier` now takes in a `Holder<Attribute>`
- `net.minecraft.world.effect.MobEffectInstance` now takes in a `Holder<MobEffect>`
- `net.minecraft.world.entity.Entity#gameEvent` now takes in a `Holder<GameEvent>`
- `net.minecraft.world.item.ArmorItem` now takes in a `Holder<ArmorMaterial>`
- `net.minecraft.world.level.LevelAccessor#gameEvent` now takes in a `Holder<GameEvent>`
- `net.minecraft.world.level.gameevent.GameEventListener#handleGameEvent` now takes in a `Holder<GameEvent>`
- `net.minecraft.world.level.gameevent.GameEventListenerRegistry#visitInRangeListeners` now takes in a `Holder<GameEvent>`

## ExtraCodecs to DataFixerUpper

Many of the methods that were commonly used in `ExtraCodecs` has been migrated or combined in some other appropriate place within `Codec` or `DataResult`. Most of the names are synonymous, so they just need to be replaced with the correct class.

- `net.minecraft.Util#getOrThrow`, `getPartialOrThrow` -> `com.mojang.serialization.DataResult#getOrThrow`, `getPartialOrThrow`
- `net.minecraft.util.ExtraCodecs
  - `withAlternative` -> `com.mojang.serialization.Codec#withAlternative`
  - `validate` -> `com.mojang.serialization.Codec#validate`
  - `strictOptionalField` -> `com.mojang.serialization.Codec#optionalFieldOf` as the method is now strict by default
    - `optionalFieldOf` previously has now been replaced by `lenientOptionalFieldOf`
  - `xor` -> `Codec#xor`
  - `either` -> `Codec#either`
  - `stringResolverCodec` -> `Codec#stringResolver`
  - `recursive` -> `Codec#recursive`
  - `lazyInitializedCodec` -> `Codec#lazyInitialized`
  - `sizeLimitedString` -> `Codec#sizeLimitedString`

## Codec Replacements

A lot of read/write methods have been replaced with Codec equivalents.

- `net.minecraft.nbt.NbtUtils#readGameProfile`, `#writeGameProfile` -> `ExtraCodecs#GAME_PROFILE`
- `net.minecraft.network.FriendlyByteBuf`
  - `writeId`, `readById` -> `Registry#getId`, `byId` and `FriendlyByteBuf#writeVarInt`
  - `readCollection`, `writeCollection`, `readList`, `readMap`, `writeMap`, `writeOptional`, `readOptional`, `writeNullable`, `readNullable` take in decoders and encoders for streams
  - `writeEither`, `readEither` -> `Codec#either`
  - `readComponent`, `readComponentTrusted`, `writeComponent` -> `ComponentSerialization#STREAM_CODEC`, `TRUSTED_STREAM_CODEC`
  - `writeItem`, `readItem` -> `ItemStack#OPTIONAL_STREAM_CODEC`
  - `readGameProfile`, `writeGameProfile` ->  `ExtraCodecs#GAME_PROFILE`
  - `readGameProfileProperties`, `writeGameProfileProperties` -> `ExtraCodecs#PROPERTY_MAP`''
  - `readProperty`, `writeProperty` -> `ExtraCodecs#PROPERTY`

## Removed Redundant PoseStacks

Most methods that passed around the raw `PoseStack` has had the parameter removed. Now, no `PoseStack`, or only the relevant `Pose` or `Matrix4f`, is passed.

- `net.minecraft.client.renderer.GameRenderer#renderLevel` no longer takes in a `PoseStack`
- `net.minecraft.client.renderer.LevelRenderer`
  - `prepareCullFrustum` no longer takes in a `PoseStack` and only takes in the `Matrix4f` that is being operated on
  - `renderLevel` no longer takes in a `PoseStack`
  - `renderSectionLayer` no longer takes in a `PoseStack`
  - `renderSky` no longer takes in a `PoseStack` and only takes in the `Matrix4f` that is being operated on

### No more Matrix4f in Lighting Setup

`Matrix4f` is no longer passed into the shader uniforms when setting up level lighting.

- `com.mojang.blaze3d.platform.Lighting#setupForEntityInInventory` - Setup light direction uniforms for entity displayed in inventory
- `net.minecraft.client.particle.ParticleEngine#render(PoseStack, MultiBufferSource$BufferSource, LightTexture, Camera, float)` -> `render(LightTexture, Camera, float)`

## ItemInteractionResult

There is now a separation when it comes to interacting with an item compared interacting with anything else. When interacting where an item is supposed to be called, methods will now return an `ItemInteractionResult`. An `ItemInteractionResult` can be mapped to an `InteractionResult`. The methods below now return an `ItemInteractionResult`.

- `net.minecraft.core.cauldron.CauldronInteraction`
  - `interact`
  - `fillBucket`
  - `emptyBucket`

## Enchantments, now with Definitions

`Enchantment`s have been overhauled to now hold a single record known as an `EnchantmentDefinition`. This record contains tags for the supported items this enchantment can be applied to, the items it would be considered a primary enchantment, the weight of obtaining the enchantment, the max level the enchantment can be, the minimum and maximum cost of the enchantment, the cost to repair in an anvil, the feature flags, and the slots the enchantment can be applied to.

These values can be created using `Enchantment#definition`, while the cost can be computed either via `constantCost` for every level, or `dynamicCost` for an increasing cost per level. Enchantments still need to be registered like any other built-in registry.

For the tags, there are `minecraft:enchantable/*` tags which refer to a list of items that can be applied with that enchant. For example, `minecraft:enchantable/sword` contains all swords. If one of these tags do not match your criteria, you can create a separate enchantable tag for specifically your enchantment. It is recommended to add other items via other enchantable tag groups first. This replaces `EnchantmentCategory`.

`EnchantmentHelper` methods have been replaced to use the `ItemEnchantment`s data component object instead of a raw `CompoundTag`.

- `getRarity` -> `getWeight`
- `getAnvilCost` - The minimum cost needed to repair in an anvil
- `getDamageBonus(int, MobType)` -> `getDamageBonus(int, EntityType<?>)`
- `doPostItemStackHurt` - Executes when an item with this enchantment damages an entity

## Blocks: From public to protected

Many of the methods within `Block` are now protected, to prevent direct access except through the `BlockState`, or has had some changes to its logic. The following methods are now protected:

- `updateIndirectNeighbourShapes`
- `isPathfindable`
- `updateShape`
- `skipRendering`
- `neighborChanged`
- `onPlace`
- `onRemove`
- `onExplosionHit`
- `useWithoutItem`
- `useItemOn`
- `triggerEvent`
- `getRenderShape`
- `useShapeForLightOcclusion`
- `isSignalSource`
- `getFluidState`
- `hasAnalogOutputSignal`
- `getMaxHorizontalOffset`
- `getMaxVerticalOffset`
- `rotate`
- `mirror`
- `canBeReplaced`
- `getDrops`
- `getSeed`
- `getOcclusionShape`
- `getBlockSupportShape`
- `getInteractionShape`
- `getLightBlock`
- `getMenuProvider`
- `canSurvive`
- `getShadeBrightness`
- `getAnalogOutputSignal`
- `getShape`
- `getCollisionShape`
- `isCollisionShapeFullBlock`
- `isOcclusionShapeFullBlock`
- `getVisualShape`
- `randomTick`
- `tick`
- `getDestroyProgress`
- `spawnAfterBreak`
- `attack`
- `getSignal`
- `entityInside`
- `getDirectSignal`
- `onProjectileHit`
- `propagatesSkylightDown`
- `isRandomlyTicking`
- `getSoundType`

Here are some other changes:

- `isPathfindable(BlockState, BlockGetter, BlockPos, PathComputationType)` -> `isPathfindable(BlockState, PathComputationType)`
- `use` -> `useWithoutItem`
- `BlockStateBase$neighborChanged` -> `handleNeighborChanged`
- `net.minecraft.world.level.block.Block`
  - `isRandomlyTicking` -> `BlockBehaviour#isRandomlyTicking`
  - `propagatesSkylightDown` -> `BlockBehaviour#propagatesSkylightDown`
  - `getSoundType` -> `BlockBehaviour#getSoundType`

## ItemStack Max Stack Size 99

The hard limit has been raised from 64 to 99 for `ItemStack`s.

## No more magicalSpecialHackyFocus

`net.minecraft.client.gui.components.events.ContainerEventHandler#magicalSpecialHackyFocus` has been removed. It did nothing but set the focused element.

## Bootstap? No! It's Bootstrap!

`net.minecraft.data.worldgen.BootstapContext` has finally been renamed to `BootstrapContext`!

## Minor Migrations

The following is a list of useful or interesting additions, changes, and removals that do not deserve their own section in the primer.

### StringUtil

Most string transformation utility methods have been moved to `net.minecraft.util.StringUtil` with the same name

- `net.minecraft.Util#isBlank`, `isWhitespace`
- `net.minecraft.SharedConstants#isAllowedChatCharacter`, `filterText`

### Advancement Network Codecs

All network methods for advancements (`read`, `write`) have been made private or removed. They are replaced by `StreamCodec`s taking in a `RegistryFriendlyByteBuf`.

### Collection Criteria Predicates

The following predicates have been added for handling criteria triggers:

- `CollectionContentsPredicate` -> Returns true if all contents of a collection match the given predicate(s)
- `CollectionCountsPredicate` -> Returns true if the number of contents that match the given predicate(s) is within the specified bounds
- `CollectionPredicate` -> Returns true depending on the above predicates and whether the size of the collection matches the size specified

### Entity Slot Criteria

A new field called `slots` on the `EntityPredicate` can now check the entity's inventory for a match.

### Recipe Book Categories throw on default

Recipe book methods that attempt to get a `RecipeBookCategories` will now throw a `MatchException` if no corresponding category exists.

- `net.minecraft.client.ClientRecipeBook#getCategory`
- `net.minecraft.client.RecipeBookCategories#getCategories`

### Gui Breakup

Most logic within the `Gui` class has been separated to private methods. Additionally, some fields have been removed in favor of using their `GuiGraphics` counterparts.

- `renderEffects`, `renderJumpMeter`, `renderExperienceBar`, `renderSelectedItemName`, `renderDemoOverlay` is now private
- `renderSavingIndicator` is now public and takes in a float representing the tick rate
- `screenWidth`, `screenHeight` -> `GuiGraphics#guiWidth`, `GuiGraphics#guiHeight`
- `itemRenderer` has been completely removed

### Map Decoration Textures

Map decoration textures are now managed by an atlas that is stiched together. Because of this, the map id is now stored within a record `MapId` holding the current id of the map to render as a texture. `MapRenderer` uses this `MapId` instead of a integer.

To access the texture manager, call `Minecraft#getMapDecorationTextures`.

- `net.minecraft.client.multiplayer.ClientLevel`
  - `getMapData(String)` -> `getMapData(MapId)`
  - `overrideMapData(String, MapItemSavedData)` -> `overrideMapData(MapId, MapItemSavedData)`
  - `getAllMapData` returns a `Map<MapId, MapItemSavedData>`
  - `addMapData(Map<String, MapItemSavedData>)` -> `addMapData(Map<MapId, MapItemSavedData>)`
- `net.minecraft.world.level.Level`
  - `setMapData(String, MapItemSavedData)` -> `setMapData(MapId, MapItemSavedData)`
  - `getFreeMapId` returns a `MapId`

### Font Options

Font providers now have variant filters. There are currently two available: uniform and japanese variants. These filters determine whether the variant should be used. Most font cosntruction methods allows a filter or a font option to be passed in.

- `com.mojang.blaze3d.font.GlyphProvider$Conditional` - Filters whether a glyph provider should be added to the list of available providers when loading a font option
- `net.minecraft.client.Options#japaneseGlyphVariants` - A new option that, when enabled, will use different variants of Japanese glyphs
- `net.minecraft.client.gui.font.FontManager#updateOptions` - Reloads the available font options to choose from
- `net.minecraft.client.Minecraft#selectMainFont` -> `updateFontOptions`
  - This is not one-to-one as the fonts are now options in the menu
- `net.minecraft.client.gui.font.FontSet#reload` takes in a conditional holding the filters for a glyph provider and the set of font options to reload

### Screen Backgrounds

`Screen#renderBackground` has been broken into three steps: `renderPanorama` if the level is null, `renderBlurredBackground` which processes the blur effect to apply to the background, and `renderMenuBackground` which rends the background texture to the screen. There is also an additional method for rendering a transparent background called `renderTransparentBackground` which is called when rendering the background in `AbstractContainerScreen`.

`renderDirtBackground` -> `renderMenuBackground`

### Holder Lookups in Data Providers

Some methods in data providers now take in a `HolderLookup$Provider` to get any relevant registry objects that are not explicitly accessible or built in.

- `net.minecraft.data.recipes.RecipeProvider` now takes in a `CompletableFuture<HolderLookup.Provider>`
  - `buildAdvancement` now takes in a `HolderLookup$Provider`

### ResourceKey References

Many references to `ResourceLocation`s now take in an associated `ResourceKey` instead with the generic tied to the registry object in question.

- `net.minecraft.data.loot.BlockLootSubProvider` now takes in a `Map<ResourceKey<LootTable>, LootTable.Builder>` instead of a `Map<ResourceLocation, LootTable.Builder>`
- `net.minecraft.data.loot.LootTableProvider(PackOutput, Set<ResourceLocation>, List<LootTableProvider.SubProviderEntry>)` -> `LootTableProvider(PackOutput, Set<ResourceKey<LootTable>>, List<LootTableProvider.SubProviderEntry>, CompletableFuture<HolderLookup.Provider>)
- `net.minecraft.data.loot.LootTableSubProvider#generate(BiConsumer<ResourceLocation, LootTable.Builder>)` -> `generate(HolderLookup.Provider, BiConsumer<ResourceKey<LootTable>, LootTable.Builder>)`

### Static methods in FriendlyByteBuf

Static methods for writing and reading data have been added to `FriendlyByteBuf` which takes in the same parameters of the instance method, along with the `ByteBuf` to write to.

### Loot Registries

Loot data are now handled through dynamic registries, specifically ones that can be reloaded by the `/reload` command. This means that any previous references, such as `LootDataManager`, no longer exist. They can be queried via `reloadableRegistries` in `MinecraftServer` or `Level`

### PackLocationInfo

Pack information stored within resources are now stored within a `PackLocationInfo` object. Parameters such as `name` or `isBuiltIn` is stored directly on this object (e.g. `id` and `knownPackInfo`, respectively). All classes within `net.minecraft.server.packs` have been updated to accomdate this change.

A `KnownPack` is simply a synced key and version indicating where the resource orginated from. As such, both the server and client must have the same contents as the `KnownPack` is used to avoid syncing the entire pack when unnecessary.

### DispatchCodecs now MapCodecs

`Codec#dispatch` now takes in a `MapCodec` instead of a `Codec`. All other codecs used for dispatch have been updated to a `MapCodec`:

- `net.minecraft.core.particles.ParticleType#codec`
- `net.minecraft.client.renderer.texture.atlas.SpriteSources#register`
- `net.minecraft.client.renderer.texture.atlas.SpriteSourceType`
- `net.minecraft.util.valueproviders.FloatProviderType#codec`
- `net.minecraft.util.valueproviders.IntProviderType#codec`
- `net.minecraft.world.item.crafting.RecipeSerializer#codec`
- `net.minecraft.world.level.biome.BiomeSource#codec`
- `net.minecraft.world.level.chunk.ChunkGenerator#codec`
- `net.minecraft.world.level.gameevent.PositionSourceType#codec`
- `net.minecraft.world.level.levelgen.blockpredicates.BlockPredicateType#codec`
- `net.minecraft.world.level.levelgen.carver.WorldCarver#configuredCodec`
- `net.minecraft.world.level.levelgen.feature.Feature#configuredCodec`
- `net.minecraft.world.level.levelgen.feature.featuresize.FeatureSizeType#codec`
- `net.minecraft.world.level.levelgen.feature.foliageplacers.FoliagePlacerType#codec`
- `net.minecraft.world.level.levelgen.feature.rootplacers.RootPlacerType#codec`
- `net.minecraft.world.level.levelgen.feature.stateproviders.BlockStateProviderType#codec`
- `net.minecraft.world.level.levelgen.feature.treedecorators.TreeDecoratorType#codec`
- `net.minecraft.world.level.levelgen.feature.trunkplacers.TrunkPlacerType#codec`
- `net.minecraft.world.level.levelgen.heightproviders.HeightProviderType#codec`
- `net.minecraft.world.level.levelgen.placement.PlacementModifierType#codec`
- `net.minecraft.world.level.levelgen.structure.Structure#simpleCodec`
- `net.minecraft.world.level.levelgen.structure.StructureType#codec`
- `net.minecraft.world.level.levelgen.structure.placement.StructurePlacementType#codec`
- `net.minecraft.world.level.levelgen.structure.pools.StructurePoolElementType#codec`
- `net.minecraft.world.level.levelgen.structure.pools.alias.PoolAliasBinding#codec`
- `net.minecraft.world.level.levelgen.structure.templatesystem.PosRuleTestType#codec`
- `net.minecraft.world.level.levelgen.structure.templatesystem.RuleTestType#codec`
- `net.minecraft.world.level.levelgen.structure.templatesystem.StructureProcessorType#codec`
- `net.minecraft.world.level.levelgen.structure.templatesystem.rule.blockentity.RuleBlockEntityModifierType#codec`
- `net.minecraft.world.level.storage.loot.entries.CompositeEntryBase#createCodec`
- `net.minecraft.world.level.storage.loot.entries.LootPoolEntryType`
- `net.minecraft.world.level.storage.loot.functions.LootItemFunctionType`
- `net.minecraft.world.level.storage.loot.predicates.LootItemConditionType`
- `net.minecraft.world.level.storage.loot.providers.nbt.LootNbtProviderType`
- `net.minecraft.world.level.storage.loot.providers.number.LootNumberProviderType`
- `net.minecraft.world.level.storage.loot.providers.score.LootScoreProviderType`

### Packrat Parser

The [pakrat parser](https://en.wikipedia.org/wiki/Packrat_parser) has been added to `net.minecraft.util.parsing.packrat` to read and parse strings from commands. However, this implementation can be used generically by creating your own grammar.

### Entity Attachments

An `EntityAttachment` is a definition of where a given point lies the entity. For example, a name tag could be directly on top of an entity or underneath it. Each `EntityAttachment` contains a fallback of where the point should go if there is none set. Multiple points can be registered for a given `EntityAttachment`.

`EntityAttachment`s are added from the `EntityType$Builder` using one of the `*Attachments`, `*Offset`, or `attach` methods. They can then be accessed from the entity using `#getAttachments`. To get an attachment vector value, specify the attachment, index, and y rotation, calling `EntityAttachments#getNullable` if you want null to be returned, or `get` if you want an error to be thrown.

### Dyeables

Dyable armor is now a setting on the `ArmorMaterial$Layer` and an item tag `minecraft:dyeable` rather than a completely separate class. When true, this will tint the armor texture provided. The `ItemStack` must have a `DYED_COLOR` data component to read the tint color from. The tag allows this component to be added through the armor dyeing crafting table recipe.

### Potion Brewing

`PotionBrewing` is now an instance class on the `MinecraftServer` that is passed through the `Level`. This means all static methods are now instance methods.

### TooltipProvider

Tooltips for a particular object stored on an item (usually for `DataComponent`s) are now implemented via `TooltipProvider`. This interface has one method which takes in the context of the item tooltip, a consumer to supply the tooltip contents to, and the currently set tooltip flags. Calling this is usualy within `Item#appendHoverText`:

```java
@Override
public void appendHoverText(ItemStack stack, Item.TooltipContext ctx, List<Component> tooltips, TooltipFlag flags) {
  // Get some TooltipProvider implementation provider
  provider.addToTooltip(ctx, tooltips::add, flags);
}
```

### New Criteria Triggers

- `BAD_OMEN` -> `RAID_OMEN`
- `DEFAULT_BLOCK_USE` (`minecraft:default_block_use`)
- `ANY_BLOCK_USE` (`minecraft:any_block_use`)
- `CRAFTER_RECIPE_CRAFTED` (`minecraft:crafter_recipe_crafted`)
- `FALL_AFTER_EXPLOSION` (`minecraft:fall_after_explosion`)

### New Loot Context Parameters

- `EQUIPMENT` -> `ORIGIN`, `THIS_ENTITY`
- `VAULT` -> `ORIGIN`, `THIS_ENTITY`?
- `BLOCK_USE` -> `ORIGIN`, `THIS_ENTITY`, `BLOCK_STATE`
- `SHEARING` -> `ORIGIN`, `THIS_ENTITY`?

### Additions

- `com.mojang.blaze3d.vertex.PoseStack$Pose`
  - `transformNormal` - Applies a transform to the normal within the given pose
  - `copy` - Makes a deep copy of the current pose. Does not add to stack
- `com.mojang.math.MatrixUtil`
  - `isPureTranslation` - Returns true if the matrix has only been translated from the identity
  - `isOrthonormal` - Returns true if the upper left 3x3 submatrix is orthogonal
- `net.minecraft.Util`
  - `toMutableList` - Creates a collector for a mutable list
  - `getRegisteredName` - Gets the registry name of an object, or `[unregistered]` if not registered
  - `allOf` - Combines all predicates into a single predicate using ANDs
  - `copyAndAdd` - Creates a new immuatble list with the added element
  - `copyAndPut` - Creates a new immutable map with the added element
- `net.minecraft.client.GuiMessage#icon` - Returns the icon for the message tag when present, otherwise null
- `net.minecraft.client.Minecraft#disconnect(Screen, boolean)` where the boolean, when true, will not clear downloaded resource packs
- `net.minecraft.client.MouseHandler#handleAccumulatedMovement` - Handles the movement of the mouse when running the tick
- `net.minecraft.client.OptionInstance#createButton(Options)` - Creates a button at (0,0) with a width of 150
- `net.minecraft.client.Options`
  - `menubackgroundBlurriness` - A new option to determine how blurry the background of a menu should be
- `net.minecraft.client.gui.GuiGraphics`
  - `containsPointInScissor` - Returns true if the given point is within the top entry of the scissor stack
  - `fillRenderType` - Creates a filled quad for the given `RenderType`
- `net.minecraft.client.LayeredDraw` - A class that renders objects on top of each other. The distance between each layer is translated 200 units in the Z direction to allow proper stacking
- `net.minecraft.client.gui.components.AbstractSelectionList`
  - `updateSize` - Updates the size of the list to display
  - `updateSizeAndPosition` - Updates the size and position of the list to display
  - `getDefaultScrollbarPosition` - Gets the x position where the scrollbar would normally render
  - `getListOutlinePadding` - Gets the padding to offset the x position of the scrollbar
  - `getRealRowLeft` - Gets the actual left position of the list
  - `getRealRowRight` - Gets the actual right position of the list
- `net.minecraft.client.gui.components.ChatComponent`
  - `storeState` - Returns the current state of the chat messages stored in the component
  - `restoreState` - Sets the current state of the chat messages to a previous setup
- `net.minecraft.client.gui.components.CycleButton#create(Component, CycleButton$OnValueChange)` - Constructs a new button at (0,0) of size (150,20)
- `net.minecraft.client.gui.components.DebugScreenOverlay
  - `showFpsCharts` - When true, renders the FPS chart
  - `logRemoteSample` - Logs a full sample for the specified sample type
- `net.minecraft.client.gui.components.FocusableTextWidget#containWithin` - Sets the max width to contain the specified size excluding padding
- `net.minecraft.client.gui.components.TabButton$renderMenuBackground` - Renders the background of the screen
- `net.minecraft.client.gui.components.debugchart.AbstractDebugChart`
  - `drawDimensions` - Draws the sample and additional information
  - `drawMainDimension` - Draws the sample to the display
  - `drawAdditionalDimensions` - Draws additional information about the sample
  - `getValueForAggregation` - Reads the data within the sample storage
- `net.minecraft.client.gui.components.toasts.SystemToast`
  - `onLowDiskSpace` - Creates a system toast that notifies the disk space needed to write the chunk is low
  - `onChunkLoadFailure` - Creates a system toast that notifies the chunk has failed to load
  - `onChunkSaveFailure` - Creates a system toast that notifies the chunk has failed to save
- `net.minecraft.client.gui.font.FontSet#name` - the name of the font set
- `net.minecraft.client.gui.font.providers.FreeTypeUtil` - A utility for interactions the Freetype Font API
- `net.minecraft.client.gui.layouts.HeaderAndFooterLayout
  - `getContentHeight` - Gets the y position of the content
  - `addTitleHeader` - Adds a string widget to the header
- `net.minecraft.client.gui.navigation.ScreenRectangle#containsPoint` - Checks whether the point is within the current rectangle
- `net.minecraft.client.gui.screens.Screen#setInitialFocus` - This method should be overridden to set the initial focused widget on screen
- `net.minecraft.client.multiplayer.ClientPacketListener`
  - `scoreboard` - Gets the sent scoreboard from the server
  - `potionBrewing` - Gets the sent brewing information from the server
- `net.minecraft.client.multiplayer.KnownPacksManager` - A class that holds the known packs on the client from the server
- `net.minecraft.client.multiplayer.RegistryDataCollector` - A collector which holds the contents and tags of the available registries
- `net.minecraft.client.multiplayer.TagCollector` - A collector which holds the tags of the available registries
- `net.minecraft.client.multiplayer.ServerData`
  - `state` - Gets the current state of the server data
  - `setState` - Sets the current state of the server data
- `net.minecraft.client.particle.Particle$LifetimeAlpha` - A particle utility for handing animated alpha over the particle's lifetime using linear interpolation
- `net.minecraft.client.renderer.GameRenderer`
  - `loadBlurEffect` - Loads the effect of the blur post processor (private)
  - `processBlurEffect` - Processes the blur effect, applies the 'Radius' uniform using the menu blackground blurriness option
  - `getRendertypeCloudsShader` - Gets the shader for the clouds render type
- `net.minecraft.client.renderer.entity.EntityRenderer#getShadowRadius` - Returns the radius of the entity's shadow
- `net.minecraft.client.renderer.entity.layers.WolfArmorLayer` - Render layer for wolf armor
- `net.minecraft.client.sounds.ChunkedSampleByteBuf` - A byte buffer chunked into multiple byte buffers
- `net.minecraft.commands.arguments.ResourceOrIdArgument` - An argument that operates on a resource or registry object identifier
- `net.minecraft.commands.arguments.SlotsArgument` - An argument that operates on a specific slot within a slot range
- `net.minecraft.commands.functions.CommandFunction#checkCommandLineLength` - Commands with over 2 million characters will throw an error
- `net.minecraft.core.BlockBox` - A rectangular prism as defined by two block positions
- `net.minecraft.core.BlockPos`
  - `min` - Takes the minimum coordinates between two block positions and creates a new one
  - `max` - Takes the maximum coordinates between two block positions and createa a new one
- `net.minecraft.core.Direction#getNearest` - Gets the direction closest to the normalized vector
- `net.minecraft.core.Direction$Plane#length` - Gets the number of faces in a given plane
- `net.minecraft.core.Holder`
  - `is(Holder)` - Checks if two holders are equivalent, this method is deprecated
  - `getRegisteredName` - Gets the registry object's identifer
- `net.minecraft.core.HolderGetter$Provider#get` - Gets a refence holder to a registry object from the registry key and resource key
- `net.minecraft.core.HolderLookup#createSerializationContext` - Creates a registry ops for the current registries provider
- `net.minecraft.core.HolderSet#empty` - Gets an empty holder set
- `net.minecraft.core.IdMap#getIdOrThrow` - Gets the id of a registry object or throws an error if not present
- `net.minecraft.core.RegistrationInfo` - Holds information related to a specific registry object in a registry (the pack it came from and the lifecycle of the object)
- `net.minecraft.core.Registry`
  - `getHolder(ResourceLocation)` - Gets the object holder from its registry id
  - `getRandomElementOf` - Gets a random element from the registry filtered by a tag
- `net.minecraft.core.RegistrySynchronization$PackedRegistryEntry` - Holds the data for a given registry entry
- `net.minecraft.data.tags.TagsProvider$TagAppender#addAll` - Adds a list of resource keys to the tag
- `net.minecraft.gametest.framework.GameTest`
  - `skyAccess` - When false, when preparing a test structure, this will spawn barrier blocks on top of the structure
  - `manualOnly` - When true, the test can only be manually triggered and not run as part of all available tests
- `net.minecraft.gametest.framework.GameTestHelper`
  - `spawnItem(Item, Vec3)` - Spawns an item at the specified relative vector
  - `findOneEntity` - Finds the first entity that matches the type
  - `findClosestEntity` - Finds the closest entity to the specifid (x,y,z) within a given radius
  - `findEntities` - Finds all entities within the given radius centered at (x,y,z)
  - `moveTo` - Moves a mob to the specified position using the `Mob#moveTo` method
  - `getEntities` - Get all entities of a given type within the test bounds
  - `assertValueEqual` - Asserts that two objects are equivalent
  - `assertEntityNotPresent` - Asserts that an entity is not present between the given vectors
- `net.minecraft.gametest.framework.GameTestListener#testAddedForRerun` - A listener method that is called if the test needs to be rerun
- `net.minecraft.gametest.framework.GameTestRunner$Builder` - A builder for constructing a runner to run the game tests
- `net.minecraft.network.RegistryFriendlyByteBuf` - A byte buffer which holds a `RegistryAccess`
- `net.minecraft.resources.RegistryDataLoader#load` - Methods have been added to add a `LoadingFunction` which handles how the data in the registry is loaded
- `net.minecraft.resources.RegistryOps`
  - `injectRegistryContext` - Wraps a `Dynamic` with some ops into a `Dynamic` with a `RegistryOps`
  - `withParent` - Creates a `RegistryOps` with the passed in `DynamicOps` as a delegate
- `net.minecraft.resources.ResouceLocation#readNonEmpty` - Reads a `ResourceLocation`, throwing an error if the reader buffer is empty
- `net.minecraft.server.MinecraftServer`
  - `reloadableRegistries` - Holds an accessor to the dynamic registries on the server
  - `subscribeToDebugSample` - Registers a player to receive debug samples
  - `acceptsTransfers` - Determines whether transfer packets can be sent to the server
  - `reportChunkLoadFailure` - Reports a chunk has failed to load
  - `reportChunkSaveFailure` - Reports a chunk has failed to save
- `net.minecraft.server.ReloadableServerRegistries` - A class which holds registries that can be reloaded while in game (e.g., loot tables)
- `net.minecraft.server.level.ServerLevel#getPathTypeCache` - Gets the cache of `PathType`s when computed by an entity
- `net.minecraft.server.level.ServerPlayer
  - `setSpawnExtraParticlesOnFall` - Sets a boolean to spawn more particles when landing from a fall
  - `setRaidOmenPosition`, `clearRaidOmenPosition`, `getRaidOmenPosition` - Handles creating a raid from the given position
- `net.minecraft.util.ListAndDeque` - An interface which defines an object as a list and deque that can be randomly accessed
  - `ArrayListDeque` now implements this interface
- `net.minecraft.util.ExtraCodecs`
  - `sizeLimitedMap` - Makes sure the map is no larger than the specified size
  - `optionalEmptyMap` - Returns an optional map
- `net.minecraft.util.FastColor`
  - `as8BitChannel` - Gets an 8-bit color value from a float between 0-1
  - `color(int,int,int)` - Creates an opaque 32-bit color value from RGB
  - `opaque` -> Sets the alpha of a color to 255
  - `color(int,int)` - Creates a color with the R channel separated from the GB
  - `colorFromFloat` - Creates a 32-bit color value from floats between 0-1
- `net.minecraft.util.Mth#mulAndTruncate` - Multiplies a fraction by some value. Since integer division is used, the resulting division is floored
- `net.minecraft.util.NullOps` - An intermediary that only represents a null value
- `net.minecraft.util.ParticleUtils`
  - `spawnParticleInBlock` - Spawns particles within a block
  - `spawnParticles` - Spawns particles at the given position with some offset
- `net.minecraft.util.profiling.jfr.JvmProfiler#onRegionFile(Read|Write)` - Handles logic when the region file for a given world has been read from or written to
- `net.minecraft.world.Container#getMaxStackSize(ItemStack)` - Gets the max stack size for an `ItemStack` by getting the minimum of the container max stack size and the stack's max stack size
- `net.minecraft.world.InteractionResult#SUCCESS_NO_ITEM_USED` - The interaction was successful but the context entity did not use an item
- `net.minecraft.world.effect.MobEffect`
  - `onEffectAdded` - Called when the effect is first added to the entity
  - `onMobRemoved` - Called when the entity has been removed from the level when the effect is applied
  - `onMobHurt` - Called when the entity is hurt when the effect is applied
  - `createParticleOptions` - Creates the particle options to spawn around the entity
  - `withSoundOnAdded` - Sets the sound when the effect is added to an entity
- `net.minecraft.world.effect.MobEffectInstance`
  - `is` - Returns true if the effect holders are equal
  - `skipBlending` - Tells the effect not to blend with other environmental renderers (e.g. fog)
- `net.minecraft.world.entity.AnimationState#fastForward` - Speeds up the animation based upon the animation duration and a scale amount between 0-1
- `net.minecraft.world.entity.Crackiness` - Handles the state of how much armor has been damaged, or 'cracked'
- `net.minecraft.world.entity.Entity`
  - `getDefaultGravity` - Gets the default gravity value to apply to the entity
  - `getGravity` - Gets the gravity to apply to the entity, or if there is no gravity 0
  - `applyGravity` - Applies gravity to the delta movement of the entity.
  - `getNearestViewDirection` - Gets the direction nearest to the current view vector
  - `getDefaultPassengerAttachmentPoint` - Determines the default attachment point based upon the entity's current dimensions
  - `deflection` - Determines how an entity interacts with this projectile
  - `getPassengerClosestTo` - Gets the passenger closest to the given vector
  - `onExplosionHit` - Handles when the entity is hit with an explosion
  - `registryAccess` - Gets the current registry access
- `net.minecraft.world.entity.EntityType$Builder#spawnDimensionsScale` - Sets the scale factor to apply to the entity's dimensions when attempting to spawn
- `net.minecraft.world.entity.EquipmentSlot#BODY`
- `net.minecraft.world.entity.EquipmentSlotGroup` - Indicates a grouping of `EquipmentSlot`s
- `net.minecraft.world.entity.EquipmentTable` - Indicates the drop chances of a given `EquipmentSlot`
- `net.minecraft.world.entity.EquipmentUser` - Indicates that the entity can wear equipment
  - All `Mob`s and `ArmorStand`s are equipment users
- `net.minecraft.world.entity.LivingEntity`
  - `getComfortableFallDistance` - Increases the fall distance that the entity can survive without taking damage by three blocks
  - `doHurtEquipment` - Handles when a piece of equipment should be damaged
  - `canUseSlot` - Whether a slot can be used on an entity
  - `getJumpPower(float)` - Gets the power of a jump from the attribute, scaling by a float and the block jump factor and adding the jump boost power
  - `getDefaultDimensions` - Gets the base dimensions for a given pose
  - `getSlotForHand` - Gets the `EquipmentSlot` for a given `InteractionHand`
  - `hasInfiniteMaterials` - Whether the entity has an infinite amount of materials in its inventory. This is currently only used by the `Player`
- `net.minecraft.world.entity.Mob`
  - `getTargetFromBrain` - Gets the attack target of the entity, or null if none is available
  - `stopInPlace` - Stops all navigation and entity movement
  - `clampHeadRotationToBody` - Keeps the head movement within the bounds of the body
  - `getBodyArmorItem`, `canWearBodyArmor`, `isWearingBodyArmor`, `isBodyArmorItem`, `setBodyArmorItem` - Logic for handling armor in the `BODY` slot
  - `mayBeLeashed` - If the entity can be leashed
- `net.minecraft.world.entity.SlotAccess#of` - Creates a `SlotAccess` using a supplier, consumer setup
- `net.minecraft.world.entity.SpawnPlacements#isSpawnPositionOk` - If the entity can spawn in this position
- `net.minecraft.world.entity.ai.attributes.AttributeInstance#addOrUpdateTransientModifier` - Updates a modifier value to the current one if it is already present, otherwise adds it
- `net.minecraft.world.entity.ai.behavior.Swim#shouldSwim` - Checks whether the entity can swim
- `net.minecraft.world.entity.ai.navigation.PathNavigation#moveTo` - A `moveTo` method which takes in a close enough distance
- `net.minecraft.world.entity.monster.AbstractSkeleton`
  - `getHardAttackInterval` - After how many ticks the entity will attack when in the hard difficulty
  - `getAttackInterval` - After how many ticks the entity will attack when not in the hard difficulty
- `net.minecraft.world.entity.player.Inventory#contains(Predicate)` - Whether a stack in the inventory matches the predicate
- `net.minecraft.world.entity.player.Player#canInteractWithEntity` - Whether the player can interact with another entity
- `net.minecraft.world.entity.projectile.AbstractArrow`
  - `getDefaultPickupItem`, `setPickupItemStack` - Gets the default pickup item if the actual item stack cannot be obtained or is empty
  - `getSlot` - Gets the slot access of the pickupable stack
- `net.minecraft.world.entity.projectile.Projectile`
  - `getMovementToShoot` - Gets the movement to apply to the entity when shooting
  - `hitTargetOrDeflectSelf` - Gets the deflection status of the projectile
  - `deflect` - Deflects the entity
  - `onDeflection` - What to do when this projectile is deflected
- `net.minecraft.world.entity.projectile.ProjectileDeflection` - The status of how a projectile can be deflected, typically on punch
- `net.minecraft.world.entity.raid.Raider`
  - `isCaptain` - Whether the entity is a captain of a raid
  - `hasRaid` - If the entity has a raid to do
- `net.minecraft.world.entity.vehicle.ContainerEntity#getBoundingBox` - Gets the bounding box of the entity
- `net.minecraft.world.flag.FeatureFlagSet`
  - `isEmpty` - Whether there are no feature flags enabled
  - `intersects` - Whether two feature flag sets contain at least one similar feature flag
  - `subtract` - Returns a feature flag set without the flags of the parameter
- `net.minecraft.world.food.FoodConstants#saturationByModifier` - Takes in the number of hearts to heal and how the saturation should be multiplied by
- `net.minecraft.world.inventory.SlotRange` - Defines a range of slot indexes that can be referenced from some string prefix plus the index (all available `SlotRange`s are in `SlotRanges`
- `net.minecraft.world.item.ArmorItem$Type` is now `StringRepresentable`
  - `hasTrims` - Whether the armor type supports trims
- `net.minecraft.world.item.ArmorMaterial#layers` - Stores information about the texture location of the armor type
- `net.minecraft.world.item.DiggerItem#createAttributes` - Creates the attribute modifiers given for a tier, attack damage, and attack speed
- `net.minecraft.world.item.Item`
  - `components` - Gets the data component map
  - `getDefaultMaxStackSize` - Reads the max stack size of the component, or defaults to 1
  - `getAttackDamageBonus` - Applies a bonus to the attack damage when in the main hand
  - `getBreakingSound` - The sound even to play when the item breaks
  - `$Properties#component` - Adds a data component to the item
  - `$Properties#attributes` - Sets the attribute modifiers for the item
  - `$TooltipContext` - Holds access to the available registries, tick rate, and `MapItemSavedData` when coming from a `Level`
- `net.minecraft.world.item.ItemStack`
  - `getPrototype` - Gets the component map from the item
  - `getComponentsPatch` - Gets the patches to the component map on the current `ItemStack`
  - `validateComponents` - Validates whether certain components can be on the component map
  - `parse` - Parses the stack from a tag
  - `parseOptional` - Parses the stack from a tag, or returns an empty stack when not present
  - `save` - Saves the stack to a tag, or returns an empty tag
  - `transmuteCopy` - Creates a new copy, providing the component patch
  - `transmuteCopyIgnoreEmpty` - Creates a new copy without checking whether the stack is empty or not
  - `listMatches` - Returns whether two lists of stacks match each other 1-to-1
  - `lenientOptionalFieldOf` - Creates a map codec with an optional stack
  - `hashItemAndComponents` - Computes the hash code for a given stack
  - `hashStackList` - Computes the hash code for a list of stacks
  - `set` - Sets a data component value
  - `update` - Updates a data component value
  - `applyComponentsAndValidate` - Applies a component patch and validates the components
  - `applyComponents` - Applies a patch to the component map and validates
  - `getEnchantments` - Gets the item enchantments from the data component
  - `forEachModifier` - Applies a consumer for every modifier in the attribute modifiers
  - `limitSize` - Sets the count of the stack if the current count is larger than the limit
  - `consume` - Consumes a single item
- `net.minecraft.world.item.ProjectileItem` - Defines an item that can function like a projectile. Contains logic for constructing the projectile and shooting it either from the entity or dispenser
  - `ProjectileWeaponItem` contains protected implementations of these methods to be set when implementing `ProjectileItem`
- `net.minecraft.world.item.SwordItem#createAttributes` - Creates the attribute modifiers given for a tier, attack damage, and attack speed
- `net.minecraft.world.item.Tier`
  - `getIncorrectBlocksForDrops` - A negative restraint preventing certain blocks from receiving a boost from this tier
  - `createToolProperties` - Creates a `Tool` given the tag containing the blocks to speed up mining for
- `net.minecraft.world.level.ExplosionDamageCalculator#getKnockbackMultiplier` - Returns a scalar on how much knockback to apply to the entity
- `net.minecraft.world.level.SpawnData#isValidPosition` - Whether the entity can spawn within the specified block and sky light
- `net.minecraft.world.level.block.BonemealableBlock`
  - `getParticlePos` - Gets the position the particle should spawn
  - `getType` - Gets the type of effect the bonemeal has on particle spawning
- `net.minecraft.world.level.block.entity.AbstractFurnaceBlockEntity#invalidateCache` - Clears the fuel map
- `net.minecraft.world.level.block.entity.BannerBlockEntity#getPatterns` - Gets the stored patterns on the banner
- `net.minecraft.world.level.block.entity.BlockEntity`
  - `applyImplicitComponents` - Reads any data stored on the component input for the block entity
  - `collectImplicitComponents` - Writes any data to be stored on the stack
  - `removeComponentsFromTag`- Removes any information that is stored doubly within the `BlockEntityTag` that is handled by a data component
  - `components`, `setComponents` - Getters and setters for the stored data component map
- `net.minecraft.world.level.block.entity.Hopper#isGridAligned` - Returns true if the hopper is always aligned to the block grid
- `net.minecraft.world.level.block.entity.trialspawner.PlayerDetector$EntitySelector` - Determines how to select an entity from a given context
- `net.minecraft.world.level.chunk.storage.RegionFile#getPath` - Gets the path of the region file
- `net.minecraft.world.level.chunk.storage.RegionFileInfo` - A record which contains the current level name, dimension, and the type of the region (e.g. chunk)
- `net.minecraft.world.level.levelgen.structure.Structure#getMeanFirstOccupiedHeight` - Gets the average height of the four corners of the structure
- `net.minecraft.world.level.levelgen.structure.BoundingBox#inflatedBy(int, int, int)` - Inflates the size of the box by the specified (x,y,z) in both directions
- `net.minecraft.world.level.levelgen.structure.placement.StructurePlacement`
  - `applyAdditionalChunkRestrictions` - Returns whether the structure should generate given some level of frequency reduction
  - `applyInteractionsWithOtherStructures` - Returns whether the structure should generate based on the exclusion zones
- `net.minecraft.world.level.levelgen.structure.templatesystem.StructureTemplate#updateShapeAtEdge(LevelAccessor, int, DiscreteVoxelShape, BlockPos)` - Calls `BlockState#updateShape` on all blocks on the edges of the structure
- `net.minecraft.world.level.pathfinder.NodeEvaluator#getPathType(Mob, BlockPos)` - Gets the path type by calling `getPathType(PathfindingContext, int, int, int)` where the `PathfindingContext` is constructed from the mob and the three integers by unwrapping the `BlockPos`
- `net.minecraft.world.level.pathfinder.PathfindingContext` - A context object which holds information related to the current world, cache, and mob position
- `net.minecraft.world.level.storage.LevelStorageSource$LevelStorageAccess`
  - `estimateDiskSpace` - Returns how much disk space is left to write to
  - `checkForLowDiskSpace` - Checks if the amount of disk space remaining is less than 64MiB
- `net.minecraft.world.level.storage.LevelSummary#canUpload` - Returns whether the current level summary can be uploaded in a realm
- `net.minecraft.world.level.storage.loot.ContainerComponentManipulator` - A helper for setting container contents for items in their data components
  - A list of available manipulators can be found in `ContainerComponentManipulators`
- `net.minecraft.world.level.storage.loot.functions.ListOperation` - Defines an operation that can be performed on a list
- `net.minecraft.world.level.storage.loot.functions.LootItemConditionalFunction#getType` - Gets the item function type that can be conditionally loaded
- `net.minecraft.world.phys.shapes.BitSetDiscreteVoxelShape#isInterior` - Checks whether the position is surrounded by fully encompassing shapes on all sides
- `net.minecraft.client.gui.font.providers.FreeTypeUtil#checkError` - Returns true if an error was found
- `net.minecraft.client.gui.screens.Screen`
  - `advancePanoramaTime` - Updates the panorama by a set interval every frame
    - Does not use the delta time frame provided in `renderPanorama`
  - `clearTooltipForNextRenderPass` - Clears the tooltip rendering data 
- `net.minecraft.nbt.CompoundTag#shallowCopy` - Creates a shallow copy of the tag
- `net.minecraft.network.protocol.PacketUtils#makeReportedException` - Fills the crash report before returning the reported exception to throw
- `net.minecraft.server.packs.repository.PackRepository#displayPackList` - Collects all packs into a single string by their id
- `net.minecraft.world.item.crafting.RecipeManager#getOrderedRecipes` - Gets all recipes ordered by their `RecipeType`
- `net.minecraft.world.level.chunk.ChunkGenerator#validate` - Resolves the feature step data to generate

### Changes

- `com.mojang.blaze3d.font.SheetGlyphInfo`
  - `getBearingX` -> `getBearingLeft`
  - `getBearingY` -> `getBearingTop`
  - `getUp` -> `getTop`
  - `getDown` -> `getBottom`
- `com.mojang.blaze3d.font.TrueTypeGlyphProvider` now uses `org.lwjgl.util.freetype` over `org.lwjgl.stb` for font data storage
- `com.mojang.blaze3d.platform.NativeImage#copyFromFont` -> `(FT_Face, int)`
  - This method also returns a boolean, indicating whether the operation was successful
- `com.mojang.blaze3d.systems.RenderSystem#getModelViewStack` now returns a `Matrix4fStack`
- `com.mojang.blaze3d.vertex.PoseStack#mulPoseMatrix` -> `#mulPose`
- `com.mojang.blaze3d.vertex.SheetedDecalTextureGenerator` now takes in a `PoseStack$Pose`
- `com.mojang.blaze3d.vertex.VertexConsumer#putBulkData` now takes in the alpha parameter for the color
- `net.minecraft.Util#LINEAR_LOOKUP_THRESHOLD` is now public
- `net.minecraft.advancements.Advancement#validate(ProblemReporter, LootDataResolver)` -> `validate(ProblemReporter, HolderGetter$Provider)`
- `net.minecraft.advancements.AdvancementRewards$Builder#loot`, `addLootTable` now take in a `ResourceKey<LootTable>`
- `net.minecraft.client.MouseHandler#turnPlayer` is now private
- `net.minecraft.client.Options$FieldAccess#process(String, OptionInstance)` -> `Options$OptionAccess$process`
- `net.minecraft.client.gui.components.AbstractSelectionList#renderList` -> `renderListItems`
- `net.minecraft.client.gui.components.AbstractWidget#setTooltipDelay(int)` -> `setTooltipDelay(Duration)`
- `net.minecraft.client.gui.components.ChatComponent
  - `render` now takes in if the chat is focused as a boolean parameter
  - `isChatFocused` is now public
- `net.minecraft.client.gui.components.Checkbox#getBoxSize` is now public
- `net.minecraft.client.gui.components.FocusableTextWidget` now takes in an integer representing the padding of the widget
- `net.minecraft.client.gui.components.ObjectSelectionList$Entry#mouseClicked` returns true by default
- `net.minecraft.client.gui.components.OptionList`
  - The constructor now takes in a `OptionsSubScreen` containing the content height and header height
  - `addSmall` can now take in a varargs of `OptionInstance`s, a list of widgets, or two widgets
  - `getMouseOver` now holds an optional of `GuiEventListener`
  - `$Entry` -> `$OptionEntry`, `$Entry` is now an abstracted form which takes in the widgets directly
- `net.minecraft.client.gui.components.SpriteIconButton` now takes in a `Button$CreateNarration` object
  - This has been updated for all subclasses and provided a builder option called `narration`
- `net.minecraft.client.gui.components.Tooltip#setDelay`, `refreshTooltipForNextRenderPass`, `createTooltipPositioner` have been moved to `WidgetTooltipHolder`
  - The holder is stored on an `AbstractWidget` and is final. The tooltip within is mutable
- `net.minecraft.client.gui.components.debugchart.AbstractDebugChart` now takes in a `SampleStorage`, which is an interface to the old `SampleLogger` (now renamed `LocalSampleLogger`)
- `net.minecraft.client.gui.screens.ChatScreen#handleChatInput` no longer returns anything
- `net.minecraft.client.gui.screens.ConnectScreen#startConnecting` takes in the current `TransferState`
- `net.minecraft.client.gui.screens.GenericDirtMessageScreen` -> `GenericMessageScreen`
- `net.minecraft.client.gui.screens.OptionsSubScreen` now holds a `HeaderAndFooterLayout` to initialize widgets and reposition them
  - `addTitle`, `addFooter` have been added, replacing specific `createTitle` and `createFooter` methods in other implementations
- `net.minecraft.client.gui.screens.Screen#setTooltipForNextRenderPass` is now public
- `net.minecraft.client.gui.screens.advancements.AdvancementsScreen` can hold the last screen the user has seen
- `net.minecraft.client.gui.screens.inventory.InventoryScreen#renderEntityInInventory` now takes in a float for the scale parameter
- `net.minecraft.client.gui.screens.worldselection.WorldCreationUiState`
  - `setAllowCheats` -> `setAllowCommands`
  - `isAllowCheats` -> `isAllowCommands`
- `net.minecraft.client.gui.screens.worldselection.WorldOpenFlows`
  - `checkForBackupAndLoad(String, Runnable)` -> `openWorld`
  - `checkForBackupAndLoad(LevelStorageAccess, Runnable)` -> `openWorldLoadLevelData`
  - `loadLevel` -> `openWorldLoadLevelStem`
- `net.minecraft.client.multiplayer.ServerStatusPinger#pingServer` now takes in a runnable that runs on response from the pong packet
- `net.minecraft.client.particle.FireworkParticles$SparkParticle#setFlicker` -> `setTwinkle`
- `net.minecraft.client.player.inventory.Hotbar` holds a list of `Dynamic`s instead of the raw itemstack list
- `net.minecraft.client.renderer.EffectInstance` now takes in a `ResourceProvider` instead of a `ResourceManager`
  - `ResourceManager` implements `ResourceProvider`, so there is no major change in implementation
- `net.minecraft.client.renderer.LevelRenderer#renderClouds` takes in a `Matrix4f` containing the pose to transform the clouds to the correct location
- `net.minecraft.client.renderer.PanoramaRenderer#render(float, float)` -> `render(GuiGraphics, int, int, float, float)`
- `net.minecraft.client.renderer.PostChain` now takes in a `ResourceProvider` instead of a `ResourceManager`
  - `ResourceManager` implements `ResourceProvider`, so there is no major change in implementation
- `net.minecraft.client.renderer.PostChain#addPass` now takes in a boolean which determines whether to use `GL_LINEAR` when true or `GL_NEAREST` when false
- `net.minecraft.client.renderer.PostPass` now takes in a `ResourceProvider` instead of a `ResourceManager`
  - `ResourceManager` implements `ResourceProvider`, so there is no major change in implementation
- `net.minecraft.client.renderer.PostPass#getFilterMode`
  - Either `GL_LINEAR` or `GL_NEAREST`
- `net.minecraft.client.renderer.blockentity.SkullBlockRenderer#getRenderType(SkullBlock$Type, GameProfile)` -> `getRenderType(SkullBlock$Type, ResolvableProfile)`
- `net.minecraft.client.renderer.entity.LivingEntityRenderer#setupRotations` now takes in a scale representing the float parameter
- `net.minecraft.client.renderer.entity.ArrowRenderer#vertex` combines the `Matrix4f` and `Matrix3f` into a `PoseStack$Pose`
- `net.minecraft.client.renderer.entity.EntityRenderer#renderNameTag` now takes in a float representing the partial tick
- `net.minecraft.client.renderer.entity.layers.StrayClothingLayer` -> `SkeletonClothingLater`
- `net.minecraft.client.renderer.item.ItemProperties#getProperty(Item, ResourceLocation)` -> `getProperty(ItemStack, ResourceLocation)`
- `com.mojang.blaze3d.audio.OggAudioStream` replaced by `net.minecraft.client.sounds.FiniteAudioStream`, `FloatSampleSource`, `JOrbisAudioStream`
- `net.minecraft.commands.CommandBuildContext` now implements `HolderLookup$Provider`, replacing `holderLookup` and `configurable`
- `net.minecraft.commands.arguments.ComponentArgument` now takes in a `HolderLookup$Provider` when constructing a text component
- `net.minecraft.commands.arguments.ParticleArgument#readParticle` takes in a `HolderLookup$Provider` instead of a `HolderLookup` specifically for particle types
- `net.minecraft.commands.arguments.ResourceLocationArguments`
  - `getPredicate` -> `ResourceOrIdArgument#getLootPredicate`
  - `getItemModifier` -> `ResourceOrIdArgument#getLootModifier`
- `net.minecraft.commands.arguments.StyleArgument` now takes in a `HolderLookup$Provider` when constructing a style, or `CommandBuildContext` when calling `style`
- `net.minecraft.commands.arguments.item.ItemInput` no longer implements `Predicate<ItemStack>`
- `net.minecraft.commands.functions.CommandFunction#instantiate` no longer takes in the generic object
  - The generic object is now stored as an entry directly within the function itself (e.g. `MacroFunction`)
- `net.minecraft.ccommands.functions.FunctionBuilder#addMacro` takes in the generic object
- `net.minecraft.core.GlobalPos` is now a record, though there is no change in usage
- `net.minecraft.core.HolderLookup` the below logic is now applied specifically for `$RegistryLookup`
  - `filterElements` -> `HolderLookup$RegistryLookup#filterElements)
  - `$Delegate` -> `HolderLookup$RegistryLookup$Delegate`
- `net.minecraft.core.Registry#lifecycle(T)` -> `registrationInfo(ResourceKey<T>)`
- `net.minecraft.core.RegistrySetBuilder#lookupFromMap` now takes in a `HolderOwner`
- `net.minecraft.core.RegistrySetBuilder$CompositeOwner` -> `$UniversalOwner`
  - This is not one to one, it is simply a replacement
- `net.minecraft.core.WritableRegistry#register(ResourceKey, T, Lifecycle)` -> `register(ResourceKey, T, RegistrationInfo)`
- `net.minecraft.core.dispenser.AbstractProjectileDispenseBehavior` -> `ProjectileDispenseBehavior`
- `net.minecraft.data.DataProvider#saveStable` also takes in a `HolderLookup$Provider` to get the `RegistryOps` lookup context
- Game test batches have been reimplemented. The logic is now split between `net.minecraft.gametest.framework.GameTestBatch`, `GameTestBatchFactory`, and `GameTestBatchListener`
- `net.minecraft.gametest.framework.GameTestHelper#makeMockSurvivalPlayer`, `makeMockPlayer()` -> `makeMockPlayer(GameType)`
- `net.minecraft.gametest.framework.GameTestListener#testPassed`, `testFailed` now take in a `GameTestRunner`
- `net.minecraft.nbt.NbtUtils`
  - `readBlockPos` now takes in a key for the int array within the `CompoundTag`
  - `writeBlockPos` now returns an `IntArrayTag`
- `net.minecraft.network.chat.ChatType#CODEC` -> `#DIRECT_CODEC`
- `net.minecraft.network.chat.Component$Serializer` methods also take in a `HolderLookup$Provider`
- `net.minecraft.network.syncher.EntityDataAccessor` is now a record
- `net.minecraft.server.MinecraftServer`
  - `logTickTime` replaced by `getTickTimeLogger`, `isTickTimeLoggingEnabled`
  - `endMetricsRecordingTick` is now public
- `net.minecraft.server.ReloadableServerResources#getLootData` replaced by `fullRegistries`
- `net.minecraft.server.level` chunk-querying methods now return a `ChunkResult` instead of an `Either<ChunkAccess, ChunkHolder$ChunkLoadingFailure>`
- `net.minecraft.server.players.PlayerList`
  - `setAllowCheatsForAllPlayers` -> `setAllowCommandsForAllPlayers`
  - `isAllowCheatsForAllPlayers` -> `isAllowCommandsForAllPlayers`
- `net.minecraft.tags.TagNetworkSerialization#deserializeTagsFromNetwork` is now package-private
- `net.minecraft.util.SampleLogger` -> `net.minecraft.util.debugchart.LocalSampleLogger`
  - This is now an implementation of `SampleStorage`
- `net.minecraft.util.profiling.jfr.JvmProfiler#onPacketReceived` `onPacketSent` now take in the `PacketType` instead of an integer.
- `net.minecraft.util.profiling.jfr.stats.NetworkPacketSummary` -> `IoSummary`
- `net.minecraft.util.random.WeightedEntry$Wrapper` is now a record
- `net.minecraft.world.Container#stillValidBlockEntity` now takes in a float representing the radius past the block interaction distance to check whether the block entity can be interacted with
- `net.minecraft.world.ContainerHelper#saveAllItems`, `loadAllItems` now take in a `HolderLookup$Provider`
- `net.minecraft.world.InteractionResult#shouldAwardStats` -> `indicateItemUse`
- `net.minecraft.world.LockCode` is now a record
- `net.minecraft.world.RandomizableContainer#getLootTable`, `setLootTable`, `setBlockEntityLootTable` take in a `ResourceKey<LootTable>` instead of a `ResourceLocation`
- `net.minecraft.world.SimpleContainer#fromTag`, `createTag` now take in a `HolderLookup$Provider`
- `net.minecraft.world.damagesource.CombatRules#getDamageAfterAbsorb` now takes in a `DamageSource` to calculate armor breach
- `net.minecraft.world.effect.MobEffect`
  - `applyEffectTick` now returns a boolean that, when false, will remove the effect if `shouldApplyEffectTickThisTick` returns true
  - `createFactorData`, `setFactorDataFactory` has been replaced by `getBlendDurationTicks`, `setBlendDuration`
  - `getAttributeModifiers` is replaced by `createModifiers`
- `net.minecraft.world.effect.MobEffectInstance$FactorData` has been replaced by `$BlendState` along with the logic to apply
- `net.minecraft.world.entity.AreaEffectCloud#setPotion` -> `setPotionContents`
- `net.minecraft.world.entity.Entity` now implement `SyncedDataHolder` to handle network communication for entity properties
  - `defineSynchedData` now takes in a `SynchedEntityData$Builder`
  - `setSecondsOnFire` -> `igniteForSeconds`
    - There is also an `igniteForTicks` variant
  - `calculateViewVector` is now public
  - `getMyRidingOffset`, `ridingOffset` -> `getVehicleAttachmentPoint`
  - `getPassengerAttachmentPoint` now returns a `Vec3`
  - `getHandSlots`, `getArmorSlots`, `getAllSlots`, `setItemSlot` are no longer on `Entity`, but still on `LivingEntity`
  - `getEyeHeight` is now final and reads from the entity's dimensions
    - Eye height is set through `EntityType$Builder`
  - `getNameTagOffsetY` is now replaced by an `EntityAttachment`
  - `getFeetBlockState` -> `getInBlockState`
- `net.minecraft.world.entity.EntityDimensions` is now a record
- `net.minecraft.world.entity.EntityType`
  - `spawn`, `create` now takes in a `Consumer` of the entity to spawn rather than a `CompoundTag`
  - `updateCusutomEntityTag` now takes in a `CustomData` object instead of a `CompoundTag`
  - `getDefaultLootTable` now returns a `ResourceKey<LootTable>`
  - `getAABB` -> `getSpawnAABB`
- `net.minecraft.world.entity.LivingEntity`
  - `getScale` -> `getAgeScale`
    - `getScale` still exists and handles scaling based on an attribute property
- `net.minecraft.world.entity.Mob#finalizeSpawn` no longer takes in a `CompoundTag`
- `net.minecraft.world.entity.SpawnPlacements$Type` -> `SpawnPlacementsType`
  - All `SpawnPlacementType`s are in `SpawnPlacementTypes`
- `net.minecraft.world.entity.TamableAnimal`
  - `setTame` now takes in a second boolean that, when true, applies any side effects from taming
  - `reassessTameGoals` -> `applyTamingSideEffects`
- `net.minecraft.world.entity.ai.attributes.AttributeInstance`
  - `getModifiers` is now package-private
  - `removeModifier` is now public
- `net.minecraft.world.entity.ai.attributes.AttributeModifier` is now a record
  - `$Operation`
    - `ADDITION` -> `ADD_VALUE`
    - `MULTIPLY_BASE` -> `ADD_MULTIPLIED_BASE`
    - `MULTIPLY_TOTAL` -> `ADD_MULTIPLIED_TOTAL`
- `net.minecraft.world.entity.ai.behavior.BehaviorUtils#lockGazeAndWalkToEachOther` now takes in an integer representing the close enough distance
- `net.minecraft.world.entity.ai.village.poi.PoiManager` now takes in a `RegionStorageInfo`
- `net.minecraft.world.entity.animal.Animal#isFood` is now abstract
- `net.minecraft.world.entity.player.Player`
  - `disableShield` no longer takes in a boolean
  - `getPickRange` -> `blockInteractionRange`, `entityInteractionRange`
- `net.minecraft.world.entity.projectile.AbstractArrow#deflect` -> `Entity#deflection`
- `net.minecraft.world.entity.raid.Raid`
  - `getMaxBadOmenLevel` -> `getMaxRaidOmenLevel`
  - `getBadOmenLevel` -> `getRaidOmenLevel`
  - `setBadOmenLevel` -> `setRaidOmenLevel`
  - `absorbBadOmen` -> `absorbRaidOmen` and returns whether the entity has the effect
  - `getLeaderBannerInstance` now takes in a `HolderGetter<BannerPattern>`
- `net.minecraft.world.entity.raid.Raids#createOrExtendRaid` now takes in a `BlockPos` the raid is centered around
- `net.minecraft.world.food.FoodData#eat` no longer takes in an `Item`
- `net.minecraft.world.food.FoodProperties` is now a record
  - `fastFood` has been replaced by a `eatSeconds` float representing time
  - `Builder$saturationMod` -> `saturationModifier`
  - `Builder$alwaysEat` -> `alwaysEdible`
  - `Builder$meat` has been replaced with item tags for the given entity (e.g., `minecraft:wolf_food`
- `net.minecraft.world.inventory.tooltip.BundleTooltip` stores its information within `BundleContents` via `#contents`
- `net.minecraft.world.item.AdventureModeCheck` -> `AdventureModePredicate`
- `net.minecraft.world.item.ArmorMaterial` is now a record
  - `getDurabilityForType` -> `ArmorItem$Type#getDurability`
  - `getDefenseForType` -> `defense`
- `net.minecraft.world.item.AxeItem` is now public
- `net.minecraft.world.item.CrossbowItem#performShooting` is now an instance method
- `net.minecraft.world.item.HoeItem` is now public
- `net.minecraft.world.item.Item`
  - `verifyTagAfterLoad` -> `verifyComponentsAfterLoad`
  - `getMaxStackSize` -> `ItemStack#getMaxStackSize`
  - `getMaxDamage` -> `ItemStack#getMaxDamage`
  - `canBeDepleted` -> `ItemStack#isDamageableItem`
  - `isCorrectToolForDrops` now takes in an `ItemStack`
  - `appendHoverText` holds an `Item$TooltipContext` instead of a `Level`
  - `getDefaultAttributeModifiers` now returns an `ItemAttributeModifiers`
  -  `canBeHurtBy` -> `ItemStack#canBeHurtBy`
- `net.minecraft.world.item.ItemStack` now implements `DataComponentHolder`
  - The constructor takes in a `DataComponentPatch` or `PatchedDataComponentMap` instead of a `CompoundTag`
  - `save(CompoundTag)` -> `save(HolderLookup$Provider, Tag)`
  - `hurt` -> `hurtAndBreak`
    - A `Runnable` is the last parameter which determines what to do when a stack exceeds the max damage
  - `hurtAndBreak(int, T, Consumer<T>)` -> `hurtAndBreak(int, LivingEntity, EquipmentSlot)`
  - `isSameItemSameTags` -> `isSameItemSameComponents`
  - `getTooltipLines(Player, TooltipFlag)` -> `getTooltipLines(Item$TooltipContext, Player, TooltipFlag)`
  - `hasAdventureModePlaceTagForBlock` -> `canPlaceOnBlockInAdventureMode`
  - `hasAdventureModeBreakTagForBlock` -> `canBreakBlockInAdventureMode`
- `net.minecraft.world.item.ItemStackLinkedSet#createTypeAndTagSet` -> `createTypeAndComponentsSet`
- `net.minecraft.world.item.ItemUtils#onContainerDestroyed` now takes a `Iterable<ItemStack>` instead of a `Stream<ItemStack>`
- `net.minecraft.world.item.SmithingTemplateItem#createArmorTrimTemplate` takes in a varargs of `FeatureFlag`s
- `net.minecraft.world.item.SpawnEggItem#spawnsEntity`, `getType` now takes in an `ItemStack` instead of a `CompoundTag`
- `net.minecraft.world.item.alchemy.Potion#getName` now takes in an `Optional<Holder<Potion>>`
- `net.minecraft.world.item.alchemny.PotionUtils` -> `PotionContents`
  - Method names are roughly equivalent, ignorning all instances where a tag is wanted
- `net.minecraft.world.item.armortrim.ArmorTrim` now takes in a boolean indicating whether the tooltip component will show up
  - This tooltip can be toggled via `withTooltip`
- `net.minecraft.world.item.crafting.Recipe`
  - `assemble(C, RegistryAccess)` -> `assemble(C, HolderLookup.Provider)`
  - `getResultItem(RegistryAccess)` -> `getResultItem(HolderLookup.Provider)`
- `net.minecraft.world.item.crafting.RecipeCache#get` now returns a `Optional<RecipeHolder<CraftingRecipe>>`
- `net.minecraft.world.item.crafting.RecipeManager`
  - `getRecipeFor(RecipeType<T>, C, Level, ResourceLocation)` now returns an `Optional<RecipeHolder<T>>`
    - The `RecipeHolder` contains the `ResourceLocation`
  - `byType` now returns a `Collection<RecipeHolder` instead of a `Map<ResourceLocation, RecipeHolder<T>>`
- `net.minecraft.world.item.traiding.MerchantOffer` now takes in `ItemCost`s for cost stack instead of `ItemStack`s
- `net.minecraft.world.level.BaseCommandBlock#setName` -> `setCustomName`
- `net.minecraft.world.level.Level`
  - `getMapData`, `setMapData`, `getFreeMapId` now take in a `MapId` instead of an integer
  - `createFireworks` now takes in a `List<FireworkExplosion>` instead of a `CompoundTag`
- `net.minecraft.world.level.SpawnData` now takes in an `EquipmentTable`, which contains the armor an entity should spawn with
- `net.minecraft.world.level.StructureManager`
  - `getStructureWithPiecesAt` now takes in some set of `Sturcture`s
  - `checkStructurePresence` now takes in a `StructurePlacement`
- `net.minecraft.world.level.block.Block#appendHoverText` now takes in an `Item$TooltipContext` instead of a `BlockGetter`
- `net.minecraft.world.level.block.EnchantmentTableBlock` -> `EnchantingTableBlock`
- `net.minecraft.world.level.block.FlowerBlock` now takes in a `SuspiciousStewEffects`
- `net.minecraft.world.level.block.entity.BeehiveBlockEntity`
  - `addOccupantWithPresetTicks` -> `addOccupant`
  - `storeBee(CompoundTag, int, boolean)` -> `storeBee(BeehiveBlockEntity$Occupant)`
- `net.minecraft.world.level.block.entity.BlockEntity`
  - `load` -> `loadAdditional`
  - `saveAdditional`, `saveWithFullMetadata`, `saveWithId`, `saveWithoutMetadata`, `saveToItem`, `loadStatic`, `getUpdateTag` now takes in a `HolderLookup$Provider`
- `net.minecraft.world.level.block.entity.RandomizableContainerBlockEntity`
  - `getItems` -> `BaseContainerBlockEntity#getItems`
  - `setItems` -> `BaseContainerBlockEntity#setItems`
- `net.minecraft.world.level.block.entity.trialspawner.PlayerDetector#detect(ServerLevel, BlockPos, int)` -> `detect(ServerLevel, PlayerDetector$EntitySelector, BlockPos, double, boolean)`
- `net.minecraft.world.level.block.state.StateHolder#getValues` now returns a regular `Map` as it stores an `Reference2ObjectArrayMap`
- `net.minecraft.world.level.chunk.ChunkAccess#getBlockEntityNbtForSaving` now takes in a `HolderLookup$Provider`
- `net.minecraft.world.level.chunk.ChunkStatus` -> `net.minecraft.world.level.chunk.status.ChunkStatus`
  - Some other classes that were inner classes have also been moved to the `net.minecraft.world.level.chunk.status` package (e.g. `ChunkType`)
- `net.minecraft.world.level.gameevent.GameEvent` is now a record`
- `net.minecraft.world.level.gameevent.GameEventListener$Holder` -> `GameEventListener$Provider`
- `net.minecraft.world.level.levelgen.WorldDimensions(Map<ResourceKey<LevelStem>, LevelStem>)` is the new base constructor, the previous constructor with the `Registry` still exists
- `net.minecraft.world.level.levegen.presets.WorldPreset#createRegistry` -> `dimensionsInOrder`
- `net.minecraft.world.level.levelgen.structure.StructureCheck#checkStart` now takes in a `StructurePlacement`
- `net.minecraft.world.level.levelgen.structure.StructurePiece#createChest` now takes in a `ResourceKey<LootTable>` instead of a `ResourceLocation`
- `net.minecraft.world.level.levelgen.structure.StructurePiece#createDispenser` now takes in a `ResourceKey<LootTable>` instead of a `ResourceLocation`
- `net.minecraft.world.level.pathfinder.BlockPathTypes` -> `PathType`
- `net.minecraft.world.level.pathfinder.NodeEvaluator`
  - `getGoal` -> `getTarget`
  - `getTargetFromNode` -> `getTargetNodeAt`
    - Gets the node based on a position rather than supplying the node itself
  - `BlockPathTypes getBlockPathType(BlockGetter, int, int, int, Mob)` -> `PathType getPathTypeOfMob(PathfindingContext, int, int, int, Mob)`
  - `BlockPathTypes getBlockPathType(BlockGetter, int, int, int)` -> `PathType getPathType(PathfindingContext, int, int, int)`
- `net.minecraft.world.level.pathfinder.SwimNodeEvaluator`
  - `isDiagonalNodeValid` -> `hasMalus`
  - `getCachedBlockType` returns `PathType`
- `net.minecraft.world.level.pathfinder.WalkNodeEvaluator`
  - `isDiagonalValid` removes the accepted node value, only returning whether the diagonal can be made
    - Checking the accepted node has been moved to a separate `isDiagonalValid(Node)` method
  - `getBlockPathTypes`, `evaluateBlockPathType` -> `getPathTypeWithinMobBB`
  - `getCachedBlockType` -> `getCachedPathType`
  - `getBlockPathTypeStatic` -> `getPathTypeStatic`
  - `checkNeighbourBlocks(BlockGetter, BlockPos$MutableBlockPos, BlockPathTypes)` -> `checkNeighbourBlocks(PathfindingContext, int, int, int, PathType)`
  - `getBlockPathTypeRaw` -> `getPathTypeFromState`
  - `isBurningBlock` -> `NodeEvaluator#isBurningBlock`
- `net.minecraft.world.level.saveddata.SavedData`
  - `save(CompoundTag)` -> `save(CompoundTag, HolderLookup.Provider)`
  - `save(File)` -> `save(File, HolderLookup.Provider)`
  - `$Factory(Supplier<T>, Function<CompoundTag, T>, DataFixTypes)` -> `$Factory(Supplier<T>, BiFunction<CompoundTag, HolderLookup.Provider, T>, DataFixTypes)`
- `net.minecraft.world.level.saveddata.maps.MapBanner` is now a record
- `net.minecraft.world.levle.saveddata.maps.MapDecoration$Type` -> `MapDecorationType`
  - `MapDecorationType` is now a built-in registry object
- `net.minecraft.world.level.storage.LevelData#getXSpawn`, `getYSpawn`, `getZSpawn` -> `getSpawnPos`
- `net.minecraft.world.level.storage.LevelSummary#hasCheats` -> `hasCommands`
- `net.minecraft.world.level.storage.ServerLevelData#getAllowCommands` -> `isAllowCommands`
- `net.minecraft.world.level.storage.WorldData#getAllowCommands` -> `isAllowCommands`
- `net.minecraft.world.level.storage.WritableLevelData#setXSpawn`, `setYSpawn`, `setZSpawn`, `setSpawnAngle` -> `setSpawn`
- `net.minecraft.world.level.storage.loot.LootContext` now takes in a `HolderGetter$Provider`
  - `getResolver` now returns the `HolderGetter$Provider`
- `net.minecraft.world.level.storage.loot.LootDataType` is now a record
- `net.minecraft.world.level.storage.loot.entries.LootTableReference` -> `NestedLootTable`
- `net.minecraft.world.ticks.ContainerSingleItem#getContainerBlockEntity` -> `$BlockContainerSingleItem#getContainerBlockEntity`
- `net.minecraft.client.Minecraft#setLevel(ClientLevel)` -> `setLevel(ClientLevel, ReceivingLevelScreen$Reason)`
- `net.minecraft.client.OptionInstance$IntRange` now takes in a boolean that applies the option change immediately instead of after 0.6 seconds. The value is not considered saved at that moment when applied immediately
- `net.minecraft.client.gui.font.providers.FreeTypeUtil#checkError` -> `assertError`
  - `checkError` is now a boolean-returning method that doesn't throw an error
- `net.minecraft.core.particles.DustParticleOptionsBase` -> `ScalableParticleOptionsBase`
- `net.minecraft.nbt.CompoundTag#entries` -> `entrySet`
  - This is a logic replacement, the returns are similar to those of a `Map`
- `net.minecraft.network.PacketListener#shouldPropagateHandlingExceptions` -> `onPacketError`
  - This is a logic replacement as the new method simply decides how to handle the error

### Removals

- `com.mojang.blaze3d.systems.RenderSystem#inverseViewRotationMatrix` along with subsequent getters and setters
- `net.minecraft.client.Minecraft#is64Bit`
- `net.minecraft.client.gui.components.AbstractSelectionList#setRenderBackground`
- `net.minecraft.client.gui.components.DebugScreenOverlay#logTickDuration`
- `net.minecraft.client.gui.font.FontManager#setRenames`, `getActualId`
- `net.minecraft.client.gui.screens.MenuScreen#create` no longer checks nullability of `MenuType`
- `net.minecraft.client.gui.screens.Screen#hideWidgets`
- `net.minecraft.client.gui.screens.achievement.StatsUpdateListener`
- `net.minecraft.client.gui.screens.worldselection.WorldOpenFlows#loadBundledResourcePack`
- `net.minecraft.client.multiplayer.ClientLevel#setScoreboard`
- `net.minecraft.client.multiplater.MultiPlayerGameMode`
  - `getPickRange`
  - `hasFarPickRange`
- `net.minecraft.client.multiplayer.ServerData`
  - `setEnforcesSecureChat`
  - `enforcesSecureChat`
- `net.minecraft.client.renderer.GameRenderer`
  - `cycleEffect`
  - `getPositionTexColorNormalShader`
  - `getPositionTexLightmapColorShader`
- `net.minecraft.commands.arguments.ArgumentSignatures#get`
- `net.minecraft.core.RegistryCodecs`
  - `withNameAndId`
  - `networkCodec`
  - `fullCodec`
  - `$RegistryEntry`
- `net.minecraft.core.dispenser.DispenseItemBehavior#getEntityPokingOutOfBlockPos`
- `net.minecraft.server.level.ChunkHolder#getFullChunk`
- `net.minecraft.util.JavaOps`
- `net.minecraft.world.effect.AttributeModifierTemplate`
- `net.minecraft.world.entity.Entity#setMaxUpStep`
- `net.minecraft.world.entity.LivingEntity`
  - `getMobType`
  - `getEyeHeight`, `getStandingEyeHeight`
- `net.minecraft.world.entity.MobType`
- `net.minecraft.world.entity.ai.goal.GoalSelector`
  - `getRunningGoals`
  - `setNewGoalRate`
- `net.minecraft.world.entity.player.Player#isValidUsername`
- `net.minecraft.world.entity.projectile.ThrowableItemProjectile#getItemRaw`
- `net.minecraft.world.item.BlockItem#getBlockEntityData`
- `net.minecraft.world.item.Vanishable`
- `net.minecraft.world.item.DyeableArmorItem`
- `net.minecraft.world.item.DyeableHorseArmorItem`
- `net.minecraft.world.item.DyeableLeatherItem`
- `net.minecraft.world.item.EnchantedGoldenAppleItem`
- `net.minecraft.world.item.FireworkStarItem#appendHoverText(CompoundTag, List<Component>)`
- `net.minecraft.world.item.HorseArmorItem`
- `net.minecraft.world.item.Item`
  - `shouldOverrideMultiplayerNbt`
  - `getRarity`
  - `isEdible`
  - `getFoodProperties`
  - `isFireResistant`
- `net.minecraft.world.item.ItemStack`
  - `of`
  - `hasTag`
  - `getTag`
  - `getOrCreateTag`
  - `getOrCreateTagElement`
  - `getTagElement`
  - `removeTagKey`
  - `getEnchantmentTags`
  - `setTag`
  - `setHoverName`
  - `hasCustomHoverName`
  - `addTagElement`
  - `getBaseRepairCost`
  - `setRepairCost`
  - `getAttributeModifiers`
  - `isEdible`
  - `$TooltipPart`
- `net.minecraft.world.item.SuspiciousStewItem`
  - `saveMobEffects`
  - `appendMobEffects`
  - `listPotionEffects`
- `net.minecraft.world.item.armortrim.ArmorTrim`
  - `setTrim`
  - `getTrim`
  - `appendUpgradeHoverText`
- `net.minecraft.world.item.crafting.Ingredient`
  - `toNetwork`
  - `fromNetwork`
- `net.minecraft.world.level.Level#dimensionTypeId`
- `net.minecraft.world.level.NaturalSpawner#isSpawnPositionOk`
- `net.minecraft.world.level.block.entity.BaseContainerBlockEntity#setCustomName`
- `net.minecraft.world.level.block.entity.BeehiveBlockEntity#addOccupant`
- `net.minecraft.world.level.storage.loot.LootDataId`
  - Basically a `ResourceKey`
- `net.minecraft.world.level.storage.loot.LootDataManager`
  - Basically a `HolderGetter$Provider`
- `net.minecraft.world.level.storage.loot.LootDataResolver`
