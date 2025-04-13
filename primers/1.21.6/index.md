# Minecraft 1.21.5 -> 1.21.6 Mod Migration Primer

This is a high level, non-exhaustive overview on how to migrate your mod from 1.21.5 to 1.21.6. This does not look at any specific mod loader, just the changes to the vanilla classes. All provided names use the official mojang mappings.

This primer is licensed under the [Creative Commons Attribution 4.0 International](http://creativecommons.org/licenses/by/4.0/), so feel free to use it as a reference and leave a link so that other readers can consume the primer.

If there's any incorrect or missing information, please file an issue on this repository or ping @ChampionAsh5357 in the Neoforged Discord server.

## Pack Changes

There are a number of user-facing changes that are part of vanilla which are not discussed below that may be relevant to modders. You can find a list of them on [Misode's version changelog](https://misode.github.io/versions/?id=1.21.6&tab=changelog).

## GUI Changes

The GUI rendering system has significantly changed from the previous version. While it may seem somewhat similar on the surface, the actual implementation details are much more complex and roundabout. This will explain a high-level overview of the new system with some basic examples.

### Prepare and Render

Rendering a GUI has been broken into two phases: 'prepare' and 'render'.

The prepare phase is what we typically consider the GUI render methods (e.g., `Gui#render`, `AbstractContainerScreen#renderBg`, etc.). These methods, instead of actually rendering the components, now submit them to the `GuiRenderState` to be stored for use during the 'render' phase. The `GuiRenderState` is stored on the `GameRenderer` and passed to the `GuiGraphics` which uses the previous methods to add the desired objects to the render state.

The render phase is what actually handles the rendering of all the objects. This is handled through the `GuiRenderer`, which reads the data from the `GuiRenderState` and, after a bit of delegated preparation and sorting, renders the objects to the screen. The `GuiRenderer` is also stored on the `GameRenderer`.

### Element Ordering

Now, how are elements rendered onto screen? In the previous versions, this happened mainly based on the order of the render calls. However, the new system sorts elements using a comparator based on four things (in order of priority): the `GuiLayer`, the `ScreenRectangle` scissor, the `RenderPipeline` order, and the element's `z` or depth value. These are all stored on the `GuiElementRenderState` passed to the `GuiRenderState` via `GuiRenderState#submitGuiElement`.

As a warning, due to the element sort order, certain custom configurations may result in incorrect screen renders (e.g., a transparent element rendering before the background element). As such, it is imperative that you understand how the comparator and their assigned values work when sorting elements.

#### Gui Layers

`GuiLayer` is simply an enum which uses its ordinal to determine the order of which elements should be rendered. This can be stored via `GuiElementRenderState#layer` For example, elements on the `SCREEN_BACKGROUND` layer will be drawn first, followed by elements on the `SCREEN` layer. As this is an enum that relies on ordering, if you want to add new elements, you must make sure your mod loader supports adding arbitrary enum objects or has a separate method of order handling.

#### Screen Rectangle Scissor

The `ScreenRectangle` is simply the area that the element is allowed to draw to, stored via `GuiElementRenderState#scissorArea`. Elements with no specified `ScreenRectangle` will be ordered first, followed by the minimum Y, the maximum Y, the minimum X, and the maximum X.

#### Render Pipeline

`RenderPipeline`s define the pipeline used to render some object to the screen, including its shaders, format, uniforms, etc. This is stored via `GuiElementRenderState#pipeline`. Pipelines are sorted in the order they are built. This means that `RenderPipelines#ENTITY_TRANSLUCENT` will be rendered before `RenderPipelines#GUI` if on the same layer and scissor rectangle. As this is a system that relies on classloading constants, if you want to add new elements, make sure your mod loader supports some kind of dynamic pipeline ordering.

#### Depth

The element's depth, or `GuiElementRenderState#z`, is the final comparator that determines what element gets rendered first. Assuming that all of your submit calls have been through calling one of the `GuiGraphics` methods, the `z` value is based upon the order of the method calls. For every submit action, `GuiGraphics` calls `getNextDepth`, which increases the Z value by `0.01`.

### GuiElementRenderState

Now that we understand ordering, what exactly is the `GuiElementRenderState` that we've been using? Well essentially, every object rendered to the screen is represented by a `GuiElementRenderState`, from the player seen in the inventory menu to each individual item. A `GuiElementRenderState` defines six methods. First, there are the common ones used for ordering (`z`, `layer`) and how to render to the screen (`pipeline`, `scissorArea`). Then there is the `TextureSetup` (via `textureSetup`) which defines `Sampler0`, `Sampler1`, and `Sampler2` that is used in the shaders when present. Finally, `buildVertices` is responsible for filling the buffer with its required vertices. For GUIs, this typically calls `VertexConsumer#addVertexWith2DPose`.

There are three types of `GuiElementRenderState`s provided by vanilla: `BlitRenderState`, `ColoredRectangleRenderState`, and `GlyphRenderState`. `ColoredRectangleRenderState` and `GlyphRenderState` are simple cases for handling a basic color rectangle and text character, respectively. `BlitRenderState` covers every other case as almost every method eventually writes to a `GpuTexture` which is then consumed by this.

`GuiElementRenderState`s are added to the `GuiRenderState` via `GuiRenderState#submitGuiElement`. This is called by `GuiGraphics#*line`, `fill`, and `blit*` methods.

#### GuiItemRenderState

`GuiItemRenderState` is a special case used to render an item to the screen. It takes in the stringified name of the item, the current pose, its XYZ coordinates, a layer, and its scissor area. The `ItemStackRenderState` it holds is what defines how the item is rendered.

Just before the 'render' phase, the `GuiRenderer` effectively turns the `GuiItemRenderState` into a `GuiElementRenderState`, more specifically a `BlitRenderState`. This is done by constructing an item atlas `GpuTexture` which the item is drawn to, and then that texture is submitted as a `BlitRenderState`. All `GuiitemRenderState`s use `RenderPipelines#GUI_TEXTURED_PREMULTIPLIED_ALPHA`.

`GuiElementRenderState`s are added to the `GuiRenderState` via `GuiRenderState#submitItem`. This is called by `GuiGraphics#render*Item*` methods.

#### GuiTextRenderState

`GuiTextRenderState` is a special case used to render text to the screen. It takes in the `TextRenderState`; the current pose; its XY coordinates; Z values for the background, text and shadow, and effect and shadow, a layer, and its scissor area. The `TextRenderState` simply defines whether to draw the drop shadow, the default main background colors, how the text is displayed, light coordinates, the text and its effects as a list of glyphs, the line height, the empty glyph, and the max X.

Just before the 'render' phase, the `GuiRenderer` turns the `GuiTextRenderState` into a `GuiElementRenderState`, more specifically a `GlyphRenderState`. This performs a similar process as the item render state where the text is written to a `GpuTexture` to be consumed.

`GuiElementRenderState`s are added to the `GuiRenderState` via `GuiRenderState#submitText`. This is called by `GuiGraphics#draw*String` and `render*Tooltip` methods.

#### Picture-in-Picture

Picture-in-Picture is a special case used to render arbitrary objects to a `GpuTexture` to be passed into a `BlitRenderState`. A Picture-in-Picture is made up of two components the `PictureInPictureRenderState`, and the `PictureInPictureRenderer`.

`PictureInPictureRenderState` is an interface which can store some data used to render the object to the texture. By default, it must supply the minimum and maximum XY coordinates, the Z coordinate, the texture scale, the scissor area, and the layer to render on. Any other data can be added by the implementor.

```java
public record ExamplePIPRenderState(boolean data, int x0, int x1, int y0, int y1, float z, float scale, @Nullable ScreenRectangle scissorArea, GuiLayer layer)
    implements PictureInPictureRenderState {}
```

`PictureInPictureRenderer` is the renderer which writes the data to the `GpuTexture`. It does this by taking advantage of the fact that the `RenderSystem#output*Override` textures are written to in a `BufferSource` if they are not null. A `PictureInPictureRenderer` method must implement three methods. `getRenderStateClass` returns the class of the `PictureInPictureRenderState` implementation. `getTextureLabel` returns the texture label for debugging purposes. `renderToTexture` is where the render logic is called to write the data to the texture. 

```java
public class ExamplePIPRenderer extends PictureInPictureRenderer<ExamplePIPRenderState> {

    // Takes in the buffer source from `RenderBuffers`
    public ExamplePIPRenderer(MultiBufferSource.BufferSource bufferSource) {
        super(bufferSource);
    }

    @Override
    public Class<ExamplePIPRenderState> getRenderStateClass() {
        // The class of the render state
        return ExamplePIPRenderState.class;
    }

    @Override
    protected void renderToTexture(ExamplePIPRenderState renderState, PoseStack pose) {
        // Render whatever you want here
        // You can make use of the buffer source via `this.bufferSource`
    }

    @Override
    protected String getTextureLabel() {
        // This can be anything, but it should be unique
        return "examplemod: example pip";
    }
}
```

To be able to use the renderer, is must be added to `GuiRenderer#pictureInPictureRenderers`. As the constructor takes in an immutable list while the renderer stores an immutable map, use whatever method is provided by your mod loader.

```java
// We will assume:
// - `GuiRenderer#bufferSource` is accessible
// - The map is made mutable
// For some GuiRenderer renderer

var examplePIP = new ExamplePIPRenderer(renderer.bufferSource);

renderer.pictureInPictureRenderers.put(
    examplePIP.getRenderStateClass(),
    examplePIP
);
```

`PictureInPictureRenderState`s are added to the `GuiRenderState` via `GuiRenderState#submitPicturesInPictureState`. This is called by `GuiGraphics#submit*RenderState` methods.

### Contextual Bars

Many HUD elements on the screen take over where the experience bar is rendered. However, how these are rendered and which one is prioritized is handled through the contextual bar system. A contextual bar is made up of two parts, the `ContextualBarRenderer`, responsible for rendering the bar to the screen, and the `Gui$ContextualInfo`, which is used as the bar's identifier.

A `ContextualBarRenderer` requires two methods to be implemented: `renderBackground`, which renders the bar itself, and `render`, which renders elements that appear on top of the bar.

```java
public class ExampleContextualBarRenderer implements ContextualBarRenderer {

    @Override
    public void renderBackground(GuiGraphics graphics, DeltaTracker delta) {
        // Render background bar sprites here
    }

    @Override
    public void render(GuiGraphics graphics, DeltaTracker delta) {
        // Render anything that might sit on top of the bar
    }
}
```

`Gui$ContextInfo` is an enum, so your mod loader will need to either allow extensions or some other method of identifying a bar renderer. Then, the renderer needs to be added to map of renderers stored in `Gui#contextualInfoBarRenderers`. As this is an immutable map, use whatever method is provided by your mod loader.

```java
// We will assume:
// - Created a new enum value `EXAMPLEMOD_EXAMPLE`
// - The map is made mutable
// For some Gui gui
gui.contextualInfoBarRenderers.put(
    EXAMPLEMOD_EXAMPLE,
    ExampleContextualBarRenderer::new
);
```

Finally, to get your bar to be called and prioritized, you need to modify `Gui#nextContextualInfoState` to return your enum value, or whatever method is provided by your mod loader.

- `com.mojang.blaze3d.vertex.VertexConsumer#addVertexWith2DPose` - Adds a vertex to be rendered in 2d space.
- `net.minecraft.client.gui`
    - `Font`
        - `drawInBatch` no longer returns anything
        - `extractTextRenderState` - Creates the render state used to render text to the screen.
        - `$StringRenderOutput` no longer takes in the `MultiBufferSource` and the `Matrix4f` representing the pose
    - `Gui`
        - `shouldRenderDebugCrosshair` - Returns true if the debug crosshair should be rendered.
        - `$RenderFunction` - An interface that defines how a certain section or element is rendered on the gui.
    - `GuiGraphics` now takes in a `GuiRenderState` instead of the `MultiBufferSource$BufferSource`
        - `MAX_GUI_Z`, `MIN_GUI_Z` are removed
        - `pose` now returns a `Matrix3x2fStack`
        - `flush` is removed
        - `pushGuiLayer`, `popPushGuiLayer`, `popGuiLayer` - Handles what layer to render to when submitting a draw call. These are used to sort the elements initially before using the depth values from the generic render call order.
        - All methods no longer take in a `RenderType`, `VertexConsumer`, or `Function<ResourceLocation, RenderType>`, instead specifying a `RenderPipeline` and a `TextureSetup` depending on the call
        - `drawString`, `drawStringWithBackdrop` no longer returns anything
        - `renderItem(ItemStack, int, int, int, int)` is removed
        - `renderItemDecorations` now has an overload to specify an `$ItemSlotContext`
        - `drawSpecial` is removed, relaced by individual `submit*RenderState` depending on the special case
        - `$ItemSlotContext` - Specifies an indirect link to the layer to render item decorations to.
        - `$ScissorStack#peek` - Gets the last rectangle on the stack.
    - `LayeredDraw` class is removed
- `net.minecraft.client.gui.contextualbar`
    - `ContextualBarRenderer` - An interface which defines an object with some background to render.
    - `ExperienceBarRenderer` - Draws the experience bar.
    - `JumpableVehicleBarRenderer` - Draws the jump power bar (e.g. horse jumping).
    - `LocatorBarRenderer` - Draws the bar that points to given waypoints.
- `net.minecraft.client.gui.font.glyphs.BakedGlyph` now takes in a `GpuTexture`
    - `extractChar` - Extracts a glyph instance submits the element to be rendered.
    - `extractEffect` - Extracts a glyph effect (e.g. drop shadow) submits the element to be rendered.
    - `extractBackground` - Extracts a glyph effect and submits the element to be rendered on the background Z.
- `net.minecraft.client.gui.navigation.ScreenRectangle#transformAxisAligned` now takes in a `Matrix3x2f` instead of a `Matrix4f`
- `net.minecraft.client.gui.render`
    - `GuiLayer` - An enum that defines the order of how elements render on the screen.
    - `GuiRenderer` - A class that renders all submitted elements to the screen.
    - `TextureSetup` - A record that specifies samplers 0-2 for use in a render pipeline. The first two textures are arbitrary with the third being for the current lightmap texture.
- `net.minecraft.client.gui.render.pip`
    - `GuiBannerResultRenderer` - A renderer for the banner result preview.
    - `GuiBookModelRenderer` - A renderer for the book model in the enchantment screen.
    - `GuiEntityRenderer` - A renderer for a given entity.
    - `GuiProfilerChartRenderer` - A renderer for the profiler chart.
    - `GuiSignRenderer` - A renderer for the sign background in the edit screen.
    - `GuiSkinRenderer` - A renderer for a player with a given skin.
    - `PictureInPictureRenderer` - An abstract class meant for rendering dynamic elements that are not standard 2d elements, items, or text.
- `net.minecraft.client.gui.render.state`
    - `BlitRenderState` - An element state for a basic 2d texture blit.
    - `ColoredRectangleRenderState` - An element state for a simple rectangle with a tint.
    - `GlyphRenderState` - An element state for a glyph (font text).
    - `GuiElementRenderState` - An interface representing the state of a element to render.
    - `GuiItemRenderState` - A record representing the state of an item to render.
    - `GuiRenderState` - The state of the GUI to render to the screen.
    - `GuiTextRenderState` - A record representing the state of the text and its location to render.
    - `TextRenderState` - A record representing the state of the text to render.
- `net.minecraft.client.gui.render.state.pip`
    - `GuiBannerResultRenderState` - The state of the banner result preview.
    - `GuiBookModelRenderState` - The state of the book model in the enchantment screen.
    - `GuiEntityRenderState` - The state of a given entity.
    - `GuiProfilerChartRenderState` - The state of the profiler chart.
    - `GuiSignRenderState` - The state of the sign background in the edit screen.
    - `GuiSkinRenderState` - The state of a player with a given skin.
    - `PictureInPictureRenderState` - An interfaces that defines the basic state necessary to render the picture-in-picture to the screen.
- `net.minecraft.client.gui.screens.inventory`
    - `AbstractContainerScreen#SLOT_ITEM_BLIT_OFFSET` is removed
    - `AbstractSignEditScreen#offsetSign` -> `getSignYOffset`, not one-to-one
    - `InventoryScreen#renderEntityInInventory` no longer takes in the XY offset, instead taking in 4 `int`s to represent the region to render to
- `net.minecraft.client.gui.screens.inventory.tooltip`
    - `ClientTooltipComponent#renderText` no longer takes in the pose and buffer, instead the `GuiGraphics` to submit the text for rendering
    - `TooltipRenderUtil#renderTooltipBackground` no longer takes in a Z offset
- `net.minecraft.client.player.LocalPlayer#experienceDisplayStartTick` - Represents the start tick of when the experience display should be prioritized.
- `net.minecraft.client.renderer`
    - `GameRenderer` no longer takes in a `ResourceManager`
        - `ITEM_ACTIVATION_ANIMATION_LENGTH` is removed
        - `setRenderHand` is removed
        - `renderZoomed` is removed
    - `MapRenderer#WIDTH`, `HEIGHT` are now public
    - `RenderPipelines`
        - `GUI_OVERLAY`, `GUI_GHOST_RECIPE_OVERLAY`, `GUI_TEXTURED_OVERLAY` are removed
        - `GUI_TEXTURED_PREMULTIPLIED_ALPHA` - A pipeline that assumes the textures have already had their transparency premultiplied during the composite stage.
    - `RenderStateShard`
        - `$Builder#add` no longer takes in whether the shard should be blurred
        - `$TextureStateShard` no longer takes in a `TrisState` to set the blur mode
    - `RenderType`
        - `debugLine` is removed
        - `gui`, `guiOverlay`, `guiTexturedOverlay`, `guiOpaqueTexturedBackground`, `guiNauseaOverlay`, `guiTextHighlight`, `guiGhostRecipeOverlay`, `guiTextured` are removed
        - `vignette` is removed
        - `crosshair` is removed
        - `mojangLogo` is removed
    - `ScreenEffectRenderer` is now an instance implementation instead of simply a static method
        - The constructor takes in the current `Minecraft` instance and a `MultiBufferSource`
        - `renderScreenEffect` is now an instance method, taking in whether the entity is sleeping and the partial tick
        - `resetItemActivation`, `displayItemActivation` - Handles when an item is automatically activated (e.g., totem).
- `net.minecraft.client.renderer.item.ItemStackRenderState`
    - `setAnimated`, `isAnimated` - Returns whether the item is animated (e.g., foil).
    - `appendModelIdentityElement`, `getModelIdentity` - Returns the identity component being rendered.
- `net.minecraft.client.renderer.texture.TextureAtlasSprite#isAnimated` - Returns whether the sprite has any animation.

## Waypoints

Waypoints are simply a method to track the position of some object in the game. The underlying system is handled by the `WaypointManager`, responsible for holding the waypoints it is tracking while also allowing updates and removals as necessary. The server handles waypoints through the `ServerWaypointManager` (obtained via `ServerLevel#getWaypointManager`), which holds an active connect to the transmitter to receive live updates. The client receives these waypoints via the `ClientWaypointManager` (obtained via `ClientPacketListener#getWaypointManager`), which simply holds some identifier along with an exact position, a chunk if the position is not within the view distance, or an angle if the distance is further than the stored `Waypoint$Fade#farDist`, which is over 332 blocks away by default.

Entities track waypoints by default.

### Icons

Every waypoint holds an icon of some kind, which handles both the distance fade and the color of the icon. The distance fade specifies two distance values, in blocks along with two alpha values which is used to lerp between the near and far distance. This can be obtained wither via `WaypointTransmitter#waypointIcon` or `TrackedWaypoint#icon` on the server or client, respectively.

```java
// We will assume that this constructor is made public for a more dynamic usage
public static Waypoint.Icon EXAMPLE_ICON = new Waypoint.Icon(
    new Waypoint.Icon.Fade(
        // The number of blocks that the location will be tracked precisely to, provided the chunk is loaded
        // It additionally represents that any value closer will use the near alpha when rendering the bar
        256,
        // The maximum number of blocks that the location will be tracked within its chunks before switching to an angle
        // It additionally represents that any value larger will use the far alpha when rendering the bar
        // Values between this and the above will lerp the alpha value
        1024,
        // The alpha value to use when the bar is less than the near distance
        1f,
        // The alpha value to use when the bar is greater than the far distance
        0.2f
    ),
    // The color of the waypoint
    // When not present, uses the hashcode of the waypoint identifier
    Optional.of(0xFFFFFF)
);
```

### Connections

Waypoints, when tracked, are managed through their connections via `WaypointTransmitter$Connection`. A connection is responsible for syncing information to the client, whether connecting, disconnecting, or updating.

```java
public class ExampleBlockConnection implements WaypointTransmitter.Connection {

    private final BlockState state;
    private final BlockPos pos;
    private final Waypoint.Icon icon;
    private final ServerPlayer receiver;

    public ExampleBlockConnection(BlockState state, BlockPos pos, Waypoint.Icon icon, ServerPlayer receiver) {
        this.state = state;
        this.pos = pos;
        this.icon = icon;
        this.recevier = receiver;
    }

    public static boolean doesSourceIgnoreReceiver(BlockPos pos, ServerPlayer player) {
        double receiveRange = player.getAttributeValue(Attributes.WAYPOINT_RECEIVE_RANGE);
        return pos.distSqr(player.blockPosition()) >= receiveRange * receiveRange;
    }

    @Override
    public boolean isBroken() {
        // When true, it will attempt to remake the connection to this transmitter
        // This is only called when updating the position
        return ExampleBlockConnection.doesSourceIgnoreReceiver(this.pos, this.receiver);
    }

    @Override
    public void connect() {
        // This sends the tracking packet to the client
        this.receiver.connection.send(ClientboundTrackedWaypointPacket.addWaypointPosition(this.state.toString() + ": " + this.pos.toString(), this.icon, this.pos));
    }

    @Override
    public void disconnect() {
        // This sends the removal packet to the client
        this.receiver.connection.send(ClientboundTrackedWaypointPacket.removeWaypoint(this.state.toString() + ": " + this.pos.toString()));
    }

    @Override
    public void update() {
        // This updates the tracked value on the client assuming that the connect has not broken
        // In our case, we can assume this never changes as the position of a block should remain consistent
    }
}
```

### Transmitters

A `WaypointTransmitter` is responsible for constructing the connection between it and the server. For your object to transmit a location, it must implement `WaypointTransmitter` and its three methods. `waypointIcon` simply returns the icon to display. `isTransmittingWaypoint` will determine whether the waypoint can be transmitted from this object. `makeWaypointConnectionWith` actually handles constructing the connection used to track the position or angle of the connection.

```java
public class ExampleWaypointBlock extends BlockEntity implements WaypointTransmitter {

    // ...

    @Override
    public boolean isTransmittingWaypoint() {
        // This should return false if no connection should be made
        return true;
    }

    @Override
    public Optional<WaypointTransmitter.Connection> makeWaypointConnectionWith(ServerPlayer player) {
        // Make a connection if nothing is ignored
        return ExampleBlockConnection.doesSourceIgnoreReceiver(this.getBlockPos(), player)
            ? Optional.empty()
            : Optional.of(new ExampleBlockConnection(this.getBlockState(), this.getBlockPos(), this.waypointIcon(), player));
    }

    @Override
    public Waypoint.Icon waypointIcon() {
        return EXAMPLE_ICON;
    }
}
```

Then, all you need to do is track, update, or untrack the transmitter as necessary. This can be done using the methods provided by `ServerWaypointManager`:

```java
// We will assume we have access to:
// - ServerLevel serverLevel
// - ExampleWaypointBlock be

// Tracking the waypoint, such as on some initialization
serverLevel.getWaypointManager().trackWaypoint(be);

// Update the waypoint if the position changes
serverLevel.getWaypointManager().updateWaypoint(be);

// Remove the waypoint once it no longer exists
serverLevel.getWaypointManager().untrackWaypoint(be);
```

- `net.minecraft.client.multiplayer.ClientPacketListener#getWaypointManager` - Gets the client manager used for tracking waypoints.
- `net.minecraft.client.renderer.GameRenderer` now implements `TrackedWaypoint$Projector`
- `net.minecraft.client.waypoints.ClientWaypointManager` - A client-side manager for tracked waypoints.
- `net.minecraft.commands.arguments.WaypointArgument` - A static method holder that gets some waypoint transmitter from an entity.
- `net.minecraft.network.protocol.game`
    - `ClientboundTrackedWaypointPacket` - A packet that sends some operation for a waypoint to the client.
    - `ClientGamePacketListener#handleWaypoint` - Handles the waypoint packet on the client.
- `net.minecraft.server.ServerScoreboard$Method` enum is removed
- `net.minecraft.server.level.ServerLevel#getWaypointManager` - Gets the server manager used for tracking waypoints.
- `net.minecraft.server.level.ServerPlayer#isReceivingWaypoints` - Whether the player is receiving waypoints from other locations or entities.
- `net.minecraft.server.waypoints.ServerWaypointManager` - A server-side manager for tracked waypoints (e.g., players and transmitters).
- `net.minecraft.world.entity`
    - `Entity#getRequiresPrecisePosition`, `setRequiresPrecisePosition` - Handles when a more precise position is needed for tracking.
    - `LivingEntity` now implements `WaypointTransmitter`
- `net.minecraft.world.waypoints`
    - `TrackedWaypoint` - A waypoint that is identified by some UUID or string, displayed via a `Waypoint$Icon`, and specified via `$Type`.
    - `TrackedWaypointManager` - A waypoint manager for `TrackedWaypoint`s.
    - `Waypoint` - An interface that represents some positional location. This holds no information, and is typically used in the context of `TrackedWaypoint`.
    - `WaypointManager` - An interface that tracks and updates waypoints.
    - `WaypointTransmitter` - An object that transmits some position that can be connected to and tracked.

## Minor Migrations

The following is a list of useful or interesting additions, changes, and removals that do not deserve their own section in the primer.

### Render Pass Scissoring now only OpenGL

The scissoring state has been removed from the generic pipeline code, now only accessible through the OpenGL implementation. Generic region handling has been delegated to `CommandEncoder#clearColorAndDepthTextures`. Note that this does not affect the existing `ScreenRectangle` system that handles the blit facing scissoring.

- `com.mojang.blaze3d.opengl.GlRenderPass`
    - `isScissorEnabled` - Returns whether the scissoring state is enabled which crops the area that is rendered to the screen.
    - `getScissorX`, `getScissorY`, `getScissorWidth`, `getScissorHeight` - Returns the values representing the scissored rectangle.
    - `$ScissorState` - A class that holds a bounding rectangle to render within.
- `com.mojang.blaze3d.systems.CommandEncoder#clearColorAndDepthTextures` now has an overload that takes in four `int`s representing the region to clear the texture information within
- `com.mojang.blaze3d.systems`
    - `RenderPass#enableScissor` is removed
    - `RenderSystem#SCISSOR_STATE`, `enableScissor`, `disableScissor` are removed
    - `ScissorState` class is removed

### Attribute Receivers

Living entities now implement an interface called `AttributeReceiver`, which is meant as a callback to perform some logic when an `AttributeInstance` is modified in some way. The receiver interacts with the stored attributes by passing it in when constructing the `AttributeMap`. Note that the modifier is only called if it changes the attribute value.

- `net.minecraft.world.entity.LivingEntity` now implements `AttributeReceiver`
- `net.minecraft.world.entity.ai.attributes`
    - `AttributeMap` now has an overload that takes in an `AttributeReceiver`
    - `AttributeReceiver` - An interface that performs an operation if some attribute is modified.

### Tag Changes

- `minecraft:block`
    - `plays_ambient_desert_block_sounds` is split into `triggers_ambient_desert_sand_block_sounds`, `triggers_ambient_desert_dry_vegetation_block_sounds`
- `minecraft:entity_type`
    - `can_equip_harness`
    - `followable_friendly_mobs`
- `minecraft:item`
    - `harnesses`
    - `happy_ghast_food`
    - `happy_ghast_tempt_items`

### List of Additions

- `com.mojang.blaze3d.pipeline`
    - `BlendFunction#TRANSLUCENT_PREMULTIPLIED_ALPHA` - A blend function that assumes the target has a premultiplied alpha from the composite step.
    - `RenderPipeline`
        - `getSortKey` - Returns a value representing how the element should be sorted for rendering. Used for layer sorting.
        - `updateSortKeySeed` - Updates the seed of the sort key, currently unused.
- `com.mojang.blaze3d.systems.RenderSystem`
    - `outputColorTextureOverride` - Holds a texture containing the override color used instead of whatever is specified in the `RenderType` target.
    - `outputDepthTextureOverride` - Holds a texture containing the override depth used instead of whatever is specified in the `RenderType` target.
- `com.mojang.blaze3d.textures.GpuTexture#setUseMipmaps` - Sets whether the texture should use mipmaps at different distances.
- `net.minecraft.WorldVersion$Simple` - A simple implementation of the current world version.
- `net.minecraft.advancements.critereon.ItemUsedOnLocationTrigger$TriggerInstance#placedBlockWithProperties` - Creates a trigger where a block was placed with the specified property.
- `net.minecraft.client`
    - `GameNarrator#saySystemChatQueued` - Narrates a component if either system or chat message narration is enabled.
    - `NarratorStatus#shouldNarrateSystemOrChat` - Returns whether the current narration status is anything but `OFF`.
- `net.minecraft.client.data.models.BlockModelGenerators#createDriedGhastBlock` - Creates the dired ghast block model definition.
- `net.minecraft.client.data.models.model`
    - `ModelTemplates#DRIED_GHAST` - A template that uses the `minecraft:block/dried_ghast` parent.
    - `TextureMapping#driedGhast` - Creates the default texture mapping for the dried ghast model.
    - `TextureSlot#TENTACLES` - Provides a texture key `tentacles`.
- `net.minecraft.client.model`
    - `GhastModel#animateTentacles` - Animates the tentacles of a ghast.
    - `HappyGhastHarnessModel` - A model representing the a ghast harness.
    - `HappyGhastModel` - A model representing a 'tamed' ghast.
    - `QuadrupedModel#createBodyMesh` now takes in two booleans for handling if the left and right hind leg textures are mirrored, respectively. 
- `net.minecraft.client.renderer.entity.HappyGhastRenderer` - The renderer for a 'tamed' ghast.
- `net.minecraft.client.renderer.entity.state.HappyGhastRenderState` - The state of a 'tamed' ghast.
- `net.minecraft.client.renderer.texture.AbstractTexture#setUseMipmaps` - Sets whether the texture should use mipmapping.
- `net.minecraft.client.resources.model.EquipmentclientInfo$LayerType#HAPPY_GHAST_BODY` - A layer representing the body of a happy ghast.
- `net.minecraft.client.resources.sounds.RidingHappyGhastSoundInstance` - A tickable sound instance that plays when riding a happy ghast.
- `net.minecraft.commands.arguments.HexColorArgument` - An integer argument that takes in a hexadecimal color.
- `net.minecraft.data.recipes.RecipeProvider`
    - `dryGhast` - The recipe for a dried ghast.
    - `harness` - The recipe for a colored harness.
- `net.minecraft.network.FriendlyByteBuf#writeEither`, `readEither` - Handles an `Either` with the given stream encoders/decoders.
- `net.minecraft.network.codec.ByteBufCodecs#RGB_COLOR` - A stream codec that writes the RGB using three bytes.
- `net.minecraft.server.level.ServerLevel#updateNeighboursOnBlockSet` - Updates the neighbors of the current position. If the blocks are not the same (not including their properties), then `BlockState#affectNeighborsAfterRemoval` is called.
- `net.minecraft.util`
    - `ARGB#setBrightness` - Returns the brightness of some color using a float between 0 and 1.
    - `Mth#smallestSquareSide` - Takes the ceiled square root of a number.
- `net.minecraft.world.entity`
    - `Entity`
        - `isInClouds` - Returns whether the entity's Y position is between the cloud height and four units above.
        - `teleportSpectators` - Teleports the spectators currently viewing from the player's perspective.
    - `Mob#isWithinRestriction` - Returns whether the position is within the entity's restriction radius.
- `net.minecraft.world.entity.ai.control.MoveControl#setWait` - Sets the operation to `WAIT`.
- `net.minecraft.world.entity.ai.goal.TemptGoal`
    - `stopNavigation`, `navigateTowards` - Handles navigation towards the player.
    - `$ForNonPathfinders` - A tempt goal that navigates towards a wanted position rather than immediately pathfinding.
- `net.minecraft.world.entity.ai.navigation.PathNavigation#canNavigateGround` - Returns whether the entity can pathfind while on the ground.
- `net.minecraft.world.entity.ai.sensing.AdultSensorAnyType` - An adult sensor that ignores whether the entity is the same type as the child.
- `net.minecraft.world.entity.animal`
    - `HappyGhast` - An entity representing a happy ghast.
    - `HappyGhastAi` - The brain of the happy ghast.
- `net.minecraft.world.entity.monster.Ghast`
    - `faceMovementDirection` - Rotates the entity to face its current movement direction.
    - `$RandomFloatAroundGoal#getSuitableFlyToPosition` - Gets a position that the ghast should fly to.
- `net.minecraft.world.item.component.ItemAttributeModifiers`
    - `forEach` - Applies the consumer to all attributes within the slot group.
    - `$Builder#add` - Adds an attribute to apply for a given slot group with a display.
    - `$Display` - Defines how an attribute modifier should be displayed within its tooltip.
    - `$Default` - Shows the default attribute display.
    - `$Hidden` - Does not show any attribute info.
    - `$OverrideText` - Overrides the attribute text with the component provided.
- `net.minecraft.world.item.equipment.Equippable#harness` - Represents a harness to equip.
- `net.minecraft.world.level.Level#precipitationAt` - Returns the precipitation at a given position.
- `net.minecraft.world.level.block.DriedGhastBlock` - A block that represents a dried ghast.
- `net.minecraft.world.level.dimension.DimensionDefaults`
    - `CLOUD_THICKNESS` - The block thickness of the clouds.
    - `OVERWORLD_CLOUD_HEIGHT` - The cloud height level in the overworld.
- `net.minecraft.world.phys.Vec3#rotateClockwise90` - Rotates the vector 90 degrees clockwise (flip x and z and invert new x value).

### List of Changes

- `com.mojang.blaze3d.platform.Window#setGuiScale`, `getGuiScale` now deals with an `int` instead of a `double`
- `net.mineraft`
    - `DetectedVersion` no longer implements `WorldVersion`
    - `WorldVersion` methods now use record naming schema due to `WorldVersion$Simple` usage
        - `getDataVersion` -> `dataVersion`
        - `getId` -> `id`
        - `getName` -> `name`
        - `getProtocolVersion` -> `protocolVersion`
        - `getPackVersion` -> `packVersion`
        - `getBuildTime` -> `buildTime`
        - `isStable` -> `stable`
- `net.minecraft.client.GameNarrator`
    - `sayChat` -> `sayChatQueued`
    - `say` -> `saySystemQueued`
    - `sayNow` -> `saySystemNow`
- `net.minecraft.client.data.models.ItemModelGenerators#generateWolfArmor` -> `generateTwoLayerDyedItem`
- `net.minecraft.client.gui.components.DebugScreenOverlay#render3dCrosshair` now takes in the current `Camera`
- `net.minecraft.client.gui.components.debugchart.ProfilerPieChart`
    - `RADIUS` is now public
    - `CHART_Z_OFFSET` -> `PIE_CHART_THICKNESS`, now public
- `net.minecraft.client.renderer`
    - `DimensionSpecialEffects` no longer takes in the current cloud level and whether there is a ground
    - `LightTexture#getTarget` -> `getTexture`
- `net.minecraft.data.recipes.RecipeProvider#colorBlockWithDye` -> `colorItemWithDye`
- `net.minecraft.world.entity`
    - `AreaEffectCloud#setParticle` -> `setCustomParticle`
    - `FlyingMob` is replaced by calling `LivingEntity#travelFlying`
    - `LivingEntity#canBreatheUnderwater` is no longer `final`
- `net.minecraft.world.entity.ai.behavior`
    - `AnimalPanic` now has overloads that take in a radius or a position getter
    - `BabyFollowAdult#create` now returns a `OneShot<LivingEntity>`
    - `FollowTemptation` now has an overload that checks whether the entity needs to track the entity's eye height.
- `net.minecraft.world.entity.ai.goal.TemptGoal` now has an overload that takes in the stop distance
    - `mob` is now a `Mob`
    - `speedModifier` is now `protected`
- `net.minecraft.world.entity.ai.memory.MemoryModuleType#NEAREST_VISIBLE_ADULT` now holds a `LivingEntity`
- `net.minecraft.world.entity.ai.navigation`
    - `FlyingPathNavigation`, `GroundPathNavigation#setCanOpenDoors` -> `PathNavigation#setCanOpenDoors`
- `net.minecraft.world.entity.ai.sensing.AdultSensor` now looks for a `LivingEntity`
    - `setNearestVisibleAdult` is now `protected`
- `net.minecraft.world.entity.animal.Fox#isJumping` -> `LivingEntity#isJumping`
- `net.minecraft.world.entity.animal.horse.AbstractHorse#isJumping` -> `LivingEntity#isJumping`
- `net.minecraft.world.entity.monster`
    - `Ghast` now implements `Mob`
        - `$GhastLookGoal` is now public, taking in a `Mob`
        - `$GhastMoveControl` is now public, taking in whether it should be careful when moving and its speed
        - `$RandomFloatAroundGoal` is now `public`, taking in a `Mob` and a block distance
    - `Phantom` now implements `Mob`
- `net.minecraft.world.item.ItemStack#forEachModifier` now takes in a `TriConsumer` that provides the modifier display
- `net.minecraft.world.level.block.sounds.AmbientDesertBlockSoundsPlayer#playAmbientBlockSounds` has been split into `playAmbientSandSounds`, `playAmbientDryGrassSounds`, `playAmbientDeadBushSounds`, `shouldPlayDesertDryVegetationBlockSounds`; not one-to-one
- `net.minecraft.world.level.dimension.DimensionType` now takes in an optional integer representing the cloud height level
- `net.minecraft.world.level.storage.DataVersion` is now a record

### List of Removals

- `net.minecraft.client.renderer.DimensionSpecialEffects#getCloudHeight`, `hasGround`
- `net.minecraft.client.renderer.texture.AbstractTexture`
    - `defaultBlur`
    - `setFilter`
- `net.minecraft.world.entity.monster.Drowned#waterNavigation`, `groundNavigation`
- `net.minecraft.world.level.block.TerracottaBlock`
