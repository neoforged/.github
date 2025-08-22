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

The entirety of the rendering pipeline, from entities to block entities, has been reworked into a submission / render phase system known as features. It is likely that the feature system will also take over particles given that the render phase calls have already been added, although they currently do nothing. This guide will go over the basics of the feature system itself followed by how each major type are implemented using it.

### Submission and Rendering

The feature system, like GUIs, is broken into two phases: submission and rendering. The submission phase is handled by the `SubmitNodeCollector`, which basically collects the data required to abstractly render an object to the screen. This is all done through the `submit*` methods, which generally take a `PoseStack` to place the object in 3D space, and any other required data such as the model, state, render type, etc.

Here is a quick overview of what each method takes in:

Method                 | Parameters
:---------------------:|:----------
`submitHitbox`         | A pose stack, render state of the entity, and the hitboxes render state
`submitShadow`         | A pose stack, the shadow radius, and the shadow pieces
`submitNameTag`        | A pose stack, an optional position offset, the text component, if the text should be seethrough (like when sneaking), light coordinates, and square distance to the camera
`submitText`           | A pose stack, the XY offset, the text sequence, whether to add a drop shadow, the font display mode, light coordinates, color,  background color, and outline color
`submitFlame`          | A pose stack, render state of the entity, and a rotation quaternion
`submitLeash`          | A pose stack and the leash state
`submitModel`          | A pose stack, entity model, render state, render type, light coordinates, overlay coordinates, tint color, an optional texture, outline color, and an optional crumbling overlay
`submitModelPart`      | A pose stack, model part, render type, light coordinates, overlay coordinates, an optional texture, whether to use item glint over entity glint if render type is not transparent, whether to render the glint overlay, and tint color
`submitBlock`          | A pose stack, block state, light coordinates, overlay coordinates, and outline color
`submitMovingBlock`    | A pose stack and the moving block render state
`submitBlockModel`     | A pose stack, the render type, block state model, RGB floats, light coordinates, overlay coordinates, and outline color
`submitItem`           | A pose stack, item display context, light coordinates, overlay coordinates, outline color, tint layers, quads, render type, and foil type
`submitCustomGeometry` | A pose stack, render type, and a function that takes in the current pose and `VertexConsumer` to create the mesh

Technically, the `submit*` methods are provided by the `OrderedSubmitNodeCollector`, of which the `SubmitNodeCollector` extends. This is because features can be submitted to different orders, which function similarly to strata in GUIs. By default, all submit calls are pushed onto order 0. Using `SubmitNodeCollector#order` with some integer and then calling the `submit*` method, you can have an object render before or after all features on a given order. This is stored as an AVL tree, where each order's data is stored in a `SubmitNodeCollection`. With the current default feature rendering order, this is only used in very specific circumstances, such as rendering a slime's outer body or equipment layers.

```java
// Assume we have access to the `SubmitNodeCollector` collector

// This will render in order 0
collector.submitModel(...);

// This will render before the `submitModel` call
collector.order(-1).submitShadow(...);

// This will render after the `submitModel` call
collector.order(1).submitNameTag(...);
```

The render phase is handled through the `FeatureRenderDispatcher`, which renders the object using their submitted feature renderers. What are feature renderers? Quite literally an arbitrary method that loops through the node contents its going to push to the buffer. Currently, for a given order, the features push their vertices like so: shadows, models, model parts, flame animations, entity name tags, arbitrary text, hitboxes, leashes, items, blocks, and finally custom render pipelines. Each order, starting from the smallest number to the largest, will rerun all of the feature renders until it reaches the end of the tree. All submissions are then cleared for next use.

Most of the feature dispatchers are simply run a loop except for `ModelFeatureRenderer`, which sorts its translucent models by distance from the camera and adds them to the buffer after all opaque models.

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

### Block Entity Renderer

`BlockEntityRenderer`s also use the new submission method, replacing almost every `render*` with `submit`. Instead of taking in the `MultiBufferSource`, the method now takes in the `ModelFeatureRenderer$CrumblingOverlay`, for handling block breaking on `Model`s; and the `SubmitNodeCollector` for pushing the elements to render. Like entities, when submitting any element, the location in 3D space is taken by getting the last pose on the `PoseStack` and storing that for future use.

```java
// A basic block entity renderer

// We will assume all the classes listed exist
public class ExampleBlockEntityRenderer implements BlockEntityRenderer<ExampleBlockEntity> {

    public ExampleBlockEntityRenderer(BlockEntityRendererProvider.Context ctx) {
        
    }

    @Override
    public void submit(ExampleBlockEntity blockEntity, float partialTick, PoseStack poseStack, int lightCoords, int overlayCoords, Vec3 cameraPos, @Nullable ModelFeatureRenderer.CrumblingOverlay crumblingOverlay, SubmitNodeCollector collector) {
        // An example of submitting something
        collector.submitModel(..., crumblingOverlay);
    }
}
```

### Special Item Models

Since special item models also make use of custom rendering, they have been updated to with the `submit` change, only replacing `MultiBufferSource` with the `SubmitNodeCollector`.

```java
// A basic special item model

// We will assume all the classes listed exist
public class ExampleSpecialModelRenderer implements NoDataSpecialModelRenderer {

    public ExampleSpecialModelRenderer() {}

    @Override
    public void submit(ItemDisplayContext displayContext, PoseStack poseStack, SubmitNodeCollector collector, int lightCoords, int overlayCoords, boolean hasFoil) {
        // An example of submitting something
        collector.submitModelPart(...);
    }

    @Override
    public void getExtents(Set<Vector3f> extents) {}
}
```

- `assets/minecraft/shaders/core/blit_screen.json` -> `screenquad.json`, using no-format triangles instead of positioned quads
- `net.minecraft.client.gui.GuiGraphics`
    - `renderOutline` -> `submitOutline`
    - `renderDeferredTooltip` -> `renderDeferredElements`, not one-to-one
- `net.minecraft.client.gui.render.GuiRenderer` now takes in the `SubmitNodeCollector` and `FeatureRenderDispatcher`
    - `MIN_GUI_Z` is now public
- `net.minecraft.client.gui.render.pip`
    - `GuiBannerResultRenderer` now takes in a `MaterialSet`
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
    - `CopperGolemStatueModel` - A model for the coper golem statue.
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
    - `ItemInHandRenderer`
        - `renderItem` now takes in a `SubmitNodeCollector` instead of a `MultiBufferSource`
        - `renderHandsWithItems` now takes in a `SubmitNodeCollector` instead of a `MultiBufferSource$BufferSource`
    - `LevelRenderer` now takes in the `LevelRenderState` and `FeatureRenderDispatcher`
        - `getSectionRenderDispatcher` is now nullable
    - `LevelRenderState` - The render state of the dynamic features in a level.
    - `MapRenderer#render` now takes in a `SubmitNodeCollector` instead of a `MultiBufferSource`
    - `OrderedSubmitNodeCollector` - A submission handler for holding elements to be drawn in a given order to the screen whenever the features are dispatched.
    - `OutlineBufferSource` no longer takes in any parameters
        - `setColor` now takes in a single integer
    - `PlayerSkinRenderCache` - A render cache for the player skins.
    - `RenderPipelines`
        - `GUI_TEXT` - The pipeline for text in a gui.
        - `GUI_TEXT_INTENSITY` - The pipeline for text intensity when not colored in a gui.
    - `RenderType#pipeline` - The `RenderPipeline` the type uses.
    - `ScreenEffectRenderer` now takes in a `MaterialSet`
        - `renderScreenEffect` now takes in a `SubmitNodeCollector`
    - `ShapeRenderer`
        - `renderLineBox` now takes in a `PoseStack$Pose` instead of a `PoseStack`
        - `renderFace` now takes in a `Matrix4f` instead of a `PoseStack`
    - `Sheets`
        - `BLOCK_ENTITIES_MAPPER` - A mapper for block textures onto block entities.
        - `*COPPER*` - Materials for copper chests.
    - `SpecialBlockModelRenderer`
        - `vanilla` now takes in a `SpecialModelRenderer$BakingContext` instead of an `EntityModelSet`
        - `renderByBlock` now takes in a `SubmitNodeCollector` instead of a `MultiBufferSource`
    - `SubmitNodeCollection` - An implementation of `OrderedSubmitNodeCollector` that holds the submitted features in separate lists.
    - `SubmitNodeCollector` - An `OrderedSubmitNodeCollector` that provides a method to change the current order that an element will be rendered in.
    - `SubmitNodeStorage` - A storage of collections held by some order.
- `net.minecraft.client.renderer.block`
    - `BlockRenderDispatcher` now takes in a `MaterialSet`
    - `MovingBlockRenderState` - A render state for a moving block that implements `BlockAndTintGetter`.
    - `LiquidBlockRenderer#setupSprites` now take in the `BlockModelShaper` and `MaterialSet`
- `net.minecraft.client.renderer.blockentity`
    - Most methods here that take in the `MultiBufferSource` have been replaced by a `SubmitNodeCollector`, and a `ModelFeatureRenderer$CrumblingOverlay` if the method is not used for item rendering
    - Most methods change their name from `render*` to `submit*`
    - `AbstractSignRenderer`
        - `getSignModel` now returns a `Model$Simple`
        - `renderSign` now takes in a `Model$Simple`
    - `BannerRenderer` has an overload that takes in a `SpecialModelRenderer$BakingContext`
        - The `EntityModelSet` constructor now takes in the `MaterialSet`
        - `renderPatterns` now takes in the `MaterialSet`
    - `BedRenderer` has an overload that takes in a `SpecialModelRenderer$BakingContext`
        - The `EntityModelSet` constructor now takes in the `MaterialSet`
    - `BlockEntityRenderDispatcher` now takes in the `MaterialSet` and `PlayerSkinRenderCache`
        - `render` -> `submit`
    - `BlockEntityRenderer#render` -> `submit`
    - `BlockEntityRendererProvider$Context` is now a record, taking in a `MaterialSet` and `PlayerSkinRenderCache`
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
        - `finalizeRenderState` - Extracts the information of the render state as a last step after `extractRenderState`, such as shadows.
    - `EntityRendererProvider$Context` now takes in a `PlayerSkinRenderCache`
        - `getModelManager` is removed
        - `getMaterials` - Returns a mapper of material to atlas sprite.
        - `getPlayerSkinRenderCache` - Gets the render cache of player skins.
    - `ItemEntityRenderer`
        - `renderMultipleFromCount` -> `submitMultipleFromCount`
        - `renderMultipleFromCount(PoseStack, MultiBufferSource, int, ItemClusterRenderState, RandomSource)` -> `renderMultipleFromCount(PoseStack, SubmitNodeCollector, int, ItemClusterRenderState, RandomSource)`
    - `ItemRenderer`
        - `getArmorFoilBuffer` -> `getFoilRenderTypes`, not one-to-one
        - All `renderStatic` -> `renderUpwardsFrom`, now taking in a `SubmitNodeCollector` instead of a `MultiBufferSource`
        - `getBoundingBox` - Gets the model bounding box of the stack.
    - `PiglinRenderer` takes in a `ArmorModelSet` instead of a `ModelLayerLocation`
    - `TntMinecartRenderer#renderWhiteSolidBlock` -> `submitWhiteSolidBlock`, now takes in an outline color
    - `ZombieRenderer` takes in a `ArmorModelSet` instead of a `ModelLayerLocation`
    - `ZombifiedPiglinPiglinRenderer` takes in a `ArmorModelSet` instead of a `ModelLayerLocation`
- `net.minecraft.client.renderer.entity.layers`
    - `BlockDecorationLayer` - A layer that handles a block model transformed by an entity.
    - `BreezeWindLayer` now takes in the `EntityModelSet` instead of the `EntityRendererProvider$Context`
    - `CustomHeadLayer` now takes in the `PlayerSkinRenderCache`
    - `EquipmentLayerRenderer#renderLayers` now takes in the render state, `SubmitNodeCollector`, outline color, and an initial order instead of a `MultiBufferSource`
    - `HumanoidArmorLayer` now takes in `ArmorModelSet`s instead of models
        - `setPartVisibility` is removed
    - `ItemInHandLayer#renderArmWithItem` -> `submitArmWithItem`
    - `LivingEntityEmissiveLayer` now takes in a function for the texture instead of a `ResourceLocation` and a model instead of the `$DrawSelector`
        - `$DrawSelector` is removed
    - `RenderLayer`
        - `renderColoredCutoutModel`, `coloredCutoutModelCopyLayerRender` now takes in a `Model` instead of an `EntityModel`, a `SubmitNodeCollector` instead of a `MultiBufferSource`, and an integer representing the order layer for rendering
        - `render` -> `submit`, taking in a `SubmitNodeCollector` instead of a `MultiBufferSource`
    - `SimpleEquipmentLayer` now takes in an order integer
    - `StuckInBodyLayer` now has an additional generic for the render state, also taking in the render state in the constructor
    - `VillagerProfessionLayer` now takes in two models
- `net.minecraft.client.renderer.entity.player.PlayerRenderer#render*Hand` now takes in a `SubmitNodeCollector` instead of a `MultiBufferSource`
- `net.minecraft.client.renderer.entity.state`
    - `CopperGolemRenderState` - The render state for the copper golem entity.
    - `EntityRenderState`
        - `NO_OUTLINE` - A constant that represents the color for no outline.
        - `outlineColor` - The outline color of the entity.
        - `lightCoords` - The packed light coordinates used to light the entity.
        - `shadowPieces`, `$ShadowPiece` - Represents the relative coordinates of the shadow(s) the entity is casting.
    - `FallingBlockRenderState` fields and implementations have all been moved to `MovingBlockRenderState`
    - `LivingEntityRenderState#appearsGlowing` -> `EntityRenderState#appearsGlowing`, now a method
    - `PaintingRenderState#lightCoords` -> `lightCoordsPerBlock`
    - `PlayerRenderState#useItemRemainingTicks`, `swinging` are removed
    - `WitherSkullRenderState#xRot`, `yRot` -> `modeState`, not one-to-one
- `net.minecraft.client.renderer.feature`
    - `BlockFeatureRenderer` - Renders the submitted blocks, block models, or falling blocks.
    - `CustomFeatureRenderer` - Renders the submitted geometry via a passed function.
    - `FeatureRenderDispatcher` - Dispatches all features to render from the submitted node collector objects.
    - `FlameFeatureRenderer` - Renders the submitted entity on fire animation.
    - `HitboxFeatureRenderer` - Renders the submitted entity hitbox.
    - `ItemFeatureRenderer` - Renders the submitted items.
    - `LeashFeatureRenderer` - Renders the submitted leash attached to entities.
    - `ModelFeatureRenderer` - Renders the submitted `Model`s.
    - `ModelPartFeatureRenderer` - Renders the submitted `ModelPart`s.
    - `NameTagFeatureRenderer` - Renders the submitted name tags.
    - `ShadowFeatureRenderer` - Render the submitted entity shadow.
    - `TextFeatureRenderer` - Renders the submitted text.
- `net.minecraft.client.renderer.item`
    - `ItemModel$BakingContext` now takes in a `MaterialSet` and `PlayerSkinRenderCache`, and implements `SpecialModelRenderer$BakingContext`
    - `ItemStackRenderState#render` -> `submit`, taking in a `SubmitNodeCollector` instead of a `MultiBufferSource` and an outline color
- `net.minecraft.client.renderer.special`
    - Most methods here that take in the `MultiBufferSource` have been replaced by `SubmitNodeCollector`
    - `ChestSpecialRenderer` now takes in a `MaterialSet`
        - `*COPPER*` - Textures for the copper chest.
    - `ConduitSpecialRenderer` now takes in a `MaterialSet`
    - `CopperGolemStatueSpecialRenderer` - A special renderer for the copper golem statue as an item.
    - `HangingSignSpecialRenderer` now takes in a `MaterialSet` and a `Model$Simple` instead of a `Model`
    - `NoDataSpecialModelRenderer#render` -> `submit`
    - `PlayerHeadSpecialRenderer` now takes in the `PlayerSkinRenderCache`
    - `ShieldSpecialRenderer` now takes in a `MaterialSet`
    - `SpecialModelRenderer`
        - `render` -> `submit`
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
    - `ModelBakery` now takes in a `MaterialSet` and `PlayerSkinRenderCache`
    - `ModelManager` is no longer `AutoCloseable`
        - The constructor takes in the `PlayerSkinRenderCache`

## The Font Glyph Pipeline

Glyphs have been partially reworked in the backend to both unify both characters and their effects to a single renderable and add a baking flow, albeit delayed. This will provide a brief overview of this new flow.

Let's start from the top when the assets are reloaded, causing the `FontManager` to run. The font definitions are loaded and passed to their appropriate `GlyphProviderDefinition`, which for this example with immediately unpack into a `GlyphProvider$Loader`. The loader reads whatever information it needs (most likely a texture) and maps it to an associated codepoint as an `UnbakedGlyph`. An `UnbakedGlyph` represents a single character containing its `GlyphInfo` with the position metadata and a `bake` method to write it to a texture. `bake` takes in a `UnbakedGlyph$Stitcher`, which itself takes in a `GlyphBitmap` to correctly position and upload the glyph along with the `GlyphInfo` for any offset adjustments. Then, after and resolving and finalizing, the `FontSet`s are created using `FontSet#reload`. This resets the textures and cache and calls `FontSet#selectProviders` to store the active list of `GlyphProvider`s and populates the `glyphsByWidth` by mapping the character advance to a list of matching codepoints. At this point, all of the loading has been done. While the glyphs are stored in their unbaked format, they are not baked and cached until that specific character is rendered.

Now, let's move onto the `FontDescription`. A `FontDescription` provides a one-to-one map to a `GlyphSource`, which is responsible for obtaining the glyphs. Vanilla provides two `FontDescription`s: `$Resource` for mapping to the `FontSet` (the font JSON name) as mentioned above, and `$AtlasSprite` to interpret an atlas as a bunch of glyphs. For a given `Component`, the `FontDescription` used can be set as its `Style` via `Style#withFont`.

So, let's assume you are drawing text to the screen using `FontDescription#DEFAULT`, which uses the defined `assets/minecraft/font/default.json`. Eventually, this will either call `Font#drawInBatch` or `drawInBatch8xOutline`. Then, whether through the `StringSplitter` or directly, the `GlyphSource` is obtained from the `FontDescription`. Then, `GlyphSource#getGlyph` is called. For the `FontDescription$AtlasSprite`, this just returns a wrapper around the `TextureAtlasSprite`. For the `$Resource` though, this internally calls `FontSet#getGlyph`, which stores the `UnbakedGlyph` as part of a `FontSet$DelayedBake`, which it then immediately resolves to the `BakedGlyph` by calling `UnbakedGlyph#bake` to write the glyph to some texture.

A `BakedGlyph` contains two methods: the `GlyphInfo` for the position metadata like before, and a method called `createGlyph`, which returns a `TextRenderable`: a renderer to draw the glyph to the screen. Internally, all resource `BakedGlyph`s are `BakedSheetGlyph`s, as the `UnbakedGlyph$Stitcher#stitch` called just wraps around `GlyphStitcher#stitch`. You can think of the `BakedSheetGlyph` as a view onto a 256x256 `FontTexture`s, where if there is not enough space on one `FontTexture`, then a new `FontTexture` is created.

`BakedSheetGlyph` also implements `EffectGlyph`, which is likely supposed to render some sort of effect on the text. This functionality, while it is currently implemented as just another object, is only used for the white glyph, which goes unused.

- `com.mojang.blaze3d.font`
    - `GlyphInfo`
        - `bake` -> `UnbakedGlyph#bake`, now takes in a `UnbakedGlyph$Stitcher` instead of a function
            - The function behavior is replaced by `UnbakedGlyph$Stitcher#stitch`
        - `$SpaceGlyphInfo` -> `GlyphInfo#simple` and `EmptyGlyph`, not one-to-one
        - `$Stitched` -> `UnbakedGlyph`, not one-to-one
    - `GlyphProvider#getGlyph` now returns an `UnbakedGlyph`
    - `SheetGlyphInfo` -> `GlyphBitmap`
- `net.minecraft.client.gui`
    - `Font` no longer takes in a function and boolean, instead a `Font$Provider`
        - `random` is now private
        - `$GlyphVisitor`
            - `acceptGlyph` now takes in a `TextRenderable` instead of a `BakedGlyph$GlyphInstance`
            - `acceptEffect` now only takes in a `TextRenderable`
        - `$PreparedTextBuilder#accept` now has an override that takes in a `BakedGlyph` instead of its codepoint
        - `$Provider` - A simple interface for providing the glyph source based on its description, along with the empty glyph.
    - `GlyphSource` - An interface that holds the baked glyphs based on its codepoint.
- `net.minecraft.client.gui.font`
    - `AtlasGlyphProvider` - A glyph provider based off a texture atlas.
    - `FontManager` now takes in the `AtlasManager`
    - `FontSet` now takes in a `GlyphStitcher` instead of a `TextureManager` and no longer takes in the name
        - `name` is removed
        - `source` - Returns the glyph source depending given whether only non-fishy glyphs should be seen.
        - `getGlyphInfo`, `getGlyph` -> `getGlyph`, now package-private, not one-to-one
        - `getRandomGlyph` now takes in a `RandomSource` and a codepoint instead of the `GlyphInfo`, returning a `BakedGlyph`
        - `whiteGlyph` now returns an `EffectGlyph`
        - `$GlyphSource` - A source that can get the glyphs to bake.
    - `FontTexture#add` now takes in a `GlyphInfo` and `GlyphBitmap`, returning a `BakedSheetGlyph`
    - `GlyphStitcher` - A class that creates `BakedGlyph`s from its glyph information.
    - `TextRenderable` - Represents how a text representation, like a glyph in a font, is rendered.
- `net.minecraft.client.gui.font.glyphs`
    - `BakedGlyph` is now an interface that creates the renderable object
        - It original purpose has been moved to `BakedSheetGlyph`
    - `EffectGlyph` - An interface that creates the renderable effect of some glyph.
    - `EmptyGlyph` now implements `UnbakedGlyph` instead of extending `BakedGlyph`
        - The constructor takes in the text advance of the glyph.
    - `SpecialGlyphs#bake` now returns a `BakedSheetGlyph`
        - This method is technically new.
- `net.minecraft.client.gui.font.providers`
    - `BitmapProvider$Glyph` now implements `UnbakedGlyph` instead of `GlyphInfo$Stitched`
    - `UnihexProvider$Glyph` now implements `UnbakedGlyph` instead of `GlyphInfo$Stitched`
- `net.minecraft.client.gui.render.state`
    - `GlyphEffectRenderState` is removed
        - Use `GlyphRenderState`
    - `GlyphRenderState` now takes in a `TextRenderable` instead of a `BakedGlyph$Instance`
- `net.minecraft.network.chat`
    - `FontDescription` - An identifier that describes a font, typically as a location or sprite.
    - `Style` is now final
        - `getFont` now returns a `FontDescription`
        - `withFont` now takes in a `FontDescription` instead of a `ResourceLocation`
        - `DEFAULT_FONT` is removed

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
        - `getTextureAtlas` -> `AtlasManager#getAtlasOrThrow`, not one-to-one
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
    - `AtlasIds` -> `net.minecraft.data.AtlasIds`
    - `AtlasSet` -> `AtlasManager`, not one-to-one
        - `forEach` - Iterates through each of the atlas sheets.
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

Minecraft has further abstracted the holder of an item in item models to its own interface: `ItemOwner`. An `ItemOwner` is defined by three things: the `level` the owner is in, the `position` of the owner, and the Y rotation (in degrees) of the owner to indicate the facing direction. `ItemOwner`s are implemented on every `Entity`; however, their position can be offset by using `ItemOwner#offsetFromOwner` or creating a `ItemOwner$OffsetFromOwner`.

Currently, only the new shelf block uses the offset owner so that compasses are not randomly spinning when placed.

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
    - `ItemOwner` - Represents the owner of the item being rendered.

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
- `net.minecraft.network.codec.ByteBufCodecs#PLAYER_NAME` - A 16 byte string stream codec of the player name.
- `net.minecraft.server`
    - `MinecraftServer`
        - `getProfilePermissions` now takes in a `NameAndId` instead of a `GameProfile`
        - `isSingleplayerOwner` now takes in a `NameAndId` instead of a `GameProfile`
    - `Services` now takes in a `UserNameToIdResolver` instead of a `GameProfileCache`, and a `ProfileResolver`
- `net.minecraft.server.players`
    - `GameProfileCache` -> `UserNameToIdResolver`, `CachedUserNameToIdResolver`; not one-to-one
        - `getAsync` is removed
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

### The 'On Shelf' Transform

Models now have a new transform for determining how an item sits on a shelf called `on_shelf`.

- `net.minecraft.client.renderer.block.model.ItemTransforms#fixedFromBottom` - A transform that expects the item to be sitting on some fixed bottom, like a shelf.
- `net.minecraft.world.item.ItemDisplayContext#ON_SHELF` - When an item is placed on a shelf.

### New Tags

- `minecraft:block`
    - `wooden_shelves`
    - `copper_chests`
    - `lightning_rods`
    - `copper`
    - `copper_golem_statues`
    - `incorrect_for_copper_tool`
    - `chains`
    - `lanterns`
    - `bars`
- `minecraft:entity_types`
    - `cannot_be_pushed_onto_boats`
    - `accepts_iron_golem_gift`
    - `candidate_for_iron_golem_gift`
- `minecraft:item`
    - `wooden_shelves`
    - `copper_chests`
    - `lightning_rods`
    - `copper`
    - `copper_golem_statues`
    - `copper_tool_materials`
    - `repairs_copper_armor`
    - `chains`
    - `lanterns`
    - `bars`
    - `shearable_from_copper_golem`

### List of Additions

- `net.minecraft`
    - `SharedConstants`
        - `RESOURCE_PACK_FORMAT_MINOR`, `DATA_PACK_FORMAT_MINOR` - The minor component of the pack version.
        - `DEBUG_SHUFFLE_MODELS` - A flag that likely shuffles the model loading order.
    - `Util#createGameProfile` - Creates a game profile from the UUID, name, and property map.
- `net.minecraft.client`
    - `KeyMapping`
        - `shouldSetOnIngameFocus` - Returns whether the key is mapped to some value on the keyboard.
        - `restoreToggleStatesOnScreenClosed` - Restores the toggled keys to its state during in-game actions.
    - `Minecraft`
        - `isOfflineDevelopedMode` - Returns whether the game is in offline developer mode.
        - `canSwitchGameMode` - Whether the player entity exists and has a game mode.
        - `playerSkinRenderCache` - Returns the cache holding the player's render texture.
        - `canInterruptScreen` - Whether the current screen can be closed by another screen.
    - `Options`
        - `invertMouseX` - When true, inverts the X mouse movement.
        - `toggleAttack` - When true, sets the attack key to be toggleable.
        - `toggleUse` - When true, sets the use key to be toggleable.
        - `sprintWindow` - Time window in ticks where double-tapping the forward key activates sprint.
        - `saveChatDrafts` - Returns whether the message being typed in chat should be retained if the box is closed.
    - `ToggleKeyMapping#shouldRestoreStateOnScreenClosed` - Whether the previous toggle state should be restored after screen close.
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
    - `createCopperLantern` - Creates a lantern and hanging lantern.
    - `createCopperChain` - Creates a chain.
    - `createCopperChainItem` - Creates a chain item model.
- `net.minecraft.client.data.models.model`
    - `ModelTemplate`
        - `SHELF_*` - Model templates for constructing a shelf into a multipart.
        - `LIGHTNING_ROD` - Model template for a lightning rod.
        - `CHAIN` - Model template for a chain.
        - `BARS_*` - Model templates for constructing a bars-like block.
    - `TexturedModel#CHAIN` - A model provider for a chain.
    - `TextureMapping#bars` - A texture mapping for a bars-like block.
    - `TextureSlot#BARS` - A reference to the `#bars` texture.
- `net.minecraft.client.gui`
    - `Gui#renderDeferredSubtitles` - Renders the subtitles on screen.
    - `GuiGraphics#getSprite` - Gets a `TextureAtlasSprite` from its material.
- `net.minecraft.client.gui.components`
    - `AbstractSelectionList`
        - `sort` - Sorts the entries within the list.
        - `swap` - Swaps the position of two entries in the list
        - `clearEntriesExcept` - Removes all entries that don't match the provided element.
        - `getNextY` - Returns the Y component to render the scrolled component.
        - `entriesCanBeSelected` - Whether the entries in the list can be selected.
        - `scrollToEntry` - Scrolls the bar such that the selected entry is on screen.
        - `removeEntries` - Removes all entries that are in the provided list.
        - `$Entry`
            - `set*` - Sets the component size.
            - `getContent*` - Get the content size and positions.
    - `ChatComponent`
        - `saveAsDraft`, `discardDraft` - Handles the draft message in the chat box.
        - `createScreen`, `openScreen`, `preserveCurrentChatScreen`, `restoreChatScreen` - Handles screen behavior depending on the draft chat option.
        - `$ChatMethod` - Defines what type of message is the chat.
        - `$Draft` - Defines a draft message in the chat box.
    - `EditBox$TextFormatter` - An interface used to format the string in the box.
    - `FittingMultiLineTextWidget#minimizeHeight` - Sets the height of the widget to the inner height and padding.
    - `FocusableTextWidget$BackgroundFill` - An enum that represents when the background should be filled.
    - `ItemDisplayWidget#renderTooltip` - Submits the tooltip to render.
    - `MultiLineLabel$Align` - Handles the alignment of the label.
    - `MultilineTextField#selectWordAtCursor` - Selects the word whether the cursor is currently located.
    - `SpriteIconButton$Builder#withTooltip` - Sets the tooltip to display the message.
    - `StringWidget`
        - `setMaxWidth` - Sets the max width of the string and how to handle the extra width.
        - `$TextOverflow` - What to do if the text is longer than the max width.
- `net.minecraft.client.gui.components.events.GuiEventListener#shouldTakeFocusAfterInteraction` - When true, sets the element as focused when clicked.
- `net.minecraft.client.gui.navigation.CommonInputs#confirm` - Returns whether the keycode is the enter button on the main keyboard or numpad.
- `net.minecraft.client.gui.screens`
    - `ChatScreen`
        - `isDraft` - Whether there is a message that hasn't been sent in the box.
        - `exitReason` - The reason the chat screen was closed.
        - `shouldDiscardDraft` - Whether the current draft message should be discarded.
        - `$ChatConstructor` - A constructor for the chat screen based on the draft state.
        - `$ExitReason` - The reason the chat screen was closed.
    - `FaviconTexture#isClosed` - Returns whether the texture has been closed.
    - `LevelLoadingScreen`
        - `update` - Updates the current load tracker and reason.
        - `$Reason` - The reason for changing the level loading screen.
    - `Screen`
        - `panoramaShouldSpin` - Returns whether the rendered panorama should spin its camera.
        - `setNarrationSuppressTime` - Sets until when the narration should be suppressed for.
        - `isInGameUi` - Returns whether the screen is opened while playing the game, not the pause menu.
        - `isAllowedInPortal` - Whether this screen can be open during the portal transition effect.
        - `canInterruptWithAnotherScreen` - Whether this screen can be closed by another screen, defaults to 'close on escape' screens.
- `net.minecraft.client.gui.screens.inventory.AbstractCommandBlockScreen#addExtraControls` - Adds any extra widgets to the command block screen.
- `net.minecraft.client.gui.screens.multiplayer`
    - `CodeOfConductScreen` - A screens that displays the code of conduct text for a server.
    - `ServerSelectionList$Entry#matches` - Returns whether the provided entry matches this one.
- `net.minecraft.client.gui.screens.reporting.ChatSelectionScreen$ChatSelectionList#ITEM_HEIGHT` - The height of each entry in the list.
- `net.minecraft.client.gui.screens.worldselection.WorldSelectionList`
    - `returnToScreen` - Returns to the previous screen after reloading the world list.
    - `$Builder` - Creates a new `WorldSelectionList`.
    - `$Entry#getLevelSummary` - Returns the summary of the selected world.
    - `$EntryType` - A enum representing what type of world the entry is.
    - `$NoWorldsEntry` - An entry where there are no worlds to select from.
- `net.minecraft.client.main.GameConfig$GameData#offlineDeveloperMode` - Whether the game is offline and is ran for development.
- `net.minecraft.client.multiplayer`
    - `ClientCommonPacketListenerImpl`
        - `seenPlayers` - All players that have been seen, regardless of if they are currently online, by this player.
        - `seenInsecureChatWarning` - If the player has seen the insecure chat warning.
        - `$CommonDialogAccess` - A dialog access used by the client network.
    - `ClientExplosionTracker` - Tracks the explosions made on the client, specifically to handle particles.
    - `ClientLevel`
        - `endFlashState` - Handles the state of the flashes of light that appear in the end.
        - `trackExplosionEffects` - Tracks an explosion to handle its particles.
    - `ClientPacketListener`
        - `getSeenPlayers` - Returns all players seen by this player.
        - `getPlayerInfoIgnoreCase` - Gets the player info for the provided profile name.
- `net.minecraft.client.player.LocalPlayerResolver` - A profile resolver for the local player.
- `net.minecraft.client.renderer`
    - `EndFlashState` - The render state of the end flashes.
    - `SkyRenderer#renderEndFlash` - Renders the end flashes.
- `net.minecraft.client.resources.WaypointStyle#ICON_LOCATION_PREFIX` - The prefix for waypoint icons.
- `net.minecraft.client.resources.sounds.DirectionalSoundInstance` - A sound that changes position based on the direction of the camera.
- `net.minecraft.core`
    - `BlockPos#betweenCornersInDirection` - An iterable that iterates through the provided bounds in the direction provided by the vector.
    - `Direction#axisStepOrder` - Returns a list of directions that the given vector should be checked in.
- `net.minecraft.core.particles.ExplosionParticleInfo` - The particle that used as part of an explosion.
- `net.minecraft.data.loot.packs`
    - `VanillaBlockInteractLoot` - A sub provider for vanilla block loot from entity interaction.
    - `VanillaChargedCreeperExplosionLoot` - A sub provider for vanilla entity loot from charged creeper explosions.
    - `VanillaEntityInteractLoot` - A sub provider for vanilla entity loot from entity interaction.
- `net.minecraft.data.recipes.RecipeProvider#shelf` - Constructs a shelf recipe from one item.
- `net.minecraft.gametest.framework.GameTestHelper#getTestDirection` - Gets the direction that the test is facing.
- `net.minecraft.network`
    - `FriendlyByteBuf#readLpVec3`, `writeLpVec3` - Handles syncing a compressed `Vec3`.
    - `LpVec3` - A vec3 network handler that compresses and decompresses a vector into at most two bytes and two integers.
- `net.minecraft.network.chat.CommonComponents#GUI_COPY_TO_CLIPBOARD` - The message to display when copying a chat to the clipboard.
- `net.minecraft.network.chat.contents.ObjectContents` - An arbitrary piece of content that can be written as part of a component, like a sprite.
- `net.minecraft.network.protocol.configuration`
    - `ClientboundCodeOfConductPacket` - A packet the sends the code of conduct of a server to the joining client.
    - `ClientConfigurationPacketListener#handleCodeOfConduct` - Handles the code of conduct sent from the server.
    - `ServerboundAcceptCodeOfConductPacket` - A packet that sends the acceptance response of a client reading the code of conduct from a server.
    - `ServerConfigurationPacketListener#handleAcceptCodeOfConduct` - Handles the acceptance of the code of conduct from the client.
- `net.minecraft.network.syncher.EntityDataSerializers`
    - `WEATHERING_COPPER_STATE` - The weather state of the copper on the golem.
    - `COPPER_GOLEM_STATE` - The logic state of the copper golem.
- `net.minecraft.server.MinecraftServer`
    - `selectLevelLoadFocusPos` - Returns the loading center position of the server, usually the shared spawn position.
    - `getLevelLoadListener` - Returns the listener used to track level loading.
    - `getCodeOfConducts` - Returns a map of file names to their code of conduct text.
- `net.minecraft.server.commands.FetchProfileCommand` - Fetches the profile of a given user.
- `net.minecraft.server.dedicated.DedicatedServerProperties#codeOfConduct` - Whether the server has a code of conduct.
- `net.minecraft.server.level`
    - `ChunkLoadCounter` - Keeps track of chunk loading when a level is loading or the player spawns.
    - `ChunkMap#getLatestStatus` - Returns the latest status for the given chunk position.
    - `ServerChunkCache`
        - `hasActiveTickets` - Checks whether the current level has any active tickets keeping it loaded.
        - `addTicketAndLoadWithRadius` - Adds a ticket to some location and loads the chunk and radius around that location.
    - `ServerPlayer$SavedPosition` - Holds the current position of the player on disk.
- `net.minecraft.server.level.progress.ChunkLoadStatusView` - A status view for the loading chunks.
- `net.minecraft.server.network.ConfigurationTask#tick` - Calls the task every tick until it returns true, then finishes the task.
- `net.minecraft.server.network.config`
    - `PrepareSpawnTask` - A configuration task that finds the spawn of the player.
    - `ServerCodeOfConductConfigurationTask` - A configuration task that sends the code of conduct to the client.
- `net.minecraft.server.packs.OverlayMetadataSection`
    - `codecForPackType` - Constructs a codec for the section given the pack type.
    - `forPackType` - Gets the metadata section type from the pack type.
- `net.minecraft.server.packs.metadata.MetadataSectionType#withValue`, `$WithValue` - Holds a metadata section along with its provided value.
- `net.minecraft.server.packs.metadata.pack`
    - `PackFormat` - Represents the major and minor revision of a pack.
    - `PackMetadataSection#forPackType` - Gets the metadata section type from the pack type.
- `net.minecraft.server.packs.repository.PackCompatibility#UNKNOWN` - The compatibility of the pack with the game is unknown as the major version is set to the max integer value.
- `net.minecraft.server.packs.resources.ResourceMetadata#getTypedSection`, `getTypedSections` - Gets the metadata sections for the provided types.
- `net.minecraft.server.players.ProfileResolver` - Resolves a game profile from its name or UUID.
- `net.minecraft.util`
    - `ExtraCodecs#gameProfileCodec` - Creates a codec for a `GameProfile` given the UUID codec.
    - `InclusiveRange#map` - Maps a range from one generic to another.
    - `Mth#ceilLong` - Returns the double rounded up to the nearest long.
- `net.minecraft.util.random`
    - `Weighted#streamCodec` - Constructs a stream codec for the weighted entry.
    - `WeightedList#streamCodec` - Constructs a stream codec for the weighted list.
- `net.minecraft.util.thread.BlockableEventLoop#shouldRunAllTasks` - Returns if there are any blocked tasks.
- `net.minecraft.world.entity`
    - `EntityReference`
        - `getEntity` - Returns the entity given the type class and the level.
        - Static `getEntity`, `getLivingEntity`, `getPlayer` - Returns the entity given its `EntityReference` and level.
    - `EntityType`
        - `STREAM_CODEC` - The stream codec for an entity type.
        - `$Builder#notInPeaceful` - Sets the entity to not spawn in peaceful mode.
    - `Entity#canInteractWithLevel` - Whether the entity can interact with the current level.
    - `InsideBlockEffectType#CLEAR_FREEZE` - A in-block effect that clears the frozen ticks.
    - `LivingEntity#dropFromEntityInteractLootTable` - Drops loot from a table from an entity interaction.
    - `PositionMoveRotation#withRotation` - Creates a new object with the provided XY rotation.
    - `Relative`
        - `rotation` - Gets the set of relative rotations from the XY booleans.
        - `position` - Gets the set of relative positions from the XYZ booleans.
        - `direction` - Gets the set of relative deltas from the XYZ booleans.
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
- `net.minecraft.world.entity.vehicle.MinecartFurnace#addFuel` - Adds fuel to the furnace to push the entity.
- `net.minecraft.world.item`
    - `Item$TooltipContext#isPeaceful` - Returns true if the difficulty is set to peaceful.
    - `ToolMaterial#COPPER` - The copper tool material.
    - `WeatheringCopperItems` - A record of items that represent each of the weathering copper states.
- `net.minecraft.world.item.equipment`
    - `ArmorMaterials#COPPER` - The copper armor material.
    - `EquipmentAssets#COPPER` - The key reference to the copper equipment asset.
    - `ResolvableProfile`
        - `$Static` - Uses the already resolved game profile.
        - `$Dynamic` - Dynamically resolves the game profile on use.
        - `$Partial` - Represents part of the game profile depending on whatever information is provided to the component.
- `net.minecraft.world.level.Level`
    - `getEntityInAnyDimension` - Gets the entity by UUID.
    - `getPlayerInAnyDimension` - Gets the player by UUID.
- `net.minecraft.world.level.block`
    - `Block#dropFromBlockInteractLootTable` - Drops the loot when interacting with a block.
    - `ChestBlock`
        - `chestCanConnectTo` - Returns whether the chest can merge with another block.
        - `getConnectedBlockPos` - Gets the connected block position of the chest.
        - `getOpenChestSound`, `getCloseChestSound` - Returns the sounds played on chest open / close.
    - `ChiseledBookShelfBlock#FACING`, `SLOT_*_OCCUPIED` - The block state properties of the chiseled bookshelf.
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
    - `WeatheringCopperBarsBlock` - The block for weathering copper bars.
    - `WeatheringCopperBlocks` - A record of blocks that represent each of the weathering copper states.
    - `WeatheringCopperChainBlock` - The block for weathering copper chains.
    - `WeatheringCopperChestBlock` - The block for the weathering copper chest.
    - `WeatheringCopperGolemStatueBlock` - The block for the weathering copper golem statue.
    - `WeatheringLanternBlock` - The block for weathering lanterns.
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
- `net.minecraft.world.phys.Vec3#X_AXIS`, `Y_AXIS`, `Z_AXIS` - The unit vector in the positive direction of each axis.
- `net.minecraft.world.phys.shapes.CollisionContext#alwaysCollideWithFluid` - Whether the collision detector should always collide with a fluid in its path.
- `net.minecraft.world.waypoints.PartialTickSupplier` - Gets the partial tick given the provided entity.

### List of Changes

- `com.mojang.blaze3d.buffers.GpuBuffer#size` is now private
- `com.mojang.blaze3d.opengl`
    - `DirectStateAccess#bufferSubData`, `mapBufferRange`, `unmapBuffer`, `flushMappedBufferRange` now take in the buffer usage bit mask
    - `GlStateManager#_texImage2D`, `_texSubImage2D` now takes in a `ByteBuffer` instead of an `IntBuffer`
    - `VertexArrayCache#bindVertexArray` can now take in a nullable `GlBuffer`
- `com.mojang.blaze3d.systems.CommandEncoder#writeToTexture` now takes in a `ByteBuffer` instead of an `IntBuffer`
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
    - `Minecraft`
        - `setLevel` no longer takes in the `ReceivingLevelScreen$Reason`
        - `cameraEntity` is now private
        - `openChatScreen` is now public, taking in the `ChatComponent$ChatMethod`
        - `setCamerEntity` can now take in a null entity
        - `forceSetScreen` -> `setScreenAndShow`
        - `getMinecraftSessionService` -> `services`, now returning a `Service` instance
            - The `MinecraftSessionService` can be obtained via `Service#sessionService`
    - `KeyMapping`
        - `key` is now protected
        - `release` is now protected
    - `Options#invertYMouse` -> `invertMouseY`
    - `ToggleKeyMapping` now has an overload that takes in an input type
    - `User` no longer takes in the user type
- `net.minecraft.client.animation.Keyframe` now has an overload that takes in the `preTarget` and `postTarget` instead of one simple `target`
- `net.minecraft.client.data.models.BlockModelGenerators`
    - `condition` now has an overload that takes in an enum or boolean property
    - `createLightningRod` now takes in the regular and waxed blocks
    - `createIronBars` -> `createBarsAndItem`, `createBars`; not one-to-one
- `net.minecraft.client.gui.GuiGraphics#submitSignRenderState` now takes in a `Model$Simple` instead of a `Model`
- `net.minecraft.client.gui.components`
    - `AbstractSelectionList` no longer takes in the header height
        - `itemHeight` -> `defaultEntryHeight`
        - `addEntry`, `addEntryToTop` now takes in the height of the element
        - `removeEntryFromTop` no longer returns anything
        - `updateSizeAndPosition` can now take in an X component
        - `renderItem` now takes in the entry instead of five integers
        - `renderSelection` now takes in an `AbstractSelectionList$Entry` instead of four integers
        - `removeEntry` no longer returns anything
        - `$Entry` now implements `LayoutElement`
        - `render`, `renderBack` -> `renderContent`
    - `ContainerObjectSelectionList` no longer takes in the headerh height
    - `EditBox#setFormatter` -> `addFormatter`, not one-to-one
    - `FocusableTextWidget` now takes in a `$BackgroundFill` instead of a boolean
    - `MultiLineLabel`
        - `renderCentered`, `renderLeftAligned`, `renderLeftAlignedNoShadow` -> `render`, not one-to-one
        - `getStyleAtCentered`, `getStyleAtLeftAligned` -> `getStyle`, not one-to-one
    - `ObjectSelectionList` no longer takes in the headerh height
    - `SpriteIconButton` now takes in a `WidgetSprites` instead of a `ResourceLocation`, and toolip `Component`
        - `sprite` is now a `WidgetSprites`
        - `$Builder#sprite` now has an overload that takes in a `WidgetSprites`
        - `$CenteredIcon` now takes in a `WidgetSprites` instead of a `ResourceLocation`, and toolip `Component`
        - `$TextAndIcon` now takes in a `WidgetSprites` instead of a `ResourceLocation`, and toolip `Component`
    - `WidgetSprites` now has an overload with a single `ResourceLocation`
- `net.minecraft.client.gui.render.state`
    - `GuiElementRenderState#buildVertices` no longer takes in the `z` component
    - `GuiRenderState#forEachElement` now takes in a `Consumer<GuiElementRenderState>` instead of a `GuiRenderState$LayeredElementConsumer`
- `net.minecraft.client.gui.render.state.pip.GuiSignRenderState` now takes in a `Model$Simple` instead of a `Model`
- `net.minecraft.client.gui.screens`
    - `ChatScreen` now takes in a boolean representing whether the message is a draft
        - `initial` is now protected
    - `InBedChatScreen` now takes in the initial text and whether there is a draft mesage in the box
    - `LevelLoadingScreen` now takes in a `LevelLoadTracker` and `LevelLoadingScreen$Reason`
    - `PauseScreen#disconnectFromWorld` -> `Minecraft#disconnectFromWorld`
    - `ReceivingLevelScreen` has been merged into `LevelLoadingScreen`
    - `Screen`
        - `renderWithTooltip` -> `renderWithTooltipAndSubtitles`
        - `$NarratableSearchResult` is now a record
- `net.minecraft.client.gui.screens.achievement.StatsScreen`
    - `initLists`, `initButtons` merged into `onStatsUpdated`
    - `$ItemRow#getItem` is now protected
- `net.minecraft.client.gui.screens.inventory.tooltip.ClientActivePlayersTooltip$ActivePlayersTooltip#profiles` now is a list of `PlayerSkinRenderCache$RenderInfo`s instead of `ProfileResult`s
- `net.minecraft.client.gui.screens.multiplayer.JoinMultiplayerScreen#*_BUTTON_WIDTH` are now private
- `net.minecraft.client.gui.screens.options.OptionsScreen#CONTROLS` is now public
- `net.minecraft.client.gui.screens.packs`
    - `PackSelectionModel` now takes in a `Consumer<PackSelectionModel$EntryBase>` instead of a `Runnable`
        - `$EntryBase` is now public
    - `TransferableSelectionList` now is a list of `$Entry`s instead of `$PackEntry`s
        - `$PackEntry` is no longer static and extends `$Entry`
- `net.minecraft.client.gui.screens.worldselection`
    - `CreateWorldScreen#openFresh`, `testWorld`, `createFromExisting` now take in a `Runnable` instead of a `Screen`
    - `WorldSelectionList` now has a package-private constructor
        - `getScreen` now returns a basic `Screen`
        - `$WorldListEntry` is now static
            - `canJoin` -> `canInteract`, not one-to-one
- `net.minecraft.client.gui.spectator.PlayerMenuItem` now only takes in the `PlayerInfo`
- `net.minecraft.client.main.GameConfig$UserData` no longer takes in the `PropertyMap`s
- `net.minecraft.client.multiplayer`
    - `ClientHandshakePacketListenerImpl` now takes in a `LevelLoadTracker`
    - `CommonListenerCookie` now takes in a `LevelLoadTracker`, map of seen players, and whether the insecure chat warning has been shown
    - `LevelLoadStatusManager` -> `LevelLoadTracker`, not one-to-one
        - `CLOSE_DELAY_MS` -> `LEVEL_LOAD_CLOSE_DELAY_MS`, now public
    - `TransferState` now takes in a map of seen players and whether the insecure chat warning has been shown
- `net.minecraft.client.multiplayer.chat.ChatListener#clearQueue` -> `flushQueue`
- `net.minecraft.client.renderer.DimensionSpecialEffects#forceBrightLightmap` -> `hasEndFlashes`, not one-to-one
- `net.minecraft.client.resources`
    - `SkinManager` now takes in `Services` instead of a `MinecraftSessionService`
        - `lookupInsecure` -> `createLookup`, now taking in a boolean of whether to check for insecure skins
        - `getOrLoad` -> `get`
    - `WaypointStyle#validate` is now public
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
- `net.minecraft.network.VarInt#MAX_VARINT_SIZE` is now public
- `net.minecraft.network.protocol.game`
    - `ClientboundAddEntityPacket#getXa`, `getYa`, `getZa` -> `getMovement`
    - `ClientboundExplodePacket` now takes in a radius, block count, and the block particles to display
    - `ClientboundPlayerRotationPacket` now takes in whether the XY rotation is relative
    - `ClientboundSetEntityMotionPacket#getXa`, `getYa`, `getZa` -> `getMovement`
- `net.minecraft.server`
    - `SPAWN_POSITION_SEARCH_RADIUS` is now public
    - `MinecraftServer` now takes in a `LevelLoadListener` instead of a `ChunkProgressListenerFactory`
        - `createLevels` no longer takes in a `ChunkProgressListener`
        - `getSessionService`, `getProfileKeySignatureValidator`, `getProfileRepository`, `getProfileCache` -> `services`, not one-to-one
            - `getProfileCache` is now `nameToIdCache`, not one-to-one
- `net.minecraft.server.dedicated.DedicatedServer` now takes in a `LevelLoadListener` instead of a `ChunkProgressListenerFactory`
- `net.minecraft.server.level`
    - `ChunkMap` no longer takes in the `ChunkProgressListener`
        - `getTickingGenerated` -> `allChunksWithAtLeastStatus`, not one-to-one
    - `PlayerRespawnLogic` -> `PlayerSpawnFinder`, not one-to-one
    - `ServerChunkCache` no longer takes in the `ChunkProgressListener`
    - `ServerEntityGetter#getNearestEntity` now has an overload that takes in a `TagKey` of entities instead of a class
    - `ServerLevel` no longer takes in the `ChunkProgressListener`
        - `waitForChunkAndEntities` -> `waitForEntities`, not one-to-one
    - `ServerPlayerGameMode#setGameModeForPlayer` now takes in a `TriState` of how to update the flying of the player when switching to creative mode.
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
- `net.minecraft.util.ExtraCodecs`
    - `GAME_PROFILE_WITHOUT_PROPERTIES` -> `AUTHLIB_GAME_PROFILE`, now a `Codec` and public
    - `GAME_PROFILE` -> `STORED_GAME_PROFILE`
- `net.minecraft.world.entity`
    - `AgeableMob#finalizeSpawn` is now nullable
    - `Entity`
        - `startRiding(Entity)` is now final
        - `startRiding(Entity, boolean)` now takes in an additional boolean of whether to trigger the game event and criteria triggers
        - `killedEntity` now takes in the `DamageSource`
        - `moveOrInterpolateTo` now has overloads that take in an XY rotation, a `Vec3` position, or all three as optionals
        - `lerpMotion` now takes in a `Vec3` instead of three doubles
        - `forceSetRotation` now takes in whether the XY rotation is relative
    - `EntityReference` is now private, constructed using the `of` static constructors
        - `getEntity` now takes in a `UUIDLookup<? extends UniquelyIdentifyable>` instead of a `UUIDLookup<? super StoredEntityType>`
        - `get` now takes in a `Level` instead of a `UUIDLookup`
    - `EntityType` now takes in whether the entity is allowed in peaceful mode
        - `create`, `loadEntityRecursive` now has an overload that takes in an `EntityType`
    - `LivingEntity`
        - `shouldDropLoot` now takes in the `ServerLevel`
        - `dropFromLootTable` now has overloads to take in a specific loot table key and how the items should be dispensed
        - `getSlotForHand` -> `InteractionHand#asEquipmentSlot`
    - `Mob#shouldDespawnInPeaceful` -> `EntityType#isAllowedInPeaceful`, not one-to-one
- `net.minecraft.world.entity.animal.Animal#usePlayerItem` -> `Mob#usePlayerItem`
- `net.minecraft.world.entity.animal.armadillo.Armadillo#brushOffScute` now takes in an `Entity` and `ItemStack`
- `net.minecraft.world.entity.projectile.Projectile`
    - `deflect` now takes in an `EntityReference` of the owner instead of the `Entity` itself
    - `onDeflection` no longer takes in the direct `Entity`
- `net.minecraft.world.entity.vehicle.MinecartBehavior#lerpMotion` now takes in a `Vec3` instead of three doubles
- `net.minecraft.world.item.SpawnEggItem` no longer takes in the `EntityType`
    - `spawnsEntity` no longer takes in the `HolderLookup$Provider`
    - `getType` no longer takes in the `HolderLookup$Provider`
- `net.minecraft.world.item.component`
    - `Bees#STREAM_CODEC` now requires a `RegistryFriendlyByteBuf`
    - `ResolvableProfile` is now a sealed class with a protected constructor, created through the static `createResolved`, `createUnresolved`
        - `pollResolve`, `resolve`, `isResolved` -> `resolveProfile`, not one-to-one
- `net.minecraft.world.item.enchantment.effects.ExplodeEffect` now takes in a list of block particles to display
- `net.minecraft.world.level`
    - `GameType#updatePlayerAbilities` now takes in a `TriState` to determine whether the player is flying in creative
    - `Level` no longer implements `UUIDLookup`
        - `explode` now takes in a weighter list of explosion particles to display
    - `ServerExplosion#explode` now returns the number of blocks exploded
- `net.minecraft.world.level.block`
    - `BeehiveBlock#dropHoneycomb(Level, BlockPos)` -> `dropHoneyComb(ServerLevel, ItemStack, BlockState, BlockEntity, Entity, BlockPos)`
    - `CaveVines#use` entity is no longer nullable
    - `ChestBlock` now takes in an open and close sound
    - `ChiseledBookShelfBlock` now implements `SelectableSlotContainer`
        - `BOOKS_PER_ROW` is now private
- `net.minecraft.world.level.block.entity`
    - `ChiseledBookShelfBlockEntity` now implements `ListBackedContainer`
        - `count` -> `ListBackedContainer#count`
    - `ContainerOpenersCounter`
        - `isOwnContainer` is now public
        - `incrementOpeners`, `decrementOpeners` now takes in a `LivingEntity` instead of a `Player`
        - `getPlayersWithContainerOpen` -> `getEntitiesWithContainerOpen`, now public, not one-to-one
- `net.minecraft.world.level.block.state.BlockBehaviour`
    - `getAnalogOutputSignal`, `$BlockStateBase#getAnalogOutputSignal` now takes in the direction the signal is coming from
    - `$Properties#noCollission` -> `noCollision`
- `net.minecraft.world.level.block.state.properties.BlockStateProperties#CHISELED_BOOKSHELF_SLOT_*_OCCUPIED` -> `SLOT_*_OCCUPIED`
- `net.minecraft.world.level.entity.UUIDLookup#getEntity` -> `lookup`
- `net.minecraft.world.level.levelgen`
    - `Beardifier` now takes in lists instead of iterators and a nullable `BoundingBox`
    - `NoiseRouter#initialDensityWithoutJaggedness` -> `preliminarySurfaceLevel`
- `net.minecraft.world.level.levelgen.structure.pools.JigsawPlacement#addPieces` now takes in a `JigsawStructure$MaxDistance` instead of an integer
- `net.minecraft.world.level.levelgen.structure.structures.JigsawStructure` now takes in a `JigsawStructure$MaxDistance` instead of an integer
- `net.minecraft.world.phys.shapes.EntityCollisionContext` now takes in a `boolean` instead of a `Predicate<FluidState>`
- `net.minecraft.world.waypoints.TrackedWaypoint`
    - `STREAM_CODEC` is now final
    - `yawAngleToCamera` now takes in a `PartialTickSupplier`
    - `pitchDirectionToCamera` now takes in a `PartialTickSupplier`

### List of Removals

- `com.mojang.blaze3d.audio.Listener#setGain`, `getGain`
- `com.mojang.blaze3d.opengl`
    - `GlShaderModule#compile`
    - `GlStateManager`
        - `_glUniform1`, `_glUniform2`, `_glUniform3`, `_glUniform4`
        - `_glUniformMatrix4`
        - `glActiveTexture`, `_getActiveTexture`
- `com.mojang.blaze3d.pipeline.RenderTarget#viewWidth`, `viewHeight`
- `com.mojang.blaze3d.systems.RenderSystem`
    - `getQuadVertexBuffer`
    - `setModelOffset`, `resetModelOffset`, `getModelOffset`
- `com.mojang.blaze3d.vertex`
    - `DefaultVertexFormat#BLIT_SCREEN`
    - `VertexConsumer#setWhiteAlpha`
- `net.minecraft.SharedConstants#VERSION_STRING`
- `net.minecraft.client`
    - `Camera#FOG_DISTANCE_SCALE`
    - `Minecraft`
        - `getProgressListener`
        - `getProfileKeySignatureValidator`, `canValidateProfileKeys`
    - `Options#RENDER_DISTANCE_TINY`, `RENDER_DISTANCE_NORMAL`
    - `User#getType`, `$Type`
- `net.minecraft.client.gui.components`
    - `AbstractSelectionList`
        - `headerHeight`, associated constructor has been removed
        - `setSelectedIndex`, `getFirstElement`, `getEntry`
        - `isSelectedItem`
        - `renderHeader`, `renderDecorations`
        - `ensureVisible`
        - `remove`
    - `OptionsList#getMouseOver`
    - `StringWidget#alignLeft`, `alignCenter`, alignRight`
- `net.minecraft.client.gui.render.state.GuiRenderState`
    - `down`, `$Node#down`
    - `$LayeredElementConsumer`
- `net.minecraft.client.gui.screens.LevelLoadingScreen#renderChunks`
- `net.minecraft.client.gui.screens.achievement.StatsScreen#setActiveList`
- `net.minecraft.client.gui.screens.dialog.DialogConnectionAccess#disconnect`
- `net.minecraft.client.gui.screens.multiplayer.JoinMultiplayerScreen`
    - `BUTTON_ROW_WIDTH`, `FOOTER_HEIGHT`
    - `setSelected`
- `net.minecraft.client.main.GameConfig$UserData#userProperties`, `profileProperties`
- `net.minecraft.client.renderer.chunk.ChunkSectionLayer#outputTarget`
- `net.minecraft.client.resources.SkinManager#getInsecureSkin`
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
- `net.minecraft.world.entity.vehicle.MinecartTNT#explode(double)`
- `net.minecraft.world.item.Item#verifyComponentsAfterLoad`
- `net.minecraft.world.level`
    - `BlockGetter#MAX_BLOCK_ITERATIONS_ALONG_TRAVEL`
    - `GameRules#RULE_SPAWN_CHUNK_RADIUS`
- `net.minecraft.world.level.block.FletchingTableBlock`
- `net.minecraft.world.level.block.entity.SkullBlockEntity`
    - `CHECKED_MAIN_THREAD_EXECUTOR`
    - `setup`, `clear`
    - `fetchGameProfile`
    - `setOwner`
