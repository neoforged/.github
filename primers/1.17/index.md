# Minecraft 1.16.5 -> 1.17 Modding Migration Primer

This is a high level and non-exhaustive overview of the 1.16.5 to 1.17 migration primer from a mostly Forge oriented perspective. 

This primer is licensed under Creative Commons, so feel free to use it as a reference.

Please note that this can be updated, so leave a [link to this file](https://gist.github.com/50ap5ud5/beebcf056cbdd3c922cc8993689428f4) so readers can see the updated information themselves.

If there's any incorrect or missing information please leave a comment down below. Thanks!

## Debug Profiler
The /debug command now does nothing, and can be executed through F3 + L for at most 10 seconds

It now provides a lot more detail besides tick profiling including system data, jvm data, game settings, level data, and any deviations from normal behavior.

## Tile Entities
The TileEntity is now named BlockEntity with Forge moving to using Mojang mapping names.
### TileEntityProvider (now EntityBlock)
TileEntityProvider is now called EntityBlock.

The following are some of the new methods you need to register new TileEntities (BlockEntities).
1.  Implement `BaseEntityBlock` or `EntityBlock` on your Block class
2. Override `EntityBlock#newBlockEntity` and return a new instance of your block entity

E.g.
```
public class BlockEntityEnabledBlock extends BaseEntityBlock {

    public BlockEntityEnabledBlock(BlockBehaviour.Properties properties) {
        super(properties);
    }
    
    @Override
    public BlockEntity newBlockEntity(BlockPos pos, BlockState state) {
        return new MyModBlockEntity(pos, state);
    }

}
```

### Ticking Block Entities
#### BlockEntityTicker
A new object that replaces the ITickableTileEntity interface. This allows you to separate ticking logic from the logical side.

E.g. Vanilla chests only have a client side Ticker for their lid open animation, and have no need for a server side one.

Basic steps to make a ticking block entity:

1. Implement `BaseEntityBlock` on your Block Class
2. Create a static method in your`` BlockEntity`` (TileEntity) class to handle the ticking logic for the server.

If you need separate ticking logic on the client, create another static method to handle that logic.

3. Override `EntityBlock#getTicker` and return either your client side or server side Ticker depending on the logical side. 

To return the `BlockEntityTicker` instance, call `BaseEntityBlock$createTickHelper` in the abstract block entity block to create an instance of your Ticker. You can then call the static tick method from your BlockEntity class.

The logical side check can done with a ``Level#isClientSide`` check as normal.

E.g. Example below shows us creating a Block class which has a `BlockEntityTicker` for the server side only.

```
public class BlockEntityEnabledBlock extends BaseEntityBlock {

    public BlockEntityEnabledBlock(BlockBehaviour.Properties properties) {
        super(properties);
    }
    
    @Override
    public BlockEntity newBlockEntity(BlockPos pos, BlockState state) {
        return new MyModBlockEntity(pos, state);
    }
   
    @Override
    @Nullable
    public <T extends BlockEntity> BlockEntityTicker<T> getTicker(Level level, BlockState       state, BlockEntityType<T> type) {
        return level.isClientSide() ? null : createTickerHelper(type, ModBlockEntityType.MY_BLOCK_ENTITY.get(), MyModBlockEntity::serverTick);
    }

}
```

## TileEntityRenderer
In the Forge environment, you need to Subscribe to the ``EntityRenderersEvent`` event.

- Call ``EntityRenderersEvent$RegisterRenderers#registerBlockEntityRenderer`` to register a BlockEntityRenderer
- You now need a `BlockEntityRendererProvider$Context` instance as a parameter in the constructor of your BlockEntityRenderer class.
- You now need a `BlockEntityRendererProvider` instance in the ``registerBlockEntityRenderer`` method, which takes in the ``BlockEntityRendererProvider$Context`` from your BlockEntityRenderer class.



## Rendering
### Shader-based Rendering
Minecraft has moved to OpenGL 3.2 Core, which brings with it shader-based rendering.

 Any `bind` calls in `RenderSystem` should now move to using `RenderSystem#setShaderTexture(0, resourceLocation)` instead. 

Setting the color values has changed from `RenderSystem#color` to `RenderSystem#setShaderColor`. 

Shaders can be used by calling `RenderSystem#setShader` and provide a method reference, usually from `GameRenderer`. 

Shaders are loaded from json and are now used across vanilla and in Forge 1.17. 

For more examples, see vanilla rendering.
### ClampedItemPropertyFunction
Item properties are forced in vanilla to be clamped between 0 and 1. 

Forge has patched that out to allow unclamped values to be set (though you could’ve just extended the interface and overridden call as well to remove it yourself).

ItemPropertyFunction now takes in a fourth parameter which represents the id of the entity which is holding the item.
### Entity Models
The Entity Model format has changed a lot. 
Model information is no longer stored in the constructor and has been heavily abstracted.

There is evidence in the codebase that Mojang plans to allow Entity Models to be modified through JSON files, though this feature has not been fully implemented in 1.17 yet.

Definitions are used to create the specification of the model to which is then baked by the other components.

#### New Objects

There are now new objects in Entity Models

**ModelPart**
- The ModelRenderer or the group/bone
- Can be multiple parts within a model, and the parts can contain multiple cubes.
- You can also have each part have childs of other parts meaning that rotation and position applied to this part will also be applied to other parts in a relative fashion

**ModelPart$Cube**
- The boxes inside the group/bone
- Just a collections of polygons and the min and max positions

**PartPose**
- Position and rotation of a ModelPart, nothing really else of note

**CubeDefinition**
- The definition of a cube, when baked creates a ModelPart$Cube

**CubeListBuilder**
- Holds the cube definitions for a particular defined PartDefinition
- Comparison to ModelPart holding ModelPart$Cubes

**MeshDefinition**
- The definition of the model, holds all data associated with it

**LayerDefinition**
- The definition of the model + texture
- Needs to be registered during ``EntityRenderersEvent$RegisterLayerDefinitions``

**MaterialDefinition**
- The definition of the texture data, specifically the size of the texture

**PartDefinition**
- The definition of the model parts, when baked creates a ModelPart

**CubeDeformation**
- Represents a cube inflation in x,y,z direction separately
- Only increases the max value and then realigned, doesn’t increase on both sides
- See armor models for an example of cube inflation

#### Registration
37.0.8 added events for registering this data: EntityRenderersEvent
- $RegisterLayerDefinitions, for registering the model layers itself
- $RegisterRenderers, for registering the entity renderer
- $AddLayers, for adding layers to a specific entity renderer

You can refer to Forge’s <a href="https://github.com/MinecraftForge/MinecraftForge/blob/1.17.x/src/test/java/net/minecraftforge/debug/client/rendering/EntityRendererEventsTest.java">EntityRendererEventTest</a> for examples.

Additional community guides can be seen here:
- <a href="https://gist.github.com/gigaherz/7115024820f55717bc40a6e2247c6aca">Giga’s Explanation</a>
- <a href="https://github.com/SizableShrimp/EntityModelJson/blob/1.17.x/docs/SCHEMA.json">Shrimp’s Json Spec<a> (jsons for Entity Models aren’t available yet, Shrimp just did some workaround for it)
## World Generation
Most of the core foundations from 1.16 have remained mostly the same. The main changes relate to the technical foundations in preparations for 1.18 Caves and Cliffs generation.
### World Height Changes
The default world height from 0-256 has been un-hardcoded. 

### LevelHeightAccessor
An interface that allows for world height context to be gotten. This is a parameter for many world gen objects like Features and Structures.

ServerLevel (MCP: ServerWorld) and ClientLevel (MCP: ClientWorld) now inherit this interface.

### ProtoChunks
(MCP: ChunkPrimer)

Now have a LevelHeightAccessor context.
### Configs and Placements
Configs and Placements have changed in syntax and behaviour in many ways due to preparation for 1.18

For example, height related configs now separate their “top" and “bottom" offset configurations into new, separate objects.

### Aquifers
A new world gen object type that allows for a pool of water to be held within a “cup" of blocks.
### Features
Remain mostly the same as 1.16.5. Parameters for the generate method are combined into a single wrapper object.

### Carvers 
Parameters for the generate method are combined into a single wrapper object.

### Chunk Generators
ChunkGenerators now have a BaseStoneSource method. 

ChunkGenerators now have a Aquifer method 

Noise Step/Status of Chunk Generation now returns a CompletableFuture, it is possible this is multithreaded.
### Structures

Structures have now got a new parameter that can be passed into a chunk generator’s heightmap method to get the terrain height or a column of blocks

### Surface Builders
Surface builders have a new Min Surface height parameter
## Game Events
In preparation for the 1.18 SculkSensor block which emits redstone triggered by nearby sound events, Mojang has now added their own “Event" system.

Currently this is limited to vanilla use cases, such as block placement, entity movement etc.

It does not appear to be moddable out of the box.
## Game Test Framework
Mojang has recently exposed their game test framework, a set of tools which the Mojang developers use to simulate and test game mechanics.

- Example usage: https://www.youtube.com/watch?v=vXaWOJTCYNg 

Currently there is no API to use it as part of mods, but there are PRs being made to make it usable for modders.
- <a href="https://github.com/MinecraftForge/MinecraftForge/pull/8024">PR 8024</a>

All the components are under the ``net.minecraft.gametest.framework`` package.

## Forge API
### Full Migration to Mojang Mappings
In 1.17 Forge fully migrated its codebase to Mojang’s Official Obfuscation Mappings. 

Previously, only methods and fields were migrated.

This time, Class names will be using Mojang’s class names.

Parameter names are not included in the Mojang Mappings, but there are plans to add a Forge Gradle feature to add crowdsourced parameter names on top of Mojang Mappings. 

Examples of crowdsourced mappings include the Parchment Project, a popular choice in the Forge environment, though there are others.
#### Migration of Method and field names to Mojang Mappings
Migrating your method and field names to mojang mappings is similar to the 1.16 process.

A summary of updating existing mods to Mojang Mappings is as follows (Referenced from Forge Discord, !updateMappings Bot command): 

Make a backup! If you're not already using some form of Version Control System (VCS) or have uncommitted changes, it's important to make a backup. 

The steps outlined below do not backup your files, and they irreversibly change them. Be warned. Note that you can switch back mappings at any time, but backups are still not made. 

1. Run ``gradlew -PUPDATE_MAPPINGS_CHANNEL="official" -PUPDATE_MAPPINGS="1.16.5"`` updateMappings in a terminal in your mod's project directory. 
2. Prepend ``./`` if you're using a Unix-based system. 
3. Wait for the process to finish. 
4. Update your mappings in your `build.gradle` and/or `gradle.properties` file and change the mappings line to match something similar to this effect: ``mappings channel: "official", version: "1.16.5"``
5. Refresh or reimport your Gradle project. 
6. Done! 

Please note that there are still some bugs associated with changing your mappings. Make sure to try building your mod project or running it to see if there are any compilation errors and fix them.

**Note:** You can run `!updatemappings` in the Forge Discord #bot-commands channel to get help switching to another mappings channel. ﻿ 

Read more about the <a href="https://gist.github.com/JDLogic/bf16deed3bcf99bd9e1a22eb21148389">updateMappings command here</a>.
#### Migration of Class Names to Mojang Mappings
To migrate class names to Mojang Mapping names, it is recommended you have compilable 1.16.5 code, then migrate to Mojang mappings.

The migration script was designed by Forge contributor, SizableShrimp.

See their detailed explanation of the system in their <a href="https://github.com/SizableShrimp/Forge-Class-Remapper">Github repo</a>.

There is also a simpler guide written by one of the Forge team members, Gigaherz.
- <a href="https://gist.github.com/gigaherz/6fc52ee532f36ec432db62458c1620b5">Gigaherz’s class name port steps</a>

### Separation of Client-Only Methods within Common Classes
The OnlyIn annotation has now been stripped from all Common code.
This means methods that used to be client only will no longer crash dedicated servers instantly.

Relevant PRs:
- <a href="https://github.com/MinecraftForge/MinecraftForge/pull/7773">PR 7773</a> Migrated into 1.17
- #initializeClient which can consume a custom client instance containing properties only available on the client.
- An anonymous class should be consumed

### Minecraft Authentication in Dev Environments
Mojang has made backend changes to their authentication system and Minecraft launcher in a way that breaks the previous way to authenticate Minecraft accounts in a Forge dev environment.

As a side note, any Minecraft accounts migrated to Microsoft accounts will no longer be able to use the --username and --password arguments in 1.16.5 environments.

The main things you need to know about how Forge’s changes will affect your mod are:

- You can no longer use --username and --password run arguments. 
- You must now only use the --accessToken argument, with the value being the access token for your Minecraft account. This approach is that it is far safer as it can expire, and limit the amount of access someone may have to your account.


Your runClient file should now have the following arguments:
- "--uuid", "UUIDWITHOUTDASHES"
- "--accessToken", "TOKEN"
- "--username", "USERNAME"
- "--userType", "mojang"

#### Getting Your Access Token

Due to a recent Minecraft Launcher update, account access tokens are no longer stored anywhere on your Minecraft instance (.minecraft) folder.

To get the access token, you will need to manually ping the Mojang authentication servers, make a mod that does that, or use a third party launcher.


If you choose to make a Forge mod that gets your account’s access token, you can do so by either using the Minecraft instance or Mojang’s Yggdrasil Authentication system directly.

The example code below was kindly shared by community member **vash** on the Minecraft Forge discord.

To use it, 

1. Setup a Forge workspace, add the necessary code, then compile it. 
2. Then place the mod in your regular Minecraft instance and load up Forge.
3. Check your latest log file and copy paste your access token to be used in your other mod development environments.
##### Approach 1 - Use Minecraft Instance

```
@Mod.EventBusSubscriber(modid = YourMod.MOD_ID, bus = Mod.EventBusSubscriber.Bus.MOD, value = Dist.CLIENT)
public class ClientEvents {

  @SubscribeEvent
  public static void onFMLClientSetupEvent(final FMLClientSetupEvent event) {
      System.out.println("ACCESS_TOKEN "+ Minecraft.getInstance().getUser().getAccessToken());
      System.out.println("UUID "+ Minecraft.getInstance().getUser().getUuid());
  }

}
```
##### Approach 2 - Use Mojang Authentication System

```
@Mod.EventBusSubscriber(modid = YourMod.MOD_ID, bus = Mod.EventBusSubscriber.Bus.MOD, value = Dist.CLIENT)
public class ClientEvents {

  @SubscribeEvent
  public static void onFMLClientSetupEvent(final FMLClientSetupEvent event) {
      var auth = new YggdrasilUserAuthentication(new YggdrasilAuthenticationService(Proxy.NO_PROXY, ""), "", Agent.MINECRAFT);
      auth.setUsername("username/email"); //Set your account’s Minecraft username or account email here
      auth.setPassword("password");
      try {
        auth.logIn();
        System.out.println("ACCESS_TOKEN " + auth.getAuthenticatedToken());
        System.out.println("UUID " + auth.getUserID());
      } catch (AuthenticationException e) {
        e.printStackTrace();
      }
      System.exit(0);
  }

}
```

### Extension Points
ExtensionPoints, an object that allows Forge mods to attach custom objects like Config GUIs or side specific markers, are now an interface called IExtensionPoint.

To use these, register through the class type and a supplier of the class type in the mod bus inside your main mod class constructor.


E.g. Registering an extension point to mark a client side only mod as client only so that when connecting to servers, it does not think it is required on servers and won’t show the red “cross" icon.
  ``ModLoadingContext.get().registerExtensionPoint(IExtensionPoint.DisplayTest.class, () -> new IExtensionPoint.DisplayTest(() -> FMLNetworkConstants.IGNORESERVERONLY, (incoming, isNetwork) -> true));``

### HUD Overlay API
A new HUD Overlay API has been added for 1.17 exclusively. The API does make some breaking changes, so it is not possible to backport to 1.16 in its entirety.

- <a href="https://github.com/MinecraftForge/MinecraftForge/pull/7770">PR 7770</a> Migrated into 1.17

This PR adds in OverlayRegistry to register IngameOverlay for an ElementType#Layer
- Four methods are added which can register the top, bottom, before, or after a particular overlay type, can be enabled or disabled
- All methods are synchronized
### Removal of position isAir Method
Previously Forge added a method to the Block class called isAir(World, Position) which intended to place the vanilla method of isAir() which had no parameters. 

In 1.16 and below this Forge hook was inconsistent in that it was only used in some patches, whilst other parts of the codebase still used the vanilla method.

This made it confusing for modders so the Forge-provided method isAir(World, Position) is being removed.

### Capabilities
#### CapabilityToken and @CapabilityInject Deprecation
Due to Java 16 backend changes and a request to move away from Java reflection, a new way of initialising Capability instances is being added to 1.17 Forge in the future.
- <a href="https://github.com/MinecraftForge/MinecraftForge/pull/8116"> See Pull Request 8116 for more information </a>.

**Previous Design**:
- One would use the ``@CapabilityInject`` annotation on a null field
```
@CapabilityInject(InterfaceForMyCapability.class)
public static void Capability<InterfaceForMyCapability> CAP_INSTANCE = null;
```

**New Design**
- Introduction of a CapabilityToken object, which can capture generic types, similar to Google’s TypeToken.
- Capability will now have a ``isRegistered`` function to replace sanity checks of ``!= null``.
- It also has a addListener(Consumer<Capability>) function, which is analogous too the old @CapabilityInject on a method.

Example use of new design:
```
public static Capability<IDataHolder> DATA_HOLDER_CAPABILITY = CapabilityManager.get(new CapabilityToken<>(){});
```
Note that we use an anonymous class. This forces the compiler to include the generic information we need.
We are not going to pass in a Class object because we need to keep the soft dependency system.

#### Removal of IStorage
The IStorage object, which was a default implementation of your Capability, has been removed, with no replacement. 

This was originally intended as an easy API hook for other mods to use your capability, but since barely any mods knew or used it properly, it has been removed.

For pre 1.17 mods that used Capabilities, move the code in the write and read methods to your ICapabilitySerializer implementation.

#### RegisterCapabilitiesEvent
As of <a href="https://github.com/MinecraftForge/MinecraftForge/pull/8021">PR 8021</a> there is now a `RegisterCapabilitiesEvent` that allows you to register capabilities properly.

Please subscribe to the `RegisterCapabilitiesEvent` and use ``RegisterCapabilitiesEvent#register`` for registering capabilities in the future.

#### Lazy Initialization of ItemStack Capabilities
Delays the call of capability initialisation to later for performance enhancement.

### Forge Event Changes

#### Custom Boat Models/textures 
Can now be applied by overriding BoatRenderer#getModelWithLocation
 
#### RegisterClientReloadListenersEvent 
A new event has been added to allow client reload listeners to be registered properly. I.e. Things you would put under the ``assets/modid`` folder.

#### OnDatapackSyncEvent
A new event for syncing datapack content on player login and reload command. This is also available in 1.16.5 Forge.
### Java Version, OpenGL and Forge Gradle Updates
As you may know, Minecraft 1.17 updated its Java version to Java 16 and OpenGL version from 2.1 to 3.2 Core.

**Java Version**

You must use Java 16 now. 

For modders, this means:
1. You need to update your Java JDK version to JDK 16.
2. You need to be using Forge Gradle version 5.0+.

**_Java 17 Support?_**

In light of the recent Java 17 release, no, Java 17 **_will not_** work on Forge dev environments. A recent patch to Forge prevents it from crashing your environment, but Forge does not officially support Java 17


**Open GL**

Everything in terms of shaders is done through RenderSystem now
- ``RenderSystem#setShader`` which includes vertex consumer data and old RenderType things

<a href="https://github.com/MinecraftForge/MinecraftForge/commit/80d08dbf3a8e0341962c23dc4737b9af289ef7de">37.0.15</a> added RegisterShadersEvent for registering new shaders
1. First param is a constructed instance of the the event param, the unique name of the shader, and the vertex format
2. Second param can be used to cache the instance on load
3. You can see <a href="https://github.com/MinecraftForge/MinecraftForge/blob/80d08dbf3a8e0341962c23dc4737b9af289ef7de/src/main/java/net/minecraftforge/client/ForgeHooksClient.java#L863-L881">ForgeHooksClient</a> for example on usage

See <a href="https://gist.github.com/gigaherz/b8756ff463541f07a644ef8f14cb10f5">Gigaherz’s Guide - How to use Shaders In 1.17<a>

### Removal of ToolType and Tool System Redesign

The patches Forge made to allow for blocks to have a “correct" tool (be mined faster by certain tools) has now been overhauled. All 1.17 mods will need to change their code to account for this new system.

This new system was designed by Forge team member Gigaherz. This is added through  <a href="https://github.com/MinecraftForge/MinecraftForge/pull/7970">PR 7970</a>, 

To use this new tool system, please see the <a href="https://gist.github.com/gigaherz/691f528a61f631af90c9426c076a298a"> guide written by Gigaherz </a>.

A brief summary of the system is as follows:
- Register a tier by ``TierSortingRegistry#registerTier``, must be initialized before specific item so best to do statically
- Create tier using ForgeTier

Specific actions a tool make take is done by registering a ToolAction via ``ToolAction#get``, defaults in ToolActions.
- ``Item#isCorrectToolForDrops``. This determines if block can be dropped 
- ``Item#getDestroySpeed``. Determines how fast the block can be mined
- ``Item#canPerformAction``. Checks if a particular tool can perform a specific action, currently unused in the vanilla system

There is a datapack override for modpack creators for overriding the sorting list for custom behavior, mods will define within the tier registering
## Potential Changes in Forge
### Fluids
Fluids are getting a complete overhaul in 1.17. 

Previously, Fluids were limited to water-like behaviour only due to Mojang hardcoding their water references in many parts of the codebase. Since patching all those references would be major changes, the Fluid Overhaul will only be considered in 1.17 and above.

You can keep track of the PR here: Fluid Overhaul <a href="https://github.com/MinecraftForge/MinecraftForge/pull/7992">PR 7992</a>

## References and Contributors
- Gigaherz - various guides and documentation for the 1.17 systems
- SizableShrimp - information about 1.17 Mapping migration
- CorgiTaco - World generation changes
- vash#6194 - Access Token getter example code, Forge Discord, 2021.
- ChampionAsh5357 - Helped write this guide.

*Sourced from https://gist.github.com/50ap5ud5/beebcf056cbdd3c922cc8993689428f4*
