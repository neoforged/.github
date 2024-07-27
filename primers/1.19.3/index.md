# Minecraft 1.19.2 -> 1.19.3 Mod Migration Primer

This is a high level, non-exhaustive overview on how to migrate your mod from 1.19.2 to 1.19.3 using. This does not look at any specific mod loader, just the changes to the vanilla classes.

This primer is licensed under the [Creative Commons Attribution 4.0 International](http://creativecommons.org/licenses/by/4.0/), so feel free to use it as a reference and leave a link so that other readers can consume the primer.

If there's any incorrect or missing information, please leave a comment below. Thanks!

## Feature Flags

Feature Flags are a behavior/logic toggle for blocks and items. They determine whether certain actions or generations related to the object can trigger (e.g. bamboo boats can only be placed when the `minecraft:update_1_20` flag is enabled).

Vanilla currently provides three flags as of this version (via `FeatureFlags`):
- `minecraft:vanilla`: For everything in vanilla minecraft
- `minecraft:bundle`: For the bundle feature pack and in the creative tab
- `minecraft:update_1_20`: For objects related to the upcoming 1.20 release

You can force an item or block to require one of these features by adding it as a property via `#requiredFeatures`. This will affect all behavior and logic associated with the object. 

```java
// In some supplied instance
new Block(BlockBehaviour.Properties.of(/*...*/).requiredFeatures(/*features here*/));

new Item(new Item.Properties().requiredFeatures(/*features here*/));
```

If you only want to affect some behavior or logic associated with the object, then you will need to check `FeatureFlagSet#contains` yourself on whatever system you are using.

> Do not add your own `FeatureFlag`s. Currently, they are limited to only 6 and are hardcoded to vanilla's highly-specific implementation.

## Registries

Registries have received a significant overhaul, resulting in better stability.

### No In-Code Entries in Dynamic Registries

Dynamic registries such as dimension types or features can no longer be declared in-code. Instead, a JSON of each must now be provided.

### Key and Registry Locations

The locations of the registry keys and the registries themselves have moved. Registry keys are now in `net.minecraft.core.registries.Registries`. Static registries are now in `net.minecraft.core.registries.BuiltInRegistries`.

Dynamic registries can only be obtained from the `RegistryAccess` from the `MinecraftServer`. There is no more static access to it, such as using `RegistryAccess#builtinCopy`.

### Registry Order

The registry order has changed as well due to a bunch of stability fixes. This should not matter in most cases if you want to support registry replacement as a `Supplier` of the value is necessary, but this will be noted down regardless. This is the current order in which registries load:

- `minecraft:sound_event`
- `minecraft:fluid`
- `minecraft:block`
- `minecraft:attribute`
- `minecraft:mob_effect`
- `minecraft:particle_type`
- `minecraft:item`
- `minecraft:entity_type`
- `minecraft:sensor_type`
- `minecraft:memory_module_type`
- `minecraft:potion`
- `minecraft:game_event`
- `minecraft:enchantment`
- `minecraft:block_entity_type`
- `minecraft:painting_variant`
- `minecraft:stat_type`
- `minecraft:custom_stat`
- `minecraft:chunk_status`
- `minecraft:rule_test`
- `minecraft:pos_rule_test`
- `minecraft:menu`
- `minecraft:recipe_type`
- `minecraft:recipe_serializer`
- `minecraft:position_source_type`
- `minecraft:command_argument_type`
- `minecraft:villager_type`
- `minecraft:villager_profession`
- `minecraft:point_of_interest_type`
- `minecraft:schedule`
- `minecraft:activity`
- `minecraft:loot_pool_entry_type`
- `minecraft:loot_function_type`
- `minecraft:loot_condition_type`
- `minecraft:loot_number_provider_type`
- `minecraft:loot_nbt_provider_type`
- `minecraft:loot_score_provider_type`
- `minecraft:float_provider_type`
- `minecraft:int_provider_type`
- `minecraft:height_provider_type`
- `minecraft:block_predicate_type`
- `minecraft:worldgen/carver`
- `minecraft:worldgen/feature`
- `minecraft:worldgen/structure_processor`
- `minecraft:worldgen/structure_placement`
- `minecraft:worldgen/structure_piece`
- `minecraft:worldgen/structure_type`
- `minecraft:worldgen/placement_modifier_type`
- `minecraft:worldgen/block_state_provider_type`
- `minecraft:worldgen/foliage_placer_type`
- `minecraft:worldgen/trunk_placer_type`
- `minecraft:worldgen/root_placer_type`
- `minecraft:worldgen/tree_decorator_type`
- `minecraft:worldgen/feature_size_type`
- `minecraft:worldgen/biome_source`
- `minecraft:worldgen/chunk_generator`
- `minecraft:worldgen/material_condition`
- `minecraft:worldgen/material_rule`
- `minecraft:worldgen/density_function_type`
- `minecraft:worldgen/structure_pool_element`
- `minecraft:cat_variant`
- `minecraft:frog_variant`
- `minecraft:banner_pattern`
- `minecraft:instrument`
- `minecraft:worldgen/biome`

## Existing Creative Tabs

`CreativeModeTab`s are no longer set through a property on the item; they are harcoded onto the creative tab itself.

## The JOML Library

Mojang has migrated from using their own math classes for rendering to using [JOML](https://github.com/JOML-CI/JOML), a open source math library for OpenGL. You can fix most of these issues by changing the package of the vector, matrix, etc. to `org.joml`; however it is not a 1 to 1 translation. Some changes are functional: `Matrix*` methods modify a mutable instance instead of creating a new one, or `Quaternionf` using `AxisAngle*` instead of `Vector3f`.

## SoundEvent

`SoundEvent`s have slightly changed in this version. Specifically, they are constructed through a static method constructor which introduces a new system: fixed range sound. The standard `SoundEvent` from previous versions can be constructed using `SoundEvent#createVariableRangeEvent`, whose range changes depending on the volume of the sound with a minimum of 16 blocks. Sounds constructed from `SoundEvent#createFixedRangeEvent` can be heard from the range specified, regardless of the volume.

```java
// In some supplied instance

// Will change depending on volume, minimum 16 blocks
SoundEvent.createVariableRangeEvent("sound.example_mod.variable_example");

// Will only be heard within specified range (e.g. 5 blocks)
SoundEvent.createFixedRangeEvent("sound.example_mod.fixed_example", 5f);
```

## Packs and PackResources

`Pack`s and `PackResources` have changed slightly between the two versions.

First `PackResources#hasResource` no longer exists. Instead `#getResource` should be checked for nullability instead before getting the `InputStream` from the `IoSupplier`.

```java
// Given some PackResources resources
var io = resources.getResource(PackType.SERVER_DATA, new ResourceLocation(/**/));

if (io != null) {
  InputStream input = io.get();
  // Same as before
}
```

Additionally, `#getResources` has been replaced by `#listResources` with the `ResourceLocation` predicate changing to a `PackResources$ResourceOutput` which acts as a consumer taking in the resource and the `IoSupplier` the resource would be located in.

Finally, `Pack` has become a private constructor, only constructed through one of the static constructors `#readMetaAndCreate`, which acts similarly to the 1.19.2 `#create`, or `#create`.

## Data Generators

Data Generators have changed quite a bit from how providers are added to the providers themselves.

Starting off, all providers now take in a `PackOutput` compared to the `DataGenerator` itself. The `PackOutput` simply specifies the directory where the pack will be generated. Because of this change, the `#addProvider` method now takes in a function that takes in a `PackOutput` and constructs a `DataProvider`.

### Data Providers and CompleteableFutures

All `DataProvider`s now return a `CompletableFuture` on `#run`, which is used to write the data to its apropriate file.

### RecipeProvider and RecipeCategory

`RecipeProvider`s now construct recipes in `#buildRecipes`. Additionally, each recipe builder, besides dynamic recipes, must specify a `RecipeCategory` which determines the subdirectory on where the recipe will be generated.

```java
// In RecipeProvider#buildRecipes(writer)
ShapedRecipeBuilder builder = ShapedRecipeBuilder.shaped(RecipeCategory.MISC, result)
  .pattern("a a") // Create recipe pattern
  .define('a', item) // Define what the symbol represents
  .unlockedBy("criteria", criteria) // How the recipe is unlocked
  .save(writer); // Add data to builder
```

### TagsProvider and IntrinsicHolderTagsProvider

`TagsProvider`s can only add objects through their `ResourceKey`. To add objects directly, an `IntrinsicHolderTagsProvider` should be used instead. This takes in a function which extracts a key from the object itself. Vanilla creates these for `Item`s, `EntityType`s, `Fluid`s, and `GameEvent`s. Any others need to specify the function.

```java
// Subtype of `IntrinsicHolderTagsProvider`
public AttributeTagsProvider(PackOutput output, CompletableFuture<HolderLookup.Provider> registries) {
  super(
    output,
    Registries.ATTRIBUTE,
    registries,
    attribute -> BuiltInRegistries.ATTRIBUTE.getResourceKey(attribute).get()
  );
}
```

### LootTableProvider

`LootTableProvider` has received a massive overhaul, no longer needing to subtype the class and instead just supply arguments to the constructor. In addition to the `PackOutput`, it takes in a set of table names to validate whether they have been created and a list of `SubProviderEntry`s, which are used to generate the tables for each `LootContextParamSet`.

A `LootTableSubProvider` is used to generate the loot tables in the `SubProviderEntry`, essentially functioning the same as a `Supplier<Consumer<BiConsumer<ResourceLocation, LootTable.Builder>>>`.

```java
public class ExampleSubProvider implements LootTableSubProvider {

  // Used to create a factory method for the wrapping Supplier
  public ExampleSubProvider() {}

  // The method used to generate the loot tables
  @Override
  public void generate(BiConsumer<ResourceLocation, LootTable.Builder> writer) {
    // Generate loot tables here by calling writer#accept
  }
}

// In the list passed into the LootTableProvider constructor
new LootTableProvider.SubProviderEntry(
  ExampleSubProvider::new,
  // Loot table generator for the 'empty' param set
  LootContextParamSets.EMPTY
)
```

The overrides for blocks and entities still exist being called `BlockLootSubProvider` and `EntityLootSubProvider` respectively. They both take in the feature flags, with the block sub provider taking in a set of items to which are resistant to explosions. The method used to generate the tables have been changed from `#addTables` to `#generate`. You still need to override the `#getKnown*` methods for validation.

```java
// In some BlockLootSubProvider subclass
public MyBlockLootSubProvider() {
  super(Collections.emptySet(), FeatureFlags.REGISTRY.allFlags());
}

@Override
public void generate() {
  // Add tables here
}
```

### AdvancementProvider

`AdvancementProvider` has also received a massive overhaul, no longer needing to subtype the class and instead just supply arguments to the constructor. In addition to the `PackOutput`, it takes in the holder lookup for registries and the `AdvancementSubProvider`s, which are used to generate the advancements.

## Rendering Changes

Rendering has changed a massive amount in this update. The most common being those explicitly mentioned by minecraft such as item and block textures only being in `models/item` and `models/block` respectively or the asynchronous loading and writing of assets and data. Textures for items and blocks are also required to be in `textures/item` and `textures/block` respectively as textures are loaded before model are processed. These texture directories are specified in JSON in the `atlases` directory.

This section will try to cover all the updates in 1.19.3 as these issues get fixed.

## Behaviors by DataFixerUpper

Behaviors are being migrated to use a new system built around the abstractions and overengineering provided by DataFixerUpper (DFU). As these systems tend to be complex and convoluted, this will only provide a brief overview of the changes and implementations.

### BehaviorControl

The `Behavior` class is no longer the base for behavior logic. That has been relegated to `BehaviorControl`: an interface which `Behavior` implements. `BehaviorControl` checks whether the logic can start (`#tryStart`), its ticking and stop check (`#tickOrStop`), and finally the stop logic (`#doStop`).

Some classes affected by this are `DoNothing` and `GateBehavior` which implements `BehaviorControl`.

### OneShot

`OneShot` is a behavior control which executes once via a `Trigger` statement. If the `Trigger#trigger` returns true, the behavior control is executed; though it only sets the state to running for a single tick.

### BehaviorBuilder and TriggerGate

BehaviorBuilder and TriggerGate are essentially the DFU-implemented classes which creates `OneShot` triggers such as waking up or strolling to a point of interest.

For a basic understanding, a `BehaviorBuilder` acts similarly to a `RecordCodecBuilder`: it creates the behavior instance, takes in the necessary `MemoryModuleType`s and whether it is registered, present, or absent, and then applies to construct a `Trigger`. The `BehaviorBuilder` also contains methods to make triggers act sequentially or if a given predicate is met.

`TriggerGate` is an holder of state methods for `GateBehavior`s. It either triggers a single instance at random (`#triggerOneShuffled`) or executed based on the ordering and run policy selected (`#triggerGate`).

## Minor Changes

This is a list of minor changes which probably won't affect the modding experience but is convenient to know anyways.

### Entity#getAddEntityPacket no longer abstract

`Entity#getAddEntityPacket` is no longer abstract, instead defaulting to the `ClientboundAddEntityPacket`.

### AbstractContainerMenu#stillValid Static Method

`AbstractContainerMenu` has now migrated from using the `Container` to check whether the menu could still be kept open to using a static method called `#stillValid` in the instance of the same name. To use this method, you need to provide a `ContainerLevelAccess`, the player, and the `Block` the menu is attached to.

```java
// Client menu constructor
public MyMenuAccess(int containerId, Inventory playerInventory) {
  this(containerId, playerInventory, ContainerLevelAccess.NULL);
}

// Server menu constructor
public MyMenuAccess(int containerId, Inventory playerInventory, ContainerLevelAccess access) {
  // ...
}

// Assume this menu is attached to RegistryObject<Block> MY_BLOCK
@Override
public boolean stillValid(Player player) {
  return AbstractContainerMenu.stillValid(this.access, player, MY_BLOCK.get());
}
```

### BlockStateProvider#simpleBlockWithItem

The `BlockStateProvider` now contains a new method to generate a simple block state for a single block model along with an associated item model.

```java
// In some BlockStateProvider#registerStatesAndModels
// Assume there is a RegistryObject<Block> BLOCK
this.simpleBlockWithItem(BLOCK.get(), this.cubeAll(BLOCK.get()));
```

### Removal of Context-Sensitive Tree Growers

The context-sensitive methods within `AbstractMegaTreeGrower` and `AbstractTreeGrower` have been removed and as such no longer have access to the world or position as the methods now return the keys of the `ConfiguredFeature` rather than the feature itself.

### Music, now with Holders

The `Music` class, used to provide a background track during different situations, now takes in a `Holder<SoundEvent>` rather than the `SoundEvent` itself.

### KeyboardHandler#sendRepeatsToGui removed

Minecraft has removed `KeyboardHandler#sendRepeatsToGui` when a key was held down such that it repeated within a GUI. There is no replacement, so the handler will now always send repeats to the GUI.

## Renames and Refactors

The following classes were renamed or refactored within Minecraft:

- `net.minecraft.client.gui.components.Widget` -> `net.minecraft.client.gui.components.Renderable`
- `net.minecraft.data.loot.BlockLoot` -> `net.minecraft.data.loot.BlockLootSubProvider`
- `net.minecraft.data.loot.EntityLoot` -> `net.minecraft.data.loot.EntityLootSubProvider`
