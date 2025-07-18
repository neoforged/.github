# Minecraft 1.19.2 -> 1.19.3 Forge Mod Migration Primer

This is a high level, non-exhaustive overview on how to migrate your mod from 1.19.2 to 1.19.3 using Forge. All provided names use the official mojang mappings.

This primer is licensed under the [Creative Commons Attribution 4.0 International](http://creativecommons.org/licenses/by/4.0/), so feel free to use it as a reference and leave a link so that other readers can consume the primer.

If there's any incorrect or missing information, please file an issue on this repository or ping @ChampionAsh5357 in the Neoforged Discord server.

## Vanilla Changes

Vanilla changes are listed [here](./index.md).

## Registry Order

This is the current order in which registries are loaded:

- `minecraft:instrument` (Everything above is the same as vanilla)
- `forge:biome_modifier_serializers`
- `forge:entity_data_serializers`
- `forge:fluid_type`
- `forge:global_loot_modifier_serializers`
- `forge:holder_set_type`
- `forge:structure_modifier_serializers`
- `minecraft:worldgen/biome`

## Existing Creative Tabs

`CreativeModeTab`s are no longer set through a property on the item; they are harcoded onto the creative tab itself. To get around this, the `CreativeModeTabEvent` was added, allowing a modder to create a creative tab or add items to an existing creative tab. All events are on the **mod** event bus.

### Adding Items

Items can be added to a creative tab via `CreativeModeTabEvent$BuildContents`. The event contains the tab to add contents to, the set feature flags, and whether the user as OP permissions. An item can be added to the creative tab by calling `#accept` or `#acceptAll`. If you want to inject between an item already in the creative tab, you can call `MutableHashedLinkedMap#putBefore` or `MutableHashedLinkedMap#putAfter` within the provided entry list (`#getEntries`).

```java
// Registered on the MOD event bus
// Assume we have RegistryObject<Item> and RegistryObject<Block> called ITEM and BLOCK
@SubscribeEvent
public void buildContents(CreativeModeTabEvent.BuildContents event) {
  // Add to ingredients tab
  if (event.getTab() == CreativeModeTabs.INGREDIENTS) {
    event.accept(ITEM);
    event.accept(BLOCK); // Takes in an ItemLike, assumes block has registered item
  }
}
```

### Custom Creative Tab

A custom `CreativeModeTab` can be created via `CreativeModeTabEvent$Register#registerCreativeModeTab`. This takes in the name of the tab along with a consumer of the builder. An additional overload is specified to determine between which tabs this tab should be located.

```java
// Registered on the MOD event bus
// Assume we have RegistryObject<Item> and RegistryObject<Block> called ITEM and BLOCK
@SubscribeEvent
public void buildContents(CreativeModeTabEvent.Register event) {
  event.registerCreativeModeTab(new ResourceLocation(MOD_ID, "example"), builder ->
    // Set name of tab to display
    builder.title(Component.translatable("item_group." + MOD_ID + ".example"))
    // Set icon of creative tab
    .icon(() -> new ItemStack(ITEM.get()))
    // Add default items to tab
    .displayItems((enabledFlags, populator, hasPermissions) -> {
      populator.accept(ITEM.get());
      populator.accept(BLOCK.get());
    })
  );
}
```

## Packs and PackResources

`Pack` has become a private constructor, only constructed through one of the static constructors `#readMetaAndCreate`, which acts similarly to the 1.19.2 `#create`, or `#create`. A common case that this will encounter is for the `AddPackFindersEvent`. Here are the list of changes:

- `Supplier<PackResources>` has turned into `Pack$ResourcesSupplier` which takes in the pack id.
- `PackType` is now a parameter to check whether the pack is compatible for the current version.
- The description, feature flags, and hidden check is stored on the `Pack$Info`, which is either obtained from the metadata file or constructed from the record.

## Data Generators

Data Generators have changed quite a bit from how providers are added to the providers themselves.

Starting off, all providers now take in a `PackOutput` compared to the `DataGenerator` itself. The `PackOutput` simply specifies the directory where the pack will be generated. Because of this change, the `#addProvider` method now takes in either the provider instance or a function that takes in a `PackOutput` and constructs a `DataProvider`.

```java
// On the mod event bus
@SubscribeEvent
public void gatherData(GatherDataEvent event) {
  event.addProvider(true, output -> /*create provider here*/);
}
```

### The Lookup Provider

`GatherDataEvent` now provides a `CompletableFuture` containing a `HolderLookup$Provider`, which is used to get registries and their objects. This can be obtained via `#getLookupProvider`.

### TagsProvider and IntrinsicHolderTagsProvider

Forge adds a tag provider for `Block`s. Any others need to specify the function.

```java
// Subtype of `IntrinsicHolderTagsProvider`
public AttributeTagsProvider(PackOutput output, CompletableFuture<HolderLookup.Provider> registries, ExistingFileHelper fileHelper) {
  super(
    output,
    ForgeRegistries.Keys.ATTRIBUTES,
    registries,
    attribute -> ForgeRegistries.ATTRIBUTES.getResourceKey(attribute).get(),
    MOD_ID,
    fileHelper
  );
}
```

### AdvancementProvider

 For ease of access to the `ExistingFileHelper`, Forge has added an extension on the provider aptly named `ForgeAdvancementProvider` which takes in `ForgeAdvancementProvider$AdvancementGenerator`s which contain the `ExistingFileHelper` as a parameter.

```java
// On the MOD event bus
@SubscribeEvent
public void gatherData(GatherDataEvent event) {
  event.getGenerator().addProvider(
    // Tell generator to run only when server data are generating
    event.includeServer(),
    output -> new ForgeAdvancementProvider(
      output,
      event.getLookupProvider(),
      event.getExistingFileHelper(),
      // Sub providers which generate the advancements
      List.of(subProvider1, subProvider2, /*...*/)
    )
  );
}
```

The `ForgeAdvancementProvider$AdvancementGenerator` is responsible for generating advancements. To be able to effectively generate advancements the `ExistingFileHelper` should be passed in such that the advancement can be built using the `Advancement$Builder#save` method which takes it in.

```java
// In some ForgeAdvancementProvider$AdvancementGenerator or as a lambda reference

@Override
public void generate(HolderLookup.Provider registries, Consumer<Advancement> writer, ExistingFileHelper existingFileHelper) {
  // Build advancements here
}
```

### LootTableProvider

`LootTableProvider` has received a massive overhaul, no longer needing to subtype the class and instead just supply arguments to the constructor. In addition to the `PackOutput`, it takes in a set of table names to validate whether they have been created and a list of `SubProviderEntry`s, which are used to generate the tables for each `LootContextParamSet`.

```java
// On the MOD event bus
@SubscribeEvent
public void gatherData(GatherDataEvent event) {
  event.getGenerator().addProvider(
    // Tell generator to run only when server data are generating
    event.includeServer(),
    output -> new MyLootTableProvider(
      output,
      // Specify registry names of tables that are required to generate, or can leave empty
      Collections.emptySet(),
      // Sub providers which generate the loot
      List.of(subProvider1, subProvider2, /*...*/)
    )
  );
}
```

### DatapackBuiltinEntriesProvider and the removal of JsonCodecProvider#forDatapackRegistry

`JsonCodecProvider#forDatapackRegistry` was removed in favor of using the vanilla `RegistriesDatapackGenerator` for generating dynamic registry objects. To expand upon this provider, Forge introduced the `DatapackBuiltinEntriesProvider`, which can take in a `RegistrySetBuilder` to generate the specific dynamic objects to use, and a set of mod ids to determine which mod's dynamic registry objects to generate.

```java
// On the MOD event bus
@SubscribeEvent
public void gatherData(GatherDataEvent event) {
  event.getGenerator().addProvider(
    // Tell generator to run only when server data are generating
    event.includeServer(),
    output -> new DatapackBuiltinEntriesProvider(
      output,
      event.getLookupProvider(),
      // The objects to generate
      new RegistrySetBuilder()
        .add(Registries.NOISE_SETTINGS, context -> {
          // Generate noise generator settings
          context.register(
            ResourceKey.create(Registries.NOISE_SETTINGS, new ResourceLocation(MOD_ID, "example_settings")),
            NoiseGeneratorSettings.floatingIslands(context)
          );
        }),
      // Generate dynamic registry objects for this mod
      Set.of(MOD_ID)
    )
  );
}
```

## Resolving nested models

To resolve models that are nested within other models, `IUnbakedGeometry#resolveParents` was added. This is a defaulted method, so no issues should occur in custom loaders; however, those who are resolving models within models will need to pivot to use this new method.

## ModelEvent$ModifyBakingResult

`ModelEvent$ModifyBakingResult` is an event fired on the **mod** event bus used to handle the caching of state -> model map done previously in `ModelEvent$BakingCompleted`. The usecases typically stem from users who need to make adjustments to models due to cases where custom loaders are not possible or that the context provided is insufficient.

Due to the event being fired on a worker thread, you should only access those objects provided by the event itself as it is otherwise unsafe.

## Entity#getAddEntityPacket no longer abstract

`Entity#getAddEntityPacket` is no longer abstract, instead defaulting to the `ClientboundAddEntityPacket`. If you need to send additional data on entity creation, you should still use `NetworkHooks#getEntitySpawningPacket`.

## Minor Refactors

- `net.minecraft.data.tags.BlockTagsProvider` -> `net.minecraftforge.common.data.BlockTagsProvider`
