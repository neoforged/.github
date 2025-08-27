# 1.12 -> 1.13/1.14 Update Primer

This primer is licensed under your choice of MIT or CC0, do whatever you want.

BUT do note that this can be updated, so leave a [link here](https://gist.github.com/williewillus/353c872bcf1a6ace9921189f6100d09a) so readers can see the updated information themselves.

1.13 and 1.14 are lumped together in this doc, you're on your own if you just want to go to 1.13 and not 1.14, for some reason.

## Things in Advance
* `ResourceLocation` now throw on non-snake-case names instead of silently lowercasing for you, so you probably should go and change all those string constants now. More precisely, domains must only contain alphanumeric lowercase, underscore (\_), dash (-), or dot (.). Paths have the same restrictions, but can also contain forward slashes (/).
* MOD LIFECYCLE EVENTS RUN IN PARALLEL. YOU NEED PROPER CONCURRENCY CONTROL IF YOU'RE COMMUNICATING TO OTHER MODS
  * In most cases, just use IMC and it should all be fine (tm)
  * Otherwise, go learn some threading or ask on discord
* Gradle info: https://gist.github.com/mcenderdragon/6c7af2daf6f72b0cadf0c63169a87583
* [MCP renames](https://github.com/ModCoderPack/MCPBot-Issues/tree/master/migrations)
  * There is a script [here](https://github.com/ModCoderPack/MCPBot-Issues/tree/master/bin) that can take the csv's from the folder above and generated IntelliJ migration mappings, allowing you to perform MCP renames with a few clicks across your whole project.
  * The csv for 1.14 isn't up yet, so for now use [this](https://gist.github.com/williewillus/2dfc945b7b7fdb69cc3ff830072d22fe)
  * Blocks.GRASS moved to Blocks.GRASS_BLOCK, with the 1-tall grass shrub taking its place. Check your codebase for bugs!
* Registry events are now fired on mod-specific event buses. Check the `bus` argument to `EventBusSubscriber` for more info
* slicedlime (mojang dev) has made a high-level overview of 1.14's technical changes: https://www.youtube.com/watch?v=D6P5BvItdoc

## Rendering Changes
* ModelLoader.setCustomModelResourceLocation has been removed since it is no longer needed due to the flattening.
  * Items are looked up using their own registry name, meaning `foomod:fooitem` will look in `assets/foomod/models/item/fooitem.json` by default
  * If you need something else, look into using a custom baked model in `ModelBakeEvent`.
* Statemappers are now gone, replaced with a function hardcoding the old default logic in `BlockModelShapes`
  * This is because for vanilla usecases, the flattening has completely removed any need they have for statemappers
  * For most mods, the flattening should also solve most of the cases you need a statemapper (for appending a suffix, just do it manually; for splitting on a variant, should not happen anymore due to flattening)
* Vanilla blockstate jsons are smarter now
  * Before: the variant string was expected in a very specific format: alphabetized list of propertyname=value pairs, based on the statemapper
  * Now: the string is split on "," and dynamically built into a `Predicate<IBlockState>`. This obsoletes statemapper ignoring since if you don't care about changing rendering for a given property, just don't specify it in the blockstate json
  * How overlapping predicates are handled is unknown, but from brief investigation it seems to be an error.
  * Side effect: the `"normal"` variant should now be named `""`, defining a predicate that matches all incoming `IBlockState`s
* Vanilla model jsons are slightly smarter now
  * `{}` is now valid and represents an empty model
  * Only setting `textures` without setting `elements` is now valid, useful for e.g. setting the break particle of a TESR
* The `textures/blocks` and `textures/items` folders have been renamed to `textures/block` and `textures/item`, matching `model/`. Various vanilla textures have been renamed for consistency.
  * Yes, this is automatable. Read up on `sed` and `grep`, and/or write a shell or Python script
  * Texture errors should make it obvious where the errors are (you DO keep your mods clean and you DO try not to spam the log with useless errors so you can see the real ones, right? ... right? ...)
* In blockstate jsons, the `block` subdirectory is no longer inferred.
  * In other words, in a vanilla blockstate json, `foo:bar` no longer points to `foo:models/block/bar`, but rather `foo:models/bar`. You must put the `block` back in by specifying `foo:block/bar` in the blockstate json
  * Yes, this is *also* automatable, stop complaining.
  * The benefit is that you can organize models in subdirectories other than `block` now.
* Custom `ItemMeshDefinition`s are gone as their functionality can be completely replaced with a custom baked model with custom `ItemOverrideList`
* LWJGL has been bumped to version 3.x. So you can now use anything we've missed from the last 2+ years of LWJGL3 development.
    * Mojang appears to have ditched Java `BufferedImage` in favour of LWJGL 3's stbimage bindings. This shouldn't affect most modders.
    * Keyboard.KEY_FOO => GLFW.GLFW_KEY_FOO

## Lang changes
* Lang files are now json. instead of key=value, now one large json object with every translation inside e.g. `{ "block.minecraft.dirt": "Dirt" }`
    * Before you even complain, tterrag has already made an online tool for you
    * https://tterrag.com/lang2json/
    * If you prefer running the code yourself, ichttt also made a tool: https://github.com/ichttt/MCLang2Json
* By default, blocks and item translation keys now use their registry name (replacing the colon with a dot), and the block prefix is now "block." instead of "tile.". E.g. `block.minecraft.dirt`, `item.minecraft.diamond`
    * Not as scriptable due to the flattening, but still doable
* ".name" is also no longer added to the end of the lang

## Data packs

* Move your advancements, functions, loot tables, recipes, and structures from assets/ to data/. 
* **assets/ should once again become a place where client-only resources live**. Anything needed server side should be found in data/ instead.
* It appears that Mojang has a general util method for walking a subdirectory of `data/`, while respecting datapack cascading/overriding, which will be great. I imagine you could move most of your configuration or machine recipes here, and gain the benefit of the cascading resource system.
   * That method is IResourceManager.getAllResourceLocations. Example usage [HERE](https://github.com/sinkillerj/ProjectE/blob/5efac2656a0f4fbe50f0030fb354940033092d2b/src/main/java/moze_intel/projecte/emc/mappers/customConversions/CustomConversionMapper.java#L72-L98)
   * **IMPORTANT**: If you do move some of your gameplay (e.g. machine recipes) to datapacks, be very wary of namespace collisions.
   * E.g. Mojang has already reserved `data/<domain>/recipes` across ALL domains for recipes participating in the vanilla recipe system, and `data/<domain>/structures` across ALL domains for structure NBT files. Your subfolder in the datapack probably needs to have your mod-id somewhere again. E.g. `data/botania/botania/petal_apothecary/` or `data/projecte/pe_custom_conversions`, such that an addon or modpack can add to `data/my_addon/botania/petal_apothecary/` or `data/my_modpack/pe_custom_conversions`
   * So if you aren't careful and put something unrelated in `data/yourmod/recipes`, vanilla will try to load it as a vanilla recipe JSON
* See https://minecraft.gamepedia.com/Data_pack for more information

## What's the deal with metadata?

* Metadata has been removed. All of it. No more magic numbers!
* Information formerly expressed using metadata is either no longer needed (blocks), flattened (blocks and items), or moved to NBT (item tool damage).
* STOP BEING SCARED of using more block and item ID's. They are virtually **unlimited** now. (Well, the limit is how many `Block` and `Item` instances you can hold in memory at once. The answer is: millions of them. Blocks and Items are tiny). Of course, there are things that should still stay as variants of a single Block or Item (or as fields on a TE), but use your best judgment. Prefer flattening and using a new ID to not doing so, and just don't be stupid. Ask experienced modders on Discord or #minecraftforge if unsure.
* Also if you are unsure, check out the vanilla [wiki page](https://minecraft.gamepedia.com/1.13/Flattening) on the flattening to get a feel for what should be split out into a new ID and what should stay as blockstate properties.
  * No, before you ask, you still can't make a block with 500 blockstate properties each with a dozen values. You'd run out of memory because all possible `IBlockState` are generated at startup. If you need something that dynamic, use a TE. 
* Using more ID's does **not** mean you can't reuse code, you have always been able to instantiate a class multiple times and register it under different names.

### Dealing with Item metadata

* If dealing with tool damage, move it to an NBT tag inside the "tag" tag.
    * This can be done by just using the `getDamage` and `setDamage` ItemStack calls
* Otherwise, flatten the Item, moving each of its subvariants to a different ID.
    * Example: Instead of `botania:manaresource @ <arbitrary magic number>`, we now have `botania:mana_diamond`, `botania:mana_pearl`, etc.
    * If you didn't use a god item with a billion meta values for everything, good for you, you have almost nothing to do (e.g. Pam's Harvestcraft already uses a new item ID for everything instead of metas => literally zero work)
    * Split the field apart, comparing item meta values should just change to an == check on the flattened Item instances.
    * Old == checks on the unflattened item can either check tags (see below) or be an `instanceof`, if appropriate.
    * Update your recipes and loot tables
    * Update your lang files
* Please *don't* just do NBT hacks in your Items. Unless absolutely necessary (e.g. you have infinitely many variants, in which you would have been using NBT already).
* The cost of an extra ID is the Item instance (a handful of bytes allocated once), and the ID (a handful of bytes for the string). 
* The cost of NBT hacking is extra tags on EVERY stack (= extra storage space, extra processing overhead, extra network bandwidth consumed). Syncing a plain itemstack in our new flattened world is just sending two ints (int ID and stacksize). Adding an NBT tag significantly increases the overhead, and this overhead is paid on EVERY stack. It adds up fast.
* In addition, many vanilla and modded subsystems will not (and should not, IMO) support NBT, most notably being the Tag system. NBT should be the exception, not the norm.
* NO, JUST BECAUSE YOU ALREADY USED GET/SETITEMDAMAGE AND IT COMPILES DOES NOT MEAN YOU CAN LEAVE IT. FLATTEN YOUR ITEMS PROPERLY OR I WILL HUNT YOU DOWN.

### Dealing with Block metadata
Here's the extremely quick rundown. Assumes prior knowledge of blockstates:

* ALL vanilla `IBlockState`s are completely saved now (yes, this includes things that used to be under `getActualState` like fence connections, which were not saved before 1.13).
  * Things like fence connections are handled by `Block.func_196242_c` (updateNeighbors) and `Block.func_196271_a` (currently updatePostPlacement). See MCP comments for more information
  * For rendering-only properties that don't need to be set serverside, use extended states instead (an improvement to them is TBD and so this is not working just yet)
* Remove your getMetaFromState/getStateFromMeta methods, they are no longer needed. Hopefully you aren't heavily relying on them because you were going the easy way when porting to 1.8...
* Flatten all of your blocks, meaning that anything that might need a separate ItemBlock, needs to become a separate block itself. Inherent properties of the Block itself remain as blockstate properties.
    * Example: Vanilla logs are no longer crammed into `minecraft:log[axis, variant]`, but are `minecraft:oak_log[axis=x,y,z]`, `minecraft:oak_bark`, etc. Because a different axis log doesn't need another ItemBlock (since axis depends purely on placement position), but a different type of log needs another ItemBlock (since Items no longer have metas)
    * A simple heuristic that works in many (but not all) cases is to take any PropertyEnum VARIANT you might have had, and just pass that Enum instance into the Block's constructor and hold it as a field instead. Instead of checking the blockstate property, just check the field. See BlockShulkerBox to see how vanilla does this - it holds onto a final EnumDyeColor field for each block instance.
    * If otherwise unsure when to flatten or not, look at vanilla and emulate.
* Flatten all of your ItemBlocks accordingly. It should just be passing the newly flattened block instances into more instances of the same class.
    * Please don't do NBT hacks, for all the reasons described above.
* Accordingly, comparing variants should just change to == check on the flattened Block instances.
* If you were saving blockstates using registry name + meta, you should now use NBTUtil.read/writeBlockstate. See how Endermen save their carried block for an example.

## Tags (OreDictionary for ctrl+f)

* Read https://mcforge.readthedocs.io/en/1.13.x/utilities/tags/
* Remember: Block tags (`tags/blocks/...`)are only used for commands like /execute which query the blocks in world directly. Advancements and recipes both operate on *ItemBlock*s and use *item* tags (`tags/items/...`). Yes this will result in some duplication. If you're lazy, you can skip out on the block tags since the item tags are much more important, but doing them is still recommended for good compatibility.
      * For example, mods which have in-world recipes like Botania might want to use block tags. Prior to 1.13 such mods must slowly iterate the OreDictionary (which is built for items, not blocks) and/or perform caching logic. With block tags, it would just be a constant time set lookup.

## Recipes
* You need a json for every recipe now (*dramatic music plays*)
  * But not really: You just need a json to allow it to be disabled by datapacks (and syncing to client)
  * Basically there is a registry from type -> Json deserializer, so your recipe jsons can now have whatever custom JSON format you want, including all code (think Loot table LootFunctions and LootConditions). 
  * See vanilla mapcloning and mapextending.json and their in-code equivalents, the json simply specifies the type, and the entire recipe is implemented in code.
  * For ordinary recipes, you probably should convert them to jsons.
* TBD: Mojang seems to have some kind of auto recipe dumper at `net.minecraft.data.RecipeProvider` that generates the recipe json as well as an advancement to unlock the recipe. This tool appears to be runnable by running the minecraft jar with a different start class.
* Furnace recipes are now be specified using the normal recipe JSONs.

## Advanced Recipes
* You can now register custom deserializers for recipes, which are dispatched based on the `type` field in the recipe json.
* Your recipe type must be syncable over the network (since recipes are now sent to all clients on login and datapack reloads)
    * No, don't just send the json text over. These are sent on login, so you don't want the client to waste time reparsing stuff. Ideally, you want to send the absolute least amount of information you can to define the recipe.
* One-to-many and super dynamic recipes are still TBD, talk with forge devs or modders of interest (tterrag) if you want to discuss a solution for this

## Commands
* Commands have been overhauled, and now use a separate Mojang library called Brigadier (it's not obfuscated, so you can just look at the source)
* Command syntax is now much more declarative (instead of trying to manually parse things out of the command string each time)
* Command syntax now synced from server to client
* Tab completion only asks the server if the specific argument in the command says that it should
* Completion for things that have a static fixed set of options done clientside (e.g. /gamemode)
* Not much is known yet, see vanilla for examples

## Fluids and waterlogged blocks
* 1.13 brings a new "water in block" mechanic
* The interfaces are very general - World now has getFluidState and setFluidState methods, and fluids are elevated to almost block-level status (they can even have their own `IProperty`'s)
* However, the current fluid state implementations usually just defers to blockstate properties on the underlying block such as the `waterlogged` property.
  * The system is likely incomplete; expect this to evolve in future versions to be like Bedrock Ed (where water can reside in any block)
* more info on classes to implement, etc. when everything's deciphered
* Fluids are now a registry object in vanilla, and your == Blocks.WATER/LAVA checks probably need to be changed to tag checks (see `BlockCoral.canLive` for an example)

## World gen changes
* HUGE amount of changes, likely the largest subsystem change in this update
* Now threaded. I hope your worldgen functions are pure!
* TBD as things get named, but it's highly likely you will have to make significant changes.
* Barteks has a WIP subprimer here: https://gist.github.com/Barteks2x/41122efc766afdd47aeb457a3c19b275

## Data Fixers
* Got an extremely category-/type-theory heavy makeover due to the extreme changes to the world format
* No really, here's the paper: https://arxiv.org/ftp/arxiv/papers/1703/1703.10857.pdf
* TBD as it gets deciphered, but I imagine fixers will not be very useful for mods anymore.

## Containers
* https://github.com/MinecraftForge/MinecraftForge/blob/1.14.x/src/test/java/net/minecraftforge/debug/misc/ContainerTypeTest.java

## Particles
Particles are now handled in a much more complete and cohesive manner. There are two components to the particle system, the logical side (teaching the game about your particle, how to spawn it, etc.) and the rendering side (teaching the game how to render your particle).

### Logical Particles

#### IParticleType
`IParticleType` describes the general "type" of the particle (e.g. "redstone dust"). This class is similar to `Block` and `Item`, you subclass it and register it on both sides using the standard Forge registry events (`RegistryEvent.Register<IParticleType>`).

#### IParticleData
This class is to `IParticleType` as `ItemStack` is to `Item`. That is, `IParticleType` describes the general type of the particle, while `IParticleData` describes a specific instance of it (e.g. "redstone dust with mint green color"). It's the unit of handling for particles, and is what you pass to `world.addParticle` to actually spawn stuff.

Subclasses of this type carry additional data that describe a particle's appearance. For example, vanilla's `ItemParticleData`, used for showing the item pickup animation, holds the ItemStack being picked up, and `RedstoneParticleData`, as mentioned above, holds the color of the particle. If you're implementing your own data, you'll have to teach the game how to write/read it from the network `PacketBuffer`, as well as how to read it from the command line (to support the `/particle` command). See vanilla for examples.

If your particle needs no additional data, then just use `BasicParticleType`, a convenience class provided by vanilla that implements both `IParticleType` and `IParticleData` as a singleton that writes and reads nothing from the network. The vast majority of vanilla particles in `ParticleTypes` use this.

#### Particle
The `Particle` class is what actually represents a moving particle in game. It's client-side only and has a tick method for you to do custom movement, etc. 

#### Particle Factories
The above two classes are all that's needed serverside -- the server only needs to know the particle type, the additional data, and how 
to get it onto the network. On the client, we have to concern ourselves with how to get from the `IParticleData` instance to a subclass 
of `Particle`, the *actual* particle type. This is the job of `IParticleFactory`. During `ParticleFactoryRegisterEvent`, you must 
inform the game by calling `Minecraft.getInstance().particleManager.registerFactory(yourType, yourIParticleFactory)`. The interface 
only has one method, which receives the world, position, motion, and `IParticleData` (basically everything you passed to 
`world.addParticle`) and returns a `Particle`. The code here depends purely on your own needs, and can be as simple as just 
constructing your Particle subclass and passing the `IParticleData` to it. Again, see vanilla for examples.

#### Example: Botania
Botania wisps uses vanilla's systems for the above, though it renders the particles on its own (so the next section doesn't apply).
The particle type class is [here](https://github.com/Vazkii/Botania/blob/1.14/src/main/java/vazkii/botania/client/fx/WispParticleType.java),
and the data carried by each particle is [here](https://github.com/Vazkii/Botania/blob/1.14/src/main/java/vazkii/botania/client/fx/WispParticleData.java).
As you can see, the factory literally just pulls data out of the data class and passes it into the particle type `FXWisp`.
To spawn it, I call a `WispParticleData.wisp` helper, then pass it to `world.addParticle`. 
[Example](https://github.com/Vazkii/Botania/blob/b4e0a076e78ff6f551e4c1fc8949efb29230ed1a/src/main/java/vazkii/botania/common/item/ItemSextant.java#L82-L88).
This also demonstrates that your `IParticleData` should be immutable, and if so, you can reuse it as many times as you want.

### Rendering Particles

#### Particle JSON
That's most of the story done, but now we want to actually render the particle. First off, note that for any of your `IParticleType`, you need a particle json corresponding to it. For `botania:wisp`, there exists `assets/botania/particles/wisp.json` with a json object inside. This is required even if you aren't using vanilla's animated sprite system.

#### AnimatedSprite system
What's the animated sprite system? It's the vanilla way of having "default-looking" particles (billboarded flat textures, possibly 
switching between multiple textures during the particle's lifetime.
For an example, view the json and code for particle type `minecraft:poof` (`/assets/minecraft/particles/poof.json`).
You can see that it has a list of particle textures, which will be loaded and stitched by the game on startup.
Also observe that vanilla's `registerFactory` call for `POOF` is a weird overload that requires a function that takes an 
`IAnimatedSprite` and itself produces a factory.
That `IAnimatedSprite` is the code representation of the list of textures in the json. `IAnimatedSprite` has two methods, one to select a texture randomly, and one to select a texture based on how old the particle is.
This is how for example vanilla's dust particles fade away into fewer pixels, by switching textures based on particle age.
This all sounds super abstract but if you just follow what vanilla does for `POOF` from the factory all the way to the `PoofParticle` 
constructor and tick methods (especially what it does with the `IAnimatedSprite` every tick), it'll be pretty clear what's going on.
Subclass from `SpriteTexturedParticle`, create the json, and call registerFactory, and update the sprite every tick as necessary.
The actual rendering is pretty much completely handled by vanilla.
Note that you don't have to have multiple textures to use this system, all "default-looking" vanilla particles use this, even if they
have only one texture.

#### Doing your own rendering
Simply override `renderParticle` and feed your vertex data into the passed `BufferBuidler` (or do your own GL rendering for 
`IParticleRenderType.CUSTOM`).
Note that you'll have to pick a `IParticleRenderType` which determines the GL state under which your particle renders, or create your own implementation.

### Spawning Particles
A couple notes on spawning particles. The `World.addParticle` methods only work clientside, meaning that they do not communicate over 
the network and calling them on the logical server does nothing. To actually send a message, use the `ServerWorld.addParticle`
overloads, which actually build a packet with your serialized `IParticleData` and send it to the client, which will then deserialize it
, call the factory, and render it.

Of course, if you're spawning tens to hundreds of particles, it would be prudent to send a single custom packet to the client and have
the for loop 1 to 100 clientside, instead of looping server side and sending 100 particle packets, wasting bandwidth.

Additionally note that after all this, your particle should be completely supported by the `/particle` vanilla command, provided
you implemented the necessary method in your IParticleData Deserializer.

## GUI/Mouse/Keyboard/Input/Keybindings
* LWJGL3 now uses GLFW as the backing windowing system, so things have shifted accordingly.
* If you need something, look around to see if `MouseHelper` and `KeyboardListener` have what you need already (both accessible from `Minecraft`). Common things like the mouse position and button states etc are already here.
* Otherwise, you can use direct `GLFW` calls to get input state, see [glfw input guide](https://www.glfw.org/docs/latest/input_guide.html). The article is for C, but you can generally directly translate by adding the `GLFW` class in front of the function names and constants given. E.g. `glfwGetKey(window, GLFW_KEY_E) == GLFW_PRESS` => `int state = GLFW.glfwGetKey(mc.mainWindow.getHandle(), GLFW.GLFW_KEY_E) == GLFW.GLFW_PRESS`
* Notably, `Gui` method signatures have changed a bit. See `IGuiEventListener`
  * `mouseClicked` and `mouseReleased` receives the x, y, and modifiers
  * `mouseDragged` receives the same, plus the x and y drag amounts
  * `mouseScrolled` receives the amount scrolled
  * `keyPressed` and `keyReleased` receives the keyCode, scanCode, and modifiers. The scan code is likely not useful to you, check the keyCode with constants in GLFW instead, e.g. `keyCode == GLFW.GLFW_KEY_E`
  * `charTyped` receives a any textual input, as the form of unicode codepoint and the modifiers. Note that java `char` is only 16 bit so cannot encode all possible unicode points, but should be "enough" for most keyboard input uses. This call should be used for text input purposes, while `keyPressed` should be used for "key bind" purposes.
* To check whether a key binding is pressed, given the keyCode and scanCode, call `<KEYBINDING_OBEJCT>.isActiveAndMatches(InputMappings.getInputByCode(keyCode, scanCode))`
* Misc
  * mc.setIngameFocus -> mc.mainWindow.grabMouse/ungrabMouse

## Dimensions
* The class names are a bit screwy, so here's a brief rundown
* `DimensionType`: Vanilla class, unique handle to a world. That is, there is a 1:1 correspondence between `DimensionType` and `WorldServer` (so it's not really the dimension type, the name is a bad one). This has a string name, and replaces any ints you used in previous versions. AKA if you save stuff to disk, use the string name of this object.
* `ModDimension`: Forge class, the actual template/type for new dimensions. That is, you can have one `ModDimension` and produce multiple `DimensionType`s from it
* `Dimension`: Vanilla class, created by and tied to `DimensionType`. Not very useful on it's own. See vanilla subclasses for more info, you can probably reuse one of them directly.
* To register:
  * During `RegistryEvent.Register<ModDimension>`, register your `ModDimension`s like you would blocks and items
  * During `RegisterDimensionsEvent`, `DimensionType`s are registered. Some special handling is required. This event does *NOT* behave like other events, in that registrations are "sticky". Once a `DimensionType` has been registered once in a given save, it will show up again already without needing to be re-registered. Thus you should perform an existence check and only then register one or more `DimensionType` using your `ModDimension`. See [this](https://github.com/Tropicraft/Tropicraft/blob/1.14/src/main/java/net/tropicraft/core/common/dimension/TropicraftWorldUtils.java#L26-L58) for an example.
* `MinecraftServer.getWorld`
* `entity.dimension.getType()`

## Misc Things

* Vanilla now has a "different AABB depending on where you look" system (aka vanilla diet hoppers but even better). Play with the hopper in 1.13 to see what I mean. See hopper or end portal frame block classes for examples.
* Structure Blocks (and anything you made using them) need to be reloaded into a 1.12 world, the world run in 1.13, then resaved in 1.13. No idea why Mojang doesn't have an automated way to fix this, but it is what it is
* Block attributes like their material or hardness are now passed in a builder/POJO to the block constructor, and are final
  * Similar for Items
* Blocks no longer have a creativetab, that belongs on their ItemBlocks
* Biome ids are internally ints now instead of bytes, which raises the maximum number of allowed biomes from 255 to 2 billion.
* Enchantments are now stored by registry name instead of int id (about time...)
* A word about integer ID's
  * For Items and Blocks, you should have already switched  to fully using registry names for everything. The registry name to int ID maps still exist in vanilla for performance purposes, but they should *never* be used for anything other than network communication because they are NOT guaranteed to be the same after joining a different world. This should already be an established modding convention since 1.8, but people still break it, so I'll mention it again :P
* There's lots of superinterfaces on world now representing different things you can do (e.g. there's an interface for read-only access to the world). If possible, try using the least specific one.
* Most nameable things that still used strings now use text components, so translate all the things!
* There's multiple kinds of air block now (air, void_air, and cave_air). So if you're doing == Blocks.AIR checks still, change them to IBlockState.isAir checks
* More things are stored in registries now: Entities, BiomeProviders, ChunkGenerators, ParticleTypes, Stats, Paintings
* As always, check the vanilla 1.13.x patch notes for anything I missed that's gameplay-facing
* Your world renderers (RenderWorldLastEvent) look weird? give this a read: https://discordapp.com/channels/313125603924639766/454376090362970122/588257071812837376

## Nitty Gritty Random Things (Ctrl+F section)
* SideOnly(Side.CLIENT) => OnlyIn(Dist.CLIENT)
* SidedProxy => DistExecutor
  * e.g. `public static IProxy proxy = DistExecutor.runForDist(() -> ClientProxy::new, () -> ServerProxy::new)`;
  * Why the additional lambda at the front? It's to prevent the ClientProxy and ServerProxy classes from directly being referenced by the main mod class.
* mcmod.info moved to mods.toml, see MDK examplemod src/main/resources/META-INF/mods.toml
* EventBus is a separate library (fix imports). Can register handlers without annotations (so you can use private methods for example). See MDK ExampleMod
* Loader.isModLoaded => ModList.get().isLoaded()
* important renames
  * Entity.onUpdate => tick
  * Entity.onEntityUpdate => basetick
  * Entity.setDead => remove
  * World.setBlockToAir => removeBlock
  * World.scheduleUpdate => world.getPendingBlockTicks().scheduleTick (blocks and fluids have separate scheduled ticks now)
  * Block methods: IBlockState moved to front of all forge methods
* checking physical side => FMLEnvironment.dist
* checking logical sides: 
  * getEffectiveSide => EffectiveSide.get (HIGHLY encouraged to move to world.isRemote checks)
  * getting logical server: ServerLifecycleHooks.getCurrentServer()
* network: SimpleImpl -> SimpleChannel
  - enqueuework instead of addScheduledTask
  - register free funcs
  - See https://github.com/sinkillerj/ProjectE/blob/c17ff6e1b7151b9ef12396af47a937bb599bf7bf/src/main/java/moze_intel/projecte/network/PacketHandler.java#L23-L52
* `ScaledResolution.<x>` => `Minecraft.getInstance().mainWindow.<x>`
* isOpaqueCube => remove, mc now checks this with the block's voxelshape
* ArmorMaterial => implement `IArmorMaterial`, no more enum hackery needed
* ToolMaterial => implement `IItemTier`, no more enum hackery needed
* EnumHelper => gone. some enums were abstracted to interfaces by vanilla for us (like the above), and the rest have moved to dedicated `create` methods on each individual enum. For example, `BannerPattern.create(...)`
* TileEntity.shouldRefresh => logic moved to Block.onReplaced. see BlockFurnace.onReplaced for an example.
* getBlockFaceShape => gone, the game is smart enough to check your VoxelShapes from `getShape` now

## Cross-referencing, a quick how-to
* Oh no! Method FooClass.BarMethod disappeared in 1.13/1.14! Where did it go? Follow these easy steps for a guaranteed 80% success rate!

1. Open a 1.12 workspace (this is why you use a separate workspace to update, by the way)
2. Browse to FooClass.BarMethod
3. Use your IDE's find usages tool to see where it was called from in vanilla
4. Pick a call site
5. Go to that same call site in 1.13/1.14
6. What does it call instead?
7. ???
8. Profit

(if these steps don't apply, *then* you're allowed to ask)

## Some parting words
* Ask experienced modders for help on Discord/IRC or look at their own ports
* Help others needing help, but only if you're 100% certain what you're talking about. As always, vanilla should be a first reference regarding mechanisms and conventions

## Reporting Errata
* If you notice something incorrect in this primer, please let me (williewillus) know as soon as possible on the Modded Minecraft discord, twitter, or #minecraftforge IRC if I'm there. I don't want to spread misinformation, so I appreciate the speedy notification.

*Sourced from https://gist.github.com/williewillus/353c872bcf1a6ace9921189f6100d09a*
