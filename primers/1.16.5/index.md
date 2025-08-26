# Minecraft 1.15.2 -> 1.16.5 Modding Migration Primer

This is a high level and non-exhaustive overview of the 1.15.2 to 1.16.5 migration primer from a mostly Forge oriented perspective.

This primer is licensed under Creative Commons, so feel free to use it as a reference.

Please note that this can be updated, so leave a [link to this file](https://gist.github.com/50ap5ud5/f4e70f0e8faeddcfde6b4b1df70f83b8) so readers can see the updated information themselves.

If there's any incorrect or missing information please leave a comment down below. Thanks!

## DataFixerUpper and Codecs
In 1.16 Mojang updated their library DataFixerUpper (used to migrate old world data from older game versions to new versions) to include a new set of tools called Codecs.

Codecs are a bridging object that allows for easy serialisation/deserialisation between two different objects.
E.g. Java -> JSON and JSON -> Java.

### Uses
Alot of codecs are now used in core registry and utility objects, such as common Java primitive types, BlockState, ResourceLocation and World Generation.

This makes Codecs very useful in can reducing the amount of code needed for the serialisation/deserialisation of objects as this is mostly handled for you.

They are especially useful for data generation and for data driven modded registries. 

E.g. Using the Blockstate Codec to easily read and write blockstate information for a custom object that uses JSON.

### Limitations and Quirks
One behaviour to note is that during deserialisation Codecs will **discard any erroring entries** and will return a PartialResult with only the successfully parsed objects inside.

Most Codecs will print a log file entry when this happens, so if anything goes wrong be sure to check either the latest.log or debug.log.


There are also some flaws with certain Codec types which can cause unintended issues.

For example, the UnboundedMapCodec is highly sensitive when it encounters errors, and will discard all entries prior to an erroring entry, even if those entries were not erroring.

This is a big issue for some implementations, as seen in cases like <a href="https://bugs.mojang.com/browse/MC-197860">MC-197860</a>, in which the cause was due to the error handling behaviour of UnboundedMapCodec.


## Dynamic Registries
In 1.16+ alot of registry objects became data driven.
The DynamicRegistries manages the syncing of these data driven registries.

Examples of registries that are now a Dynamic Registry:
- Dimension
- DimensionType
- Biome
- ConfiguredFeature
- ConfiguredStructureFeature

The objects part of the Dynamic Registries only need to be registered within code if they are to be used within a pre-existing registry object (e.g. a ConfiguredFeature to add it to a Biome within the BiomeLoadingEvent). 

Otherwise, it is recommended to use a JSON file.

### Accessing DynamicRegistries and Potential Pitfalls
During runtime you can get an object from a dynamic registry from either the MinecraftServer or the ClientPlayNetHandler, depending on the logical side you are on.

E.g. 
```
RegistryKey<Biome> myBiomeRegistryKey = RegistryKey.get(new ResourceLocation(MODID, "test_biome"));

Biome biome = serverWorld.getServer().registryAccess().registryOrThrow(Registry.BIOME).get(myBiomeRegistryKey);
```

However, please note that **not all dynamic registries are present on the client's copy** of the dynamic registries.

DO NOT access dynamic registries that are not available on the client, as this will crash dedicated servers.

## Registry Keys
The RegistryKey is a new object introduced in 1.16 which is combination of a registry's ID and the object's registry name.

It is a universally unique identifier for an object.


Most vanilla registry objects in 1.16 are now identified via this key-value pair.

For example, a Biome object has a RegistryKey which is made up of the biome and the Biome Registry.

The Biome Registry itself also has a RegistryKey, which is made up of a registry name and the Biome Registry. 

E.g. ``RegistryKey<Biome> testBiomeKey = RegistryKey.get(Registry.BIOME, new ResourceLocation(MODID, "test_biome"));``

### Usage

RegistryKey(s) can be compared with an equality sign ``==``.

RegistryKey(s) can work for any modded or vanilla registry.

If you want to make a custom registry that is non dynamic, Forge's Custom Deferred Register system is the easiest and very stable solution.

## World Generation
Most world generation objects have become data driven via the Minecraft Datapack system.

While some objects still require code based objects, most world generation is now datapack based.

This may seem like a surprise for some as in all previous versions, it was code based. 
This is because large breaking changes were made in a MINOR version, 1.16.2. 

It is possible these changes were originally part of the initial 1.16 release, but had to be delayed for other reaons.

We can expect this to happen in future minor versions, though it is unlikely that they will introduce changes on this scale often.

### Registriation Process Changes
With the move to data-driven registries, it is important to know how world generation objects are registered in the new system.

There are now 2 types of registries for World generation objects:

1. WorldGenRegistries. A temporary registry that stores the initial code-based world gen registry objects. This is only used when the game is first launched.
2. DynamicRegistries. All entries from WorldGenRegistries are copied to the DynamicRegistries during server startup. When the server has started the game will only look from the DynamicRegistries.

If you are making code-based world generation objects like Features or Structures, you MUST add them to **WorldGenRegistries** during post-registry construction events like FMLCommonSetupEvent, and in an enqueueWork lambda function to make this call thread-safe. 

If you do not do this you can prevent other mod's world generation objects from spawning.

**DO NOT** call DynamicRegistries before the server has started.

### Datapack Loading

Another behaviour change that came with data-driven registries is the datapack loading stage.

In the new environment, as soon as a world is created (i.e. Clicking the "Create World" button) vanilla reloads all datapacks 2-3 times, and recreates all data-driven registries each time.

This makes adding API hooks a bit tricky for modloaders, as one would need to experiment and find a good place to inject hooks.

On the user side, the repeated datapack reloading also has a side effect of making the game appear to be "slow" during world creation.

This change was likely done to ensure all datadriven objects are picked up and registered. Modders should not try to mess with these internal processes.


### Biomes

#### Biome Creation

Biomes should be created using JSON files. This is due to the Biome registry being a data-driven registry so non dynamic registries like ForgeRegistries are not longer suitable for it.

Code-based Biomes are still possible, but not recommended.

It is not recommended to do code-based biomes in general because of amount of potential stability issues you can face using these workarounds.

One exception for code-based biome objects is for making dummy objects to take up registry IDs which datapack jsons can use to replace the code-based ones later.
  
#### Biome Registration
You no longer use the Forge Registry for new Biomes. It is recommended to use datapack jsons to do this.

Make a json file named with the registry name of your biome in ``data/your_modid/worldgen/biome``

E.g. ``data/modid/worldgen/biome/my_test_biome.json``

Vanilla will automatically pickup and register the json during datapacks loading.

#### Biome Modification (Forge)

If you want to add custom objects such as ConfiguredStructures, ConfiguredFeatures to existing vanilla/other mod biomes, you must now use the BiomeLoadingEvent provided by Forge.
This is an event that fires during datapack parsing, when the Biome JSONs are being read, but BEFORE the MinecraftServer has started.

E.g. We want to add a ConfiguredStructureFeature to an existing biome.
1. Make a code based version of your world gen object such as a code-based ConfiguredStructureFeature.
2. Register the code-based object to WorldGenRegistries during FMLCommonSetupEvent in an enqueueWork lambda function to make this call thread-safe.
3. Add the code-based object to the BiomeLoadingEvent provided by Forge.

In the event, to check for a specific biome, make a RegistryKey<Biome> instance using the event's biome registry name, and compare this registry key to another registry key.

e.g. 
```
@SubscribeEvent(priority = EventPriority.HIGH)
public static void onBiomeLoad(BiomeLoadingEvent event) 
{
    RegistryKey<Biome> eventBiomeKey = RegistryKey.get(Registry.BIOME_REGISTRY, event.getName());
    RegistryKey<Biome> targetBiomeKey = RegistryKey.get(Registry.BIOME_REGISTRY, new ResourceLocation("minecraft", "plains"));
    if (eventBiomeKey == targetBiomeKey)
    {
	    //Do work here
    }
}
```

#### Limitations
At the moment, users cannot use datapacks to modify the code-based world generation objects that you added to the BiomeLoadingEvent.

Because the event fires during datapack parsing, this means the object in the BiomeLoadingEvent always overrides that of objects in external datapack as the external datapack could be read after the event has finished.

Instead, a work around is to add options to Forge's COMMON Config Specification. 

This is because the BiomeLoadingEvent fires before config types, such as SERVER config are not initialised yet.
Hence, the Common config is the ONLY config type that is available before the server has started.

This is set to change once Forge adds an event that is injected in a better spot to allow datapacks to modify objects in the event.

#### Adding Biomes to Worlds
To add biomes to the Overworld, you will need to make a RegistryKey instance for your Biome.
Then, call BiomeManager#addAdditionalOverworldBiomes in FMLCommonSetupEvent in an enqueueWork lambda function to make this call thread-safe. 
You can use the RegistryKey you made earlier in this method.

E.g.
```

public static final RegistryKey<Biome> testBiomeKey = RegistryKey.get(Registry.BIOME_REGISTRY, new ResourceLocation(MODID, "test_biome));

public void commonSetup(FMLCommonSetupEvent event)
{
    event.enqueueWork(() -> {
        BiomeManager.addAdditionalOverworldBiomes(testBiomeKey);
    });
}
```

The data driven systems are still a bit over the place in some areas so some of these workarounds are necessary.

To add biomes to your custom dimensions, you can add the biome's registry name to the dimension json. i.e. ``"modid:my_biome"``

#### Biome Properties

A biome's properties have also been changed significantly in this update.

Some of these changes, documented by community member SuperCoder797, is as follows:
- Biomes' water color and water fog color parameters have now been moved to a new class called BiomeAmbience, where you can now specify other things as well, such as particles, fog color, and sounds.
- Fog color specified in the effects will only work in the Nether. It won't work in the Overworld.
- You can now have particles spawn in biomes, this works for both the nether and the overworld. You can make some cool stuff with this, like flying sand in desert biomes.
- Sounds can also be specified per biome, and you can specify 3 different sound types.
- Loop sounds are continously played when a player is in the biome (useful for ocean waves on beaches).
- Mood sounds are played when the player is in an area with 0 sky light and less than 7 light. Overworld biomes use the cave sounds as mood sounds while the nether biomes have their own unique variants.
- Additions sounds have a 1.1% chance to play every tick. The nether biomes use these to add to their atmosphere, but you can probably use them for other stuff in the overworld, like bird sounds or something.
- Now have a new field for BiomeContainer, which are theoretical points on a 4d plane used for biome calculation in the Nether.


#### Referencing a Biome (Forge)

In this new environment the Biome's ``getRegistryName`` method added by IForgeRegistry CAN NOW BE NULL.

When getting an instance of a biome, always use a RegistryKey to get the value from the Biome Registry in the DynamicRegistries.

Do NOT use the ForgeRegistry's Biome registry because that registry does not account for any biomes added via datapacks.

#### Biome ID Shifting

Mojang changed the internal ID of biomes from string back to an integer (Who knows why). 

This has led to issues where changing the world generation objects in datapacks on an existing save, can change the ID of an existing biome.

Some symptoms of this issue can include blocks, structures and features from other biomes generating in the same location as the old one, changes to biome weather patterns etc.

This has not been fixed in Vanilla as of 1.16.5, but has been fixed in Forge.

#### Missing Forge Biome Hooks

The new Biome logic and structural changes has necessitated the removal of several Forge Biome hooks previously available in 1.15.2 Forge:
- Edge Biome Hook
- Ocean Hook
- Hills Biome Hook

Additionally, there are still some missing modding functionality in the new biome system which you should be aware of:

- Custom Nether Biome Hook
- End Biome Hook
- River Biome Hook

### ChunkGenerator

When using a custom ChunkGenerator in your custom dimension, it is recommended that you reference custom ChunkGenerators in a Dimension JSON file rather than trying to construct a code based Dimension. 

It seems a code based ChunkGenerator is still necessary for now to define the actual generation logic, but vanilla is trying to move them to become data-driven in the future too.

To register the ChunkGenerator, register it to WorldGenRegistries and call it in FMLCommonSetupEvent within an enqueueWork lambda.

You will also notice that a Codec is required as part of the ChunkGenerator. See Vanilla and existing mods for examples.

#### Special Cases
Some edge cases require special handling. 

A notable example is the SingleBiomeProvider.

For example, if your ChunkGenerator uses the SingleBiomeProvider, you may notice that the SingleBiomeProvider does not properly get biomes from the ``Biome.CODEC``. 
SingleBiomeProvider also contains a bug that doesn't allow Biome colours to be sent the client properly.

One workaround is to use a MultiBiomeProvider and supply a single biome to it, as MultiBiomeProvider does not have those problems.


### Entity Spawns

In Forge environments, these are now handled via the BiomeLoadingEvent.

### Structures

Structures are now more complex to register and add to existing biomes.
It also needs to be registered to WorldGenRegistries to prevent it from overriding other modded structures.

#### Registration

Registering a Structure requires you to add it to a number of different objects.

Additionally, some of the new backend changes have become more hardcoded in approach, which requires modders to use work arounds.

- ConfiguredStructureFeature(s) MUST be registered to the WorldGenRegistries after registry events have fired. If this is not done your structure will prevent other mod structures from spawning.
- Structures must be added to ``Structure.STRUCTURES_REGISTRY`` so the game is aware of the structure when it is used in other contexts. This also ensures the /locate command will list your structure.
- We also need to add to ``FlatGenerationSettings.STRUCTURES`` to prevent any sort of crash or issue with other mod's custom ChunkGenerators.
- The StructureSeparationSettings also needs to added to a world's ChunkGenerator to allow the chunk generator to know how many chunks it should iterate over before spawning the structure. Without this, the structure simply won't generate.
- However, if mods want to change the StructureSeparationSettings dynamically, they must construct the seperation settings while a world is loading, because a chunk generator won't know which world it is binded to.

A community member, TelepathicGrunt has kindly shared a working solution that shows <a href="https://github.com/TelepathicGrunt/StructureTutorialMod">how to register a structure in the Forge Environment</a>.

The summary of their solution is as follows

1. In the Mod Constructor Register a ``Structure<?>`` instance to the ``STRUCTURE_FEATURES`` Forge registry so the registry IDs are safely taken up.
2. In FMLCommonSetupEvent, add the structure to ``Structure.STRUCTURES_REGISTRY`` in an enqueueWork lambda. This is necessary to allow MC to know about the structure and to allow the /locate command to work for this structure.
3. In FMLCommonSetupEvent, add the ConfiguredStructureFeature to WorldGenRegistries in an enqueueWork lambda to prevent it from overriding other modded structures.
4. In FMLCommonSetupEvent, add a StructureSeparationSettings to ``DimensionStructuresSettings.DEFAULTS`` to allow the structure's seperation settings to be registered. (This field is an immutable map so it needs some non-api modding methods like Mixins/AccessTransformers/Reflection to access it).
5. In FMLCommonSetupEvent, add the ConfiguredStructureFeature instance to ``FlatGenerationSettings.STRUCTURES`` to prevent any sort of crash or issue with other mod's custom ChunkGenerators.
6. (Optional) If we want the structure to seamlessly merge with terrain like Villages, add the structure to ``Structure.NOISE_AFFECTING_FEATURES`` in FMLCommonSetupEvent.
7. In the WorldEvent.Load event, add the structure to ChunkGenerator of the world you want to add the structure to. This allows modders to blacklist structures from spawning in specific worlds, as well allowing users to configure the StructureSeparationSettings via Config files.
8. In the BiomeLoadingEvent, add the ConfiguredStructureFeature instance to our biome of choice if we want to structure to spawn in an existing biome.

Please note that this approach does not use proper Forge API hooks as the hooks for proper structure registration do not exist yet.

### Features

Features need to be code-based if we want to add them to existing biomes.
It also needs to be registered to WorldGenRegistries to prevent it from overriding other modded features.

1. In the Mod Constructor Register a Feature<?> instance to the ``FEATURES`` Forge registry so the registry IDs are safely taken up.
2. In FMLCommonSetupEvent, add the ConfiguredFeature to WorldGenRegistries in an enqueueWork lambda to prevent it from overriding other modded structures.
3. In the BiomeLoadingEvent, add the ConfiguredFeature instance to the biome if we want to feature to spawn in an existing biome.

### Tree Generation

This has been refactored once again.

It has now been flattened and split into ``FoliagePlacer``s, ``TrunkPlacer``s, and ``FeatureSize``s.

A summarised proccess of generating is as follows (referenced from SuperCoder7979):
1. First, the tree's FeatureSize checks if it can spawn in a given position.
2. TrunkPlacerss then generate the trunk and then generate places for leaves to generate through ``FoliageAttachment``s.
3. For each FoliageAttachment, the tree's FoliagePlacer generates its leaves.
4. If you have FoliagePlacers from 1.15, it is recommended that those are rewritten entirely.

## Dimensions/Worlds

### Dimension
Dimensions are now data-driven and have significantly changed in its design.

- Dimensions are now more of a set of extended settings for a World. It pairs a ChunkGenerator and DimensionType togethor. (World = Level and Dimension = LevelStem in Mojang mappings)
- Dimensions now have a One to Many Relationship with a DimensionType, meaning you can now have many Dimensions using the same DimensionType.
- Dimensions now have a One to One relationship with a World, meaning the World itself is the instance of a Dimension.

Forge's DimensionManager class has also been removed due to it being obselete in this new environment.

#### Registration

Custom dimensions/worlds should no longer be made in code except for special cases like dynamically created dimensions during runtime.

Create a JSON file in ``data/modid/dimension/`` to register and define your dimension.

Vanilla will automatically pickup and register this dimension/world during datapack loading whilst also creating the world for the data-driven dimension during world creation.

#### Referencing a Dimension

A Dimension/World is now referenced via a ``RegistryKey<World>`` instance instead of the DimensionType.

To get an instance of ``RegistryKey<World>``, use ``RegistryKey.get(Registry.DIMENSION_REGISTRY, new ResourceLocation(MODID, "registry_name_here"))``

To access a ``RegistryKey<World>`` from a world, use ``World#dimension``.

To get a World on the server, use MinecraftServer#getWorld, which takes a ``RegistryKey<World>`` as the parameter.

While ``RegistryKey<Dimension>`` may also seem correct, it is merely for internal use.

Use ``RegistryKey<World>`` for most implementations.

#### Getting the Dimension Registry (Vanilla Internal Use Only)
While the dimension registry is data-driven like other world generation objects, it is NOT in the DynamicRegistries like the others.

Instead it resides in the MinecraftServer's level.dat file. In code this is a field in WorldGenSettings (MCP class name)

To get the dimension registry from the server you would call MinecraftServer#worldData#worldGenSettings#dimensions

Do not get the dimension registry unless you are registering a dimension in code for special cases such as dynamic dimensions.

If you want to get a dimension by registry key, use ServerWorld#getLevel instead.

### DimensionType
These are no longer the unique instance of a dimension. It is now merely a set of settings that can configure additional properties of a dimension.

You can make multiple dimensions use the same dimension type.

#### Registration

To make a custom DimensionType, use a JSON file add it to ``data/modid/dimension_type``.

### DimensionRenderInfo

This is a rendering object bound to a DimensionType. You can modify a DimensionType's Sky Box, Weather rendering and Cloud rendering here, Sky colour, Fog type and Light level here.

#### Registration

Vanilla has hardcoded this feature to only work for vanilla dimensions.

You need to use some non-api methods such as Mixins/AccessTransformer/Reflection to access the DimensionRenderInfo's private immutable map.

Then, in FMLClientSetupEvent, in an enqueueWork lambda, put a new entry into the DimensionRenderInfo map to register your custom DimensionRenderInfo.

To make a custom Dimension Type use your DimensionRenderInfo, add the registry name of your DimensionRenderInfo to the "effects" field of your DimensionType.

### Quality of Life Features
#### Lightmap Modification
You can no longer modify the Lightmap for a Dimension/World in 1.16, which was previously provided by Forge in 1.15 and below. This is because the changes to dimensions meant Forge's patches were not suitable.

Examples of lightmap modification include changing the hue colour of the world or darkening the light in a dimension.

This feature may be restored once a suitable solution is provided to Forge.
## Rendering

Most rendering methods now include a MatrixStack in their parameters.

### GUI
GUIs are an exception in which some methods still use old ``RenderSystem`` and ``GlStateManager`` calls.

This is due to when/how the buffer is drawn to the screen, so systems like RenderSystem have become a temporary option until vanilla fully migrates their rendering engine to the batched style.

## Item Properties

These are now a static builder function you call during FMLClientSetupEvent.
Use ItemProperties#register. The supplier in that method takes a LivingEntity, ClientWorld and ItemStack.

## Command Registration

Register your commands in Forge's RegisterCommandsEvent.

## Reload Listener Registration

Register your ReloadListener(s) in Forge's AddReloadListenerEvent event. This event fires when datapacks are reloaded. 

Datapacks will reload at least twice in total during the game's loading process.

It will be fired at least once before the server has started, so if you are running syncing packets in your reload listener, be sure to check if the MinecraftServer is not null via ``ServerLifecycleHooks#getCurrentServer`` before running your sync packet code.

## Entity Attribute Creation and Modification

Entity Attributes are no longer a method you inherit from the Entity class.

Internally, it is now created by using a builder function to add attributes to the GlobalEntityTypeAttributes map.

For Forge 1.16.5 mods, you should be using the dedicated events:
 - EntityAttributeCreationEvent - For adding attributes to your custom entities.
 - EntityAttributeModificationEvent - To add attributes to existing vanilla/other mod entity types.

## Networking

### Registries data
The introduction of data-driven registries has led to changes in the data the player receives

The player is now sent its own set of registry data via a packet on login:
- The client now stores its own list of Dimensions (Worlds).
- The client holds its own set of DynamicRegistries. As mentioned earlier, not all registries on the server are available on the client - E.g. ConfiguredFeature is not on the client.


## Miscellaneous

### Multipart Entity hitboxes
There is now an <a href="https://github.com/MinecraftForge/MinecraftForge/pull/7554"> Forge API hook </a> that allows you to register multiple hitboxes for your entity, similar to the Ender Dragon.

### Forge ChunkManager
The Forge ChunkManager from older versions has been readded to Forge.

This allows modders to control their own ChunkTickets for their Force Chunk Loading implementations. 

Additionally, ticking and non-ticking tickets can be created.

Chunk ticks(weather, mob spawning, random block ticks) will be fired when the ticking tickets are created.

### Deprecated Forge-specific Config and Commands

Some Forge utilities have been deprecated in 1.16, many of which relate to world generation related objects becoming data driven.

These include:

- Deprecation of /forge setdimension command in favour of the vanilla /execute in command.
- forgeCloudsEnabled - Leftover artifact from the old dimension system


## Forge API

### EventBusSubscriber Annotation and Mod IDs
There has been a newly discovered bug in the ``@EventBusSubscriber`` annotation where the Mod ID that is firing the event is no longer known unless the class that this annotation is used on, also has a ``@Mod`` annotation.

This bug will affect all mods which use a specialised class for their events.

To fix this issue, please add your Mod ID to the ``modid=`` field in the annotation to ensure your event fires for your mod.

Even when the fix for this bug is merged in, it is still recommended to adopt this practice to reduce similar issues in the future.

E.g.
```
@Mod.EventBusSubscriber(modid = MyMod.MODID) //Add your Mod ID to the modid field
public class ForgeEventHandler{

}
```

### Mods.TOML License
In 1.16.5+ all mods are now required to define the "license" field in their ``mods.toml`` file.

This license is to allow modders to define how their mod and its assets can be used by others, such as one from https://choosealicense.com/.

The value can either be the name of the license, or a URL link to a custom license the modder defines.

E.g. https://github.com/ExampleModderName/MyExampleModRepo/blob/1.16/LICENSE

### Mojang Mappings

#### About

Mojang Mappings (nicknamed Mojmaps by modders) are the official obsfucation mappings provided by the developers of Minecraft, Mojang.

It is a bridging tool to turn Mojang's obsfucated Minecraft code into human readable names.

E.g. func_1234_a_ -> setupHelloWorld

#### Purpose and Scope

These mappings cover all methods, fields, class names, but NOT parameter names or Javadocs.

These mappings are revolutionary for mods as it means modders can see the Minecraft code just like the Mojang developers.

This makes it alot easier to understand when referencing it as an example.

#### Forge's Adoption

**In the Forge API**

In 1.16.5, Forge has decided to adopt the use of Mojang's Official Mappings in their Mod Developer Kits (MDKs) by default.

All methods and fields will use Mojang Mappings.

Class names still use MCP names for backward compatibility reasons. (If Mojang class names were used in 1.16.5 Forge, all existing mods will break)

In 1.17 Forge intends to fully transition their class names to Mojang Mappings too.


#### Legal Concerns

Previously, the legal terms for these mappings were in a state where Forge did not feel comfortable using them.

While there still isn't a guarantee that the use of these mappings are legally safe, Forge has now decided to adopt them in good faith that Mojang wants them to use it.

Read more about <a href="https://github.com/MinecraftForge/MCPConfig/blob/master/Mojang.md">Forge's stance here.</a>

#### Pros and Cons

[+] ALL methods and fields have human readable names that show the ORIGINAL logic Mojang intended.

[-] Lack of parameter names and Javadocs. Parameter names and Javadocs are only available in crowdsourced tools like MCP or Yarn. Forge is working with various projects to re-add parameter names.

#### Usage

##### New Mods
In the latest 1.16.5 Forge Mod Developer Kits (MDKs) Mojang mappings are automatically used.

##### Existing Mods
You can still use the MCP names if you so wish, there are updated MCP mappings made by the community, but these exports are not done very often.

If you choose to upgrade to Mojang Mappings, there is a handy ```upgradeMappings``` Gradle command that can convert existing mapping names to Mojang names, and vice versa.

A summary of updating existing mods to Mojang Mappings is as follows (Referenced from Forge Discord, !updateMappings Bot command):
1. Make a backup! If you're not already using some form of Version Control System (VCS) or have uncommitted changes, it's important to make a backup. The steps outlined below do not backup your files, and they irreversibly change them. Be warned. Note that you can switch back mappings at any time, but backups are still not made. 
2. Run ``gradlew -PUPDATE_MAPPINGS_CHANNEL="official" -PUPDATE_MAPPINGS="1.16.5" updateMappings`` in a terminal in your mod's project directory. Prepend ./ if you're using a Unix-based system. 
3. Wait for the process to finish. 
4. Update your mappings in your build.gradle and/or gradle.properties file and change the mappings line to match something similar to this effect: ``mappings channel: "official", version: "1.16.5"`` 
5. Refresh or reimport your Gradle project. 
6. Done! Please note that there are still some bugs associated with changing your mappings. Make sure to try building your mod project or running it to see if there are any compilation errors and fix them. 

Note: You can run !updatemappings <mappings channel> to get help switching to another channel. ﻿

Read <a href="https://gist.github.com/JDLogic/bf16deed3bcf99bd9e1a22eb21148389">more about the updateMappings command here. </a> 

### Mixins in Forge!
#### About
Mixins are a powerful and lightweight third party modding tool developed by the organisation SpongeForge (NOT affiliated with Forge). 

It allows modders to edit parts of the core Minecraft code that is slightly less invasive than traditional Javascript Core Mods.

In 1.16+, Forge now natively supports the use of Mixins.

#### Usage

Mixins should be considered a LAST RESORT tool for modders, as they are still considered a form of coremod, which is invasive and can still cause incompatibilities and conflicts.

It is recommended you continue to use Forge's APIs and hooks where possible.

If there is no current API that supports your usecase, consider contributing one to Forge, so that other modders can benefit from it, and reduce mod incompatibilities.

For large, invasive patches, Mixins are a good temporary solution, but in the Forge environment it still recommended that the patches are made available to Forge. :)

For smaller patches, or patches that have more niche usecases, Forge tends to accept them if enough evidence of support for the patch is shown.

### Forge Gradle and Maven
Due to unforseen issues, the maven location for Forge's files has changed.

In existing projects, please update all instances of maven url in your build.gradle file.

New MDKs will have the new maven location prepared.

### Java 15 support and Gradle 6.8 Support

Forge nows supports Java 15 and Gradle 6.8.

You must only use Java 8 code as Minecraft is built on Java 8. 

It is still recommended to compile against Java 8.

### FMLConstructModEvent - INTERNAL FORGE USE ONLY

This is fired when the mod's constructor has been called but just before mod registry events (such as block registry, item registry etc.) are fired .

This is for internal Forge use only.

DO NOT use this event for your mods.

## References

- <a href="https://gist.github.com/SuperCoder7979/511e038714fb5f4fb59c06a8aa6c0281"> SuperCoder7979's 1.16 RC primer (SuperCoder7979, Accessed 2021)</a>
- <a href="https://github.com/TelepathicGrunt/StructureTutorialMod"> TelepathicGrunt's Structure Tutorial Mod (TelepathicGrunt, Accessed 2021)</a>
- <a href="https://discord.gg/UvedJ9m"> Official Minecraft Forge Discord (MinecraftForge, Accessed 2021)</a>

*Sourced from https://gist.github.com/50ap5ud5/f4e70f0e8faeddcfde6b4b1df70f83b8*
