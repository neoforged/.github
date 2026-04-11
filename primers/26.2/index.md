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
    - `BlendEquation` - An equation that blends the source and destination factors using the provided operation.
    - `BlendFunction` now takes in `BlendEquation`s, `BlendFactor`s, or `BlendOp`s instead of `SourceFactor`s and `DestFactor`s
    - `RenderPipeline`
        - `$Builder#withUniform` now takes in the `GpuFormat` instead of the `TextureFormat`
        - `$UniformDescription` now takes in the `GpuFormat` instead of the `TextureFormat`
    - `RenderTarget#blitToScreen` is removed
        - Usage replaced by `GpuSurface#blitFromTexture`
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
    - `CommandEncoderBackend`
        - `isInRenderPass` -> `CommandEncoder#isInRenderPass`, now `protected` from `public`
        - `submitRenderPass` - Submits the current render pass, handled as part of the try-with-resources.
        - `presentTexture` -> `GpuSurface#blitFromTexture`, not one-to-one
        - `timerQueryBegin`, `timerQueryEnd` replaced by `writeTimestamp`
    - `DeviceInfo` - A record containing information about the GPU device.
    - `DeviceLimits` - A record containing information about the limits of GPU features.
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
    - `RenderPass` now takes in a `Runnable` for what to do on close, typically from a try-with-resources
        - `writeTimestamp` - Writes the timestamp to the pool at the given index.
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
    - `StagingBuffer` - A buffer for staging changes to write at a later point in time.
    - `Tesselator` class is removed
    - `UberGpuBuffer` now takes in the `StagingBuffer` instead of the `GpuDevice`, buffer size `int`, and `GraphicsWorkarounds`
        - `uploadStagedAllocations` now takes in the `StagingBuffer$Uploader` instead of the `CommandEncoder`
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
    - `Options`
        - `hideGui` is replaced by `Hud#isHidden`, `GuiRenderState#isHudHidden`
        - `preferredGraphicsBackend` - Gets the preferred graphics library to use when rendering the game.
        - `hasPreferredGraphicsBackendChanged` - Whether the preferred graphics library selected has changed from when the game initially started up.
    - `PreferredGraphicsApi` - The graphics libraries vanilla supports to render the game.
- `net.minecraft.client.gui`
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
- `net.minecraft.client.particle`
    - `BreakingItemParticle$SulfurCubeProvider` - A breaking item particle provider for the sulfur cube.
    - `NoxiousGasCloudParticle` - A particle for the noxious gas cloud.
    - `NoxiousGasParticle` - A particle for the noxious gas.
    - `SulfurBubbleParticle` - A particle for the sulfur bubbles.
- `net.minecraft.client.renderer`
    - `DebugCrosshairRenderer` - A renderer for the debug 3D crosshair reticle.
    - `DynamicUniforms#writeTransform` now has a view overloads for taking in the various transform parameters, or the `$Transform` object itself
    - `OrderedSubmitNodeCollector`
        - `submitModelPart` no longer takes in the `boolean`s for whether its sheeted or has foil
        - `submitShapeOutline` - Submits a voxel shape to draw an outline of.
    - `RenderPipelines`
        - Pipelines using the `DepthStencilState` have their values inverted
            - `CompareOp` depth test uses `GREATER_THAN_OR_EQUAL` instead of `LESS_THAN_OR_EQUAL`
            - Values for the two `int`s are now their additive inverses
        - `TEXT_INTENSITY` -> `TEXT_GRAYSCALE`
        - `GUI_TEXT_INTENSITY` -> `GUI_TEXT_GRAYSCALE`
        - `TEXT_INTENSITY_SEE_THROUGH` -> `TEXT_GRAYSCALE_SEE_THROUGH`
        - `DRAGON_RAYS_DEPTH` is removed
            - Merged with `DRAGON_RAYS`
    - `ScreenEffectRenderer` no longer takes in the `MultiBufferSource$BufferSource`
        - `renderScreenEffect` -> `submit`
    - `ShapeRenderer` -> `ShapeOutlineFeatureRenderer`, not one-to-one
    - `Sheets#cutoutBlockSheet`, `translucentBlockSheet` are removed
        - Replaced by direct construction calls or `*ItemSheet` variants
    - `SubmitNodeCollection`
        - `getModelPartSubmits` is removed
        - `getShapeOutlineSubmits` - The submitted shape outlines.
    - `SubmitNodeStorage`
        - `$ModelPartSubmit` record is removed
        - `$ShapeOutlineSubmit` - A submitted shape outline.
    - `WorldBorderRenderer` now implements `AutoCloseable`
- `net.minecraft.client.renderer.entity`
    - `AbstractCubeMobRenderer` - An entity renderer for cube-like mobs.
    - `MagmaCubeRenderer` now extends `AbstractCubeMobRenderer`
    - `SlimeRenderer` now extends `AbstractCubeMobRenderer`
    - `SulfurCubeRenderer` - The entity renderer for a sulfur cube.
- `net.minecraft.client.renderer.entity.layers.SulfurCubeInnerLayer` - The inner layer of a sulfur cube.
- `net.minecraft.client.renderer.entity.state.SulfurCubeRenderState` - The render state for a sulfur cube.
- `net.minecraft.client.renderer.feature`
    - `FeatureRenderDispatcher#renderTranslucentParticles` -> `renderTranslucentAfterTerrain`
    - `ItemFeatureRenderer`
        - `getFoilBuffer(MultiBufferSource, RenderType, boolean, boolean)` is removed
            - Use the other `getFoilBuffer` instead
        - `getFoilRenderType` is removed
            - Directly merged into the other `getFoilBuffer`
    - `ModelPartFeatureRenderer` class is removed
    - `ShapeOutlineFeatureRenderer` - A feature renderer for `VoxelShape` outlines.
- `net.minecraft.client.renderer.rendertype`
    - `RenderTypes`
        - `textIntensity` -> `textGrayscale`
        - `textIntensityPolygonOffset` -> `textGrayscalePolygonOffset`
        - `textIntensitySeeThrough` -> `textGrayscaleSeeThrough`
        - `dragonRaysDepth` is removed
            - Merged with `dragonRays`
    - `TextureTransform#getMatrix` -> `createMatrix`
- `net.minecraft.client.renderer.state.OptionsRenderState#hideGui` -> `GuiRenderState#isHudHidden`
- `net.minecraft.client.renderer.state.gui.BlitRenderState` now takes in a `Matrix3x2fc` instead of a `Matrix3x2f` for the pose
- `net.minecraft.client.renderer.state.gui.pip`
    - `GuiEntityRenderState` now takes in `Vector3fc` and `Quaternionfc`s instead of `Vector3f` and `Quaternionf`s
    - `GuiSkinRenderState` now takes in a `Model$Simple` instead of a `PlayerModel` for the model
    - `PictureInPictureRenderState#IDENTITY_POSE`, `pose` now returns a `Matrix3x2fc` instead of a `Matrix3x2f`
- `net.minecraft.client.renderer.state.level.LevelRenderState#render3dCrosshair` - Whether to render the 3d crosshair reticle.
- `net.minecraft.client.resources.model.ModelManager$MaterialBakerImpl` - An implemented `MaterialBaker`.
- `net.minecraft.client.resources.model.sprite.SpriteId#buffer` are removed

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

// Do something given two collections, zipping them per entry
ColorCollection.zipApply((exampleBlock, dye) -> {
    // ...
}, EXAMPLE_BLOCK, Items.DYE);

// Get a specific object based on the collection variant
Block pink = EXAMPLE_BLOCK.pick(DyeColor.PINK);
```

- `net.minecraft.client.data.models.BlockModelGenerators`
    - `createBanner` no longer takes in the `Block`s for the ground and wall variants
    - `createBanners` is removed
    - `createBed` no longer takes in the `Block`s for the bed and particle
    - `createBeds` is removed
- `net.minecraft.client.renderer.block.BuiltInBlockModels#createBanners` no longer takes in the `Block`s for the ground and wall variants
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

### Tag Changes

- `minecraft:block`
    - `glazed_terracotta`
    - `concrete`
    - `shears_extreme_breaking_speed`
    - `shears_major_breaking_speed`
    - `shears_minor_breaking_speed`
    - `suppresses_bounce`
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
- `net.minecraft.client.Minecraft#getProfileResult` - Gets the loaded profile of the user, or `null`.
- `net.minecraft.core.BlockPos#neighborColumn` - An iterable of positions that starts at some given XYZ, goes until some other Y, and then loop through the same column space in the four horizontally orthogonal directions.
- `net.minecraft.core.dispenser.SulfurCubeBlockDispenseItemBehavior` - The dispense behavior allowing sulfur cubes to equip items.
- `net.minecraft.data.BlockFamilies`
    - `SULFUR`, `POLISHED_SULFUR`, `SULFUR_BRICKS` - Sulfur block variants.
    - `CINNABAR`, `POLISHED_CINNABAR`, `CINNABAR_BRICKS` - Cinnabar block variants.
- `net.minecraft.data.worldgen`
    - `BiomeDefaultFeatures#addSulfurCavesVegetationFeatures` - Vegetation features for the sulfur caves biome.
    - `TerrainProvider#peaksAndValleys` - Calculates the peaks and valleys scalar given the weirdness.
- `net.minecraft.data.worldgen.biome.OverworldBiomes#sulfurCaves` - The sulfur caves biome.
- `net.minecraft.gametest.framework.GameTestHelper#assertValueInBetween` - Assert the `Comparable` value is between some bounds.
- `net.minecraft.util`
    - `CubicSpline`
        - `minValue`, `maxValue` - The bounds of the spline.
        - `sample`, `$Multipoint#sample` - Samples the spline at the given coordinates.
        - `asSampler` - Returns the sampler for the spine.
        - `$Multipoint#codec` - Gets the codec for the spline.
    - `Util#join` - Flattens a varargs of lists of elements into a single list.
- `net.minecraft.world.entity`
    - `AgeableMob#canBeABaby` - Whether the mob can have a baby variant.
    - `Entity`
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
- `net.minecraft.world.level.block`
    - `CopperChestBlock#getHingeSound` - Gets the hinge sound to play based on the current state of the chest.
    - `PotentSulfurBlock` - A sulfur block that can apply noxious gas effects.
    - `WeatheringCopper$WeatherState#forEach` - Loops through the available weather states.
- `net.minecraft.world.level.block.entity.PotentSulfurEntity` - A sulfur block entity that can apply noxious gas effects.
- `net.minecraft.world.level.block.state.BlockBehavior#bounceRestitution`, `$Properties#bounceRestitution` - How much the entity should bounce after colliding with this block.
- `net.minecraft.world.level.levelgen.SurfaceRules#noiseGradient` - Creates a noise gradient rule from the given noise parameters and blockstate gradient.
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
- `net.minecraft.client.Minecraft`
    - `saveReport` now has an overload that takes in the crash report exit code `int`
    - `crash` now takes in the crash report exit code `int`
    - `saveReportAndShutdownSoundManager` is now `public` from `private`, taking in the crash report exit code `int`
    - `destroy` -> `exitWorldAndClose`, not one-to-one
    - `isMultiplayerServer` is now `public` from `private`
    - `getRunningThread` has been expanded to `public` from `protected`
    - `$GameLoadCookie` -> `GameLoadCookie`, now `public` from `private`
- `net.minecraft.client.data.models.BlockModelGenerators`
    - `createColoredBlockWithRandomRotations`, `createColoredBlockWithStateRotations` now take in a list of `Block`s instead of a varargs
    - `createPointedDripstoneVariant` now takes in a `SpeleothemThickness` instead of a `DripstoneThickness`
    - `createBanner` no longer takes in the `Block`s for the standalone and wall variant
- `net.minecraft.core.Holder` is now `sealed` to `$Direct` and `$Reference`
    - `$Reference` is now `non-sealed`
- `net.minecraft.data.worldgen.TerrainProvider` methods no longer bind the `BoundedFloatFunction` generic
    - `buildErosionOffsetSpline` now takes in a `Float2FloatFunction` instead of a `BoundedFloatFunction`
- `net.minecraft.server.level.ServerEntityGetter#getNearestEntity` now has an overload that takes in the list of `Entity`s to loop through along with the center position as three `double`s
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
    - `BucketItem#content` is now `protected` from `private`
    - `DyeColor` now takes in the `MapColor` for the terracotta variant.
- `net.minecraft.world.item.trading`
    - `TradeRebalanceVillagerTrades` now extends `VillagerTrades`
    - `VillagerTrade` one constructor now takes in the raw `HolderSet` of `Enchantment`s instead of being optionally-wrapped
- `net.minecraft.world.level.block.Block#updateEntityMovementAfterFallOn` replaced by `getBounceRestitution`
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
- `net.minecraft.world.level.levelgen.feature`
    - `DripstoneClusterFeature` -> `SpeleothemClusterFeature`
    - `DripstoneUtils` -> `SpeleothemUtils`
    - `Feature`
        - `DRIPSTONE_CLUSTER` -> `SPELEOTHEM_CLUSTER`
        - `POINTED_DRIPSTONE` -> `SPELEOTHEM`
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

- `net.minecraft.util.Tuple`
- `net.minecraft.util.thread.BlockableEventLoop#hasDelayedCrash`
- `net.minecraft.world.entity.Mob#setBodyArmorItem`
- `net.minecraft.world.level.levelgen.DensityFunction#DIRECT_CODEC`, `HOLDER_HELPER_CODEC`
