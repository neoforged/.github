# Minecraft 1.21.8 -> 1.21.9 Mod Migration Primer

This is a high level, non-exhaustive overview on how to migrate your mod from 1.21.8 to 1.21.9. This does not look at any specific mod loader, just the changes to the vanilla classes. All provided names use the official mojang mappings.

This primer is licensed under the [Creative Commons Attribution 4.0 International](http://creativecommons.org/licenses/by/4.0/), so feel free to use it as a reference and leave a link so that other readers can consume the primer.

If there's any incorrect or missing information, please file an issue on this repository or ping @ChampionAsh5357 in the Neoforged Discord server.

## Pack Changes

There are a number of user-facing changes that are part of vanilla which are not discussed below that may be relevant to modders. You can find a list of them on [Misode's version changelog](https://misode.github.io/versions/?id=1.21.9&tab=changelog).

## Debug Screen Rework

The debug screen has been completely overhauled, allowing users to enable, disable, or only show in F3 specific components of the screen. This modular system allows for modders to add their own debug entries to the screen. Not all parts of this explanation is accessible without a bit more modding work, so those areas will be specifically pointed out.

### `DebugScreenEntry`

Every debug option has its own entry that either defines what is being displayed (e.g., fps, memory), or no-ops to be handled by a separate implementation (e.g., entity hitboxes, chunk borders). This is known as a `DebugScreenEntry`, which defines three methods.

First is the `category`. In vanilla, this is almost always `DebugEntryCategory#SCREEN_TEXT`, as all a debug entry does is draw text to the screen. The other available option `RENDERER` is only for no-op as the rendering options always rendered independent from the debug screen. `DebugEntryCategory` is simply a record with a label and some sort of sort key value, so more can be added by calling the constructor. All the category is used for is searching in the debug options screen.

Next is `isAllowed`. This method determines whether the debug option should render on the screen independent of the entry status. By default, this is true only when the `Minecraft#showOnlyReducedInfo` accessibility option is false. Some debug entries override this method to always return true, or if some other check passes.

Finally, there is the `display` method. This is responsible for drawing the text to the screen using the `DebugScreenDisplayer`. It also takes in the current `Level`, client chunk, and server chunk. The `DebugScreenDisplayer` has four methods that each draw text to the screen. First, there is the standard `addLine` class, which just adds the string to either the left or right side depending on what element its rendered as. These will always come one right after the other. Then, there is `addPriorityLine`, which will always be added to the top of either the left or the right side. Finally, there is `addToGroup`, which takes an additional key to render the lines as one separate group with an extra new line added at the end.

```java
public class ExampleDebugEntry implements DebugScreenEntry {

    public static final ResourceLocation GROUP_ONE = ResourceLocation.fromNamespaceAndPath("examplemod", "group_one");
    public static final ResourceLocation GROUP_TWO = ResourceLocation.fromNamespaceAndPath("examplemod", "group_two");


    @Override
    public void display(DebugScreenDisplayer displayer, @Nullable Level level, @Nullable LevelChunk clientChunk, @Nullable LevelChunk serverChunk) {
        // The following will display like so if it is the only entry on the screen:
        // First left!                                                First Right!
        // 
        // Hello world!                                               Random text!
        // Lorem ipsum.
        //                                                            I am another group!
        // I am one group                                             This will appear after with no line breaks!
        // All in a row
        // Provided in a list.
        //

        displayer.addLine("Hello world!");
        displayer.addLine("Lorem ipsum.");
        displayer.addLine("Random text!");

        // These will be displayed first
        displayer.addPriorityLine("First left!");
        displayer.addPriorityLine("First right!");

        // These will be grouped separately based on the key
        displayer.addToGroup(GROUP_ONE, List.of(
            "I am one group",
            "All in a row",
            "Provided in a list."
        ));

        displayer.addToGroup(GROUP_TWO, "I am another group!");
        displayer.addToGroup(GROUP_TWO, "This will appear after with no line breaks!");
    }

    @Override
    public boolean isAllowed(boolean reducedDebugInfo) {
        // Always show regardless of accessibility option
        return true;
    }
}
```

Then, simply register your entry to `DebugScreenEntries` to have it display. It can be toggled on through the debug menu using the provided key

```java
// This method is private, so its access will need to be widened
DebugScreenEntries.register(
    // The id, this will be displayed on the options screen
    ResourceLocation.fromNamespaceAndPath("examplemod", "example_entry"),
    // The screen entry
    new ExampleScreenEntry();
);
```

### External Toggles and Checks

What if you want to toggle the active status separately from the options menu? What if you want to check is an entry is enabled to display debug data in-game? This can be done by accessing the `DebugScreenEntryList` through the `Minecraft` instance.

Toggling the current status can be done via `DebugScreenEntryList#toggleStatus`. How this behaves changes depending on what screen is active. Basically, calling toggle will always flip the debug entry on or off: if it's not currently on screen, it will render on screen and vice versa. If in F3, then toggling on will only render that debug entry when F3 is on.

The status of the entry can then be checked using `DebugScreenEntryList#isCurrentlyEnabled`. This will only check if the debug screen is in the currently on list and not check `DebugScreenEntry#isAllowed`.

```java
// Lets create another entry
public static final ResourceLocation EXAMPLE_TOGGLE = DebugScreenEntries.register(
    ResourceLocation.fromNamespaceAndPath("examplemod", "example_toggle"),
    // We're using noop as nothing is being displayed as text
    new DebugEntryNoop();
);

// To toggle:
Minecraft.getInstance().debugEntries.toggleStatus(EXAMPLE_TOGGLE);

// To check if enabled:
if (Minecraft.getInstance().debugEntries.isCurrentlyEnabled(EXAMPLE_TOGGLE)) {
    // ...
}
```

### Profiles

Profiles are defined presets that can be configured to the user's desire. Currently, profiles are hardcoded to either default or performance. To extend the system, you need to be able to dynamically add an entry to the `DebugScreenProfile` enum, make the `DebugScreenEntries#PROFILES` map mutable to add your profile and preset, and modify the debug option screen with your profile button.

- `net.minecraft.client.Minecraft`
    - `debugEntries` - Returns a list of debug features and what should be shown on screen.
    - `fpsString`, `sectionPath`, `sectionVisibility` are removed
- `net.minecraft.client.gui.Gui`
    - `renderDebugOverlay` is now public
    - `shouldRenderDebugCrosshair` is removed
- `net.minecraft.client.gui.components.DebugScreenOverlay`
    - `drawGameInformation`, `drawSystemInformation` are removed
    - `getGameInformation`, `getSystemInformation` are removed
    - `toggleOverlay` is removed
- `net.minecraft.client.gui.components.debug`
    - `DebugEntryBiome` - A debug entry displaying the biome the camera entity is within.
    - `DebugEntryCategory` - A category that describes how a debug entry is displayed.
    - `DebugEntryChunkGeneration` - A debug entry displaying the current chunk generation info.
    - `DebugEntryChunkRenderStats` - A debug entry displaying the current section statistics.
    - `DebugEntryChunkSourceStats` - A debug entry displaying the general metadata of the chunk sources.
    - `DebugEntryEntityRenderStats` - A debug entry displaying the general metadata of the entity storage.
    - `DebugEntryFps` - A debug entry displaying the frames per second and vsync info.
    - `DebugEntryGpuUtilization` - A debug entry displaying the GPU utilization.
    - `DebugEntryHeightmap` - A debug entry displaying the height maps at the current position.
    - `DebugEntryLight` - A debug entry displaying the client light information.
    - `DebugEntryLocalDifficulty` - A debug entry displaying the current world difficulty and time.
    - `DebugEntryLookingAtBlock` - A debug entry displaying the block the camera is currently looking at.
    - `DebugEntryLookingAtEntity` - A debug entry displaying the entity the camera is currently looking at.
    - `DebugEntryLookingAtFluid` - A debug entry displaying the fluid the camera is currently looking at.
    - `DebugEntryMemory` - A debug entry displaying the memory allocated and used by the game.
    - `DebugEntryNoop` - A debug entry displaying nothing.
    - `DebugEntryParticleRenderState` - A debug entry displaying the number of particles being rendered.
    - `DebugEntryPosition` - A debug entry displaying the current position and rotation of the camera entity.
    - `DebugEntryPostEffect` - A debug entry displaying the currently applied post effect.
    - `DebugEntrySectionPosition` - A debug entry displaying the current section position.
    - `DebugEntrySimplePerformanceImpactors` - A debug entry displaying the graphics mod, cloud status, and biome blend radius.
    - `DebugEntrySoundMood` - A debug entry displaying the current sound played and player mood.
    - `DebugEntrySpawnCounts` - A debug entry displaying the entity spawn counts per mob category.
    - `DebugEntrySystemSpecs` - A debug entry displaying the specs of the running machine.
    - `DebugEntryTps` - A debug entry displaying the general ticks per second.
    - `DebugEntryVersion` - A debug entry displaying the current Minecraft version.
    - `DebugScreenDisplayer` - An interface that the debug entries use to display elements to the screen.
    - `DebugScreenEntries` - The debug entries registered by Minecraft.
    - `DebugScreenEntry` - An element that shows up, if enabled, when the debug overlay is enabled.
    - `DebugScreenEntryList` - The options information for what debug elements are displayed on screen.
    - `DebugScreenEntryStatus` - The status of when a debug entry should be displayed.
    - `DebugScreenProfile` - An enum denoting the profiles that the debug screen can set when deciding what entries to show.
- `net.minecraft.client.gui.screen.debug.DebugOptionsScreen` - A screen that allows the user to change the displayed debug entries for a profile.
- `net.minecraft.client.renderer.LevelRenderer`
    - `getSectionStatistics` is now nullable
    - `getEntityStatistics` is now nullable
- `net.minecraft.client.renderer.debug.DebugRenderer`
    - `switchRenderChunkborder` -> `DebugScreenEntries#CHUNK_BORDERS`, not one-to-one
    - `toggleRenderOctree` -> `DebugScreenEntries#CHUNK_SECTION_OCTREE`, not one-to-one

## Feature Submissions: The Movie

The entirety of the entity rendering pipeline, from the models to the text floating above their heads, has been reworked into a submission / render phase system known as features. It is likely that the feature system will take over block entities and particles as well given that the render phase calls has already been added, although they currently do nothing. This guide will go over the basics of the feature system itself followed by how entities are implemented using it.

### Submission and Rendering

The feature system, like GUIs, is broken into two phases: submission and rendering. The submission phase is handled by the `SubmitNodeCollector`, which as the name implies, collects the data required to abstractly render some data to the screen. This is all done through the `submit*` methods, which generally take a `PoseStack` to place the object in 3D space, and any other required data such as the model, state, render type, etc.

Here is a quick overview of what each method takes in:

Method                 | Parameters
:---------------------:|:----------
`submitHitbox`         | A pose stack, render state of the entity, and the hitboxes render state
`submitShadow`         | A pose stack, the shadow radius, and the shadow pieces
`submitNameTag`        | A pose stack, the position offset, the text component, if the text should be seethrough (like when sneaking), light coordinates, and square distance to the camera
`submitText`           | A pose stack, the XY offset, the text sequence, whether to add a drop shadow, the font display mode, light coordinates, color, and background color
`submitFlame`          | A pose stack, render state of the entity, and a rotation quaternion
`submitLeash`          | A pose stack and the leash state
`submitModel`          | A pose stack, entity model, render state, render type, light coordinates, overlay coordinates, tint color, texture, outline color, and a priority (e.g., 0 for base model, 1 for armor model)
`submitBlock`          | A pose stack, block state, light coordinates, and overlay coordinates
`submitFallingBlock`   | A pose stack and the falling render state
`submitBlockModel`     | A pose stack, the render type, block state model, RGB floats, light coordinates, and overlay coordinates
`submitItem`           | A pose stack, stack render state, light coordinates, and overlay coordinates
`submitCustomGeometry` | A pose stack, render type, and a function that takes in the current pose and `VertexConsumer` to create the mesh

The render phase is handled through the `FeatureRenderDispatcher`, which renders the elements using feature renderers. What are feature renderers? Quite literally an arbitrary method that loops through the node contents its going to push to the buffer. Currently, the features push their vertices in the following order: entity shadows, entity models, entity on fire animations, entity name tags, arbitrary floating text, hitboxes, leashes, items, blocks, and finally custom render pipelines. All submissions are then cleared for next use.

Most of the feature dispatchers are simply run a loop except for two: `EntityModelFeatureRenderer` for entity models, and `CustomFeatureRenderer` for custom geometry. First, both entity models and custom geometry group elements by their `RenderType` to upload all the vertex data for the feature objects in one pass per `RenderType`. Additionally, entity models has a priority integer that is used to have certain models render on top of another rather than in the same pass. The most common use case is when the base model is rendered on `0` while armor is rendered on layer `1` and on depending on trim and color layers. When to use priority depends on whether you believe the element is rendered on top of or is part of another model (e.g., sheep and its wool are both on `0` while the enderman eyes are on `1`).

### Entity Models

So, how does this affect entities? Let's start with the root `Model` that makes up all entity models. `Model` now has a generic which is used to pass in the state of the backing object to `setupAnim`, which has also been moved to `Model`. This means that the base model class is rarely passed around, instead opting for some subtype, like `Model$Simple` for signs. Given that most `EntityModel`s already require a generic of `EntityRenderState`, this does not affect anything.

The main change comes from how model part visibility work, such as armors and capes. Every single individual part (e.g. helmet, chestplate) now has their own separate model, meaning that the general part visibility system has been completely removed. You can still provide visibility through the mutable model part in `setupAnim`, but the general movement is to simply have parts that should be separate models as separate models.

To facilitate this, `PartDefinition`s now has methods to selectively keep certain parts of models and remove the others. This is done through the `clear*` and `retain*` methods. Basically, all of these methods are doing are keeping the part pose while removing any cubes associated with the children queries. `retain*` allows for certain parts and potentially subparts to keep their cubes. This addition provides a twofold benefit: model layers can deterministically use `setupAnim` to setup the parts similarly to the base model, and the model texture will only need to contain the retained elements.

Here is an example for creating the armor models for a creeper:

```java
// Deformations for humanoid armor
private static final CubeDeformation OUTER_ARMOR_DEFORMATION = new CubeDeformation(1.0F);
private static final CubeDeformation INNER_ARMOR_DEFORMATION = new CubeDeformation(0.5F);

// Creeper has the following parts:
// - head
// - body
// - right_hind_leg, left_hind_leg
// - right_front_leg, left_front_leg

// We use separate meshes as we are basically isolating the parts we want to keep in each

public static ArmorModelSet<MeshDefinition> createCreeperArmor() {
    // Helmet

    // Create mesh
    MeshDefinition helmetMesh = CreeperModel.createBodyLayer(OUTER_ARMOR_DEFORMATION);
    // Only retain the desired parts
    // Note that body and the legs still exist, they simply have no cubes
    helmetMesh.getRoot().retainExactParts(Set.of("head"));

    // Chestplate

    // Create mesh
    var chestplateMesh = CreeperModel.createBodyLayer(OUTER_ARMOR_DEFORMATION);
    // Only retain the desired parts
    chestplateMesh.getRoot().retainExactParts(Set.of("body"));

    // Leggings

    // Create mesh
    var leggingsMesh = CreeperModel.createBodyLayer(INNER_ARMOR_DEFORMATION);
    // Only retain the desired parts
    leggingsMesh.getRoot().retainExactParts(Set.of("right_hind_leg", "left_hind_leg", "right_front_leg", "left_front_leg"));

    // Boots

    // Create mesh
    var bootsMesh = CreeperModel.createBodyLayer(OUTER_ARMOR_DEFORMATION);
    // Only retain the desired parts
    bootsMesh.getRoot().retainExactParts(Set.of("right_hind_leg", "left_hind_leg", "right_front_leg", "left_front_leg"));

    // Store this all in an ArmorModelSet, which is basically just object holder and mapper
    return new ArmorModelSet<>(
        helmetMesh,
        chestplateMesh,
        leggingsMesh,
        bootsMesh
    );
}

// To register the layer definitions, basically the same process of using the ArmorModelSet
public static final ArmorModelSet<ModelLayerLocation> CREEPER_ARMOR = new ArmorModelSet<>(
    new ModelLayerLocation(ResourceLocation.fromNamespaceAndPath("examplemod", "creeper"), "helmet"),
    new ModelLayerLocation(ResourceLocation.fromNamespaceAndPath("examplemod", "creeper"), "chestplate"),
    new ModelLayerLocation(ResourceLocation.fromNamespaceAndPath("examplemod", "creeper"), "leggings"),
    new ModelLayerLocation(ResourceLocation.fromNamespaceAndPath("examplemod", "creeper"), "boots")
);

// In some method where you have access to the Map<ModelLayerLocation, LayerDefinition> builder
ArmorModelSet<LayerDefinition> creeperArmorLayers = createCreeperArmor().map(mesh -> LayerDefinition.create(mesh, 64, 32));
CREEPER_ARMOR.putFrom(creeperArmorLayers, builder);
```

### Entity Renderer

With the change to submission, the `EntityRenderer` and its associated `RenderLayer`s have also changed. Basically, you can assume almost every method that has the word `render` has been changed to `submit`, and `MultiBufferSource` and light coordinates integer have generally been replaced by `SubmitNodeCollector` and the associated entity render state.

The new `submit` method that replaces `render` in `EntityRenderer` now takes in the render state of the entity, the `PoseStack`, and the `SubmitNodeCollector`. When submitting any element, the location in 3D space is taken by getting the last pose on the `PoseStack` and storing that for future use.

```java
// A basic entity renderer

// We will assume all the classes listed exist
public class ExampleEntityRenderer extends MobRenderer<ExampleEntity, ExampleEntityRenderState, ExampleModel> {

    public ExampleEntityRenderer(EntityRendererProvider.Context ctx) {
        super(ctx, ctx.bakeLayer(EXAMPLE_MODEL_LAYER), 0.5f);
    }

    @Override
    public void submit(ExampleEntityRenderState renderState, PoseStack poseStack, SubmitNodeCollector collector) {
        super.submit(renderState, poseStack, collector);

        // An example of submitting something
        collector.submitCustomGeometry(
            poseStack, // The current pose
            RenderType.solid(), // The render type to use
            ExampleEntityRenderer::addVertices // The method to write the geometry data
        );
    }

    private static void addVertices(PoseStack.Pose pose, VertexConsumer consumer) {
        // Add custom geometry
    }
}

// A render layer

public class CreeperArmorLayer extends RenderLayer<CreeperRenderState, CreeperModel> {

    private final ArmorModelSet<CreeperModel> modelSet;
    private final EquipmentLayerRenderer equipment;

    public CreeperArmorLayer(RenderLayerParent<CreeperRenderState, CreeperModel> parent, ArmorModelSet<CreeperModel> modelSet, EquipmentLayerRenderer equipment) {
        super(parent);
        this.modelSet = modelSet;
        this.equipment = equipment;
    }

    // We will assume we added headEquipment, chestEquipment, legsEquipment, feetEquipment to the CreeperRenderState somehow
    @Override
    public void submit(PoseStack poseStack, SubmitNodeCollector collector, int lightCoords, CreeperRenderState renderState, float yRot, float xRot) {
        this.renderArmorPiece(poseStack, collector, renderState.chestEquipment, EquipmentSlot.CHEST, lightCoords, renderState);
        this.renderArmorPiece(poseStack, collector, renderState.legsEquipment,  EquipmentSlot.LEGS,  lightCoords, renderState);
        this.renderArmorPiece(poseStack, collector, renderState.feetEquipment,  EquipmentSlot.FEET,  lightCoords, renderState);
        this.renderArmorPiece(poseStack, collector, renderState.headEquipment,  EquipmentSlot.HEAD,  lightCoords, renderState);
    }

    // Taken from humanoid armor layer
    private void renderArmorPiece(PoseStack poseStack, SubmitNodeCollector collector, ItemStack stack, EquipmentSlot slot, int lightCoords, CreeperRenderState renderState) {
        Equippable equippable = stack.get(DataComponents.EQUIPPABLE);
        if (equippable != null && equippable.assetId().isPresent() && equippable.slot() == slot) {
            CreeperModel model = this.modelSet.get(slot);
            EquipmentClientInfo.LayerType layer = slot == EquipmentSlot.LEGS
                ? EquipmentClientInfo.LayerType.HUMANOID_LEGGINGS
                : EquipmentClientInfo.LayerType.HUMANOID;
            this.equipmentRenderer.renderLayers(
                layer, // The equipment layer to use
                equippable.assetId().orElseThrow(), // The equipment asset to pull
                model, // The armor model
                renderState, // The entity render state
                stack, // The armor stack
                poseStack, // The pose stack
                collector, // The collector to add the model data to
                lightCoords, // The light coordinates
                renderState.outlineColor // The outline color of the entity
            );
        }
    }
}

// Then, to add it to the creeper renderer constructor
public CreeperRenderer(EntityRendererProvider.Context ctx) {
    // ...
    this.addLayer(new CreeperArmorLayer(
        this, // The parent is the renderer itself
        ArmorModelSet.bake( // Baking the model set
            CREEPER_ARMOR, // The model layer locations
            ctx.getModelSet(), // The model set to map the models from the layer locations
            CreeperModel::new // The mapper for the root part to the model
        ),
        ctx.getEquipmentRenderer() // The renderer for the equipment
    ));
}
```

- `assets/minecraft/shaders/core/blit_screen.json` -> `screenquad.json`, using no-format triangles instead of positioned quads
- `net.minecraft.client.gui.render.pip`
    - `GuiBannerResultRenderer` now takes in a `MaterialSet`
    - `GuiEntityRenderer` now takes in a `FeatureRenderDispatcher`
    - `GuiSignRenderer` now takes in a `MaterialSet`
- `net.minecraft.client.model`
    - `AbstractPiglinModel#createArmorMeshSet` - Creates the model meshes for each of the humanoid armor slots.
    - `ArmedModel` now has a generic of the `EntityRenderState`
        - `translateToHand` now takes in the entity render state
    - `ArmorStandArmorModel#createBodyLayer` -> `createArmorLayerSet`, not one-to-one
    - `BellModel$State` - Represents the state of the backing object.
    - `BookModel$State` - Represents the state of the backing object.
    - `BreezeModel`
        - `createBodyLayer` -> `createBaseMesh`, now private
            - Replaced by `createBodyLayer`, `createWindLayer`, `createEyesLayer`
    - `CopperGolemModel` - A model for the copper golem entity.
    - `CreakingModel`
        - `NO_PARTS`, `getHeadModelParts` are removed
        - `createEyesLayer` - Creates the eyes of the model.
    - `EntityModel#setupAnim` -> `Model#setupAnim`
    - `HumanoidArmorModel` -> `HumanoidModel#createArmorMeshSet`, not one-to-one
    - `HumanoidModel#copyPropertiesTo` is removed
    - `Model` now takes in a generic representing the render state
    - `PlayerCapeModel` now extends `PlayerModel`
    - `PlayerEarsModel` now extends `PlayerModel`
    - `PlayerModel` static fields are now `protected`
        - `createArmorMeshSet` - Creates the model meshes for each of the humanoid armor slots.
    - `SkullModelBase$State` - Represents the state of the backing object.
    - `VillagerLikeModel` now takes in a generic for the render state
        - `hatVisible` is removed
            - Replaced by `VillagerModel#createNoHatModel`
        - `translateToArms` now takes in the render state
    - `WardenModel`
        - `createTendrilsLayer`, `createHeartLayer`, `createBioluminescentLayer`, `createPulsatingSpotsLayer` - Creates the layers used by the warden's `RenderLayer`s.
        - `getTendrilsLayerModelParts`, `getHeartLayerModelParts`, `getBioluminescentLayerModelParts`, `getPulsatingSpotsLayerModelParts` are removed
    - `ZombieVillagerModel`
        - `createArmorLayer` -> `createArmorLayerSet`, not one-to-one
        - `createNoHatLayer` - Creates the model without the hat layer.
- `net.minecraft.client.model.geom.ModelPart#copyFrom` is removed
- `net.minecraft.client.model.geom.builders.PartDefinition`
    - `clearRecursively` - Clears all children parts and its sub-children.
    - `retainPartsAndChildren` - Retains the specified parts from its root and any sub-children.
    - `retainExactParts` - Retains only the top level part, clearing out all others and sub-children.
- `net.minecraft.client.renderer`
    - `GameRenderer` now takes in the `BlockRenderDispatcher`
        - `getSubmitNodeStorage` - Gets the node submission for feature-like objects.
        - `getFeatureRenderDispatcher` - Gets the dispatcher for rendering feature-like objects.
        - `getLevelRenderState` - Gets the render state of dynamic features in a level.
    - `LevelRenderer` now takes in the `LevelRenderState` and `FeatureRenderDispatcher`
        - `getSectionRenderDispatcher` is now nullable
    - `LevelRenderState` - The render state of the dynamic features in a level.
    - `MapRenderer#render` now takes in a `SubmitNodeCollector` instead of a `MultiBufferSource`
    - `OutlineBufferSource` no longer takes in any parameters
        - `setColor` now takes in a single integer
    - `RenderPipelines`
        - `GUI_TEXT` - The pipeline for text in a gui.
        - `GUI_TEXT_INTENSITY` - The pipeline for text intensity when not colored in a gui.
    - `ScreenEffectRenderer` now takes in a `MaterialSet`
    - `Sheets`
        - `BLOCK_ENTITIES_MAPPER` - A mapper for block textures onto block entities.
        - `*COPPER*` - Materials for copper chests.
    - `SpecialBlockModelRenderer#vanilla` now takes in a `SpecialModelRenderer$BakingContext` instead of an `EntityModelSet`
    - `SubmitNodeCollector` - A submission handler for holding elements to be drawn to the screen at a given time.
    - `SubmitNodeStorage` - A node collector that hold the submitted features in the provided pose.
- `net.minecraft.client.renderer.block`
    - `BlockRenderDispatcher` now takes in a `MaterialSet`
    - `LiquidBlockRenderer#setupSprites` now take in the `BlockModelShaper` and `MaterialSet`
- `net.minecraft.client.renderer.blockentity`
    - `AbstractSignRenderer`
        - `getSignModel` now returns a `Model$Simple`
        - `renderSign` now takes in a `Model$Simple`
    - `BannerRenderer` has an overload that takes in a `SpecialModelRenderer$BakingContext`
        - The `EntityModelSet` constructor now takes in the `MaterialSet`
        - `renderPatterns` now takes in the `MaterialSet`
    - `BedRenderer` has an overload that takes in a `SpecialModelRenderer$BakingContext`
        - The `EntityModelSet` constructor now takes in the `MaterialSet`
    - `BlockEntityRenderDispatcher` now takes in the `MaterialSet`
    - `BlockEntityRendererProvider$Context` is now a record, taking in a `MaterialSet`
    - `CopperGolemStatueBlockRenderer` - A block entity renderer for the copper golem statue.
    - `DecoratedPotRenderer` has an overload that takes in a `SpecialModelRenderer$BakingContext`
        - The `EntityModelSet` constructor now takes in the `MaterialSet`
    - `HangingSignRenderer`
        - `createSignModel` now returns a `Model$Simple`
        - `renderInHand` now takes in a `MaterialSet`
    - `ShelfRenderer` - A block entity renderer for a shelf.
    - `ShulkerBoxRenderer` has an overload that takes in a `SpecialModelRenderer$BakingContext`
        - The `EntityModelSet` constructor now takes in the `MaterialSet`
    - `SignRenderer`
        - `createSignModel` now returns a `Model$Simple`
        - `renderInHand` now takes in a `MaterialSet`
    - `SkullBlockRenderer#submitSkull` - Submits the skull model to the collector.
    - `SpawnerRenderer#renderEntityInSpawner` no longer takes in the light coordinates integer
- `net.minecraft.client.renderer.entity`
    - Most methods here that take in the `MultiBufferSource` and light coordinates integer have been replaced by `SubmitNodeCollector` and a render state param
    - Most methods change their name from `render*` to `submit*`
    - `AbstractBoatRenderer#renderTypeAdditions` -> `submitTypeAdditions`
    - `AbstractMinecartRenderer#renderMinecartContents` -> `submitMinecartContents`
    - `AbstractSkeletonRenderer` takes in a `ArmorModelSet` instead of a `ModelLayerLocation`
    - `AbstractZombieRenderer` takes in a `ArmorModelSet` instead of a model
    - `ArmorModelSet` - A holder that maps some object to each humanoid armor slot. Typically holds the layer definitions, which are then baked into their associated models.
    - `BreezeRenderer#enable` is removed
    - `CopperGolemRenderer` - The renderer for the copper golem entity.
    - `DisplayRenderer#renderInner` -> `submitInner`
    - `EnderDragonRenderer#renderCrystalBeams` -> `submitCrystalBeams`
    - `EntityRenderDispatcher`
        - `prepare` no longer takes in the `Level`
        - `setRenderShadow`, `setRenderHitBoxes`, `shouldRenderHitBoxes` are removed
        - `extractEntity` - Creates the render state from the entity and the partial tick.
        - `render` -> `submit`
        - `setLevel` -> `resetCamera`, not one-to-one
    - `EntityRenderer`
        - `NAMETAG_SCALE` is now public
        - `render(S, PoseStack, MultiBufferSource, int)` -> `submit(S, PoseStack, SubmitNodeCollector)`
        - `renderNameTag` ->` submitNameTag`
    - `EntityRendererProvider$Context`
        - `getModelManager` is removed
        - `getMaterials` - Returns a mapper of material to atlas sprite.
    - `ItemEntityRenderer`
        - `renderMultipleFromCount` -> `submitMultipleFromCount`
    - `ItemRenderer#getArmorFoilBuffer` -> `getFoilRenderTypes`, not one-to-one
    - `PiglinRenderer` takes in a `ArmorModelSet` instead of a `ModelLayerLocation`
    - `TntMinecartRenderer#renderWhiteSolidBlock` -> `submitWhiteSolidBlock`
    - `ZombieRenderer` takes in a `ArmorModelSet` instead of a `ModelLayerLocation`
    - `ZombifiedPiglinPiglinRenderer` takes in a `ArmorModelSet` instead of a `ModelLayerLocation`
- `net.minecraft.client.renderer.entity.layers`
    - `BreezeWindLayer` now takes in the `EntityModelSet` instead of the `EntityRendererProvider$Context`
    - `EquipmentLayerRenderer#renderLayers` now takes in the render state, `SubmitNodeCollector`, and outline color instead of a `MultiBufferSource`
    - `HumanoidArmorLayer` now takes in `ArmorModelSet`s instead of models
        - `setPartVisibility` is removed
    - `ItemInHandLayer#renderArmWithItem` -> `submitArmWithItem`
    - `LivingEntityEmissiveLayer` now takes in a function for the texture instead of a `ResourceLocation` and a model instead of the `$DrawSelector`
        - `$DrawSelector` is removed
    - `RenderLayer`
        - `renderColoredCutoutModel`, `coloredCutoutModelCopyLayerRender` now takes in a `Model` instead of an `EntityModel`, a `SubmitNodeCollector` instead of a `MultiBufferSource`, and an integer representing the priority layer for rendering
        - `render` -> `submit`, taking in a `SubmitNodeCollector` instead of a `MultiBufferSource`
    - `StuckInBodyLayer` now has an additional generic for the render state, also taking in the render state in the constructor
    - `VillagerProfessionLayer` now takes in two models
- `net.minecraft.client.renderer.entity.state`
    - `CopperGolemRenderState` - The render state for the copper golem entity.
    - `EntityRenderState`
        - `NO_OUTLINE` - A constant that represents the color for no outline.
        - `outlineColor` - The outline color of the entity.
        - `lightCoords` - The packed light coordinates used to light the entity.
        - `shadowPieces`, `$ShadowPiece` - Represents the relative coordinates of the shadow(s) the entity is casting.
    - `LivingEntityRenderState#appearsGlowing` -> `EntityRenderState#appearsGlowing`, now a method
    - `PaintingRenderState#lightCoords` -> `lightCoordsPerBlock`
    - `WitherSkullRenderState#xRot`, `yRot` -> `modeState`, not one-to-one
- `net.minecraft.client.renderer.feature`
    - `BlockFeatureRenderer` - Renders the submitted blocks, block models, or falling blocks.
    - `CustomFeatureRenderer` - Renders the submitted geometry via a passed function.
    - `EntityModelFeatureRenderer` - Renders the submitted entity models.
    - `FeatureRenderDispatcher` - Dispatches all features to render from the submitted node collector objects.
    - `FlameFeatureRenderer` - Renders the submitted entity on fire animation.
    - `HitboxFeatureRenderer` - Renders the submitted entity hitbox.
    - `ItemFeatureRenderer` - Renders the submitted items.
    - `LeashFeatureRenderer` - Renders the submitted leash attached to entities.
    - `NameTagFeatureRenderer` - Renders the submitted name tags.
    - `ShadowFeatureRenderer` - Render the submitted entity shadow.
    - `TextFeatureRenderer` - Renders the submitted text.
- `net.minecraft.client.renderer.item.ItemModel$BakingContext` now takes in a `MaterialSet` and implements `SpecialModelRenderer$BakingContext`
- `net.minecraft.client.renderer.special`
    - `ChestSpecialRenderer` now takes in a `MaterialSet`
        - `*COPPER*` - Textures for the copper chest.
    - `ConduitSpecialRenderer` now takes in a `MaterialSet`
    - `CopperGolemStatueSpecialRenderer` - A special renderer for the copper golem statue as an item.
    - `HangingSignSpecialRenderer` now takes in a `MaterialSet` and a `Model$Simple` instead of a `Model`
    - `ShieldSpecialRenderer` now takes in a `MaterialSet`
    - `SpecialModelRenderer`
        - `$BakingContext` - The context used to bake a special item model.
        - `$Unbaked#bake` now takes in a `SpecialModelRenderer$BakingContext` instead of an `EntityModelSet`
    - `SpecialModelRenderers#createBlockRenderers` now takes in a `SpecialModelRenderer$BakingContext` instead of an `EntityModelSet`
    - `StandingSignSpecialRenderer` now takes in a `MaterialSet` and a `Model$Simple` instead of a `Model`
- `net.minecraft.client.renderer.texture`
    - `SpriteContents` now takes in an optional `AnimationMetadataSection` and `MetadataSectionType$WithValue` list instead of a `ResourceMetadata`
        - `metadata` -> `getAdditionalMetadata`, not one-to-one
    - `SpriteLoader`
        - `DEFAULT_METADATA_SECTIONS` is removed
        - `stitch` is now private
        - `runSpriteSuppliers` is now private
        - `loadAndStitch(ResourceManager, ResourceLocation, int, Executor)` isn removed
        - `loadAndStitch` now takes a set of `MetadataSectionType`s instead of a collection
        - `$Preparations`
            - `waitForUpload` is removed
            - `getSprite` - Returns the atlas sprite for a given resource location.
    - `TextureAtlasSprite#isAnimated` -> `SpriteContents#isAnimated`
- `net.minecraft.client.renderer.texture.atlas.SpriteResourceLoader#create` now takes a set of `MetadataSectionType`s instead of a collection
- `net.minecraft.client.resources.model`
    - `Material`
        - `sprite` -> `MaterialSet#get`
        - `buffer` now takes in a `MaterialSet`
    - `MaterialSet` - A map of material to its atlas sprite.
    - `ModelBakery` now takes in a `MaterialSet`
    - `ModelManager` is no longer `AutoCloseable`

## `Level#isClientSide` now private

The `Level#isClientSide` field is now private, so all queries must be made to the method version:

```java
// For some Level level
level.isClientSide();
```

- `net.minecraft.world.level.Level#isClientSide` field is now private

## Minor Migrations

The following is a list of useful or interesting additions, changes, and removals that do not deserve their own section in the primer.

### Atlas Handler Consolidation

The atlas handler has had some of its logic modified to consolidate other sprites and change how to obtain a `TextureAtlasSprite`.

First, map decorations, paintings, and GUI sprites are now proper atlases with their own sheets: `Sheets#MAP_DECORATIONS_SHEET`, `PAINTINGS_SHEET`, and `GUI_SHEET` respectively.

Obtaining the `TextureAtlasSprite` from said sheets are now completely routed through the `MaterialSet`: a functional interface that takes in a `Material` (basically a sheet location and texture location), and returns the associated `TextureAtlasSprite`. The `MaterialSet` handles texture grabs for item models, block entity renderers, and entity renderers:

```java
// Here is an example material to grab the apple texture from the appropriate sheet
public static final Material APPLE = new Material(
    TextureAtlas.LOCATION_BLOCKS, // The sheet where the item textures are stored
    ResourceLocation.fromNamespaceAndPath("minecraft", "item/apple") // The texture name according to the sprite contents
);
// You can also do the same using Sheets.ITEMS_MAPPER.defaultNamespaceApply("apple")

// For some item model
public class ExampleUnbakedItemModel implements ItemModel.Unbaked {
    
    // ...

    @Override
    public ItemModel bake(ItemModel.BakingContext ctx) {
        TextureAtlasSprite appleTexture = ctx.materials().get(APPLE);
        // ...
    }
}

// For some special item model
public class ExampleUnbakedSpecialModel implements SpecialModelRenderer.Unbaked {
    
    // ...

    @Override
    @Nullable
    public SpecialModelRenderer<?> bake(SpecialModelRenderer.BakingContext ctx) {
        TextureAtlasSprite appleTexture = ctx.materials().get(APPLE);
        // ...
    }
}

// For some block entity renderer
public class ExampleBlockEntityRenderer implements BlockEntityRenderer<ExampleBlockEntity> {

    public ExampleBlockEntityRenderer(BlockEntityRendererProvider.Context ctx) {
        TextureAtlasSprite appleTexture = ctx.materials().get(APPLE);
        // ...
    }

    // ...
}


// For some entity renderer
public class ExampleEntityRenderer implements EntityRenderer<ExampleEntity, ExampleEntityState> {

    public ExampleEntityRenderer(EntityRendererProvider.Context ctx) {
        TextureAtlasSprite appleTexture = ctx.getMaterials().get(APPLE);
        // ...
    }

    // ...
}
```

- `net.minecraft.client.Minecraft`
        - `getTextureAtlas` -> `AtlasManager#getAtlas`, not one-to-one
        - `getPaintingTextures`, `getMapDecorationTextures`, `getGuiSprites` -> `getAtlasManager`, not one-to-one
- `net.minecraft.client.gui.GuiSpriteManager` is removed
- `net.minecraft.client.particle.ParticleEngine` no longer takes in the `TextureManager`
    - `close` is removed
- `net.minecraft.client.renderer`
    - `MapRenderer` now takes in an `AtlasManager` instead of a `MapDecorationTextureManager`
    - `Sheets#GUI_SHEET`, `MAP_DECORATIONS_SHEET`, `PAINTINGS_SHEET` - Atlas textures.
- `net.minecraft.client.renderer.entity`
    - `EntityRenderDispatcher` now takes in the `AtlasManager`
    - `EntityRendererProvider$Context` now takes in the `AtlasManager`
        - `getAtlas` - Returns the atlas for that location.
- `net.minecraft.client.resources`
    - `MapDecorationTextureManager` class is removed
    - `PaintingTextureManager` class is removed
    - `TextureAtlasHolder` class is removed
- `net.minecraft.client.resources.model`
    - `AtlasSet` -> `AtlasManager`, not one-to-one
    - `ModelManager` now takes in a `AtlasManager` instead of the `TextureManager` and the max mipmap levels integer
        - `getAtlas` -> `MaterialSet#get`
        - `updateMaxMipLevel` -> `AtlasManager#updateMaxMipLevel`

### Double-Click Expansion

`GuiEventListener#mouseClicked` now takes in whether the click was in fact a double-click as a boolean.

```java
// For some Screen subclass (or AbstractWidget or any GUI object rendering)

@Override
public boolean mouseClicked(double x, double y, int button, boolean doubleClick) {
    // ...
    return false;
}
```

- `net.minecraft.client.gui.components.AbstractWidget#onClick` now takes in whether the button was double-clicked
- `net.minecraft.client.gui.components.events.GuiEventListener`
    - `DOUBLE_CLICK_THRESHOLD_MS` -> `MouseHandler#DOUBLE_CLICK_THRESHOLD_MS`
    - `mouseClicked` now takes in a boolean if the button has been double-clicked
- `net.minecraft.client.gui.screens.recipebook.RecipeBookPage#mouseClicked` now takes in a boolean if the button has been double-clicked

### Item Owner

Minecraft has further abstracted the holder of an item in item models to its own interface: `ItemOwner`. An `ItemOwner` is defined by three things: the `level` the owner is in, the `position` of the owner, and the Y rotation (in degrees) of the owner to indicate the facing direction. `ItemOwner`s are implemented on every `Entity`; however, there can also be detached owners by calling `ItemOwner#custom` or creating a `ItemOwner$CustomOwner`.

Currently, only the new shelf block uses item owners in a detached state so that compasses are not randomly spinning when placed.

- `net.minecraft.client.renderer.entity.ItemRenderer#renderStatic` now takes in an optional `ItemOwner` instead of a `LivingEntity`
- `net.minecraft.client.renderer.item`
    - `ItemModel#update` now takes in an `ItemOwner` instead of a `LivingEntity`
    - `ItemModelResolver#updateForTopItem`, `appendItemLayers` now takes in an `ItemOwner` instead of a `LivingEntity`
- `net.minecraft.client.renderer.item.properties.numeric`
    - `NeedleDirectionHelper#get`, `calculate` now takes in an `ItemOwner` instead of a `LivingEntity`
    - `RangeSelectItemModelProperty#get` now takes in an `ItemOwner` instead of a `LivingEntity`
    - `Time$TimeSource#get` now takes in an `ItemOwner` instead of a `Entity`
- `net.minecraft.world.entity`
    - `Entity` now implements `ItemOwner`
    - `ItemOwner` - Represents the owner of the item being rendered, or a nonspecific owner given the position, visual rotation, and level.

### Container User

As this new version of Minecraft introduces the copper golem (a vanilla mob that can interact with inventories), the concept of an entity that can use a container has been abstracted into its own interface: `ContainerUser`. `ContainerUser`s are expected to be implemented on some `LivingEntity` subclass. Since the actual inventory logic is handled elsewhere, `ContainerUser` only has two methods: `hasContainerOpen`, which checks whether the living entity is currently opening some container (e.g., barrel, chest); and `getContainerInteractionRange`, which determines the radius in blocks that the entity can interact with some container.

- `net.minecraft.world.Container`
    - `startOpen`, `stopOpen` now take in a `ContainerUser` instead of a `Player`
    - `getEntitiesWithContainerOpen` - Returns the list of `ContainerUser`s that are interacting with this container.
- `net.minecraft.world.entity.ContainerUser` - Typically an entity that can interact with and open a container.
- `net.minecraft.world.entity.player.Player` now implements `ContainerUser`
- `net.minecraft.world.level.block.entity.EnderChestBlockEntity#startOpen`, `stopOpen` now take in a `ContainerUser` instead of a `Player`

### Name And Id

Unless a `GameProfile` is needed, Minecraft now passes around a `NameAndId` for every player instead. As the name implies, `NameAndId` contains the UUID and name of the player, as that is usually all that's necessary for most logic involving a player entity.

- `net.minecraft.commands.arguments.GameProfileArgument`
    - `getGameProfiles` now return a collections of `NameAndId`s
    - `$Result#getNames` now return a collections of `NameAndId`s
- `net.minecraft.server`
    - `MinecraftServer`
        - `getProfilePermissions` now takes in a `NameAndId` instead of a `GameProfile`
        - `isSingleplayerOwner` now takes in a `NameAndId` instead of a `GameProfile`
    - `Services` now takes in a `UserNameToIdResolver` instead of a `GameProfileCache`
- `net.minecraft.server.players`
    - `GameProfileCache` -> `UserNameToIdResolver`, `CachedUserNameToIdResolver`; not one-to-one
    - `NameAndId` - An object that holds the UUID and name of a profile.
    - `PlayerList`
        - `load` -> `loadPlayerData`, not one-to-one
        - `canPlayerLogin` now takes in a `NameAndId` instead of a `GameProfile`
        - `disconnectAllPlayersWithProfile` now takes in a `UUID` instead of a `GameProfile`
        - `op`, `deop` now take in a `NameAndId` instead of a `GameProfile`
        - `isWhiteListed`, `isOp` now take in a `NameAndId` instead of a `GameProfile`
        - `canBypassPlayerLimit` now takes in a `NameAndId` instead of a `GameProfile`
    - `ServerOpList` now takes in `NameAndId` as its first generic
        - `canBypassPlayerLimit` now takes in a `NameAndId` instead of a `GameProfile`
    - `ServerOpListEntry` now takes in `NameAndId` as its generic and for the constructor
    - `UserBanList` now takes in `NameAndId` as its first generic
        - `isBanned` now takes in a `NameAndId` instead of a `GameProfile`
    - `UserBanListEntry` now takes in `NameAndId` as its generic and for the constructor
    - `UserWhiteList` now takes in `NameAndId` as its first generic
        - `isWhiteListed` now takes in a `NameAndId` instead of a `GameProfile`
    - `UserWhiteListEntry` now takes in `NameAndId` as its generic and for the constructor
- `net.minecraft.world.entity.player.Player#nameAndId` - Returns the name and id of the player.
- `net.minecraft.world.level.storage.PlayerDataStorage#load(Player, ProblemReporter)` -> `load(NameAndId)`, returns an `Optional<CompoundTag>`

### Typed Entity Data

Data components for storing block and entity data have been changed from using `CustomData` to the type safe variant `TypedEntityData`. `TypedEntityData` is pretty much the same as `CustomData`, storing a `CompoundTag`, except that it is represented by some `IdType` generic like `EntityType` or `BlockEntityType`. All of the methods within `CustomData` relating to block entities and entities have moved to `TypedEntityData`.

Spawn eggs use the `TypedEntityData` to store the `EntityType` that it will spawn. The logic of spawning is still tied to the `SpawnEggItem` subclass.

- `net.minecraft.core.component.DataComponents#ENTITY_DATA`, `BLOCK_ENTITY_DATA` are now `TypeEntityData` with `EntityType` and `BlockEntityType` ids, respectively
- `net.minecraft.world.entity.EntityType#updateCustomEntityTag` now takes in a `TypedEntityData` instead of `CustomData`
- `net.minecraft.world.item.Item$Properties#spawnEgg` - Adds the `ENTITY_DATA` component with the entity type.
- `net.minecraft.world.item.component`
    - `CustomData`
        - `CODEC_WITH_ID` is removed
        - `COMPOUND_TAG_CODEC` - A codec for a flattened compound tag.
        - `parseEntityId`, `parseEntityType` are removed
        - `loadInto` -> `TypedEntityData#loadInto`
        - `update`, `read`, `size` are removed
        - `contains` -> `TypedEntityData#contains`
        - `getUnsafe` -> `TypedEntityData#getUnsafe`
    - `TypedEntityData` - A custom data that provides a type-safe identifier.
- `net.minecraft.world.level.Spawner#appendHoverText`, `getSpawnEntityDisplayName` now take in a `TypedEntityData` instead of a `CustomData`
- `net.minecraft.world.level.block.entity.BeehiveBlockEntity$Occupant#entityData` is now a `TypedEntityData`

### Reload Listener Shared State

`PreparableReloadListener` now have an option to share their data between other reload listeners through the use of the `PreparableReloadListener#prepareSharedState`. `prepareSharedState` is ran on the main thread before `reload` is called, which can store arbitrary values to some `$SharedKey` as basically a dictionary lookup. Then, once `reload` is called, the stored data can be obtained and used via the key. Normally, the shared values used are some kind of `CompletableFuture` and joined within.

```java
public class FirstExampleListener implements PreparableReloadListener {

    // Create the key to write the shared data to
    public static final PreparableReloadListener.StateKey<CompletableFuture<Void>> EXAMPLE = new PreparableReloadListener.StateKey<>();

    @Override
    public void prepareSharedState(PreparableReloadListener.SharedState state) {
        // Add the data to the shared state
        state.set(EXAMPLE, CompletableFuture.allOf());
    }

    @Override
    public CompletableFuture<Void> reload(PreparableReloadListener.SharedState state, Executor tasksExecutor, PreparableReloadListener.PreparationBarrier barrier, Executor reloadExecutor) {
        // Use the shared state
        var example = state.get(EXAMPLE);
    }
}


public class SecondExampleListener implements PreparableReloadListener {

    @Override
    public CompletableFuture<Void> reload(PreparableReloadListener.SharedState state, Executor tasksExecutor, PreparableReloadListener.PreparationBarrier barrier, Executor reloadExecutor) {
        // Use the shared state in a different listener
        var example = state.get(FirstExampleListener.EXAMPLE);
    }
}
```

- `net.minecraft.server.packs.resources`
    - `PreparableReloadListener`
        - `reload` now takes in a `PreparableReloadListener$SharedState` instead of a `ResourceManager`
        - `prepareSharedState` - A method called before the reload listener is reloaded to share data between reloading listeners.
        - `$SharedState` - A general dictionary that can provide resources between reload listeners, though they should typically be in the form of `CompletableFuture`s.
        - `$StateKey` - A key into the shared state.
    - `SimpleReloadInstance$StateFactory#create` now takes in a `PreparableReloadListener$SharedState` instead of a `ResourceManager`

### Ticket Flags

`TicketType` has been updated to take a bit flag representing the flag's properties rather than a boolean and enum.

```java
// This needs to be registered to BuiltInRegistries#TICKET_TYPE
public static final TicketType EXAMPLE = new TicketType(
    20L, // The number of ticks before the ticket is timed out and removed, set to 0 if the ticket should never time out
    // The bit flags to set, see below for replacements and explanations of the new flags
    TicketType.FLAG_SIMULATION | TicketType.FLAG_LOADING
);
```

- `net.minecraft.server.level.TicketType` now takes in a bitmask of flags rather than a boolean and ticket use
    - `FLAG_PERSIST`, `persist` replace the boolean
    - `FLAG_LOADING` replaces `$TicketUse#LOADING`
    - `FLAG_SIMULATION` replaces `$TicketUse#SIMULATION`
    - `FLAG_KEEP_DIMENSION_ACTIVE`, `shouldKeepDimensionActive` - When set, prevents the level from unloading.
    - `FLAG_CAN_EXPIRE_IF_UNLOADED`, `canExpireIfUnloaded` - When set, the ticket expires if the chunk is unloaded.
    - `START` -> `PLAYER_SPAWN`, `SPAWN_SEARCH`
    - `$TicketUse` class is removed
- `net.minecraft.world.level.TicketStorage`
    - `shouldKeepDimensionActive` - Checks if the dimension active flag is set on any tickets in the storage.
    - `removeTicketIf` now takes a `$TicketPredicate` instead of a `BiPredicate`
    - `$TicketPredicate` - Tests the ticket at a given chunk position.

### New Tags

- `minecraft:block`
    - `wooden_shelves`
    - `copper_chests`
    - `lightning_rods`
    - `copper`
    - `copper_golem_statues`
    - `incorrect_for_copper_tool`
- `minecraft:item`
    - `wooden_shelves`
    - `copper_chests`
    - `lightning_rods`
    - `copper`
    - `copper_golem_statues`
    - `copper_tool_materials`
    - `repairs_copper_armor`

### List of Additions

- `com.minecraft.SharedConstants#RESOURCE_PACK_FORMAT_MINOR`, `DATA_PACK_FORMAT_MINOR` - The minor component of the pack version.
- `net.minecraft.client`
    - `KeyMapping#shouldSetOnIngameFocus` - Returns whether the key is mapped to some value on the keyboard.
    - `Minecraft#isOfflineDevelopedMode` - Returns whether the game is in offline developer mode.
    - `Options`
        - `invertMouseX` - When true, inverts the X mouse movement.
        - `toggleAttack` - When true, sets the attack key to be toggleable.
        - `toggleUse` - When true, sets the use key to be toggleable.
        - `sprintWindow` - Time window in ticks where double-tapping the forward key activates sprint.
- `net.minecraft.client.animation.definitions.CopperGolemAnimation` - The animations for the copper golem.
- `net.minecraft.client.data.models.BlockModelGenerators`
    - `and` - ANDs the given conditions together.
    - `createShelf` - Creates a shelf block state.
    - `addShelfPart` - Adds a variant for a model for all the shelf's states.
    - `forEachHorizontalDirection` - Performs an operation for a pair of directions and rotations.
    - `shelfCondition` - Constructs a condition based on the state of the shelf.
    - `createCopperGolemStatues` - Creates all copper golem statues.
    - `createCopperGolemStatue` - Creates a copper golem statue for the provided block, copper block, and weathered state.
    - `createCopperChests` - Creates all copper chests.
- `net.minecraft.client.data.models.model.ModelTemplate`
    - `SHELF_*` - Model templates for construct a shelf into a multipart.
    - `LIGHTNING_ROD` - Model template for a lightning rod.
- `net.minecraft.client.gui`
    - `Font$Provider` - A simple manager for providing the glyph source based on its location, along with the empty glyph.
    - `GlyphSource` - An interface that holds the baked glyphs based on its codepoint.
    - `Gui#renderDeferredSubtitles` - Renders the subtitles on screen.
    - `GuiGraphics#getSprite` - Gets a `TextureAtlasSprite` from its material.
- `net.minecraft.client.gui.components.MultilineTextField#selectWordAtCursor` - Selects the word whether the cursor is currently located.
- `net.minecraft.client.gui.components.events.GuiEventListener#shouldTakeFocusAfterInteraction` - When true, sets the element as focused when clicked.
- `net.minecraft.client.gui.font`
    - `FontSet`
        - `source` - Returns the glyph source depending given whether only non-fishy glyphs should be seen.
        - `$GlyphSource` - A source that can get the glyphs to bake.
    - `GlyphStitcher` - A class that creates `BakedGlyph`s from its glyph information.
- `net.minecraft.client.gui.font.glyphs.BakeableGlyph` - An interface that represents a glyph to bake into something usable by the renderer.
- `net.minecraft.client.gui.screens`
    - `LevelLoadingScreen`
        - `update` - Updates the current load tracker and reason.
        - `$Reason` - The reason for changing the level loading screen.
    - `Screen`
        - `panoramaShouldSpin` - Returns whether the rendered panorama should spin its camera.
        - `setNarrationSuppressTime` - Sets until when the narration should be suppressed for.
- `net.minecraft.client.gui.screens.options.OptionsScreen#CONTROLS` is now public
- `net.minecraft.client.main.GameConfig$GameData#offlineDeveloperMode` - Whether the game is offline and is ran for development.
- `net.minecraft.data.loot.packs`
    - `VanillaBlockInteractLoot` - A sub provider for vanilla block loot from entity interaction.
    - `VanillaChargedCreeperExplosionLoot` - A sub provider for vanilla entity loot from charged creeper explosions.
    - `VanillaEntityInteractLoot` - A sub provider for vanilla entity loot from entity interaction.
- `net.minecraft.data.recipes.RecipeProvider#shelf` - Constructs a shelf recipe from one item.
- `net.minecraft.gametest.framework.GameTestHelper#getTestDirection` - Gets the direction that the test is facing.
- `net.minecraft.network.syncher.EntityDataSerializers`
    - `WEATHERING_COPPER_STATE` - The weather state of the copper on the golem.
    - `COPPER_GOLEM_STATE` - The logic state of the copper golem.
- `net.minecraft.server.MinecraftServer`
    - `selectLevelLoadFocusPos` - Returns the loading center position of the server, usually the shared spawn position.
    - `getLevelLoadListener` - Returns the listener used to track level loading.
- `net.minecraft.server.level`
    - `ChunkLoadCounter` - Keeps track of chunk loading when a level is loading or the player spawns.
    - `ChunkMap#getLatestStatus` - Returns the latest status for the given chunk position.
    - `ServerChunkCache`
        - `hasActiveTickets` - Checks whether the current level has any active tickets keeping it loaded.
        - `addTicketAndLoadWithRadius` - Adds a ticket to some location and loads the chunk and radius around that location.
    - `ServerPlayer$SavedPosition` - Holds the current position of the player on disk.
- `net.minecraft.server.level.progress.ChunkLoadStatusView` - A status view for the loading chunks.
- `net.minecraft.server.network.ConfigurationTask#tick` - Calls the task every tick until it returns true, then finishes the task.
- `net.minecraft.server.network.config.PrepareSpawnTask` - A configuration task that finds the spawn of the player.
- `net.minecraft.server.packs.OverlayMetadataSection`
    - `codecForPackType` - Constructs a codec for the section given the pack type.
    - `forPackType` - Gets the metadata section type from the pack type.
- `net.minecraft.server.packs.metadata.MetadataSectionType#withValue`, `$WithValue` - Holds a metadata section along with its provided value.
- `net.minecraft.server.packs.metadata.pack`
    - `PackFormat` - Represents the major and minor revision of a pack.
    - `PackMetadataSection#forPackType` - Gets the metadata section type from the pack type.
- `net.minecraft.server.packs.repository.PackCompatibility#UNKNOWN` - The compatibility of the pack with the game is unknown as the major version is set to the max integer value.
- `net.minecraft.server.packs.resources.ResourceMetadata#getTypedSection`, `getTypedSections` - Gets the metadata sections for the provided types.
- `net.minecraft.util.InclusiveRange#map` - Maps a range from one generic to another.
- `net.minecraft.util.thread.BlockableEventLoop#shouldRunAllTasks` - Returns if there are any blocked tasks.
- `net.minecraft.world.entity`
    - `EntityType`
        - `STREAM_CODEC` - The stream codec for an entity type.
        - `$Builder#notInPeaceful` - Sets the entity to not spawn in peaceful mode.
    - `LivingEntity#dropFromEntityInteractLootTable` - Drops loot from a table from an entity interaction.
- `net.minecraft.world.entity.ai.behavior.TransportItemsBetweenContainers` - A behavior where an entity will move items between visited containers.
- `net.minecraft.world.entity.ai.memory.MemoryModuleType`
    - `VISITED_BLOCK_POSITIONS` - Important block positions visited.
    - `TRANSPORT_ITEMS_COOLDOWN_TICKS` - How many ticks to wait before transporting items.
- `net.minecraft.world.entity.animal.coppergolem`
    - `CopperGolem` - The copper golem entity.
    - `CopperGolemAi` - The brain logic for the copper golem.
    - `CopperGolemOxidationLevel` - A record holding the source events and textures for a given oxidation level.
    - `CopperGolemOxidationLevels` - All vanilla oxidation levels.
    - `CopperGolemState` - The current logic state the copper golem is in.
- `net.minecraft.world.item`
    - `Item$TooltipContext#isPeaceful` - Returns true if the difficulty is set to peaceful.
    - `ToolMaterial#COPPER` - The copper tool material.
- `net.minecraft.world.item.equipment`
    - `ArmorMaterials#COPPER` - The copper armor material.
    - `EquipmentAssets#COPPER` - The key reference to the copper equipment asset.
- `net.minecraft.world.level.block`
    - `Block#dropFromBlockInteractLootTable` - Drops the loot when interacting with a block.
    - `ChestBlock`
        - `chestCanConnectTo` - Returns whether the chest can merge with another block.
        - `getConnectedBlockPos` - Gets the connected block position of the chest.
        - `getOpenChestSound`, `getCloseChestSound` - Returns the sounds played on chest open / close.
    - `CopperChestBlock` - The block for the copper chest.
    - `CopperGolemStatueBlock` - The block for the copper golem statue.
    - `SelectableSlotContainer` - A container whose slot can be selected based on in-game interaction.
    - `ShelfBlock` - A block for the shelf.
    - `SideChainPartBlock` - A helper for providing how the chain of a shelf should be displayed.
    - `WeatheringCopper$WeatherState`
        - `BY_ID` - The mapper of enum ordinal to state.
        - `STREAM_CODEC` - The stream codec for the state.
        - `next` - Gets the next state in the ordinal.
        - `previous` - Gets the previous state in the ordinal.
    - `WeatheringCopperChestBlock` - The block for the weathering copper chest.
    - `WeatheringCopperGolemStatueBlock` - The block for the weathering copper golem statue.
    - `WeatheringLightningRodBlock` - The block for the weathering lightning rod.
- `net.minecraft.world.level.block.entity`
    - `CopperGolemStatueBlockEntity` - The block entity for the copper golem statue.
    - `ListBackedContainer` - A container that is baked by a list of items.
    - `ShelfBlockEntity` - The block entity for the shelf.
- `net.minecraft.world.level.block.state.BlockBehaviour#shouldChangedStateKeepBlockEntity`, `$BlockStateBase#shouldChangedStateKeepBlockEntity` - Returns whether the block entity should be kept if the block is changed to a different block.
- `net.minecraft.world.level.block.state.properties.SideChainPart` - The location of where the chain of an object is connected to.
- `net.minecraft.world.level.levelgen`
    - `Beardifier#EMPTY` - An instance with no pieces or junctions.
    - `DensityFunction#invert` - Inverts the density output by putting the result one over the value.
    - `DensityFunctions#findTopSurface` - Finds the top surface between the two bounds.
    - `NoiseChunk#maxPreliminarySurfaceLevel` - Returns the largest preliminary surface level.
    - `NoiseRouterData#NOISE_ZERO` - A constant containing the base noise level for layer zero in the overworld.
- `net.minecraft.world.level.levelgen.blending.Blender#isEmpty` - Returns whether there is no blending data present.
- `net.minecraft.world.level.levelgen.structure.BoundingBox#encapsulating` - Returns the smallest box that includes the given boxes.
- `net.minecraft.world.level.levelgen.structure.structures.JigsawStructure$MaxDistance` - The maximum horizontal and vertical distance the jigsaw can expand to.
- `net.minecraft.world.level.storage.loot.LootContext$EntityTarget`
    - `TARGET_ENTITY` - The entity being targeted by another, typically the object dropping the loot.
    - `INTERACTING_ENTITY` - The entity interacting with the object dropping the loot.
- `net.minecraft.world.level.storage.loot.parameters`
    - `LootContextParams`
        - `TARGET_ENTITY` - The entity being targeted by another, typically the object dropping the loot.
        - `INTERACTING_ENTITY` - The entity interacting with the object dropping the loot.
    - `LootContextParamSets`
        - `ENTITY_INTERACT` - An entity being interacted with.
        - `BLOCK_INTERACT` - A block being interacted with.

### List of Changes

- `com.mojang.blaze3d.font.GlyphInfo#bake` now takes in a `GlyphStitcher` instead of a function
    - The function behavior is replaced by `GlyphStitcher#stitch`
- `com.mojang.blaze3d.opengl`
    - `DirectStateAccess#bufferSubData`, `mapBufferRange`, `unmapBuffer`, `flushMappedBufferRange` now take in the buffer usage bit mask
    - `VertexArrayCache#bindVertexArray` can now take in a nullable `GlBuffer`
- `com.mojang.blaze3d.vertex`
    - `PoseStack$Pose#set` is now public
    - `VertexConsumer#addVertexWith2DPose` no longer takes in the z component
- `net.minecraft`
    - `CrashReportCategory#formatLocation` now has an overload that doesn't take in the `LevelHeightAccessor`
    - `DetectedVersion#createFromConstants` -> `createBuiltIn`, now public, taking in the id, name, and optional stable `boolean`
    - `SharedConstants#*_PACK_FORMAT` -> `*_PACK_FORMAT_MAJOR`
    - `WorldVersion`
        - `packVersion` now returns a `PackFormat`
        - `$Simple` now takes in a `PackFormat` for the `resourcePackVersion` and `datapackVersion`
- `net.minecraft.client`
    - `Minecraft#setLevel` no longer takes in the `ReceivingLevelScreen$Reason`
    - `Options#invertYMouse` -> `invertMouseY`
    - `ToggleKeyMapping` now has an overload that takes in an input type
    - `User` no longer takes in the user type
- `net.minecraft.client.animation.Keyframe` now has an overload that takes in the `preTarget` and `postTarget` instead of one simple `target`
- `net.minecraft.client.data.models.BlockModelGenerators`
    - `condition` now has an overload that takes in an enum or boolean property
    - `createLightningRod` now takes in the regular and waxed blocks
- `net.minecraft.client.gui`
    - `Font` no longer takes in a function and boolean, instead a `Font$Provider`
        - `random` is now private
        - `$PreparedTextBuilder#accept` now has an override that takes in a `BakeableGlyph` instead of its codepoint
    - `GuiGraphics#submitSignRenderState` now takes in a `Model$Simple` instead of a `Model`
- `net.minecraft.client.gui.font.FontSet` now takes in a `GlyphStitcher` instead of a `TextureManager`
    - `getGlyphInfo`, `getGlyph` -> `getGlyph`, now package-private, not one-to-one
    - `getRandomGlyph` now takes in a `RandomSource` and a codepoint instead of the `GlyphInfo`
    - `whiteGlyph` now returns a `BakeableGlyph`
- `net.minecraft.client.gui.render.GuiRenderer#MIN_GUI_Z` is now public
- `net.minecraft.client.gui.render.state.GuiElementRenderState#buildVertices` no longer takes in the `z` component
- `net.minecraft.client.gui.render.state.pip.GuiSignRenderState` now takes in a `Model$Simple` instead of a `Model`
- `net.minecraft.client.gui.screens`
    - `LevelLoadingScreen` now takes in a `LevelLoadTracker` and `LevelLoadingScreen$Reason`
    - `ReceivingLevelScreen` has been merged into `LevelLoadingScreen`
    - `Screen`
        - `renderWithTooltip` -> `renderWithTooltipAndSubtitles`
        - `$NarratableSearchResult` is now a record
- `net.minecraft.client.multiplayer`
    - `ClientHandshakePacketListenerImpl` now takes in a `LevelLoadTracker`
    - `CommonListenerCookie` now takes in a `LevelLoadTracker`
    - `LevelLoadStatusManager` -> `LevelLoadTracker`, not one-to-one
- `net.minecraft.client.multiplayer.chat.ChatListener#clearQueue` -> `flushQueue`
- `net.minecraft.client.server.IntegratedServer` now takes in a `LevelLoadListener` instead of a `ChunkProgressListenerFactory`
- `net.minecraft.client.sounds`
    - `SoundEngine#updateCategoryVolume` no longer takes in the gain
    - `SoundManager#updateSourceVolume` no longer takes in the gain
- `net.minecraft.core.Registry#getRandomElementOf` -> `HolderGetter#getRandomElementOf`
- `net.minecraft.gametest.framework`
    - `GameTestHelper`
        - `spawn` now has overloads that take in some number of entities to spawn
        - `setBlock` now has overloads to take in a direction for block facing
        - `fail` now has an overload that takes in a string
    - `GameTestRunner` now takes in whether the level should be cleared for more space to spawn between batches
        - `$Builder#haltOnError` no longer takes in any parameters
- `net.minecraft.nbt.NbtUtils#addDataVersion`, `addCurrentDataVersion` now has an overload that takes in a `Dynamic` instead of a `CompoundTag` or `ValueOutput`
- `net.minecraft.network.chat.Style` is now final
- `net.minecraft.server`
    - `SPAWN_POSITION_SEARCH_RADIUS` is now public
    - `MinecraftServer` now takes in a `LevelLoadListener` instead of a `ChunkProgressListenerFactory`
        - `createLevels` no longer takes in a `ChunkProgressListener`
        - `getProfileCache` -> `nameToIdCache`, not one-to-one
- `net.minecraft.server.dedicated.DedicatedServer` now takes in a `LevelLoadListener` instead of a `ChunkProgressListenerFactory`
- `net.minecraft.server.level`
    - `ChunkMap` no longer takes in the `ChunkProgressListener`
        - `getTickingGenerated` -> `allChunksWithAtLeastStatus`, not one-to-one
    - `PlayerRespawnLogic` -> `PlayerSpawnFinder`, not one-to-one
    - `ServerChunkCache` no longer takes in the `ChunkProgressListener`
    - `ServerLevel` no longer takes in the `ChunkProgressListener`
        - `waitForChunkAndEntities` -> `waitForEntities`, not one-to-one
- `net.minecraft.server.level.progress`
    - `ChunkProgressListener` -> `LevelLoadListener`, not one-to-one
    - `ChunkProgressListenerFactory` -> `MinecraftServer#createChunkLoadStatusView`, not one-to-one
    - `LoggerChunkProgressListener` -> `LoggingLevelLoadListener`, not one-to-one
    - `ProcessorChunkProgressListener`, `StoringChunkProgressListener` -> `LevelLoadProgressListener`, not one-to-one
- `net.minecraft.server.packs`
    - `AbstractPackResources#getMetadataFromStream` now takes in a `PackLocationInfo`
    - `OverlayMetadataSection`
        - `TYPE` -> `CLIENT_TYPE`, `SERVER_TYPE`
        - `overlaysForVersion` now takes in a `PackFormat` instead of an integer
        - `$OverlayEntry` now takes in an `InclusiveRange<PackFormat>` instead of an `InclusiveRange<Integer>`
            - `isApplicable` now takes in a `PackFormat` instead of an integer
- `net.minecraft.server.packs.metadata.pack.PackMetadataSection` now takes in an `InclusiveRange<PackFormat>` instead of the raw pack format and integer pack.
    - `CODEC` -> `FALLBACK_CODEC`, now private
    - `TYPE` -> `CLIENT_TYPE`, `SERVER_TYPE`, `FALLBACK_TYPE`
- `net.minecraft.server.packs.repository`
    - `Pack#readPackMetadata` now takes in a `PackFormat` and `PackType` instead of an integer
    - `PackCompatibility#forVersion` now takes in `PackFormat`s instead of integers
- `net.minecraft.world.entity`
    - `AgeableMob#finalizeSpawn` is now nullable
    - `Entity`
        - `startRiding(Entity)` is now final
        - `startRiding(Entity, boolean)` now takes in an additional boolean of whether to trigger the game event and criteria triggers
        - `killedEntity` now takes in the `DamageSource`
    - `EntityType` now takes in whether the entity is allowed in peaceful mode
        - `create`, `loadEntityRecursive` now has an overload that takes in an `EntityType`
    - `LivingEntity`
        - `shouldDropLoot` now takes in the `ServerLevel`
        - `dropFromLootTable` now has overloads to take in a specific loot table key and how the items should be dispensed
    - `Mob#shouldDespawnInPeaceful` -> `EntityType#isAllowedInPeaceful`, not one-to-one
- `net.minecraft.world.entity.animal.Animal#usePlayerItem` -> `Mob#usePlayerItem`
- `net.minecraft.world.entity.animal.armadillo.Armadillo#brushOffScute` now takes in an `Entity` and `ItemStack`
- `net.minecraft.world.item.SpawnEggItem` no longer takes in the `EntityType`
    - `spawnsEntity` no longer takes in the `HolderLookup$Provider`
    - `getType` no longer takes in the `HolderLookup$Provider`
- `net.minecraft.world.item.component.Bees#STREAM_CODEC` now requires a `RegistryFriendlyByteBuf`
- `net.minecraft.world.level.block`
    - `BeehiveBlock#dropHoneycomb(Level, BlockPos)` -> `dropHoneyComb(ServerLevel, ItemStack, BlockState, BlockEntity, Entity, BlockPos)`
    - `CaveVines#use` entity is no longer nullable
    - `ChestBlock` now takes in an open and close sound
    - `ChiseledBookShelfBLock` now implements `SelectableSlotContainer`
        - `BOOKS_PER_ROW` is now private
- `net.minecraft.world.level.block.entity`
    - `ChiseledBookShelfBlockEntity` now implements `ListBackedContainer`
        - `count` -> `ListBackedContainer#count`
    - `ContainerOpenersCounter`
        - `isOwnContainer` is now public
        - `incrementOpeners`, `decrementOpeners` now takes in a `LivingEntity` instead of a `Player`
        - `getPlayersWithContainerOpen` -> `getEntitiesWithContainerOpen`, now public, not one-to-one
    - `SkullBlockEntity`
        - `CHECKED_MAIN_THREAD_EXECUTOR` -> `ResolvableProfile#CHECKED_MAIN_THREAD_EXECUTOR`
        - `setup` -> `ResolvableProfile#setupResolver`
        - `clear` -> `ResolvableProfiler#clearResolver`
- `net.minecraft.world.level.block.state.BlockBehaviour#getAnalogOutputSignal`, `$BlockStateBase#getAnalogOutputSignal` now takes in the direction the signal is coming from
- `net.minecraft.world.level.levelgen`
    - `Beardifier` now takes in lists instead of iterators and a nullable `BoundingBox`
    - `NoiseRouter#initialDensityWithoutJaggedness` -> `preliminarySurfaceLevel`
- `net.minecraft.world.level.levelgen.structure.pools.JigsawPlacement#addPieces` now takes in a `JigsawStructure$MaxDistance` instead of an integer
- `net.minecraft.world.level.levelgen.structure.structures.JigsawStructure` now takes in a `JigsawStructure$MaxDistance` instead of an integer

### List of Removals

- `com.mojang.blaze3d.audio.Listener#setGain`, `getGain`
- `com.mojang.blaze3d.vertex.DefaultVertexFormat#BLIT_SCREEN`
- `net.minecraft.SharedConstants#VERSION_STRING`
- `net.minecraft.client`
    - `Minecraft#getProgressListener`
    - `Options#RENDER_DISTANCE_TINY`, `RENDER_DISTANCE_NORMAL`
    - `User#getType`, `$Type`
- `net.minecraft.client.gui.screens.LevelLoadingScreen#renderChunks`
- `net.minecraft.gametest.framework.GameTestTicker#startTicking`
- `net.minecraft.server.MinecraftServer#getSpawnRadius`
- `net.minecraft.server.level`
    - `ServerChunkCache#getTickingGenerated`
    - `ServerPlayer#loadGameTypes`
- `net.minecraft.server.packs.resources.ResourceMetadata`
    - `copySections`
    - `$Builder`
- `net.minecraft.world.entity.Entity#spawnAtLocation(ServerLevel, ItemLike, int)`
- `net.minecraft.world.entity.item.ItemEntity#copy`
- `net.minecraft.world.entity.monster`
    - `Creeper#canDropMobsSkull`, `increaseDroppedSkulls`
    - `Zombie#getSkull`
- `net.minecraft.world.level.GameRules#RULE_SPAWN_CHUNK_RADIUS`
- `net.minecraft.world.level.block.entity.SkullBlockEntity#fetchGameProfile`
