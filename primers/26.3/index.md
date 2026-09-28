# Minecraft 26.2 -> 26.3 Mod Migration Primer

This is a high level, non-exhaustive overview on how to migrate your mod from 26.2 to 26.3. This does not look at any specific mod loader, just the changes to the vanilla classes. All provided names use the official mojang mappings.

This primer is licensed under the [Creative Commons Attribution 4.0 International](http://creativecommons.org/licenses/by/4.0/), so feel free to use it as a reference and leave a link so that other readers can consume the primer.

If there's any incorrect or missing information, please file an issue on this repository or ping @ChampionAsh5357 in the Neoforged Discord server.

Thank you to:

- @RogueLogix for reviews on the Blaze3d and Renderpearl changes
- @crendgrim since the E key is not the M key

## Pack Changes

There are a number of user-facing changes that are part of vanilla which are not discussed below that may be relevant to modders. You can find a list of them on [Misode's version changelog](https://misode.github.io/versions/?id=26.3&tab=changelog).

## There's Always a Client Rewrite

### LWJGL with SDL

The Lightweight Java Game Library (LWJGL) now ships with Simple DirectMedia Layer (SDL) instead of the Graphics Library Framework (GLFW). As such, most of the backend logic that called out to GLFW has either been completely removed or refashioned to use SDL method calls.

The most common changes visible to users involve handling input from the keyboard and mouse.

The associated input codes now have different values. To remap these scancodes, you should either use the constants defined in `InputConstants` or `SDLScancode`:

```java
new KeyMapping(
    "examplemod.key.example_key",
    // The SDL scancode to capture when pressed.
    InputConstants.KEY_M, // or SDLScancode#SDL_SCANCODE_M
    KeyMapping.Category.MISC
);
```

Note that we are now using the scancode as our 'key' with SDL instead of the logical key from GLFW. These both function equivalently, as GLFW keys are known as scancodes in SDL, while GLFW scancodes refer to SDL keycodes.

For `KeyEvent`, this means that `key` refers to the values from `SDLScancode`, and `keycode` (renamed from `scancode`) refers to the values from `SDLKeycode`. `MouseButtonInfo#button` reference the `SDLMouse#SDL_BUTTON_*` constants. As for the input modifiers (e.g., lctrl, rshift), they are a mask of the `SDLKeycode#SDL_KMOD_*` constants.

The other important change is how text input is handled. Due to the differences in how GLFW and SDL setup the callbacks, SDL requires `SDLKeyboard#SDL_StartTextInput` and `SDL_StopTextInput` to know when the player is typing. Vanilla implements these through `TextInputManager#startTextInput` and `stopTextInput`, taking in the object consuming the input. For use in GUIs, vanilla calls `TextureInputManager#onTextInputFocusChange` through `Minecraft#onTextInputFocusChange` to manage GUI element selection and focusing, while also handling any preedit events.

As such, modders must make sure to call `TextureInputManager#onTextInputFocusChange` whenever the player should start or stop typing. For GUI elements, this should be within `GuiEventListener#setFocused` at a minimum:

```java
// For some basic gui element
public class ExampleWidget extends AbstractWidget {

    // ...

    @Override
    public void setFocused(boolean focused) {
        super.setFocused(focused);

        // Change the input focus.
        // This may need to be called in other places depending on
        // how your text input element is handled.
        // You can also call the following instead if outside a GUI context:
        // Minecraft.getInstance().textInputManager().onTextInputFocusChange(this, focused);
        Minecraft.getInstance().onTextInputFocusChange(this, focused);
    }
}
```

### Renderpearl

The render API that previously lived in Blaze3d has been split into a separate project called Renderpearl. All rendering classes that originally lived in Blaze3d have been migrated to `com.mojang.renderpearl`.

Packages in Renderpearl are roughly structured like so:

- `com.mojang.renderpearl.api.*` - APIs for the elements the vanilla game interacts with.
- `com.mojang.renderpearl.backend.*` - The objects that manage the actual logic to communicate with a graphics library.
- `com.mojang.renderpearl.frontend.*` - The implemented APIs that the vanilla game interacts with. These classes handle most of the validation.

### No More Texture Override Shenanigans

`RenderSystem#outputColorTextureOverride` and `outputDepthTextureOverride` have been fully removed, closing one of the shenanigans then it came to writing to the GPU texture, especially for `FeatureRenderDispatcher`. Now, each try-with-resources creates the `RenderPass` to specify the textures to output to, along with any other information, like the `FeatureRenderDispatcher$PreparedFrame`:

```java
// Setup everything possible before the render frame

// Create the texture to write to
GpuDevice device = RenderSystem.getDevice();
GpuTexture texture = device.createTexture(/*...*/);
GpuTexture textureView = device.createTextureView(texture);
GpuTexture depthTexture = device.createTexture(/*...*/);
GpuTexture depthTextureView = device.createTextureView(depthTexture);
device.createCommandEncoder().clearColorAndDepthTextures(texture, new Vector4f(0f), depthTexture, 0);

// ...

// Write data for rendering
SubmitNodeStorage storage = new SubmitNodeStorage();
storage.submitCustomGeometry(/*...*/);

// ...

// Render to the texture.
// This should only contain those critical to modify the render pass with
// the correct info. Everything else should've been done prior.
try (
    FeatureRenderDispatcher.PreparedFrame frame = featureRenderDispatcher.prepareFrame(storage);
    RenderPass pass = RenderSystem.getDevice()
        .createCommandEncoder()
        .createRenderPass(() -> "Example", textureView, Optional.empty(), depthTextureView, OptionalDouble.empty());
) {
    // Setup anything required by the pass.
    RenderSystem.bindDefaultUniforms(pass);

    //...

    // Render to the textures.
    FeatureRenderDispatcher.renderAllFeatures(pass, frame);
}
```

### Shader Extensions and Layouts

The Shader GLSL now has some additional features and changes regarding the the underlying syntax as Shaderc is now used for compilation on both OpenGL and Vulkan via the `GlslCompiler`.

As such, `#moj_import` was renamed to `#include` to import functions from other shaders. Additionally, both the `in` and `out` uniforms must specify their location in the layout to maintain a fixed read/write location when passing from vertex to fragment. This is especially valuable when some are included based on shader defines.

Speaking of defines, `GlslCompiler` provides three macros: `RENDERPEARL_DEPTH_IS_ZERO_TO_ONE` for if the device coordinates are between 0-1 (always defined for Vulkan, if the machine has the `GL_ARB_clip_control` extension for OpenGL), `RENDERPEARL_EXPLICIT_DEPTH_INVARIANCE` for if the depth must be explicitly set (currently only Apple silicon machines), and `RENDERPEARL_INSTANCE_INDEX_INCLUDES_BASE_INSTANCE` for if the `gl_InstanceIndex` parameter either represents the instance index or `gl_InstanceID` (always defined for Vulkan, if the machine has the `GL_ARB_shader_draw_parameters` extension for OpenGL).

Finally, most vanilla shaders require the machine to have the `GL_ARB_separate_shader_objects` extension as a requirement of Vulkan, failing if not present. This is functionally the same as requiring a higher GLSL version.

```glsl
// In some shader file.

// Required extensions for the shader.
#extension GL_ARB_separate_shader_objects : require

// Includes for functions.
#include <minecraft:light.glsl>
#include <minecraft:fog.glsl>
#include <minecraft:dynamictransforms.glsl>
#include <minecraft:projection.glsl>
#include <minecraft:sample_lightmap.glsl>

// Layout followed by the location.
layout(location = 0) in vec3 Position;
layout(location = 1) in vec4 Color;
layout(location = 2) in vec2 UV0;
layout(location = 3) in ivec2 UV1;
layout(location = 4) in ivec2 UV2;

// The location specified by the out uniforms in
// the vertex shader must match the `in`s in the
// fragment shader.
// Since it matches by location now, the variable names
// are ignored, but it is recommended to keep them consistent.
layout(location = 0) out float sphericalVertexDistance;
layout(location = 1) out float cylindricalVertexDistance;
```

### Order Independent Transparency (OIT)

Minecraft now supports the use of Order-Independent Transparency (OIT) through its improved transparency option. As the name implies, OIT is a rendering technique that tries to resolve alpha compositing without the underlying geometry having to be manually sorted by depth. Vanilla specifically uses [Wavelet OIT](https://arxiv.org/pdf/2201.00094), which is a topic too complicated for a Minecraft primer. If you're interested, I suggest reading up on the topic yourself.

For our understanding, OIT runs through three stages: depth bounds, transmittance, and accumulate. In both depth bounds and transmittance, only the alpha is updated. It is only during the accumulate phase that the pixel color is set. As such, OIT requires updating both the vertex and the fragment shader. For both depth bounds and transmittance, alpha only OIT is denoted by the shader define `OIT_ALPHA_ONLY`, which you will see scattered across the supported shaders.

Note that the following is the common usecase in vanilla's shaders. What you actually do will depend on your own.

In the vertex shader, `OIT_ALPHA_ONLY` is commonly checked to prevent the fog, lightmap, or overlay color as they do not contribute to the alpha passes:

```glsl
// In some .vsh file.
// In `assets/examplemod/shaders/core/example_oit_supported.vsh`
#version 330
#extension GL_ARB_separate_shader_objects : require

#include <minecraft:fog.glsl>
#include <minecraft:dynamictransforms.glsl>
#include <minecraft:projection.glsl>
#include <minecraft:sample_lightmap.glsl>

// Some passed in uniforms.
// We'll say UV0 is for our texture.
// UV2 is the lightmap.

layout(location = 0) in vec3 Position;
layout(location = 1) in vec4 Color;
layout(location = 2) in vec2 UV0;
layout(location = 3) in ivec2 UV2;

// The uniforms to pass to the fragment shader.

// If the flag is not defined, include the fog distance.
// Otherwise, it is not necessary for the alpha pass.
#ifndef OIT_ALPHA_ONLY
layout(location = 0) out float sphericalVertexDistance;
layout(location = 1) out float cylindricalVertexDistance;
#endif
layout(location = 2) out vec4 vertexColor;
layout(location = 3) out vec2 texCoord0;

// If the flag is not defined, include the lightmap.
// Otherwise, it is not necessary for the alpha pass.
#ifndef OIT_ALPHA_ONLY
uniform sampler2D Sampler2;
#endif

// Basic main function.
void main() {
    vec3 pos = Position + ModelOffset;
    gl_Position = ProjMat * ModelViewMat * vec4(pos, 1.0);

    // If the flag is not defined, set the color multiplied
    // by the lightmap; otherwise, just leave the color for
    // the alpha pass.
    #ifndef OIT_ALPHA_ONLY
    sphericalVertexDistance = fog_spherical_distance(Position);
    cylindricalVertexDistance = fog_cylindrical_distance(Position);
    vertexColor = Color * sample_lightmap(Sampler2, UV2);
    #else
    vertexColor = Color;
    #endif
    texCoord0 = UV0;
}
```

In the fragment shader, it must include `minecraft:oit.glsl`, making sure to handle the correct inputs based on `OIT_ALPHA_ONLY`. During the depth bounds and transmittance phase, the fragment color is not set as part of the alpha phase, and instead `executeAlphaOnlyPhase` is called. Then, during accumulate, the fragment color is determined from the texture color passed into `sampleColorForAccumulation`.

```glsl
// In some .fsh file.
// In `assets/examplemod/shaders/core/example_oit_supported.fsh`
#version 330
#extension GL_ARB_separate_shader_objects : require

#include <minecraft:fog.glsl>
#include <minecraft:dynamictransforms.glsl>
#include <minecraft:oit.glsl>

// The passed in uniforms.

uniform sampler2D Sampler0;

// Matching the flag based on the vertex shader outputs
// for the same reasons.
#ifndef OIT_ALPHA_ONLY
layout(location = 0) in float sphericalVertexDistance;
layout(location = 1) in float cylindricalVertexDistance;
#endif
layout(location = 2) in vec4 vertexColor;
layout(location = 3) in vec2 texCoord0;

// The output uniforms.

// If the flag is not defined, include the fragment color.
// Otherwise, it is not necessary for the alpha pass.
#ifndef OIT_ALPHA_ONLY
layout(location = 0) out vec4 fragColor;
#endif

// Calculate the final fragment color.
vec4 calculateFinalColor(vec4 color) {
    // If in the accumulate phase, sample the OIT calculated color.
    #ifdef OIT_ACCUMULATE
    color = sampleColorForAccumulation(color);
    vec4 fogColor = vec4(FogColor.rgb * color.a, FogColor.a);
    #else
    vec4 fogColor = FogColor;
    #endif

    // Compute the fragment color.
    return apply_fog(color, sphericalVertexDistance, cylindricalVertexDistance, FogEnvironmentalStart, FogEnvironmentalEnd, FogRenderDistanceStart, FogRenderDistanceEnd, fogColor);
}

void main() {
    vec4 color = texture(Sampler0, texCoord0) * vertexColor * ColorModulator;

    // If in an alpha only phase, update the OIT values.
    #ifdef OIT_ALPHA_ONLY
    executeAlphaOnlyPhase(gl_FragCoord.z, color.a);
    #else
    // Otherwise, set the fragment color.
    fragColor = calculateFinalColor(color);
    #endif
}
```

These shaders can then be used like any other `RenderPipeline`:

```java
public static final RenderPipeline.Snippet GENERIC_EXAMPLE_SNIPPET = RenderPipeline.builder()
    // Setting up all the layouts and transforms used.
    .withBindGroupLayout(BindGroupLayouts.GLOBALS)
    .withBindGroupLayout(BindGroupLayouts.FOG)
    .withBindGroupLayout(BindGroupLayouts.SAMPLER0)
    .withVertexBinding(0, DefaultVertexFormat.BLOCK)
    .withPrimitiveTopology(PrimitiveTopology.QUADS)
    .withDepthStencilState(DepthStencilState.DEFAULT)
    .withBindGroupLayout(BindGroupLayouts.DYNAMIC_TRANSFORMS)
    // Specify our vertex shaders and the like.
    .withVertexShader(Identifier.withNamespaceAndPath("examplemod", "core/example_oit_supported"))
    .withFragmentShader(Identifier.withNamespaceAndPath("examplemod", "core/example_oit_supported"))
    .buildSnippet();

public static final RenderPipeline.Snippet EXAMPLE_OIT_SUPPORTED_SNIPPET = RenderPipeline.builder(GENERIC_EXAMPLE_SNIPPET)
    // Setting up all the layouts and transforms used.
    .withBindGroupLayout(BindGroupLayouts.SAMPLER2)
    .withBindGroupLayout(BindGroupLayouts.PROJECTION)
    .buildSnippet();

public static final RenderPipeline CLASSIC_EXAMPLE_OIT_SUPPORTED = RenderPipeline.builder(EXAMPLE_OIT_SUPPORTED_SNIPPET)
    .withLocation(Identifier.withNamespaceAndPath("examplemod", "pipeline/example_oit_supported"))
    .withColorTargetState(new ColorTargetState(BlendFunction.TRANSLUCENT))
    .withDepthStencilState(DepthStencilState.DEFAULT)
    .build();
```

The pipeline on its own cannot make use of OIT, however. For that, we need to create an `OitPipelineSet`, created through `builder`. At its core, the builder constructs three `RenderPipeline`s corresponding to each of the three phases. It takes in a base snippet that will be accessible to all three phases, but if one pipeline requires additional uniforms or layouts, it can be specified using `with*Modifier`. Finally, the OIT pipeline is built using `build`:

```java
public static final OitPipelineSet EXAMPLE_OIT_SUPPORTED = OitPipelineSet.builder(
        // A suffix for naming the pipeline.
        "example_oit_supported",
        // The basic builder for the three phases.
        // Note that we pass in a snippet containing all of the
        // non-phase isolated shader code.
        RenderPipeline.builder(GENERIC_EXAMPLE_SNIPPET)
    )
    // If we need to modify the pipeline, we can call:
    // - withDepthBoundsModifier
    // - withTransmittanceModifier
    // - withAccumulateModifier
    // In our case, our shader makes use of Sampler2 for the final color,
    // so it should be provided during the final phase (i.e., accumulate).
    .withAccumulateModifier(accumulate -> accumulate.withBindGroupLayout(BindGroupLayouts.SAMPLER2))
    .build();
```

Then, within our `RenderType`, we can specify the OIT pipeline to use by calling `RenderSetup#setOitPipelines`:

```java
public static RenderType oitSupported(Identifier texture) {
    // Set the classic pipeline without OIT support.
    RenderSetup state = RenderSetup.builder(CLASSIC_EXAMPLE_OIT_SUPPORTED)
        // Set the OIT pipelines.
        .setOitPipelines(EXAMPLE_OIT_SUPPORTED)
        .withTexture("Sampler0", texture)
        .useLightmap()
        .sortOnUpload()
        .createRenderSetup();
    return RenderType.create("examplemod:oit_supported", state);
}
```

With all that, we can then make use of the OIT pipeline by calling `RenderType#prepare` and finally `PreparedRenderType#drawFromBufferOit` within a `RenderPass`, setting all the required uniforms.

### Feature Phase Changes

The render phases used by the `FeatureRenderDispatcher` have been partially reorganized due to some renames.

`SubmitNodeCollection#seeThroughNameTags` is now called `seeThrough` to hold both name tags and text with fonts having `DisplayMode#SEE_THROUGH`. Additionally, `nameTag` only contains the translucent portion, with anything solid being sent to `solid`. `gizmos` was renamed to `translucentGizmos`, while `alwaysOnTop` was renamed to `alwaysOnTopGizmos`. Solid gizmos were also merged into `solid`.

Additionally, if improved transparency (OIT) is enabled, then `seeThrough`, `shadows`, `nameTags`, `texts`, `shapeOutlines`, `translucentBlocksAndItems`, `translucentModels`, `translucentCustomGeometry`, `breakingOverlay`, `afterTerrain`, and `translucentGizmos` all use `oitTranslucent`. This means that if you have a `SubmitNodeStorage` opted into OIT via `setUseImprovedTransparency`, you cannot use `FeatureRenderDispatcher#renderAllFeatures` as it will render the same phase elements multiple times.

### Palette Metadata

A new metadata section has been added to the PNG mcmeta for marking a texture as paletted, replacing the armor trims atlas. The section is denoted by a `palette` key, specifying the `base_palette` of the corresponding texture relative to `textures/palettes`.

Textures making use of the palette section are handled through `PalettedTextureManager`, which dynamically constructs a texture containing the permutations for all palettes defined relative to `textures/palettes`. Note that this is only used for entity textures. Item textures still remain the same.

```json5
// For some PNG texture
// In assets/examplemod/textures/trims/example_trim.png.mcmeta
{
    // The palette metadata section
    "palette" : {
        // Points to `assets/minecraft/textures/palettes/trim_base.png`
        "base_palette" : "minecraft:trim_base"
    }
}
```

To actually make use of the paletted texture, it can be requested through `PalettedTextureManager#getOrPrepare`, passing in the base texture id along with the palette texture id. The identifier to the associated texture can then be obtained through the returned `PalettedTextureManager$Handle#textureLocation`, like `EquipmentLayerRenderer` does.

The metadata section also changes how trim materials are defined and overridden. Now the `TrimMaterial` takes in a `palette_id` instead of an `asset_name`, pointing directly to the associated palette to use. The overrides are then specified on the `EquipmentClientInfo` through `trim_overrides`. Each override defines a `EquipmentClientInfo$TrimPredicate` indicating for what material and pattern the override should match, and the replacement texture and palette identifiers to use instead of the ones specified by the material and pattern.

```json5
// For some equipment assets.
// In assets/examplemod/equipment/example.json
{
    // ...
    "trim_overrides": [
        {
            "when": {
                // When the trim material is diamond.
                "material": "minecraft:diamond",
                // And when the pattern is coast.
                "pattern": "minecraft:coast"
            },
            // Use the overridden palette instead of the 
            // one specified by the trim material.
            // Points to `assets/minecraft/textures/palettes/trim/diamond_darker.png`
            "palette": "minecraft:trim/diamond_darker",
            // Use the overridden texture.
            // How it resolves depends on use case.
            // In `EquipmentLayerRenderer`, it overrides the
            // trim pattern asset.
            // Points to `assets/minecraft/textures/trims/entity/<layer>/snout.png`
            "texture": "minecraft:snout"
        }
  ]
}
```

### Item Quads

Items now handle submitting their quads through `ItemQuads`. This functionally is a wrapper around a list of `BakedQuad`s, separating the quads into whether they are solid and translucent in response to the feature phase changes. `ItemQuads#solid` are submitted to the general `SubmitNodeCollection#solid` phase, `ItemQuads#translucent` are submitted to `SubmitNodeCollection#translucentBlocksAndItems`, and if the item has an outline, `ItemQuads#all` are submitted to `SubmitNodeCollection#outline`.

Creating an `ItemQuads` from a list of `BakedQuad`s can be easily done by calling `ItemQuads#split`:

```java
// For some List<BakedQuad> quads
ItemQuads itemQuads = ItemQuads.split(quads);

// For some QuadCollection collection
ItemQuads itemQuads = ItemQuads.split(collection.getAll());
```

Given that the quads could be called every frame, the `ItemQuads` should be stored in their used location whenever possible:

```java
// An example `ItemModel` implementation
public class ExampleItemModel implements ItemModel {

    private final ItemQuads itemQuads;

    private ExampleItemModel(QuadCollection collection) {
        // Store the item quads
        this.itemQuads = ItemQuads.split(collection.getAll());

        // ...
    }

    @Override
    public void update(ItemStackRenderState output, ItemStack item, ItemModelResolver resolver, ItemDisplayContext displayContext, @Nullable ClientLevel level, @Nullable ItemOwner owner, int seed) {
        output.appendModelIdentityElement(this);
        ItemStackRenderState.LayerRenderState layer = output.newLayer();

        // ...

        // Set the quads on the layer
        layer.setQuads(this.itemQuads);

        // ...
    }

    // ...
}
```

### Submitting Crumbling Overlays

The `ModelFeatureRenderer$CrumblingOverlay` applied to entity models (e.g., banners, chests, etc.) are now passed to `OrderedSubmitNodeCollector#submitCrumblingOverlay` for rendering. As such, `submitModel` and `submitModelPart` no longer take in the crumbling overlay.

`OrderedSubmitNodeCollector#submitCrumblingOverlay` takes in the same fields as `submitModel` and `submitModelPart` aside from the `TextureAtlasSprite` and the outline color:

```java
// For some `OrderedSubmitNodeCollector` collector
collector.submitCrumblingOverlay(
    // The entity model to render.
    model,
    // The render state extracted from the associated entity.
    state,
    // The pose stack.
    poseStack,
    // The render type. This should be what was used to render
    // the actual entity model.
    renderType,
    // The light coordinates. This should normally be obtained
    // from the submission context.
    LightCoordsUtil.FULL_BRIGHT,
    // The overlay coordinates. This should normally be obtained
    // from the submission context.
    OverlayTexture.NO_OVERLAY,
    // The tint color to apply to the texture.
    -1,
    // The crumbling overlay of how much progress has been made.
    crumblingOverlay
);
```

Like before, the passed in `RenderType` is only used to check whether it `RenderType#affectsCrumbling`, which must be `true` for the overlay to be submitted; and if the `RenderType#hasBlending`, which submits to the `SubmitNodeCollection#breakingOverlay` phase if `true`, and the `SubmitNodeCollection#solid` phase if `false`. The actual `RenderType` submitted is pulled from `ModelBakery#DESTROY_TYPES` (or `DESTROY_TYPES_OIT` if improved transparency is enabled) using the progress display.

- `assets/minecraft/atlases/armor_trims.json` is removed
- `assets/minecraft/models/block/template_farmland.json` -> `template_cube_bottom_top_indented.json`, not one-to-one
- `assets/minecraft/models/item/light.json` is removed
- `assets/minecraft/post_effect/transparency.json` is removed
- `assets/minecraft/shaders/core`
    - `blit_depth.fsh` - A fragment shader that sets the depth based on the red channel of the sampled texture.
    - `integrate_depth.fsh` - A fragment shader that sets the depth based on the red channel of the sampled texture, discarding if the depth is 0.
    - `oit_composite.fsh` - An order-independent transparency fragment shader that composites a sampler with its depth bounds.
    - `oit_depth_bounds_cull.fsh` - An order-independent transparency fragment shader that sets the depth from its bounds.
    - `rendertype_clouds` -> `clouds`
    - `rendertype_world_border` -> `world_border`
    - `text_background` is removed
        - Merged into `text`
- `assets/minecraft/shaders/include`
    - `oit*.glsl` - Utilities for order-independent transparency shaders.
    - `terrainglobals.glsl` - The UBO definition for the terrain uniform.
    - `texture_sampling.glsl` - Utility for texture sampling.
- `assets/minecraft/shaders/post/transparency.json` is removed
- `com.mojang.blaze3d`
    - `GLFWErrorCapture` class is removed
    - `GLFWErrorScope` class is removed
    - `GpuDeviceLossException` -> `renderpearl.api.device.GpuDeviceLossException`
    - `GpuFormat` -> `renderpearl.api.GpuFormat`
    - `GpuOutOfMemoryException` -> `renderpearl.api.device.GpuOutOfMemoryException`
    - `IndexType` -> `renderpearl.api.pipeline.IndexType`
    - `PrimitiveTopology` -> `renderpearl.api.pipeline.PrimitiveTopology`
- `com.mojang.blaze3d.buffers`
    - `GpuBuffer` -> `renderpearl.api.buffers.GpuBuffer`, now an interface from a class
        - The class portion has been moved into `renderpearl.backend.common.BaseGpuBuffer`
    - `GpuBufferSlice` -> `renderpearl.api.buffers.GpuBufferSlice`
    - `GpuFence` -> `renderpearl.api.commands.GpuFence`
        - `NO_TIMEOUT` - A constant that represents that awaiting the fence completion should not timeout with an exception.
- `com.mojang.blaze3d.opengl.*` -> `renderpearl.backend.opengl.*`
    - `FrameBufferCache`
        - `getFbo` now has an overload that takes in the mipmap offset `int`
        - `$CacheKey` now takes in a mipmap offset `int`
    - `GlCommandEncoder#executeDrawMultiple` is removed
        - The frontend now just calls `executeDraw` multiple times
    - `GlDevice` now takes in the `GlBackend` instead of the `long` window handle and default `ShaderSource`
        - `heuristics` - Returns the OpenGl device heuristics.
        - `getOrCompilePipeline`, `getOrCompileShader` are removed
            - Handled by the `GlPipelineRecompiler`
        - `precompilePipeline` -> `GpuDeviceBackend#compilePipeline`, not one-to-one
        - `vertexArrayCache` is removed
    - `GlHeuristics#isNvidia`, `couldBeIntelGen7` - Checks for specific graphics card types.
    - `GlProgram`
        - `BUILT_IN_UNIFORMS`, `INVALID_PROGRAM` are removed
        - `link` now takes in a list of `GlShaderModules` instead of the specific vertex and fragment shader along with the `VertexFormat` bindings
        - `setupBindGroupLayouts` now takes in a list of `BindGroupLayout$UniformDescriptions` instead of the raw `BindGroupLayout`s
        - `getUniform` now takes in an `int` index instead of the `String` name
        - `uniformCount` - Returns the number of uniforms used by the program.
        - `getUniforms` is removed
        - `pushConstant` - Returns the UBO uniform emulating the push constant.
    - `GlRenderPass` no longer takes in a `boolean` for whether there is a depth texture
        - `indexBufferDirty` - Whether the index buffer has been bound since the last draw.
        - `scissorStateDirty` - Whether the scissor state has changed since the last draw.
        - `dirtyUniforms`, `anyUniformDirty` - Handles the marking of new data in uniforms.
        - `pushConstants`, `pushConstantsDirty` - Push constants defined in the pass.
    - `GlRenderPipeline` is now a final class instead of a record
        - The constructor is now package-private from `public`
        - `program` - The program used by the pipeline.
        - `vertexArray` - The vertex attributes used by the pipeline.
        - `bind` - Binds the pipeline for use.
        - `primitiveTopology` - The topology used by the pipeline.
    - `GlShaderModule`
        - `getId`, `getDebugLabel` replaced by `getLabel`
        - `getType` - Returns the type of the shader.
    - `GlStateManager#_glReadBuffer` - Reads the buffer using the given color buffer.
    - `GlSurface` constructor is now package-private from `public`
    - `Uniform$Utb` no longer takes in the `int` location
    - `VertexArrayCache` -> `VertexArray`, now sealed to `$Emulated`, `$Seperate`; not one-to-one
        - The array objects are now owned by a given `GlRenderPipeline` instead of being globally cached
        - `create` replaced by `createSource`, providing the lambda to construct the array instead of the array itself
        - `bindVertexArray` replaced by `bind`, only taking in the `GpuBufferSlice` vertex buffers
        - `$Emulated`, `$Separate` constructors are now `private` instead of `public`
- `com.mojang.blaze3d.pipeline`
    - `BindGroupLayout` -> `renderpearl.api.pipeline.BindGroupLayout`
        - `getSamplers`, `getUniforms` are removed
        - `flattenSamplers` is removed
        - `$Builder#withSampler` is removed
    - `BlendEquation` -> `renderpearl.api.pipeline.BlendEquation`
    - `BlendFunction` -> `renderpearl.api.pipeline.BlendFunction`
        - `MAX` - Blends the source and destination by selecting the pixel with the maximum value.
    - `CompiledRenderPipeline` -> `renderpearl.api.pipeline.CompiledRenderPipeline`
        - `isValid` is removed
        - `isClosed` - Checks whether the pipeline has been closed.
        - `$Pending` - A pipeline representation that hasn't been fully compiled yet.
    - `DepthStencilState` -> `renderpearl.api.pipeline.DepthStencilState`
    - `PipelineCache` - A class for caching the compiled pipelines.
    - `RenderPipeline` -> `renderpearl.api.pipeline.RenderPipeline`
        - The constructor now takes in a map of `ShaderType` to ids instead of specifying the `Identifier` for each shader type directly, and the `int` size of the push constant
        - `getColorTargetState` is removed
        - `getVertexShader`, `getFragmentShader` merged in `getShaders`
        - `pushConstantSize` - Returns the size of the push constant.
        - `$Snippet` now takes in a map of `ShaderType` to ids instead of specifying the `Identifier` for each shader type directly, and the `int` size of the push constant
    - `RenderTarget` now takes in a nullable `GpuFormat` for the depth instead of just a `boolean` to determine whether to use depth
        - `useDepth` replaced by `depthFormat`
            - The original `boolean` can be queried via `hasDepth`
        - `format `-> `colorFormat`
        - `copyColorFrom` - Copies the color texture from another `RenderTarget`.
    - `TextureTarget` now takes in a nullable `GpuFormat` for the depth instead of just a `boolean` to determine whether to use depth
- `com.mojang.blaze3d.platform`
    - `BlendFactor` -> `renderpearl.api.pipeline.BlendFactor`
    - `BlendOp` -> `renderpearl.api.pipeline.BlendOp`
    - `ClipboardManager`
        - `FORMAT_UNAVAILABLE` is removed
        - `getClipboard` no longer has any arguments
        - `setClipboard` no longer takes in the `Window`
    - `CompareOp` -> `renderpearl.api.pipeline.CompareOp`
    - `GLX` is removed, replaced by `SdlDebug` and `SDLEventHandler`
    - `IconSet`
        - `getStandardIcons`, `getFile` now take in `PackMetadataResources` instead of `PackResources`
        - `getMacIcon` is removed
    - `InputConstants` have been remapped for SDL
        - `CURSOR*` are removed
        - `KEYCODE_*` - The logical codes for the given key.
        - `isKeyDown` no longer takes in the `Window`
        - `setupKeyboardCallbacks`, `setupMouseCallbacks` are removed
        - `grabOrReleaseMouse` split into `grabMouse`, `releaseMouse`
        - `isRawMouseInputSupported`, `updateRawMouseInput` are removed
        - `$Type`
            - `KEYSYM` -> `KEYBOARD`
            - `SCANCODE` is removed
    - `MacosUtil` is now `final`, with its constructor made `private`
        - `exitNativeFullscreen`, `clearResizableBit`, `loadIcon`, `setWindowColorSpaceForOpenGLBecauseGLFWDoesnt` are removed
        - `disableCloseWindowMenuItem` - Disables the close window menu.
        - `setFullscreenMenuVisibility` - Toggles the fullscreen menu visibility.
        - `setCtrlClickEmulatesRightClick` - Toggles whether control emulates right click.
    - `MessageBox` constants are either `private` or removed
    - `Monitor` now takes in an `int` id instead of a `long` handle
    - `MonitorManager` is no longer `AutoCloseable`
        - `getMonitor` now takes in an `int` id instead of a `long` handle
        - `clamp` is removed
        - `onDisplayConnected`, `onDisplayDisconnected`, `onDisplayModeChanged` - Handles a display's state.
    - `NativeImage`
        - `read(NativeImage$Format, InputStream)`, `read(NativeImage$Format, ByteBuffer)` are removed
        - `$Format#supportedByStb` is removed
    - `NativeLibrariesBootstrap#loadGlfw` replaced by `loadSdl`
    - `PolygonMode` -> `renderpearl.api.pipeline.PolygonMode`
    - `SdlDebug` - A class for handling SDL debug logging.
    - `SDLEventHandler` - An event handler for SDL.
    - `TextInputManager`
        - `notifyIMEChanged` is removed
        - `tick` is removed
        - `startTextInput` now takes in an `Object` owner
        - `stopTextInput` now has an overload that takes in an `Object` owner
        - `onTextInputFocusChange` now takes in an `Object` owner
    - `VideoMode` now takes in a `SDL_DisplayMode` instead of a `Buffer` or `GLFWVidMode`
        - `CODEC` - The codec for `VideoMode`.
        - `getRefreshRate` now returns a `float` instead of an `int`
        - `refreshRateLabel` - A formatted refresh rate.
    - `Window` no longer throws a `BackendCreationException`, and now takes in an `int` for the maximum size of the window
        - `MIN_WINDOW_WIDTH`, `MIN_WINDOW_HEIGHT` - The minimum bounds of the game window.
        - `createGlfwWindow` replaced by `createWindow`, now private
        - `createIconSurface` - Creates the surface for the given icon image.
        - `getActiveVideoMode` -  Returns the active video mode on the current display.
        - `getActiveDisplayMode` - Returns the active display mode.
        - `getRefreshRate` is removed
        - `checkGlfwError` is removed
        - `handleEvent` - Handles events sent from SDL.
        - `setIcon` now takes in `PackMetadataResources` instead of `PackResources`
        - `getErrorSection` - Returns the error sent by the window.
        - `defaultErrorCallback` is removed
        - `queryFramebufferSize` - Gets the size of the main framebuffer from the given window.
        - `toggleFullScreen` replaced by `setFullscreen`, `setExclusiveFullscreen`, taking in the `boolean` for fullscreen mode
        - `isExclusiveFullscreen` - Whether the window is in exclusive fullscreen mode.
        - `isFullscreen` is removed
        - `setWindowMaxSize` - Sets the window maximum size.
        - `updateRawMouseInput` is removed
        - `isMinimized` is removed
        - `getPixelDensity` - Returns the pixel density in the current window
        - `setQuitShortcuts` - Sets whether windows should close on alt + f4.
        - `backend` is removed
        - `$FramebufferSize` - The size of the current window framebuffer.
    - `WindowEventHandler#fullscreenStateChanged` - Handles when the game changes its fullscreen state.
- `com.mojang.blaze3d.platform.cursor.CursorType#select` no longer takes in a `Window`
- `com.mojang.blaze3d.preprocessor.GlslPreprocessor` class is removed
- `com.mojang.blaze3d.resource.RenderTargetDescriptor` now takes in `$TextureProperties` for the color and depth instead of the depth `boolean`, `Vector4fc` clear color, and `GpuFormat`
    - `$TextureProperties` - The properties for a given texture.
- `com.mojang.blaze3d.shaders`
    - `GpuDebugOptions` -> `renderpearl.api.device.GpuDebugOptions`
    - `ShaderSource` -> `renderpearl.api.pipeline.ShaderSource`, now implements `AutoCloseable`
        - `get` -> `getShader`
        - `getInclude` - Returns the cached included shader.
        - `$CachedIncludeSource` - A reference to the cached include source.
    - `ShaderType` -> `renderpearl.api.pipeline.ShaderType`
    - `UniformType` -> `renderpearl.api.pipeline.UniformType`
- `com.mojang.blaze3d.systems`
    - `BackendCreationException` -> `renderpearl.api.device.BackendCreationException`
    - `CommandEncoder` -> `renderpearl.frontend.FrontendCommandEncoder`
        - Definition interface in `renderpearl.api.commands.CommandEncoder`
    - `CommandEncoderBackend` -> `renderpearl.backend.api.CommandEncoderBackend`
    - `DeviceFeatures` -> `renderpearl.api.device.DeviceFeatures`
        - `wireframeFillMode` - If `PolygonMode#WIREFRAME` can be used to render the model.
    - `DeviceInfo` -> `renderpearl.api.device.DeviceInfo`
    - `DeviceLimits` -> `renderpearl.api.device.DeviceLimits`
        - `maxDrawIndirectDrawCount` - The maximum number of indirect draws that can be made per `drawIndirect` call.
    - `DeviceType` -> `renderpearl.api.device.DeviceType`
    - `GpuBackend` -> `renderpearl.api.device.GpuBackend`
        - `setWindowHints`, `handleWindowCreationErrors` are removed
        - `loadLibrary`, `unloadLibrary` - Handles the native libraries required for the backend to run.
        - `createWindow` - Creates the window for the application.
        - `createDevice` now only takes in the `GpuDebugOptions`
    - `GpuDevice` -> `renderpearl.frontend.FrontendGpuDevice`, no longer taking in the `Runnable` for the critical shader loader
        - Definition interface in `renderpearl.api.device.GpuDevice`
        - `STRICT_VALIDATION` - Whether the GPU calls should be strictly validated.
        - `createSurface` now takes in a `BooleanSupplier` for if the game is minimized
        - `precompilePipeline` replaced by `compilePipeline`
        - `clearPipelineCache` is removed
        - `loadCriticalShaders` is removed
        - `getTimestampNow` is removed
    - `GpuDeviceBackend` -> `renderpearl.backend.api.GpuDeviceBackend`
        - `createSurface` now takes in a `BooleanSupplier` for if the game is minimized
        - `precompilePipeline` replaced by `compilePipeline`
        - `clearPipelineCache` is removed
        - `getTimestampNow` is removed
        - `getTimestampCalibrationOffset` - Returns the nanosecond offset between the current host time and the device time.
    - `GpuQuery` -> `renderpearl.api.commands.GpuQuery` now implements `UncheckedAutoCloseable`
    - `GpuQueryPool` -> `renderpearl.api.commands.GpuQueryPool` now implements `UncheckedAutoCloseable`
    - `GpuSurface` -> `renderpearl.frontend.FrontendGpuSurface`
        - Definition interface in `renderpearl.api.device.GpuSurface`
        - `$Configuration` -> `GpuSurface$Configuration`
        - `$PresentMode` -> `GpuSurface$PresentMode`
    - `GpuSurfaceBackend` -> `renderpearl.backend.api.GpuSurfaceBackend`
    - `HintsAndWorkarounds` -> `renderpearl.api.device.HintsAndWorkarounds`
        - `isExplicitDepthRequired` - If the depth of the fragment coordinate must be explicitly specified.
        - `multiDrawIndirectHasKnownIssues` - If indirect multidraw has issues when calling.
    - `RenderPass` -> `renderpearl.frontend.FrontendRenderPass`
        - Definition interface in `renderpearl.api.commands.RenderPass`
        - The class now implements `RenderPass$UniformUploader`
        - `indexBuffer` - The current index buffer.
        - `uniforms` - The current uniforms.
        - `setPipeline` now takes in a `CompiledRenderPipeline` instead of the `RenderPipeline`
        - `bindTexture` is removed
        - `pushConstants` - Pushes constants to the buffer.
        - `$Draw` -> `RenderPass$Draw`
        - `$RenderArea` -> `RenderPass$RenderArea`
        - `$UniformUploader` -> `RenderPass$UniformUploader`
            - `upload` -> `setUniform`
            - `pushConstants` - Pushes constants to the buffer.
    - `RenderPassBackend` -> `renderpearl.backend.api.RenderPassBackend`
        - `setUniform` merged into one call with an `Object` value
        - `drawMultipleIndexed` moved to `RenderPass#drawMultipleIndexed`
    - `RenderPassDescriptor` -> `renderpearl.api.commands.RenderPassDescriptor`, now a record
        - The original descriptor is now in `RenderPassDescriptor$Builder`
        - `create` -> `builder`, not one-to-one
        - `$Builder#build` - Builds the pass descriptor.
    - `RenderSystem`
        - `outputColorTextureOverride`, `outputDepthTextureOverride` are removed
        - `isRenderingLevel` - Whether the level is currently being rendered.
        - `setFallbackPipelineCache`, `setCurrentPipelineCache` - Handles setting the pipeline cache.
        - `getCompiledPipelineNullable`, `getCompiledPipeline` - Gets the compiled pipeline.
        - `pollEvents` now takes in an `SDLEventHandler`
        - `pumpEvents` - Flushes and repolls the SDL events.
        - `setErrorCallback` is removed
        - `trackBackendLibraryForShutdown`, `unloadTrackedBackendLibrary` - Handles tracking the `GpuBackend`.
        - `isWireframeAvailable` - Whether the models can be rendered as wireframes.
        - `getDynamicUniforms` now returns `DynamicGpuData` instead of `DynamicUniforms`
        - `resizeAllAutoStorageIndexBuffers` - Resizes the auto storage index buffers.
        - `$AutoStorageIndexBuffer`
            - `requestIndexCount` - Sets the index count.
            - `resizeToRequestedIndexCount` - Ensures the buffer as the requested index count.
            - `getBuffer` - Gets the backing `GpuBuffer`.
    - `SurfaceException` -> `renderpearl.api.device.SurfaceException`
    - `TracyGpuProfiler` -> `renderpearl.frontend.TracyGpuProfiler`
        - The constructor now takes in a `FrontendGpuDevice` instead of a `GpuDevice`
- `com.mojang.blaze3d.textures.*` -> `renderpearl.api.textures.*`
    - `GpuSampler` is now an interface instead of an abstract class
        - `isClosed` - Whether the sampler has been closed.
    - `GpuTexture` is now an interface from a class
        - The class portion has been moved into `renderpearl.backend.common.BaseGpuTexture`
        - `isClosed` - Whether the texture has been closed.
    - `GpuTextureView` is now an interface from a class
        - The class portion has been moved into `renderpearl.backend.common.BaseGpuTextureView`
        - `isClosed` - Whether the texture view has been closed.
- `com.mojang.blaze3d.util.TransientBlockAllocator` -> `renderpearl.backend.util.TransientBlockAllocator`
    - The generic now extends `$Allocator$Block`
    - `$Allocator` generic now extends `$Allocator$Block`
    - `$Block` - A representation of a 'block' of data.
    - `$CpuBlock` - A block on the CPU.
- `com.mojang.blaze3d.vertex`
    - `DefaultVertexFormat`
        - `UV3_SEMANTIC_NAME` - The name for UV3.
        - `CHUNK_POSITION_SEMANTIC_NAME`, `CHUNK_VISIBILITY_SEMANTIC_NAME` - Names for chunk settings.
        - `CHUNK_DATA_INSTANCED` - The format for chunk data.
        - `ENTITY_GLINT_SPECIAL` - The format for special entity glints.
    - `PoseStack`
        - `mulPose` -> `rotate`
            - Now has an overload that takes in an `Axis` and the radian angle
        - `rotateDegrees` - Rotates around the `Axis` in degrees.
        - `$Pose`
            - `rotate` now has an overload that takes in an `Axis` and the radian angle
            - `rotateDegrees` - Rotates around the `Axis` in degrees.
        - `SheetedDecalTextureGenerator#setSheetedDecalUv` - Sets the UV coords for the decal.
    - `VertexConsumer`
        - `setUv3` - Sets the UV coords for the sheeted decal.
        - `putBakedQuadWithGlint` - Puts the baked quad data that has a glint.
    - `VertexFormat` -> `renderpearl.api.vertex.VertexFormat`
    - `VertexFormatElement` -> `renderpearl.api.vertex.VertexFormatElement`
- `com.mojang.blaze3d.vulkan.*` -> `renderpearl.backend.vulkan.*`
    - `VulkanBackend`
        - Constants moved to `VulkanFeatureSets`
            - `REQUIRED_DEVICE_EXTENSIONS`, `REQUIRED_DEVICE_FEATURES` merged into `VulkanFeatureSets#REQUIRED_FEATURESET`
    - `VulkanBindGroupLayout` record is removed
    - `VulkanConst#toVk` now has an overload that takes in the `ShaderType`
    - `VulkanDevice` no longer takes in the `ShaderSource`
    - `VulkanGpuTextureView` now extends `BaseGpuTextureView` instead of `GpuTextureView`
    - `VulkanRenderPass#VALIDATION` -> `FrontendGpuDevice#STRICT_VALIDATION`
        - `uniforms` is now a `ReferenceList` instead of a `HashMap`
        - `textures` is removed
        - `$TextureViewAndSampler` -> `renderpearl.util.TextureViewAndSampler`
    - `VulkanRenderPipeline` is now a final class instead of a record
        - The constructor no longer takes in a `RenderPipeline` and takes in a `LongList` shader modules instead of `long` modules for the vertex and fragment, the `long` descriptor set, and a list of `BindGroupLayout$UniformDescription`s instead of the `VulkanBindGroupLayout`
        - `compile` now takes in the `BackendRenderPipeline$CreateInfo` instead of the `VulkanBindGroupLayout`, `RenderPipeline`, and `long` modules
        - `device` - The gpu device of the pipeline.
        - `withDepthPipeline`, `withoutDepthPipeline` - The specific pipelines given the depth.
        - `pipelineLayout` - The layout for the pipeline, references a `VkPipelineLayout`.
        - `uniforms` - The uniforms used by the pipeline.
    - `VulkanUtils`
        - `enumerateExtensions` - Returns all available extensions for the device.
        - `enumerateFeatures` - Returns all available features for the device.
- `com.mojang.blaze3d.vulkan.checkpoints.*` -> `renderpearl.backend.vulkan.checkpoints.*`
- `com.mojang.blaze3d.vulkan.glsl`
    - `GlslCompiler` -> `renderpearl.frontend.shaders.GlslCompiler`
        - The constructor now takes in the `boolean`s for zero to one, and draw parameters
        - `createIntermediary` is removed
        - `compile` replaced by `compileSpv`
        - `$CompiledModules`is removed
    - `IntermediaryShaderModule` record is removed
        - This is technically replaced in usage by `SpvModule`, but it is completely different
    - `ShaderCompileException` -> `renderpearl.util.ShaderCompileException`
    - `SpvcUtil` class is removed
    - `SpvSampler` record is removed
    - `SpvUniformBuffer` record is removed
    - `SpvVariable` record is removed
- `com.mojang.blaze3d.vulkan.init.*` -> `renderpearl.backend.vulkan.init.*`
    - `VulkanPNextStruct` now takes in the next struct `Class`
        - There is also an overload that only takes the `Class`
    - `fieldOffset` - Returns the offset of the given field name.
- `com.mojang.renderpearl.backend.api.SpvModule` - An intermediate module representing the compiled shader in SPV.
- `com.mojang.renderpearl.backend.opengl.GlPipelineRecompiler` - Handles recompiling the SPV shader module into GLSL.
- `com.mojang.renderpearl.backend.vulkan.VulkanFeatureSets` - The feature sets for vulkan used by vanilla.
- `com.mojang.renderpearl.frontend.shaders`
    - `PipelineBuilder` - The compiler for pipelines and their associated shaders.
    - `SPIRVModule` - An intermediate shader module implementation in the SPIR-V format.
    - `SpvUtil` - A utility for working with SPVs.
- `com.mojang.renderpearl.util.UncheckedAutoCloseable` - An `AutoCloseable` that removes the `Exception` throws from `close`.
- `net.minecraft.client`
    - `Camera#extractRenderState` now takes in the `DeltaTracker` instead of the `float` partial tick
    - `ClientClockManager`
        - `getInstance` is now `public` from `private`, returning a `$ClientClockInstance`
        - `$ClockInstance` -> `$ClientClockInstance`, now `public` from `private`, implementing `ClockInstance`
    - `KeyboardManager`
        - `keyPress` is now `public` from `private`
        - `charTyped` is now `public` from `private`
        - `textInput`, `textEditing` - Handles input into a text box, including preedit.
        - `setup` is removed
    - `Minecraft`
        - `multiDrawIndirect` - Whether the terrain should make use of indirect multidraw calls.
        - `getPalettedTextureManager` - Gets the texture manager for palletted permutations.
    - `MouseHandler`
        - `onButton` is now `public` from `private`
        - `onScroll` is now `public` from `private`
        - `onDrop` is now `public` from `private`, taking in a list of `String` paths instead of `Path`s and the `int` number of failed files
        - `setup` is removed
        - `onMove` is now `public` from `private`, taking in the relative `double` x and y position
        - `resyncMousePosition` - Resyncs the mouse position.
    - `Options`
        - `DEBUG_GUI_SCALE_UNCHANGED` - A constant represent that the debug GUI scaling should remain unchanged.
        - `rawMouseInput` is removed
        - `quitShortcuts` - Whether quit shortcuts (alt + f4) can be used.
        - `ctrlClickEmulatesRightClick` - Whether control emulates right click.
        - `macFullscreenMenuVisibility` - Whether the fullscreen menu visibility on Macs.
        - `debugGuiScale` - The GUI scaling for the debug items.
- `net.minecraft.client.color.item`
    - `ItemTintSources` `minecraft:map_color` is removed
    - `MapColor` record is removed
- `net.minecraft.client.data.models`
    - `BlockModelGenerators`
        - `createFlatItemModelWithBlockTextureAndOverlay` -> `createTwoLayeredItemModel`, not one-to-one
        - `createFlatItemModel` - Creates a flat item with the given material.
        - `registerTwoLayerFlatItemModel` is removed
        - `createRotatedVariantBlock` now has an overload to specify the model either via a `TexturedModel$Provider` or `Identifier`
        - `createStrawBed` - Creates a straw bed.
        - `createFarmland` now takes in the farmland `Block`, the base and bottom `Material`s, and the `ModelTemplate`
        - `createDirtPath` is removed
        - `createShelfMushroom` - Creates a shelf mushroom.
        - `$BlockFamilyProvider#carpet` - Adds the carpet model for a block family.
        - `$PlantType#createItemModel` replaced by `createItemModelUsingBlockTexture`
    - `EquipmentAssetProvider#onlyHumanoid`, `humanoidAndMountArmor` now return a `$Builder` instead of the `EquipmentClientInfo` itself
    - `ItemModelGenerators`
        - `prefixForSlotTrim` is now `private` from `public`
        - `generateFlatItem` now as an overload that takes in the `Identifier` for the texture instead of an `Item`
        - `generateTrimmableItem` now takes in a map of `TrimMaterials$Palette` to `TrimMaterials$Palette` instead of a `ResourceKey<EquipmentAsset>`
        - `generateTrimmableArmorSet` - Generates a trimmable item model for each piece of armor.
        - `$TrimMaterialData#assets` -> `palette`, now a `TrimMaterials$Palette` instead of a `MaterialAssetGroup`
- `net.minecraft.client.data.models.model`
    - `ModelTemplates#FARMLAND` replaced by `CUBE_BOTTOM_TOP_INDENTED`
    - `TexturedModel`
        - `CUBE_TOP_BOTTOM` -> `CUBE_BOTTOM_TOP`
        - `CUBE_BOTTOM_TOP_INDENTED` - A model provider for an cube indented on the top.
- `net.minecraft.client.gui`
    - `Font#prepareBackground` - Creates a text effect for the given rectangle with the given color.
    - `GuiGraphicsExtractor`
        - `textWithWordWrap` now returns the next `int` y position free after wrapping
        - `setTooltipForNextFrame`, `tooltip` now have an overload that takes in a `boolean` of whether to have a bit more space between the first line and the rest
- `net.minecraft.client.gui.components`
    - `AbstractSelectionList#INWORLD_MENU_LIST_BACKGROUND` is now `public` from `private`
    - `RealmsButton` - A button for adding a new realm.
    - `ScrollableLayout#getScrollAmount` - The amount the layout has been scrolled.
    - `SubtitleOverlay$Subtitle#getClosest` -> `getBestSubtitleCandidate`
- `net.minecraft.client.gui.components.debug`
    - `DebugEntryPlayerSpeed`, `DebugScreenEntries#PLAYER_SPEED` - A debug entry that displays the camera entity's movement speed.
    - `DebugEntryPostEffect` -> `DebugEntryPostEffects`, now listing all active post effects
        - `DebugScreenEntries#POST_EFFECT` -> `POST_EFFECTS`
    - `DebugEntrySystemSpecs#getCpuInfo` - Information about the current CPU.
- `net.minecraft.client.gui.components.debugchart`
    - `AbstractDebugChart#extractRenderState` now takes in an `int` for the screen height
    - `ProfilerPieChart#extractRenderState` now takes in `int`s for the scaled screen width and height
- `net.minecraft.client.gui.components.events.GuiEventListener#capturesInput` - Whether this listener captures user input.
- `net.minecraft.client.gui.components.tabs.TabNavigationBar` no longer implements `NarratableEntry`, `Renderable`
- `net.minecraft.client.gui.font.FontTexture$Node` is removed
- `net.minecraft.client.gui.narration`
    - `NarrationElementOutput#narrationTrigger` - How the narration for the element is triggered.
    - `NarrationTrigger` - The possible triggers for narration of an element.
    - `ScreenNarrationCollector#update` now takes in the `NarrationTrigger`
- `net.minecraft.client.gui.screens`
    - `ConfirmLinkScreen` now takes in a `URI` instead of a `String`
        - `confirmLinkNow` methods that take in a `String` for the URI are removed
        - `confirmLink` methods that take in a `String` for the URI are removed
    - `MultiplayerOptionsScreen` class is removed
        - Replaced by `WorldOptionsScreen`
    - `PrivacyConfirmLinkScreen` now takes in a `URI` instead of a `String`
        - `confirmLinkNow(Screen, String)` is removed
    - `Screen`
        - `isInputCaptured` - If one of the screen elements are currently capturing user input.
        - `fillCrashDetails` is removed
        - `scheduleNarration` - Schedules immediate narration by the system.
        - `updateNarratorStatus` now takes in the `NarrationTrigger`
- `net.minecraft.client.gui.screens.inventory`
    - `AbstractSignEditScreen` now takes in a `SignTextSlot` instead of a `boolean` for if its front text
    - `HangingSignEditScreen` now takes in a `SignTextSlot` instead of a `boolean` for if its front text
    - `SignEditScreen` now takes in a `SignTextSlot` instead of a `boolean` for if its front text
    - `DifficultyButtons` -> `WorldOptionsScreen$DifficultyButtons`
    - `OptionsScreen` no longer implements `HasGamemasterPermissionReaction`
        - The constructor no longer takes in the `boolean` for whether the player is in a world
    - `WorldOptionsScreen` -> `.screens.WorldOptionsScreen`
- `net.minecraft.client.gui.screens.options.controls.KeyBindsScreen#refreshKeybindLabels` - Refreshes the keybind information.
- `net.minecraft.client.gui.screens.social.PresenceHandler#sendOfflinePresence` - Sends that the user is currently offline.
- `net.minecraft.client.input`
    - `InputQuirks`
        - `SIMULATE_RIGHT_CLICK_WITH_LONG_LEFT_CLICK` -> `EMULATE_RIGHT_CLICK_WITH_CTRL_KEY`
        - `SHIFT_INVERTS_SCROLL_AXIS` - Whether pressing shift inverts the scroll axis.
        - `isQuitShortcutDown` - Whether the quit shortcut is currently being pressed.
        - `keyboardTranslationKey` - Provides the translation key for the key modifier.
    - `InputWithModifiers`
        - `NOT_DIGIT` is removed
        - `shortcutKey` - The semantic key pressed.
        - `getDigit` is removed
    - `KeyEvent#scancode` -> `keycode`
        - This differs from GLFW events, where `key` was the semantic value while `scancode` is the physical location value; now, `key` is the physical location value while `keycode` is the semantic value.
    - `PreeditEvent#createFromCallback` -> `fromSdlTextEditing`, not one-to-one
- `net.minecraft.client.model.HumanoidModel#setupSwimAnimation` - Sets up the swim animation.
- `net.minecraft.client.model.effects.SpearAnimations`
    - `thirdPersonHandUse` now takes in the `HumanoidArm` instead of a `boolean` indicating if its the right arm
    - `thirdPersonAttackHand` now takes in the `float` animation progress
- `net.minecraft.client.model.monster.slime.SulfurCubeModel` now extends `EntityModel<SulfurCubeRenderState>` and implements `HeadedModel`
- `net.minecraft.client.model.monster.zombie`
    - `AbstractZombieModel` class is removed
    - `GiantZombieModel` now extends `HumanoidModel`
    - `ZombieModel` now extends `HumanoidModel`
- `net.minecraft.client.model.object.cushion.CushionModel` - An entity model for the cushion.
- `net.minecraft.client.multiplayer`
    - `ClientChunkCache#replaceWithPacketData` now takes in the `ClientboundLevelChunkPacketData` instead for the raw `FriendlyByteBuf`, heightmap `Map`, and block entity `Consumer`
    - `ClientLevel#addBreakingBlockEffect` -> `addBreakingBlockEffects`, taking in a `boolean` for whether to play sound
    - `MultiPlayerGameMode`
        - `createPlayer` now takes in an `ItemActivation`
        - `piercingAttack` now takes in a `SwingAnimation`
- `net.minecraft.client.particle`
    - `FallingLeavesParticle` now extends `FallingParticle`
        - `$PoplarProvider` - A provider for poplar falling leaves.
    - `FallingParticle` - A particle that falls like a leaf.
    - `FireworkParticles$Starter` now takes in a `boolean` for whether to play sound
    - `SingleQuadParticle$Layer` now takes in a nullable `OitPipelineSet`
    - `SoulParticle` -> `EmissiveRisingParticle`, not one-to-one
- `net.minecraft.client.player`
    - `FirstPersonHandsAndItems` - A representation of the items in first person mode, including their relative heights.
    - `ItemActivation` - A representation of an activated item (e.g., totems).
    - `LocalPlayer` now takes in an `ItemActivation`
        - `itemActivation` - Handles when an item should be activated.
        - `itemUsed`, `firstPersonHandsAndItems` - Handles first person item views.
        - `sendChanges` - Sends changes from the client to the server.
        - `drop` -> `MultiPlayerGameMode#dropItem`, not one-to-one
        - `setActivePostEffects`, `getActivePostEffects` - Handles the post effects applied to the player.
- `net.minecraft.client.renderer`
    - `BindGroupLayouts`
        - `MATRICES_PROJECTION` is removed
        - `TERRAIN_INFO` - Layout for terrain shaders.
        - `GLINT_SAMPLER` - Layout for overlaying glints.
        - `DEPTH_BOUNDS_SAMPLER` - Layout for handling the depth bounds within order-independent transparency pipelines.
        - `OIT_COEFFS_DEPTH_BOUNDS_SAMPLER`, `SAMPLER0_OIT_COEFFS_DEPTH_BOUNDS_SAMPLER`, `SAMPLER0_SAMPLER2_OIT_COEFFS_DEPTH_BOUNDS_SAMPLER`, `CLOUD_INFO_OIT_COEFFS_DEPTH_BOUNDS_SAMPLER` - Layouts for specific uniforms used in order-independent transparency pipelines.
    - `CloudRenderer`
        - `render` -> `prepare`
            - `render` has been split off to only handle the rendering of the clouds
        - `renderOit` - Renders with order-independent transparency.
    - `DebugCrosshairRenderer#render` now takes in the color and depth `GpuTextureView`
    - `DynamicGpuDataStorage` - A storage handler for writing dynamic data (e.g. uniforms, push constants), to the GPU.
    - `DynamicGpuDataStorageNonMapped` - A data storage that writes data to separate CPU and GPU buffers instead of a ring buffer.
    - `DynamicUniforms` replaced by `DynamicGpuData`
    - `DynamicUniformStorage` replaced by `DynamicGpuDataStorageMapped`
    - `GameRenderer` now implements `ResourceManagerReloadListener`
        - `PROJECTION_3D_HUD_Z_FAR` is removed
        - `END_OF_FRAME_POST_EFFECT` - Identifier for the end of frame post effect.
        - `itemInHandRenderer` -> `firstPersonHandsAndItemsRenderer`
        - The constructor now takes in the `ItemModelResolver`
        - `clearPostEffect`- > `clearSpectatedEntityPostEffect`
        - `togglePostEffect` -> `toggleSpectatorPostEffect`
        - `preloadUiShader` is now `static`, taking in the `ResourceManager` instead of a `ResourceProvider`
        - `spectatedEntityPostEffect`, `getRequestedPostEffects`, `getAppliedPostEffects` - Getters for the post effects applied.
        - `render` no longer takes in any arguments
        - `renderLevel` no longer takes in any arguments
        - `displayItemActivation` -> `LocalPlayer#displayItemActivation`
        - `useImprovedTransparency` - Whether the game should attempt to render with order-independent transparency.
    - `GlobalSettingsUniform#update` now takes in a `float` for the world's partial tick instead of the `DeltaTracker`
    - `ItemInHandRenderer`  -> `FirstPersonHandsAndItemsRenderer`, not one-to-one
    - `LevelRenderer`
        - `OIT_WAVELET_RANK`, `OIT_COEFFICIENT_COUNT`, `OIT_TRANSMITTANCE_TARGET_COUNT` - Order-independent transparency constants.
        - `render` no longer takes in the `DeltaTracker` or model view `Matrix4fc`, instead taking in a `boolean` for whether to require consistent depth
        - `prepareChunkRenders` now takes in a `boolean` for whether to respsect translucency ordering
        - `prepareChunkRendersIndirect` - Prepare the chunk section to render using indirect draw calls.
        - `doEntityOutline` -> `blitEntityOutline`
        - `isSectionCompiledAndVisible` now takes in the `long` chunk fade duration
        - `entityOutlineTarget`, `translucentTarget`, `itemEntityTarget`, `particlesTarget`, `weatherTarget`, `cloudsTarget` are removed
        - `isChunkRenderingUsingMultiDrawIndirect` - If the chunk renderer will use indirect multidraw calls.
        - `addTransientBlock`, `removeTransientBlocksInSection` - Handles transient block management.
    - `LevelTargetBundle`
        - `TRANSLUCENT_TARGET_ID`, `ITEM_ENTITY_TARGET_ID`, `PARTICLES_TARGET_ID`, `WEATHER_TARGET_ID`, `CLOUDS_TARGET_ID` are removed
        - `SORTING_TARGETS` is removed
        - `translucent`, `itemEntity`, `particles`, `weather`, `clouds` are removed
        - `alwaysOnTopDepth` - Resource handle for render targets that should always appear on top.
        - `depthBounds`, `depthBoundsCulled`, `transmittance`, `accumulate`, `oitCloudDepth`, `oitTerrainWithWaterPatchDepth` - Resource handles for order-independent transparency rendering.
    - `OrderedSubmitNodeCollector`
        - `submitTextBackground` - Submits a rectangle as a font effect.
        - `submitModel` now takes in a `UvMapping` instead of the `TextureAtlasSprite`
        - `submitModel` overloads no longer take in a `ModelFeatureRenderer$CrumblingOverlay`
        - `submitCrumblingOverlay` - Submits a crumbling overlay for an entity model.
        - `submitModelPart` now takes in a `UvMapping` instead of the `TextureAtlasSprite`
        - `submitModelPart` overloads no longer take in a `ModelFeatureRenderer$CrumblingOverlay`
        - `submitBreakingBlockModel` now takes in a `boolean` of whether the block is translucent
        - `submitItem` now takes in an `ItemQuads` instead of a list of `BakedQuad`s
    - `PostChain`
        - `id` - The identifier of the post effect.
        - `getReferencedExternalTargets` - Gets any referenced targets in the post effect.
        - `closePersistentTargets` - Destroys the buffers for any persistent targets.
    - `RenderPipelines`
        - `OIT_DEPTH_BOUNDS_SNIPPET`, `OIT_TRANSMITTANCE_SNIPPET`, `OIT_ACCUMULATE_SNIPPET` - Snippets for order-independent transparency.
        - `SOLID_TERRAIN_MULTIDRAW` - Pipeline for drawing solid terrain with multidraw calls.
        - `WIREFRAME_MULTIDRAW` - Pipeline for drawing wireframes with multidraw calls.
        - `CUTOUT_TERRAIN_MULTIDRAW` - Pipeline for drawing cutout terrain with multidraw calls.
        - `TRANSLUCENT_TERRAIN_MULTIDRAW` - Pipeline for drawing translucent terrain with multidraw calls.
        - `OIT_*` - Pipelines for drawing with order-independent transparency.
        - `ARMOR_TRANSLUCENT` -> `WOLF_ARMOR_CRACKS`
        - `ENTITY_SHADOW` - Pipeline for drawing an entity's shadow.
        - `GLINT_SNIPPET`, `GLINT_SPECIAL_SNIPPET`, `ARMOR_CUTOUT_NO_CULL_GLINT`, `ENTITY_SOLID_GLINT`, `ITEM_CUTOUT_GLINT`, `ITEM_CUTOUT_GLINT_SPECIAL`, `ITEM_TRANSLUCENT_GLINT`, `ITEM_TRANSLUCENT_GLINT_SPECIAL` - Snippets and pipelines for applying the glint decal.
        - `TEXT_BACKGROUND`, `TEXT_BACKGROUND_SEE_THROUGH` are removed
        - `LINES_TRANSLUCENT` replaced by `LINES_TRANSLUCENT_NO_DEPTH_WRITE`
            - `LINES_TRANSLUCENT` now uses the default depth stenctil state
        - `WEATHER_DEPTH_WRITE`, `WEATHER_NO_DEPTH_WRITE` replaced by `WEATHER`
        - `BLIT_DEPTH_BOUNDS`, `BLIT_DEPTH_DURING_DEPTH_BOUNDS`, `BLIT_DEPTH`, `INTEGRATE_DEPTH` - Pipelines for handling depth bounds during order-independent transparency.
        - `getStaticPipelines` -> `requiredPipelines`
        - `optionalPipelines` - Pipelines that are optionally used by the game.
    - `ScreenEffectRenderer` now takes in the `GameRenderer` instead of the `Minecraft` instance
        - `ITEM_ACTIVATION_ANIMATION_LENGTH` is removed
        - `submit` no longer takes in the `boolean` for first person and sleeping, instead taking in the `PlayerRenderState` and `CameraRenderState`
        - `tick` is removed
        - `resetItemActivation` -> `LocalPlayer#resetItemActivation`
        - `displayItemActivation` -> `LocalPlayer#displayItemActivation`
    - `ShaderManager` now implements `PreparableReloadListener` instaed of extending `SimplePreparableReloadListener`
        - `SHADER_INCLUDE_PATH` is now `public` from `private`
        - `SHADER_INCLUDE_EXTENSION`, `SHADER_INCLUDE_CONVERTER` - References to include shaders.
        - `listAllIncludes` - List all include shaders.
        - `isPostEffectValid` - Checks if a post effect is only targetting valid render targets.
        - `getShader`  -> `ShaderSource#getShader`
        - `getAvailablePostEffects` - Returns all known post effects.
    - `Sheets`
        - `ARMOR_TRIMS_SHEET`, `armorTrimsSheet` are removed
        - `*Glint*Sheet` - Glint sheets.
    - `SkyRenderer`
        - `extractRenderState`
        - `renderSkyDisc` -> `render`
        - `renderSunriseAndSunset` is now `private` from `public`
        - `renderEndSky` is now `private` from `public`
        - `renderEndFlash` is now `private` from `public`
    - `SpriteCoordinateExpander` is now a record
        - The constructor takes in a `UvMapping` instead of a `TextureAtlasSprite`
    - `StagedVertexBuffer`
        - `requestIndexCount` - Increases the maximum size of the index buffer to hold the entire draw.
        - `$ExecuteInfo#indexBuffer` -> `customIndexBuffer`
            - `indexBuffer` calls to `customIndexBuffer` before replicating default behavior
    - `SubmitNodeCollection`
        - `seeThroughNameTags` merged into `seeThrough`
        - `gizmos` is removed
            - Submits solid gizmos to `solid`, and tranlucent gizmos to `translucentGizmos`
        - `alwaysOnTop` -> `alwaysOnTopGizmos`
        - `oitTranslucent` - Order-independent transparency storage.
    - `SubmitNodeStorage`
        - `setUseImprovedTransparency` - Toggles whether to use order-independent transparency.
        - `seeThrough` - The translucent features that can be seen through.
    - `WeatherEffectRenderer`
        - `render` -> `prepare`
            - `render` has been split off to only handle the rendering of the weather
        - `renderOit` - Renders with order-independent transparency.
    - `WorldBorderRenderer`
        - `render` -> `prepare`
            - `render` has been split off to only handle the rendering of the world border
        - `renderOit` - Renders with order-independent transparency.
- `net.minecraft.client.renderer.blockentity`
    - `AbstractEndPortalRenderer#submitSpecial` now takes in the outline color
    - `BannerRenderer#submitPatterns` no longer takes in the `ModelFeatureRenderer$CrumblingOverlay`
    - `SkullBlockRenderer#getPlayerSkinRenderTypeCutout` - Gets the entity cutout `RenderType` for the given texture.
- `net.minecraft.client.renderer.chunk`
    - `ChunkSectionLayer#pipeline` now takes in a `boolean` for whether to use a pipeline with multidraw calls.
    - `ChunkSectionLayerGroup#outputTarget` is removed
    - `ChunkSectionsToRender` is now an abstract `class` from a `record`
        - `renderGroup` now takes in the `GpuTextureView` atlas and whether to render as a wireframe `boolean`
        - `render`, `renderOit` - Renders the chunk sections.
        - `$DrawIndirect` - Renders the chunk sections using indirect draw calls.
        - `$DrawSeparate` - Renders the chunk sections using normal draw calls.
        - `$GpuMultiDrawIndexedIndirect` - A record that contains the buffer data for indirect multidraw calls.
    - `CompiledSectionMesh` now takes in a `long` for the nanosecond start time for the compile task
    - `SectionMesh#getCompileTaskStartTime` - When the mesh had started compiling.
    - `SectionRenderDispatcher$RenderSection`
        - `getVisibility` now takes in the `long` fade duration
        - `setFadeDuration` is removed
        - `setWasPreviouslyEmpty`, `wasPreviouslyEmpty` are removed
        - `updateUploadTime` - Updates the upload time to the current time in milliseconds if not set.
- `net.minecraft.client.renderer.culling.Frustum#getNearPlaneBounds` - Gets the bounding box for the near plane.
- `net.minecraft.client.renderer.entity`
    - `CushionRenderer` - Renderer for a cushion.
    - `EntityRenderDispatcher` no longer takes in the `Minecraft` instance, instead taking in the `PalettedTextureManager`
        - `getPlayerRenderer` -> `getRenderer`, taking in the `AvatarRenderState` instead of the `AbstractClientPlayer`
        - `shouldRender` now takes in the partial tick
        - `getItemInHandRenderer` is removed
    - `EntityRenderer`
        - `shouldRender` now takes in the partial tick
        - `getBoundingBoxForCulling` now takes in the partial tick
    - `EntityRendererProvider$Context` now takes in the `PalettedTextureManager`
    - `HumanoidMobRenderer#usesSpearPose` - Returns whether the entity is swinging a spear.
    - `SulfurCubeRenderer#CUSTOM_HEAD_TRANSFORMS` - Transforms when wearing a head.
- `net.minecraft.client.renderer.entity.layers`
    - `CustomHeadLayer`
        - `SKULL_SCALE` is now `public` from `private`
        - `$Transforms` now takes in the `float` vertical scale and a function for resolving the player skin    
            - `TRANSLUCENT_PLAYER_SKIN_RESOLVER`, `CUTOUT_PLAYER_SKIN_RESOLVER` - Resolvers for rendering a player's skin.
    - `EquipmentLayerRenderer` now takes in a `PalettedTextureManager` instead of a `TextureAtlas`
- `net.minecraft.client.renderer.entity.state`
    - `ArmedEntityRenderState`
        - `attackArm`, `swingAnimationType` merged into `currentSwing`
        - `attackTime` -> `swingAnimation`
        - `getArmPose` - Gets the pose for the given arm.
    - `CushionRenderState` - The render state for a cushion.
    - `IllagerRenderState#attackAnim` is removed
- `net.minecraft.client.renderer.extract`
    - `LevelExtractor`
        - `isEntityVisible` now takes in the `float` partial tick and the `long` chunk fade duration
        - `queueTransientBlock` - Adds a transient block for submission.
    - `TransientBlock` - A block that exists at a position for only a certain amount of time, by default a millisecond.
- `net.minecraft.client.renderer.feature`
    - `FeatureRenderDispatcher`
        - `renderAllFeatures` is now `static`, taking in the `RenderPass` and the `$PreparedFrame` instead of the `SubmitNodeStorage`
        - `$PreparedFrame#execute*` methods now take in the `RenderPass`
            - `executeWaterMask`, `hasAnyWaterMask` - Handles any water masks.
            - `executeOit` - Renders any order-independent transparency.
            - `executeSeeThrough`, `hasAnySeeThrough` - Handles any see through objects.
            - `hasAnyAlwaysOnTop` is removed
            - `isEmpty` - Checks if there's nothing submitted for rendering.
        - `$PreparedGroup#execute` now takes in the `OitStage` and the `RenderPass`
    - `FeatureRenderer#executeGroup` now takes in the `OitStage` and the `RenderPass`
    - `ItemFeatureRenderer`
        - `SPECIAL_FOIL_TEXTURE_SCALE` is now `public` from `private`
        - `$Submit#hasTranslucency` is removed
    - `ModelFeatureRenderer$Submit` now takes in a `UvMapping` instead of the `TextureAtlasSprite`
    - `MovingBlockFeatureRenderer$Submit` now takes in a `boolean` of whether to force translucency
    - `NameTagFeatureRenderer` class is removed
    - `TextFeatureRenderer`
        - `$Content` - Represents a specific piece of text or effect to render.
        - `$Submit` now implements `TranslucentSubmit`
            - `x`, `y`, `string`, `dropShadow`, `color`, `backgroundColor`, `outlineColor` has been merged into `content`
- `net.minecraft.client.renderer.fog.environment.FogEnvironment#getBaseColor` now returns a `Vector3fc` instead of an `int`
- `net.minecraft.client.renderer.gizmos.DrawableGizmoPrimitives#isEmpty` - Returns if there are no gizmos submitted for rendering.
- `net.minecraft.client.renderer.item.ItemStackRenderState$LayerRenderState#prepareQuadList` replaced by `setQuads`
- `net.minecraft.client.renderer.oit`
    - `OitPipelineSet` - A set of pipelines for rendering using order-independent transparency.
    - `OitRenderPassProvider` - A provider for creating the `RenderPass`es for order-independent transparency.
    - `OitStage` - The stages involved for rendering with order-independent transparency.
- `net.minecraft.client.renderer.rendertype`
    - `OutputTarget` class is removed
    - `PreparedRenderType` now takes in a `String` name and a `OitPipelineSet` instead of an `OutputTarget`
        - `drawFromBuffer(StagedVertexBuffer$ExecuteInfo)` now takes in a `RenderPass`
        - `drawFromBuffer(GpuBuffer...)` is removed
        - `drawFromBufferOit` - Draws the render type using order-independent transparency.
    - `RenderSetup$RenderSetupBuilder`
        - `setOutputTarget` is removed
        - `setOutline` now has an overload that takes in the `String` texture name
        - `setOitPipelines` - Sets the pipelines that render via order-independent transparency.
        - `withForcedSolidModelPhase` - Forces a non-solid model to render as a solid model.
    - `RenderType`
        - `outputTarget` is removed
        - `forceSolidModelPhase` - Whether to force a non-solid model to render as a solid model.
    - `RenderTypes`
        - `createArmorDecalCutoutNoCull` split into `armorCutoutNoCullGlint`, `armorTrim`
        - `armorTranslucent` replaced by `wolfArmorCracks`
        - `*Glint*` - Glint overlay render types.
        - `entityTranslucentCullItemTarget` is removed
        - `entityTranslucentCull` - Entity rendering with translucency and culling.
        - `entityTranslucentEmissive(Identifier, boolean)` is removed
        - `armorEntityGlint`, `glintTranslucent`, `glint`, `entityGlint` are removed
        - `textBackground`, `textBackgroundSeeThrough` are removed
        - `linesTranslucentNoDepthWrite`, `linesDepthBias` - Line render types.
- `net.minecraft.client.renderer.state.GameRenderState`
    - `requestedPostEffects` - Post effects used.
    - `shouldRenderLevel` - Whether the level should be rendered.
    - `readyForLevelRendering` - If the level is ready to be rendered.
    - `useShaderTransparency` is removed
- `net.minecraft.client.renderer.state.gui.pip`
    - `GuiProfilerChartRenderState` now takes in a `Matrix3x2fc` pose
    - `PictureInPictureRenderState#getBounds` now has an overload that takes in the `Matrix3x2fc` pose
- `net.minecraft.client.renderer.state.level`
    - `CameraRenderState`
        - `isFirstPerson` - If the camera is showing from a first person perspective.
        - `cameraEntityPartialTicks` - The partial tick of the camera entity.
    - `FirstPersonHandsAndItemsRenderState` - The first person render state for held items.
    - `LevelRenderState`
        - `playerRenderState` - The render state of the client player.
        - `worldPartialTicks` - The partial tick of the world
        - `renderWireframeTerrain` - Whether the terrain should be rendered as a wireframe.
        - `shouldUseMultiDrawIndirectForTerrain` - Whether indirect multidraw calls should be used to render the terrain.
    - `PlayerRenderState` - The render state of the client player.
    - `SkyRenderState#sunriseAndSunsetColor`, `skyColor` are now `Vector4fc`s instead of `int`s
    - `TransientBlockRenderState` - The render state of a moving block that lives for a certain amount of time.
- `net.minecraft.client.renderer.texture`
    - `DynamicAtlasTree` - A tree for dynamically allocating texture space within a specific bounds.
    - `DynamicAtlasTreeSlot` - A node of the `DynamicAtlasTree` that represents a slot to take.
    - `MipmappedTexture` - A reloadable texture that contains all mipmaps up to the specified level.
    - `ReloadableTexture#setSampler` - Sets the sampler based on its contents.
    - `TextureAtlasSprite` now implements `UvMapping`
        - `wrap` -> `UvMapping#wrap`
    - `TextureManager#INTENTIONAL_MISSING_TEXTURE` is removed
    - `UvMapping` - An interface that represents that an object can be mapped to a UV coordinate.
- `net.minecraft.client.resources.metadata.texture.PaletteMetadataSection` - Palette metadata for a texture.
- `net.minecraft.client.resources.model`
    - `EquipmentClientInfo` now takes in a list of `$TrimOverride`s
        - `$Builder#replaceTrimPalette` - Replaced the palette of a specific trim material with the given identifier.
        - `$TrimOverride` - An override to replace one `TrimMaterial` palette with another.
        - `$TrimPredicate` - The predicate to match to replace a `TrimMaterial`'s palette.
    - `ModelBakery`
        - `DESTROY_STAGE_COUNT`, `DESTROY_STAGES`, `BREAKING_LOCATIONS` are now `private` from `public`
        - `DESTROY_TYPES_OIT` - The destroy types used during order-independent transparency pipelines.
- `net.minecraft.client.resources.model.cuboid`
    - `CuboidModelElement` now takes in a nullable `Direction` for the shade instead of a `boolean`
    - `FaceBakery#bakeQuad` now takes in a nullable `Direction` for the shade instead of a `boolean`
- `net.minecraft.client.resources.model.geometry`
    - `BakedQuad$MaterialInfo` now takes in a nullable `Direction` for the shade instead of a `boolean`, along with `RenderType`s for the item glint and special item glint
        - `of` now takes in a nullable `Direction` for the shade instead of a `boolean`
    - `ItemQuads` - The baked quads of an item broken into solid and translucent groups.
- `net.minecraft.client.resources.model.sprite`
    - `Material#withSuffix` - Suffixes the sprite of the material.
    - `MaterialBaker` is no longer abstract
        - The constructor now takes in `SpriteLoader$Preparations` for the block and item atlas, and now the missing `TextureAtlasSprite`
        - `replacementForMissingMaterial` is now `private` from `public`
        - `bake`, `bakeForAtlas` are now `private` from `public`
- `net.minecraft.client.resources.palette`
    - `Palette` - A texture represented as an array of `int`s.
    - `PalettedTextureManager` - A texture manager to handle the permutations of a texture for each palette.
    - `PaletteMapping` - A mapping of colors in one `Palette` to another.
    - `PaletteMappingCache` - A cache containing the palette mappings, droping any unused entries after five minutes.
- `net.minecraft.data.AtlasIds#ARMOR_TRIMS` is removed
- `net.minecraft.network.protocol.game`
    - `ClientboundAddTransientBlockPacket` - Sends a block to the client that exists for a finite amount of time.
    - `ClientGamePacketListener#handleAddTransientBlockPacket` - Handles the transient block packet.
- `net.minecraft.server.level.ServerPlayer#sendPostEffects`, `addPostEffect`, `clearPostEffects`, `getPostEffects`, `removePostEffect` - Handles the post effects applied on the player.
- `net.minecraft.util.PngInfo` now takes in a `byte` for the bit depth and color type
- `net.minecraft.world.entity.player.Player#postEffects` - The post effects currently applied to the player.
- `net.minecraft.world.item.equipment.trim`
    - `ArmorTrim#layerAssetId` is removed
    - `MaterialAssetGroup` record is removed
    - `TrimMaterial` now takes in an `Identifier` representing the palette id instead of a `MaterialAssetGroup`
    - `TrimMaterials$Palette` - Identifiers for trim palettes.
- `net.minecraft.world.level.block.SignBlock#openTextEdit` now takes in a `SignTextSlot` instead of a `boolean` for if it's front text
- `net.minecraft.world.level.block.entity`
    - `SignBlockEntity`
        - `createDefaultSignText` is removed
        - `isFacingFrontText` -> `getSlotPlayerIsFacing`, returning the `SignTextSlot`
        - `getText` now takes in a `SignTextSlot`
        - `getFrontText`, `getBackText` are removed
        - `updateSignText`, `updateText` now take in a `SignTextSlot` instead of a `boolean` for if it's front text
        - `setText` is removed
        - `canExecuteClickCommands`, `executeClickCommandsIfPresent` now take in a `SignTextSlot` instead of a `boolean` for if it's front text
    - `SignText` now takes in lists of `Component` messages instead of arrays
        - `DIRECT_CODEC` -> `CODEC`
        - `STREAM_CODEC` - The network codec.
        - `FRONT_TEXT`, `BACK_TEXT` - Tooltip providers for the text on a given side of the sign.
        - `EMPTY` - A sign with no text.
        - `setHasGlowingText` -> `withGlowingText`
        - `setColor` -> `withColor`
        - `setMessage` methods are removed
        - `hasMessage` now takes in a `boolean` filter instead of the `Player`
        - `getMessages` now returns a list of `Component`s instead of an array
        - `hasAnyClickCommands` now takes in a `boolean` filter instead of the `Player`
        - `hasEditableText` - Whether the sign has text that can still be editted.
        - `asMutable`, `$Mutable` - A mutable version of the sign components.
        - `createTooltip` - Creates the getter for the sign text tooltips.
    - `SignTextSlot` - The location of the text on the sign.

## Registries and Data Components

Many different registries and data components have been rewritten in a variety of different methods. As they are so intertwined, they have combine them all into one mega section, similarly to the usual client updates.

### Reloadable Datapack Registries

A new layer has been introduced into the datapack registry process that allows for some datapack registries to be marked as reloadable (can be reloaded via `/reload` command). As such, `LootTable`s, `LootItemCondition`s, `ContextFloatProvider`s, `ContextIntProvider`s, `LootItemFunction`s, `SlotSource`s, `Advancement`s, and `Recipe`s are now datapack registries. Adding a new reloadable datapack registry requires adding to the `RegistryDataLoader#RELOADABLE_REGISTRIES` list.

With this change comes many other changes, especially for data generation and implementation details, which have their own sections below. However, the general gist is that most `ResourceKey` or direct value references have been replaced with either a `Holder`-wrapped, or `HolderSet`-wrapped registry object. Which is chosen depends entirely on the specific use case.

### Registry Bootstrap "Providers"

Since data generation for datapack registry entries are handled through `RegistriesDatapackGenerator`, reloadable registries now no longer have their own `DataProvider`s. Instead, the provider classes now implements either `SingleRegistryBootstrap` or `MultiRegistryBootstrap` to 'provide' their entries to `RegistriesDatapackGenerator#forReloadableLayer`.

`SingleRegistryBootstrap`, previously `RegistrySetBuilder$RegistryBootstrap`, is the familiar consumer that takes in a `BootstrapContext` to register entries to. Both `LootTableProvider` and `AdvancementProvider` now implement `SingleRegistryBootstrap`, setting the generic to the registry object type.

This means that their underlying implementations are completely different.

`AdvancementProvider` now takes in a list of `AdvancementSubProvider$Factory`s, which construct the `AdvancementSubProvider` with the `BootstrapContext` supplied to the main provider. `AdvancementSubProvider` is now an abstract class as well, taking in the `BootstrapContext` to be used as part of the no argument `generate` method. If another advancement needs to be referenced for a parent, only the `Identifier` is now required.

```java
public class ExampleAdvancementSubProvider extends AdvancementSubProvider {

    // The constructor with the bootstrap context
    public ExampleAdvancementSubProvider(BootstrapContext<Advancement> output) {
        super(output);
    }

    @Override
    public void generate() {
        // The output can be obtained via `this.output`.
        // Additionally, `AdvancementSubProvider` already provides fielded
        // access to the damage types registry via `this.damageTypes`.
        Holder.Reference<Advancement> example = Advancement.Builder.advancement()
            // Set a parent to another advancement.
            .parent(Identifier.withDefaultNamespace("adventure/sleep_in_bed"))
            // Add whatever other information.
            // Register the advancement for generation.
            .save(this.output, "examplemod:example_advancement");
    }
}

// For some RegistrySetBuilder registry passed to a `RegistriesDatapackGenerator`:
registry.add(
    Registries.ADVANCEMENT,
    // The advancement provider to register.
    new AdvancementProvider(
        // The list of sub providers to generate.
        List.of(
            ExampleAdvancementSubProvider::new
        )
    )
);
```

`LootTableProvider` and its associated `LootTableSubProvider` are in a slightly different situation. Most of the implementation logic is quite similar in nature, with the main difference being some minor changes to the underlying interfaces. If you are not implementing your own custom provider methods, then you will likely only have to change the constructor and provider addition.

If you are, here are the main points. `LootTableProvider$SubProviderEntry` now takes in a `LootTableSubProvider$Factory` similar to the advancement provider, except that this factory takes in a `LootTableSubProvider$Context`. This is a wrapper around the `BootstrapContext<LootTable>` meant to set the random sequence and param set for you. Meanwhile, `LootTableSubProvider` replaces `generate` with a no argument `run`. Instead you are supposed to store the `LootTableSubProvider$Context` and call` accept`, similar to how the output consumer from `generate` was handled.

```java
public class ExampleLootSubProvider implements LootTableSubProvider {
    protected final LootTableSubProvider.Context output;

    public ExampleLootSubProvider(LootTableSubProvider.Context output) {
        // Store the output
        this.output = output;
    }

    @Override
    public void run() {
        // Write the loot table.
        this.output.accept(
            // The registry key for the table.
            ResourceKey.create(Registries.LOOT_TABLE, Identifier.fromNamespaceAndPath("examplemod", "example_table")),
            // A builder for the loot table.
            // Call whatever methods desired to add pools, conditions, etc.
            LootTable.lootTable()
        );
    }
}
```

`BlockLootSubProvider` and `EntityLootSubProvider` are similar, except they take in the `LootTableSubProvider$Context` instead of the `HolderLookup$Provider`. They each provide the common `HolderGetter`s that is used to get the objects. The main difference is that some elements replace the `LootItemCondition$Builder` with a `Holder<LootItemCondition>` depending on if the condition is now datapack registered instead of inlined. But otherwise, they are functionally the same.

```java
public class ExampleBlockLoot extends BlockLootSubProvider {

    public ExampleBlockLoot(LootTableSubProvider.Context output) {
        super(
            // Items that are explosion resistant.
            Set.of(),
            // The feature flags to check for generation.
            FeatureFlags.REGISTRY.allFlags(),
            // The passed in context.
            output
        );
    }

    @Override
    protected void generate() {
        // More or less same as prior versions.
    }
}
```

Then, all you need to do is add them to the provider with the desired `ContextKeySet`:

```java
// For some RegistrySetBuilder registry passed to a `RegistriesDatapackGenerator`:
registry.add(
    Registries.LOOT_TABLE,
    // The loot table provider to register.
    new LootTableProvider(
        // The built in loot tables to check for.
        Set.of()
        // The list of sub providers to generate.
        List.of(
            new LootTableProvider.SubProviderEntry(ExampleLootSubProvider::new, LootContextParamSets.SELECTOR),
            new LootTableProvider.SubProviderEntry(ExampleBlockLoot::new, LootContextParamSets.BLOCK)
        )
    )
);
```

One caveat is that validation is no longer handled in the provider itself. Rather it is checked when constructing the reloadable lookup through `VanillaRegistries#validateLootData`. Depending on the circularness of your registry entries (e.g. referring to other loot tables in loot tables), you may want to perform a similar validation, which will likely have to be wrapped or injected in somehow.

`MultiRegistryBootstrap` handles generating registry entries for multiple registries in the same provider. The registries it generates for are requested in `requestedRegistries` via their `ResourceKey`s. Then `run` is used to get the `BootstrapContext`s required for registration.

This is currently only used by `RecipeProvider` since it generates both recipes and their associated advancements at the same time. Given the bootstrap nature, `RecipeProvider$Runner` is removed. Instead, we create our own `MultiRegistryBootstrap` to handle the recipe building.

`RecipeProvider` now takes in the `BoostrapContext`s for the recipe and advancement registries. The `RecipeOutput` is created within the construtor, acting as our `BootstrapContextAccess` to get any other required registries. Actually generating the recipes is more or less the same: override `buildRecipes` and call `RecipeOutput#accept` with the recipe to register, or more commonly using `RecipeBuilder#save`. The `MultiRegistryBootstrap` can be created as an anonymous class where `RecipeProvider#buildRecipes` is called within `run`:

```java
public class ExampleRecipeProvider extends RecipeProvider {

    // The constructors with the desired outputs.
    public ExampleRecipeProvider(BootstrapContext<Recipe<?>> recipeOutput, BootstrapContext<Advancement> advancementOutput) {
        super(recipeOutput, advancementOutput);
    }

    @Override
    protected void buildRecipes() {
        // Call whatever methods to generate the recipes.
    }

    // ...

    // Construct the registry bootstrap.
    public static MultiRegistryBootstrap create() {
        return new MultiRegistryBootstrap() {
            @Override
            public Set<ResourceKey<? extends Registry<?>>> requestedRegistries() {
                // Return the registries we are adding entries to.
                return Set.of(Registries.RECIPE, Registries.ADVANCEMENT);
            }

            @Override
            public void run(MultiRegistryBootstrap.BootstrapGetter registries) {
                // Run the recipe provider.
                new ExampleRecipeProvider(registries.get(Registries.RECIPE), registries.get(Registries.ADVANCEMENT)).buildRecipes();
            }
        };
    }
}

// For some RegistrySetBuilder registry passed to a `RegistriesDatapackGenerator`:
registry.add(
    Registries.RECIPE,
    // The bootstrap to register.
    ExampleRecipeProvider.create()
);
```

### Removal of `ContextAwarePredicate`

`ContextAwarePredicate` was originally a wrapper around a list of `LootItemCondition`s for use in advancement criteria triggers. Now, `ContextAwarePredicate` has been completely removed, replaced by the `Holder<LootItemCondition>`:

```java
// For some trigger implementation
public record ExampleTriggerInstance(Optional<Holder<LootItemCondition>> player) implements SimpleCriterionTrigger.SimpleInstance {
    // ...
}
```

### Registering `ContextKeySet`s for Some Reason

With the transition to reloadable registries, A new registry has been added for `ContextKeySet`s used by the loot table. These are still defined in `LootContextParamSets`, meaning that the behavior remains almost identical. However, it now validates that all `ContextKey`s (e.g. the ones defined in `LootContextParams`) are added as required keys within `LootContextParamSets#ALL_PARAMS`.

While this makes it more difficult to add custom `ContextKey`s within a new loot `ContextKeySet`, this validation only occurs once when `Bootstrap#bootStrap` is called. Still, due to validation, it is highly recommended to both inject into `LootContextParamSets#ALL_PARAMS` your custom `ContextKey` before registering your specific loot `ContextKeySet`.


### New Loot Data Types

With the new reloadable registries also comes associated `LootDataType`s for `SlotSource` (`LootDataType#SLOT_SOURCE`), `ContextFloatProvider` (`LootDataType#FLOAT_PROVIDER`), and `ContextIntProvider` (`LootDataType#INT_PROVIDER`).

As a refresher, `LootDataType`s are typically used for validation and context tracking. All loot data is validated after registration, checking for any non-obvious issues like recursion or parameters not specified in the used `ContextKeySet`. Then, when generating the loot, elements are tracked by visitation using `LootContext#pushVisitedElement` and `LootContext#popVisitedElement` to ensure that an element is only visited once and is not recursing on itself.

Of these, all three new data types validate against `LootContextParamSets#ALL_PARAMS`. However, only `LootDataType#SLOT_SOURCE` is tracked for visitation when running the `/item` commands. 

### Reorganizing Pool Containers

The underlying structure of `LootPoolEntryContainer`s has been slightly reorganized to separate the difference between a single entry vs one that provides a set of entries.

For this, `LootPoolSingletonContainer` was split into two classes: `UniformContainerBase` which handles the entry(s) and their weight(s), and its subclass `SingleEntryContainerBase` which indicates that the container only provides a single entry. `LootItem`, `DynamicLoot`, `SlotLoot`, and `EmptyLootItem` all extend `SingleEntryContainerBase`, with its methods being pretty much identical to `LootPoolSingletonContainer`, aside from the reloadable registry changes.

For `TagEntry` and `NestedLootTable`, a new `UniformContainerBase` subtype was added called `ExpandableContainerBase`, which, if `expand` is `true`, allows an entry to be treated as a list of entries of one item each to select from, rather than one entry that provides all items:

```java
// A basic example showing the implementation difference.
public class ExampleExpandableEntry extends ExpandableContainerBase {
    // The map codec to register
    public static final MapCodec<ExampleExpandableEntry> MAP_CODEC = RecordCodecBuilder.mapCodec(instance ->
        instance.group(
            RegistryCodecs.holderSet(Registries.ITEM).fieldOf("items").forGetter(e -> e.tag)
        ).and(expandableFields(instance))
        .apply(instance, ExampleExpandableEntry::new)
    );
    private final HolderSet<Item> items;

    // The last five parameters are required for each expandable
    // container.
    public ExampleExpandableEntry(HolderSet<Item> items, boolean expand, int weight, int quality, Optional<Holder<LootItemCondition>> condition, Optional<Holder<LootItemFunction>> modifier) {
        super(expand, weight, quality, condition, modifier);
        this.items = items;
    }

    @Override
    public MapCodec<ExampleExpandableEntry> codec() {
        return MAP_CODEC;
    }

    @Override
    protected boolean addExpandedEntries(Consumer<LootPoolEntry> output) {
        // Adds as a separate entry per item.
        // As separate entries, they can each be rolled individually, applying with
        // the weight, conditions, and modifiers for each.
        this.items.forEach(item -> output.accept(new UniformContainerBase.EntryBase() {
            @Override
            public void createItemStack(Consumer<ItemStack> output, LootContext context) {
                output.accept(new ItemStack(item));
            }
        }));

        // Returns whether the entries were added successfully.
        return true;
    }

    @Override
    protected boolean addUnexpandedEntry(Consumer<LootPoolEntry> output) {
        // Adds a single entry for all items.
        // As a single entry, if this is selected, all values inside will be provided.
        output.accept(new UniformContainerBase.EntryBase() {
            @Override
            public void createItemStack(Consumer<ItemStack> output, LootContext context) {
                ExampleExpandableEntry.this.items.forEach(item -> output.accept(new ItemStack(item)));
            }
        });

        // Returns whether the entries was added successfully.
        return true;
    }

    // A basic helper to construct the builder for data generation.
    public static UniformContainerBase.Builder<?> contents(boolean expand, Holder<Item>... items) {
        return simpleBuilder(
            (weight, quality, conditions, functions) -> new ExampleExpandableEntry(
                HolderSet.direct(items), expand, weight, quality, conditions, functions
            )
        );
    }
}
```

### Loot Conditions and Functions: Tweaks and Registrations

With the addition of reloadable registries, the `LootItemCondition`s and `LootItemFunction`s have some minor tweaks to better fit the datapack registry design. As a result, some conditions are referenced registered instead of inlined, like checking whether a tool can silk touch or shear (see `LootPredicates`).

```java
// A basic example.
// For actually registering loot tables, it should be done
// through the `LootTableSubProvider`.
private static void registerTables(BootstrapContext<LootTable> context) {
    LootTable.lootTable().withPool(
        LootPool.lootPool()
            // Reference a registered condition.
            .when(context.lookup(Registries.PREDICATE).getOrThrow(LootPredicates.TOOL_CAN_SILK_TOUCH))
            // ...
    );
}
```

For `LootItemCondition`s, the `ConditionUserBuilder` can now take in a `Holder`-wrapped condition in its `when` clause. When composing a condition within the loot table context, any list of conditions are merged into a single inlined holder via `ConditionUserBuilder#buildCondition`. The actual implementation of the conditions remain exactly the same.

For `LootItemFunction`, the common superclass `LootItemConditionalFunction` also takes in a single optional, `Holder`-wrapped condition instead of a list due to composition changes. Similarly, the `FunctionUserBuilder` can now take in a `Holder`-wrapped function in its `apply` clause. Any list of functions are merged into a single inlined holder via `FunctionUserBuilder#buildFunction`, and as such, `LootItemFunction#decorate` now takes in that optional, `Holder`-wrapped function the modify the dropped outputs. If implementing a custom `LootItemConditionalFunction`, the constructor will need to be updated:

```java
// A basic conditional function.
public class NoOpFunction extends LootItemConditionalFunction {

    // Takes in a single condition that is composed together rather than
    // a list.
    public NoOpFunction(Optional<Holder<LootItemCondition>> condition) {
        super(condition);
    }
}
```

### Registered Slot Sources

With the addition of reloadable registries, `SlotSource`s can now be registered to `Registries#SLOT_SOURCE`. While there is no restriction on where a referenced
slot source can be used, vanilla only references them specifically for the `/item` and `/execute` commands. The rest are inlined within
a loot table, if used at all.

```java
// For some RegistrySetBuilder builder to generate the datapack entries.

// The resource key to register
public static final ResourceKey<SlotSource> EMPTY = ResourceKey.create(
    Registries.SLOT_SOURCE,
    Identifier.fromNamespaceAndPath("examplemod", "empty")
);

builder.add(Registries.SLOT_SOURCE, bootstrap -> {
    bootstrap.register(
        EMPTY,
        // An empty slot source.
        new EmptySlotSource()
    );
});
```

```json5
// In data/examplemod/slot_source/empty.json
{
    // An empty slot source.
    "type": "minecraft:empty"
}
```

Which can be referenced like:

```json5
// For some loot table.
{
    "pools": [
        {
            "rolls": 1,
            "entries": [
                {
                    "type": "minecraft:slots",
                    // Replaces the inlined entry.
                    "slot_source": "examplemod:empty"
                }
                // ...
            ]
        }
        // ...
    ]
}
```

### Splitting Numbers into Floats and Ints

`NumberProvider` which handled retrieving a `float` or `int` given some context, has now been split into two interfaces: `ContextFloatProvider` for `float`s, and `ContextIntProvider` for `int`s. Their implementations are nearly identical, just with the word `float` swapped out for `int`, and vice versa.

Both providers are `Validatable` and must implement `getFloatUnsafe` or `getIntUnsafe`, which returns the value and could potentially through an `ArithmeticException`. If the exception should be ignored, the values can be obtained using `getFloat` or `getInt`, returning `0` if the exception is thrown. Given that floats have non-finite representations, `ContextFloatProvider` also returns `0` if the result is non-finite, or can throw an `ArithmeticException` using `getFloatOrThrow`. To register the provider for use, a `MapCodec` must be created, returned by `codec`, and registered to `BuiltInRegistries#CONTEXT_FLOAT_PROVIDER_TYPE` or `CONTEXT_INT_PROVIDER_TYPE`.

```java
// Some basic number providers.
public record FloatZeroValue() implements ContextFloatProvider {
    public static final FloatZeroValue INSTANCE = new FloatZeroValue();
    // The codec to register.
    public static final MapCodec<FloatZeroValue> MAP_CODEC = MapCodec.unit(INSTANCE);

    @Override
    public MapCodec<FloatZeroValue> codec() {
        // Links the codec to the provider implementation for
        // serialization.
        return MAP_CODEC;
    }

    @Override
    public void validate(ValidationContext context) {
        // If using any loot data type, validate it here.
    }

    @Override
    public float getFloatUnsafe(LootContext random) {
        // Return the value provided.
        return 0f;
    }
}

public record IntZeroValue() implements ContextIntProvider {
    public static final IntZeroValue INSTANCE = new IntZeroValue();
    // The codec to register.
    public static final MapCodec<IntZeroValue> MAP_CODEC = MapCodec.unit(INSTANCE);

    @Override
    public MapCodec<IntZeroValue> codec() {
        // Links the codec to the provider implementation for
        // serialization.
        return MAP_CODEC;
    }

    @Override
    public void validate(ValidationContext context) {
        // If using any loot data type, validate it here.
    }

    @Override
    public int getIntUnsafe(LootContext random) {
        // Return the value provided.
        return 0;
    }
}

// Register the map codecs.
Registry.register(
    BuiltInRegistries.CONTEXT_FLOAT_PROVIDER_TYPE,
    Identifier.fromNamespaceAndPath("examplemod", "float_zero"),
    FloatZeroValue.MAP_CODEC
);
Registry.register(
    BuiltInRegistries.CONTEXT_INT_PROVIDER_TYPE,
    Identifier.fromNamespaceAndPath("examplemod", "int_zero"),
    IntZeroValue.MAP_CODEC
);
```

As reloadable registries, the providers can be registed and used by referenced rather than inlined to `Registries#CONTEXT_FLOAT_PROVIDER` or `CONTEXT_INT_PROVIDER`:

```java
// For some RegistrySetBuilder builder to generate the datapack entries.

// The resource keys to register.
public static final ResourceKey<ContextFloatProvider> FLOAT_ZERO = ResourceKey.create(
    Registries.CONTEXT_FLOAT_PROVIDER,
    Identifier.fromNamespaceAndPath("examplemod", "zero")
);
public static final ResourceKey<ContextIntProvider> INT_ZERO = ResourceKey.create(
    Registries.CONTEXT_INT_PROVIDER,
    Identifier.fromNamespaceAndPath("examplemod", "zero")
);

// Add them to the appropriate provider.
builder.add(Registries.CONTEXT_FLOAT_PROVIDER, bootstrap -> {
    bootstrap.register(
        FLOAT_ZERO,
        // Our float provider.
        new FloatZeroValue()
    );
});
builder.add(Registries.CONTEXT_INT_PROVIDER, bootstrap -> {
    bootstrap.register(
        INT_ZERO,
        // Our int provider.
        new IntZeroValue()
    );
});
```

And the generated JSON:

```json5
// For some float provider.
// In `data/examplemod/worldgen/context_float_provider/zero.json`
{
    // Our float provider.
    "type": "examplemod:zero"
}

// For some int provider.
// In `data/examplemod/worldgen/context_int_provider/zero.json`
{
    // Our int provider.
    "type": "examplemod:zero"
}
```

Due to the many similarities bewteen the types of providers implemented, vanilla also provides some simple interfaces that can be implemented in addition to the provider of your choice. These implement `Validatable#validate` along with providing a way to construct the `MapCodec`. The other methods defined typically match the signature of an associated argument in a record constructor.

```java
// A provider using a basic interface.
// Since we are modifying one value of another provider, we use `UnaryProvider`.
// `input` implemented by the record argument.
public record CubeRoot(Holder<ContextFloatProvider> input) implements ContextFloatProvider, UnaryProvider<ContextFloatProvider> {
    // The codec to register.
    public static final MapCodec<CubeRoot> MAP_CODEC = UnaryProvider.codec(
        // The value codec, or the codec for the generic type.
        ContextFloatProviders.CODEC,
        // A method that takes in the `input` and returns the constructed object.
        CubeRoot::new
    );

    @Override
    public MapCodec<CubeRoot> codec() {
        return MAP_CODEC;
    }

    @Override
    public float getFloatUnsafe(LootContext context) {
        // Perform the operation through the available API methods.
        return (float) Math.cbrt(this.input().value().getFloatUnsafe(context));
    }
}

// Register the map codec.
Registry.register(
    BuiltInRegistries.CONTEXT_FLOAT_PROVIDER_TYPE,
    Identifier.fromNamespaceAndPath("examplemod", "cube_root"),
    CubeRoot.MAP_CODEC
);
```

Finally, `ContextFloatProvider` and `ContextIntProvider` can be used outside the inlined, loot table context, for such things like data components. Given the reloadable nature, vanilla provided `ResolvableFloat` and `ResolvableInt` to hold either a constant or `ResourceKey` for the associated provider to resolve against. You can see examples of this in the other subsections below.

### Block Transformers

`BlockTransformer` is a new datapack component and world datapack registry that is used to turn one block into another, such as when right-clicking with an item. This replaces `AxeItem#STRIPPABLES` with `BlockTransformers#AXE`, `ShovelItem#FLATTENABLES` with `BlockTransformers#SHOVEL`, and `HoeItem#TILLABLES` with `BlockTransformers#HOE`. As this was the last remaining difference compared to a normal `Item`, `AxeItem`, `ShovelItem`, and `HoeItem` have been removed.

`BlockTransformer` takes in a list of `BlockTransformData`, which is checked against to determine whether the transformation can apply. Each `BlockTransformData` takes in the `BlockStateProvider` holder that determines the block to place, the `SoundEvent` holder to play on a successful transformation, the `$TransformParticle` to spawn on success, a list of `Direction`s that prevent the transformation (e.g., shovel flatenning cannot be done on the `Direction#DOWN` face of the block), an optional `LootTable` key for any items to drop on success, the `$DropStrategy` for where the loot should be dropped from, whether to update the newly placed block using the neighbor states (e.g. connecting a fence to other nearby fence posts), a `$TransformType` which is only added for fixing copper chest behavior, whether the held item should be consumed on use, and how much damage to apply to the item when used. Some of these settings are exclusive to one another and are limited in the modded context due to their implementation.

Note that the transformations are entirely controlled by the `BlockStateProvider`, meaning that the block transformer can transform any block into any other block, regardless of condition.

As a world datapack registry, it is highly recommended to generate the entry instead of inlining it. Additionally, for broader support, it is worthile to create a `BlockStateProvider` that can determine its entries using a find first approach:

```java
// A basic block state provider that resolves using a tag.
// With this, new entries can be added to the transformer based on a tag
// rather than overriding the entire transformer.
public class FindFirstStateProvider(@Nullable Holder<BlockStateProvider> fallback, HolderSet<BlockStateProvider> providers) implements BlockStateProvider {
    public static final MapCodec<FindFirstStateProvider> CODEC = RecordCodecBuilder.mapCodec(instance ->
        instance.group(
            BlockStateProvider.CODEC.optionalFieldOf("fallback").forGetter(provider -> Optional.ofNullable(provider.fallback)),
            RegistryCodecs.holderSet(Registries.BLOCK_STATE_PROVIDER, BlockStateProvider.DIRECT_CODEC).fieldOf("providers")
                .forGetter(FindFirstStateProvider::providers)
        ).apply(instance, FindFirstStateProvider::new)
    );

    @Override
    public MapCodec<FindFirstStateProvider> codec() {
        return CODEC;
    }

    @Override
    public BlockState getState(LevelAccessor level, RandomSource random, BlockPos pos) {
        // Redirect to optional for convenience.
        @Nullable
        BlockState result = this.getOptionalState(level, random, pos);
        return result != null ? result : level.getBlockState(pos);
    }

    @Nullable
    @Override
    public BlockState getOptionalState(LevelAccessor level, RandomSource random, BlockPos pos) {
        return this.providers.stream()
            // Get the optional state.
            .map(provider -> provider.value().getOptionalState(level, random, pos))
            // Filter out any null values.
            .filter(Objects::nonNull)
            // Get the first value that matches.
            .findFirst()
            // If all values returned null, evaluate fallback.
            .orElseGet(
                () -> this.fallback == null ? null : this.fallback.value().getOptionalState(level, random, pos)
            );
    }
}

// Register the map codec.
Registry.register(
    BuiltInRegistries.BLOCK_STATE_PROVIDER_TYPE,
    Identifier.fromNamespaceAndPath("examplemod", "find_first"),
    FindFirstStateProvider.CODEC
);
```

As for the transformer itself:

```java
// For some RegistrySetBuilder builder to generate the datapack entries.

public static final TagKey<BlockStateProvider> EXAMPLE_TRANSFORMER_ENTRIES = TagKey.create(
    Registries.BLOCK_TRANSFORMER,
    Identifier.fromNamespaceAndPath("examplemod", "transformer/example_entries")
);

// The resource key to register.
public static final ResourceKey<BlockTransformer> EXAMPLE_TRANSFORMER = ResourceKey.create(
    Registries.BLOCK_TRANSFORMER,
    Identifier.fromNamespaceAndPath("examplemod", "example_transformer")
);

// Add to the appropriate provider.
builder.add(Registries.BLOCK_TRANSFORMER, bootstrap -> {
    bootstrap.register(
        EXAMPLE_TRANSFORMER,
        // Create the transformer.
        new BlockTransformer(
            // The list of data entries to apply.
            List.of(
                // Create the transformer, typically through `BlockTransformData#builder`
                BlockTransformData.builder(
                    // The provider that returns the block to place.
                    new FindFirstStateProvider(
                        null, bootstrap.lookup(Registries.BLOCK_TRANSFORMER).getOrThrow(EXAMPLE_TRANSFORMER_ENTRIES)
                    )
                ).sound(
                    // The sound event to play on a successful transformation.
                    // If your sound event isn't already a holder, it can be wrapped
                    // using `Registry#wrapAsHolder`.
                    // Defaults to the empty sound.
                    BuiltInRegistries.SOUND_EVENT.wrapAsHolder(SoundEvents.AMETHYST_CLUSTER_HIT)
                ).particle(
                    // The particle to show on a successful transformation.
                    // This must be a `$TranformParticle`:
                    // - NONE (No particles)
                    // - SCRAPE (`ParticleTypes#SCRAPE` via level event 3005)
                    // - WAX_ON (`ParticleTypes#WAX_ON` via level event 3003)
                    // - WAX_OFF (`ParticleTypes#WAX_OFF` via level event 3004)
                    // Defaults to NONE.
                    BlockTransformer.TranformParticle.NONE
                ).disallowedFaces(
                    // A list of directions that this transform data does not apply for.
                    // In the item use context, if a direction in this list matches the
                    // clicked face of a block, the transform data is skipped.
                    // Defaults to an empty list.
                    List.of()
                ).loot(
                    // A loot table of any additional items to drop on a successful
                    // transformation. Vanilla uses this to drop hanging roots when
                    // tilling rooted dirt.
                    // If none is present, then no additional loot drops.
                    // Defaults to an empty optional.
                    BuiltInLootTables.END_CITY_TREASURE
                ).dropStrategy(
                    // If a loot table is specified, how the items should drop on a
                    // successful transformation. This is ignored if no loot table
                    // is set.
                    // This must be a `$DropStrategy`:
                    // - CLICKED_FACE (Drops from the clicked face position)
                    // - FROM_MIDDLE (Drops from the middle of the block)
                    // Defaults to FROM_MIDDLE.
                    BlockTransformer.DropStrategy.CLICKED_FACE
                ).updateFromNeighbors(
                    // Whether this block should be updated by its adjacent
                    // neighbors via `BlockState#updateShape`. This should
                    // only be `false` in very particularized situations that
                    // are not normally encountered.
                    // Defaults to `true`.
                    true
                ).transformType(
                    // The type of transform being applied on success. This
                    // is currently only for handling some update differences
                    // for copper chests.
                    // This must be a `$TransformType`:
                    // - SINGLE_BLOCK (A regular old block)
                    // - COPPER_CHEST (A copper chest transformation)
                    // Defaults to SINGLE_BLOCK.
                    BlockTransformer.TransformType.SINGLE_BLOCK
                ).consumeOnUse(
                    // When `true`, the held `ItemStack` is consumed on a successful
                    // transformation. This is ignored if the item is not stackable.
                    false
                ).itemDamagePerUse(
                    // The amount of durability to take away from the item when performing
                    // a successful transformation. This is ignored if the item is stackable.
                    1
                ).build()
            )
            // ...
        )
    );
});
```

And the generated JSON:

```json5
// For some block transformer.
// In `data/examplemod/block_transformer/example_transformer.json`
// The list of data entries to apply.
[
    {
        // The provider that returns the block to place.
        "block_state_provider": {
            "type": "examplemod:find_first",
            "providers": "#examplemod:transformer/example_entries"
        },
        // The sound event to play on a successful transformation.
        // Defaults to the empty sound.
        "sound": "minecraft:block.amethyst_cluster.hit",
        // The particle to show on a successful transformation.
        // Either:
        // - none (No particles)
        // - scrape (`ParticleTypes#SCRAPE` via level event 3005)
        // - wax_on (`ParticleTypes#WAX_ON` via level event 3003)
        // - wax_off (`ParticleTypes#WAX_OFF` via level event 3004)
        // Defaults to none.
        "particle": "none",
        // A list of directions that this transform data does not apply for.
        // In the item use context, if a direction in this list matches the
        // clicked face of a block, the transform data is skipped.
        // Defaults to an empty list.
        "disallowed_faces": [],
        // A loot table of any additional items to drop on a successful
        // transformation. Vanilla uses this to drop hanging roots when
        // tilling rooted dirt.
        // If none is present, then no additional loot drops.
        // Defaults to an empty optional.
        "loot": "minecraft:chests/end_city_treasure",
        // If a loot table is specified, how the items should drop on a
        // successful transformation. This is ignored if no loot table
        // is set.
        // Either:
        // - clicked_face (Drops from the clicked face position)
        // - from_middle (Drops from the middle of the block)
        // Defaults to from_middle.
        "drop_strategy": "clicked_face",
        // Whether this block should be updated by its adjacent
        // neighbors via `BlockState#updateShape`. This should
        // only be `false` in very particularized situations that
        // are not normally encountered.
        // Defaults to `true`.
        "update_from_neighbors": true,
        // The type of transform being applied on success. This
        // is currently only for handling some update differences
        // for copper chests.
        // Either:
        // - single_block (A regular old block)
        // - copper_chest (A copper chest transformation)
        // Defaults to single_block.
        "transform_type": "single_block",
        // When `true`, the held `ItemStack` is consumed on a successful
        // transformation. This is ignored if the item is not stackable.
        "consume_on_use": false,
        // The amount of durability to take away from the item when performing
        // a successful transformation. This is ignored if the item is stackable.
        "item_damage_per_use": 1
    }
    // ...
]
```

For the item, the component can be added through `Item$Properties#delayedComponent` via `DataComponents#BLOCK_TRANSFORMER`:

```java
// For some `Item`.
new Item(
    new Item.Properties()
        .delayedComponent(DataComponents.BLOCK_TRANSFORMER, context -> context.getOrThrow(EXAMPLE_TRANSFORMER))
);
```

### Datapack Brewing Recipes

The brewing system has been converted to a `Recipe` for the mix and a `DataComponents#BREWING_FUEL` data component for the fuel, replacing `PotionBrewing` and `ItemTags#BREWING_FUEL`.

The `Recipe` implementation is handled by `BrewingRecipe` under `RecipeType#BREWING`. `BrewingRecipe` takes in a `PotionIngredient` for the input slots and the fuel (reagent), and outputs an `ItemStackTemplate`. The length of time it takes for a potion to be brewed remains fixed at twenty seconds. A `PotionIngredient` is an `Ingredient` with an optional `PotionsPredicate` on the `DataComponents#POTION_CONTENTS`. Generating the recipes is handled through `BrewingRecipeBuilder`, where potion mixes are handled through `brewingMix` and container transformers through `brewingContainerTransform`. Unfortunately, both of these methods of handling recipes require you to specify every possible combination. For brewing mixes, this means at least four recipes, one for each vanilla container. And as for the container transforms, its one per every potion in the bottle.

Essentially, if you do not know a potion or container exists, the recipe will no longer work. As such, the more flexible method is to create a subtype of `BrewingRecipe` to handle mixes and container transforms:

```java
// The following examples are for basic potions as vanilla has
// done before in `PotionBrewing`.

// We extend `BrewingRecipe` for these to put our items within
// the relevant `RecipePropertySet`s. If we choose not to extend
// `BrewingRecipe`, we would need to patch in our recipe checks
// where the set is used.

public class BrewingMixRecipe extends BrewingRecipe {
    // The serialization objects.
    public static final MapCodec<BrewingMixRecipe> MIX_MAP_CODEC = RecordCodecBuilder.mapCodec(
        i -> i.group(
                PotionsPredicate.CODEC.fieldOf("input").forGetter(o -> o.input.potions().get()),
                Ingredient.CODEC.fieldOf("reagent").forGetter(o -> o.reagent.ingredient()),
                PotionContents.CODEC.fieldOf("output").forGetter(o -> o.output.components().get(DataComponentMap.EMPTY, DataComponents.POTION_CONTENTS))
            )
            .apply(i, BrewingMixRecipe::new)
    );
    public static final StreamCodec<RegistryFriendlyByteBuf, BrewingMixRecipe> MIX_STREAM_CODEC = StreamCodec.composite(
        PotionsPredicate.STREAM_CODEC,
        o -> o.input.potions().get(),
        Ingredient.STREAM_CODEC,
        o -> o.reagent.ingredient(),
        PotionContents.STREAM_CODEC,
        o -> o.output.components().get(DataComponentMap.EMPTY, DataComponents.POTION_CONTENTS),
        BrewingMixRecipe::new
    );
    public static final RecipeSerializer<BrewingMixRecipe> MIX_SERIALIZER = new RecipeSerializer<>(MIX_MAP_CODEC, MIX_STREAM_CODEC);

    // Take in the potion on the item, the ingredient as part of the reagent,
    // and the ouputted contents.
    public BrewingMixRecipe(PotionsPredicate input, Ingredient reagent, PotionContents output) {
        // Provide default values for the super ingredients.
        super(
            new PotionIngredient(Ingredient.of(Items.POTION), Optional.of(input)),
            new PotionIngredient(reagent, Optional.empty()),
            new ItemStackTemplate(
                Items.POTION, DataComponentPatch.builder()
                    .set(DataComponents.POTION_CONTENTS, output)
                    .build()
            )
        );
    }

    @Override
    public boolean matches(BrewingInput brew) {
        // Override matches to properly test the input without looking at the item.
        return this.input.potions().get().matches(brew.input()) && this.reagent.test(brew.reagent());
    }

    @Override
    public ItemStack assemble(BrewingInput brew) {
        // Override assemble to copy the input stack with the new components.
        ItemStack result = brew.input().copyWithCount(1);
        result.applyComponents(this.output().components());
        return result;
    }

    @Override
    public RecipeSerializer<BrewingMixRecipe> getSerializer() {
        // Override to set to our serializer.
        return MIX_SERIALIZER;
    }
}

public class BrewingContainerTransformRecipe extends BrewingRecipe {
    // The serialization objects.
    public static final MapCodec<BrewingContainerTransformRecipe> CONTAINER_MAP_CODEC = RecordCodecBuilder.mapCodec(
        i -> i.group(
                Ingredient.CODEC.fieldOf("input").forGetter(o -> o.input.ingredient()),
                Ingredient.CODEC.fieldOf("reagent").forGetter(o -> o.reagent.ingredient()),
                ItemStackTemplate.CODEC.fieldOf("output").forGetter(o -> o.output)
            )
            .apply(i, BrewingContainerTransformRecipe::new)
    );
    public static final StreamCodec<RegistryFriendlyByteBuf, BrewingContainerTransformRecipe> CONTAINER_STREAM_CODEC = StreamCodec.composite(
        Ingredient.STREAM_CODEC,
        o -> o.input.ingredient(),
        Ingredient.STREAM_CODEC,
        o -> o.reagent.ingredient(),
        ItemStackTemplate.STREAM_CODEC,
        o -> o.output,
        BrewingContainerTransformRecipe::new
    );
    public static final RecipeSerializer<BrewingContainerTransformRecipe> CONTAINER_SERIALIZER = new RecipeSerializer<>(CONTAINER_MAP_CODEC, CONTAINER_STREAM_CODEC);

    // Take in the original container, the ingredient as part of the reagent,
    // and the ouputted contents.
    public BrewingContainerTransformRecipe(Ingredient input, Ingredient reagent, ItemStackTemplate output) {
        // Provide default values for the super ingredients.
        super(
            new PotionIngredient(input, Optional.empty()),
            new PotionIngredient(reagent, Optional.empty()),
            output
        );
    }

    @Override
    public boolean matches(BrewingInput brew) {
        // Override matches to properly test the input without looking at the potions.
        return this.input.ingredient().matches(brew.input()) && this.reagent.test(brew.reagent());
    }

    @Override
    public ItemStack assemble(BrewingInput brew) {
        // Override assemble to create the output stack with the input stack's components.
        return this.output.apply(1, brew.input().getComponentsPatch());
    }

    @Override
    public RecipeSerializer<BrewingContainerTransformRecipe> getSerializer() {
        // Override to set to our serializer.
        return CONTAINER_SERIALIZER;
    }
}

// Register the serializers.
Registry.register(
    BuiltInRegistries.RECIPE_SERIALIZER,
    Identifier.fromNamespaceAndPath("examplemod", "mix"),
    BrewingMixRecipe.MIX_SERIALIZER
);
Registry.register(
    BuiltInRegistries.RECIPE_SERIALIZER,
    Identifier.fromNamespaceAndPath("examplemod", "container_transform"),
    BrewingContainerTransformRecipe.CONTAINER_SERIALIZER
);
```

Then, we can generate the recipes using a custom `RecipeBuilder` or directly:

```java
// In some `RecipeProvider` subtype
@Override
protected void buildRecipes() {
    // A generic mix recipe.
    this.output.accept(
        ResourceKey.create(
            Registries.RECIPE, Identifier.fromNamespaceAndPath("examplemod", "brewing/water_to_thick")
        ),
        new BrewingMixRecipe(
            PotionsPredicate.ofPotion(Potions.WATER),
            Ingreident.of(Items.GLOWSTONE_DUST),
            new PotionContents(Potions.THICK)
        ),
        null
    );

    // A generic container recipe.
    this.output.accept(
        ResourceKey.create(
            Registries.RECIPE, Identifier.fromNamespaceAndPath("examplemod", "brewing/potion_to_splash_potion")
        ),
        new BrewingContainerTransformRecipe(
            Ingredient.of(Items.POTION),
            Ingreident.of(Items.GUNPOWDER),
            new ItemStackTemplate(Items.SPLASH_POTION)
        ),
        null
    );
}
```

And the generated JSONs:

```json5
// In `data/examplemod/recipe/brewing/water_to_thick.json`
{
    "type": "examplemod:mix",
    "input": {
        "potions": "minecraft:water"
    },
    "reagent": "minecraft:glowstone_dust",
    "output": {
        "potion": "minecraft:thick"
    }
}

// In `data/examplemod/recipe/brewing/potion_to_splash_potion.json`
{
    "type": "examplemod:container_transform",
    "input": "minecraft:potion",
    "reagent": "minecraft:gunpowder",
    "output": {
        "id": "minecraft:splash_potion"
    }
}
```

The fuel is handled by `DataComponents#BREWING_FUEL`. The associated `BrewingFuel` takes in a `ResolvableInt` for how any uses the fuel provides, and a `ResolvableFloat` for how much to speed up the brew time. Both resolvable values are either a constant, or a `ContextIntProvider` / `ContextFloatProvider` reference that are resolved against the `LootContextParamSets#CONTAINER_PROCESS` context.

Note that, for brewing stands, the brew time is calculated every tick, meaning that if the speed multiplier changes between burning fuels, the amount of time it takes to brew will also be adjusted.

The component can be added through `Item$Properties#brewingFuel`, or directly through `Item$Properties#component`. Using `brewingFuel` expects a reference to datapack entries.

```java
// For some `Item`.
new Item(
    new Item.Properties()
        .component(DataComponents.BREWING_FUEL, new BrewingFuel(
            // How many times can the fuel be used.
            new ResolvableInt.Constant(600),
            // A scalar of how much faster it takes to brew the input.
            // A value of `1` is the normal default.
            // A value less than `1` makes brewing take longer.
            // A value greater than `1` makes brewing faster.
            new ResolvableFloat.Cosntant(1.0f)
        ))
);
```

It is generally recommended to use datapack-registered providers. Vanilla uses `minecraft:brewing/speed_default` for the common speed multiplier, which is always `1`. As for the fuel uses, vanilla specifies the default uses in `minecraft:brewing/uses_default`, which is always `20`.

```java
// For some RegistrySetBuilder builder to generate the datapack entries.

// The resource key to register
public static final ResourceKey<ContextIntProvider> BREWING_USES_EXAMPLE = ResourceKey.create(
    Registries.CONTEXT_INT_PROVIDER,
    Identifier.fromNamespaceAndPath("examplemod", "brewing/uses_example")
);

builder.add(Registries.CONTEXT_INT_PROVIDER, bootstrap -> {
    bootstrap.register(
        BREWING_USES_EXAMPLE,
        // The number of uses the fuel provides.
        new ConstantValue(10)
    );
});

// For some `Item`.
new Item(
    new Item.Properties()
        .component(DataComponents.BREWING_FUEL, new BrewingFuel(
            // How many times can the fuel be used.
            BREWING_USES_EXAMPLE,
            // A scalar of how much faster it takes to brew the input.
            // A value of `1` is the normal default.
            // A value less than `1` makes brewing take longer.
            // A value greater than `1` makes brewing faster.
            ContextFloatProviders.BREWING_DEFAULT_SPEED_MULTIPLIER
        ))
);
```

For reference, the JSON for our `ContextIntProvider` would look like so:

```json5
// In data/examplemod/context_int_provider/brewing/uses_example.json
10
```

### Cooking Fuel Component

What items can be used as fuel within furnace-like blocks are now specified by `DataComponents#COOKING_FUEL`, replacing `FuelValues`. The associated `CookingFuel` takes in a `ResolvableInt` for how any ticks the fuel should burn for, and a `ResolvableFloat` for how much to speed up the cook time. Both resolvable values are either a constant, or a `ContextIntProvider` / `ContextFloatProvider` reference that are resolved against the `LootContextParamSets#CONTAINER_PROCESS` context.

Note that, for furnace-like blocks, the cook time is calculated every tick, meaning that if the speed multiplier changes between burning fuels, the amount of time it takes to cook will also be adjusted.

The component can be added through `Item$Properties#cookingFuel`, or directly through `Item$Properties#component`. Using `cookingFuel` expects a reference to datapack entries.

```java
// For some `Item`.
new Item(
    new Item.Properties()
        .component(DataComponents.COOKING_FUEL, new CookingFuel(
            // How many ticks the fuel should burn for.
            new ResolvableInt.Constant(600),
            // A scalar of how much faster it takes to cook the input.
            // A value of `1` is the normal default.
            // A value less than `1` makes cooking take longer.
            // A value greater than `1` makes cooking faster.
            new ResolvableFloat.Cosntant(1.0f)
        ))
);
```

It is generally recommended to use datapack-registered providers. Vanilla uses `minecraft:cooking/speed_default` for the common speed multiplier, saying that the fuel cooks inputs twice as fast if within a smoker or blast furnace. As for the fuel time, vanilla typically uses a `minecraft:div` int provider to make the fuel burn twice as fast within a smoker or blast furnace:

```java
// For some RegistrySetBuilder builder to generate the datapack entries.

// The resource key to register
public static final ResourceKey<ContextIntProvider> COOKING_TIME_EXAMPLE = ResourceKey.create(
    Registries.CONTEXT_INT_PROVIDER,
    Identifier.fromNamespaceAndPath("examplemod", "cooking/time_example")
);

builder.add(Registries.CONTEXT_INT_PROVIDER, bootstrap -> {
    bootstrap.register(
        COOKING_TIME_EXAMPLE,
        // Divides the first value by the second value.
        ContextIntProviders.div(
            // The dividend, or in our case, the base number of ticks
            // the fuel should burn for.
            ContextIntProviders.exactly(600),
            // The divisor, or in our case, whether to use fast or normal
            // burn times depending on what block our fuel is within.
            Holder.direct(new ConditionalValue(
                // The predicate for our condition, or in our case,
                // the blocks for which our fuel burns faster.
                context.lookup(Registries.PREDICATE).getOrThrow(LootPredicates.FAST_FURNACE),
                // The value to use if the condition returns true,
                // or in our case, the fast burn divisor.
                context.lookup(Registries.CONTEXT_INT_PROVIDER)
                    .getOrThrow(ContextIntProviders.COOKING_FAST_BURN_TIME_REDUCTION_FACTOR),
                // The value to use if the condition returns false,
                // or in our case, the normal burn divisor.
                context.lookup(Registries.CONTEXT_INT_PROVIDER)
                    .getOrThrow(ContextIntProviders.COOKING_NORMAL_BURN_TIME_REDUCTION_FACTOR)
            ))
        ).value()
    );
});

// For some `Item`.
new Item(
    new Item.Properties()
        .component(DataComponents.COOKING_FUEL, new CookingFuel(
            // How many ticks the fuel should burn for.
            COOKING_TIME_EXAMPLE,
            // A scalar of how much faster it takes to cook the input.
            // A value of `1` is the normal default.
            // A value less than `1` makes cooking take longer.
            // A value greater than `1` makes cooking faster.
            ContextFloatProviders.COOKING_DEFAULT_SPEED_MULTIPLIER
        ))
);
```

For reference, the JSON for our `ContextIntProvider` would look like so:

```json5
// In data/examplemod/context_int_provider/cooking/time_example.json
{
    // Divides the first value by the second value.
    "type": "minecraft:div",
    // The dividend, or in our case, the base number of ticks
    // the fuel should burn for.
    "left": 600,
    // The divisor, or in our case, whether to use fast or normal
    // burn times depending on what block our fuel is within.
    "right": {
        "type": "minecraft:conditional",
        // The predicate for our condition, or in our case,
        // the blocks for which our fuel burns faster.
        "condition": "minecraft:block/fast_cooking",
        // The value to use if the condition returns false,
        // or in our case, the normal burn divisor.
        "on_false": "minecraft:cooking/normal_burn_time_reduction_factor",
        // The value to use if the condition returns true,
        // or in our case, the fast burn divisor.
        "on_true": "minecraft:cooking/fast_burn_time_reduction_factor"
    }
}
```

### Villager Food Component

Villager food is now stored as a data component via `DataComponents#VILLAGER_FOOD`, replacing `Villager#FOOD_POINTS`. The associated `VillagerFood` object only stores one positive `int`, representing the nutrition value of the food. Vanilla typically uses `1` for this value, with bread being the only item that provides `4` nutrition. For reference, villagers want more food if they have less than `12` nutrition, and are considered to have too much food if they have more than `24` nutrition.

The component can be added through `Item$Properties#villagerFood`, or directly through `Item$Properties#component`:

```java
// For some `Item`.
new Item(
    new Item.Properties()
        // How much nutrition the food should give to the villager.
        .villagerFood(1)
);
```

### Compostable Component

What items can be thrown in a composter are now specified by `DataComponents#COMPOSTABLE`, replacing `ComposterBlock#COMPOSTABLES`. The associated `Compostable` takes in a `ResolvableInt` for how many layers should be added when used. The `ResolvableInt` can either be a constant or a `ContextIntProvider` reference that is resolved against the `LootContextParamSets#BLOCK_INTERACT` context.

The component can be added through `Item$Properties#compostable`, or directly through `Item$Properties#component`. Using `compostable` expects a reference to a datapack `ContextIntProvider` entry.

```java
// For some `Item`.
new Item(
    new Item.Properties()
        .component(DataComponents.COMPOSTABLE, new Compostable(
            // How many layers to add to the composter.
            // If the value is less than 0, no layer will be added.
            // The composter level will always be clamped to between
            // 0-7 inclusive.
            new ResolvableInt.Constant(1)
        ))
);
```

It is generally recommended to use datapack-registered providers. Vanilla provides five providers for adding a layer to the composter: `minecraft:compostable/low` for a 30% chance to add a layer, `minecraft:compostable/low_medium` for a 50% chance, `minecraft:compostable/medium` for a 65% chance, `minecraft:compostable/medium_high` for a 85% chance, and `minecraft:compostable/always_add_one` for a 100% chance. However, if a composter is empty (only the vanilla `Blocks#COMPOSTER`), then it will always add a single layer.

```java
// For some `Item`.
new Item(
    new Item.Properties()
        // How many layers to add to the composter.
        // If the value is less than 0, no layer will be added.
        // The composter level will always be clamped to between
        // 0-7 inclusive.
        .compostable(ContextIntProviders.COMPOSTABLE_ALWAYS_ADD_ONE)
);
```

### Hiding Amongst the Mobs: A Data Component

A new data component `DataComponents#MOB_VISIBILITY` has been added that can affect how visible the holding or wearing entity is to other mobs. The associated `MobVisibility` takes in two arguments: a `HolderSet` containing the `EntityType`s who are affected by the visbility change, and a scalar between 0-10 inclusive that determines how much the computed range of the viewing entity should be affected. For the mob visibility to be applied, it must have the `DataComponents#EQUIPPABLE` component with the item in the equipped slot.

As an example, an enderman has a target range of 64 blocks by default. Assuming no other modifiers, an item with a visibility of `0.5` will shrink the detection radius to 32 blocks. Likewise, a visbility of `2` will increase the detection radius to 128 blocks. However, a visibility of `0` will still have a minimum detection radius of 2 blocks, as defined within `TargetingConditions`.

The component can be added through `Item$Properties#loweredMobVisibility`, or directly through `Item$Properties#delayedComponent`. Using `loweredMobVisibility` will create a direct `HolderSet` with a visibility of `0.5`:

```java
// For some `Item`.
new Item(
    new Item.Properties()
        .delayedComponent(DataComponents.MOB_VISIBILITY, context -> new MobVisibility(
            // The entities whose visibility are affected by this item.
            // Can either be an entity type id, such as "minecraft:zombie",
            // or a list of entity type ids, such as ["minecraft:zombie", "minecraft:skeleton", ...],
            // or an entity type tag, such as "#minecraft:zombies".
            // `HolderSet#direct` can be used instead of a tag lookup.
            context.lookup(Registries.ENTITY_TYPE).getOrThrow(EntityTypeTags.ZOMBIES),
            // The scalar that is multiplied to the mob's visibility range.
            // Must be between 0-10 inclusive.
            // A value less than `1` shrinks the visibility range.
            // A value greater than `1` increases the visibility range.
            0.1f
        ))
        // Mob visibility only works with `DataComponents#EQUIPPABLE`.
        // Applied when item is in main hand.
        .equippableUnswappable(EquipmentSlot.MAINHAND)
);
```

### Mob Spawn Settings Environment Attribute

`MobSpawnSettings` have been moved off the `Biome` and are now an `EnvironmentAttribute` instead using `EnvironmentAttributes#NATURAL_MOB_SPAWNS`. As a positional attribute, they can be defined on the `DimensionType`, `Biome`, or within a `Timeline`.

```json5
// Dimension type example.
{
    "attributes": {
        "minecraft:gameplay/natural_mob_spawns": {
            // Data has not changed from previous version.
            "spawn_costs": {
                // ... 
            },
            "spawns_by_category": {
                // ...
            }
        }
    }
    // ...
}

// Biome example.
{
    "attributes": {
        "minecraft:gameplay/natural_mob_spawns": {
            "argument": {
                // Data has not changed from previous version.
                "spawn_costs": {
                    // ... 
                },
                "spawns_by_category": {
                    // ...
                }
            },
            // This is the modifier to use for spawns.
            // The quirks of this modifier are explained below.
            "modifier": "overlay"
        }
    }
    // ...
}

// Timeline example.
{
    "tracks": {
        "minecraft:gameplay/natural_mob_spawns": {
            "keyframes": [
                {
                    "ticks": 0,
                    "value": {
                        // Data has not changed from previous version.
                        "spawn_costs": {
                            // ... 
                        },
                        "spawns_by_category": {
                            // ...
                        }
                    }
                },
                {
                    "ticks": 1000,
                    "value": {
                        // Data has not changed from previous version.
                        "spawn_costs": {
                            // ... 
                        },
                        "spawns_by_category": {
                            // ...
                        }
                    }
                }
            ],
            // This is the modifier to use for spawns.
            // The quirks of this modifier are explained below.
            "modifier": "overlay"
        }
    }
    // ...
}
```

When layering spawn settings, they can either be overridden via `AttributeModifier$OperationId#OVERRIDE` (timeline versions take precedence, followed by biome, then dimension), or overlayed via `OVERLAY`: a new modifier operation added in this version. As the name implies, overlay acts as a layering system, where the data is merged together. If the data uses an object that stores data with some kind of key system, then the latter layer's value will override the former.

The overlay for `MobSpawnSettings` works similarly, though there are a few quirks. To review, `MobSpawnSettings` stores two maps: one for entity spawns, where the key is the `MobCategory` and the value is a `WeightedList` containing the spawned entities; and one for the mop costs, where the key is the `EntityType` and the value is the `MobSpawnSettings$MobSpawnCost`.

Now imagine we are trying to overlay two spawn settings that look like so:

```json5
// The former spawn settings.
{
    "spawns_by_category": {},
    "spawn_costs": {}
}

// The latter spawn settings.
{
    "spawns_by_category": {
        "creature": [
            {
                "type": "minecraft:pig",
                "count": 8,
                "weight": 1
            }
        ]
    },
    "spawn_costs": {}
}
```

First, the overlay modifier checks whether both `spawns_by_category` and `spawn_costs` are empty, like in the former settings. If so, then the other settings will be used, in this case, the latter. The same would be true if the latter was empty, then the former will be picked.

Now, let's update the settings so that neither are completely empty:

```json5
// The former spawn settings.
{
    "spawns_by_category": {
        "creature": [
            {
                "type": "minecraft:sheep",
                "count": 4,
                "weight": 1
            }
        ]
    },
    "spawn_costs": {}
}

// The latter spawn settings.
{
    "spawns_by_category": {
        "creature": [
            {
                "type": "minecraft:pig",
                "count": 8,
                "weight": 1
            }
        ]
    },
    "spawn_costs": {}
}
```

Now, the overlay modifier will check whether both settings have the same keys in `spawns_by_category` and `spawn_costs`. If they do, then the latter settings will be used. In our case, both settings specify the `creature` category and have no keys in `spawn_costs`, meaning the former is discarded and the latter is used. Meaning, our final mob settings will look like so:

```json5
// This is the latter settings.
// The former settings were discarded since they
// both defined the same mob categories and spawn costs.
{
    "spawns_by_category": {
        "creature": [
            {
                "type": "minecraft:pig",
                "count": 8,
                "weight": 1
            }
        ]
    },
    "spawn_costs": {}
}
```

So, let's modify our settings one last time so that they can be merged together for the final step:

```json5
// The former spawn settings.
{
    "spawns_by_category": {
        "creature": [
            {
                "type": "minecraft:sheep",
                "count": 4,
                "weight": 1
            }
        ],
        "ambient": [
            {
                "type": "minecraft:bat",
                "count": 8,
                "weight": 1
            }
        ]
    },
    "spawn_costs": {
        "minecraft:sheep": {
            "charge": 0.0001,
            "energy_budget": 10000
        },
        "minecraft:bat": {
            "charge": 0.0001,
            "energy_budget": 10000
        }
    }
}

// The latter spawn settings.
{
    "spawns_by_category": {
        "creature": [
            {
                "type": "minecraft:pig",
                "count": 8,
                "weight": 1
            }
        ]
    },
    "spawn_costs": {
        "minecraft:pig": {
            "charge": 0.0001,
            "energy_budget": 10000
        },
        "minecraft:bat": {
            "charge": 0.05,
            "energy_budget": 20
        },
    }
}
```

The settings are merged together in two steps. For `spawns_by_category`, it loops through all available `MobCategory`s. If both setting specify the same category, then only the latter is used. Otherwise, it will use whatever category is not empty. As for `spawn_costs`, the former settings are added first, followed by the latter settings, replacing any defined in the former.

That means, for our example, it will look like so:

```json5
// The merged spawn settings.
{
    "spawns_by_category": {
        // Uses the creature data defined by
        // the latter settings, replacing the former.
        "creature": [
            {
                "type": "minecraft:pig",
                "count": 8,
                "weight": 1
            }
        ],
        // Uses the former settings data since
        // the latter didn't define any ambient spawns.
        "ambient": [
            {
                "type": "minecraft:bat",
                "count": 8,
                "weight": 1
            }
        ]
    },
    "spawn_costs": {
        // Bat costs were replaced with the latter.
        "minecraft:bat": {
            "charge": 0.05,
            "energy_budget": 20
        },
        // Defined by the latter.
        "minecraft:pig": {
            "charge": 0.0001,
            "energy_budget": 10000
        },
        // Defined by the former.
        "minecraft:sheep": {
            "charge": 0.0001,
            "energy_budget": 10000
        }
    }
}
```

Note that sheeps no longer spawn, even though their costs still remain. This is a side effect of the overlay method, as it assumes that replacing any category means you intend to define what mobs spawn there altogether, rather than a partial merge.

### The Death of the Unused Block Types

The block type `MapCodec` registry, and subsequently `Block#codec`, have been completely removed. As such, any `codec` overrides or direct `MapCodec`s should be deleted as well:

```diff
// For some block.
public class ExampleBlock extends Block {
-    public static final MapCodec<ExampleBlock> CODEC = BlockBehaviour.simpleCodec(ExampleBlock::new);

    public ExampleBlock(BlockBehaviour.Properties properties) {
        super(properties);
    }

-    @Override
-    public MapCodec<? extends Block> codec() {
-        return CODEC;
-    }

    // ...
}
```

### Simplifying Types to Codecs

`BlockStateProvider`, `StructurePlacement`, and `PlacementModifier` no longer use a wrapping object type to act as their registered instances. Now, the registries directly take in the `MapCodec` used for the serialization and deserialization process. As such, `BlockStateProviderType`, `StructurePlacementType`, and `PlacementModifierType` have been removed. Additionally, `type` is now renamed to `codec`, taking in the registered `MapCodec`.

```java
// The following is an example with `PlacementModifier`s, but can roughly apply to the
// other instances as well.
public record IdentityPlacement() implements PlacementModifier {
    public static final IdentityPlacement INSTANCE = new IdentityPlacement();
    // The map codec used as the registry object.
    public static final MapCodec<IdentityPlacement> MAP_CODEC = MapCodec.unit(INSTANCE);

    // ...

    // Replaces `type`
    @Override
    public MapCodec<IdentityPlacement> codec() {
        // Return the registry object.
        return MAP_CODEC;
    }
}

// Register the map codec to the appropriate registry.
Registry.register(
    BuiltInRegistries.PLACEMENT_MODIFIER_TYPE,
    Identifier.fromNamespaceAndPath("examplemod", "identity"),
    IdentityPlacement.MAP_CODEC
);
```

### Configuring Features without `ConfiguredFeature`s

`Feature`, `ConfiguredFeature`, and `FeatureConfiguration` have been merged together into a single `Feature` interface, similar to most other type and dapatack registry implementations: where a `MapCodec` represents the statically registered type, and the `Feature` interface itself is the world datapack registry. As such, the registries have been renamed to reflect that fact: `BuiltInRegistries#FEATURE` is now `FEATURE_TYPE`, while `Registries#CONFIGURED_FEATURE` (`minecraft:worldgen/configured_feature`) is now `FEATURE` (`minecraft:worldgen/feature`).

The methods on `Feature` are either now defaulted methods or have been moved to the `Feature` subtype that used them (e.g. `AbstractOreFeature`). The three most important that remain are `codec` to attach the feature type to the feature; `place`, which now takes in the `WorldGenLevel`, `ChunkGenerator`, `RandomSource`, and origin `BlockPos`; and `getSubFeatures`, which returns any other features used by the current feature.

```java
// An example feature.
public record PlaceFirstFeature(HolderSet<PlacedFeature> features) implements Feature {
    // The feature type map codec to register.
    public static final MapCodec<PlaceFirstFeature> CODEC = ExtraCodecs.nonEmptyHolderSet(PlacedFeature.LIST_CODEC)
        .fieldOf("features").xmap(PlaceFirstFeature::new, PlaceFirstFeature::features);

    @Override
    public MapCodec<PlaceFirstFeature> codec() {
        // Attach the feature type to the feature.
        return CODEC;
    }

    @Override
    public Stream<Holder<Feature>> getSubFeatures() {
        // Return all additional potential features.
        // Otherwise, this should return an empty stream.
        return this.features.stream().flatMap(f -> f.value().getFeatures());
    }

    @Override
    public boolean place(WorldGenLevel level, ChunkGenerator chunkGenerator, RandomSource random, BlockPos origin) {
        // Place any blocks for the feature.
        // Usually using `setBlock` or `safeSetBlock`.
        // Returns `true` if the feature was placed successfully, or otherwise `false`.

        // Create the placer for the feature.
        FeaturePlacer placer = new FeaturePlacer(level, chunkGenerator);

        for (Holder<PlacedFeature> feature : this.features) {
            if (placer.place(feature.value(), random, origin)) {
                return true;
            }
        }

        return false;
    }
}

// Register the map codec.
Registry.register(
    BuiltInRegistries.FEATURE_TYPE,
    Identifier.fromNamespaceAndPath("examplemod", "place_first"),
    PlaceFirstFeature.CODEC
);
```

Note the `FeaturePlacer` call. This is how `PlacedFeature`s are placed into the world and where all the placement methods like `placeWithBiomeCheck` now live.

The `Feature` can then be registered to the world datapack registry:

```java
// For some RegistrySetBuilder builder to generate the datapack entries.

// The resource key to register.
public static final ResourceKey<Feature> EXAMPLE_FEATURE = ResourceKey.create(
    Registries.FEATURE,
    Identifier.fromNamespaceAndPath("examplemod", "example_feature")
);

builder.add(Registries.FEATURE, bootstrap -> {
    bootstrap.register(
        EXAMPLE_FEATURE,
        // Our feature.
        new PlaceFirstFeature(
            bootstrap.lookup(Registries.FEATURE).getOrThrow(FeatureTags.CAN_SPAWN_FROM_BONE_MEAL)
        )
    );
});
```

And the generated JSON:

```json5
// For some feature.
// In `data/examplemod/worldgen/feature/example_feature.json`
{
    // Our feature.
    "type": "examplemod:place_first",
    "features": "#minecraft:can_spawn_from_bone_meal"
}
```

As for the `PlacedFeature`, it remains the same, including the order of the `PlacementModifier`s; however, `PlacementModifier#getPositions` has been replaced by `modify`. `modify` takes the `BlockPos`s through a `Consumer<BlockPos>` instead of returning a `Stream<BlockPos>`:

```java
// An example placement modifier.
public record IdentityPlacement() implements PlacementModifier {
    public static final IdentityPlacement INSTANCE = new IdentityPlacement();
    // The map codec used as the registry object.
    public static final MapCodec<IdentityPlacement> MAP_CODEC = MapCodec.unit(INSTANCE);

    @Override
    public MapCodec<IdentityPlacement> codec() {
        // Return the registry object.
        return MAP_CODEC;
    }

    @Override
    public void modify(PlacementContext context, RandomSource random, BlockPos origin, Consumer<BlockPos> output) {
        // Like previously, the origin represents either the feature starting point or a position generated from
        // a previous modifier.
        // Make any modifications and pass the position to the consumer output for it to be used by the attached
        // feature.
        output.accept(origin);
    }
}

// Register the map codec.
Registry.register(
    BuiltInRegistries.PLACEMENT_MODIFIER_TYPE,
    Identifier.fromNamespaceAndPath("examplemod", "identity"),
    IdentityPlacement.MAP_CODEC
);
```

### Configured Carvers without `ConfiguredWorldCarver`

`WorldCarver`, `ConfiguredWorldCarver`, and `CarverConfiguration` have been merged together into a single `WorldCarver` interface, similar to most other type and dapatack registry implementations: where a `MapCodec` represents the statically registered type, and the `WorldCarver` interface itself is the world datapack registry. As such, the registries have been renamed to reflect that fact: `BuiltInRegistries#CARVER` is now `CARVER_TYPE`, while `Registries#CONFIGURED_CARVER` (`minecraft:worldgen/configured_carver`) is now `CARVER` (`minecraft:worldgen/carver`).

The methods on `WorldCarver` are either now defaulted / static methods or have been removed. The three most important that remain are `codec` to attach the carver type to the carver; `carver`, which now takes in the `WorldGenerationContext`, `RandomSource`, chunk and source chunk `ChunkPos`, and the `CarverOutput`; and `isStartChunk`, which returns whether this is where the carver should start carving.

```java
// An example carver.
public record SingleEllipsoidCarver(float probability, HeightProvider y) implements WorldCarver {
    // The carver type map codec to register.
    public static final MapCodec<SingleEllipsoidCarver> MAP_CODEC = RecordCodecBuilder.mapCodec(instance ->
        instance.group(
            Codec.floatRange(0.0F, 1.0F).fieldOf("probability").forGetter(SingleEllipsoidCarver::probability),
            HeightProvider.CODEC.fieldOf("y").forGetter(SingleEllipsoidCarver::y),
        )
        .apply(instance, CanyonWorldCarver::new)
    );

    @Override
    public MapCodec<SingleEllipsoidCarver> codec() {
        // Attach the carver type to the carver.
        return MAP_CODEC;
    }

    @Override
    public boolean isStartChunk(RandomSource random) {
        // Check whether this is the starting chunk for the carver.
        return random.nextFloat() <= this.probability;
    }

    @Override
    public boolean carve(WorldGenerationContext context, RandomSource random, ChunkPos chunkPos, ChunkPos sourceChunkPos, CarverOutput output) {
        // Carve out any blocks.
        // Usually `CarverOutput#carve` or `WorldCarver#carveEllipsoid`.
        // Returns `true` if the carver was executed successfully, or otherwise `false`.

        WorldCarver.carveEllipsoid(
            chunkPos,
            sourceChunkPos.getBlockX(random.nextInt(16)),
            this.y.sample(random, context),
            sourceChunkPos.getBlockZ(random.nextInt(16)),
            1,
            1,
            output,
            (xd, yd, zd, y1) -> false
        );
        return true;
    }
}

// Register the map codec.
Registry.register(
    BuiltInRegistries.CARVER_TYPE,
    Identifier.fromNamespaceAndPath("examplemod", "single"),
    SingleEllipsoidCarver.MAP_CODEC
);
```

The `WorldCarver` can then be registered to the world datapack registry:

```java
// For some RegistrySetBuilder builder to generate the datapack entries.

// The resource key to register.
public static final ResourceKey<WorldCarver> EXAMPLE_CARVER = ResourceKey.create(
    Registries.CARVER,
    Identifier.fromNamespaceAndPath("examplemod", "example_carver")
);

builder.add(Registries.CARVER, bootstrap -> {
    bootstrap.register(
        EXAMPLE_CARVER,
        // Our carver.
        new SingleEllipsoidCarver(
            0.1f, VerticalAnchor.absolute(60)
        )
    );
});
```

And the generated JSON:

```json5
// For some carver.
// In `data/examplemod/worldgen/carver/example_carver.json`
{
    // Our carver.
    "type": "examplemod:single",
    "probability": 0.1,
    "y": 60
}
```

### More Template Rule Tests!

Three more `RuleTest`s have been added for use in the template system: `AllOfRuleTest`, `AnyOfRuleTest`, and `NotRuleTest` representing AND, OR, and NOT repsectively. While these can be used individual, vanilla uses them to create an IF/ELSE statment (via `RuleTest#either`) to determine what blocks to replace with ores based on the current height of the feature position.

### Interfacing with `BlockStateProvider`s

`BlockStateProvider` is now an interface instead of an abstract class. The only change aside from changing `extends` to `implements` is that `getState` now takes in a `LevelAccessor` instead of a `WorldGenLevel`.

```java
// An example block state provider.
public record AirProvider() implements BlockStateProvider {
    public static final AirProvider INSTANCE = new AirProvider();
    // The map codec used as the registry object.
    public static final MapCodec<AirProvider> CODEC = MapCodec.unit(INSTANCE);

    @Override
    public BlockState getState(LevelAccessor level, RandomSource random, BlockPos pos) {
        // The block to provide.
        return Blocks.AIR.defaultBlockState();
    }

    @Override
    public MapCodec<AirProvider> codec() {
        // The codec used for serialization.
        return CODEC;
    }
}

// Register the map codec.
Registry.register(
    BuiltInRegistries.BLOCK_STATE_PROVIDER_TYPE,
    Identifier.fromNamespaceAndPath("examplemod", "air"),
    AirProvider.CODEC
);
```

Block state providers can either be inlined or referenced as a world datapack registry (via `Registries#BLOCK_STATE_PROVIDER`) depending on reusability:

```java
// For some RegistrySetBuilder builder to generate the datapack entries.

// The resource key to register.
public static final ResourceKey<BlockStateProvider> EXAMPLE_PROVIDER = ResourceKey.create(
    Registries.BLOCK_STATE_PROVIDER,
    Identifier.fromNamespaceAndPath("examplemod", "example_provider")
);

builder.add(Registries.BLOCK_STATE_PROVIDER, bootstrap -> {
    bootstrap.register(
        EXAMPLE_PROVIDER,
        // Our state provider.
        new AirProvider()
    );
});
```

And the generated JSON:

```json5
// For some block state provider.
// In `data/examplemod/worldgen/block_state_provider/example_provider.json`
{
    // Our state provider.
    "type": "examplemod:air"
}
```

### Interfacing with `StructurePlacement`s

`StructurePlacement` is now an interface instead of an abstract class to allow more flexibility for where structures can generate. The original implementation is now called `AbstractSpreadingStructurePlacement`, which implements `StructurePlacement`.

`StructurePlacement` inherits five methods from `AbstractSpreadingStructurePlacement`: `codec` for the registry type object used during serialization, `isStructureChunk` to determine if a structure can generate at the given chunk XZ, `applyAdditionalChunkRestrictions` for any additional restrictions that prevent the structure from generating in that chunk, `getLocatePos` that returns the `BlockPos` of the structure start when locating the structure, and `locateOffset` which offsets the `getLocatePos`. Only `isStructureChunk` and `codec` need to be implemented:

```java
// An example structure placement.
public record InChunkStructurePlacement(ChunkPos pos) implements StructurePlacement {
    // The codec for the placement type registry object.
    public static final MapCodec<InChunkStructurePlacement> CODEC = ChunkPos.CODEC.fieldOf("pos")
        .xmap(InChunkStructurePlacement::new, InChunkStructurePlacement::pos);

    @Override
    public boolean isStructureChunk(ChunkGeneratorStructureState state, int sourceX, int sourceZ) {
        // Returns whether the structure can start generation in this chunk.
        return this.pos.x() == sourceX && this.pos.z() == sourceZ;
    }

    @Override
    public MapCodec<InChunkStructurePlacement> codec() {
        // The codec used for serialization.
        return CODEC;
    }
}

// Register the map codec.
Registry.register(
    BuiltInRegistries.STRUCTURE_PLACEMENT,
    Identifier.fromNamespaceAndPath("examplemod", "in_chunk"),
    InChunkStructurePlacement.CODEC
);
```

Then, the `StructurePlacement` can be used as part of a structure set:

```java
// For some RegistrySetBuilder builder to generate the datapack entries.

// The resource key to register.
public static final ResourceKey<StructureSet> EXAMPLE_SET = ResourceKey.create(
    Registries.STRUCTURE_SET,
    Identifier.fromNamespaceAndPath("examplemod", "example_set")
);

builder.add(Registries.STRUCTURE_SET, bootstrap -> {
    bootstrap.register(
        EXAMPLE_SET,
        new StructureSet(
            List.of(/*...*/),
            // Our structure placement.
            new InChunkStructurePlacement(new ChunkPos(1, 1))
        )
    );
});
```

And the generated JSON:

```json5
// For some structure set.
// In `data/examplemod/worldgen/structure_set/example_set.json`
{
    // Our structure placement.
    "placement": {
        "type": "examplemod:in_chunk",
        "pos": [ 1, 1 ]
    },
    "structures": [
        // ...
    ]
}
```

### Rewriting Density Functions

`DensityFunction` has been split into the raw, registered values in `DensityFunction`, and the compiled values to sample within `DensitySampler`. This is similar to the material rules and conditions (previously surface rules) splitting the actual raw value with the compiled evaluator.

`DensityFunction` has five methods that require implementing. First is the `codec` representing the function type, now a standard `MapCodec`. Next is the `range` of values the function can produce, replacing `minValue` and `maxValue`. `domainAxes` represent a bit mask corresponding to the `Direction$Axis`es operate within: `1` for X, `2` for Y, and `4` for Z. `rewriteChildren` takes in a `DfRewriteRule` used to rewrite the child functions, typically for inlining or simplifying slices. Finally, `compileSampler` creates the `DensitySampler` used to compute the value at a given position. `DensitySampler` inherits `compute` and `fillArray` as `sampleValue` and `sampleVolume`, respectively.

```java
// An example density function
public record SineFunction(DensityFunction input) implements DensityFunction {
    // The function type map codec to register.
    public static final MapCodec<SineFunction> CODEC = DensityFunction.CODEC.fieldOf("input")
        .xmap(SineFunction::new, SineFunction::input);
    
    @Override
    public MapCodec<SineFunction> codec() {
        // Attach the function type to the function.
        return CODEC;
    }

    @Override
    public Interval range() {
        // Sine function can only output [-1, 1].
        return Interval.of(-1f, 1f);
    }

    @Override
    public @DensityFunction.Axes int domainAxes() {
        // Should match the axes operating upon.
        // In our case, the input function's axes.
        return this.input.domainAxes();
    }

    @Override
    public DensityFunction rewriteChildren(DfRewriteRule rule) {
        // Rewrite the children, and if there's a change, construct a new function.
        DensityFunction input = rule.rewrite(this.input);
        return input == this.input ? this : new SineFunction(input);
    }

    @Override
    public DensitySampler compileSampler(DensityFunction.CompileContext context) {
        // We can optimize by checking if the input is constant.
        if (this.input instanceof ConstantFunction(float value)) {
            return new ConstantFunction.Sampler(Mth.sin(value));
        }
        // Otherwise, we pipe the sampler to our own.

        // Compile the sampler into a density sampler.
        DensitySampler input = this.input.compileSampler(context);
        return new SineFunction.Sampler(input);
    }

    // Our sampler instance.
    public record Sampler(DensitySampler input) implements DensitySampler {
        @Override
        public void sampleVolume(SamplerContext context, DensityBuffer outputBuffer, DensityVolume volume) {
            // Sample the input first.
            this.input.sampleVolume(context, outputBuffer, volume);

            // Then replace with the computed value.
            for (int i = 0; i < outputBuffer.size(); i++) {
                outputBuffer.set(i, Mth.sin(outputBuffer.get(i)));
            }
        }

        @Override
        public float sampleValue(SamplerContext context, int blockX, int blockY, int blockZ) {
            // Sample the value at the given position.
            return Mth.sin(this.input.sampleValue(context, blockX, blockY, blockZ));
        }
    }
}

// Register the map codec.
Registry.register(
    BuiltInRegistries.DENSITY_FUNCTION_TYPE,
    Identifier.fromNamespaceAndPath("examplemod", "sin"),
    SineFunction.CODEC
);
```

Then, the `DensityFunction` can be registered or inlined:

```java
// For some RegistrySetBuilder builder to generate the datapack entries.

// The resource key to register.
public static final ResourceKey<DensityFunction> EXAMPLE_FUNCTION = ResourceKey.create(
    Registries.DENSITY_FUNCTION,
    Identifier.fromNamespaceAndPath("examplemod", "example_function")
);

builder.add(Registries.DENSITY_FUNCTION, bootstrap -> {
    bootstrap.register(
        EXAMPLE_FUNCTION,
        // Our function.
        new SineFunction(
            new ConstantFunction(0)
        )
    );
});
```

And the generated JSON:

```json5
// For some function.
// In `data/examplemod/worldgen/density_function/example_function.json`
{
    // Our function.
    "type": "examplemod:sin",
    "input": 0
}
```

### Not a Surface, but a Material

All references to the surface system have been replaced with the word 'material' (e.g. `SurfaceSystem` -> `MaterialSystem`). Additionally, `MaterialRules` (once `SurfaceRules`) has had its inner types split into separate files in response to material rules and conditions now having their own world datapack registries. As such, `BuiltInRegistries#MATERIAL_CONDITION`, `MATERIAL_RULE` are now `MATERIAL_CONDITION_TYPE`, `MATERIAL_RULE_TYPE`.

The underlying classes themselves remain relatively the same for implementation.

`SurfaceRules$RuleSource` is now `MaterialRule`, `compile`d to a `RuleEvaluator` (previously `SurfaceRules$SurfaceRule`) using the `MaterialRuleContext` (previously `SurfaceRules$Context`):

```java
public record RandomRule(List<BlockState> states) implements MaterialRule {
    // The material rule type map codec to register.
    public static final MapCodec<RandomRule> CODEC = ExtraCodecs.nonEmptyList(BlockState.CODEC.listOf())
        .fieldOf("states").xmap(RandomRule::new, RandomRule::states);

    @Override
    public MapCodec<RandomRule> codec() {
        // Attach the material rule type to the material rule.
        return CODEC;
    }

    // Previously `apply`.
    @Override
    public RuleEvaluator compile(MaterialRuleContext context) {
        // Compiles the evaulator to return a block state
        // based on the passed position.
        PositionalRandomFactory factory = context.getOrCreateRandomFactory(
            Identifier.fromNamepsaceAndPath("examplemod", "random")
        );
        return (blockX, blockY, blockZ) -> {
            RandomSource random = randomFactory.at(blockX, blockY, blockZ);
            return states.get(random.nextInt(states.size()));
        };
    }
}

// Register the map codec.
Registry.register(
    BuiltInRegistries.MATERIAL_RULE_TYPE,
    Identifier.fromNamespaceAndPath("examplemod", "random_rule"),
    RandomRule.CODEC
);
```

Then, the `MaterialRule` can be registered or inlined:

```java
// For some RegistrySetBuilder builder to generate the datapack entries.

// The resource key to register.
public static final ResourceKey<MaterialRule> EXAMPLE_RULE = ResourceKey.create(
    Registries.MATERIAL_RULE,
    Identifier.fromNamespaceAndPath("examplemod", "example_rule")
);

builder.add(Registries.MATERIAL_RULE, bootstrap -> {
    bootstrap.register(
        EXAMPLE_RULE,
        // Our material rule.
        new RandomRule(
            List.of(Blocks.DIRT.defaultBlockState())
        )
    );
});
```

And the generated JSON:

```json5
// For some material rule.
// In `data/examplemod/worldgen/material_rule/example_rule.json`
{
    // Our material rule.
    "type": "examplemod:random_rule",
    "states": [
        "minecraft:dirt"
    ]
}
```

`MaterialCondition` (previously `SurfaceRules$ConditionSource`) is similar, where the condition is `compile`d to a `ConditionEvaluator` (previously `SurfaceRules$Condition`) using the `MaterialRuleContext`:

```java
public record ProbabilityCondition(float probability) implements MaterialCondition {
    // The material condition type map codec to register.
    public static final MapCodec<ProbabilityCondition> CODEC = Codec.floatRange(0.0F, 1.0F)
        .fieldOf("probability").xmap(
            ProbabilityCondition::new, ProbabilityCondition::probability
        );

    @Override
    public MapCodec<ProbabilityCondition> codec() {
        // Attach the material condition type to the material condition.
        return CODEC;
    }

    // Previously `apply`.
    @Override
    public ConditionEvaluator compile(MaterialRuleContext context) {
        // Compiles the evaulator to test whether the given condition is
        // valid.
        RandomSource random = context.getOrCreateRandomFactory(
            Identifier.fromNamepsaceAndPath("examplemod", "random")
        ).fromHashOf(Identifier.fromNamepsaceAndPath("examplemod", "random"));
        return () -> random.nextFloat() < this.probability;
    }
}

// Register the map codec.
Registry.register(
    BuiltInRegistries.MATERIAL_CONDITION_TYPE,
    Identifier.fromNamespaceAndPath("examplemod", "random_condition"),
    ProbabilityCondition.CODEC
);
```

Then, the `MaterialCondition` can be registered or inlined:

```java
// For some RegistrySetBuilder builder to generate the datapack entries.

// The resource key to register.
public static final ResourceKey<MaterialCondition> EXAMPLE_CONDITION = ResourceKey.create(
    Registries.MATERIAL_CONDITION,
    Identifier.fromNamespaceAndPath("examplemod", "example_condition")
);

builder.add(Registries.MATERIAL_CONDITION, bootstrap -> {
    bootstrap.register(
        EXAMPLE_CONDITION,
        // Our material condition.
        new ProbabilityCondition(0.1f)
    );
});
```

And the generated JSON:

```json5
// For some material condition.
// In `data/examplemod/worldgen/material_condition/example_condition.json`
{
    // Our material condition.
    "type": "examplemod:random_condition",
    "probability": 0.1
}
```

### Noisy Musical Chairs

The noise classes have been reworked into a more hierarchical structure, causing the majority of the classes to be either renamed or reimplemented. Now, all noise classes implement `Noise` interface, which defines the `range` of values, and two `get` methods to return a `float` at a given XY or XYZ coordinate. `Noise` has two types: `GradientNoise` and `NoiseStack`.

`GradientNoise`, as the name implies, generates random gradients which are then inerpolated between using their dot product. Both `PerlinNoise` and `SimplexNoise` are a type of `GradientNoise`. `PerlinNoise` also has a `SmearedPerlinNoise` subtype (replaces `ImprovedNoise`), which fudges the Y scaling.

`NoiseStack`, on the other hand, represents a layering of multiple different noise levels together. The original `PerlinSimplexNoise` impementation is best represented as a `NoiseStack` of `SimplexNoise` layers. The same applies to the original `PerlinNoise` as a `NoiseStack` of `PerlinNoise` layers.

Additionally, `Registries#NOISE` that originally took in a `NormalNoise$NoiseParameters` now takes in a `NormalNoise`. However, the backing codec just simply remaps to the newly named `NormalNoise$Parameters`. Still, there are a number of additions.

`amplitudes` was renamed to `amplitude_modifiers` while `firstOctave` was renamed to `base_octave`. There is also `ocatve_count`, which is the length of the `amplitude_modifiers`. `base_amplitude` represents the base amplitude of the noise value before the modifiers are applied. And finally, `normalize`, when `true`, normalizes the base amplitude based on the number of octaves.

```java
// For some RegistrySetBuilder builder to generate the datapack entries.

// The resource key to register.
public static final ResourceKey<NormalNoise> EXAMPLE_NOISE = ResourceKey.create(
    Registries.NOISE,
    Identifier.fromNamespaceAndPath("examplemod", "example_noise")
);

builder.add(Registries.NOISE, bootstrap -> {
    bootstrap.register(
        EXAMPLE_NOISE,
        // The normal noise.
        // Computes the base amplitude from the base octave
        // and amplitude modifiers.
        NormalNoise.createParity(-7, 0.4, 0.5, 1.0)
    );
});
```

And the generated JSON:

```json5
// For some noise.
// In `data/examplemod/worldgen/noise/example_noise.json`
{
    // The normal noise.
    // If the modifier list is all 1s, it can be excluded.
    "amplitude_modifiers": [
        0.4,
        0.5,
        1.0
    ],
    // Computed from the base octave and amplitude modifiers.
    "base_amplitude": 0.8500634887071167,
    "base_octave": -7,
    // The number of octaves. Matches the number of modifiers.
    "octave_count": 3
}
```

- `net.minecraft.advancements`
    - `Advancement$Builder#display` -> `rootDisplay`, `Identifier` can no longer be null
        - `display` method still exists, but no longer takes in an `Identifier`
        - `save` now takes in a `BootstrapContext<Advancement>` instead of a `Consumer<AdvancementHolder>`
    - `AdvancementHolder`
        - `LIST_STREAM_CODEC` is removed
        - `register` - Registers an `Advancement` to the given context.
    - `AdvancementNode`
        - `isTask` - Checks whether the node has a parent.
        - `isRoot` - Whether the node does not have a parent.
    - `AdvancementProgress#serializeToNetwork`, `fromNetwork` replaced by `STREAM_CODEC`
    - `AdvancementRequirements(FriendlyByteBuf)`, `#write` replaced by `STREAM_CODEC`
    - `AdvancementRewards` now takes in a `HolderSet` of `LootTable`s instead of a list of table `ResourceKey`s
        - `$Builder#loot`, `addLootTable` now takes in a `Holder` of a `LootTable` instead of its `ResourceKey`
    - `AdvancementTree`
        - `addAll` now takes in an `Iterable` of `AdvancementHolder` instead of a `Collection`
        - `tasks` - Returns the child nodes.
        - `setListener`, `$Listener` are removed
        - `repositionNodes` - Updates the screen position of all nodes currently in the tree.
    - `AdvancementType#STREAM_CODEC` - The network codec for the type.
    - `CriterionProgress#serializeToNetwork`, `fromNetwork` replaced by `STREAM_CODEC`
    - `DisplayInfo` is now a record
        - `announceChat` -> `announceToChat`
        - `setLocation` -> `AdvancementNode#setLocation`
        - `getX`, `getY` -> `AdvancementNode#x`, `y`
- `net.minecraft.advancements.predicates`
    - `BlockPredicate`
        - `MAP_CODEC` - A map codec for the predicate.
        - `matchesState` is now `public` from `private`
        - `matchesBlockEntity` is now `public` from `private`, no longer taking in the `NbtPredicate`
        - `willMatchBlockEntity` - If the predicate has NBT data present but no components.
    - `ContextAwarePredicate` class is removed
        - Replaced by a `Holder<LootItemCondition>`
    - `MobEffectsPredicate` now implements `Predicate<MobEffectInstance>`
        - `STREAM_CODEC`, `$MobEffectInstancePredicate#STREAM_CODEC`, `$MobEffectInstancePredicate#MAP_STREAM_CODEC` - Network codecs for the predicate.
    - `TagPredicate` now takes in a `HolderSet` instead of a `TagKey`
        - `is`, `isNot` now take in the `HolderGetter`
            - They also now have an overload that only takes in the `HolderSet`
- `net.minecraft.advancements.predicates.entity`
    - `EntityPredicate`
        - `ADVANCEMENT_CODEC` is removed
        - All references of `ContextAwarePredicate` replaced by `Holder<LootItemCondition>`
- `net.minecraft.advancements.triggers`
    - All references of `ContextAwarePredicate` replaced by `Holder<LootItemCondition>`
    - `BeeNestDestroyedTrigger$TriggerInstance` now takes in an optional `HolderSet<Block>` instead of a `Holder<Block>` and an optional `StatePropertiesPredicate` for the block state
        - `destroyedBeeNest` now takes in a `HolderGetter<Block>`, or instead a `HolderSet<Block>` instead of a `Block`
    - `BrewedPotionTrigger`
        - `trigger`, `$TriggerInstance#matches` now takes in `PotionContents` instead of a `Holder<Potion>`
        - `$TriggerInstance#potion` is now an optional `PotionsPredicate` instead of a `Holder<Potion>`
    - `EnterBlockTrigger$TriggerInstance` now takes in an optional `HolderSet<Block>` instead of a `Holder<Block>`
        - `entersBlock` now takes in a `HolderSet<Block>` instead of a `Holder<Block>`
    - `ItemUsedOnLocationTrigger$TriggerInstance`
        - `placedBlock`, `placedBlockWithProperties` now take in a `HolderGetter<Block>`
        - `placedBlock(LootItemCondition.Builder...)` -> `placedBlock(LootItemCondition.Builder)`
    - `LootTableTrigger$TriggerInstance` now takes in a `HolderSet<LootTable>` instead of a `ResourceKey`
        - `lootTableUsed` can now either take a `Holder<LootTable>` or `HolderSet<LootTable>` instead of a `ResourceKey`
    - `RecipeCraftedTrigger$TriggerInstance` now takes in a `HolderSet<Recipe>` instead of a `ResourceKey`
        - `craftedItem`, `crafterCraftedItem` now take in a `HolderSet<Recipe>` instead of a `ResourceKey`
    - `RecipeUnlockedTrigger`
        - `unlocked` now takes in a `Holder<Recipe>` or `HolderSet<Recipe>` instead of a `ResourceKey`
        - `$TriggerInstance` now takes in a `HolderSet<Recipe>` instead of a `ResourceKey`
    - `SlideDownBlockTrigger$TriggerInstance` now takes in an optional `HolderSet<Block>` instead of a `Holder<Block>`
        - `slidesDownBlock` now takes in an optional `HolderSet<Block>` instead of a `Block`
- `net.minecraft.client.gui.screens.advancements`
    - `AdvancementTab` now takes in the `AdvancementWidget`, `ItemStackTemplate`, title `Component`, and `Identifier` background instead of the `AdvancementNode` and `DisplayInfo`
        - `copyPosition` - Copies the position of another tab.
        - `getRootNode` -> `getRootAdvancement`, returning the `AdvancementHolder` instead of the `AdvancementNode`
        - `getDisplay` is removed
    - `AdvancementWidget` is now `private` from `public`, no longer taking in the `AdvancementTab`
        - `createWidget` - Creates a widget for the advancement node.
        - `extractHover` now takes in the screen width
        - `attachToParent` now takes in the `AdvancementTab`
        - `getAdvancement` - Returns the referenced advancement.
        - `getDisplay` - The display information for the advancement.
- `net.minecraft.client.multiplayer`
    - `ClientAdvancements`
        - `progress` - Returns all advancement progress.
        - `getTree` -> `tree`
        - `$Listener` no longer extends `AdvancementTree$Listener`
            - `onUpdateAdvancementProgress` replaced by `onAdvancementsUpdated`, `onAdvancementsCleared`
    - `ClientPacketListener`
        - `potionBrewing` is removed
        - `fuelValues` is removed
- `net.minecraft.core`
    - `Holder#getRegisteredNameIfPresent` - Returns an optional continaing the registered identifier if present.
    - `HolderGetter` now extends `HolderOwner`
    - `HolderLookup$RegistryLookup` no longer extends `HolderOwner`
    - `HolderOwner#canSerializeIn` -> `canSerialize`
    - `RegistryCodecs` -> `.core.registries.codec.RegistryCodecs`
        - `homogeneousList` -> `holderSet`
    - `RegistrySetBuilder`
        - `add` no longer takes in a `Lifecycle`
            - It also has an overload that takes in only a `MultiRegistryBootstrap`
        - `build` now takes in a `HolderLookup$Provider` instead of a `RegistryAccess`
        - `build` now takes in a `HolderLookup$Provider` instead of a `RegistryAccess` for the context
        - `$BuildState` split betweeen `$BuildState` and `$BootstrappedRegistryState`
        - `$RegistryBootstrap` -> `SingleRegistryBootstrap`
        - `$EmptyTagLookupWrapper` -> `EmptyTagLookupWrapper`, not one-to-one
- `net.minecraft.core.component`
    - `BlockTransformer`, `DataComponents#BLOCK_TRANSFORMER` - A component that performs a transformation on a block when right clicked by an item.
    - `DataComponents`
        - `SWING_ANIMATION` split into `ATTACK_ANIMATION`, `INTERACT_ANIMATION`
        - `MAP_COLOR` is removed
        - `VILLAGER_FOOD` - Marks an item that can be eaten by a villager.
        - `COMPOSTABLE` - Marks an item as compostable in a composter.
        - `COOKING_FUEL` - Marks an item that can be burned as fuel in a furance.
        - `BREWING_FUEL` - Marks an item that can be used as fuel in a brewing stand.
        - `MOB_VISIBILITY` - Affects the visibility of the holder to another entity when targetting.
        - `PROVIDES_POTTERY_PATTERN` - Marks the item as able to provide a deocration for a pot.
        - `SIGN_TEXT_FRONT`, `SIGN_TEXT_BACK` - Sign block front and back text.
        - `WAXED` - If a block has been waxed (set as part of the block entity like signs).
        - `CUSHION_COLOR` - The color of a cushion block.
    - `Removed` - Marks a component object of having its data removed.
- `net.minecraft.core.component.predicate`
    - `PotionsPredicate` can now specify an optional collection of effects the potion must have
        - `STREAM_CODEC` - The network codec.
        - `potions` -> `ofPotions`
        - `ofPotion` - Creates a predicate that checks against one potion.
- `net.minecraft.core.registries.codec.RegistryCodecs#holder` - Returns a holder codec for the specified registry.
- `net.minecraft.core.registries`
    - `BootstrapRegistry` - A registry used when bootstrapping values for a datapack.
    - `BuiltInRegistries`
        - `LOOT_NUMBER_PROVIDER_TYPE` split into `CONTEXT_FLOAT_PROVIDER_TYPE`, `CONTEXT_INT_PROVIDER_TYPE`
        - `CARVER` -> `CARVER_TYPE`, now a `MapCodec`
        - `FEATURE` -> `FEATURE_TYPE`, now a `MapCodec`
        - `STRUCTURE_PLACEMENT` is now a `MapCodec` instead of a `StructurePlacementType`
        - `BLOCKSTATE_PROVIDER_TYPE` -> `BLOCK_STATE_PROVIDER_TYPE`, now a `MapCodec` instead of a `BlockStateProviderType`
        - `PLACEMENT_MODIFIER_TYPE` is now a `MapCodec` instead of a `PlacementModifierType`
        - `MATERIAL_CONDITION` -> `MATERIAL_CONDITION_TYPE`
        - `MATERIAL_RULE` -> `MATERIAL_RULE_TYPE`
        - `BLOCK_TYPE` is removed
        - `DECORATED_POT_PATTERN` is removed
            - Now a datapack registry
        - `CONTEXT_KEY_SET` - A registry containing loot context sets.
        - `NOISE` is now a `NormalNoise` insetad of `NormalNoise$NoiseParameters`
    - `MultiRegistryBootstrap` - A registry bootstrap for multiple registries at the same time (e.g., recipes).
    - `PatchedRegistry` - A registry used when adding dynamic values from a datapack.
    - `Registries`
        - `BLOCKSTATE_PROVIDER_TYPE` -> `BLOCK_STATE_PROVIDER_TYPE`, now a `MapCodec` instead of a `BlockStateProviderType`
        - `BLOCK_TYPE` is removed
        - `CARVER` -> `CARVER_TYPE`, now a `MapCodec`
        - `FEATURE` -> `FEATURE_TYPE`, now a `MapCodec`
        - `LOOT_NUMBER_PROVIDER_TYPE` split into `CONTEXT_FLOAT_PROVIDER_TYPE`, `CONTEXT_INT_PROVIDER_TYPE`
        - `MATERIAL_CONDITION` -> `MATERIAL_CONDITION_TYPE`
        - `MATERIAL_RULE` -> `MATERIAL_RULE_TYPE`
        - `PLACEMENT_MODIFIER_TYPE` is now a `MapCodec` instead of a `PlacementModifierType`
        - `STRUCTURE_PLACEMENT` is now a `MapCodec` instead of a `StructurePlacementType`
        - `CONTEXT_KEY_SET` - A registry containing loot context sets.
        - `BLOCK_STATE_PROVIDER` - Datapack registry for `BlockStateProvider`s.
        - `CONFIGURED_CARVER` -> `CARVER`
        - `CONFIGURED_FEATURE` -> `FEATURE`
        - `MATERIAL_CONDITION` - Datapack registry for `MaterialCondition`s.
        - `MATERIAL_RULE` - Datapack registry for `MaterialRule`s.
        - `NOISE` is now a `NormalNoise` insetad of `NormalNoise$NoiseParameters`
        - `BLOCK_TRANSFORMER` - Datapack registry for `BlockTransformer`s.
        - `SLOT_SOURCE` - Datapack registry for `SlotSource`s
        - `CONTEXT_FLOAT_PROVIDER`, `CONTEXT_INT_PROVIDER` - Datapack registries for number providers.
- `net.minecraft.resources`
    - `HolderSetCodec` -> `.minecraft.core.registries.codec.HolderSetCodec`
    - `RegistryFileCodec` -> `.minecraft.core.registries.codec.RegistryFileCodec`
    - `RegistryFixedCodec` -> `.minecraft.core.registries.codec.RegistryFixedCodec`
- `net.minecraft.data.advancements`
    - `AdvancementProvider` now implements `SingleRegistryBootstrap` instead of `DataProvider`
        - The constructor only takes in a list of `AdvancementSubProvider$Factory`s
    - `AdvancementSubProvider` is now an abstract `class` from an `interface`
        - `generate` no longer takes in any arguments
        - `createPlaceholder` is removed
        - `$Factory` - Creates a sub provider with the given advancement registration context.
- `net.minecraft.data.loot`
    - `BlockLootSubProvider` now takes in a `LootTableSubProvider$Context` instead of a `HolderLookup$Provider`
        - `registries` replaced by `output`
        - `enchantments`, `items`, `blocks`, `predicates` - Common registries to query from.
        - `hasSilkTouch` now returns a `Holder<LootItemCondition>` instead of a `LootItemCondition$Builder`
        - `hasShears` now returns a `Holder<LootItemCondition>` instead of a `LootItemCondition$Builder`
        - `createSelfDropDispatchTable` now takes a `Holder<LootItemCondition>` instead of a `LootItemCondition$Builder` for the condition
        - `createSingleItemTable` now takes a `Holder<ContextIntProvider>` instead of a `NumberProvider` for the count
        - `createSingleItemTableWithSilkTouch` now takes a `Holder<ContextIntProvider>` instead of a `NumberProvider` for the count
    - `EntityLootSubProvider` now takes in a `LootTableSubProvider$Context` instead of a `HolderLookup$Provider`
        - `registries` replaced by `output`
        - `enchantments`, `items`, `entityTypes`, `frogVariants`, `damageTypes`, `lootTables` - Common registries to query from.
        - `createSheepDispatchPool` now takes a `ColorCollection<Holder<LootTable>>` instead of a `ResourceKey`
        - `killedByFrog` no longer takes in the `HolderGetter` registry
        - `killedByFrogVariant` no longer takes in the `HolderGetter` registries
    - `LootTableProvider` now implements `SingleRegistryBootstrap` instead of `DataProvider`
        - The constructor no longer takes in the `PackOutput` or registries `CompletableFuture`
        - `$MissingTableProblem` is removed
        - `$SubProviderEntry#provider` replaced by `bootstrap`, now a `LootTableSubProvider$Factory`
    - `LootTableSubProvider`
        - `generate` replaced by `$Context#accept`, not one-to-one
        - `run` - Runs the sub provider.
        - `$Context` - A registrar wrapper for registering loot tables.
        - `$Factory` - Creates a sub provider with the given loot table registration context.
- `net.minecraft.data.recipes`
    - `BrewingProvider` - A provider for brewing recipes.
    - `BrewingRecipeBuilder` - A builder for brewing mixes and container transformations.
    - `RecipeOutput` now extends `BootstrapContextAccess`
        - `includeRootAdvancement` is removed
    - `RecipeProvider` now takes `BootstrapContext`s for the recipe and avancements instead of a `HolderLookup$Provider` and `RecipeOutput`
        - `registries` is removed
        - `advancementOutput` - Registrar to register advancements.
        - `$Runner` is removed
    - `TransmuteRecipeBuilder#transmute` now has an overload that takes in a `TransmuteResult` for the result
- `net.minecraft.data.recipes.packs.VanillaBrewingProvider` - Vanilla brewing recipes.
- `net.minecraft.data.registries`
    - `RegistriesDatapackGenerator` now takes in a `String` name along with a collection of `RegistryDataLoader$RegistryData`
        - `forWorldLayer`, `forReloadableLayer` - Registries for specific layers of the game.
    - `RegistryPatchGenerator#createLookup` split into `createWorldLookup`, `createReloadableLookup`
        - Original usage is `createWorldLookup`
    - `TradeRebalanceRegistries#createLookup` split into `createPatchedWorldRegistries`, `createPatchedReloadable`
        - Original usage is `createPatchedWorldRegistries`
    - `VanillaRegistries`
        - `validateThatAllBiomeFeaturesHaveBiomeFilter(HolderLookup$Provider)` is now `public` from `private`
            - Replaces the method of the same name
        - `validateLootData` - Validates registered loot tables.
        - `createLookup` split into `createWorldLookup`, `createReloadableLookup`
            - Original usage is `createWorldLookup`
- `net.minecraft.data.tags.BlockItemTagsProvider$CombinedAppender#addTag` now has an overload that takes in a varargs of `BlockItemTagId`s
- `net.minecraft.data.worldgen`
    - `AbandonedCampStructurePools` - Structure template pools for abandoned camps.
    - `BiomeDefaultFeatures`
        - `addDappledForestVegetation` - Vegetation for dappled forest.
    - `BlockStateProviders` - Commonly used block state providers in worldgen.
    - `BootstrapContext` now extends `BootstrapContextAccess`
        - `register` no longer takes in a `Lifecycle`
        - `lookup` -> `BootstrapContextAccess#lookup`
    - `BootstrapContextAccess` - An accessor to other registries accessable during bootstrap.
    - `Carvers` is now an `interface` from a `class`
    - `NoiseRouterData` -> `.minecraft.world.level.levelgen.NoiseRouterData`
- `net.minecraft.data.worldgen.material`
    - `EndMaterialRules` - Material rules for the end.
    - `NetherMaterialRules` - Material rules for the nether.
    - `OverworldMaterialRules` - Material rules for the overworld.
    - `VanillaMaterialConditions` - Vanilla conditions for material placement.
    - `VanillaMaterialRules` - Common material rules.
- `net.minecraft.network.protocol.game.ClientboundUpdateAdvancementsPacket` is now a record
    - The constructor now takes in a list of `$PositionedAdvancement`s instead of `AdvancementHolder`s
    - `$PositionedAdvancement` - An advancement at a given relative screen location.
- `net.minecraft.resources`
    - `RegistryDataLoader`
        - `WORLDGEN_REGISTRIES` -> `WORLD_REGISTRIES`
        - `RELOADABLE_REGISTRIES` - Registries that can reloaded while in game.
    - `RegistryLoadTask#createRegistryInfo` is removed
    - `RegistryOps`
        - `owner` is removed
        - `$RegistryInfo` is removed
        - `$RegistryInfoLookup#lookup` now returns an optional `HolderGetter` instead of a `$RegistryInfo`
    - `ResourceKey#REGISTRY_STREAM_CODEC` - A network codec for a registry resource key.
- `net.minecraft.server`
    - `MinecraftServer`
        - `potionBrewing` is removed
        - `fuelValues` is removed
    - `RegistryLayer#WORLDGEN` -> `WORLD`
    - `ServerAdvancementManager` no longer implements `SimpleJsonResourceReloadListener`
- `net.minecraft.server.packs`
    - `AbstractPackResources` -> `AbstractPackMetadataResources`
    - `CompositePackResources` -> `OverlayedPackResources`
    - `FixedPathPackResources` - A pack that has all resources defined on class initialization.
    - `PackMetadataResources` - An interface that accesses the metadata of a pack.
    - `PackResources` now extends `PackMetadataResources`
        - `getRootResource` -> `PackMetadataResources#getRootResource`
        - `getMetadataSection` -> `PackMetadataResources#getMetadataSection`
        - `location` -> `PackMetadataResources#location`
        - `$Filter` - A predicate that checks whether a resource can be accessed.
    - `VanillaPackResources` no longer implements `PackResources`
        - The constructor is now `public` from package-private, taking in a `FixedPathPackResources` and the list of `PackResources`
            - `fullResources` - Returns all resourcee provided.
            - `asResourcesSupplier` - Constructs a resource supplier for the vanilla resources.
            - `asProvider` is removed
            - `asResourceManager` - Constructs a manager for accessing the vanilla client resources.
    - `VanillaPackResourcesBuilder#pushLayer` - Constructs a new pack layer with its own metadata section.
- `net.minecraft.server.packs.repository`
    - `BuiltInPackSource`
        - `createVanillaPack` now takes in a `Pack$ResourcesSupplier` instead of the `PackResources`
        - `fixedResources` is removed
    - `Pack`
        - `openMetadata` -> Returns the pack metadata.
        - `$ResourcesSupplier`
            - `openPrimary` -> `openMetadata`
            - `openFull` -> `openResources`, now returning a stream of `PackResources`
- `net.minecraft.server.packs.resources`
    - `FallbackResourceManager`
        - `fallbacks` in now `private` from `protected`
        - `push`, `pushFilterOnly` now take in a `PackResources$Filter` instead of an `Identifier` predicate
    - `Resource#readAllAsString` - Reads the resource data into a string.
    - `ResourceManager`
        - `listResources`, `listResourceStacks` now tak in a `ResourceManager$Selector` instead of an `Identifier` predicate
        - `$Selector` - A predicate that checks whether a resource can be accessed.
    - `SimpleJsonResourceReloadListener`
        - `SimpleJsonResourceReloadListener(HolderLookup$Provider, Codec, ResourceKey)` is removed
        - `scanDirectory` is removed
- `net.minecraft.util`
    - `BoundedFloatFunction#minValue`, `maxValue` merged into `range`
    - `CubicSpline`
        - `mapCoordinates` now takes in a function that returns a `BoundedFloatfunction` instead of a `UnaryOperator`
        - `forEachCoordinate` - Loops through all coordinates in the spline.
        - `minValue`, `maxValue` merged into `range`
        - `$Multipoint` no longer takes in the main and max `float` values
    - `Interval` - Represents a range between two values.
- `net.minecraft.util.context.ContextMap` is now final
    - `EMPTY` - An empty context map.
    - `builder` - Creates a builder for a context map.
    - `getOptional` -> `get`
    - `$Builder` constructor is now `private` from `public`
        - `withOptionalParameter` -> `set`
        - `withParameter`, `getParameter` are removed
        - `getOptionalParameter` -> `get`
        - `create` -> `buildAndValidate`
        - `build` - Creates the context map without validation against some key set.
- `net.minecraft.util.valueproviders.VeryBiasedToBottomInt` - An int provider that biases towards the minimum value by running the random call three times.
- `net.minecraft.world.attribute`
    - `AttributeRange#UNIT_FLOAT_EPSILON` - Represents a number between 0 and 1 with an episilon factor included.
    - `AttributeType` now takes in a `ToIntFunction` to convert the attribute value to an integer
        - `toInt` - Converts the attribute value to an integer.
    - `AttributeTypes`
        - `RGB_COLOR` is now a `Vector3fc` from an `Integer`
        - `ARGB_COLOR` is now a `Vector4fc` from an `Integer`
        - `MOB_SPAWN_SETTINGS` - Settings for mob spawning in an environment.
    - `BedRule` now splits the explodes `boolean` into one for destroying on use, and one for destroying on leaving the bed
    - `EnvironmentAttribute#isFullResolutionBiomes`, `$Builder#fullResolutionBiomes` - When true, gets the biome through `BiomeManager#getBiome` instead of `BiomeManager#getNoiseBiomeAtPosition`
    - `EnvironmentAttributes`
        - `STRAW_BED_RULE` - A rule for what happens with straw beds.
        - `NATURAL_MOB_SPAWNS` - A rule for how mobs should spawn.
        - `CREATURE_WORLD_GEN_SPAWN_PROBABILITY` - A rule for the likelihood creatures should spawn for each chunk on generation.
    - `EnvironmentAttributeSystem$Builder`
        - `addStaticLayers` - Adds the biome and dimension layers to the attribute system.
        - `addDynamicLayers` - Adds the weather and timeline layers to the attribute system.
    - `LerpFunction`
        - `ofColorVec3`, `ofColorVec4` - SRGB lerp functions.
        - `ofListCrossFade` - Cross fades the element values in a list by merging both lists toegether, setting the front elements with one minus the alpha value, and the to as the alpha value.
        - `$AlphaScaler` - A function that converts an item based on some alpha `float`.
- `net.minecraft.world.attribute.modifier`
    - `AttributeModifier`
        - `MOB_SPAWN_SETTINGS_LIBRARY` - Operations for mob spawn settings (only overlay).
        - `listLibrary` - Operations for a list of elements (only append).
        - `$OperationId`
            - `OVERLAY` - Merges the elements together, having the latter's contents take priority over the former.
            - `APPEND` - Appends the elements into a single collective.
    - `ColorModifier` now uses an arbitrary subject instead of an `Integer`
        - All constant modifiers are now split into `*_RBG` or `*_ARGB`, using a `Vector3fc` or `Vector4fc`, repsectively.
        - `$ArgbModifier` now uses an arbitrary subject instead of an `Integer`
        - `$BlendToGray#lerp` - Linearly interpolates between two gray blends based on an alpha.
        - `$RgbModifier` now uses an arbitrary subject instead of an `Integer`
    - `ListModifier` - A modifier that joins two lists together.
    - `MobSpawnSettingsModifier` - Merges the spawn settings together, having the latter's contents take priority over the former.
- `net.minecraft.world.clock`
    - `ClockInstance` - An interfaces that defines a clock's data.
    - `ClockManager#getTotalTicks` -> `getInstance`, now returning a `ClockInstance` instead of a `long`
    - `ServerClockManager`
        - `moveToTimeMarker` now returns a `$MoveResult` instead of a `boolean`
        - `$MoveResult` - The result of moving a time marker.
        - `$ClockInstance` -> `$ServerClockInstance`, now implementing `ClockInstance`, `public` from `private`
- `net.minecraft.world.food.VillagerFood` - A data component representing that the item is food for a villager, providing a certain amount of nutrition.
- `net.minecraf.tworld.inventory.BrewingStandMenu`
    - `getTotalFuel` - The amount of fuel remaining in the brewing stand.
    - `getTotalBrewingTicks` - How long it takes for the brewing stand to finish brewing.
    - `$PotionSlot` now takes in a `RecipeAccess`
        - `mayPlaceItem` is removed
- `net.minecraft.world.item`
    - `AxeItem` class is removed
    - `HoeItem` class is removed
    - `Instrument` now takes in an `int` for how much durability to remove when used
    - `Instruments`
        - `GOAT_HORN_INSTRUMENT_DAMAGE` - How much damage a goat horn takes when used.
        - `register` now takes in an `int` for how much durability to remove when used
    - `Item$Properties`
        - `villagerFood` - Adds the `DataComponents#VILLAGER_FOOD` component, marking that a villager can eat this item.
        - `potPattern` - Adds the `DataComponents#PROVIDES_POTTERY_PATTERN` component, marking that the item displays a pattern on a decorated pot.
        - `signText` - Adds the `DataComponents#SIGN_TEXT_*` components, marking that the item has text to display on its front and/or back.
        - `cookingFuel` - Adds the `DataComponents#COOKING_FUEL` component, marking that the item can be used as fuel in a furnace.
        - `brewingFuel` - Adds the `DataComponents#BREWING_FUEL` component, marking that the item can be used as fuel in a brewing stand.
        - `compostable` - Adds the `DataComponents#COMPOSTABLE` component, marking that the item can be put in a composter.
        - `loweredMobVisibility` - Adds the `DataComponents#MOB_VISIBILITY` component, marking that the item can decrease range a mob can detect the holder from.
    - `ShovelItem` class is removed
    - `SignItem` class is removed
- `net.minecraft.world.item.alchemy`
    - `PotionBrewing` replaced by `BrewingRecipeBuilder`
    - `PotionContents`
        - `is` now has an overload that takes in the `TagKey` instead of the `Holder`
        - `isPotionWithoutCustomEffects` - Returns whether the brew is a registered potion with no additional mob effects.
- `net.minecraft.world.item.component`
    - `BlockTransformers` - All vanilla block transformers.
    - `BrewingFuel` - A data component that marks the item can be used as fuel in a brewing stand.
    - `BundleContents` now implements `ContainerComponent`
        - `$Mutable` now extends `GrowableMutableContainer`
            - The constructor no longer takes in anything
    - `ChargedProjectiles` now implements `ContainerComponent`
        - `$Mutable` - A mutable version of the container component.
    - `Compostable` - A data component that marks the item can be put in a composter.
    - `ContainerComponent` - An interface which defines a component that represents a backing container.
    - `CookingFuel` - A data component that marks the item can be used as fuel in a furnace.
    - `GrowableMutableContainer` - A mutable container that can grow in size.
    - `ItemContainerContents` now implements `ContainerComponent`
        - `MAX_SIZE` is now `public` from `private`
        - `$Mutable` - A mutable version of the container component.
    - `MapItemColor` record is removed
    - `MobVisibility` - A data component that marks the item can decrease range a mob can detect the holder from.
    - `SimpleMutableContainer` - A basic mutable container backed by a list of stacks.
    - `TooltipProvider$Getter` - Gets a `TooltipProvider` from some object, usually a data component.
    - `TypedEntityData#loadInto` now has an overload that takes in the `SpawnData` and the `DefaultedRegistry`
- `net.minecraft.world.item.consume_effects.TeleportRandomlyConsumeEffect` now takes in a `boolean` of whether the particles should be placed directionally towards the teleport target location
- `net.minecraft.world.item.crafting`
    - `AbstractCookingRecipe#cookingMapCodec` no longer takes in the `int` default time
    - `BrewingInput` - A `RecipeInput` that represents the input to a brewing recipe.
    - `BrewingRecipe` - A recipe that takes two potion ingredients and transmutes them into an item stack.
    - `Ingredient#getSingleItem` - Returns the backing item if there is only one valid value, otherwise an empty optional.
    - `MapExtendingRecipe` now takes in a `TransmuteResult` instead of an `ItemStackTemplate`
    - `PotionIngredient` - An ingredient that contains some `PotionsPredicate`.
    - `Recipe`
        - `CODEC` -> `DIRECT_CODEC`
            - `CODEC` now represents the holder `Recipe`
        - `LIST_CODEC` - Codec for a set of recipes.
    - `RecipeManager` no longer extends `SimplePreparableReloadListener`
        - `getLearnableRecipes` - Returns all recipes that can be seen in a recipe book (non-special recipes).
        - `fromJson` is removed
    - `RecipeMap#create` now takes in a `HolderLookup` for the recipe rather than an `Iterable`
    - `RecipePropertySet`
        - `BREWING_INPUTS` - Inputs to a brewing stand (potions).
        - `BREWING_REAGENTS` - Catalysts to the brewing stand inputs (blaze powder).
    - `TransmuteRecipe` now takes in a `TransmuteResult` instead of an `ItemStackTemplate`
    - `TransmuteResult` - An `ItemStackTemplate` that can either specify a new item to transmute to, or use the input item.
- `net.minecraft.world.item.crafting.display`
    - `SlotDisplay$TagSlotDisplay` now takes in a `HolderSet` instead of a `TagKey`
    - `SlotDisplayContext`
        - `FUEL_VALUES` is removed
        - `REGISTRIES` now uses a `RegistryAccess` generic instead of a `HolderLookup$Provider`
- `net.minecraft.world.item.enchantment`
    - `ConditionalEffect` now takes in a holder `LootItemCondition` instead of the raw condition for the requirements
    - `TargetedConditionalEffect` now takes in a holder `LootItemCondition` instead of the raw condition for the requirements
- `net.minecraft.world.item.enchantment.effects`
    - `ReplaceBlock` now takes in a holder `BlockStateProvider` instead of the raw provider for the block state
    - `ReplaceDisk` now takes in a holder `BlockStateProvider` instead of the raw provider for the block state
- `net.minecraft.world.item.slot`
    - `CompositeSlotSource` now takes in a `HolderSet` instead of a `List`
        - `createCodec`, `createInlineCodec` now takes in a function with a `HolderSet` input instead of a `List`
    - `CountingModifier` - A consumer that modifies the `ItemStack`, keeping track of how many was updated.
    - `RangeSlotSource#slotRange` - Creates a slot source from a range of slots in a container.
    - `SlotCollection`
        - `size` - The size of the collection.
        - `replaceSlotItems` - Replaces the selected slots with the items in the provider.
        - `modifySlots` - Modifies the selected slots using the given consumer.
    - `SlotSelector` - Selects slots based on the provided `ItemStack`.
    - `SlotSources`
        - `CODEC` -> `DIRECT_CODEC`
            - `CODEC` now represents the holder `SlotSource`
        - `LIST_CODEC` - Codec for a set of slot sources.
        - `group` -> `CompositeSlotSource#group`, now `private` from `public`
    - `TransformedSlotSource` now takes in a holder `SlotSource` instead of the raw source
        - `commonFields` now returns a `P1` that transforms into a holder `SlotSource` instead of the raw source
- `net.minecraft.world.item.trading`
    - `TradeCost` now takes in a holder `ContextIntProvider` instead of the raw `NumberProvider` for the count
    - `TradeSet` is now a record
        - The constructor now takes in a holder `ContextIntProvider` instead of the raw `NumberProvider` for the amount
    - `VillagerTrade` now takes in holder `LootItemCondition`s instead of the raw conditions, holder `ContextIntProvider`s instead of `NumberProvider`s for the max use and xp, and a holder `ContextFloatProvider` instead of a `NumberProvider` for the reputation discount
        - The other constructors are replaced by `builder`
        - `builder`, `$Builder` - A builder for creating a villager trade.
- `net.minecraft.world.level.Level`
    - `potionBrewing` is removed
    - `fuelValues` is removed
- `net.minecraft.world.level.biome`
    - `Biome` no longer takes in the `MobSpawnSettings`
        - `FROZEN_TEMPERATURE_NOISE` is now `public` from `private`, returning a `Noise` instead of `PerlinSimplexNoise`
        - `BIOME_INFO_NOISE` is now `Noise` instead of `PerlinSimplexNoise`
        - `getMobSettings` replaced by `EnvironmentAttributes#NATURAL_MOB_SPAWNS`
    - `BiomeManager` now takes in a `BiomeResolver` instead of a `$NoiseBiomeSource`
        - `getBiome` now hsa an overload that takes in the XYZ `int` coordinates
        - `$NoiseBiomeSource` is removed, typically replaced by `BiomeResolver`
    - `BiomeResolver#getNoiseBiome` no longer takes in the `Climate$Sampler`
    - `BiomeSource` no longer implements `BiomeResolver`
        - `getBiomesWithin` -> `BiomeResolver#getBiomesWithin`, no longer taking in the `Climate$Sampler`
        - `findClosestBiome3d` now takes in a `RandomState` instead of the `Climate$Sampler`
        - `findBiomeHorizontal` now takes in a `RandomState` instead of the `Climate$Sampler`
        - `createUncachedResolver`, `createCachingResolver`, `createResolver`, `createResolverForChunk` - Handles creating the `BiomesResolver` depending on context.
    - `CheckerboardColumnBiomeSource` now implements `BiomeResolver`
    - `Climate`
        - `empty` is removed
        - `findSpawnPosition` is removed
        - `$Parameter#distance(Climate$Parameter)` is removed
        - `$ParameterList#rebuildWithChildrenPerNode` - Creates a new parameter list with a given number of children in the per node in the index tree.
        - `$ParameterPoint#fitness` is now `public` from `private`
        - `$RTree#create` now has an overload that takes in the number of children per node in the tree
        - `$Sampler` now takes in `DensitySampler$Bound`s instead of `DensityFunction`s, no longer taking in the list of `$ParameterPoint`s for the spawn targets
            - `findSpawnPosition` is removed
        - `$SpawnFinder` -> `NoiseSpawnFinder`, not one-to-one
    - `FixedBiomeSource` now implements `BiomeResolver`
    - `MobSpawnSettings` no longer takes in the `float` creature generation probability
        - `DEFAULT_CREATURE_WORLD_GEN_SPAWN_PROBABILITY` is now `public` from `private`
        - `NO_SPAWNS` - Mobs do not spawn in any category.
        - `CODEC` is now a `Codec` instead of a `MapCodec`
        - `getCreatureProbability` replaced by `EnvironmentAttributes#CREATURE_WORLD_GEN_SPAWN_PROBABILITY`
        - `getMobs` -> `getMobsToSpawn`
        - `getMobsInCategory` - Gets the spawn settings for the given category.
        - `definedCategories` - The defined categories in the settings.
        - `allSpawnCosts` - A map of entity type to spawn cost.
        - `$Builder`
            - `addSpawn` now has overloads that take in the `EntityType`, an `int` weight, some min and max count, and optionally the `MobCategory`
            - `addAllSpawns` - Adds all spawns to the given category.
            - `noSpawns` - Spawns nothing for the given category.
            - `dontOverride` - Removes the spawn entries for the given category.
            - `addMobCharge` -> `addMobSpawnCost`
            - `addAllCosts` - Adds all spawns costs.
            - `creatureGenerationProbability` replaced by `EnvironmentAttributes#CREATURE_WORLD_GEN_SPAWN_PROBABILITY`
        - `$SpawnerData` now takes in an `IntProvider` instead of a min and max `int` for the count
    - `OverworldBiomeBuilder#spawnTarget` now takes in an `OverworldFunctionSet` and a holder `DensityFunction` for the weirdness, returning a list of `SpawnTargetPoint`s instead of `Climate$ParameterPoint`s
- `net.minecraft.word.level.block`
    - `*Block#CODEC` constants are removed
    - `AbstractBedBlock` - An abstract representaiton of a bed.
    - `BedBlock` now extends `AbstractBedBlock`
        - `PART` -> `AbstractBedBlock#PART`
        - `OCCUPIED` -> `AbstractBedBlock#OCCUPIED`
        - `getConnectedDirection` -> `AbstractBedBlock#getConnectedDirection`
        - `getBlockType` -> `AbstractBedBlock#getBlockType`
        - `findStandUpPosition` -> `AbstractBedBlock#findStandUpPosition`
    - `Block`
        - `dropFromBlockInteractLootTable` is now `public` from `private`, taking in the interacted `BlockPos`
        - `playerDestroy` now takes in a `ServerLevel` and `ServerPlayer` instead of a `Level` and `Player`
        - `bounceOn` - Runs when an entity's vertical resitution after moving is greater than 0, typically due an entity's or the hit block's bounciness.
        - `getFallDistanceReduction` - Returns how much the block reduces fall distance by, where 0 is no reduction, and 1 is 100% reduction.
        - `spawnDestroyParticles` no longer takes in the `Player`
            - `spawnDestroyByEntityParticles` replaces the `Player` version by taking in a nullable `Entity`
    - `BlockTypes` class is removed
    - `BonemealBlock#isValidBonemealTarget`, `isBonemealSuccess`, `performBonemeal` now takes in the `BonemealSource`
    - `BonemealSource` - The source who bonemealed the targetted block.
    - `BushBlock` now has an overload that takes in the block shape height.
        - `DEFAULT_SHAPE_HEIGHT` - The default max Y for the block shape.
    - `Campfire#dowse` -> `douse`
    - `ComposterBlock#bootstrap` replaced by `DataComponents#COMPOSTABLE`
    - `DirtPathBlock` -> `PathBlock`, not one-to-one
    - `DragonEggBlock#HORIZONTAL_TELEPORT_RADIUS`, `VERTICAL_TELEPORT_RADIUS` - The teleport radius on egg right click.
    - `FallingParticlesLeavesBlock` - An abstract leaves block that spawn falling particles.
    - `FarmlandBlock` now takes in the base `Block` it was created from
        - `turnToDirt` -> `turnToBaseBlock`, now non-static
    - `FlowerBedBlock` now takes in the block shape height
    - `FlowerBlock#EFFECTS_FIELD` is removed
    - `LeavesBlock` is no longer abstract
        - The constructor now takes in an `AmbientLeavesBlockSoundPlayer`
        - `leafParticleChance` -> `FallingParticlesLeavesBlock#leafParticleChance`
        - `spawnFallingLeavesParticle` -> `FallingParticlesLeavesBlock#spawnFallingLeavesParticle`
    - `RedStoneWireBlock` -> `RedstoneWireBlock`
    - `SaplingBlock`
        - `BRIGHTNESS_FOR_SAPLING_GROWTH` - The amount of light required for the sapling to grow.
        - `TICK_CHANCE_FOR_SAPLING_GROWTH` - The chance for a sapling to grow when randomly ticked.
    - `SculkBlock#GROWTH_INHIBITOR_RANGE` - The range that skulk blocks can spread within, provided there are at most two inhibitors.
    - `ShelfMushroomBlock` - The block for a shelf mushroom.
    - `StrawBedBlock` - The block for a straw bed.
    - `TintedParticleLeavesBlock` now extends `FallingParticlesLeavesBlock`
    - `TntBlock#prime(Level, BlockPos, LivingEntity)` is now `public` from `private`, now taking in the `ItemStack` primer
    - `UntintedParticleLeavesBlock` now extends `FallingParticlesLeavesBlock`
- `net.minecraft.world.level.block.entity`
    - `AbstractFurnaceBlockEntity`
        - `getBurnDuration` now takes in the `ServerLevel` instead of the `FuelValues`
        - `getSpeedMultiplier` - A scalar of how much faster it causes the input to be cooked.
    - `BaseContainerBlockEntity#getLootContext` - The base context for getting loot from a container.
    - `BrewingStandBlockEntity`
        - `FUEL_USES` replaced by `BrewingFuel#uses`
        - `DATA_TOTAL_*` - Identifiers for the data slots.
        - `BREWING_TIME_SECONDS` - The default amount of time it takes to brew.
        - `getUses` - How many times can the fuel be used.
        - `getSpeedMultiplier` - A scalar of how much faster it causes the input to be brewed.
        - `serverTick` now takes in the `ServerLevel` instead of the `Level`
    - `DecoratedPotPattern#CODEC` - A serialization codec.
    - `DecoratedPotPatterns`
        - `CODEC`, `STREAM_CODEC` - Codecs for datapack entries.
        - `itemToPatternMappings` replaced by `bootstrap`
    - `FuelValues` class is removed, replaced by `DataComponents#COOKING_FUEL`
    - `PotDecorations` now takes in optional `ItemStackTemplate`s instead of `Item`s
        - `allBrick` - Create a decoration of all bricks.
- `net.minecraft.world.level.block.grower.TreeGrower` now takes in `WeightedList`s of the trees, mega trees, and flower trees to spawn, instead of optionals of each specific tree type; and the `ResourceKey` of the shortest tree type instead of a `float` for the secondary chance
    - `canGrow` - If the tree can grow in the given location.
- `net.minecraft.world.level.block.state`
    - `BlockBehavior`
        - `fallDistanceReduction` - How much the block reduces fall distance by, where 0 is no reduction, and 1 is 100% reduction.
        - `codec`, `propertiesCodec`, `simpleCodec` are removed
        - `showAsInteractableInSpectatorMode` - Whether the block is shown as able to be interacted with in spectator mode.
        - `shouldRedstoneWireConnectTo` - If redstone wire from a given direction is able to connect to this block.
        - `$BlockStateBase`
            - `blocksMotion` is removed
            - `shouldRedstoneWireConnectTo` - If redstone wire from a given direction is able to connect to this block.
            - `showAsInteractableInSpectatorMode` - Whether the block is shown as able to be interacted with in spectator mode.
            - `isLightPermeable` - Whether light can permeate through this block.
            - `isViewBlocking` now takes in the `AABB` near plane box
            - `withPropertiesOf`, `copyProperty` - Constructs a `BlockState` with the associated properties.
        - `$Properties`
            - `CODEC` is removed
            - `fallDistanceReduction` - How much the block reduces fall distance by, where 0 is no reduction, and 1 is 100% reduction.
            - `isViewBlocking` now takes in a `$StateArgumentsPredicate<AABB>` instead of a `$StatePredicate`
    - `BlockState#CODEC` -> `FULL_CODEC`
        - `CODEC` now is able to serialize the `BlockState` as either a simple `Block` or the `BlockState` with its properties
- `net.minecraft.world.level.chunk`
    - `CarverOutput` - An interface that defines the basic carver functions, along with the min and max Y to carve between.
    - `CarvingMask` now implements `CarverOutput`
        - The constructor now takes in the min and max Y `int`s instead of the height and min Y
        - `setAdditionalMask` is removed
        - `set` -> `CarverOutput#carve`
        - `get`, `stream` replaced by `visit`, not one-to-one
        - `toArray` is removed
        - `isEmpty` - Whether the mask is empty.
        - `$Mask` -> `$Filter`
        - `$Visitor` - A consumer for visiting a column defined by the carver.
    - `ChunkAccess`
        - `noiseChunk` is removed
        - `incrementInhabitedTime` no longer takes in the `long` delta
        - `getOrCreateNoiseChunk` is removed
        - `fillBiomesFromNoise` no longer takes in the `Climate$Sampler`
        - `hasAnyStructureReferences` is removed
    - `ChunkGenerator`
        - `getOrigin` - Returns the origin position.
        - `applyCarvers` is removed
            - Original usage in `NoiseBasedChunkGenerator#generateCarvers`
        - `decorateBiomeResolver` - Wraps around the original resolver to determine how biomes are placed.
        - `buildSurface` is removed
            - Original usage in `NoiseBasedChunkGenerator#buildSurface`
        - `getMobsAt` now takes in a `Level` instead of the `Biome` holder
        - `fillFromNoise` -> `buildTerrain` now taking in the `BiomeManager`, `WorldGenRegion`, and set of possible holder `Biome`s
        - `addDebugScreenInfo` now takes in the `SamplerContext`
    - `ChunkGeneratorStructureState`
        - `createForFlat`, `createForNormal` now takes in the origin `ChunkPos`
        - `getDimensionOrigin` - Return the origin position.
    - `LevelChunk#replaceWithPacketData` now takes in the chunk XZ `int`s and the `ClientboundLevelChunkPacketData` instead of the raw buffer, map, and consumer
    - `LevelChunkSection#fillBiomesFromNoise` no longer takes in the `Climate$Sampler`
    - `ProtoChunk#getCarvingMask`, `getOrCreateCarvingMask`, `setCarvingMask` are removed
- `net.minecraft.world.level.chunk.state`
    - `ChunkStatus#NOISE`, `SURFACE`, `CARVERS` merged into `TERRAIN`
        - `CARVERS` is also used in `BIOMES`
    - `ChunkStatusTasks`
        - `generateNoise`, `generateSurface`, `generateCarvers` merged into `buildTerrain`
            - `generateCarvers` also used in `generateBiomes`
- `net.minecraft.world.level.levelgen`
    - `Aquifer`
        - `create` is removed
        - `computeSubstance` now takes in the XYZ coordinate `int`s instead of a `DensityFunction$FunctionContext`
        - `$Config#exclusion` - A density value that determines what should not be part of the aquifer.
    - `Beardifier` now implements `DensitySampler` instead of `DensityFunctions$BeardifierOrMarker`
    - `Density` constants are now `float`s instead of `double`s
    - `DensityFunction` -> `.densityfunction.DensityFunction`
        - `DensityFunction` has been split into `DensityFunction` and `DensitySampler`
        - `REFERENCE_CODEC` - A codec for the holder wrapped `DensityFunction`
        - `AXIS_*` - Flags that indicate the axes that the density functions are operated on.
        - `NO_AXES` - Operates on no axes.
        - `ALL_AXES` - Operates on all axes.
        - `axesFrom` - Converts a `Direction$Axis` to an `int` flag.
        - `compute` -> `DensitySampler#sampleValue`, not one-to-one
        - `fillArray` -> `DensitySampler#sampleVolume`, not one-to-one
        - `mapChildren`, `mapAll` replaced by `rewriteChildren`
        - `compileSampler` - Creates the sampler for the function given the context.
        - `minValue`, `maxValue` merged into `range`
        - `domainAxes` - The axes the density function operates upon.
        - `codec` is now a `MapCodec` from a `KeyDispatchKeyCodec`
        - `clamp` now takes in `float`s instead of `double`s
        - `invert` -> `negate`
        - `log` - Takes the natural logarithm of the sampled value.
        - `sign` - Returns the sign of the sampled value (1 positive, -1 negative, 0 zero).
        - `add` - Adds the argument to the sampled value.
        - `sub` - Subtracts the argument from the sampled value.
        - `mul` - Multiplies the argument to the sampled value.
        - `div` - Divides the sampled value by the argument.
        - `pow` - Raises the sampled value to the argument.
        - `$Axes` - An annotation that marks an `int` as an axis flag to which the density functions operate upon.
        - `$ContextProvider` is removed
        - `$FunctionContext` is removed, likely replaced by `MaterialRuleContext`
        - `$NoiseHolder` likely removed, using `Holder<NormalNoise>` instead
        - `$SimpleFunction` is removed
        - `$SinglePointContext` is removed
        - `$Visitor` is removed, like replaced by `DfRewriteRule`
        - `$CompileContext` - The context to help compile a function into a `DensitySampler`.
    - `DensityFunctions` -> `.densityfunction.DensityFunctions`
        - `NOISE_VALUE_CODEC` now `public` from `private`, `float` instead of a `double`
        - `sqrt` - Takes the square root of the sampled value.
        - `negate` - Negates the sampled value.
        - `log` - Takes the natural logarithm of the sampled value.
        - `sign` - Returns the sign of the sampled value (1 positive, -1 negative, 0 zero).
        - `interpolated` now takes in the `int` cell sizes for the XZ and Y directions
        - `flatCache`, `cache2d`, `cacheOnces`, `cacheAllInCell` replaced by `cache`, not one-to-one
        - `shiftA`, `shiftB`, `shift` now take a holder-wrapped `NormalNoise` instead of `NormalNoise$NoiseParameters`
        - `endIslands` -> `endOuterIslands`, no longer taking in the `long` seed
        - `distanceToPoint` - Determines the distance to the provided point using the specified metric.
        - `sub` - Subtracts the argument from the sampled value.
        - `div` - Divides the sampled value by the argument.
        - `pow` - Raises the sampled value to the argument.
        - `sliceY` - Samples the input along the Y axis with the fixed coordinate.
        - `slice` - Samples the input along the given axis with the fixed coordinate.
        - `gradient` - Samples using a gradient along the specified axis using the specific `TilingMode` outside the specified range.
        - `round` - Rounds the sampled value to the nearest `int`, calculated by the multiple if specified.
        - `floor` - Floors the sampled value, calculated by the multiple if specified.
        - `ceil` - Ceils the sampled value to the nearest `int`, calculated by the multiple if specified.
        - `truncate` - Truncates the sampled value in the direct closest to 0, calculated by the multiple if specified.
        - `remap` - Remaps the input function from the specified coordinate range to the new coordinate range.
        - `clampedMap` - Remaps the input function from the specified coordinate range to the new coordinate range, where the input is clamped to the original range.
        - `map` depending on `$Mapped$Type`:
            - `ABS` -> `abs`
            - `SQUARE` -> `square`
            - `CUBE` -> `cube`
            - `HALF_NEGATIVE` -> `halfNegative`
            - `QUARTER_NEGATIVE` -> `quarterNegative`
            - `INVERT` -> `reciprocal`
            - `SQUEEZE` -> `squeeze`
        - `$Ap2` replaced by `BinaryFunction`
        - `$BeardifierMarker`, `$BeardifierOrMarker` replaced by `beardifier`, `SimpleDensityFunction#BEARDIFIER`
        - `$BlendAlpha` replaced by `SimpleDensityFunction#BLEND_ALPHA`
        - `$BlendOffset` replaced by `SimpleDensityFunction#BLEND_OFFSET`
        - `$Clamp` replaced by `clamp`, `ClampFunction`
        - `$Constant` replaced by `ConstantFunction`
        - `$EndIslandDensityFunction` replaced by `EndIslandFunction`
        - `$FindTopSurface` replaced by `FindTopSurfaceFunction`
        - `$IntervalSelect` replaced by `IntervalSelectFunction`
        - `$Mapped` replaced by `UnaryFunction`
        - `$Marker`, `$MarkerOrMarked` replaced depending on `$Marker$Type`:
            - `Interpolated` -> `InterpolatedFunction`
            - `FlatCache`, `Cache2D`, `CacheOnce`, `CacheAllInCell` merged into `CacheFunction`, `DensityFunctionCompiler#reuseOrPrepareCache`
            - `BlendDensity` -> `blendDensity`, `BlendDensityFunction`
        - `$MulOrAdd` replaced by `BinaryFunction`
        - `$Noise` replaced by `NoiseFunction`
        - `$RangeChoice` replaced by `RangeChoiceFunction`
        - `$Shift` replaced by `ShiftNoiseFunction$Shift`
        - `$ShiftA` replaced by `ShiftNoiseFunction$ShiftA`
        - `$ShiftB` replaced by `ShiftNoiseFunction$ShiftB`
        - `$ShiftNoise` replaced by `ShiftNoiseFunction`
        - `$ShiftedNoise` replaced by `NoiseFunction`
        - `$Spline` replaced by `SplineFunction`
        - `$TwoArgumentSimpleFunction` replaced by `BinaryFunction`
        - `$YClampedGradient` replaced by `GradientFunction`
    - `GeodeBlockSettings` now takes in holders for the `BlockStateProvider`s instead of the raw values
    - `NoiseBasedChunkGenerator`
        - `getInterpolatedNoiseValue` is removed
        - `buildSurface` is now `private` from `public`
        - `applyCarvers` -> `generateCarvers`, now `private` from `public`
    - `NoiseChunk` no longer implements the `DensityFunction$FunctionContext`, `$ContextProvider`
        - `forChunk` is removed
        - The constructor no longer takes in the `int` cell XZ count, minimum `int` chunk XZ positions, `NoiseSettings`; and takes in the `Beardifier` instead of the `DensityFunctions$BeardificerOrMarker`, and the `DensityVolume`
        - `cachedClimateSampler` replaced by `cachingSamplers`, `NoiseRouter#createClimateSampler`; not one-to-one
        - `getInterpolatedState`, `getInterpolatedDensity`, `stopInterpolation` are removed
        - `maxPreliminarySurfaceLevel`, `preliminarySurfaceLevel` are removed
        - `initializeForFirstCellX`, `advanceCellX`, `selectCellYZ`, `updateForY`, `updateForX`, `updateForZ` are removed
        - `forIndex` is removed
        - `swapSlices` is removed
        - `cellWidth`, `cellHeight` are removed
        - `wrap` is removed
        - `volume` - The volume being operated upon.
    - `NoiseGeneratorSettings`
        - `surfaceRule` -> `materialRule`, now a `Holder<MaterialRule>` instead of a `SurfaceRules$RuleSource`
        - `spawnTarget` is now a list of `SpawnTargetPoint`s instead of `Climate$ParameterPoint`s
        - `aquifersEnabled` replaced by `aquifers`
        - `oreVeinsEnabled` merged into `materialRule`
        - `debugFunctions`, `$DebugFunctionEntry`, `$DebugFunctions` - Functions to display debug information about the noise functions.
        - `isAquifersEnabled`, `oreVeinsEnabled` are removed
    - `NoiseRouter` has been split into `NoiseRouter` and `Aquifer$Config`
        - `barrierNoise` -> `Aquifer$Config#barrierNoise`
        - `fluidLevelFloodednessNoise` -> `Aquifer$Config#fluidLevelFloodednessNoise`
        - `fluidLevelSpreadNoise` -> `Aquifer$Config#fluidLevelSpreadNoise`
        - `lavaNoise` -> `Aquifer$Config#lavaNoise`
        - `preliminarySurfaceLevel` split into `chunkSurfaceLevel`, and `Aquifer$Config#surfaceLevel`
        - `veinToggle` replaced by `OreVeinRule#density`, not one-to-one
        - `veinRidged` merged into `OreVeinRule#density`, not one-to-one
        - `veinGap` replaced by `OreVeinRule#fillerGap`, not one-to-one
        - `mapAll` is removed
    - `NoiseRouterData`
        - `CONTINENTS`, `EROSION`, `OFFSET`, `FACTOR`, `JAGGEDNESS`, `DEPTH` merged into `OVERWORLD_FUNCTIONS`
        - `AMPLIFIED_OVERWORLD_FUNCTIONS` - Functions for amplified world generation.
        - `LARGE_OVERWORLD_FUNCTIONS` - Functions for world generation with large biomes.
        - `ORE_VEIN_*` - Functions for ore vein settings
        - `bootstrap` no longer returns anything
        - `getFunction` is now `public` from `private`
        - `peaksAndValleys` is now `public` from `private`
        - `overworld` now takes in the `OverworldFunctionSet` instead of the `NormalNoise$NoiseParameters` and `boolean` world types
        - `overworldAquifers` - The aquifer config for the overworld.
        - `floatingIslands` now takes in a getter of `NormalNoise` instead of `NormalNoise$NoiseParameters`
        - `$QuantizedSpaghettiRarity#wrapRarity2d`, `wrapRarity3d` now take in a holder-wrapped `NormalNoise` instead of `NormalNoise$NoiseParameters`
    - `Noises`
        - Constants are now resource keys of `NormalNoise` instead of `NormalNoise$NoiseParameters`
        - `instantiate` no returns a `Noise` instead of `NormalNoise`
    - `NoiseSettings` no longer takes in the vertical and horizontal size
        - `create` no longer takes in the vertical and horizontal size
        - `getCellHeight`, `getCellWidth` are removed
    - `OreVeinifier` replaced by `OreVeinRule`
    - `OverworldFunctionSet` - A set of functions applied to construct the overworld.
    - `RandomState`
        - `create` now takes in the holder getter for `NormalNoise` instead of the `HolderGetter$Provider`, and the raw `NoiseGeneratorSettings` instead of a `ResourceKey`
            - The overload also takes in default values for using legacy randomization, the default `BlockState`, `int` sea level, and `NoiseRouter`
        - `samplersWithContext` - Creates a new sampler set with the given context.
        - `createClimateSampler` - Creates a climate sampler from the given context.
        - `getOrCreateNoise` now takes in a resource key of `NormalNoise` instead of `NormalNoise$NoiseParameters`, returning the `Noise` insetad of `NormalNoise`
        - `router` is removed
        - `sampler` replaced by `getSampler`, not one-to-one
        - `aquiferRandom`, `oreRandom` are removed
        - `sampleBlockValueUncached` - Samples the block value at the given position.
        - `acquireDensityBufferPool`, `releaseDensityBufferPool`, `garbageCollect` - Handles the density buffer management.
        - `seed` - The seed of the random state.
    - `SpawnTargetPoint` - A potential location to spawn the player, determined by its map of density function to climate parameters.
        - This is analogous to `Climate$ParameterPoint`
    - `SurfaceRules` -> `MaterialRules`, not one-to-one
        - `$ConditionSource` constants have been moved to `VanillaMaterialConditions`, now referenced by their `ResourceKey`
        - `registerAndWrap` - Registers a material rule and wraps it in a holder.
        - `getRule` - Gets a material rule and wraps it in a holder.
        - `getCondition` - Gets a material condition and wraps it in a holder.
        - `$AbovePreliminarySurface` -> `AbovePreliminarySurfaceCondition`
        - `$Bandlands` -> `BandlandsRule`
        - `$BiomeConditionSource` -> `BiomeCondition`
        - `$BlockRuleSource` -> `BlockRule`
        - `$Condition` -> `ConditionEvaluator`
        - `$ConditionSource` -> `MaterialCondition`
            - `bootstrap` -> `bootstrapConditions`
        - `$Context` -> `MaterialRuleContext`, not one-to-one
            - Constructor is now package-private from `protected`
            - `updateXZ`, `updateY` are now package-private from `protected`
            - `getSurfaceSecondary`, `getBiome`, `getMinSurfaceLevel` are now `public` from `protected`
            - `getNoiseSampler` is now `public` from `protected`
            - `getOrCreateRandomFactory` - Crates a random positional factory if not present for the identifier.
            - `getDensitiesInChunk` - Provides a getter to sample the density values for the given context.
            - `possibleBiomes` - The biomes that can be possibly selected.
            - `stoneDepthAbove`, `stoneDepthBelow`, `surfaceDepth` - Depth values for material locations.
            - `waterHeight` - The height of the water.
            - `blockX`, `blockY`, `blockZ`, `blockPos` - The current block position.
            - `surfaceGradientX`, `surfaceGradientZ` - The horizontal surface gradients.
            - `resolveAnchorY` - Resolves the anchor to a Y value.
            - `getBand` - Gets the band that should generate at the given position.
            - `$AbovePreliminarySurfaceCondition`
            - `$HoleCondition` -> Anonymous class in `HoleCondition`
            - `$SteepMaterialCondition` -> Anonymous class in `SteepCondition`
            - `$TemperatureHelperCondition` -> Lambda in `TemperatureCondition`
        - `$Hole` -> `HoleCondition`
        - `$LazyCondition` is removed
        - `$LazyXZCondition` -> `MaterialRuleContext$LazyXZCondition`
        - `$LazyYCondition` -> `MaterialRuleContext$LazyYCondition`
        - `$NoiseThresholdConditionSource` -> `NoiseThresholdCondition`
        - `$NotCondition` -> Lambda in `NotCondition`
        - `$NotConditionSource` -> `NotCondition`
        - `$RuleSource` -> `MaterialRule`
            - `bootstrap` -> `bootstrapRules`
            - `apply` -> `compile`
        - `$SequenceRule` -> Lambda in `SequenceRule`
        - `$SequenceRuleSource` -> `SequenceRule`
        - `$StateRule` -> `BlockRule`
        - `$Steep` -> `SteepCondition`
        - `$StoneDepthCheck` -> `StoneDepthCondition`
        - `$SurfaceRule` -> `RuleEvaluator`
        - `$Temperature` -> `TemperatureCondition`
        - `$TestRule` -> Lambda in `ConditionRule`
        - `$TestRuleSource` -> `ConditionRule`
        - `$VerticalGradientConditionSource` -> `VerticalGradientCondition`
        - `$WaterConditionSource` -> `WaterCondition`
        - `$YConditionSource` -> `YCondition`
    - `SurfaceSystem` -> `MaterialSystem`, not one-to-one
        - The constructor no longer takes in the preliminary surface `DensityFunction`
        - `buildSurface` no longer takes in the legacy random `boolean`
        - `topMaterial` now takes in the `WorldGenerationContext`, and a `DensitySamplerSet` instead of a `NoiseChunk`
        - `preliminarySurfaceFunction` - The density function used to generate the prelimiary surface.
    - `VerticalAnchor#relativeToSeaLevel`, `seaLevel`, `$RelativeToSeaLevel` - Gets a vertical anchor offset from the sea level.
    - `WorldGenerationContext`
        - `of` - Creates the context from the `LevelAccessor`.
        - `seaLevel` - The sea level of the world.
- `net.minecraft.world.level.levelgen.blending`
    - `Blender`
        - `CONTEXT_KEY`, `ALPHA_KEY`, `OFFSET_KEY` - Keys for the blender and density samplers.
        - `blendOffsetAndFactor` now has an overload that takes in a `DensityVolume` instead of the block XZ `int`s.
        - `blendDensity` now takes in the block XYZ `int`s instead of the `DensityFunction$FunctionContext`, a `float` instead of a `double` for the noise value, and returns a `float` instead of a `double`
        - `getCarvingFilter` - Returns the carving filter.
        - `addAroundOldChunksCarvingMaskFilter` -> `createAroundOldChunksCarvingMaskFilter`, now `private` from `public`
        - `$BlendingOutput` is now `protected` from `public`
        - `$OutputBuffer` - A wrapper that handles creating the samplers that outputs the blender data.
    - `BlendingData`
        - `NO_VALUE` is now a `float` instead of a `double`
        - `getHeight`, `getDensity` now returns `float`s instead of `double`s
        - `$DensityConsumer#consume` now takes in a `float` instead of a `double` for the density
        - `$HeightConsumer#consume` now takes in a `float` instead of a `double` for the height
        - `$Packede` now takes in an array of `float`s instead of `double`s for the heights
- `net.minecraft.world.level.levelgen.blockpredicates`
    - `BlockPredicate`
        - `test` now takes in a `LevelAccessor` instead of a `WorldGenLevel`
        - `matchesBlocks(List<Block>)` is removed
        - `matchesBlocks(Vec3i, Block...)` -> `matchesBlocks(Directional, Block...)`
        - `matchesTag` now hsa an overload that takes in a `Directional`
        - `matchesFluids(Vec3i, Fluid...)` -> `matchesBlocks(Directional, Fluid...)`
        - `matchesBiomes` is removed
        - `replaceable` can no longer take in any parameters
        - `wouldSurvive` now just takes in only the `Block`
        - `hasSturdyFace(Vec3i, Direction)` -> `matchesBlocks(Directional, Direction)`
        - `solid` now takes in a `Directional` instead of a `Vec3i`
        - `noFluid` no longer takes in any parameters
        - `unobstructed` no longer takes in any parameters
        - `heightRange` - Specifies the height must be within the following range.
        - `volumeMatch` - Specifies the block must match within the volume provided by two points.
    - `HeightRangePredicate` - A predicate that tests the block is between the `VerticalAnchor`s defining the Y range.
    - `VolumeMatchPredicate` - A predicate that tests all blocks within a volume matches the provided predicate.
    - `WouldSurvivePredicate` constructor is now `public` from `protected`
- `net.minecraft.world.level.levelgen.carver`
    - `*Configuration` classes are removed, merged onto the associated world carver
    - The world carvers are now commonly records that define the configuration data
    - `CanyonWorldCarver` now only takes in the `float` probaility, y `HeightProvider`, vertical rotation `FloatProvider`, and `$Shape`
        - `$CanyonShapeConfiguration` -> `$Shape`, now taking in the y scale `FloatProvider`
    - `CarverConfiguration` is removed
    - `CarverDebugSettings` is removed
    - `CarvingContext` is removed, usage replaced by `WorldGenerationContext`
    - `CaveWorldCarver` now takes in a count `IntProvider`, thickness `FloatProvider`, a `boolean` for whether to bias thickness for weirdness, and `FloatProvider` multipliers for the vertical room and starting verial radius
    - `ConfiguredWorldCarver` is removed, replaced by `WorldCarver`
    - `NetherWorldCarver` is removed
    - `WorldCarver` is now an `interface` from an abstract `class`
        - `WorldCarver` constants are now inlined within `WorldCarverTypes`
        - `Blockstate` constants are removed
        - `DIRECT_CODEC` - The direct codec for dispatching the carver from its type.
        - `CODEC` - The holder-wrapped entry.
        - `LIST_CODEC` - The holder set entry.
        - `liquids` is removed
        - `configured` is removed
        - `configuredCodec` replaced by `codec`
        - `carveEllipsoid` is now `static`, returning nothing, no longer taking in the `CarvingContext`, configuration, `ChunkAccess`, biome getter, and `Aquifer`, instead taking in the `ChunkPos` and `CarverOutput` instead of the `CarvingMask`
        - `carveBlock` is removed
        - `carve` now takes in a `WorldGenerationContext` instead of the `CarvingContext`, a `CarverOutput` instead of the `CarvingMask`, and the `ChunkPos` instead of the `CarvingContext`, configuration, `ChunkAccess`, biome getter, and `Aquifer`
        - `isStartChunk` no longer takes in the configuration
        - `canReach` is now `public` from `protected`
        - `$CarveSkipChecker#shouldSkip` no longer takes in the `CarvingContext`
- `net.minecraft.world.level.levelgen.densityfunction`
    - `CacheId` - An identifier for a cached density function, goes unused.
    - `CachingDensitySampler` - A density sampler that caches the sampled value for the specific volume.
    - `ContextBoundSampler` - A density sampler that uses the sampler referenced by the given `ContextKey`, defaulting to the provided fallback if not available.
    - `DensityBuffer` - A buffer for containing the sampled density values.
    - `DensityBufferArena` - An interface for managing the allocation and release of density buffers.
    - `DensityBufferPool` - A buffer arena holding some pool of density buffers, discarding any buffers older than a certain number of ticks.
    - `DensityFunctionCompiler` - A compiler for optimizing and turning `DensityFunction`s into `DensitySampler`s.
    - `DensitySampler` - A compiled function to sample the density values at a given position.
    - `DensitySamplerSet` - A `DensitySampler` getter given its function.
    - `DensityVolume` - A record defining some rectangular prism along with the step size when iterating through on each axis.
    - `DfRewriteRule` - An interface for rewriting density functions, typically for execution optimization.
    - `DistanceMetric` - The available metrics to define how distance should be measured.
    - `SamplerContext` - The context provided when a `DensitySampler` is sampling a value.
    - `ScopedDensityBuffer` - A density buffer that is scoped to some arena, keeping track of its current age in ticks.
    - `TilingMode` - A mode used for when sampling outside of a mapped range (e.g. gradient functions).
- `net.minecraft.world.level.levelgen.densityfunction.generator`
    - `DistancetoPointFunction` - Determines the distance to the provided point using the specified metric.
    - `SimpleDensityFunction` - An enum for defining a sampler constructed when the `NoiseChunk` was first initialized.
- `net.minecraft.world.level.levelgen.densityfunction.op`
    - `LerpFunction` - A function that linearly interpolates between two other functions, selecting a point based on an alpha function.
    - `PowFunction` - A function that raises the base to the sampled exponent.
    - `RoundFunction` - A function that rounds the sampled value to an `int` using the provided `$Type`.
    - `SliceFunction` - A function that samples the input along the given axis with the fixed coordinate.
- `net.minecraft.world.level.levelgen.feature`
    - Features are now commonly either records that contain the configuration data, or interfaces / abstract classes that define a common subtype
    - Feature clases typically provide a `MapCodec` field named `CODEC` that is used for registration
    - `AbstractHugeMushroomFeature` is now an `interface` from an abstract `class`
        - `placeTrunk` is now `public` from `protected`, no longer taking in the configuration
        - `placeMushroomBlock` is now `public` from `protected`
        - `getTreeHeight` is now `public` from `protected`
        - `isValidPosition` is now `public` from `protected`, no longer taking in the configuration
        - `getTreeRadiusForHeight` is now `public` from `protected`
        - `makeCap` is now `public` from `protected`
        - `capProvider` - The `BlockStateProvider` used to select the blocks for the mushroom cap.
        - `stemProvider` - The `BlockStateProvider` used to select the blocks for the mushroom stem.
        - `foliageRadius` - The radius of the mushroom cap.
        - `canPlaceOn` - Whether this feature can be placed on the below block.
    - `AbstractOreFeature` - An abstract implementation to place ore.
    - `BasaltColumnsFeature` replaced by `SteppedColumnClusterFeature`, with cluster reach and column count more randomized with `WeightedRandomSelectorFeature`
    - `BasaltPillarFeature` replaced by `OverlayFeature` merging `SingleBlockPillarFeature`, sometimes with `ProjectedRandomPatchySquare` and `SimpleBlockFeature`
        - Main replacement would be `SingleBlockPillarFeature`, followed by `OverlayFeature`
        - `layer` now has an overload that takes in a holder-wrapped `BlockStateProvider` instead of the raw value
    - `ConfiguredFeature` record is removed, replaced by `Feature`
        - `getSubFeatures` -> `Feature#getSubFeatures`, now a stream of holder-wrapped `Feature`s
    - `CoralClawFeature` now takes in a holder-wrapped `PlacedFeature` to use instead of generating the coral block itself
    - `CoralFeature` class is removed
    - `CoralMushroomFeature` class is removed, replaced by an `OverlayFeature` of `SimpleBlockFeature`s
    - `CoralTreeFeature` now takes in a holder-wrapped `PlacedFeature` to use instead of generating the coral block itself
    - `CuboidPlacement` - Places the feature in the form of a cuboid, placing against the edges or interior depending on the set `boolean`s.
    - `DesertWellFeature` replaced by an `OverlayFeature` of `TemplateFeature`s
    - `EndPodiumFeature#getLocation` is removed
    - `EndSpikeFeature#NUMBER_OF_SPIKES` is now `private` from `public`
    - `FallenTreeFeature#builder` - Creates the builder for the fallen tree.
    - `Feature` is now an `interface` from an abstract `class`
        - `Feature` constants are now inlined within `FeatureTypes`
        - `DIRECT_CODEC` - The direct codec for dispatching the feature from its type.
        - `CODEC` - The holder-wrapped entry.
        - `LIST_CODEC` - The holder set entry.
        - `configuredCodec` replaced by `codec`
        - `isReplaceable` is removed
        - `safeSetBlock` is now `public` from `protected`
        - `place` now takes in the `WorldGenLevel`, `ChunkGenerator`, `RandomSource`, and origin `BlockPos`
            - All other `place` methods are removed
        - `checkNeighbors` -> `AbstractOreFeature#checkNeighbors`
        - `isAdjacentToAir` -> `AbstractOreFeature#isAdjacentToAir`
        - `markAboveForPostProcessing` is now `public` from `protected`
    - `FeatureCountTracker#featurePlaced` now takes in a `Feature` instead of a `ConfiguredFeature`
    - `FeaturePlaceContext` class is removed
    - `FeatureTypes` - All vanilla feature types.
    - `FossilFeatureConfiguration` class is removed
    - `GlowstoneFeature` replaced by `RandomNeighborSpreadFeature`
    - `HugeFungusConfiguration` class is removed
    - `KelpFeature` replaced by `BlockColumnFeature`
    - `LakeFeature$Configuration` merged onto `LakeFeature`
    - `MultifaceGrowthFeature#placeGrowthIfPossible` is no longer `static`, and no longer takes in the configuration
    - `NetherForestVegetationFeature` replaced by `SimpleBlockFeature` with some additional `PlacementModifier`s for determining spread
    - `OreFeature` now extends `AbstractOreFeature`
        - `doPlace` no longer takes in the configuration
        - `canPlaceOre` -> `AbstractOreFeature#canPlaceOre`, no longer `static`, no longer taking in the configuration
        - `shouldSkipAirCheck` -> `AbstractOreFeature#shouldSkipAirCheck`, now `private` from `protected`
    - `OverlayFeature` - A feature that places `PlacedFeature`s, returning success if any feature was placed.
        - OR variant of `SequenceFeature`
    - `ProjectedRandomPatchySquare` - A feature that spawns a square, where each block has a probability to spawn, by projecting downwards until hitting a block it cannot go through or the max height.
    - `RandomNeighborSpreadFeature` - A feature that places the provided block at the sampled position for n attempts as long as there is one accepted neighbor.
    - `ScatteredOreFeature` now extends `AbstractOreFeature`
    - `SculkPatchFeature` no longer takes in the `IntProvider` for extra rare growths and the associated `float` catalyst chance
    - `SeagrassFeature` replaced by a `WeightedRandomSelectorFeature` of `SimpleBlockFeature`s
    - `SeaPickleFeature` replaced by `SimpleBlockFeature`
    - `SingleBlockPillarFeature` - A feature that places a pillar in the associated direction, optionally providing a cap.
    - `SteppedColumnClusterFeature` - A feature that places a cluster of columns upwards.
    - `TemplateFeature` now takes in an optional holder-wrapped `StructureProcessorList` to apply
    - `TreeFeature` `BlockStateProvider`s are now holder-wrapped
    - `TwistingVinesFeature` replaced by `BlockColumnFeature`
    - `VegetationPatchFeature` is still a `class`
        - All instance fields are `protected` from `public`
        - `groundState` is now holder-wrapped
        - `placeGroundPatch` no longer takes in the configuration
        - `distributeVegetation` is now `private` from `protected`
        - `placeVegetation` no longer takes in the configuration
        - `placeGround` is now `private` from `protected`
    - `WeepingVinesFeature` replaced by an `OverlayFeature` of `RandomNeighborSpreadFeature` and `BlockColumnFeature`
- `net.minecraft.world.level.levelgen.feature.configurations` is removed
    - All `*Configuration` classes are removed, merged onto their associated feature
    - `OreConfiguration`
        - `target` -> `BlockReplacement#replace`
        - `$TargetBlockState` -> `BlockReplacement`
    - `FallenTreeConfiguration$FallenTreeConfigurationBuilder` -> `FallenTreeFeature$Builder`
    - `TreeConfiguration$TreeConfigurationBuilder` -> `TreeFeature$Builder`
    - `VegetationPatchConfiguration#CODEC` -> `VegetationPatchFeature#makeCodec`, not one-to-one
- `net.minecraft.world.level.levelgen.feature.foliageplacers`
    - `FoliagePlacer`
        - `createFoliage` now takes in the `TreeFeature` instead of the configuration
        - `foliageHeight` now takes in the `TreeFeature` instead of the configuration
        - `placeLeavesRow` now takes in the `TreeFeature` instead of the configuration
        - `placeLeavesRowWithHangingLeavesBelow` now takes in the `TreeFeature` instead of the configuration
        - `tryPlaceExtension` now takes in the `TreeFeature` instead of the configuration
        - `tryPlaceLeaf` now takes in the `TreeFeature` instead of the configuration
        - `$FoliageAttachment` is now a record
            - The constructor no longer takes in a `boolean` for the double trunk, instead taking in an `int` foliage height offset, and the XZ size `int`s
            - `radiusOffset` -> `radiusOffsetXZ`
    - `PoplarFoliagePlacer` - The foliage placer for poplar trees.
- `net.minecraft.world.level.levelgen.feature.rootplacers`
    - `AboveRootPlacement` now takes in a holder-wrapped `BlockStateProvider` instead of the raw value
    - `MangroveRootPlacement` now takes in a holder-wrapped `BlockStateProvider` instead of the raw value
    - `MangroveRootPlacer` now takes in a holder-wrapped `BlockStateProvider` instead of the raw value
    - `RootPlacer` now takes in a holder-wrapped `BlockStateProvider` instead of the raw value
        - `rootProvider` is now a holder-wrapped `BlockStateProvider` instead of the raw value
        - `placeRoots` now takes in the `TreeFeature` instead of the configuration
        - `placeRoot` now takes in the `TreeFeature` instead of the configuration
- `net.minecraft.world.level.levelgen.feature.stateproviders`
    - `BlockStateProvider` is now an `interface` from an abstract `class`
        - `CODEC` -> `TYPED_CODEC`
            - `CODEC` is now the holder-wrapped provider
        - `STATE_OR_PROVIDER_CODEC` - Takes in either a `BlockState` or `BlockStateProvider`.
        - `DIRECT_CODEC` - The datapack registry codec.
        - `simple` -> `of`
        - `holderOf` - Returns the holder-wrapped provider from the block.
        - `type` replaced by `codec`, now returning a `MapCodec`
        - `getState`, ``getOptionalState`` now take in a `LevelAccessor` instead of the `WorldGenLevel`
    - `BlockStateProviderType` class is removed
        - Constants are now inlined in `BlockStateProviderTypes`
        - `codec` -> `BlockStateProvider#codec`
    - `BlockStateProviderTypes` - Registers all vanilla block state providers.
    - `CopyPropertiesProvider` - A provider that copies the properties of the block at the placed position.
    - `DualNoiseProvider` now take in `NormalNoise`s instead of `NormalNoise$NoiseParameters` for the parameters
        - `getSlowNoiseValue` now returns a `float` from a `double`
    - `NoiseBasedStateProvider` now takes in `NormalNoise` instead of `NormalNoise$NoiseParameters` for the parameters
        - `noiseCodec` now returns a `P3` containing `NormalNoise` instead of `NormalNoise$NoiseParameters`
        - `parameters` is now `NormalNoise` instead of `NormalNoise$NoiseParameters`
        - `noise` is now `Noise` instead of `NormalNoise`
        - `getNoiseValue` now returns a `float` from a `double`
    - `NoiseProvider` now takes in `NormalNoise` instead of `NormalNoise$NoiseParameters` for the parameters
        - `noiseProviderCodec` now returns a `P4` containing `NormalNoise` instead of `NormalNoise$NoiseParameters`
        - `getRandomState` now takes in a `float` from a `double`
    - `NoiseThresholdProvider` now takes in `NormalNoise` instead of `NormalNoise$NoiseParameters` for the parameters
    - `RandomBlockProvider` - A provider that selects a random block from the given set.
    - `RandomizedIntStateProvider` now takes in a holder-wrapped `BlockStateProvider` instead of the raw value
    - `RotatedBlockProvider` is now a record
        - The constructor now takes in a holder-wrapped `BlockStateProvider` instead of a `Block`, and an optional `Direction` to face
    - `RuleBasedStateProvider` is now a record
        - The constructor now takes in a holder-wrapped `BlockStateProvider` instead of the raw value
        - `$Rule` now takes in a holder-wrapped `BlockStateProvider` instead of the raw value
    - `SimpleStateProvider` is now a record
    - `WeightedStateProvider` is now a record
- `net.minecraft.world.level.levelgen.feature.treedecorators`
    - `AlterGroundDecorator` now takes in a holder-wrapped `BlockStateProvider` instead of the raw value
    - `AttachedToLeavesDecorator` now takes in a holder-wrapped `BlockStateProvider` instead of the raw value
    - `AttachedToLogsDecorator` now takes in a holder-wrapped `BlockStateProvider` instead of the raw value
    - `PlaceOnGroundDecorator` now takes in a holder-wrapped `BlockStateProvider` instead of the raw value
    - `ShelfMushroomDecorator` - A "tree" decorator for the shelf mushroom.
    - `TreeDecorator$Context`
        - `isReplaceable` - Checks whether the block at the given position can be replaced.
        - `isWaterOrWaterNearby` - Checks if the given position or any adjacent positions are water blocks.
- `net.minecraft.world.level.levelgen.feature.trunkplacers`
    - `PoplarTrunkPlacer` - A trunk placer for the poplar tree.
    - `TrunkPlacer`
        - `placeTrunk` now takes in the `TreeFeature` instead of the configuration
        - `placeBelowTrunkBlock` now takes in the `TreeFeature` instead of the configuration
        - `placeLog` now takes in the `TreeFeature` instead of the configuration
        - `placeLogIfFree` now takes in the `TreeFeature` instead of the configuration
- `net.minecraft.world.level.levelgen.material.MaterialRuleList` record is removed
- `net.minecraft.world.level.levelgen.placement`
    - All placement modifier implementations are now generally records
    - `FeaturePlacer` - A helper for spawning a given `PlacedFeature` into the world
    - `PlacedFeature` now takes in a holder-wrapped `Feature` instead of a `ConfiguredFeature`
        - `placeWithBiomeCheck` -> `FeaturePlacer#placeWithBiomeCheck`
        - `getFeatures` now returns a stream of holder-wrapped `Feature`s instead of a `ConfiguredFeature`s
    - `PlacementContext#getCarvingMask` is removed
    - `PlacementFilter` is now an `interface` from an abstract `class`
    - `PlacementModifier` is now an `interface` from an abstract `class`
        - `getPositions` -> `modify`, now taking in a `Consumer<BlockPos>` for the placements to copy rather than returning the stream of `BlockPos`es
        - `type` replaced by `codec`
    - `PlacementModifierType` is removed
        - All constants are now inlined in `PlacementModifierTypes#bootstrap`
        - `codec` -> `PlacementModifier#codec`
    - `PlacementModifierTypes` - Registers all vanilla placement modifier types.
    - `RandomChancePlacement` - A placement that provides a probability of placing the feature.
        - Similar to `RarityFilter`, though chance there is checked against the reciprocal.
    - `RandomlySelectedPlacement` - A placement modifier that randomly selects a modifier to apply from some list.
    - `RandomOffsetPlacement` -> `OffsetPlacement`, now separating the XZ spread into their own `IntProvider`s
        - `of` now has overloads that can take in the XYZ constant `int` spread, or a `Direction`
        - `above` - Creates an offset of one block above the given position.
    - `RepeatingPlacement` is now an `interface` from an abstract `class`
- `net.minecraft.world.level.levelgen.structure`
    - `BoundingBox#intersects` now has an overload that can specify the `int` min and max XYZ bounds
    - `Structure`
        - `generate` now takes in the `Climate$Sampler`
        - `onTopOfChunkCenter` -> `onTopOfChunkCenterWithoutBiomeCheck`
            - Original `onTopOfChunkCenter` now checks whether a biome could exist on top of the chunk center and, if `false`, returns an empty optional
        - `getLowestYIn5by5BoxOffset7Blocks` -> `getLowestYIn5by5Box`, now taking in the bloxk XZ `int`s
        - `$GenerationContext` now takes in the `Climate$Sampler` and `BiomeResolver`
            - `isValidBiome` - Checks whether the given stub is in a valid biome.
            - `couldStructureExistInColumn` - Checks whether a structure could exist in the block column due to a valid biome.
            - `couldValidBiomeExistInTerrainColumn` - Checks whether a valid biome could exist in the column defined by the horizontal position.
            - `couldValidBiomeExistOnTopOfChunkCenter` - Checks whether a valid biome could exist in the column at the chunk center.
    - `StructurePiece#addChildren` now takes in the `StructurePiecesBuilder` instead of the `StructurePieceAccessor`
    - `StructurePieceAccessor` interface is removed
- `net.minecraft.world.level.levelgen.structure.pieces`
    - `PieceGenerator` interface is removed
    - `PieceGeneratorSupplier` interface is removed
    - `StructurePiecesBuilder` no longer implements `StructurePieceAccessor`
- `net.minecraft.world.level.levelgen.structure.placement`
    - `ConcentricRingsStructurePlacement` now extends `AbstractSpreadingStructurePlacement`
    - `DimensionOriginStructurePlacement` - A placement that only allows the structure to spawn at the dimension's origin.
    - `RandomSpreadStructurePlacement` now extends `AbstractSpreadingStructurePlacement`
    - `StructurePlacement` is now an `interface` from an abstract `class`
        - Original class implementation moved to `AbstractSpreadingStructurePlacement`
        - `isStructureChunk`, `applyAdditionalChunkRestrictions`, `getLocatePos`, `locateOffset` remain as defined methods
        - `type` replaced by `codec`
    - `StructurePlacement` - Registers all vanilla structure placement types.
    - `StructurePlacementType` interface is removed
        - All constants have been inlined in `StructurePlacements#bootstrap`
        - `codec` -> `StructurePlacement#codec`
- `net.minecraft.world.level.levelgen.structure.structures`
    - `IglooPieces#addPieces` now takes in the `StructurePiecesBuilder` instead of the `StructurePieceAccessor`
    - `MineshaftPieces`
        - `$MineShaftCorridor#findCorridorSize` now takes in the `StructurePiecesBuilder` instead of the `StructurePieceAccessor`
        - `$MineShaftCrossing#findCrossing` now takes in the `StructurePiecesBuilder` instead of the `StructurePieceAccessor`
        - `$MineShaftStairs#findStairs` now takes in the `StructurePiecesBuilder` instead of the `StructurePieceAccessor`
    - `NetherFortressPieces`
        - `$BridgeCrossing#createPiece` now takes in the `StructurePiecesBuilder` instead of the `StructurePieceAccessor`
        - `$BridgeEndFiller#createPiece` now takes in the `StructurePiecesBuilder` instead of the `StructurePieceAccessor`
        - `$BridgeStraight#createPiece` now takes in the `StructurePiecesBuilder` instead of the `StructurePieceAccessor`
        - `$CastleCorridorStairsPiece#createPiece` now takes in the `StructurePiecesBuilder` instead of the `StructurePieceAccessor`
        - `$CastleCorridorTBalconyPiece#createPiece` now takes in the `StructurePiecesBuilder` instead of the `StructurePieceAccessor`
        - `$CastleEntrance#createPiece` now takes in the `StructurePiecesBuilder` instead of the `StructurePieceAccessor`
        - `$CastleSmallCorridorCrossingPiece#createPiece` now takes in the `StructurePiecesBuilder` instead of the `StructurePieceAccessor`
        - `$CastleSmallCorridorLeftTurnPiece#createPiece` now takes in the `StructurePiecesBuilder` instead of the `StructurePieceAccessor`
        - `$CastleSmallCorridorPiece#createPiece` now takes in the `StructurePiecesBuilder` instead of the `StructurePieceAccessor`
        - `$CastleSmallCorridorRightTurnPiece#createPiece` now takes in the `StructurePiecesBuilder` instead of the `StructurePieceAccessor`
        - `$CastleStalkRoom#createPiece` now takes in the `StructurePiecesBuilder` instead of the `StructurePieceAccessor`
        - `$MonsterThrone#createPiece` now takes in the `StructurePiecesBuilder` instead of the `StructurePieceAccessor`
        - `$NetherBridgePiece`
            - `generateChildForward` now takes in the `StructurePiecesBuilder` instead of the `StructurePieceAccessor`
            - `generateChildLeft` now takes in the `StructurePiecesBuilder` instead of the `StructurePieceAccessor`
            - `generateChildRight` now takes in the `StructurePiecesBuilder` instead of the `StructurePieceAccessor`
        - `$RoomCrossing#createPiece` now takes in the `StructurePiecesBuilder` instead of the `StructurePieceAccessor`
        - `$StairsRoom#createPiece` now takes in the `StructurePiecesBuilder` instead of the `StructurePieceAccessor`
    - `NetherFossilPieces#addPieces` now takes in the `StructurePiecesBuilder` instead of the `StructurePieceAccessor`
    - `OceanRuinPieces#addPieces` now takes in the `StructurePiecesBuilder` instead of the `StructurePieceAccessor`
    - `ShipwreckPieces#addRandomPiece` now takes in the `StructurePiecesBuilder` instead of the `StructurePieceAccessor`
    - `StrongholdPieces`
        - `$ChestCorridor#createPiece` now takes in the `StructurePiecesBuilder` instead of the `StructurePieceAccessor`
        - `$FillerCorridor#findPieceBox` now takes in the `StructurePiecesBuilder` instead of the `StructurePieceAccessor`
        - `$FiveCrossing#createPiece` now takes in the `StructurePiecesBuilder` instead of the `StructurePieceAccessor`
        - `$LeftTurn#createPiece` now takes in the `StructurePiecesBuilder` instead of the `StructurePieceAccessor`
        - `$Library#createPiece` now takes in the `StructurePiecesBuilder` instead of the `StructurePieceAccessor`
        - `$PortalRoom#createPiece` now takes in the `StructurePiecesBuilder` instead of the `StructurePieceAccessor`
        - `$PrisonHall#createPiece` now takes in the `StructurePiecesBuilder` instead of the `StructurePieceAccessor`
        - `$RightTurn#createPiece` now takes in the `StructurePiecesBuilder` instead of the `StructurePieceAccessor`
        - `$RoomCrossing#createPiece` now takes in the `StructurePiecesBuilder` instead of the `StructurePieceAccessor`
        - `$StairsDown#createPiece` now takes in the `StructurePiecesBuilder` instead of the `StructurePieceAccessor`
        - `$Straight#createPiece` now takes in the `StructurePiecesBuilder` instead of the `StructurePieceAccessor`
        - `$StraightStairsDown#createPiece` now takes in the `StructurePiecesBuilder` instead of the `StructurePieceAccessor`
        - `$StrongholdPiece`
            - `generateSmallDoorChildForward` now takes in the `StructurePiecesBuilder` instead of the `StructurePieceAccessor`
            - `generateSmallDoorChildLeft` now takes in the `StructurePiecesBuilder` instead of the `StructurePieceAccessor`
            - `generateSmallDoorChildRight` now takes in the `StructurePiecesBuilder` instead of the `StructurePieceAccessor`
- `net.minecraft.world.level.levelgen.structure.templatesystem`
    - `AllOfRuleTest` - A test that checks if all rules return `true`, equivalent to AND.
    - `AnyOfRuleTest` - A test that checks if any rule return `true`, equivalent to OR.
    - `HeightMatchTest` - A test that checks whether the position is within the height bounds.
    - `NotRuleTest` - A test that inverts the result of the given test, equivalent to NOT.
    - `RuleTest`
        - `test` now takes in a `BlockPos`
        - `allOf` - Checks if all rules return `true`, equivalent to AND.
        - `anyOf` - Checks if any rule return `true`, equivalent to OR.
        - `not` - Inverts the result of the given test, equivalent to NOT.
        - `either` - Uses the provided test depending on the result of the condition test, equivalent to IF/ELSE.
    - `StructureTemplate$JigsawBlockInfo` now takes in a `BlockPos` and `BlockState` instead of the `$StructureBlockInfo`
        - `of` -> `parse`
        - `withInfo` now takes in a `BlockPos` and `BlockState` instead of the `$StructureBlockInfo`
- `net.minecraft.world.level.levelgen.synth`
    - `BlendedNoise` is now a record that implements `DensityFunction` instead of `DensityFunction$SimpleFunction`
        - `DATA_CODEC` -> `CODEC`, now `public` from `private`
            - The original `CODEC` has been removed
        - The constructor no longer takes in the `RandomSource`
        - `withNewRandom` replaced by `createFbmSet`, returning a `$FbmSet`
        - `createFbm` - Creates noise into layers for fractional brownian motion.
        - `compileSampler` now has an overload that takes in the `RandomSource`
        - `parityConfigString` -> `$FbmSet#parityConfigString`
        - `$FbmSet` - A record containing the blended noise using fractional brownian motion.
    - `GradientNoise` - An abstract noise implementation that generates a gradient of values.
    - `ImprovedNoise` replaced by `PerlinNoise`, `SmearedPerlinNoise`; not one-to-one
        - `noiseWithDerivate` -> `PerlinNoise#noiseWithDerivative`
        - `sampleAndLerp` -> `PerlinNoise#sampleAndLerp`, now `private` from `public`
    - `Noise` - An interface designed to sample a noise function for some position.
    - `NoiseStack` - Layers different noises on top of each other, usualy with differing frequency and amplitudes.
    - `NormalNoise` is now `final`
        - `create` replaced by `NormalNoise$builder`
            - An overload exists that takes in a `RandomSource` and returns a `Noise`
        - `maxValue` merged into `range`
        - `getValue` are removed
        - `parameters` is removed
        - `parityConfigString` now takes in the `RandomSource`
        - `$Builder` - A builder to construct the normal noise to apply.
        - `$Normalization` - How to apply normalization to the noise.
        - `$NoiseParameters` replaced by `$Parameters`, now `private` from `public`
            - `DIRECT_CODEC` replaced by `NormalNoise#DIRECT_CODEC`
            - `CODEC` replaced by `NormalNoise#CODEC`
                - The original `CODEC` is now used to store the new `$Parameters` values
    - `PerlinNoise` now extends `GradientNoise`
        - `createLegacyForLegacyNetherBiome`, or the original implementation of `PerlinNoise`, replaced by `LegacyFbmInitializer#createForLegacyNetherBiome`, `NormalNoise#createForLegacyNetherBiome`
        - `create` methods are removed
        - The constructor no longer takes in the `Pair<Integer, DoubleList>` and `boolean` for whether to initialize with a new position
        - `maxValue` merged into `range`
        - `getValue` replaced by `get`, now returning a `float`
        - `maxBrokenValue` is removed
        - `getOctaveNoise`, `wrap`, `firstOctave`, `amplitudes` are removed
    - `PerlinSimplexNoise` replaced by `SimplexNoise`
    - `SimplexNoise` split into its super implementation `GradientNoise`, and `SimplexNoise`
        - `GRADIENT` -> `GradientNoise#GRADIENT`, now an array of `GradientNoise$Gradient`
        - `RANGE` - The range of values that can be generated.
        - `STANDARD_DEVIATION` - The standard deviation of the noise value.
        - The constructor no longer takes in a `boolean` to discard the noise offset
        - `dot` -> `GradientNoise$Gradient#dot`, no longer taking in the `int[]`
        - `getValue` replaced by `get`, now returning a `float`
    - `SmearedPerlinNoise` - A perlin noise implementation that fudges the scaling in the Y direction.
- `net.minecraft.world.level.storage.loot`
    - `ContainerComponentManipulator` is now a `record` instead of an `interface`
        - `setContents(T, Stream<ItemStack>)` is removed
        - `getContents` is removed
    - `FloatRangePredicate` - A predicate that checks whether the float is on the line, or matches the point.
    - `IntRange` split into `IntLimit` for the limiter, and `IntRangePredicate` for the checker
        - Both now extend / implement `Validatable` instead of `LootContextUser`
    - `LootContext`
        - `getParameter` is removed
        - `getOptionalParameter` -> `getOptional`
        - `createVisitedEntry` now has an overload that takes in and returns a `SlotSource`
    - `LootContextArg`
        - `of` - Constructs an argument of a `ContextKey`.
        - `$ArgCodecBuilder#or` - Adds an argument to choose from.
    - `LootDataType` no longer takes in a `Codec`
        - `SLOT_SOURCE` - Slot source to choose from.
        - `FLOAT_PROVIDER` - Float providers to sample a value.
        - `INT_PROVIDER` - Integer providers to sample a value.
        - `runValidation` now has an overload that takes in a `HolderLookup$Provider` instead of a `HolderLookup`
        - `runValidationIfPresent` - Runs the validation if the registry is present.
    - `LootPool$Builder`
        - `setRolls` now takes in a holder-wrapped `ContextIntProvider` instead of a `NumberProvider`
        - `setBonusRolls` now takes in a holder-wrapped `ContextFloatProvider` instead of a `NumberProvider`
    - `LootTable#LIST_CODEC` - The holder set codec.
    - `Validatable#validateReference` replaced by `validateHolder`, `validateHolderSet`, now taking in the `String` name and either `Holder`s or `HolderSet`s
    - `ValidationContext`
        - `enterTag` - Enters into a tag object.
        - `hasVisitedTag` - If the tag has already been visited.
        - `allowsReferences` is removed
        - `$MissingReferenceProblem` is removed
        - `$MinBoundsProblem` - A problem where the backing list must contain at least the given number of elements.
        - `$RecursiveReferenceProblem` split into `$RecursiveElementReferenceProblem`, `$RecursiveTagReferenceProblem` 
        - `$ReferenceNotAllowedProblem` is removed
- `net.minecraft.world.level.storage.loot.entries`
    - Subtypes of `LootPoolEntryContainer` now take in an optional holder-wrapped `LootItemCondition` and an optional holder-wrapped `LootItemFunction` instead of other condition and function variations
    - `AlternativesEntry$Builder` now implements `CompositeEntryBase$Builder` instead of `LootPoolEntryContainer$Builder`
    - `CompositeEntryBase`
        - `$Builder` - An abstract builder for creating a composite subtype.
        - `$CompositeEntryConstructor#create` now takes in an optional holder-wrapped `LootItemCondition` instead of a list of `LootItemCondition`s, and an optional holder-wrapped `LootItemFunction`
    - `DynamicLoot` now extends `SingleEntryContainerBase`
        - `dynamicEntry` now returns a `UniformContainerBase$Builder`
    - `EmptyLootItem` now extends `SingleEntryContainerBase`
        - `emptyItem` now returns a `UniformContainerBase$Builder`
    - `EntryGroup$Builder` now implements `CompositeEntryBase$Builder` instead of `LootPoolEntryContainer$Builder`
    - `ExpandableContainerBase` - A uniform container that can expand its data into separate consumed values or given in a single entry, typically for use with pools that reference a range of objects.
    - `LootItem` now extends `SingleEntryContainerBase`
        - `lootTableItem` now returns a `UniformContainerBase$Builder`
    - `LootPoolEntryContainer` now takes in an optional holder-wrapped `LootItemCondition` and an optional holder-wrapped `LootItemFunction` instead of other condition and function variations
        - `modifier` - The function to apply to the items.
        - `commonFields` now returns a `P2` containing an optional holder-wrapped `LootItemCondition` and an optional holder-wrapped `LootItemFunction`
        - `canRun` is now `private` from `protected`
        - `expand` is now `final`
            - Use `expandRaw` instead, which is called after making sure the container can run
        - `$Builder` now implements `FunctionUserBuilder`
            - `getConditions` -> `getCondition`, now returning an optional holder-wrapped `LootItemCondition`.
            - `getModifier` - Gets the function modifier.
    - `LootPoolSingletonContainer` split into `UniformContainerBase` and `SingleEntryContainerBase` depending on usage
    - `NestedLootTable` now extends `ExpandableContainerBase`
        - `INLINE_LOOT_TABLE_PATH_ELEMENT` is removed
        - `lootTableReference` now returns a `UniformContainerBase$Builder`
    - `SequentialEntry$Builder` now implements `CompositeEntryBase$Builder` instead of `LootPoolEntryContainer$Builder`
    - `SingleEntryContainerBase` - A uniform container that outputs a single entry.
    - `SlotLoot` now extends `SingleEntryContainerBase`
    - `TagEntry` now extends `ExpandableContainerBase`
        - `tagContents`, `expandTag` now take in an item `HolderSet` instead of a `TagKey`, returning a `UniformContainerBase$Builder`
    - `UniformContainerBase` - A container whose entries can be weighted, affected by the quality scalar of luck.
- `net.minecraft.world.level.storage.loot.functions`
    - Subtypes of `LootItemConditionalFunction` now take in an optional holder-wrapped `LootItemCondition` instead of some other condition variation
    - `EnchantedCountIncreaseFunction`
        - `lootingMultiplier` now takes in an enchantment `HolderGetter` instead of the `HolderLookup$Provider`, and a holder-wrapped `ContextIntProvider` instead of a `NumberProvider`
        - `$Builder` now takes in a holder-wrapped `ContextIntProvider` instead of a `NumberProvider`
    - `EnchantRandomlyFunction#randomApplicableEnchantment` now takes in an enchantment `HolderGetter` instead of the `HolderLookup$Provider`
    - `EnchantWithLevelsFunction`
        - `enchantWithLevels` now takes in an enchantment `HolderGetter` instead of the `HolderLookup$Provider`, and a holder-wrapped `ContextIntProvider` instead of a `NumberProvider`
        - `$Builder` now takes in a holder-wrapped `ContextIntProvider` instead of a `NumberProvider`
    - `ExplorationMapFunction`
        - `DEFAULT_DESTINATION` is removed
        - `makeExplorationMap`, `$Builder` now takes in a structure `HolderSet`
        - `$Builder#setDestination` is merged into the constructor
    - `FilteredFunction$Builder#onPass`, `onFail` now take in the raw `LootItemFunction` instead an optional-wrapped one
    - `FunctionReference` class is removed
    - `FunctionUserBuilder`
        - `apply` now has an overload that takes in a holder-wrapped `LootItemFunction` instead of the `LootItemFunction$Builder`
        - `buildFunction` - Builds a function from a list of functions.
    - `LimitCount#limitCount` now takes in an `IntLimit` instead of an `IntRange`
    - `LootItemConditionalFunction` now takes in an optional holder-wrapped `LootItemCondition` instead of a list of `LootItemCondition`s
        - `commonFields` now returns a `P1` containing an optional holder-wrapped `LootItemCondition` instead of a list of `LootItemCondition`s
        - `simpleBuilder` now takes in a function with an input of an optional holder-wrapped `LootItemCondition` instead of a list of `LootItemCondition`s
        - `$BuildergetConditions` -> `getCondition`, now an optional holder-wrapped `LootItemCondition` instead of a list of `LootItemCondition`s
        - `$DummyBuilder` now takes in a function with an input of an optional holder-wrapped `LootItemCondition` instead of a list of `LootItemCondition`s
    - `LootItemFunction#decorate` now takes in an optional holder-wrapped `LootItemFunction` instead of a `BiFunction`
    - `LootItemFunctions`
        - `IDENTITY` -> `SequenceFunction#IDENTITY`, now `private` from `public`
        - `ROOT_CODEC` -> `DIRECT_CODEC`
        - `LIST_CODEC` - The holder set codec.
        - `compose` replaced by `FunctionUserBuilder#buildFunction`
    - `SequenceFunction` now extends `LootItemConditionalFunction`
        - The constructor is now `public` from `private`
        - `canUseInlineCodec` - Whether the inline codec can be used for this sequence
        - `of` now takes in a list of holder-wrapped `LootItemFunction`s instead of the raw value
        - `apply` -> `run`
    - `SetAttributesFunction#modifier`, `$ModifierBuilder` now take in a holder-wrapped `ContextFloatProvider` instead of a `NumberProvider`
    - `SetContainerLootTable`
        - `ID_ONLY_CODEC` - A codec that only takes in a loot table by its id.
        - `withLootTable` now takes in a loot table `Holder$Reference` instead of the `BlockEntityType` with the loot table `ResourceKey`
    - `SetCustomModelDataFunction` now takes holder-wrapped `ContextFloatProvider`s for the floats, and holder-wrapped `ContextIntProvider`s for the colors
    - `SetEnchantmentsFunction$Builder#withEnchantment` now takes in a holder-wrapped `ContextIntProvider` instead of a `NumberProvider`
    - `SetItemCountFunction#setCount` now takes in a holder-wrapped `ContextIntProvider` instead of a `NumberProvider`
    - `SetItemDamageFunction#setDamage` now takes in a holder-wrapped `ContextFloatProvider` instead of a `NumberProvider`
    - `SetOminousBottleAmplifierFunction#setAmplifier` now takes in a holder-wrapped `ContextIntProvider` instead of a `NumberProvider`
    - `SetRandomDyesFunction#withCount` now takes in a holder-wrapped `ContextIntProvider` instead of a `NumberProvider`
    - `SetStewEffectFunction$Builder#withEffect` now takes in a holder-wrapped `ContextIntProvider` instead of a `NumberProvider`
        - An overload also allows for an `int`
- `net.minecraft.world.level.storage.loot.parameters`
    - `LootContextParams#CONTAINER` - A parameter for the active container referenced as a `SlotProvider`.
    - `LootContextParamSets`
        - `COMMAND_SLOT_SOURCE` - For getting the slots from some container provided in a command.
        - `COMMAND_COMPUTE_DEFAULT` - For running the default compute command.
        - `COMMAND_COMPUTE_POSITION` - For running the block compute command.
        - `COMMAND_COMPUTE_ENTITY` - For running the entity compute command.
        - `CONTAINER_PROCESS` - For getting a container that is processing something (e.g. furnace cooking, brewing stand brewing).
        - `bootstrap` - Returns all parameters in the context key set.
        - `validate` - Validates that all context kets are in `ALL_PARAMS`.
- `net.minecraft.world.level.storage.loot.predicates`
    - `AllOfCondition`
        - `INLINE_CODEC` is removed
        - `allOf` now takes in a holder set of `LootItemCondition`s instead of a list
    - `AnyOfCondition$Builder#or` now has an overload that takes in a holder-wrapped `LootItemCondition`
    - `CompositeLootItemCondition` now takes in a holder set of `LootItemCondition`s instead of a list
        - `holdersToLazyPredicates` - Converts a holder set of `LootItemCondition` to a list of lazy predicates.
        - `createCodec` now takes in a function with an input of a holder set of `LootItemCondition`s instead of a list
        - `createInlineCodec` is removed
        - `$Builder`
            - `addTerm` now has an overload that takes in a holder-wrapped `LootItemCondition`
            - `create` now takes in a holder set of `LootItemCondition`s instead of a list
    - `ConditionReference` record is removed
    - `ConditionUserBuilder`
        - `when` now has an overload that takes in a holder-wrapped `LootItemCondition` instead of the `LootItemCondition$Builder`
        - `buildCondition` - Builds a condition from a list of conditions.
    - `EntityHasScoreCondition` now takes in a score map with a value of `IntRangePredicate`s instead of `IntRange`s
        - `$Builder#withScore` now takes in an `IntRangePredicate` instead of an `IntRange`
    - `InvertedLootItemCondition` now takes in a holder-wrapped `LootItemCondition` instead of the raw value
        - `invert` now has an overload that takes in a holder-wrapped `LootItemCondition`
    - `LocationCheck` now takes in a `Vec3i` instead of a `BlockPos`
        - `checkLocation` now takes in a `Vec3i` instead of a `BlockPos`
            - It also has an overload that takes in a `Direction`
    - `LootItemBlockStatePropertyCondition` replaced by `MatchBlock`, not one-to-one
    - `LootItemCondition`
        - `TYPED_CODEC` -> `DIRECT_CODEC`
            - Original `DIRECT_CODEC` is removed
        - `LIST_CODEC` - The holder set codec.
    - `LootItemConditions` -> `LootItemConditionTypes`
    - `LootItemRandomChanceCondition` now takes in a holder-wrapped `ContextFloatProvider` instead of a `NumberProvider`
        - `randomChance` now takes in a holder-wrapped `ContextFloatProvider` instead of a `NumberProvider`
    - `LootItemRandomChanceWithEnchantedBonusCondition#randomChanceAndLootingBoost` now takes in an enchantment `HolderGetter` instead of the `HolderLookup$Provider`
    - `LootPredicates` - All vanilla reference registered `LootItemCondition`s. Most conditions are inlined into the loot table.
    - `TimeCheck` now takes in an `IntRangePredicate` instead of an `IntRange`
    - `ValueCheckCondition` split into `FloatValueCheck` for `float`s, and `IntValueCheck` for `int`s
- `net.minecraft.world.level.storage.loot.providers.number`
    - `AggregateProvider` - A provider that aggegates a set of other providers together.
    - `BinaryProvider` - A provider that performs an operation on two other providers.
    - `BinomialDistributionGenerator` -> `.ints.BinomialDistributionGenerator`, no `float` variant
    - `ConditionalProvider` - A provider that determines which provider to use based on the result of a `LootItemCondition`.
    - `ConstantValue` -> `.floats.ConstantValue`, `.ints.ConstantValue`
    - `DispatcherProvider` - A provider that determines which provider to use based on the first attached `LootItemCondition` that returns true, otherwise defaulting to a specified provider. Basically a find first implementation.
    - `DistributionProvider` - A provider that selects which provider to use from a weighted list.
    - `EnchantmentLevelProvider` -> `.floats.EnchantmentLevelProvider`, no `int` variant; now implements `LootContextUser`
    - `EnvironmentAttributeProvider` - A provider that reads the value of an `EnvironmentAttribute`.
    - `EnvironmentAttributeValue` -> `.floats.EnvironmentAttributeValue`, `.ints.EnvironmentAttributeValue`; now implements `EnvironmentAttributeProvider`
    - `NumberProvider` split into `ContextIntProvider`, `ContextFloatProvider`
        - Now extends `Validatable` instead of `LootContextUser`
        - `getFloat` -> `ContextFloatProvider#getFloat`
        - `getInt` -> `ContextIntProvider#getInt`
    - `NumberProviders` split into `ContextIntProviderTypes`, `ContextFloatProviderTypes`
        - `TYPED_CODEC` merged into `ContextIntProviders#DIRECT_CODEC`, `ContextFloatProviders#DIRECT_CODEC`
        - `CODEC` -> `ContextIntProviders#DIRECT_CODEC`, `ContextFloatProviders#DIRECT_CODEC`
            - `CODEC` is now the holder-wrapped variant
    - `PowerProvider` - A provider that raises some base provider to an exponent provider.
    - `RangeProvider` - A provider that can provide a value between the provider ranges.
    - `ScoreboardValue` -> `.ints.ScoreboardValue`, no `float` variant
    - `StorageValue` -> `.floats.StorageValue`, `.ints.StorageValue`; now taking in a `StoredNumberAccess` instead of the `Identifier` and `NbtPathArgument$NbtPath`, and a fallback provider
    - `StoredNumberAccess` - An accessor that reads the number from the storage with the specified id, navigated to by the given path.
    - `Sum` -> `.floats.Sum`, `.ints.Sum`; now implements `AggregateProvider`
    - `UnaryProvider` - A provider that operates upon itself.
    - `UniformGenerator` -> `.floats.UniformGenerator`, `.ints.UniformGenerator`; now implements `RangeProvider`
- `net.minecraft.world.level.storage.loot.providers.number.{floats, ints}` (Indicates there is both a float and int provider variant)
    - `Absolute` - Takes the absolute value of the sampled provider.
    - `Average` - Takes the average of all sampled providers.
    - `ConditionalValue` - Uses the true provider if the `LootItemCondition` returns `true`, otherwise uses the false provider.
    - `Difference` - Takes the difference of the sampled left and right providers.
    - `Maximum` - Takes the maximum sampled value across the providers.
    - `Minimum` - Takes the minimum sampled value across the providers.
    - `Modulus` - Takes the modulo of the sampled left provider via the sampled right provider.
    - `Negate` - Flips the sign of the sampled provider.
    - `NumberDispatcher` - Uses the provider whose `LootItemCondition` first returns `true`, otherwise uses the fallback provider. Basically a find first implementation.
    - `Power` - Takes the sampled base provider to the sampled exponent provider.
    - `Product` - Multiplies all sampled providers together.
    - `Quotient` - Divides the sampled left provider by the sampled right provider.
    - `WeightedListValue` - Picks a random weighted provider.
- `net.minecraft.world.level.storage.loot.providers.number.floats`
    - `Ceiling` - Ceils the sampled `float`.
    - `ContextFloatProvider` - Provides a `float` value given the `LootContext`.
    - `ContextFloatProviders` - All vanilla reference registered `float` providers. Most providers are inlined in the loot table.
    - `ContextFloatProviderTypes` - All registered `float` provider types.
    - `Cosine` - Takes the cosine of the sampled `float`.
    - `Floor` - Floors the sampled `float`.
    - `FromInt` - Converts a sampled `int` to a `float`.
    - `Length` - Calculates the euclidean distance of all sampled `float`s.
    - `ResolvableFloat` - A lazy supplied reference to a constant or `ResourceKey<ContextFloatProvider>`. This is for use outside the datapack context, like data components.
    - `Round` - Rounds the sampled `float` to the nearest `int`.
    - `Sine` - Takes the sine of the sampled `float`.
    - `SquareRoot` - Takes the square root of the sampled `float`.
    - `Truncate` - Truncates the `float` to the `int` closest to 0.
- `net.minecraft.world.level.storage.loot.providers.number.ints`
    - `ContextIntProvider` - Provides an `int` value given the `LootContext`.
    - `ContextIntProviders` - All vanilla reference registered `float` providers. Most providers are inlined in the loot table.
    - `ContextIntProviderTypes` - All registered `int` provider types.
    - `FloorModulus` - Takes the modulo of the sampled left `int` via the sampled right `int`, flooring towards the largest value less than the right `int` if the signs differ.
    - `FloorQuotient` - Divides the sampled left `int` by the sampled right `int`, flooring towards the largest value less than the right `int` if the signs differ.
    - `FromFloat` - Converts a sampled `float` to an `int`.
    - `ResolvableInt` - A lazy supplied reference to a constant or `ResourceKey<ContextIntProvider>`. This is for use outside the datapack context, like data components.

## Minor Migrations

The following is a list of useful or interesting additions, changes, and removals that do not deserve their own section in the primer.

### Command Responses

`CommandResponseTracker` is a method of handling the response to send during a command, tracking how many times a given element was successfully handled. The tracker can be broken into three parts: creating the tracker, tracking what values are returned by the command, and sending the feedback message based on the result.

The tracker is created by calling `CommandResponseTracker#create`, specifying the generic of the object being tracked:

```java
// In some command method
private static int exampleCommandMethod(CommandSourceStack source, Collection<? extends Entity> entities) throws CommandSyntaxException {
    // We are tracking entities in our command
    CommandResponseTracker<Entity> tracker = CommandResponseTracker.create();

    // ...
}
```

Then, `CommandResponseTracker#track` is used to keep track of the specific count we care about. The amount counted is either 1 if only the element is passed in, a 1 or 0 if a `boolean` is provided, or the raw count `int`:

```java
// In some command method.
private static int exampleCommandMethod(CommandSourceStack source, Collection<? extends Entity> entities) throws CommandSyntaxException {
    // ...

    for (Entity entity : entities) {
        // Tracking whether the entity has a glowing tag.
        // 1 if yes, 0 is no.
        tracker.track(entity, entity.hasGlowingTag);
    }
}
```

Finally, to determine what message to display to get, we call `CommandResponseTracker#sendFeedback`, providing the `CommandSourceStack`, a `boolean` for whether the message should broadcast to the admins, and the messages to display depending on the scenario.

The messages are constructed using a `CommandResponseTracker$Messages`, `$MessagesWithArg`, or `MessagesWithArgs`, each taking zero, one, or two additional arguments, respectively, to pass to the message. They are constructed via `CommandResponseTracker#messages`, taking in an optional error handler to throw, a message if one object was successfully handled, and a general message if multiple objects were successfully handled:

```java
// In the same location as the command method.
// The first argument should match the generic of the tracker.
// The other arguments are arbitrary depending on use case.
private static final CommandResponseTracker.MessagesWithArg<Entity, Integer> RESPONSE_EXAMPLE = CommandResponseTracker.messages(
    // The error to throw on failure.
    // This is optional, if none is specified, then no error will ever be thrown.
    // Can either be a `SimpleCommandExceptionType`, or some function that takes in the argument and returns a `CommandSyntaxException`.
    new SimpleCommandExceptionType(Component.translatable("commands.examplemod.example.failed")),
    // The message to display on single success.
    // This takes in the object generic, the total value tracked (what's returned to the command), and any additional arguments used.
    // It returns the message to display.
    (entity, totalValue, arg) -> Component.translatable("commands.examplemod.example.success.single", entity.getDisplayName(), totalValue, arg),
    // The message to display on multiple success.
    // This takes in the total number of objects tracked, the total value tracked (what's returned to the command), and any additional arguments used.
    // It returns the message to display.
    (entityCount, totalValue, arg) -> Component.translatable("commands.examplemod.example.success.multiple", entityCount, totalValue, arg)
);

// In some command method.
private static int exampleCommandMethod(CommandSourceStack source, Collection<? extends Entity> entities) throws CommandSyntaxException {
    // ...

    // Returned the tracked total value
    return tracker.sendFeedback(
        // The command source stack.
        source,
        // Whether to broadcast the message to the admins.
        true,
        // What entity to provide during a single success.
        // This is either:
        // - `ANY`, which chooses the first tracked entity.
        // - `NON_ZERO`, which chooses the first tracked entity that provides a non-zero value.
        // If not specified, it defaults to `NON_ZERO`
        CommandResponseTracker.ElementType.NON_ZERO,
        // The response messages to choose from.
        RESPONSE_EXAMPLE,
        // Any additional arguments passed to the message handler.
        42
    );
}
```

If `sendFeedback` is too limiting, you can determine what to do by calling `dispatch` instead:

```java
// In some command method.
private static int exampleCommandMethod(CommandSourceStack source, Collection<? extends Entity> entities) throws CommandSyntaxException {
    // ...

    boolean success = tracker.dispatch(
        // What entity to provide during a single success.
        // This is either:
        // - `ANY`, which chooses the first tracked entity.
        // - `NON_ZERO`, which chooses the first tracked entity that provides a non-zero value.
        CommandResponseTracker.ElementType.NON_ZERO,
        // The handler for what to do
        // Either a `CommandResponseTracker$Dispatch`, `$DispatchWithArg`, or `$DispatchWithArgs`
        new CommandResponseTracker.DispatchWithArg<>(
            // If there is one match.
            // This takes in the object generic, the total value tracked (what's returned to the command), and any additional arguments used.
            (entity, totalValue, arg) -> true,
            // If there are no or multiple matches.
            // This takes in the total number of objects tracked, the total value tracked (what's returned to the command), and any additional arguments used.
            (entityCount, totalValue, arg) -> false
        ),
        // Any additional arguments passed to the message handler.
        42
    );
}
```

- `net.minecraft.server.commands`
    - `ArgProvider` - Provides an argument mapping to a specific object.
    - `CommandResponseTracker` - Tracks all targets of a command and sends any feedback to specific users.

### Conversion Tracker

`ConversionTracker` is a method of handling the conversions of mob from one form to another (e.g. zombie -> drowned, skeleton -> stray), as long as they are entities of the same type. It takes in the entity information along with any necessary storage and values, and manages the behavior until the entity is converted sucessfully, or is reset.

The tracker is typically added as an instance field on the entity itself, then hooked in via `tick`, `addAdditionalSaveData`, and `readAdditionalSaveData`:

```java
// Within some Mob subtype
// Assume we have ExampleEntity extends Mob
// And another ExampleSubEntity extends ExampleEntity
public class ExampleEntity extends Mob {
    // For syncing that the entity is being converted.
    private static final EntityDataAccessor<Boolean> SUB_CONVERSION_ID = SynchedEntityData.defineId(ExampleEntity.class, EntityDataSerializers.BOOLEAN);

    private final ConversionTracker<ExampleEntity> subTracker = new ConversionTracker<>(
        // The mob being converted.
        // This is used to handle everything related to setting and checking any mob data,
        // and eventually calling `Mob#convertTo`.
        this,
        // The accessor used to sync that the entity is currently being converted.
        SUB_CONVERSION_ID,
        // The entity this will become after conversion.
        () -> EXAMPLE_SUB_ENTITY,
        // The level event to run on the client on conversion.
        () -> LevelEvent.SOUND_GHAST_FIREBALL,
        // The predicate that determines whether the mob can be converted.
        // This must remain true throughout the entire affliction and conversion process;
        // otherwise the conversion will fail.
        this::isAlive,
        // The tag key used to store the affliction time on the entity when writing to disk.
        "SubAfflictedTime",
        // How many ticks the entity needs to be afflicted for before the conversion can start.
        600, // 30 seconds
        // The tag key used to store the conversion time on the entity when writing to disk.
        "SubConversionTime",
        // How many ticks the entity takes to convert.
        20, // 1 second
        // A consumer that is run after the entity has been converted, as a finalization step.
        (ExampleSubEntity converted, ServerLevel level) -> {
            
        }
    );

    // ...

    @Override
    protected void defineSynchedData(SynchedEntityData.Builder entityData) {
        super.defineSynchedData(entityData);
        // Register the synced data.
        entityData.define(SUB_CONVERSION_ID, false);
    }

    @Override
    public void tick() {
        super.tick();
        // Tick the tracker.
        this.subTracker.tick();
    }

    @Override
    protected void addAdditionalSaveData(ValueOutput output) {
        super.addAdditionalSaveData(output);
        // Write our tracker data.
        this.subTracker.addAdditionalSaveData(output);
    }

    @Override
    protected void readAdditionalSaveData(ValueInput input) {
        super.readAdditionalSaveData(input);
        // Read the tracker data.
        this.subTracker.readAdditionalSaveData(input);
    }
}
```

Note that, given the requirement that the converted entity must be a subtype of the converting entity, this has rather limited use cases (it's the reason why zombie villager -> villager does not use the tracker).

- `net.minecraft.world.entity.ConversionTracker` - A tracker for handling a mob turning into another mob.
- `net.minecraft.world.entity.monster.skeleton.Skeleton`
    - `isFreezeConverting` -> `ConversionTracker#isConverting`
    - `setFreezeConverting` -> `ConversionTracker#setConverting`
    - `doFreezeConversion` -> `ConversionTracker#doConversion`    
- `net.minecraft.world.entity.monster.zombie`
    - `Drowned#rangedAttackUncertainty`, `setRangedAttackUncertainty` - Handles the uncertainty when making a ranged attack (typically with a trident).
    - `Zombie`
        - `getConversionSound` - The level event to play on conversion.
        - `convertsToWhenDrowning` - The entity this converts to when drowning.
        - `doUnderWaterConversion` -> `ConversionTracker#doConversion`
        - `convertToZombieType` -> `ConversionTracker#doConversion`


### Tag Changes

- `minecraft:block`
    - `blocks_dolphin_jump`
    - `blocks_fluid_flow`
    - `blocks_lava_fire_spread`
    - `blocks_motion`
    - `blocks_motion_in_heightmap`
    - `blocks_motion_in_heightmap_no_leaves`
    - `blocks_motion_no_leaves`
    - `cannot_place_basalt_pillar_on`
    - `cat_does_not_teleport_to`
    - `cats_can_lie_on`
    - `cats_can_sit_on`
    - `causes_suffocation`
    - `concrete_slabs`
    - `concrete_stairs`
    - `conduit_effect_block`
    - `convertable_to_mud` -> `convertible_to_mud`
    - `cushion_uses_collision_shape`
    - `dangerous_for_teleportation`
    - `enderman_does_not_teleport_to`
    - `entities_can_teleport_to`
    - `height_specific_ore_replaceables`
    - `ice_melts_when_destroyed_above`
    - `nether_carver_replaceables` is removed
    - `overworld_carver_replaceables` is removed
    - `poplar_logs`
    - `required_for_poplar_leaf_ambience`
    - `sculk_growth_inhibitors`
    - `shulker_does_not_teleport_to`
    - `skulls`
    - `speeds_up_zombie_villager_curing`
    - `turns_into_dirt_path`
    - `turns_into_farmland`
    - `uncarvable`
    - `villager_babies_can_jump_on_bed`
    - `villagers_can_sleep_on_bed`
    - `washed_away_by_fluids`
    - `wool_slabs`
    - `wool_stairs`
- `minecraft:damage_type`
    - `bypasses_cooldown`
    - `no_wolf_retaliation`
- `minecraft:entity_type`
    - `cannot_be_dismounted_by_item_usage`
- `minecraft:fluid`
    - `axolotl_tries_to_find`
    - `dolphin_tries_to_find`
    - `entity_floatable`
    - `frog_tries_to_find_land_near`
- `minecraft:item`
    - `brewing_fuel` is removed
    - `brewing_potion_inputs`
    - `clonable_maps`
    - `concrete_slabs`
    - `concrete_stairs`
    - `cushions`
    - `douses_campfires`
    - `extendable_maps`
    - `furnace_fuel_bottom_takeable`
    - `mushrooms`
    - `ores`
    - `poplar_logs`
    - `wool_slabs`
    - `wool_stairs`
- `minecraft:potion`
    - `douses_fire`
    - `extinguishes_entities`
    - `hurts_water_sensitive_entities`
    - `rehydrates_axolotls`
- `minecraft:worldgen/biome/has_structure`
    - `abandoned_camp_bamboo_jungle`
    - `abandoned_camp_birch_forest`
    - `abandoned_camp_cherry_grove`
    - `abandoned_camp_dappled_forest`
    - `abandoned_camp_flower_forest`
    - `abandoned_camp_forest`
    - `abandoned_camp_meadow`
    - `abandoned_camp_old_growth_birch_forest`
    - `abandoned_camp_old_growth_pine_taiga`
    - `abandoned_camp_old_growth_spruce_taiga`
    - `abandoned_camp_pale_garden`
    - `abandoned_camp_savanna`
    - `abandoned_camp_snowy_taiga`
    - `abandoned_camp_sparse_jungle`
    - `abandoned_camp_swamp`
    - `abandoned_camp_taiga`
    - `abandoned_camp_windswept_forest`
    - `abandoned_camp_wooded_badlands`
- `minecraft:worldgen/configured_feature` -> `worldgen/feature`
- `minecraft:worldgen/structure`
    - `abandoned_camp`
    - `on_abandoned_camp_bamboo_jungle`
    - `on_abandoned_camp_birch_forest`
    - `on_abandoned_camp_cherry_grove`
    - `on_abandoned_camp_dappled_forest`
    - `on_abandoned_camp_flower_forest`
    - `on_abandoned_camp_pale_garden`
    - `on_abandoned_camp_swamp`
    - `on_abandoned_camp_windswept_forest`
    - `on_ancient_city_maps`
    - `on_desert_pyramid_maps`
    - `on_jungle_explorer_maps` -> `on_jungle_pyramid_maps`
    - `on_mineshaft_maps`
    - `on_ocean_explorer_maps` -> `on_ocean_monument_maps`
    - `on_ocean_ruin_warm_maps`
    - `on_swamp_explorer_maps` -> `on_swamp_hut_maps`
    - `on_trial_chambers_maps` -> `on_burial_trial_chambers_maps`
    - `on_woodland_explorer_maps` -> `on_woodland_mansion_maps`

### List of Additions

- `com.mojang.math.Axis`
    - `rotate` - Rotates a matrix around the given radian angle.
    - `rotateDegrees` - Rotates a matrix around the given degree angle.
- `net.minecraft.SharedConstants`
    - `DEBUG_CALCULATE_SOLID` - A flag that dumps the solidness of blogs to the debugger.
    - `DEBUG_ENABLE_FARLANDS` - A flag that enables the farlands bug.
    - `IS_RENDERDOC_ATTACHED` - Whether renderdoc is attached to the running game instance.
- `net.minecraft.client.resources.language.EmptyTranslationsException` - Throws an exception if there is a missing translation for a language.
- `net.minecraft.client.resources.sounds`
    - `AbstractSoundInstance#soundEvent` - The sounds to choose from.
    - `SoundInstance#getSoundEvent` - The sounds to choose from.
- `net.minecraft.client.server.IntegratedServer`
    - `setPersonalGameType`, `getPersonalGameMode` - Handles the host user's game mode.
    - `getCustomPermissionLevel` - Gets the current permission level for guests, or nothing if pure singleplayer.
- `net.minecraft.client.sounds.SoundEngine#LOOPING_SOUND_SUBTITLE_INTERVAL_TICKS` - The number of ticks of how often to notify new listeners to display the subtitle of a looping sound.
- `net.minecraft.commands`
    - `CommandSourceStack$NamesProvider` - A provider for the display and text name of the source.
    - `SharedSuggestionProvider#getAvailablePostEffects` - Returns all available post effects.
- `net.minecraft.commands.argument`
    - `ResourceOrIdArgument`
        - `floatProvider`, `getFloatProvider`, `$ContextFloatProviderArgument` - Float provider arguments.
        - `intProvider`, `getIntProvider`, `$ContextIntProviderArgument` - Int provider arguments.
        - `feature`, `getFeature`, `$ContextIntProviderArgument` - Feature arguments.
    - `SlotSourceArgument` - A command argument that specifies a slot source.
    - `SwingAnimationArgument` - A command argument that specifies a swing animation type.
- `net.minecraft.commands.arguments.coordinates.LocalCoordinates#apply` - Applies the local coordinates to the given vector with the specified rotation.
- `net.minecraft.commands.synchronization.SuggestionProviders#POST_EFFECTS` - Suggestions for post effects to select.
- `net.minecraft.core`
    - `BlockPos`
        - `withinClippedManhattan` - Iterates through all positions by manhattan distance within the reach bounds.
        - `withinBoxByManhattanDistance` - Iterates through all positions by manhattan distance within the box bounds.
    - `CompositeDirection` - A composition of multiple directions, with a computed step.
    - `Directional` - An interface that defines whether something has a direction.
    - `PositionAndRotation` - An interfaces that defines an object has a position and rotation.
    - `Vec3i#differsHorizontally` - Whether the passed vector has a different horizontal position.
- `net.minecraft.data`
    - `BlockFamilies`
        - `POPLAR_PLANKS` - Poplar variants.
        - `WOOL` - Wool variants.
        - `CONCRETE` - Concrete variants.
    - `BlockFamily$Builder#carpet`, `$Variant#CARPET` - Carpet variant for a base block.
- `net.minecraft.data.recipes.RecipesProvider`
    - `carpetBuilder` - Creates a carpet recipe.
    - `cushionRecipe` - Creates a cushion recipe.
- `net.minecraft.gametest.framework`
    - `GameTestDimensions` - The dimensions to test the game within.
    - `GameTestHelper`
        - `rotateEntityWithTest` - Rotates the entity relative to the test.
        - `useItemOnBlock` - Uses an item on the given block, calling `ItemStack#use`.
    - `GameTestInstance#dimension` - The dimension to run the test in.
    - `TestFunctionLoader#ALL_LOADERS` - All registered loaders for test functions.
- `net.minecraft.network.chat.FilterMask$Type#STREAM_CODEC`, `streamCodec` - Network codecs for the types of chat filters.
- `net.minecraft.network.codec.ByteBufCodecs`
    - `BIT_SET` - A network codec for a `BitSet`.
    - `INSTANT` - A network codec for an `Instant`.
    - `fixedBitSet` - A network codec for a `BitSet` with a certain size.
    - `fixedSizeCollection`, `fixedSizeList` - A network codec for a collection with a certain size.
- `net.minecraft.network.protocol.common`
    - `ClientboundPostEffectsPacket` - A packet sent to the client of what effects to activate.
    - `ClientCommonPacketListener#handlePostEffects` - Handles when the `ClientboundPostEffectsPacket` is received from the server.
- `net.minecraft.network.protocol.game`
    - `ClientboundCustomChatCompletionsPacket$Action#STREAM_CODEC` - The network codec for what action to take for the received chat completions.
    - `ClientboundMoveEntityPacket`
        - `unpackOnGround` - Unpacks whether the entity is on the ground.
        - `unpackStepCount` - Unpacks the number of steps the entity has taken.
        - `packProperties` - Unpacks the entity properties.
    - `ClientboundPlayerInfoUpdatePacket$Action#STREAM_CODEC` - The network codec for what action to take when display player information.
    - `ClientboundSwingAnimationPacket` - A packet sent to the client when the player swings one of their hands.
    - `ClientGamePacketListener#handleSwingAnimation` - Handles when the `ClientboundSwingAnimationPacket` is received from the server.
    - `MovementPacket` - A packet that changes the position and/or rotation of an object.
    - `ServerboundPlayerActionPacket$Action#CHANGE_DESTROY_DIRECTION` - Changes the face that the player is destroying the block from.
    - `VecDelta` - Defines the delta of a vector either with some kind of stepped movement (e.g., linear, keyframed).
    - `VecDeltaCodec`
        - `encodingPrecisionLoss` - Calculates the precision loss during encoding.
        - `isDeltaTooBig` - Checks whether the delta movement is larger than a short.
        - `tryEncode` - Attempts to encode the position path movement into a delta vector.
- `net.minecraft.network.syncher.EntityDataSerializers#DYE_COLOR` - Syncs the `DyeColor` of an entity on change.
- `net.minecraft.resources`
    - `FileToIdConverter`
        - `prefixMatches` - Whether the path of an identifier starts with the file prefix.
        - `matches` - Whether the identifier matches the file prefix and extension.
- `net.minecraft.server.MinecraftServer#forceGameMode`, `setForceGameMode` - Handles forcing the game mode of all players.
- `net.minecraft.server.commands`
    - `ComputeCommand` - A command that allows computing `int` and `float` providers.
    - `LootContextSources` - A helper for setting the context sources to a `LootParams$Builder`.
    - `PostEffectCommand` - A command for modifying the post effects on the camera.
- `net.minecraft.server.commands.item`
    - `BlockItemAccessor` - An accessor that modifies the items in a block.
    - `EntityItemAccessor` - An accessor that modifies the items held by an entity.
    - `ItemAccessor` - An accessor for modifying items stored on an object.
- `net.minecraft.server.level`
    - `ParticleStatus#STREAM_CODEC` - The network codec for particle settings.
    - `ServerChunkCache#sendToTrackingPlayersFiltered` - Sends packets to all players that match the predicate.
    - `ServerLevel#uncachedBiomeResolver` - Returns a biome resolver that uses a non-cached sampler.
- `net.minecraft.server.network.ServerCommandSuggestionsProvider` - A suggestions provider on the server for received command completions.
- `net.minecraft.server.notifications.NotificationService#worldUpgradeStarted`, `worldUpgradeProgress`, `worldUpgradeFinished`, `worldUpgradeFailed` - Notifications sent when upgrading a world.
- `net.minecraft.util`
    - `ARGB`
        - `colorFromVector3f` - Converts an RGB vector to an `int`.
        - `colorFromVector4f` - Converts an ARGB vector to an `int`.
    - `BitStorage#fill` - Sets all bit of the storage to the specified value.
    - `BlockUtil#MAX_POSITION_DIFFERENCE_PACKING_RADIUS`, `clampedPackDifferenceInPosition`, `packDifferenceInPosition`, `unpackDifferenceInPosition` - Handles packing the difference between two positions into a single integer, such as for running a level event when an enderman teleports.
    - `CommonLinks$ExtensionReference` - An enum representing the reference to extend a realm.
    - `ExtraCodecs#RGB_COLOR_VEC3_CODEC`, `ARGB_COLOR_VEC4_CODEC`, `STRING_RGB_VEC3_COLOR`, `STRING_ARGB_VEC4_COLOR` - Color codecs.
    - `Mth`
        - `lerp2` - Performs two levels of linear interpolation.
        - `lerp3` - Performs three levels of linear interpolation.
    - `Prediction` - An enum that represents the calculation should be predicted by the client or handled server only.
    - `ProblemReporter`
        - `$CollectionReferencePathElement` - A path element of a `TagKey`.
        - `$Collector#hasFatalProblems` - If the report contains a problem that is considered fatal when attempting to load and use the data.
        - `$Problem#isFatal` - Whether the problem prevents loading or usage of the data.
    - `StaticCache2D#map`, `$MappingFunction` - Maps each XZ coordinate value to a new value.
    - `Util`
        - `MILLIS_PER_SECOND` - The number of milliseconds in a second.
        - `toMillis` - Converts seconds to milliseconds, flooring to the closest millisecond.
        - `isAppleSiliconMac` - If the renderer string is using Apple silicon.
- `net.minecraft.util.random.WeightedList#addAll` - Add all elements from another weighted list to this one.
- `net.minecraft.util.worldupdate.UpgradeProgress#notifications` - The notification service.
- `net.minecraft.world`
    - `InteractionHand#asArm` - Gets the correct `HumanoidArm` based on the user's main arm.
    - `InteractionResult$Success#shouldSwing` - Whether the arm performing the interaction should swing.
- `net.minecraft.world.entity`
    - `Entity`
        - `TAG_INVULNERABLE_TIME` - A tag for how long an entity is invulnerable for.
        - `interpolationHandler` - A handler for interpolating between positions on a path.
        - `commonTick` - A tick method that's always executed and cannot be modified before the entity's actual tick logic.
        - `recordMovement` - Records the movement received, typically for client simulation.
        - `isInFloatableFluid` - Whether the entity is a fluid that it can float in.
        - `pushFromExplosion` - Pushes an entity due to an explosion.
        - `calculateViewQuaternion` - Calculates the quanternion for where the entity is looking.
        - `postDataManipulated` - Runs after the entity's data is manipulated from the `EntityDataAccessor`.
        - `doTeamsAllowDamage` - Whether teams can damage each other.
        - `onInterpolationStart`, `createInterpolationHandler` - Creates the interpolation handler used for movement.
        - `getLookQuaternion` - Gets the quanternion for where the entity is looking.
        - `projectileReceivesSideEffectsOnHit` - Whether the projectile performs additional actions on hitting an entity.
        - `isPermanentlyInvulnerable` - If the entity can never be hurt.
        - `isTemporarilyInvulnerable` - If the entity is currently invulnerable for a set period of time.
        - `getInterpolatedBoundingBox` - Gets the entity's bounding box interpolated by some partial tick.
        - `getMoveSimulationType` - Gets how the entity's movements are simulated.
        - `storePositionAndRotation`, `getClientPositionAndRotation`, `getClientPosition` - Handles getting the entity's position and/or rotation, typically on the client.
        - `setInvulnerableTime`, `getInvulnerableTime` - Handles temporary invulnerability.
    - `EntityEvent$Value` - An annotation that represents a value as being an entity event.
    - `EntityType`
        - `NO_UPDATE_INTERVAL`, `$Builder#noUpdateInterval` - An entity that never updates on the client.
        - `hasUpdateInterval` - Whether the entity updates on the client.
        - `$Builder#dontTrackDeltas` - Whether not to track the delta movements of an entity.
    - `InterpolationHandler#interpolationTracker` - A tracker for the current interpolated value.
    - `InterpolationTracker` - A tracker for the current interpolated value.
    - `LivingEntity`
        - `damageCooldownTime` - How long to wait between receiving damage.
        - `wasHurtRecently` - Whether the entity's hurt timer is positive.
        - `isInFluidDeeperThan` - If the fluid height is greater than some given value.
        - `isSwinging` - If the entity is swinging its arm.
        - `canRandomlyTeleportTo` - Whether this entity can randomly teleport to the given location.
        - `$SwingDescription` - A record that contains information about the hand swinging.
    - `MoverType#isServerAndClientSimulated` - If the movement is simulated on both the client and server.
    - `MoveSimulationType` - An enum containing the types of movement simulation (e.g., both client and server, only on the side running the logical server, etc.).
    - `PositionPath` - A path of movements between two positions.
    - `PositionStep` - A step representing some location on a path made at a certain tick offset.
    - `SteppedInterpolationHandler` - An interpolation handler that moves between path segments.
    - `SteppedInterpolationTracker` - An interpolation tracker that handles moving between path segments.
    - `UpdateInterval` - Handles when an entity should be next updated.
- `net.minecraft.world.entity.decoration`
    - `BlockAttachedEntity`
        - `TAG_BLOCK_POS` - A tag containing the block position name.
        - `tickAtCheckInterval` - Runs once every X ticks (by default 100).
        - `onKilled` - What to do when the entity is killed.
    - `Cushion` - A cushion entity.
    - `RangedAttackMob#rangedAttackUncertainty` - Returns the scalar affecting a mob's accuracy.
- `net.minecraft.world.entity.monster.zombie.Drowned#rangedAttackUncertainty`, `setRangedAttackUncertainty` - Handles the uncertainty when making a ranged attack (typically with a trident).
- `net.minecraft.world.entity.player.ChatVisibility#STREAM_CODEC` - The network codec.
- `net.minecraft.world.entity.projectile.Projectile`
    - `canBreakBlockInAdventureMode` - Whether the projectile can break a block while the world game mode is adventure mode.
    - `onRedirectProjectile` - What to do when a projectile is redirected via deflection.
- `net.minecraft.world.inventory`
    - `AbstractContainerMenu`
        - `CONTAINER_CLICK_PRIMARY` - A constant representing the player clicked in the inventory using the attack button.
        - `CONTAINER_CLICK_SECONDARY` - A constant representing the player clicked in the inventory using the use item button.
    - `MerchantMenu#updateSellItem` - Updates the item being offered for a trade.
    - `SlotRanges#tryRead`, `read` - Parses the slot range from its string identifier.
- `net.minecraft.world.item`
    - `ArrayItemProvider` - A provider iterating through an array of items, providing a copy of each to use.
    - `CushionItem` - An item for a cushion entity.
    - `ItemProvider` - An accessor that iterates through some backing item holder.
    - `MapItem#applyNewSavedData` - Creates new saved data for the map and stores it as a component on the item.
    - `SwingAnimationType#byName` - Gets the animation type by its name, otherwise defaulting to the provided value.
- `net.minecraft.world.level`
    - `BaseSpawner#setEntityData` - Sets the data of the spawned entity.
    - `EmptyStructureManager` - A `StructureManager` that generates nothing.
    - `GameType#OPTIONAL_STREAM_CODEC` - A codec that returns a game type from an integer when present.
    - `Level#getRelativeTickSpeed` - Gets the tick speed relative to the game's normal 20 ticks per second.
    - `Spawner#setEntityData` - Sets the data of the spawned entity.
- `net.minecraft.world.level.block.LevelEvent$Value` - An annotation that marks an `int` as representing a level event id.
- `net.minecraft.world.level.block.entity.trialspawner.TrialSpawner$FullConfig#overrideEntityData` - Sets the entity data of the trial spawner.
- `net.minecraft.world.level.block.sounds.AmbientLeavesBlockSoundPlayer` - A record containing the data to play ambient leaves sounds on the client.
- `net.minecraft.world.level.block.state.SolidDebugger` - A debugger for checking and logging the solidness of blocks.
- `net.minecraft.world.level.blockscan`
    - `BlockMatcher` - A helper that returns a list of block positions, typically provided to the matcher, by some `BlockState` or `BlockPos` predicate.
    - `BlockScanUtils` - A utility for returning if a block was found that causes the scan to abort.
    - `BlockStateConsumer` - A functional interface that, given a `BlockState` and `BlockPos`, whether the search should continue or abort.
    - `BoxBlockMatcher` - A matcher that checks the positions within a given box.
    - `FilteredSectionCache` - A cache for quickly retrieving chunk sections that has the matching `BlockState` predicate.
    - `OrderedBlockMatcher` - A matcher that checks the positions in the order provided by the passed in `Iterable`.
- `net.minecraft.world.level.dimension.DimensionDefaults#*_CELL_SIZE_*` - Defines the size of a cell in blocks in each direction during generation.
- `net.minecraft.world.level.dimension.end.EnderDragonFight#getPodiumLocation` - Returns the end podium location offset by the given `BlockPos`.
- `net.minecraft.world.level.material.FluidState#getHeightForCamera` - Returns the height of the fluid for camera positioning by always returning `1` if the source fluid has a sturdy block above it.
- `net.minecraft.world.level.pathfinder.PathType#STREAM_CODEC` - The network codec.
- `net.minecraft.world.level.storage.PrimaryLevelData#writeVersionHistory` - Writes the version history of the world.
- `net.minecraft.world.phys`
    - `AABB#nextDeflated` - Returns the deflated bounds of the AABB, moving to the next adjacent value.
    - `BlockHitResult#STREAM_CODEC` - The network codec.
    - `Vec3#atCenterOfWithY` - Centers the `Vec3i` with the given Y position.

### List of Changes

- `com.mojang.math.Axis` is no longer a functional interface
    - `of` now takes in a `Vector3fc` instead of a `Vector3f`
- `net.minecraft.client.resources.langauage.LanguageManager` now takes in the `Minecraft` instance
- `net.minecraft.client.resources.sounds`
    - `Sound#getAttenuationDistance` now has na overload that takes in the `float` volume
    - `SoundInstance#resolve` -> `getOrResolve`
    - `UnderwaterAmbientSoundInstances` is removed
        - `$SubSound` -> `$UnderLiquidSubSound`
        - `$UnderwaterAmbientSoundInstance` -> `UnderwaterLiquidAmbientSoundInstance`
- `net.minecraft.client.server.IntegratedServer`
    - `commandsAllowedForOtherPlayers` -> `getGuestCommandAccess`
    - `setCommandsAllowedForOtherPlayers` -> `setGuestCommandAccess`
- `net.minecraft.commands`
    - `CommandSourceStack` no longer takes in the `String` text name, and can optionally decide whether to provide the `Component` display name
    - `SharedSuggestionProvider`
        - `suggestRegistryElements` now specifies a generic `E` for the registry object type, now taking in a `Predicate` filter
        - `listSuggestions` now specifies a generic `E` for the registry object type,now optionally taking in a `Predicate` filter
- `net.minecraft.commands.arguments`
    - `ArgumentSignatures`
        - `ArgumentSignatures(FriendlyByteBuf)`, `#write` replaced by `STREAM_CODEC`
        - `$Entry(FriendlyByteBuf)`, `$Entry#write` replaced by `$Entry#STREAM_CODEC`
    - `ResourceKeyArgument`, `#key` can now take in a `Predicate` filter
- `net.minecraft.core`
    - `Direction` now implements `Directional`
    - `Direction8` -> `CompositeDirection$Direction8`
- `net.minecraft.gametest.framework`
    - `BuiltinTestFunctions` now implements `TestFunctionLoader` instead of extending
    - `GameTestBatch` now takes in the `ResourceKey` for the dimension to test in
    - `GameTestBatchFactory`
        - `divideIntoBatches` now takes in the `MinecraftServer` instead of the `ServerLevel`
        - `toGameTestBatch` now takes in the `ResourceKey` for the dimension to test in
    - `GameTestRunner` now takes in the `MinecraftServer` instead of the `ServerLevel`
        - `$Builder#fromBatches`, `fromInfo` now take in the `MinecraftServer` instead of the `ServerLevel`
        - `$StructureSpawner#onBatchStart` now takes in the `MinecraftServer` instead of the `ServerLevel`
    - `StructureGridSpawner` now takes in a `Function<ResourceKey<Level>, BlockPos` for the first test instead of the raw `BlockPos`
    - `TestData` now takes in the `ResourceKey` for the dimension to test in
    - `TestFunctionLoader` is now an `interface` from a `class`
    - `TestPosFinder#findTestPos` now returns a stream of `GlobalPos` instead of `BlockPos`
- `net.minecraft.network.FriendlyByteBuf#readFixedBitSet`, `writeFixedBitSet` now have static overloads that take in the `ByteBuf`
- `net.minecraft.network.chat`
    - `FilterMask#read`, `write` replaced by `STREAM_CODEC`
    - `LastSeenMessages`
        - `$Packed(FriendlyByteBuf)`, `#write` replaced by `STREAM_CODEC`
        - `$Update(FriendlyByteBuf)`, `#write` replaced by `STREAM_CODEC`
    - `MessageSignature#read`, `write` replaced by `STREAM_CODEC`
        - Original methods are now `private` from `public`
        - `$Packed$#read`, `write` replaced by `STREAM_CODEC`
            - Original methods are now `private` from `public`
    - `RemoteChatSession$Data#read`, `write` replaced by `STREAM_CODEC`
    - `ResolutionContext` now takes in an `int` resolution limit and a `MutableInt` for the resolved count
    - `SignedMessageBody$Packed(FriendlyByteBuf)`, `#write` replaced by `STREAM_CODEC`
- `net.minecraft.network.codec.StreamCodec#composite` now has overloads that take up to fourteen elements.
- `net.minecraft.network.protocol.common.ClientboundUpdateTagsPacket` is now a record
- `net.minecraft.network.protocol.configuration.ClientboundUpdateEnabledFeaturesPacket#STREAM_CODEC` can now work with a `ByteBuf`
- `net.minecraft.network.protocol.game`
    - `ClientboundAnimatePacket#SWING_MAIN_HAND`, `SWING_OFF_HAND` now handled by `ClientboundSwingAnimationPacket`
    - `ClientboundChunksBiomesPacket#STREAM_CODEC` can now work with a `ByteBuf`
        - `$ChunkBiomeData(FriendlyByteBuf)`, `#write` replaced by `STREAM_CODEC`
    - `ClientboundCustomChatCompletionsPacket#STREAM_CODEC` can now work with a `ByteBuf`
    - `ClientboundDeleteChatPacket#STREAM_CODEC` can now work with a `ByteBuf`
    - `ClientboundEntityPositionSyncPacket` now implements `MovementPacket` instead of `Packet`
        - The constructor now takes in a `PositionPath` and `float`s for the xy rotation instaed of a `PositionMoveRotation`
        - `of` now has an overload that specifies the `PositionPath`
    - `ClientboundExplodePacket` now takes in a `boolean` for whether to play a sound
    - `ClientboundLevelChunkPacketData(FriendlyByteBuf, int, int)`, `#write` replaced by `STREAM_CODEC`
        - `getBlockEntitiesTagsConsumer` replaced by `forEachBlockEntityTag`, not one-to-one
    - `ClientboundLevelChunkWithLightPacket` is now a record
    - `ClientboundLevelParticlesPacket` is now a record
        - The constructor now takes in a `$RandomizationType`
        - `$RandomizationType` - How the particle should move when spawned.
    - `ClientboundLightUpdatePacket` is now a record
    - `ClientboundLightUpdatePacketData` is now a record
    - `ClientboundMoveEntityPacket` now implements `MovementPacket` instead of `Packet`
        - The constructor now takes in a `VecDelta` instead of three `short`s
        - `xa`, `ya`, `za` merged into `VecDelta`
        - `getXa`, `getYa`, `getZa` merged into `getPositionDelta`
        - `$Pos` now takes in a `VecDelta` instead of three `short`s
        - `$PosRot` now takes in a `VecDelta` instead of three `short`s
    - `ClientboundMoveVehiclePacket` now only takes in a `PositionAndRotation`
    - `ClientboundOpenBookPacket` is now a record
    - `ClientboundOpenSignEditorPacket` is now a record
    - `ClientboundPlayerChatPacket` now uses `Optional`s for the signature and unsigned content instead of nullables
    - `ClientboundPlayerInfoRemovePacket#STREAM_CODEC` can now work with a `ByteBuf`
    - `ClientboundRemoveEntitiesPacket` is now a record
    - `CommonPlayerSpawnInfo` now uses an `Optional` for previous game type instead of a nullable
        - `CommonPlayerSpawnInfo(FriendlyByteBuf)`, `#write` replaced by `STREAM_CODEC`
    - `ServerboundAcceptTeleportationPacket` is now a record
        - The constructor now takes in the `double` xyz and `float`  xy rotation
    - `ServerboundChatCommandSignedPacket#STREAM_CODEC` can now work with a `ByteBuf`
    - `ServerboundChatPacket` now uses an `Optional` for signature instead of a nullable
    - `ServerboundChatSessionUpdatePacket#STREAM_CODEC` can now work with a `ByteBuf`
    - `ServerboundCommandSuggestionPacket` is now a record
    - `ServerboundMoveVehiclePacket` now takes in a `PositionAndRotation` instead of a `Vec3` potion and `float` xy rotations
    - `ServerboundSignUpdatePacket` is now a record
        - The constructor now takes in a `SignTextSlot` instead of a `boolean` for front text
    - `ServerboundSwingPacket` -> `ServerboundPunchPacket`
    - `ServerboundUseItemOnPacket` is now a record
    - `ServerboundUseItemPacket` is now a record
    - `ServerGamePacketListener#handleAnimate` -> `handlePunch`
- `net.minecraft.server.MinecraftServer`
    - `publishServer` no longer takes in the `GameType`
    - `getStructureManager` -> `getStructureTemplateManager`
- `net.minecraft.server.commands.ItemCommands` -> `.item.ItemCommands`
- `net.minecraft.server.commands.data`
    - `BlockDataAccessor#PROVIDER` is now a `ArgProvider$Factory`
    - `Datacommands#ALL_PROVIDERS`, `TARGET_PROVIDERS`, `SOURCE_PROVIDERS` are now lists of `ArgProvider$Factory`s
    - `EntityDataAccessor#PROVIDER` is now a `ArgProvider$Factory`
    - `StorageDataAccessor#PROVIDER` is now a `ArgProvider$Factory`
- `net.minecraft.server.level`
    - `ChunkMap` no longer takes in a supplied `SavedDataStorage` for the overworld
        - `$TracketEntity` now takes in an `UpdateInterval` insetad of an `int`
    - `ServerChunkCache` no longer takes in a supplied `SavedDataStorage` for the overworld
    - `ServerEntity` now takes in an `UpdateInterval` instead of an `int`
    - `ServerLevel`
        - `getStructureManager` -> `getStructureTemplateManager`
        - `sendParticles` now have overloads that takes in a `ClientboundLevelParticlesPacket$RandomizationType`
        - `findNearestMapStructure` now has an overload that takes in the `HolderSet<Structure>` instead of the `ResourceKey`
    - `ServerPlayer#getWardenSpawnTracker` now returns the raw `WardenSpawnTracker` instead of an optional
- `net.minecraft.tags.TagNetworkSerialization$NetworkPayload` is now a record
    - `read`, `write` replaced by `STREAM_CODEC`
- `net.minecraft.util`
    - `AbortableIterationConsumer$Continuation` -> `Continuation`
        - `abortIf` - Returns an abort if the provided `boolean` is true.
        - `continueIf` - Returns a continue if the provided `boolean` is true.
    - `ARGB`
        - `multiply` now has overloads that works with `Vector3fc`s or `Vector4fc`s
        - `addRgb` now has overloads that works with `Vector3fc`s or `Vector4fc`s
        - `subtractRgb` now has overloads that works with `Vector3fc`s or `Vector4fc`s
        - `scaleRGB` now has overloads that works with `Vector3fc`s or `Vector4fc`s
        - `greyscale` now has overloads that works with `Vector3fc`s or `Vector4fc`s
        - `alphaBlend` now has overloads that works with `Vector3fc`s or `Vector4fc`s
        - `srgbLerp` now has overloads that works with `Vector3fc`s or `Vector4fc`s
    - `CommonLinks`
        - `EXTEND_REALMS_LINK` is now a `URI` instead of a `String`
        - `extendRealms` now returns a `URI`, taking in an `$ExtensionReference` instead of a `boolean` for a trial
    - `ExtraCodecs#relaiveNormalizedSubPathCodec` -> `relativeNormalizedSubPathCodec`
    - `Mth`
        - `smoothstep`, `smoothstepDerivative` now work with `float`s instead of `double`s
        - `lengthSquared` now has an overload that works with `float`s
        - `length` now has an overload that works with `float`s
    - `Util`
        - `join` now takes in `Collection`s instead of `List`s
        - `$OS`
            - `openUri` -> `Blaze3D#openUri`
            - `openPath` -> `Blaze3D#openPath`
- `net.minecraft.util.debug.DebugBrainDump(FriendlyByteBuf)`, `#write` merged into `STREAM_CODEC`
- `net.minecraft.util.filefix.FileFixerUpper#swapInFixedWorld` now takes in the `UpgradeProgress`
- `net.minecraft.util.worldupdate.UpgradeProgress` now has an overload that takes in the `NotificationService`
- `net.minecraft.world.InteractionResult$SwingSource`
    - `CLIENT` -> `PREDICTED`
    - `SERVER` -> `SERVER_ONLY`
- `net.minecraft.world.entity`
    - `Entity`
        - `hurtMarked` -> `syncVelocity`
        - `invulnerableTime` is now `private` from `public`
        - `insideEffectCollector` is now `protected` from `private`
        - `calculateViewVector` is now `static` instead of `final`
        - `calculateUpVector` is now `static` instead of `final`
        - `moveOrInterpolateTo` now has an overload that takes in the `PositionPath` instead of a `Vec3`
        - `setInvulnerable` -> `setPermanentlyInvulnerable`
        - `canSimulateMovement` is now `final`
    - `EntityFluidInteraction`
        - `update` now returns if the entity was pushed by a fluid
        - `$Tracker` -> `$CurrentAccumulator`
            - `$Tracker` now just tracks the height of a given fluid, which can be reset via `$Tracker#reset`
            - `accumulateCurrent` -> `accumulate`
            - `applyCurrentTo` -> `applyTo`
    - `EntityType` now takes in a `boolean` for whether to track an entity's delta movement
        - `getSpawnAABB` now has an overload that takes in a `Vec3`
    - `InterpolationHandler` is now an `interface` from a `class`
        - It's original implementation has been moved to `AbstractInterpolationHandler` and `LinearInterpolationHandler`
        - `setInterpolationLength` -> `LinearInterpolationHandler#setInterpolationLength`
        - `position`, `yRot`, `xRot` merged into `target`
    - `LivingEntity`
        - `INVULNERABLE_DURATION` -> `DAMAGE_COOLDOWN_DURATION`
        - `swinging`, `swiningArm`, `swingTime`, `oAttackAnim`, `attackAnim` merged into `swingState`, now `private` from `public`
        - `interpolation` replaced by `Entity#interpolationHandler`
        - `drop` now takes in a `Prediction` instead of a random `boolean`
        - `getVisibilityPercent` now takes in a `ServerLevel`
        - `getLastDamageSource` now has an overload that takes in a tick timeout for when the damage source was received
        - `swing(InteractionHand)` -> `swingAndResetAttackStrength`, now taking in a `SwingAnimation` and a `boolean` for whether to sync the swinging motion
            - The original implementation has been moved to `Mob#swingForAttack` or `Mob#swing(InteractionHand, SwingAnimation)`
        - `swing(InteractionHand, boolean)` now takes in a `SwingAnimation`, returning whether the entity has made a successful swing
        - `updateSwingTime` -> `$SwingState#tick`
        - `getAttackAnim` replaced by `getCurrentSwing` or `getSwingAnimation`
        - `createItemStackToDrop` is now `public` from `private`
        - `randomTeleport` now takes in a `TagKey` or a `Predicate` of invalid `BlockState`s to teleport to
        - `startSleeping` now returns whether the entity has fallen asleep
        - `onEquippedItemBroken` now takes in an `ItemStack` instead of an `Item`
        - `blockUsingItem`, `blockedByItem` now takes in a `boolean` for whether the damage was fully blocked
    - `Mob#clampHeadRotationToBody` is now `public` from `protected`
- `net.minecraft.world.entity.ai.behavior`
    - `GoAndGiveItemsToTarget` no longer rquires the entity to be an `InventoryCarrier`
        - The constructor now takes in the cooldown `MemoryModuleType`, how long to cooldown for, and a `Predicate` for whether it can accept the item
        - `$ItemThrower#onItemThrown` -> `throwItem`, no longer taking in the `ItemStack`, instead taking a `Vec3` instead of the `BlockPos`, not one-to-one
    - `Swim` now has an overload that takes in the fluid `TagKey`
    - `TryFindLandNearWater` -> `TryFindLandNearLiquid`, not one-to-one
    - `TryFindLiquid` -> `TryFindWater`, not one-to-one
- `net.minecraft.world.entity.ai.goal`
    - `CatLieOnBedGoal` -> `CatLieOnBlockGoal`, not one-to-one
    - `FloatGoal` now has an overload that takes in the fluid `TagKey`
    - `PanicGoal#lookForWater` now takes in a `LevelReader` instead of the `BlockGetter`
    - `TryFindWaterGoal` -> `TryFindLiquidGoal`, not one-to-one
- `net.minecraft.world.entity.ai.gossip.GossipContainer#transferFrom` now returns the number of new gossips.
- `net.minecraft.world.entity.ai.navigation.PathNavigation#moveTo` now has an overload that takes in an `int` reach range
- `net.minecraft.world.entity.decoration`
    - `ArmorStand#kill` now takes in the `Entity` the kill is attributed to
    - `BlockAttachedEntity#kill` now takes in the `Entity` the kill is attributed to
- `net.minecraft.world.entity.monster`
    - `EnderMan` -> `Enderman`, not one-to-one
    - `Shulker#MAX_TELEPORT_DISTANCE` is now `public` from `private`
- `net.minecraft.world.entity.monster.cubemob.AbstractCubeMob`
    - `setcubeMobHealth` -> `setCubeMobHealth`
    - `isDealsDamage` -> `canDealDamage`
- `net.minecraft.world.entity.npc.villager.Villager#FOOD_POINTS` replaced by `VillagerFood`
- `net.minecraft.world.entity.player`
    - `Inventory`
        - `clearOrCountMatchingItems` now takes in a `boolean` for whether to count the items only without clearing
        - `placeItemBackInInventory` now takes in a `Prediction`
    - `Player`
        - `openTextEdit` now takes in a `SignTextSlot` instead of a `boolean`
        - `startSleepInBed` now takes in an `AbstractBedBlock`, the `BlockState`, and the `BedRule`
    - `ProfilePublicKey$Data(FriendlyByteBuf)`, `#write` replaed by `STREAM_CODEC`
- `net.minecraft.world.entity.projectile`
    - `Projectile`
        - `deflect` can now take in a power `double` or `Vec3`
        - `onItemBreak` now takes in an `ItemStack` instead of an `Item`
        - `mayBreak` now takes in a `BlockPos`
    - `ProjectileDeflection#deflect` now takes in the `Vec3` power
    - `ProjectileUtil#getManyEntityHitResult` now takes a `boolean` fo whether to use the projected surface hit, or the outside hit location
- `net.minecraft.world.entity.vehicle.boat.AbstractBoat#clampRotation` now returns the rotation difference
- `net.minecraft.world.entity.vehicle.minecart`
    - `MinecartBehavior#getInerpolation` replaced by `onInterpolationStart`, not one-to-one
    - `OldMinecartBehavior#onInterpolation` replaced by `onInterpolationStart`, not one-to-one
- `net.minecraft.world.item`
    - `HangingSignItem` now extends `StandingAndWallBlockItem`
    - `ItemStack`
        - `hurtAndBreak` now takes in a `Consumer<ItemStack>` instead of a `Consumer<Item>` on break
        - `addToTooltip` now has an overload that takes in a `TooltipProvider$Getter`
        - `getSwingAnimation` split into `getAttackAnimation`, `getInteractAnimation`
    - `ItemStackTemplate#fromNonEmptyStack` now has an overload that takes in a new count `int`
    - `ProjectileItem$DispenseConfig` now takes in an `Optional<Integer>` instead of an `OptonalInt` for overriding the dispenser level event
    - `SignApplicator#tryApplyToSign` now takes in a `SignTextSlot` instead of a `boolean` for whether it is front text
- `net.minecraft.world.item.enchantment`
    - `ConditionalEffect` now takes in a holder `LootItemCondition` instead of the raw condition for the requirements
    - `EnchantedItemInUse` now takes in an `ItemStack` consumer instead of an `Item` on break
    - `EnchantmentHelper`
        - `doPostAttackEffectsWithItemSourceOnBreak` now takes in an `ItemStack` consumer instead of an `Item` on break
        - `onProjectileSpawned` now takes in an `ItemStack` consumer instead of an `Item` on break
        - `onHitBlock` now takes in an `ItemStack` consumer instead of an `Item` on break
- `net.minecraft.world.item.trading`
    - `TradeRebalanceVillagerTrades#register` now returns nothing
    - `TradeSets#register` now returns nothing
    - `VillagerTrades#register` now returns nothing
- `net.minecraft.world.level`
    - `Level`
        - `playSound` now has an overload that takes in a `Holder<SoundEvent>` instead of the raw sound
        - `createFireworks` now takes in a `boolean` for whether to play a sound
        - `LevelsendBlockAndUpdate` -> `LevelWriter#setBlockAndUpdate`
    - `LevelReader` now implements `BiomeResolver` instead of `BiomeManager$NoiseBiomeSource`
        - `findBlocksIn`, `findBlocksInBoxByManhattanDistance`, `findBlocksInManhattan` - Returns a matcher looking for blocks in the specified range in the given order.
    - `NaturalSpawner`
        - `createState` now takes in a `ServerLevel` instead of an iterable of `Entity`s
        - `spawnMobsForChunkGeneration` now takes in the source `BlockPos` instead of a holder `Biome`
        - `$SpawnPredicate#test` now takes in a `ServerLevel`
    - `StructureManager`
        - `startsForStructure` now takes in `int`s for the XZ coordinates instead of a `ChunkPos` or `SectionPos`
        - `getStartForStructure`, `setStartForStructure`, `addReferenceForStructure` no longer takes in the `SectionPos`
        - `getStructureAt` now has an overload that takes in a holder `Structure` instead of the raw structure
        - `getStructureWithPieceAt` now takes in `int`s for the XYZ coordinates instead of a `BlockPos`
        - `structureHasPieceAt` now has an overload that takes in `int`s for the XYZ coordinates instead of a `BlockPos`
        - `getAllStructuresAt` now has an overload that takes in `int`s for the XYZ coordinates instead of a `BlockPos`
- `net.minecraft.world.level.block.entity.trialspawner`
    - `TrialSpawner#overrideEntityToSpawn` now has an overload that takes in `TypedEntityData` isntead of the `EntityType`
    - `TrialSpawnerConfig#withSpawning` now has an overload that takes in `TypedEntityData` isntead of the `EntityType`
- `net.minecraft.world.level.block.state.StateHolder#NAME_TAG` replaced by `ID_TAG`
- `net.minecraft.world.level.chunk.storage.SerializableChunkData` no longer takes in a `long[]` for the carving mask
- `net.minecraft.world.level.material.PushReaction`
    - `NORMAL` -> `PUSH_PULL`
    - `DESTROY` -> `POPPED`
    - `BLOCK` -> `IMMOVEABLE`
    - `IGNORE` -> `IGNORE_ENTITY`
    - `PUSH_ONLY` -> `PUSH`
- `net.minecraft.world.level.pathfinder`
    - `Node#writeToStream`, `createFromStream`, `readContents`, replaced by `DEBUG_STREAM_CODEC`, `createDebugStreamCodec`
    - `Path`
        - `STREAM_CODEC` replaced by `DEBUG_STREAM_CODEC`
        - `$DebugData` now takes in lists instead of arrays of `Node`s
            - `read`, `write` merged into `STREAM_CODEC`
    - `Target#createFromStream` replaced by `DEBUG_STREAM_CODEC`
- `net.minecraft.world.level.portal.PortalShape#FRAME` is now `public` from `private`
- `net.minecraft.world.level.saveddata.maps.MapDecorationType` no longer takes in the map color `int` or `boolean` for if it has an exploration element
- `net.minecraft.world.timeline.Timelines#NIGHT_FOG_COLOR_MULTIPLIER_START`, `NIGHT_FOG_COLOR_MULTIPLIER_END` merged into `NIGHT_FOG_COLOR_MULTIPLIER`, not one-to-one

### List of Removals

- `net.minecraft.SharedConstants`
    - `DEBUG_CARVERS`
    - `DEBUG_PREFER_WAYLAND`
- `net.minecraft.client.server.IntegratedServer#getGameTypeForOtherPlayers`, `setGameTypeForOtherPlayers`, `applyGameTypeToPlayers`
- `net.minecraft.commands.arguments`
    - `ResourceArgument#getConfiguredFeature`, `getStructure`, `getEntityType`
    - `ResourceKeyArgument#getConfiguredFeature`
- `net.minecraft.core.BlockPos`
    - `BlockPos(Vec3i)`
    - `squareOutSouthEast`
    - `findClosestMatch`
    - `withinManhattanStream`
- `net.minecraft.nbt.NbtUtils`
    - `writeFluidState`
    - `prettyPrint`
- `net.minecraft.network`
    - `Connection#setIntendedProfileId`, `getIntendedProfileId`
    - `FriendlyByteBuf`
        - `limitValue`
        - `readCollection`, `writeCollection`
        - `readList`
        - `readIntIdList`, `writeIntIdList`
        - `readMap`, `writeMap`
        - `readInstant`, `writeInstant`
        - `readPublicKey`, `writePublicKey`
        - `readBlockHitResult`, `writeBlockHitResult`
- `net.minecraft.server.level.ChunkMap#getStorageName`
- `net.minecraft.server.network.ServerConnectionListener#acceptChannel`
- `net.minecraft.server.players.PlayerList#setAllowCommandsForAllPlayers`, `isAllowCommandsForAllPlayers`
- `net.minecraft.util`
    - `KeyDispatchDataCodec`
    - `Util$OS#openFile`, `getOpenUriArguments`
- `net.minecraft.world.entity.ai.behavior.Swim#shouldSwim`
- `net.minecraft.world.entity.ai.memory.MemoryModuleType#IS_TEMPTED`
- `net.minecraft.world.entity.npc.villager.Villager#assignProfessionWhenSpawned`
- `net.minecraft.world.entity.player.Player`
    - `drop(ItemStack, boolean)`
    - `awardRecipesByKey`, `resetRecipes`
    - `getWardenSpawnTracker`
- `net.minecraft.world.item`
    - `BedItem`
    - `BlockItem#updateCustomBlockEntityTag`
- `net.minecraft.world.level`
    - `GameType#getNullableId`, `byNullableId`
    - `StructureManager#hasAnyStructureAt`
- `net.minecraft.world.level.saveddata.maps`
    - `MapDecorationType#NO_MAP_COLOR`, `hasMapColor`
    - `MapItemSavedData#isExplorationMap`
- `net.minecraft.world.phys.Vec3#applyLocalCoordinatesToRotation`, `addLocalCoordinates`
