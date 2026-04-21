# Minecraft 26.1.x -> 26.2 Mod Migration Primer

This is a high level, non-exhaustive overview on how to migrate your mod from 26.1.x to 26.2. This does not look at any specific mod loader, just the changes to the vanilla classes.

This primer is licensed under the [Creative Commons Attribution 4.0 International](http://creativecommons.org/licenses/by/4.0/), so feel free to use it as a reference and leave a link so that other readers can consume the primer.

If there's any incorrect or missing information, please file an issue on this repository or ping @ChampionAsh5357 in the Neoforged Discord server.

## Pack Changes

There are a number of user-facing changes that are part of vanilla which are not discussed below that may be relevant to modders. You can find a list of them on [Misode's version changelog](https://misode.github.io/versions/?id=26.2&tab=changelog).

## Too Many Rendering Changes

### Vulkan

Vanilla has added support for the Vulkan graphics API, which can be set in the options menu. As such, for all classes in `com.mojang.blaze3d.opengl`, there is generally a parallel found in `com.mojang.blaze3d.vulkan`. With this change, more of the Blaze3d components has been generalized further.

### Blend Factors

`DestFactor` and `SourceFactor` have been combined into a single `BlendFactor`, which is used within `BlendEquation`s and `BlendFunction`s. How these `BlendFactor`s are applied is through `BlendOp`, which defines the operation to merge the source and destination. `BlendOp#ADD` is used as a default when not specified in a `BlendFunction`.

### GPU Formats

`TextureFormat` and the buffer formats of `VertexFormatElement` have been replaced by the `GpuFormat` enum. The names of each enum entry can be broken down into three components: a channel with their bit size, an underscore, and finally the data type. Here are some examples:

- `R8_UNORM`: An unsigned normalized value with a single 8-bit red channel.
- `RGBA32_SINT`: A signed integer with 32-bit red, green, blue, and alpha channels.
- `D32_FLOAT_S8_UINT`: A float with a 32-bit depth channel, and a unsigned integer with an 8-bit stencil channel.

The previous data types of `RGBA8`, `RED8`, `RED8I`, `DEPTH32` can be represented using one of these `GpuFormat`s, though their mappings do not have one-to-one parity for the GL codes previously used.

### Bind Group Layouts

`BindGroupLayout` is a collection of sampler names and `$UniformDescription`s, replacing the single list of sampler names and `RenderPipeline#UniformDescription`. Due to this separate object, `RenderPipeline$Snippet`s that were made up of only samplers and uniforms are now decoupled, allowing for a modular, non-repeatable implementation when dealing with many pipelines and layouts. All vanilla layouts are stored in `BindGroupLayouts` and are attached to a `RenderPipeline` via `RenderPipeline$Builder#withBindGroupLayout`.

```java
public static final BindGroupLayout EXAMPLE_LAYOUT = BindGroupLayout.builder()
    // Specifies that the shaders have a 'Sampler0' sampler
    .withSampler("Sampler0")
    // Specifies that the shaders have access to the 'Globals' uniform
    .withUniform("Globals", UniformType.UNIFORM_BUFFER)
    .build();

public static final RenderPipeline EXAMPLE_PIPELINE = RenderPipeline.builder()
    .withLocation(Identifier.fromNamespaceAndPath("examplemod", "pipeline/example"))
    .withVertexShader(Identifier.fromNamespaceAndPath("examplemod", "example_shader"))
    .withFragmentShader(Identifier.fromNamespaceAndPath("examplemod", "example_shader"))
    .withVertexFormat(DefaultVertexFormat.ENTITY, VertexFormat.Mode.QUADS)
    // Specify the layouts to use
    .withBindGroupLayout(EXAMPLE_LAYOUT)
    // Can use multiple layouts
    .withBindGroupLayout(BindGroupLayouts.MATRICES_PROJECTION)
    .build();
```

Note that all `BindGroupLayout`s added to a `RenderPipeline` must not contain any duplicate entries (e.g. two shaders named `Sampler0`, two uniforms named `Globals`), or an error will be thrown during the shader compilation process.

### Render Areas

`RenderPass`es can now define a rectangle within a texture to draw its data to via `CommandEncoder#createRenderPass`. If this not specified, it defaults to the size of the color `GpuTextureView`.

### Gui Reorganization

The `Gui` class has been reorganized to handle all of the components of the graphical user interface (as the name implies). As such, fields like the current `Screen` or `ChatListener` have been moved off of their previous class (e.g. `Minecraft`) and into `Gui`. Additionally, the in-game heads-up display has been moved into a separate class called `Hud`, which `Gui` takes in.

As such, some field and method calls will need to go through `Gui` or `Hud`:

```diff
// Getting the current screen
- Minecraft.getInstance().screen
+ Minecraft.getInstance().gui.screen()

// Setting the overlay message
- Minecraft.getInstance().gui.setOverlayMessage(...)
+ Minecraft.getInstance().gui.hud.setOverlayMessage(...)
```

### Font Preparations

`Font` draw methods have been completely removed, and instead need to be prepared and handled manually. This should only be done if your text cannot be submitted as a feature via `OrderedSubmitNodeCollector#submitText`.

The text rendering pipeline works like so: the `Font` prepares the text into a `$PreparedText`, which can visit the glyphs in the string. Then, a `$GlyphVisitor` loops through all the glyph components, which typically call `$GlyphVisitor#acceptRenderable`. `acceptRenderable` is where the text is uploaded to the buffer, ready to be drawn to the screen.

```java
// Font has `prepareText` if don't want to implement the glyph parsing manually
// For some Font font
Font.PreparedText text = font.prepareText(
    // The text to draw
    "Hello world!",
    // The position of the text on the string
    0, 0,
    // The color of the text
    0xFF00FF00,
    // Whether to draw the drop shadow
    false,
    // The background color of the text
    0
);

// We can create a glyph visitor to loop through the text components
public class ExampleVisitor implements Font.GlyphVisitor {

    @Override
    public void acceptRenderable(TextRenderable renderable) {
        // Both glyphs and effects call this by default
        // If you want more specialized rendering, override:
        // - 'acceptGlyph' for text characters
        // - 'acceptEffect' for any effects applied to the characters

        // Render to the buffer
        VertexConsumer buffer = Minecraft.getInstance().gameRenderer.renderBuffers().bufferSource()
            .getBuffer(renderable.renderType(Font.DisplayMode.NORMAL);
        renderable.render(new Matrix4f(), buffer, 0xF000F0, false);
    }
}

// Then, we can visit all the renderables in our prepared text,
// which will upload them to the buffer.
text.visit(new ExampleVisitor());
```

### Submitting Gizmos

Gizmos are now rendered using the `GizmoFeatureRenderer`. Gizmos that do not call `GizmoProperties#setAlwaysOnTop` are render after translucents but before the translucent terrain features, while those that do are rendered after all other features.

### Dispatching Picture-In-Picture

Direct access to the `MultiBufferSource$BufferSource` has been removed from the `PictureInPictureRenderer`. Instead `prepare` now takes in the `FeatureRenderDispatcher` to render the elements to a texture, while `renderToTexture` takes in the `SubmitNodeCollector`.

```java
public class ExamplePIPRenderer extends PictureInPictureRenderer<ExamplePIPRenderState> {

    // The constructor is no longer required as it doesn't take in the `MultiBufferSource$BufferSource`.

    @Override
    protected void renderToTexture(ExamplePIPRenderState renderState, PoseStack poseStack, SubmitNodeCollector submitNodeCollector) {
        // Submit elements to render to the texture here.
    }

    // ...
}
```

### Shape Outlines Feature

Shape outlines is a new render feature that replaces `ShapeRenderer`, allowing for the outline of a `VoxelShape` to be rendered via `OrderedSubmitNodeCollector#submitShapeOutline`. This takes in the `PoseStack` of where the element should be placed, the `VoxelShape` to render, the `RenderType` to use, the `color` of the outline, the `width` of the line being outlined, and whether it should be rendered before or after the translucent terrain is rendered.

- `assets/minecraft/shaders/core`
    - `rendertype_text` -> `text`
    - `rendertype_text_background` -> `text_background`
    - `rendertype_text_background_see_through` -> `text_background` with `IS_SEE_THROUGH` shader define
    - `rendertype_text_intensity` -> `text` with `IS_GRAYSCALE` shader define
    - `rendertype_text_intensity_see_through` -> `text` with `IS_GRAYSCALE` and `IS_SEE_THROUGH` shader defines
    - `rendertype_text_see_through` -> `text` with `IS_SEE_THROUGH` shader define
- `com.mojang.blaze3d`
    - `GraphicsWorkarounds` -> `GlHeuristics`, `HintsAndWorkarounds`; not one-to-one
        - `alwaysCreateFreshImmediateBuffer` is removed
    - `GpuFormat` - An object representing the format and color components of a pixel.
- `com.mojang.blaze3d.opengl`
    - `GlCommandEncoder#finishRenderPass` -> `CommandEncoder#submitRenderPass`
    - `GlConst`
        - `toGl(DestFactor)`, `toGl(SourceFactor)` -> `toGl(BlendFactor)`
        - `toGl` now has an overload taking the in the `BlendOp` to map
        - `glFormatChannelCount` - Returns the number of channels the format has.
        - `isGlFormatInteger` - Returns whether the format is backed by an integer-like data type.
        - `isFormatNormalized` - Returns whether the format uses normalized data.
        - `toGlInternalId`, `toGlExternalId`, `toGlType` now take in a `GpuFormat` instead of the `TextureFormat`
    - `GlProgram#setupUniforms` -> `setupBindGroupLayouts`, now taking a list of `BindGroupLayout`s instead of uniforms and samplers directly
    - `GlRenderPass` now takes in the default `ScissorState`
    - `GlStateManager`
        - `_blendEquationSeparate`, `glBlendEquationSeparate` - Sets the blend mode of the RGB and alpha channels.
        - `$BlendState`
            - `modeRgb` - The blend mode of the RGB color channels.
            - `modeAlpha` - The blend mode of the alpha channel.
    - `GlSurface` - The OpenGL implementation of the surface backend.
    - `GlTexture` now takes in the `GpuFormat` instead of the `TextureFormat`
    - `GlTimerQuery` replaced by `GlQueryPool`
    - `Uniform$Utb` now takes in the `GpuFormat` instead of the `TextureFormat`
- `com.mojang.blaze3d.pipeline`
    - `BindGroupLayout` - The samplers and uniforms a shader will use.
    - `BlendEquation` - An equation that blends the source and destination factors using the provided operation.
    - `BlendFunction` now takes in `BlendEquation`s, `BlendFactor`s, or `BlendOp`s instead of `SourceFactor`s and `DestFactor`s
    - `RenderPipeline` now takes in a list of `BindGroupLayout`s instead of uniforms and samplers directly
        - `getSamplers`, `getUniforms` -> `getBindGroupLayouts`
            - Can get the samplers and uniforms via `BindGroupLayout#flattenSamplers`, `flattenUniforms`
        - `$Builder#withUniform`, `withSampler` -> `withBindGroupLayout`
            - Can add samplers and uniforms via `BindGroupLayout$Builder#withSampler`, `withUniform`
        - `$Snippet#samplers`, `uniforms` -> `bindGroupLayouts`, not one-to-one
        - `$UniformDescription` -> `BindGroupLayout$UniformDescription`
    - `RenderTarget`
        - `blitToScreen` is removed
            - Usage replaced by `GpuSurface#blitFromTexture`
        - `blitAndBlendToTexture` now takes in a `GpuTextureView` for the output depth texture
- `com.mojang.blaze3d.platform`
    - `BackendOptions` record is removed
    - `BlendOp` - The operation performed when blending the source and destination outputs.
    - `DestFactor`, `SourceFactor` -> `BlendFactor`
    - `GLX`
        - `_initGlfw` no longer takes in the `BackendOptions`
        - `getGlfwPlatform` - Returns the currently selected platform.
    - `NativeLibrariesBootstrap` - A bootstrap for validating and loading the required native libraries.
    - `TextureUtil#writeAsPNG` now only allows `RGBA8_UNORM` formatted textures to be written to disk.
    - `Window#updateVsync` is removed
        - Usage replaced by `Window#setMode` and `WindowEventHandler#framebufferSizeChanged`
    - `WindowEventHandler#framebufferSizeChanged` - Handles when the window buffer size has changed.
- `com.mojang.blaze3d.shaders.GpuDebugOptions` now takes in a `boolean` of whether to use validation layers; only used by the Vulkan backend
- `com.mojang.blaze3d.systems`
    - `BackendCreationException` now takes in a `$Reason` and optional string list of missing capabilities
        - `getReason` - The reason for the creation error.
        - `getMissingCapabilities` - The list of missing capabilities.
        - `$Reason` - The reasons why the backend creation may throw an exception.
    - `CommandEncoder` now takes in the `TracyGpuProfiler`
        - `backend` - Returns the encoder backend.
        - `submit` - Submits the elements to the GPU and ends the frame.
        - `submitRenderPass` - Submits the current render pass, handled as part of the try-with-resources.
        - `presentTexture` -> `GpuSurface#blitFromTexture`, not one-to-one
        - `timerQueryBegin`, `timerQueryEnd` replaced by `writeTimestamp`
        - `createRenderPass` now has an overload that takes in a `$RenderArea` for where to draw to
    - `CommandEncoderBackend`
        - `isInRenderPass` -> `CommandEncoder#isInRenderPass`, now `protected` from `public`
        - `submitRenderPass` - Submits the current render pass, handled as part of the try-with-resources.
        - `presentTexture` -> `GpuSurface#blitFromTexture`, not one-to-one
        - `timerQueryBegin`, `timerQueryEnd` replaced by `writeTimestamp`
        - `createRenderPass(Supplier, GpuTextureView, OptionalInt)` is removed
        - `createRenderPass` now has an overload that takes in a `$RenderArea` for where to draw to
    - `DeviceInfo` - A record containing information about the GPU device.
    - `DeviceLimits` - A record containing information about the limits of GPU features.
    - `DeviceType` - The type of the device performing the rendering (e.g., cpu, discrete graphics, integrated graphics).
    - `GpuBackend#createDevice` can now throw a `BackendCreationException`
    - `GpuDevice`
        - `createSurface` - Creates the surface for a window to write data to.
        - `getImplementationInformation` -> `getDeviceInfo`, not one-to-one
        - `getVendor` -> `DeviceInfo#vendorName`
        - `getBackendName` -> `DeviceInfo#backendName`
        - `getVersion` -> `DeviceInfo#driverInfo`
        - `getRenderer` -> `DeviceInfo#name`
        - `getMaxTextureSize` -> `DeviceLimits#maxTextureSize`
        - `getUniformOffsetAlignment` -> `DeviceLimits#minUniformOffsetAlignment`
        - `getEnabledExtensions` -> `DeviceInfo#underlyingExtensions`
        - `getMaxSupportedAnisotropy` -> `DeviceLimits#maxAnisotropy`
        - `setVsync` -> `GpuSurface#configure`
        - `presentFrame` -> `GpuSurface#present`
        - `isZZeroToOne` -> `DeviceInfo#isZZeroToOne`
        - `createTimestampQueryPool` - Creates the query pool for getting data from the GPU.
        - `getTimestampNow` - Returns the current epoch timestamp.
    - `GpuDeviceBackend`
        - `createSurface` - Creates the surface for a window to write data to.
        - `getImplementationInformation` -> `getDeviceInfo`, not one-to-one
        - `getVendor` -> `DeviceInfo#vendorName`
        - `getBackendName` -> `DeviceInfo#backendName`
        - `getVersion` -> `DeviceInfo#driverInfo`
        - `getRenderer` -> `DeviceInfo#name`
        - `getMaxTextureSize` -> `DeviceLimits#maxTextureSize`
        - `getUniformOffsetAlignment` -> `DeviceLimits#minUniformOffsetAlignment`
        - `getEnabledExtensions` -> `DeviceInfo#underlyingExtensions`
        - `getMaxSupportedAnisotropy` -> `DeviceLimits#maxAnisotropy`
        - `setVsync` -> `GpuSurface#configure`
        - `presentFrame` -> `GpuSurface#present`
        - `isZZeroToOne` -> `DeviceInfo#isZZeroToOne`
        - `createTimestampQueryPool` - Creates the query pool for getting data from the GPU.
        - `getTimestampNow` - Returns the current epoch timestamp.
    - `GpuQueryPool` - A pool for getting query objects and their associated values from the GPU.
    - `GpuSurface` - The surface wrapper and validator.
    - `GpuSurfaceBackend` - An API for modifying and writing data to linked window using its handle.
    - `HintsAndWorkarounds` - A record containing issues with the supported GPU device and what workarounds are required.
    - `RenderPass` now takes in a `Runnable` for what to do on close, typically from a try-with-resources; and a `$RenderArea` of where to draw to
        - `writeTimestamp` - Writes the timestamp to the pool at the given index.
        - `$RenderArea` - A rectangle defining where the pass can write data to.
    - `RenderPassBackend` is no longer `AutoCloseable`
        - `isClosed` is removed
        - `writeTimestamp` - Writes the timestamp to the pool at the given index.
    - `RenderSystem`
        - `DEFAULT_DEPTH_CLEAR_VALUE` - The default clear value for the depth buffer.
        - `flipFrame` is removed
        - `getApiDescription` is removed
        - `shutdownRenderer` - Shuts down the renderer and GPU device.
        - `getModelViewMatrix` -> `getModelViewMatrixCopy`, not one-to-one
        - `initBackendSystem` no longer takes in the `BackendOptions`
        - `$AutoStorageIndexBuffer` now implements `AutoCloseable`
    - `ScissorState#setFrom` - Copies the state from another scissor.
    - `SurfaceException` - An exception thrown when operating on or with a GPU surface.
    - `TimerQuery` now implements `AutoCloseable`
        - `getInstance` replaced by using the constructor
        - `isRecording` -> `getStatus`, not one-to-one
        - `endProfile` no longer returns anything
        - `$FrameProfile` class is removed
        - `$Status` - The current status of the timer query.
    - `TracyGpuProfiler` - An implementation of the Tracy profiler (hybrid frame and sampling profiler) for the GPU.
- `com.mojang.blaze3d.textures.TextureFormat` replaced with `GpuFormat`
    - `RGBA8` -> `RGBA8_UNORM`, not one-to-one
    - `RED8` -> `R8_UNORM`, not one-to-one
        - This should be taken with a grain of salt as it does not exactly map to the previous version's GL type
    - `RED8I` -> `R8_SINT`, not one-to-one
        - This should be taken with a grain of salt as it does not exactly map to the previous version's GL type
    - `DEPTH32` -> `D32_FLOAT`, not one-to-one
        - This should be taken with a grain of salt as it does not exactly map to the previous version's GL type
- `com.mojang.blaze3d.vertex`
    - `ByteBufferBuilder$Result#size` - The size of the resulting buffer.
    - `MeshData`
        - `unpackQuadCentroids` -> `decodeQuadCentroids`, now `public` from `private`, not one-to-one
        - `vertexBufferSlice` - The resulting vertex buffer containing the `MeshData`
        - `writeSortedIndexBuffer` - Writes the sorted vertices to an index buffer.
    - `StagingBuffer` - A buffer for staging changes to write at a later point in time.
    - `Tesselator` class is removed
    - `UberGpuBuffer` now takes in the `StagingBuffer` instead of the `GpuDevice`, buffer size `int`, and `GraphicsWorkarounds`
        - `uploadStagedAllocations` now takes in the `StagingBuffer$Uploader` instead of the `CommandEncoder`
    - `VertexFormat#uploadImmediateVertexBuffer`, `uploadImmediateIndexBuffer` are removed
    - `VertexFormatElement`, `#register` now takes in the `GpuFormat` instead of the `$Type`, normalized `boolean`, and count `int`
        - `type`, `$Type` -> `GpuFormat` following the `_` in the entries
        - `normalized` -> `GpuFormat` if `NORM` is in the entry name
        - `count` -> `GpuFormat` counting `R`, `G`, `B`, `A` in the initial characters before the size
    - `VertexMultiConsumer` class is removed
        - Usage replaced by `SheetedDecalTextureGenerator` for the foil buffer
- `com.mojang.blaze3d.vulkan`
    - `Destroyable` - An interface for destroying the object, typically through `vkDestroy*`.
    - `DestructionQueue` - A queue for managing the destruction of objects at a specific time.
    - `VulkanBackend` - The Vulkan implementation of the `GpuBackend`.
    - `VulkanBindGroupLayout` - A list of layout bindings for the handle.
    - `VulkanCommandEncoder` - The Vulkan implementation of the `CommandEncoderBackend`.
    - `VulkanConst` - The constant mappings for Blaze3d to Vulkan.
    - `VulkanDebug` - A utility for debugging Vulkan objects.
    - `VulkanDevice` - The Vulkan implementation of the `GpuDeviceBackend`.
    - `VulkanGpuBuffer` - The Vulkan implementation of a `GpuBuffer`.
    - `VulkanGpuSampler` - The Vulkan implementation of a `GpuSampler`.
    - `VulkanGpuSurface` - The Vulkan implementation of the `GpuSurfaceBackend`.
    - `VulkanGpuTexture` - The Vulkan implementation of a `GpuTexture`.
    - `VulkanGpuTextureView` - The Vulkan implementation of a `GpuTextureView`.
    - `VulkanInstance` - The connection between the application and the Vulkan library.
    - `VulkanPhysicalDevice` - The installed device with Vulkan support available on the system.
    - `VulkanQueryPool` - The Vulkan implementation of a `GpuQueryPool`.
    - `VulkanQueue` - A wrapped queue on a device to submit elements.
    - `VulkanRenderPass` - The Vulkan implementation of the `RenderPassBackend`.
    - `VulkanRenderPipeline` - The Vulkan implementation of a `CompiledRenderPipeline`.
    - `VulkanUtils` - A utility for common operations within the Vulkan implementation.
- `com.mojang.blaze3d.vulkan.glsl`
    - `GlslCompiler` - Compiles the GLSL shaders into SPIR-V bytecode.
    - `IntermediaryShaderModule` - An intermediary module representing the shader after initial compilation but before binding.
    - `ShaderCompileException` - An exception thrown during the Vulkan shader compilation process.
    - `SpvcUtil` - A utility for converting a dimension to its string representation in an SPV shader.
    - `SpvSampler` - A sampler for an SPV shader.
    - `SpvUniformBuffer` - A uniform buffer for an SPV shader.
    - `SpvVariable` - A variable for an SPV shader.
- `com.mojang.blaze3d.vulkan.init`
    - `VulkanFeature` - A Vulkan feature on the device.
    - `VulkanPNextStruct` - A representation of a Vulkan struct on the `pNext` chain.
- `net.minecraft.SharedConstants#DEBUG_SIMULATE_LIBRARY_LOAD_FAILURE` - A debug flag that simulates a native library failing to load.
- `net.minecraft.client`
    - `Minecraft`
        - `UNIFORM_FONT` replaced by `FontOption#UNIFORM`
        - `ALT_FONT` replaced by `EnchantmentNames#ALT_FONT`, now `private` from `public`
        - `screen` -> `Gui#screen`
        - `levelExtractor` - Extracts the level into its render state.
        - `openChatScreen` -> `Gui#openChatScreen`
        - `setScreen` -> `Gui#setScreen`
        - `setOverlay` -> `Gui#setOverlay`
        - `renderFrame` is now `public` from `private`
        - `renderNames` is replaced by `Hud#isHidden`, `GuiRenderState#isHudHidden`
        - `getToastManager` -> `Gui#toastManager`
        - `getWaypointStyles` -> `Hud#getWaypointStyles`
        - `getSplashManager` -> `Gui#splashManager`
        - `getOverlay` -> `Gui#overlay`
        - `windowSurface` - Gets the `GpuSurface` for the current game window.
        - `getChatListener` -> `Gui#chatListener`
        - `commandHistory` -> `ChatComponent#commandHistory`, now `private` from `public`
        - `invalidateSurfaceConfiguration` - Marks the `GpuSurface` as needing configuration, typically after a change to how the framebuffer renders (e.g., screen resize).
        - `buildInitialScreens` -> `Gui#buildInitialScreens`, now `public` from `private`
        - `getMainRenderTarget` -> `GameRenderer#mainRenderTarget`
        - `useShaderTransparency` -> `GameRenderState#useShaderTransparency`
        - `renderBuffers` is removed
    - `Options`
        - `hideGui` is replaced by `Hud#isHidden`, `GuiRenderState#isHudHidden`
        - `preferredGraphicsBackend` - Gets the preferred graphics library to use when rendering the game.
        - `hasPreferredGraphicsBackendChanged` - Whether the preferred graphics library selected has changed from when the game initially started up.
    - `PreferredGraphicsApi` - The graphics libraries vanilla supports to render the game.
    - `RotatingSectionStorage` - A class for managing the order of sections in relation to a center position.
    - `SectionUpdateTracker` - A class for tracking whether a section needs to be updated.
- `net.minecraft.client.gui`
    - `Font`
        - `drawInBatch` methods are removed
            - Replaced by `Font$PreparedText` and `Font$GlyphVisitor`
        - `drawInBatch8xOutline` -> `prepare8xTextOutline`, only taking in the `FormattedCharSequence`, XY position, and outline color; not one-to-one
        - `$GlyphVisitor`
            - `forMultiBufferSource` is removed
            - `acceptRenderable` - Accepts the `TextRenderable` to operate upon, usually for rendering.
        - `$PreparedTextBuilder#discardEffects` - Removes the text's effects.
    - `Gui` now takes in the `Hud` and `GuiRenderState`
        - `SAVING_TEXT` -> `SAVING_LEVEL`, not one-to-one
        - `hud` - The heads-up display of the user interface.
        - `resetTitleTimes` -> `Hud#resetTitleTimes`
        - `extractRenderState` no longer takes in the `GuiGraphicsExtract`, now taking in two `boolean`s of whether the level should be rendered, and whether the resources are loaded
        - `extractDebugOverlay` -> `Hud#extractDebugOverlay`
        - `registerReloadListeners` - Registers any reload listeners used by the GUI.
        - `extractDeferredSubtitles` -> `Hud#extractDeferredSubtitles`
        - `tick` no longer takes in whether the screen is paused `boolean`
        - `update` - Updates the GUI components.
        - `getMobEffectSprite` -> `Hud#getMobEffectSprite`
        - `overlay` - Returns the current screen overlay.
        - `addSocialInteractionsToast` - Creates and adds a social interactions toast.
        - `setPauseScreen` - Sets the pause screen with the appropriate configuration.
        - `handleKeybinds` - Handles any keybinds that directly affects the GUI.
        - `openChatAndAddText` - Opens the chat screen and inserts the specified text.
        - `setNowPlaying` -> `Hud#setNowPlaying`
        - `setOverlayMessage` -> `Hud#setOverlayMessage`
        - `setTimes` -> `Hud#setTimes`
        - `setSubtitle` -> `Hud#setSubtitle`
        - `setTitle` -> `Hud#setTitle`
        - `clearTitles` -> `Hud#clearTitles`
        - `getChat` -> `Hud#getChat`
        - `getGuiTicks` -> `Hud#getGuiTicks`
        - `getFont` -> `Hud#getFont`
        - `getSpectatorGui` -> `Hud#getSpectatorGui`
        - `getTabList` -> `Hud#getTabList`
        - `onDisconnected` -> `Hud#onDisconnected`
        - `getBossOverlay` -> `Hud#getBossOverlay`
        - `getDebugOverlay` -> `Hud#getDebugOverlay`
        - `clearCache` -> `Hud#clearCache`
        - `extractSavingIndicator` -> `Hud#extractSavingIndicator`
        - `canInterruptScreen` - Whether the screen can be closed by an external event not from user input.
        - `setClientLevelTeardownInProgress` - Sets that the current client level is being destroyed.
    - `GuiGraphicsExtractor`
        - `entity` now takes in `Vector3fc` and `Quaternionfc`s instead of `Vector3f` and `Quaternionf`s
        - `skin` now takes in a `Model$Simple` instead of a `PlayerModel` for the model
        - `$ScissorStack#push`, `pop` now return nothing instead of the `ScreenRectangle`
    - `Hud` - The heads-up display that's rendered atop the GUI when in-game.
- `net.minecraft.client.gui.components`
    - `AbstractWidget#extractTooltipForNextRenderPass` - Extracts the tooltip to render during the next pass.
    - `DebugScreenOverlay#render3dCrosshair` is replaced by `DebugCrosshairRenderer`
    - `PlayerTabOverlay` now takes in the `Hud` instead of the `Gui`
- `net.minecraft.client.gui.components.toasts.SystemToast`
    - `multiline` is replaced by the default constructor calling `update`
    - `recalculateWidth` - Recalculates the maximum width of the toast, up to 190.
- `net.minecraft.client.gui.contextualbar`
    - `ContextualBarRenderer` -> `ContextualBar`
    - `ExperienceBarRenderer` -> `ExperienceBar`
    - `JumpableVehicleBarRenderer` -> `JumpableVehicleBar`
    - `LocatorBarRenderer` -> `LocatorBar`
- `net.minecraft.client.gui.font.GlyphRenderTypes#createForIntensityTexture` -> `createForGrayscaleTexture`
- `net.minecraft.client.gui.render`
    - `GuiItemAtlas` no longer takes in the `MultiBufferSource$BufferSource`
    - `GuiRenderer#render` no longer takes in the `GpuBufferSlice` for the fog buffer
- `net.minecraft.client.gui.renderer.pip` no longer takes in the `MultiBufferSource$BufferSource` in any constructor
    - `PictureInPictureRenderer` no longer takes in the `MultiBufferSource$BufferSource`
        - `bufferSource` is removed
        - `prepare` now takes in the `FeatureRenderDispatcher`
        - `renderToTexture` now takes in the `SubmitNodeCollector`
- `net.minecraft.client.gui.screens.Overlay#isPauseScreen` -> `isPausing`
- `net.minecraft.client.gui.screens.inventory.AbstractSignEditScreen#getSignTextScale` now returns a `Vector3fc` instead of a `Vector3f`
- `net.minecraft.client.main.GameConfig$GameData` now takes in `boolean`s for whether to use the validation layers for Vulkan, and the graphics api forced by launch argument
    - `vulkanValidation` - Whether to use the validation layers for Vulkan.
    - `forcedGraphicsApi` - The graphics library forced by launch argument.
- `net.minecraft.client.model.Model#renderToBuffer(PoseStack, VertexConsumer, int, int)` is removed
- `net.minecraft.client.model.monster.slime.SulfurCubeModel` - The entity model for the sulfur cube.
- `net.minecraft.client.multiplayer`
    - `ClientLevel` now takes in a `LevelExtractor` instead of the `LevelRenderer`
        - `onSectionBecomingNonEmpty` is removed
    - `ClientPacketListener#getPlayerCompiledSectionCallback` - Gets the callback for when the section the player is on the client has finished loading.
    - `LevelLoadTracker`
        - `startClientLoad` no longer takes in the `LevelRenderer`
        - `getPlayerCompiledSectionCallback` - Gets the callback for when the section the player is on the client has finished loading.
- `net.minecraft.client.particle`
    - `BreakingItemParticle$SulfurCubeProvider` - A breaking item particle provider for the sulfur cube.
    - `NoxiousGasCloudParticle` - A particle for the noxious gas cloud.
    - `NoxiousGasParticle` - A particle for the noxious gas.
    - `SulfurBubbleParticle` - A particle for the sulfur bubbles.
- `net.minecraft.client.profiling.ClientMetricsSamplersProvider` now takes in a `LevelExtractor`
- `net.minecraft.client.renderer`
    - `BindGroupLayouts` - A collection of common uniform and sampler layouts used by a renderer.
    - `DebugCrosshairRenderer` - A renderer for the debug 3D crosshair reticle.
    - `DynamicUniforms#writeTransform` now has a view overloads for taking in the various transform parameters, or the `$Transform` object itself
    - `GameRenderer` no longer takes in the `RenderBuffers`
        - `renderBuffers` - The render buffers used for sections, outlines, and crumbling overlays.
        - `getSubmitNodeStorage` -> `submitNodeStorage`
        - `getFeatureRenderDispatcher` -> `featureRenderDispatcher`
        - `getGameRenderState` -> `gameRenderState`
        - `getNightVisionScale` -> `nightVisionScale`
        - `update` no longer takes in whether to advance the game time
        - `getMinecraft` is removed
        - `getBossOverlayWorldDarkening` -> `bossOverlayWorldDarkening`
        - `getMainCamera` -> `mainCamera`
        - `getGlobalSettingsUniform` is removed
        - `getLighting` -> `lighting`
        - `getPanorama` -> `panorama`
    - `LevelRenderer` has been split between this class and `LevelExtractor`
        - The class no longer implements `ResourceManagerReloadListener`
        - `LevelRenderer(Minecraft, EntityRenderDispatcher, BlockEntityRenderDispatcher, RenderBuffers, GameRenderState, FeatureRenderDispatcher)` -> `LevelRenderer(EntityRenderDispatcher, BlockEntityRenderDispatcher, ModelManager, TextureManager, AtlasManager, ShaderManager, GameRenderer, int, int)`
        - `SECTION_SIZE` -> `SectionPos#SECTION_SIZE`
        - `HALF_SECTION_SIZE` -> `SectionOcclusionGraph#HALF_SECTION_SIZE`
        - `NEARBY_SECTION_DISTANCE_IN_BLOCKS` -> `SectionRenderDispatcher#NEARBY_SECTION_DISTANCE_IN_BLOCKS`
        - `debugRenderer` -> `LevelExtractor#debugRenderer`
        - `gameTestBlockHighlightRenderer` -> `LevelExtractor#gameTestBlockHighlightRenderer`
        - `destructionProgress` -> `ClientLevel#destructionProgress`
        - `initOutline` is removed, merged into the main constructor
        - `shouldShowEntityOutlines` -> `LevelExtractor#shouldShowEntityOutlines`, now `private` from `public`
        - `setLevel` -> `resetLevelRenderData`, not one-to-one
        - `allChanged` -> `invalidateCompiledGeometry`, not one-to-one
        - `getSectionStatistics` -> `LevelExtractor#sectionStatistics`
        - `getSectionRenderDispatcher` -> `sectionRenderDispatcher`
        - `getTotalSections` -> `LevelExtractor#totalSections`
        - `getLastViewDistance` -> `LevelExtractor#lastViewDistance`
        - `countRenderedSections` -> `LevelExtractor#countRenderedSections`
        - `resetSampler` -> `LevelExtractor#resetSampler`, not one-to-one
        - `getEntityStatistics` -> `LevelExtractor#entityStatistics`
        - `offsetFrustum` -> `SectionOcclusionGraph#offsetFrustum`, now `private` from `public`
        - `addRecentlyCompiledSection` is removed, now passed into `SectionRenderDispatcher`
        - `update` is removed
            - Closest use is `RotatingSectionStorage#repositionCenter` calls
        - `renderLevel` -> `render`, no longer taking in the `ChunkSectionsToRender`
        - `extractLevel` -> `LevelExtractor#extract`, not one-to-one
        - `tick` is removed
            - Logic moved to `ClientLevel#tick`
        - `blockChanged` -> `LevelExtractor#blockChanged`, no longer taking in the `BlockGetter` or `BlockState`s
        - `setBlocksDirty` -> `LevelExtractor#setBlocksDirty`
        - `setSectionDirtyWithNeighbors` -> `LevelExtractor#setSectionDirtyWithNeighbors`
        - `setSectionRangeDirty` -> `LevelExtractor#setSectionRangeDirty`
        - `setSectionDirty` -> `LevelExtractor#setSectionDirty`
        - `onSectionBecomingNonEmpty` is removed
        - `destroyBlockProgress` -> `Level#destroyBlockProgress`
        - `onChunkReadyToRender` is removed
        - `needsUpdate` is removed
        - `getLightCoords` -> `LightCoordsUtil#getLightCoords`
        - `entityRenderDispatcher` - Gets the dispatcher for rendering entities.
        - `blockEntityRenderDispatcher` - Gets the dispatcher for rendering block entities.
        - `getTranslucentTarget` -> `translucentTarget`
        - `getItemEntityTarget` -> `itemEntityTarget`
        - `getParticlesTarget` -> `particlesTarget`
        - `getWeatherTarget` -> `weatherTarget`
        - `getCloudsTarget` -> `cloudsTarget`
        - `getVisibleSections` -> `visibleSections`
        - `getSectionOcclusionGraph` -> `sectionOcclusionGraph`
        - `getCloudRenderer` -> `cloudRenderer`
        - `skyRenderer` - Gets the sky renderer.
        - `weatherEffectRenderer` - Gets the weather renderer.
        - `worldBorderRenderer` - Gets the world border renderer.
        - `viewArea` - Gets the currently viewed area of the camera.
        - `nearbyVisibleSections` - Gets the visible sections that are nearby to the camera.
        - `collectPerFrameGizmos` -> `collectPerFrameRenderThreadGizmos`
        - `addMainThreadGizmos` - Adds gizmos from the main thread.
        - `$BrightnessGetter` -> `LightCoordsUtil$BrightnessGetter`
    - `MultiBufferSource`
        - `immediate` -> `create`, not one-to-one
        - `$BufferSource` is now `AutoCloseable`
            - The constructor takes in the initial buffer size and the fixed `RenderType`s instead of the shared and fixed buffers
            - `sharedBuffer` -> `stagedBuffer`, now `private` from `protected`, not one-to-one
            - `fixedBuffers` -> `fixedTypes`, now `private` from `protected`, not one-to-one
            - `startedBuilders` -> `fixedDraws`, now `private` from `protected`, not one-to-one
            - `lastSharedType` is removed
            - `endLastBatch` is removed
            - `endBatch` -> `uploadAndDraw`, not one-to-one
            - `endFrame` - When everything that should be uploaded is done for the frame.
    - `OrderedSubmitNodeCollector`
        - `submitModelPart` no longer takes in the `boolean`s for whether its sheeted or has foil
        - `submitShapeOutline` - Submits a voxel shape to draw an outline of.
        - `submitCustomGeometry` can now take in the outline color
        - `submitNameTag` no longer takes in the squared distance to the camera `double`
        - `submitBreakingBlockModel` now takes in a list of `BlockStatModelPart`s instead of a `BlockStateModel` and a `long` seed
        - `submitParticleGroup` now takes in a `QuadParticleRenderState` instead of a `SubmitNodeCollector$ParticleGroupRenderer`
        - `submitGizmoPrimitives` - Submits the gizmo primitive to render.
    - `OutlineBufferSource` is now `AutoCloseable`
        - The constructor now takes in the `MultiBufferSource$BufferSource` outline buffer
        - `endFrame` - When everything that should be uploaded is done for the frame.
    - `RenderBuffers` is now `AutoCloseable`
        - `endFrame` - When everything that should be uploaded is done for the frame.
        - `crumblingBufferSource` is removed
    - `RenderPipelines`
        - Pipelines using the `DepthStencilState` have their values inverted
            - `CompareOp` depth test uses `GREATER_THAN_OR_EQUAL` instead of `LESS_THAN_OR_EQUAL`
            - Values for the two `int`s are now their additive inverses
        - `TEXT_INTENSITY` -> `TEXT_GRAYSCALE`
        - `GUI_TEXT_INTENSITY` -> `GUI_TEXT_GRAYSCALE`
        - `TEXT_INTENSITY_SEE_THROUGH` -> `TEXT_GRAYSCALE_SEE_THROUGH`
        - `DRAGON_RAYS_DEPTH` is removed
            - Merged with `DRAGON_RAYS`
        - `MATRICES_PROJECTION_SNIPPET` -> `BindGroupLayouts#MATRICES_PROJECTION`, not one-to-one
        - `FOG_SNIPPET` -> `BindGroupLayouts#FOG`, not one-to-one
        - `GLOBALS_SNIPPET` -> `BindGroupLayouts#GLOBALS`, not one-to-one
    - `SectionOcclusionGraph`
        - `invalidateIfNeeded` - Invalidates the current graph based on the camera.
        - `onChunkReadyToRender` is removed
        - `update` now takes in the `CameraRenderState` and fov instead of the `Camera` params, and the `LongOpenHashSet` has been replaced by two for added and removed empty sections
        - `updateEmptySections` - Updates the newly empty and non-empty sections.
    - `ScreenEffectRenderer` no longer takes in the `MultiBufferSource$BufferSource`
        - `renderScreenEffect` -> `submit`
    - `ShapeRenderer` -> `ShapeOutlineFeatureRenderer`, not one-to-one
    - `Sheets`
        - `DECORATED_POT_SPRITES` -> `DecoratedPotRenderer#DECORATED_POT_SPRITES`, now `private` from `public`
        - `BED_SHEET`, `BED_MAPPER`, `getBedSprite`, `createBedSprite` are removed
        - `cutoutBlockSheet`, `translucentBlockSheet` are removed
            - Replaced by direct construction calls or `*ItemSheet` variants
        - `getDecoratedPotSprite` -> `DecoratedPotRenderer#getSideSprite`, now `private` from `public`, not one-to-one
        - `colorToResourceSprite` is removed
    - `SkyRenderer` now takes in the `RenderTarget`
    - `StagedVertexBuffer` - A vertex buffer for staging draws to upload at a later point in time.
    - `SubmitNodeCollection`
        - `getModelPartSubmits` is removed
        - `getShapeOutlineSubmits` - The submitted shape outlines.
        - `getGizmoSubmits` - The submitted gizmos.
    - `SubmitNodeCollector$ParticleGroupRenderer` interface is removed
    - `SubmitNodeStorage`
        - `$*Submit` storage classes have been moved to their feature renderer class under `<feature renderer>$Sumbit`
        - `$CustomGeometrySubmit` -> `CustomFeatureRenderer$Submit`
            - The constructor now takes in a `RenderType` and outline color
        - `$ModelPartSubmit` record is removed
    - `ViewArea` no longer takes in the `Level` or `LevelRenderer`, instead taking in the min and max Y and section Y, along with the `SectionOcclusionGraph`
        - `levelRenderer` is removed
        - `level` is removed
        - `sectionGridSizeY`, `sectionGridSizeX`, `sectionGridSizeZ` are removed
        - `sections` is now `private` from `public`, not one-to-one
        - `setViewDistance`
        - `size` - The number of sections in the storage.
        - `minY`, `maxY` - The height bounds of the level.
        - `minSectionY`, `maxSectionY` - The section height bounds of the level.
        - `sectionCount` - The number of sections per chunk.
        - `getLevelHeightAccessor` is removed
        - `setDirty` is removed
        - `getRenderSectionAt` is now `public` from `protected`
    - `WeatherEffectRenderer` is now `AutoCloseable`
        - `tickRainParticles` -> `ClientLevel#tickWeatherEffects`, not one-to-one
        - `getPrecipitationAt` -> `ClientLevel#getPrecipitationAt`, now `public` from` private`
        - `extractRenderState` now takes in the `ClientLevel` instead of the `Level`, and no longer takes in the number of ticks the game has ran
    - `WorldBorderRenderer` now implements `AutoCloseable`
- `net.minecraft.client.renderer.block.BlockModelRenderState#blockLightCoords` - The packed light coordinates of the block.
- `net.minecraft.client.renderer.blockentity`
    - `BedRenderer` class is removed
    - `BlockEntityRenderDispatcher#tryExtractRenderState` now takes in whether the block entity is globally rendered
- `net.minecraft.client.renderer.chunk`
    - `CompileTaskDynamicQueue` -> `SectionTaskDynamicQueue`
    - `SectionCompiler` no longer takes in the `BlockEntityRenderDispatcher`
    - `SectionRenderDispatcher` no longer takes in the `ClientLevel` and `LevelRenderer`, instead taking in a consumer for a listener on when the section mesh has been updated
        - `setLevel` -> `setCompiler`, not one-to-one
        - `uploadGlobalGeomBuffersToGPU` -> `uploadTerrainBuffersToGpu`
        - `rebuildSectionSync` is removed
        - `$RenderSection` now implements `RotatingSectionStorage$Value`
            - `SIZE` -> `SectionPos#SECTION_SIZE`
            - `hasAllNeighbors` is removed
            - `setDirty`, `setNotDirty`, `isDirty`, `isDirtyFromPlayer` are removed
            - `resortTransparency` no longer takes in the `SectionRenderDispatcher`
            - `cancelTasks` is now `private` from `protected`
            - `createCompileTask` is now `private` from `public`, now taking in the `RenderSectionRegion` instead of the `RenderRegionCache`
            - `rebuildSectionAsync` -> `compileAsync`, now taking in the `RenderSectionRegion` instead of the `RenderRegionCache`
            - `compileSync` now takes in the `RenderSectionRegion` instead of the `RenderRegionCache`
            - `$CompileTask` -> `SectionRenderDispatcher$SectionTask`
                - `isRecompile` is now `private` from `protected`
                - `name` is removed
            - `$RebuildTask` -> `$CompileTask`
                - `region` is now `private` from `protected`
- `net.minecraft.client.renderer.entity`
    - `AbstractCubeMobRenderer` - An entity renderer for cube-like mobs.
    - `MagmaCubeRenderer` now extends `AbstractCubeMobRenderer`
    - `SlimeRenderer` now extends `AbstractCubeMobRenderer`
    - `SulfurCubeRenderer` - The entity renderer for a sulfur cube.
- `net.minecraft.client.renderer.entity.layers.SulfurCubeInnerLayer` - The inner layer of a sulfur cube.
- `net.minecraft.client.renderer.entity.state.SulfurCubeRenderState` - The render state for a sulfur cube.
- `net.minecraft.client.renderer.extract.LevelExtractor` - A class that extracts the render state for level rendering.
- `net.minecraft.client.renderer.feature`
    - `BlockFeatureRenderer` -> `BlockModelFeatureRenderer`
        - `renderSolid`, `renderTranslucent` now only take in the `SubmitNodeCollection` and a `FeatureFrameContext`
        - `renderMovingBlockSubmits` -> `MovingBlockFeatureRenderer`, not one-to-one
    - `CustomFeatureRenderer`
        - `renderSolid`, `renderTranslucent` now take in a `FeatureFrameContext` instead of the buffer sources
        - `renderOutline` - Renders the outline of the submitted geometry.
        - `$Storage#add` now takes in the outline color
    - `FeatureFrameContext` - A record that contains the context for rendering a feature.
    - `FeatureRenderDispatcher` no longer takes in the crumbling `MultiBufferSource$BufferSource`
        - `renderTranslucentParticles` -> `renderTranslucentAfterTerrain`
        - `renderAlwaysOnTop` - Render these features last, used for on top gizmos.
        - `hasAnyAlwaysOnTop` - If there are any features that should be rendered last, used for on top gizmos.
    - `FlameFeatureRenderer#renderSolid` now only takes in the `SubmitNodeCollection` and a `FeatureFrameContext`
    - `GizmoFeatureRenderer` - A feature renderer for `Gizmo`s.
    - `ItemFeatureRenderer` 
        - `getFoilBuffer(MultiBufferSource, RenderType, boolean, boolean)` is removed
            - Use the other `getFoilBuffer` instead
        - `getFoilRenderType` is removed
            - Directly merged into the other `getFoilBuffer`
        - `renderSolid`, `renderTranslucent` now only take in the `SubmitNodeCollection` and a `FeatureFrameContext`
    - `LeashFeatureRenderer#renderSolid` now only takes in the `SubmitNodeCollection` and a `FeatureFrameContext`
    - `ModelFeatureRenderer#renderSolid`, `renderTranslucent` now only take in the `SubmitNodeCollection` and a `FeatureFrameContext`
        - `$Submit` now implements `TranslucentSubmit`
    - `ModelPartFeatureRenderer` class is removed
    - `MovingBlockFeatureRenderer` - A feature renderer for moving blocks.
    - `NameTagFeatureRenderer#renderTranslucent` now only takes in the `SubmitNodeCollection` and a `FeatureFrameContext`
        - `$Storage#add` no longer takes in the squared distance to camera `double`
        - `$Submit` now implements `TranslucentSubmit`
    - `ParticleFeatureRenderer` -> `QuadParticleFeatureRenderer`
        - `renderSolid`, `renderTranslucent` now take in a `FeatureFrameContext`
    - `ShadowFeatureRenderer#renderTranslucent` now takes in a `FeatureFrameContext` instead of the `MultiBufferSource$BufferSource`
    - `ShapeOutlineFeatureRenderer` - A feature renderer for `VoxelShape` outlines.
        - `$Submit` - A submitted shape outline.
    - `TextFeatureRenderer#renderTranslucent` now takes in a `FeatureFrameContext` instead of the `MultiBufferSource$BufferSource`
- `net.minecraft.client.renderer.feature.submit`
    - `SubmitNode` - An interface that represents a submitted feature.
    - `TranslucentSubmit` - An interface that represents a submitted feature with translucency.
- `net.minecraft.client.renderer.gizmos.DrawableGizmoPrimitives`
    - `render` replaced by `submit`
    - `isEmpty` is removed
    - `$Group` is now `public` from `private`
        - `render` is removed
    - `$Line`, `$Point`, `$Quad`, `$Text`, `$TriangleFan` are now `public` from `private`
- `net.minecraft.client.renderer.rendertype`
    - `RenderSetup` no longer takes in the buffer size
        - `$RenderSetupBuilder#bufferSize` is removed
    - `RenderType`
        - `draw` -> `drawFromBuffer`, now taking in the `StagedVertexBuffer$ExecuteInfo`; not one-to-one
        - `bufferSize` is removed
    - `RenderTypes`
        - `textIntensity` -> `textGrayscale`
        - `textIntensityPolygonOffset` -> `textGrayscalePolygonOffset`
        - `textIntensitySeeThrough` -> `textGrayscaleSeeThrough`
        - `dragonRaysDepth` is removed
            - Merged with `dragonRays`
    - `TextureTransform#getMatrix` -> `createMatrix`
- `net.minecraft.client.renderer.special.BedSpecialRenderer` class is removed
- `net.minecraft.client.renderer.state.OptionsRenderState`
    - `hideGui` -> `GuiRenderState#isHudHidden`
    - `chunkSectionFadeInTime` - The amount of seconds that should be taken for a chunk to fade in when first rendered.
    - `prioritizeChunkUpdates` - What chunk updates to prioritize.
    - `fov` - The field of view of the camera.
- `net.minecraft.client.renderer.state.gui.BlitRenderState` now takes in a `Matrix3x2fc` instead of a `Matrix3x2f` for the pose
- `net.minecraft.client.renderer.state.gui.pip`
    - `GuiEntityRenderState` now takes in `Vector3fc` and `Quaternionfc`s instead of `Vector3f` and `Quaternionf`s
    - `GuiSkinRenderState` now takes in a `Model$Simple` instead of a `PlayerModel` for the model
    - `PictureInPictureRenderState#IDENTITY_POSE`, `pose` now returns a `Matrix3x2fc` instead of a `Matrix3x2f`
- `net.minecraft.client.renderer.state.level`
    - `CameraRenderState`
        - `isFrustumCaptured` - Whether the frustum has been captured.
        - `smartCull` - Whether to use smart culling.
    - `LevelRenderState`
        - `render3dCrosshair` - Whether to render the 3d crosshair reticle.
        - `chunkSectionsToRender` is removed
        - `sectionUpdateRenderStates` - The render states of the sections to update.
        - `playerCompiledSectionCallback` - Gets the callback for when the section the player is on the client has finished loading.
        - `addedEmptySections`, `removedEmptySections` - The changes between empty sections.
        - `shouldResetChunkLayerSampler` - Whether the chunk layer sampler should be reset.
        - `shouldShowEntityOutlines` - Whether to show the outlines of entities.
        - `shouldResetSkyRenderer` - Whether the sky renderer should be reinitialized.
    - `ParticlesRenderState#submit` now takes in a `SubmitNodeCollector` instead of the `SubmitNodeStorage`
    - `QuadParticleRenderState` no longer implements `SubmitNodeCollector$ParticleGroupRenderer`
        - `prepare` replaced by `buildLayer`, not one-to-one
        - `render` -> `QuadParticleFeatureRenderer#render`, not one-to-one
        - `layers` - The set of quad layers to render.
        - `$PreparedBuffers`, `$PreparedLayer` are removed
    - `SectionUpdateRenderState` - The render state of a section to update.
- `net.minecraft.client.resources.model.ModelManager$MaterialBakerImpl` - An implemented `MaterialBaker`.
- `net.minecraft.client.resources.model.sprite.SpriteId#buffer` are removed
- `net.minecraft.client.telemetry`
    - `TelemetryEventType#GRAPHICS_CAPABILITIES` - The capabilities of the graphics card.
    - `TelemetryProperty`
        - `BACKEND_NAME` - The name of the graphics backend.
        - `BACKEND_FAILURE_MESSAGE` - The message provided by the backend when failing during construction.
        - `BACKEND_FAILURE_REASON` - The reason the backend failed during construction.
        - `BACKEND_FAILURE_MISSING_CAPABILITIES` - The backend failed during construction due to missing capabilities.
- `net.minecraft.data.AtlasIds#BEDS` is removed

## Object Collections

Many vanilla objects can be considered variants of each other but are independent in nature (e.g. different colors, different copper states). Given that most of these objects have the same properties and components, vanilla created object collections to store all the variants together and operate upon them at once. As such, fields within certain registries have been condensed into a single collection (e.g. `Items#_*DYE` is now a `Items#DYE` collection).

Vanilla currently provides two collections: a `ColorCollection` for all 16 dye colors, and `WeatheringCopperCollection` for weathered and waxed blocks. The collections can be created through the constructor, but there are static constructors for blocks (`registerBlocks`) and items (`registerItems`). The collection also contain methods for common usecases, like looping through each entry (`forEach`) or picking a specific entry from its collection method (`pick`).


```java
// Create colored blocks
public static final ColorCollection<Block> EXAMPLE_BLOCK = ColorCollection.registerBlocks(
    // The base identifier for the blocks.
    // The colors will be prefixed when creating (e.g. 'white_example_block').
    "example_block",
    // The method to register the block. Takes in:
    // - The name created from the identifier.
    // - The factory taking in the properties to create the block.
    // - The block properties.
    // And returns the block.
    (name, factory, props) -> Registry.register(
        BuiltInRegistries.BLOCK,
        Identifier.fromNamespaceAndPath("examplemod", name),
        factory.apply(props)
    ),
    // The factory that creates the block. Takes in:
    // - The dye color applied to the block.
    // - The block properties.
    // And returns the block.
    (color, props) -> new Block(props),
    // Supplies the properties for the given color. Takes in:
    // - The dye color applied to the block.
    // And returns the block properties.
    color -> BlockBehavior.Properties.of()
);

// Do something for each object in the collection
EXAMPLE_BLOCK.forEach(block -> {
    // ...
});

// Map a collection to another collection
ColorCollection<Item> items = ColorCollection.map(Block::asItem);

// Map two collections together, zipping them per entry
ColorCollection<Item> items = ColorCollection.zipMap(EXAMPLE_BLOCK, ColorCollection.VALUES, (exampleBlock, dyeColor) -> {
    val id = BuiltInRegistries.BLOCK.getKey(exampleBlock);
    return Registry.register(
        BuiltInRegistries.ITEM, id,
        new Item(new Item.Properties()
            .setId(ResourceKey.create(BuiltInRegistries.ITEM, id))
            .component(DataComponents.DYED_COLOR, new DyedItemColor(dyeColor.getTextureDiffuseColor))
        )
    );
});

// Do something given two collections, zipping them per entry
ColorCollection.zipApply(EXAMPLE_BLOCK, Items.DYE, (exampleBlock, dye) -> {
    // ...
});

// Get a specific object based on the collection variant
Block pink = EXAMPLE_BLOCK.pick(DyeColor.PINK);
```

- `net.minecraft.client.data.models.BlockModelGenerators`
    - `createBanner` no longer takes in the `Block`s for the ground and wall variants
    - `createBanners` is removed
    - `createBed` no longer takes in the `Block`s for the bed and particle
    - `createBeds` is removed
- `net.minecraft.client.renderer.Sheets#CHEST_COPPER_*` merged into `CHEST_COPPER` collection
- `net.minecraft.client.renderer.block.BuiltInBlockModels#createBanners` no longer takes in the `Block`s for the ground and wall variants
- `net.minecraft.client.renderer.special.ChestSpecialRenderer#COPPER_*` merged into `COPPER` collection
- `net.minecraft.data.BlockFamilies`
    - `COPPER_BLOCK`, `WAXED_COPPER_BLOCK`, `EXPOSED_COPPER`, `WAXED_EXPOSED_COPPER`, `WEATHERED_COPPER`, `WAXED_WEATHERED_COPPER`, `OXIDIZED_COPPER`, `WAXED_OXIDIZED_COPPER` merged into `COPPER_BLOCK`, not one-to-one
    - `CUT_COPPER`, `WAXED_CUT_COPPER`, `EXPOSED_CUT_COPPER`, `WAXED_EXPOSED_CUT_COPPER`, `WEATHERED_CUT_COPPER`, `WAXED_WEATHERED_CUT_COPPER`, `OXIDIZED_CUT_COPPER`, `WAXED_OXIDIZED_CUT_COPPER` merged into `CUT_COPPER`, not one-to-one
- `net.minecraft.data.loot.EntityLootSubProvider#createSheepDispatchPool` now takes in a `ColorCollection` instead of a map
- `net.minecraft.data.loot.packs.LootData` is removed
- `net.minecraft.world.item`
    - `Items` that have different variants may have been merged into a single entry, usually represented by some `*Collection` class
    - `WeatheringCopperItems` -> `WeatheringCopperCollection`, not one-to-one
- `net.minecraft.world.item.trading.VillagerTrades` that have different variants may have been merged into a single entry, usually represented by some `*Collection` class
- `net.minecraft.world.level.block`
    - `Blocks` that have different variants may have been merged into a single entry, usually represented by some `*Collection` class
    - `ColorCollection` - A collection of objects whose variants represent one of sixteen colors.
    - `WeatheringCopperBlocks` -> `WeatheringCopperCollection`, not one-to-one
- `net.minecraft.world.level.storage.loot.BuiltInLootTables` map fields are replaced by `ColorCollection`s

## Advancements and Entity Sub Predicates

`net.minecraft.advancements` has reorganized the location of its criteria, triggers, and predicates. Criteria and triggers are now located within `net.minecraft.advancements.triggers`, whereas predicates can be found within `net.minecraft.advancements.predicates`. `EntityPredicate` along with `EntitySubPredicate` and its implementations are within `net.minecraft.advancements.predicates.entity`.

Additionally, `EntityPredicate` is now a wrapper around a map of `EntitySubPredicate`s. Since not all predicates used by the `EntityPredicate` were `EntitySubPredicate`s (e.g. `DistancePredicate`, `SlotsPredicate`), vanilla added new `EntitySubPredicate`s that wrap around these generic implementations (e.g. `DistancePredicate` -> `DistanceToPlayerPredicate`, `SlotsPredicate` -> `EntitySlotsPredicate`). Predicates that were recursive in nature (e.g. `vehicle`, `passenger`) also have an object to wrap around the `EntityPredicate` (e.g. `vehicle` now a `VehiclePredicate`, `passenger` now a `PassengerPredicate`).

`EntitySubPredicate` has also changed slightly as its now just a functional interface with `matches`. `Registries#ENTITY_SUB_PREDICATE_TYPE` now registers a `Codec` of the predicate, which is used as the key of the `EntityPredicate` map.

```java
// Create a sub predicate to map.
public class TeamColorPredicate(int color) implements EntitySubPredicate {

    // The codec to register.
    public static final Codec<TeamColorPredicate> CODEC = ExtraCodecs.STRING_RGB_COLOR.xmap(
        TeamColorPredicate::new, TeamColorPredicate::color
    );

    public TeamColorPredicate {
        this.color = ARGB.opaque(color);
    }

    @Override
    public boolean matches(Entity entity, ServerLevel level, @Nullable Vec3 position) {
        return ARGB.opaque(entity.getTeamColor()) == this.color;
    }
}

// Register the codec.
Registry.register(
    BuiltInRegistries.ENTITY_SUB_PREDICATE_TYPE,
    Identifier.fromNamespaceAndPath("examplemod", "team_color"),
    TeamColorPredicate.CODEC
);
```

So, if our `EntityPredicate` looks something like:

```java
var predicate = new EntityPredicate(Map.of(
    TeamColorPredicate.CODEC, new TeamColorPredicate(0x00FF00),
    DistanceToPlayerPredicate.CODEC, new DistanceToPlayerPredicate(DistancePredicate.vertical(MinMaxBounds.Doubles.exactly(1d)))
));
```

Then, it will be serialized into the advancement JSON as:

```json5
// In some advancement JSON

// Only the `EntityPredicate` part
{
    "examplemod:team_color": "#00FF00",
    "minecraft:distance": {
        "y": 1
    }
}
```

- `net.minecraft.advancements`
    - `CriteriaTriggers` -> `.advancements.triggers.CriteriaTriggers`
    - `Criterion` -> `.advancements.triggers.Criterion`
    - `CriterionTrigger` -> `.advancements.triggers.CriterionTrigger`
- `net.minecraft.advancements.criterion.*`
    - `*Predicate` -> `.advancements.predicates.*Predicate`
    - `*Trigger` -> `.advancements.triggers.*Trigger`
    - `BlockPredicate` -> `.advancements.predicates.BlockPredicate`
    - `CollectionContentsPredicate` -> `.advancements.predicates.CollectionContentsPredicate`
    - `CollectionCountsPredicate` -> `.advancements.predicates.CollectionCountsPredicate`
    - `CollectionPredicate` -> `.advancements.predicates.CollectionPredicate`
    - `ContextAwarePredicate` -> `.advancements.predicates.ContextAwarePredicate`
    - `DamagePredicate` -> `.advancements.predicates.DamagePredicate`
    - `DamageSourcePredicate` -> `.advancements.predicates.DamageSourcePredicate`
    - `DataComponentMatchers` -> `.advancements.predicates.DataComponentMatchers`
    - `DistancePredicate` -> `.advancements.predicates.DistancePredicate`
    - `EnchantmentPredicate` -> `.advancements.predicates.EnchantmentPredicate`
    - `EntityEquipmentPredicate` -> `.advancements.predicates.entity.EntityEquipmentPredicate`
        - Now implements `EntitySubPredicate`
    - `EntityFlagsPredicate` -> `.advancements.predicates.entity.EntityFlagsPredicate`
        - Now implements `EntitySubPredicate`
    - `EntityPredicate` -> `.advancements.predicates.entity.EntityPredicate`
        - The constructor now takes in a map of `EntitySubPredicate`s
        - `entityType` replaced by adding an `EntityTypePredicate` to the map
        - `distanceToPlayer` replaced by adding a `DistanceToPlayerPredicate` to the map
        - `movement` replaced by adding a `DistanceToPlayerPredicate` to the map
        - `location` replaced by adding some of the following to the map:
            - `EntityLocationPredicate`
            - `SteppingOnPredicate`
            - `MovementAffectedByPredicate`
        - `effects` replaced by adding a `EntityEffectsPredicate` to the map
        - `nbt` replaced by adding a `EntityNbtPredicate` to the map
        - `flags` replaced by adding a `EntityFlagsPredicate` to the map
        - `equipment` replaced by adding a `EntityEquipmentPredicate` to the map
        - `subPredicate` -> `parts`, now `private` from `public`, not one-to-one
        - `periodicTick` replaced by adding a `PeriodicEntityTickPredicate` to the map
        - `vehicle` replaced by adding a `VehiclePredicate` to the map
        - `passenger` replaced by adding a `PassengerPredicate` to the map
        - `targetedEntity` replaced by adding a `TargetedEntityPredicate` to the map
        - `team` replaced by adding a `TeamPredicate` to the map
        - `slots` replaced by adding a `EntitySlotsPredicate` to the map
        - `components` replaced by adding either `EntityExactDataComponentsPredicate` or `EntityPartialComponentsPredicate` to the map
        - `$Builder`
            - `components` now has an overload that takes in a map of `DataComponentType`s to `DataComponentPredicate`s
            - `lightingBolt` - Adds a predicate for a lightning bolt.
            - `player` - Adds a predicate for a player.
            - `sheep` - Adds a predicate for a sheep.
            - `cubeMob` - Adds a predicate for a cube mob.
            - `raider` - Adds a predicate for a raider.
            - `fishingHook` - Adds a predicate for a fishing hook.
        - `$LocationWrapper` record is removed
    - `EntitySubPredicate` -> `.advancements.predicates.entity.EntitySubPredicate`
        - `CODEC` -> `EntityPredicate#MAP_CODEC`, not one-to-one
        - `codec` is removed
        - `ALWAYS_TRUE` - A predicate that returns true.
        - `and` - Chains another predicate with this one using an `AND` statement.
    - `EntitySubPredicates` -> `.advancements.predicates.entity.EntitySubPredicates`
        - Static fields are now inlined into `bootstrap`
    - `EntityTypePredicate` -> `.advancements.predicates.entity.EntityTypePredicate`
        - Now implements `EntitySubPredicate`
    - `FishingHookPredicate` -> `.advancements.predicates.entity.FishingHookPredicate`
    - `FluidPredicate` -> `.advancements.predicates.FluidPredicate`
    - `FoodPredicate` -> `.advancements.predicates.FoodPredicate`
    - `GameTypePredicate` -> `.advancements.predicates.GameTypePredicate`
    - `InputPredicate` -> `.advancements.predicates.InputPredicate`
    - `ItemPredicate` -> `.advancements.predicates.ItemPredicate`
    - `LightningBoltPredicate` -> `.advancements.predicates.entity.LightningBoltPredicate`
    - `LightPredicate` -> `.advancements.predicates.LightPredicate`
    - `LocationPredicate` -> `.advancements.predicates.LocationPredicate`
    - `MinMaxBounds` -> `.advancements.predicates.MinMaxBounds`
    - `MobEffectsPredicate` -> `.advancements.predicates.MobEffectsPredicate`
    - `MovementPredicate` -> `.advancements.predicates.entity.MovementPredicate`
        - Now implements `EntitySubPredicate`
    - `NbtPredicate` -> `.advancements.predicates.NbtPredicate`
    - `PlayerPredicate` -> `.advancements.predicates.entity.PlayerPredicate`
    - `RaiderPredicate` -> `.advancements.predicates.entity.RaiderPredicate`
    - `SheepPredicate` -> `.advancements.predicates.entity.SheepPredicate`
    - `SingleComponentItemPredicate` -> `.advancements.predicates.SingleComponentItemPredicate`
    - `SlimePredicate` -> `.advancements.predicates.entity.CubeMobPredicate`
    - `SlotsPredicate` -> `.advancements.predicates.SlotsPredicate`
    - `StatePropertiesPredicate` -> `.advancements.predicates.StatePropertiesPredicate`
    - `TagPredicate` -> `.advancements.predicates.TagPredicate`
- `net.minecraft.advancements.predicates.entity`
    - `DistanceToPlayerPredicate` - An entity sub predicate that checks a `DistancePredicate`.
    - `EntityEffectsPredicate` - An entity sub predicate that checks a `MobEffectsPredicate`.
    - `EntityLocationPredicate` - An entity sub predicate that checks a `LocationPredicate`.
    - `EntityNbtPredicate` - An entity sub predicate that checks a `NbtPredicate`.
    - `EntityPartialComponentsPredicate` - An entity sub predicate that checks if the specified data components match the `DataComponentPredicate`.
    - `EntitySlotsPredicate` - An entity sub predicate that checks a `SlotsPredicate`.
    - `EntityTagPredicate` - An entity sub predicate that checks if an entity is in or not in `Entity#entityTags`.
    - `MovementAffectedByPredicate` - An entity sub predicate that checks a `LocationPredicate` on `Entity#getBlockPosBelowThatAffectsMyMovement`.
    - `PassengerPredicate` - An entity sub predicate that checks an `EntityPredicate` on all passengers.
    - `SteppingOnPredicate` - An entity sub predicate that checks a `LocationPredicate` on `Entity#getOnPos`.
    - `TargetedEntityPredicate` - An entity sub predicate that checks an `EntityPredicate` on `Mob#getTarget`.
    - `TeamPredicate` - An entity sub predicate that checks if an entity is on the specified team.
    - `VehiclePredicate` - An entity sub predicate that checks an `EntityPredicate` on `Entity#getVehicle`.
- `net.minecraft.core.registries.BuiltInRegistries`, `Registries#ENTITY_SUB_PREDICATE_TYPE` is now a `Codec` instead of a `MapCodec`
- `net.minecraft.server.PlayerAdvancements`
    - `stopListening` -> `clearTriggers`, not one-to-one
    - `getTriggerMapForType` - Gets the map of advancement to trigger value for the given `CriterionTrigger`.
    - `$TriggerInstanceKey` - A record that holds the advancement and associated string criterion.

## More Resource Keys and Separating Registry Objects

`Block`s, `Item`s, `EntityType`s, `BlockEntityType`s, `Fluid`s, and `Potion`s have had their `ResourceKey`s stored in a separate `*Ids` class, which is then referenced whenever an identifier is needed. Blocks with an item representation, known as block items, have their own identifiers stored as a `BlockItemId`, which is just a pair of `ResourceKey`s, in `BlockItemIds`.

Additionally, the `BlockEntityType` and `EntityType` registry entries have been moved into their own separate class: `BlockEntityTypes` and `EntityTypes`, respectively.

### Tags Provider Changes

Since all registry objects in a tag has an accessible `ResourceKey`, all utility subclasses of `TagsProvider` have been removed (i.e. `HolderTagProvider`, `IntrinsicHolderTagsProvider`, `KeyTagProvider`), with the two `KeyTagProvider#tag` methods moved into `TagsProvider`. In addition, the `TagAppender` now only takes in `ResourceKey`s, the arbitrary element mapping completely removed.

This means that either a `ResourceKey` or `TagKey` must be provided to add an element to a tag through `TagsProvider`s.

For block items and block item tags (tags that exist within the block and item registry), `BlockItemTagsProvider` has been updated to use `BlockItemId`s and `BlockItemTagId`s, which are functionally records that hold the names of the object in the block and item registries. Entries are added in the `run` block, which is then called within the block or item `TagsProvider`, respectively.

- `net.minecraft.data.tags`
    - All `*TagsProvider` that extended `TagsProvider` or one of its subclasses now extend `TagsProvider`
    - `BlockItemTagAppender` - A `TagAppender` that converts a `BlockItemId` to its appropriate object resource key, usually for a `Block` or `Item`.
    - `BlockItemTagsProvider` now takes in a function with a `BlockItemTagId` and returning a `$CombinedAppender`
        - The original behavior of this class has moved to `VanillaBlockItemTagsProvider`
        - `tag` now takes in a `BlockItemTagId` instead of the `TagKey`s for each
        - `run` is now abstract
        - `wrapForBlocks`, `wrapForItems` - Wraps around an existing block or item `TagAppender` to take in a `BlockItemId` or `BlockItemTagId`.
        - `$CombinedAppender` - An appender that allows for `BlockItemId`s and `BlockItemTagId`s to be added.
    - `HolderTagProvider` class is removed
    - `IntrinsicHolderTagsProvider` class is removed
    - `KeyTagProvider` class is removed
        - `tag` -> `KeyTagProvider#tag`
    - `TagAppender` no longer takes in the `E` element generic
        - The `E` generic in methods have been replaced with `ResourceKey<T>`
        - `map` is removed
- `net.minecraft.references`
    - `BlockIds` now contain `ResourceKey`s for all blocks without item representations available in a tag
    - `BlockItemId` - An identifier that applies to a block with an item representation.
    - `BlockItemIds` - Identifiers for vanilla blocks with an item representation.
    - `ItemIds` now contain `ResourceKey`s for all non-block items available in a tag
- `net.minecraft.resources.ResourceKey#dependent` - Creates a new resource key by modifying the path of the object `Identifier`, either via a suffix or a `UnaryOperator`.
- `net.minecraft.tags`
    - `BlockItemTagId` - An identifier that applies to a block tag with an item tag representation.
    - `BlockItemTags` - Identifiers for block tags with an item tag representation.
- `net.minecraft.world.entity`
    - `EntityType` registry objects have been moved to `EntityTypes`
        - `byString` is removed
    - `EntityTypeIds` - Identifiers for entity types.
    - `EntityTypes` - All vanilla entity types.
- `net.minecraft.world.item.alchemy.PotionIds` - Identifiers for vanilla potions.
- `net.minecraft.world.level.block.entity`
    - `BlockEntityType` registry objects have been moved to `BlockEntityTypes`
        - The constructor is now `public` from `private`
        - `$BlockEntitySupplier` is now `public` from `private`
    - `BlockEntityTypeIds` - Identifiers for block entity types.
    - `BlockEntityTypes` - All vanilla block entity types.
- `net.minecraft.world.level.material.FluidIds` - Identifiers for fluids.

## Minor Migrations

The following is a list of useful or interesting additions, changes, and removals that do not deserve their own section in the primer.

### Sulfur Cube Archetypes

Sulfur cubes behave differently depending on the item it absorbed. This is known as a `SulfurCubeArchetype`, a datapack registry object that takes in the set of `Item`s it should apply to, the cube's attributes to modify, and whether the cube should float in liquids.

```json5
// A file located at:
// - `data/examplemod/sulfur_cube_archetype/example_archetype.json`
{
    // The held items the sulfur cube must contain for this archetype to apply.
    // Can either be an item id, such as "minecraft:apple",
    // or a list of item ids, such as ["minecraft:apple", "minecraft:carrot", ...],
    // or an entity type tag, such as "#minecraft:sulfur_cube_food".
    "items": "#minecraft:sulfur_cube_food",
    // A list of attribute modifiers to apply to the sulfur cube when holding
    // the item.
    "attribute_modifiers": [
        {
            // A unique identifier representing the modifier.
            "id": "examplemod:example_arch_add_resistance",
            // The registry name of the attribute to apply.
            "attribute": "minecraft:knockback_resistance",
            // The amount to modify the attribute value.
            "amount": -2.0,
            // The operation to use when applying the modifier.
            // - 'add_value': Adds the amount to the current value.
            // - 'add_multiplied_base': Multiplies the amount to the base value after additions.
            // - 'add_multiplied_total': Multiplies the amount to the value after all modifications.
            "operation": "add_value"
        }
    ],
    // When `true`, the sulfur cube will float in liquids.
    "buoyant": true
}
```

- `net.minecraft.core.registries.Registries#SULFUR_CUBE_ARCHETYPE` - A resource key for the archetypes of a sulfur cube.
- `net.minecraft.world.entity`
    - `SulfurCubeArchetype` - The behavior of a sulfur cube when holding the given item.
    - `SulfurCubeArchetypes` - All vanilla sulfur cube archetypes.
- `net.minecraft.world.item.component.SulfurCubeContent` - The absorbed block stored in the sulfur cube.

### Shears Breaking Speed

Shears now use three tags to determine how fast it should break an block:

- `minecraft:shears_extreme_breaking_speed` - Blocks are mined with a base of `15` speed, higher than all other vanilla tools.
- `minecraft:shears_major_breaking_speed` - Blocks are mined with a base of `5` speed, the same speed as copper tools.
- `minecraft:shears_minor_breaking_speed` - Blocks are mined with a base of `2` speed, the same speed as wooden tools.

Cobweb is the only block hardcoded to use the extreme breaking speed value.

- `net.minecraft.world.entity.Entity#attemptToShearEquipment` -> `Mob#attemptToShearEquipment`, `shearItem`; now `protected` from `private`

### Structure Processor Unrolling

`StructureProcessor`s no longer use `StructureProcessorType` as its registered instance, instead now directly using the `MapCodec` used as part of the serialization and deserialization process. `StructureProcessor` is no longer as a class as well, instead being represented by an interface. As such, all new processors should implement the `StructureProcessor` interface, replacing `getType` with `codec`:

```java
public class ExampleProcessor implements StructureProcessor {

    // Create the map codec to register
    public static final MapCodec<ExampleProcessor> MAP_CODEC = MapCodec.unit(ExampleProcessor::new);

    @Nullable
    @Override
    public StructureTemplate.StructureBlockInfo processBlock(LevelReader level, BlockPos targetPosition, BlockPos referencePos, StructureTemplate.StructureBlockInfo originalBlockInfo, StructureTemplate.StructureBlockInfo processedBlockInfo, StructurePlaceSettings settings) {
        // Modify the structure block info
        return processedBlockInfo;
    }

    // Replaces getType
    @Override
    public MapCodec<ExampleProcessor> codec() {
        return MAP_CODEC;
    }
}

// Register the map codec to the registry
Registry.register(BuiltInRegistries.STRUCTURE_PROCESSOR, Identifier.fromNamespaceAndPath("examplemod", "example_processor"), ExampleProcessor.MAP_CODEC);
```

- `net.minecraft.core.registries.BuiltInRegistries`, `Registries`
    - `STRUCTURE_PROCESSOR` now holds a `StructureProcessor` map codec instead of a `StructureProcessorType`
- `net.minecraft.world.level.levelgen.structure.templatesystem`
    - All classes now implement `StructureProcessor` instead of extending
    - `CODEC` fields are renamed to `MAP_CODEC`
    - `ProtectedBlockProcessor` is now a `record`
        - `cannotReplace` now takes in a `HolderSet` of `Block`s instead of the referenced `TagKey`
    - `StructureProcessor` is now an `interface` instead of a `class`
        - `getType` -> `codec`, now returning a `MapCodec`
    - `StructureProcessorType` no longer has a generic
        - Fields for each type are now inlined into `StructureProcessorTypes#bootstrap`
        - `codec` -> `StructureProcessor#codec`
        - `register` inlined into `StructureProcessorTypes#bootstrap`
    - `StructureProcessorTypes` - All vanilla structure processor codecs.

### Game Test Entity Builders

With the growing number of setup entities require when spawning for a game test, vanilla has split the construction of such entities into builder classes: `GameTestEntityBuilder` for generic entities, and `GameTestMobBuilder` for mobs. The builders can be acquired by calling `spawnEntity` and `spawnMob`, respectively, taking in the `EntityType` and some relative position in the test.

```java
// For some game test...
public void exampleTest(GameTestHelper helper) {
    // Spawn an entity
    ArmorStand entity = helper.spawnEntity(EntityType.ARMOR_STAND, 0, 0, 0)
        // Why the entity has spawned.
        .spawnReason(EntitySpawnReason.SPAWN_ITEM_USE)
        // How the entity should be rotated relative to the test.
        .rotation(Rotation.CLOCKWISE_90)
        // Whether the entity should be persisted.
        .requirePersistence(true)
        // Spawn the entity.
        .spawn();
    
    // Spawn a mob
    Pig mob = helper.spawnMob(EntityType.PIG, 1, 0, 1)
        // Spawns the entity with no AI.
        .withNoFreeWill()
        // Can use the entity builder methods.
        .spawnReason(EntitySpawnReason.NATURAL)
        .spawn();
}
```

- `net.minecraft.gametest.framework`
    - `GameTestEntityBuilder` - A builder that creates a mock entity for a test.
    - `GameTestHelper`
        - `makeMockServerPlayer` - Creates a mock server player in the desired `GameType`.
        - `spawn` now has overloads that take in `BlockPos` for the position
        - `spawnEntity` - Creates a builder for spawning some entity at the given position.
        - `spawnMob` - Creates a builder for spawning some mob at the given position.
    - `GameTestMobBuilder` - A builder that creates a mock mob for a test.

### Data Component Additions

- `sulfur_cube_content` - The absorbed block stored in the sulfur cube.

### Entity Attribute Additions

- `air_drag_modifier` - The drag applied to an entity when traveling through the air. Must be between `[0, 2048]` and defaults to `1`.
- `bounciness` - How much the entity should bounce after colliding with another bounding box. Must be between `[0, 1]` and defaults to `0`.
- `friction_modifier` - A scalar that used to modify the block friction when traveling in air. Must be between `[0, 2048]` and defaults to `1`.
- `knockback_resistance` now allows values between `[-2, 1]`
- `below_name_distance` - The maximum distance that the additional information below an entity's nameplate can be seen. Must be between `[0, 512]` and defaults to `10`.
- `nameplate_distance` - The maximum distance that an entity's nameplate can be seen. Must be between `[0, 512]` and defaults to `64`.

### Tag Changes

- `minecraft:block`
    - `glazed_terracotta`
    - `concrete`
    - `shears_extreme_breaking_speed`
    - `shears_major_breaking_speed`
    - `shears_minor_breaking_speed`
    - `suppresses_bounce`
    - `speleothems`
    - `sulfur_spike_replaceable_blocks`
- `minecraft:damage_type`
    - `sulfur_cube_with_block_immune_to`
- `minecraft:item`
    - `glazed_terracotta`
    - `concrete`
    - `concrete_powders`
    - `sulfur_cube_archetype/bouncy`
    - `sulfur_cube_archetype/regular`
    - `sulfur_cube_archetype/slow_flat`
    - `sulfur_cube_archetype/fast_flat`
    - `sulfur_cube_archetype/light`
    - `sulfur_cube_archetype/fast_sliding`
    - `sulfur_cube_archetype/slow_sliding`
    - `sulfur_cube_archetype/sticky`
    - `sulfur_cube_archetype/high_resistance`
    - `sulfur_cube_swallowable`
    - `sulfur_cube_food`

### List of Additions

- `net.minecraft.ExitCodes` - A list of exit codes Minecraft can throw when crashing.
- `net.minecraft.client`
    - `Minecraft`
        - `getProfileResult` - Gets the loaded profile of the user, or `null`.
        - `getMetricsRecorder` - Gets the recorder for the game metrics.
    - `OptionInstance`
        - `NO_ACTION` - A listener that ignores the changed value.
        - `$ValueUpdateListener` - A listener that handles what to do when an option value is changed.
- `net.minecraft.client.data.models.BlockModelGenerators#createBed` - Creates a bed with variants for the head and foot.
- `net.minecraft.client.data.models.model`
    - `ModelTemplates#BED_HEAD`, `BED_FOOT` - Model templates for the bed parts.
    - `TextureMapping#bed` - Creates a texture map for a bed-like model.
- `net.minecraft.client.multiplayer.ClientChunkCache#flipEmptySectionUpdates` - Updates the section arrays to their next index, clearing them for use. 
- `net.minecraft.core.BlockPos#neighborColumn` - An iterable of positions that starts at some given XYZ, goes until some other Y, and then loop through the same column space in the four horizontally orthogonal directions.
- `net.minecraft.core.dispenser.SulfurCubeBlockDispenseItemBehavior` - The dispense behavior allowing sulfur cubes to equip items.
- `net.minecraft.data.BlockFamilies`
    - `SULFUR`, `POLISHED_SULFUR`, `SULFUR_BRICKS` - Sulfur block variants.
    - `CINNABAR`, `POLISHED_CINNABAR`, `CINNABAR_BRICKS` - Cinnabar block variants.
- `net.minecraft.data.worldgen`
    - `BiomeDefaultFeatures`
        - `addSulfurCavesVegetationFeatures` - Vegetation features for the sulfur caves biome.
        - `addSulfurSpikeFeatures` - Sulfur spike features.
    - `TerrainProvider#peaksAndValleys` - Calculates the peaks and valleys scalar given the weirdness.
- `net.minecraft.data.worldgen.biome.OverworldBiomes#sulfurCaves` - The sulfur caves biome.
- `net.minecraft.gametest.framework.GameTestHelper#assertValueInBetween` - Assert the `Comparable` value is between some bounds.
- `net.minecraft.server.MinecraftServer#SERVER_THREAD_NAME` - The name of the server thread.
- `net.minecraft.util`
    - `CubicSpline`
        - `minValue`, `maxValue` - The bounds of the spline.
        - `sample`, `$Multipoint#sample` - Samples the spline at the given coordinates.
        - `asSampler` - Returns the sampler for the spine.
        - `$Multipoint#codec` - Gets the codec for the spline.
    - `Util`
        - `CONTROL_CHARACTER_ESCAPER` - An escaper for control characters.
        - `join` - Flattens lists of elements into a single list.
        - `dumpThreadInfo` - Dumps the info for all live platform threads.
- `net.minecraft.util.profiling.jfr.stats.ThreadAllocationStat#LOGGER` - The stat logger.
- `net.minecraft.util.profiling.metrics.MetricSampler#samplingPhase`, `$MetricSamplerBuilder#withSamplingPhase`, `$SamplingPhase` - The phase when the metric sampling should occur.
- `net.minecraft.util.profiling.metrics.profiling.MetricsRecorder#sampleDuringExtract` - Samples the metrics, marking the phase as during render state extraction.
- `net.minecraft.world.entity`
    - `AgeableMob#canBeABaby` - Whether the mob can have a baby variant.
    - `Entity`
        - `DEFAULT_NAMEPLATE_DISTANCE` - The default maximum distance an entity nameplate can be seen from.
        - `DEFAULT_BELOW_NAME_DISTANCE` - The default maximum distance that additional nameplate information on an entity can be seen from.
        - `getEntityBounciness` - How bouncy the entity is after a collision.
        - `getEffectiveGravity` - Gets the gravity currently being applied to the entity.
        - `omnidirectionalAirMover` - Whether the entity can omnidirectionally move in the air.
    - `LivingEntity`
        - `BASE_AIR_DRAG` - The base drag when within the air.
        - `BASE_VERTICAL_DRAG_FOR_NON_FLYERS` - The base air drag when a non-flyable entity is moving vertically.
        - `handleKillingBlow` - Handles when the killing blow has been made to this entity.
    - `MobCategory#getDebugAbbreviation` - An abbreviation of the category when looking through one of the debug menus.
- `net.minecraft.world.entity.ai.navigation.PathNavigation#getMaxVerticalDistanceToWaypoint` - Gets the maximum vertical distance from the current node the entity pathfinding through.
- `net.minecraft.world.entity.animal.turtle.Turtle#getHomePos` - Gets the home position of the turtle.
- `net.minecraft.world.entity.monster.cubemob`
    - `AbstractCubeMob` - A cube-like mob.
    - `SulfurCube` - A sulfur cube entity.
- `net.minecraft.world.entity.monster.zombie.Drowned#isSearchingForLand` - If the drowned is searching for land.
- `net.minecraft.world.item.DyeColor#getTerracottaColor` - The `MapColor` of a terracotta block dyed this color.
- `net.minecraft.world.level.Level#ACROSS_THE_WHOLE_WORLD` - The maximum diameter of a level.
- `net.minecraft.world.level.block`
    - `CopperChestBlock#getHingeSound` - Gets the hinge sound to play based on the current state of the chest.
    - `PotentSulfurBlock` - A sulfur block that can apply noxious gas effects.
    - `SpeleothemBlock` - An abstract representation of a speleothem.
    - `SulfurSpikeBlock` - A sulfur spike speleothem.
    - `WeatheringCopper$WeatherState#forEach` - Loops through the available weather states.
- `net.minecraft.world.level.block.entity`
    - `DecoratedPotPatterns#itemToPatternMappings` - Operates on the keys for an item and its associated pot pattern.
    - `PotentSulfurEntity` - A sulfur block entity that can apply noxious gas effects.
- `net.minecraft.world.level.block.state.BlockBehavior#bounceRestitution`, `$Properties#bounceRestitution` - How much the entity should bounce after colliding with this block.
- `net.minecraft.world.level.chunk`
    - `ChunkAccess#collectBiomesInPalette` - Collects all the biomes in the chunk sections.
    - `PalettedContainerRO#forEachInPalette` - Loop through all elements in the palette.
- `net.minecraft.world.level.levelgen.SurfaceRules`
    - `noiseGradient` - Creates a noise gradient rule from the given noise parameters and blockstate gradient.
    - `$Context#getBiome` - Gets the biome at the context position.
- `net.minecraft.world.level.levelgen.feature`
    - `SequenceFeature` - A feature made up of other placed features, applying them in the order they are given until either one of them fails or all succeeds.
    - `TemplateFeature` - A feature that attempts to place a structure template in the world.
- `net.minecraft.world.level.levelgen.feature.configurations.TemplateFeatureConfiguration` - A configuration for the `TemplateFeature`.
- `net.minecraft.world.level.storage.loot.LootPool#addAll` - Adds all pool entries.

### List of Changes

- `com.mojang.blaze3d.platform.ClientShutdownWatchdog#startShutdownWatchdog` now takes in the `String` callsite and `GameConfig` instead of the `File` game directory
- `com.mojang.math.MatrixUtil`
    - `eigenvalueJacobi` now takes in the result `Quaternionf` instead of returning it
    - `svdDecompose` now takes in the input, translation, and USV `Quanternionf`s and `Vector3f` instead of returning the USV
- `net.minecraft`
    - `CrashReportCategory$Entry` is now a record and `public` from `private`
    - `SystemReport#setDetail` now takes in a `CrashReportDetail` instead of a `String` supplier
- `net.minecraft.client`
    - `Minecraft`
        - `saveReport` now has an overload that takes in the crash report exit code `int`
        - `crash` now takes in the crash report exit code `int`
        - `saveReportAndShutdownSoundManager` is now `public` from `private`, taking in the crash report exit code `int`
        - `destroy` -> `exitWorldAndClose`, not one-to-one
        - `isMultiplayerServer` is now `public` from `private`
        - `getRunningThread` has been expanded to `public` from `protected`
        - `$GameLoadCookie` -> `GameLoadCookie`, now `public` from `private`
    - `OptionInstance` constructor, `#createBoolean`, `createButton` now take in a `$ValueUpdateListener` instead of a consumer
        - `$SliderableValueSet` is now `public` from package-private
        - `$ValueSet` is now `public` from package-private
            - `createButton` now takes in a `$ValueUpdateListener` instead of a consumer
- `net.minecraft.client.data.models.BlockModelGenerators`
    - `createColoredBlockWithRandomRotations`, `createColoredBlockWithStateRotations` now take in a list of `Block`s instead of a varargs
    - `createPointedDripstoneVariant` -> `createSpeleothemVariant`, now taking in a `Block`
    - `createBanner` no longer takes in the `Block`s for the standalone and wall variant
    - `createPointedDripstone` -> `createSpeleothem`, now taking in a `Block`
- `net.minecraft.client.data.models.model.ItemModelUtils#plainModel` now has an overload that takes in a `Transformation`
- `net.minecraft.client.multiplayer.ClientChunkCache#getLoadedEmptySections` split into `addedEmptySections`, `removedEmptySections`
- `net.minecraft.core.Holder` is now `sealed` to `$Direct` and `$Reference`
    - `$Reference` is now `non-sealed`
- `net.minecraft.data.worldgen`
    - `SurfaceRuleData` methods now take in the `HolderGetter<Biome>`
    - `TerrainProvider` methods no longer bind the `BoundedFloatFunction` generic
        - `buildErosionOffsetSpline` now takes in a `Float2FloatFunction` instead of a `BoundedFloatFunction`
- `net.minecraft.server.level`
    - `BlockDestructionProgress#updateTick`, `getUpdatedRenderTick` now deal with `long`s instead of `int`s
    - `ServerEntityGetter#getNearestEntity` now has an overload that takes in the list of `Entity`s to loop through along with the center position as three `double`s
- `net.minecraft.util`
    - `BoundedFloatFunction#createUnlimited` -> `constant`, not one-to-one
    - `CubicSpline` is now `sealed` to `$Multipoint` and `$Constant`
        - The generic `C` is removed, while the generic `I` is no longer bounded to a `BoundedFloatFunction`
        - `mapAll` -> `mapCoordinates`, not one-to-one
        - `builder(I, BoundedFloatFunction<Float>)`, `$Builder` now takes in a `Float2FloatFunction` instead of a `BoundedFloatFunction`
        - `$CoordinateVisitor` is removed
            - Use a standard `UnaryOperator` instead
    - `Mth`
        - `*_AXIS` are now `Vector3fc`s instead of `Vector3f`s
        - `rotationAroundAxis` now takes in a `Vector3fc` instead of a `Vector3f`
    - `NativeModuleLister`
        - `tryGetVersion` -> `tryGetModuleVersion`, now `public` from `private`
        - `$NativeModuleInfo`, `$NativeModuleVersion` are now `record`s
- `net.minecraft.util.filefix.access.ChunkNbt#updateChunk` now returns a `CompletableFuture`
- `net.minecraft.util.profiling.metrics.MetricSampler` now takes in a `$SamplingPhase`
    - `create(String, MetricCategory, T, ToDoubleFunction<T>)` -> `createExtractSampler(String, MetricCategory, DoubleSupplier)`
- `net.minecraft.world`
    - `InstantenousMobEffect` -> `InstantaneousMobEffect`
    - `MobEffect`
        - `applyInstantenousEffect` -> `applyInstantaneousEffect`
        - `isInstantenous` -> `isInstantaneous`
- `net.minecraft.world.entity`
    - `AgeableMob#isBaby`, `setBaby` are now `final`
        - Override `canBeABaby` instead
    - `LivingEntity`
        - `WATER_FLOAT_IMPULSE` -> `LIQUID_FLOAT_IMPULSE`, now `protected` from `private`
        - `travelInFluid` is now `protected` from `private`
        - `collectEquipmentChanges` is now `protected` from `private`, taking in the map of slots to last equipment stacks
    - `MobCategory` now takes in a `String` for the debug abbreviation
- `net.minecraft.world.entity.ai.control`
    - `FlyingMoveControl` now takes in a generic for the `Mob`
    - `MoveControl` now takes in a generic for the `Mob`
    - `SmoothSwimmingMoveControl` now takes in a generic for the `Mob`
- `net.minecraft.world.entity.animal`
    - `Bucketable` -> `.entity.Bucketable`
    - `FlyingAnimal` replaced by `Entity#omnidirectionalAirMover`
        - `isFlying` still exists on classes that used to implement the interface
- `net.minecraft.world.entity.animal.bee.Bee` no longer implements `FlyingAnimal`
    - `getGoalSelector` -> `Mob#getGoalSelector`
- `net.minecraft.world.entity.animal.fox.Fox#canMove` is now `public` from `private`
- `net.minecraft.world.entity.animal.happyghast.HappyGhast$HappyGhastLookControl#wrapDegrees90` -> `Mth#wrapDegrees90`
- `net.minecraft.world.entity.animal.parrot.Parrot` no longer implements `FlyingAnimal`
- `net.minecraft.world.entity.monster`
    - `Guardian#setMoving` is now `public` from `private`
    - `MagmaCube` -> `.monster.cubemob.MagmaCube`
        - The class now extends `AbstractCubeMob` and implements `Enemy`
    - `Slime` -> `.monster.cubemob.Slime`
        - The class now extends `AbstractCubeMob` and implements `Enemy`
- `net.minecraft.world.entity.monster.zombie.Drowned#wantsToSwim` is now `public` from `private`
- `net.minecraft.world.item`
    - `BucketItem`
        - `content` is now `protected` from `private`
        - `getFluidContext` - Gets the fluid clip context based on its contents.
    - `DyeColor` now takes in the `MapColor` for the terracotta variant.
- `net.minecraft.world.item.trading`
    - `TradeRebalanceVillagerTrades` now extends `VillagerTrades`
    - `VillagerTrade` one constructor now takes in the raw `HolderSet` of `Enchantment`s instead of being optionally-wrapped
- `net.minecraft.world.level.NaturalSpawner#getFilteredSpawningCategories` no longer takes in the `boolean`s for whether to spawn friendlies and enemies
- `net.minecraft.world.level.block`
    - `BedBlock` no longer implements `EntityBlock`
    - `Block#updateEntityMovementAfterFallOn` replaced by `getBounceRestitution`
    - `PointedDripstoneBlock` now extends `SpeleothemBlock`
        - The constructor now takes in the `BlockState` it can grow on
        - `TIP_DIRECTION` -> `SpeleothemBlock#TIP_DIRECTION`
        - `THICKNESS` -> `SpeleothemBlock#THICKNESS`
        - `WATERLOGGED` -> `SpeleothemBlock#WATERLOGGED`
        - `growStalactiteOrStalagmiteIfPossible` -> `SpeleothemBlock#growStalactiteOrStalagmiteIfPossible`, now an instance method
        - `canDrip` -> `isFreeHangingStalactite`, now `protected` from `public`
- `net.minecraft.world.level.block.state.BlockBehaviour`
    - `$BlockStateBase#emissiveRendering` no longer takes in the `BlockGetter` and `BlockPos`
    - `$Properties#emissiveRendering` now takes in a `BlockState` predicate instead of a `$StatePredicate`
- `net.minecraft.world.level.block.state.properties`
    - `BlockStateProperties#DRIPSTONE_THICKNESS` -> `SPELEOTHEM_THICKNESS`
    - `DripstoneThickness` -> `SpeleothemThickness`
- `net.minecraft.world.level.chucnk.LevelChunkSection`
    - `SECTION_WIDTH`, `SECTION_HEIGHT` replaced by `SectionPos#SECTION_SIZE`
    - `SECTION_SIZE` -> `SectionPos#SECTION_BLOCK_COUNT`
- `net.minecraft.world.level.dimension.DimensionType#infiniburn` now takes in a `HolderSet` of `Block`s instead of the referenced `TagKey`
- `net.minecraft.world.level.levelgen`
    - `DensityFunctions`
        - `$Spline` is now a static final `class` instead of a `record`
        - `$Coordinate` now holds the raw `DensityFunction` instead of being `Holder`-wrapped
    - `GeodeBlockSettings` is now a `record`
        - `cannotReplace`, `invalidBlocks` now take in a `HolderSet` of `Block`s instead of the referenced `TagKey`
    - `NoiseBasedChunkGenerator#buildSurface` now takes in a set of `Holder<Biome>`s instead of the `Registry<Biome>`
    - `SurfaceRules`
        - `isBiome(ResourceKey<Biome>...)` now takes in a `HolderGetter<Biome>`
        - `$ConditionSource#codec` is now a `MapCodec` instead of a `KeyDispatchDataCodec`
        - `$Context` now takes in now takes in a set of `Holder<Biome>`s instead of the `Registry<Biome>`
            - `updateY` no longer takes in the block XZ `int`s
    - `SurfaceSystem#buildSurface` now takes in a set of `Holder<Biome>`s instead of the `Registry<Biome>`
- `net.minecraft.world.level.levelgen.feature`
    - `DripstoneClusterFeature` -> `SpeleothemClusterFeature`
    - `DripstoneUtils` -> `SpeleothemUtils`
    - `Feature`
        - `DRIPSTONE_CLUSTER` -> `SPELEOTHEM_CLUSTER`
        - `POINTED_DRIPSTONE` -> `SPELEOTHEM`
    - `LakeFeature$Configuration` now takes in `BlockPredicate`s for where the feature can be placed, what blocks can be replaced with air or fluid, and what blocks can be replaced with the border/barrier blocks
    - `PointedDripstoneFeature` -> `SpeleothemFeature`
- `net.minecraft.world.level.levelgen.feature.configurations`
    - `DripstoneClusterConfiguration` -> `SpeleothemClusterConfiguration`
    - `LargeDripstoneFeature` now takes in a `HolderSet` of `Block`s that can be replaced
    - `PointedDripstoneConfiguration` -> `SpeleothemConfiguration`
    - `RootSystemConfiguration` is now a `record`
        - `rootReplaceable` now takes in a `HolderSet` of `Block`s instead of the referenced `TagKey`
    - `SimpleRandomFeatureConfiguration` -> `CompositeFeatureConfiguration`
    - `VegetationPatchConfiguration` is now a `record`
        - `replaceable` now takes in a `HolderSet` of `Block`s instead of the referenced `TagKey`
- `net.minecraft.world.level.levelgen.structure.structures.RuinedPortalPiece` now takes in the `HolderLookup$Provider` of registries
    - The other constructor now takes in a `StructurePieceSerializationContext` instead of the `StructureTemplateManager`
    - `$Properties` is now a `record`
- `net.minecraft.world.level.lighting.LightEngine#getLightBlockInto` -> `getLightDampeningInto`
- `net.minecraft.world.level.storage.loot.functions`
    - `EnchantRandomlyFunction$Builder#withOptions` now takes in the raw `HolderSet` of `Enchantment`s instead of being optionally-wrapped
    - `SetRandomPotionFunction#fromTagKey` now takes in the raw `HolderSet` of `Potion`s instead of being optionally-wrapped

### List of Removals

- `net.minecraft.client.data.models.model.ModelTemplates#BED_INVENTORY`
- `net.minecraft.client.geom.ModelLayers#BED_FOOT`, `BED_HEAD`
- `net.minecraft.util.Tuple`
- `net.minecraft.util.thread.BlockableEventLoop#hasDelayedCrash`
- `net.minecraft.world.entity.Mob#setBodyArmorItem`
- `net.minecraft.world.level.block.entity`
    - `BedBlockEntity`
    - `DecoratedPotPatterns#getPatternFromItem`
- `net.minecraft.world.level.levelgen`
    - `DensityFunction#DIRECT_CODEC`, `HOLDER_HELPER_CODEC`
    - `SurfaceRules#isBiome(List<ResourceKey<Biome>>)`
- `net.minecraft.world.waypoints.Waypoint#MAX_RANGE`
