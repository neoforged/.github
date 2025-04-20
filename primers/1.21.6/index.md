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

Now, how are elements rendered onto screen? In the previous versions, this happened mainly based on the order of the render calls. However, the new system using two different methods of sorting. The first method handles the depth using strata and a linear tree. The second method uses a comparator based on three things (in order of priorty): the `ScreenRectangle` scissor, the `RenderPipeline` order, and the `TextureSetup`'s hash code. These are all stored on the `GuiElementRenderState` passed to the `GuiRenderState` via `GuiRenderState#submitGuiElement`.

Once the elements have been ordered, they are rendered starting at a Z of `0`, with each element rendered `0.01` ahead of the previous.

As a warning, due to the element sort order, certain custom configurations may result in incorrect screen renders. As such, it is imperative that you understand how the methods and their assigned values work when sorting elements.

#### Strata and Trees

The first method of sorting handles the z-depth an object is rendered at.

First let's start with the linear tree. As the name implies, it is basically a doubly-linked `GuiRenderState$Node` list. Each node contains its own list of elements to render. Navigating the node list is handled using `GuiRenderState#up` or `down`, where each either gets the next node, or creates a new one if it doesn't exist. There are also methods for adding a new node to the top of the list (`upToTop`) or to go back to the previous `node`. You can also push a specific node to a stack for use later (`pushCheckpoint` and `backToCheckpoint`) if you don't want to fix the navigation tree. Nodes within a given tree are rendered from bottom to top, meaning that `down` will render any elements submitted before the current node, while `up` will render any elements submitted after the current node.

Then there are strata. A stratum is essentially a linear tree. Strata are rendered in the order they are created, which means calling `nextStratum` will render all elements after the previous stratum. This can be used if you want to group elements into a specific layer. There are two things to note: you can not have any checkpoints to create another stratum, and you cannot navigate to a prior stratum.

#### The Comparator

The comparator handles sorting the elements within a given node in a linear tree in a strata.

##### Screen Rectangle Scissor

The `ScreenRectangle` is simply the area that the element is allowed to draw to, stored via `GuiElementRenderState#scissorArea`. Elements with no specified `ScreenRectangle` will be ordered first, followed by the minimum Y, the maximum Y, the minimum X, and the maximum X.

##### Render Pipeline

`RenderPipeline`s define the pipeline used to render some object to the screen, including its shaders, format, uniforms, etc. This is stored via `GuiElementRenderState#pipeline`. Pipelines are sorted in the order they are built. This means that `RenderPipelines#ENTITY_TRANSLUCENT` will be rendered before `RenderPipelines#GUI` if on the same layer and scissor rectangle. As this is a system that relies on classloading constants, if you want to add new elements, make sure your mod loader supports some kind of dynamic pipeline ordering.

##### Texture Hash Code

`TextureSetup` defines `Sampler0`, `Sampler1`, and `Sampler2` for use in the render pipelines, stored via `GuiElementRenderState#textureSetup`. Elements with no textures will be ordered first, followed by the hash code of the record object. Note that this may be non-deterministic as `GpuTexture` does not implement `hashCode`, relying instead on the identity object hash.

### GuiElementRenderState

Now that we understand ordering, what exactly is the `GuiElementRenderState` that we've been using? Well essentially, every object rendered to the screen is represented by a `GuiElementRenderState`, from the player seen in the inventory menu to each individual item. A `GuiElementRenderState` defines four methods. First, there are the common ones used for ordering and how to render to the screen (`pipeline`, `scissorArea`, `textureSetup`). Then there is `buildVertices`, which takes in the `VertexConsumer` to write the vertices to and the z-depth. For GUIs, this typically calls `VertexConsumer#addVertexWith2DPose`.

There are three types of `GuiElementRenderState`s provided by vanilla: `BlitRenderState`, `ColoredRectangleRenderState`, and `GlyphRenderState`. `ColoredRectangleRenderState` and `GlyphRenderState` are simple cases for handling a basic color rectangle and text character, respectively. `BlitRenderState` covers every other case as almost every method eventually writes to a `GpuTexture` which is then consumed by this.

`GuiElementRenderState`s are added to the `GuiRenderState` via `GuiRenderState#submitGuiElement`. This is called by `GuiGraphics#*line`, `fill`, and `blit*` methods.

#### GuiItemRenderState

`GuiItemRenderState` is a special case used to render an item to the screen. It takes in the stringified name of the item, the current pose, its XY coordinates, and its scissor area. The `ItemStackRenderState` it holds is what defines how the item is rendered.

Just before the 'render' phase, the `GuiRenderer` effectively turns the `GuiItemRenderState` into a `GuiElementRenderState`, more specifically a `BlitRenderState`. This is done by constructing an item atlas `GpuTexture` which the item is drawn to, and then that texture is submitted as a `BlitRenderState`. All `GuiItemRenderState`s use `RenderPipelines#GUI_TEXTURED_PREMULTIPLIED_ALPHA`.

`GuiElementRenderState`s are added to the `GuiRenderState` via `GuiRenderState#submitItem`. This is called by `GuiGraphics#render*Item*` methods.

#### GuiTextRenderState

`GuiTextRenderState` is a special case used to render text to the screen. It takes in the `TextRenderState`, the current pose, its XY coordinates, and its scissor area. The `TextRenderState` simply defines whether to draw the drop shadow, the default colors, how the text is displayed, light coordinates, the text and its effects as a list of glyphs, the line height, the empty glyph, and the max X.

Just before the 'render' phase, the `GuiRenderer` turns the `GuiTextRenderState` into a `GuiElementRenderState`, more specifically a `GlyphRenderState`. This performs a similar process as the item render state where the text is written to a `GpuTexture` to be consumed. Any backgrounds are rendered first, followed by the background effects, then the character, then finally the foreground effects.

`GuiElementRenderState`s are added to the `GuiRenderState` via `GuiRenderState#submitText`. This is called by `GuiGraphics#draw*String` and `render*Tooltip` methods.

#### Picture-in-Picture

Picture-in-Picture is a special case used to render arbitrary objects to a `GpuTexture` to be passed into a `BlitRenderState`. A Picture-in-Picture is made up of two components the `PictureInPictureRenderState`, and the `PictureInPictureRenderer`.

`PictureInPictureRenderState` is an interface which can store some data used to render the object to the texture. By default, it must supply the minimum and maximum XY coordinates, the texture scale, and the scissor area. Any other data can be added by the implementor.

```java
public record ExamplePIPRenderState(boolean data, int x0, int x1, int y0, int y1, float scale, @Nullable ScreenRectangle scissorArea)
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
        - `depthTreeUp` - Adds a new or goes to an existing node that will render after this current one.
        - `depthTreeUpToTop` - Adds a new node at the very top of the stratum that will render last.
        - `nextStratum` - Adds another layer to traverse through that renders after all nodes in the previous tree.
        - `depthTreeDown` - Adds a new or goes to an existing node that will render before this current one.
        - `depthTreeBack` - Goes to the previously linked node, which is either above or below some parent.
        - `depthTreePushCheckpoint` - Pushes the current node to a stack such that it can be jumped back to.
        - `depthTreeBackToCheckpoint` - Retrieves the previously pushed node to the stack.
        - All methods no longer take in a `RenderType`, `VertexConsumer`, or `Function<ResourceLocation, RenderType>`, instead specifying a `RenderPipeline` and a `TextureSetup` depending on the call
        - `drawString`, `drawStringWithBackdrop` no longer returns anything
        - `renderItem(ItemStack, int, int, int, int)` is removed
        - `drawSpecial` is removed, relaced by individual `submit*RenderState` depending on the special case
        - `$ScissorStack#peek` - Gets the last rectangle on the stack.
    - `LayeredDraw` class is removed
- `net.minecraft.client.gui.components.LogoRenderer#keepLogoThroughFade` - When true, keeps the logo visible even when the title screen is fading.
- `net.minecraft.client.gui.components.spectator.SpectatorGui#renderTooltip` -> `renderAction`
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
    - `AbstractContainerScreen`
        - `SLOT_ITEM_BLIT_OFFSET` is removed
        - `renderContents` - Renders the slot and labels of an inventory.
        - `renderCarriedItem` - Renders the held item.
        - `renderSnapbackItem` - Renders the swapped item with the held item.
        - `$SnapbackData` - Holds the information about the dragging or swapped item as it moves from the held location to its slot location.
    - `AbstractSignEditScreen#offsetSign` -> `getSignYOffset`, not one-to-one
    - `EffectsInInventory`
        - `renderEffects` is now public
        - `renderTooltip` has been overloaded to a public method
    - `InventoryScreen#renderEntityInInventory` no longer takes in the XY offset, instead taking in 4 `int`s to represent the region to render to
    - `ItemCombinerScreen#renderFg` is removed
- `net.minecraft.client.gui.screens.inventory.tooltip`
    - `ClientTooltipComponent#renderText` no longer takes in the pose and buffer, instead the `GuiGraphics` to submit the text for rendering
    - `TooltipRenderUtil#renderTooltipBackground` no longer takes in a Z offset
- `net.minecraft.client.gui.screens.social`
    - `PlayerEntry#refreshHasDraftReport` - Sets whether the current context as a report for this player.
    - `SocialInteractionsPlayerList#refreshHasDraftReport` - Refreshes whether the current context as a report for all players.
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
- `net.minecraft.client.renderer.entity.ItemRenderer`
    - `GUI_SLOT_CENTER_X`, `GUI_SLOT_CENTER_Y`, `ITEM_DECORATION_BLIT_OFFSET` are removed
    - `COMPASS_*` -> `SPECIAL_*`
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

## Blaze3d Changes

Just like the previous update Blaze3d has new redesigns that change how rendering is handled.

### Buffer Slices

The most important change within the rendering system is the use of `GpuBufferSlice`s. As the name implies, it simply takes some slice of a `GpuBuffer`, using an `offset` to indicate the starting index, and the `length` to indicate its size. When dealing with anything rendering related, even when passing to a shader, you will almost always be dealing with a `GpuBufferSlice`. A way to think about it is that the buffer is just some arbitrary list of object where a slice represents a specific object.

To create a slice, simply call `GpuBuffer#slice`, optionally providing the starting index and length. In most instances, you will not be calling this method directly, instead dealing with other methods that call it for you. Note that as you are generally writing data to these slices, they should be constructed before a try-with-resources `RenderPass` is created.

### Uniform Rework

The uniform system has been completely overhauled such that, unless you are familiar with specific features, it might seem completely foreign. In a nutshell, uniforms are now stored as either interface objects, a texel buffer, or a sampler. These are populated either by `GpuBuffer`s or `GpuBufferSlice`s.

#### Uniform Types

A uniform is currently represented as one of two `UniformType`s: either as an `UNIFORM_BUFFER`/interface block, or a `TEXEL_BUFFER`. When setting up a pipeline and calling `withUniform`, you must specify your uniform with one of the above types. If you choose to use a texel buffer, you must also specify the format of the texture data. Samplers have not changed.

```java
public static final RenderPipeline EXAMPLE_PIPELINE = RenderPipeline.builder(...)
    // Uses a uniform interface block called 'ExampleUniform'
    .withUniform("ExampleUniform", UniformType.UNIFORM_BUFFER)
    // Uses a buffer called 'ExampleTexel' that stores texture data as a 8-bit R value
    .withUniform("ExampleTexel", UniformType.TEXEL_BUFFER, TextureFormat.RED8I)
    // Perform other setups
    .build();
```

#### Interface Blocks

A `UNIFORM_BUFFER` is stored as a `std140` interface block. In GLSL, this is represented like so:

```glsl
// In assets/examplemod/shaders/core/example_shader.json

// ExampleUniform is the name of the block
layout(std140) uniform ExampleUniform {
    // The data within the block
    // Post Effects can only use int, float, vec2, vec3, ivec3, vec4, or mat4
    vec4 Param1;
    float Param2;
};
```

The values can be used freely inside the main block like so:

```glsl
// In assets/examplemod/shaders/core/example_shader.json

out vec4 fragColor;

void main() {
    fragColor = Param1;
}
```

The uniform block can either be inside the same file as the main function or, if it will be reused, as a `moj_import`, where the uniform file is within `assets/<modid>/shaders/include/<path>`. Note that any shaders used before resource packs are loaded cannot use `moj_import`.

Post Effects define their interface blocks in the `uniform` input:

```json5
// In assets/examplemod/post_effect/example_post_effect.json
// Inside a 'passes' object
{
    "vertex_shader": "...",
    // ...
    "uniforms": {
        // The name of the interface block
        "ExampleUniform": [
            // A parameter within the uniform block
            // See `UniformValue` for codec formats
            {
                // The name of the parameter
                "name": "Param1",
                // The parameter type, one of:
                // - int
                // - ivec3
                // - float
                // - vec2
                // - vec3
                // - vec4
                // - matrix4x4 (mat4)
                "type": "vec4",
                // The value stored in the uniform
                // Dynamic values from the codebase are no longer supported
                "value": [ 0.0, 0.0, 0.0, 0.0 ]
            },
            {
                "name": "Param2",
                "type": "float",
                "value": 0
            }
        ]
    }
}
```

#### Texel Buffers

Texel buffers are typically represented as a `isamplerBuffer` to be queried from, typically using `texelFetch`:

```glsl
// In assets/examplemod/shaders/core/example_shader.json

uniform isamplerBuffer ExampleTexel;

void main() {
    // Read the documentation to understand how texel fetch works
    texelFetch(ExampleTexel, gl_InstanceID);
}
```

#### Writing Custom Uniforms

Writing custom uniforms can only happen when you are the one responsible for creating the `RenderPass`. Like before, you create and cache the object before opening the `RenderPass`, then you set the uniform by calling `RenderPass#withUniform`. The only difference is that, instead of providing arbitrary objects, you must now either provide a `GpuBuffer` or `GpuBufferSlice` with the uniform data written to it. For a texel buffer, this is usually some encoded data in the desired texture format (such as a mesh). For an interface block, the easiest method is to use the `Std140Builder` to populate the buffer with the correct values.

There are many different methods of writing to a `GpuBuffer` or `GpuBufferSlice`, such as using the newly created `MappableRingBuffer`. It is up to you to figure out what method works best in your scenario. The following is simply one solution out of many.

```java
// Buffer for 'ExampleUniform'
// Takes in the name of the buffer, its usage, and the size
private final MappableRingBuffer ubo = new MappableRingBuffer(
    // Buffer name
    "Example UBO",
    // Buffer Usage
    // We set 128 as its used for a uniform and 2 since we are writing to it
    // Other bits can be found as constants in `GpuBuffer`
    GpuBuffer.USAGE_UNIFORM | GpuBuffer.USAGE_MAP_WRITE,
    // The size of the buffer
    // Easiest method is to use Std140SizeCalculator to properly calculate this
    new Std140SizeCalculator()
        // Param1
        .putVec4()
        // Param2
        .putFloat()
        .get()
);

// Buffer for 'ExampleTexel'
private final MappableRingBuffer utb = new MappableRingBuffer(
    // Bufer name
    "Example UTB",
    // We set 256 as its used for a texel buffer and 2 since we are writing to it
    GpuBuffer.USAGE_UNIFORM_TEXEL_BUFFER | GpuBuffer.USAGE_MAP_WRITE,
    // The size of the buffer
    // It will likely be larger than this for you
    3
);

// In some render method

// Populate the buffers as required

// As we are using a ring buffer, this simply uses the next available buffer in the list
this.ubo.rotate();
// Write the data to the buffer
try (GpuBuffer.MappedView view = RenderSystem.getDevice().createCommandEncoder().mapBuffer(this.ubo.currentBuffer(), false, true)) {
    Std140Builder.intoBuffer(view.data())
        .putVec4(0f, 0f, 0f, 0f)
        .putFloat(0f);
}

// Similar thing here
this.utb.rotate();
try (GpuBuffer.MappedView view = RenderSystem.getDevice().createCommandEncoder().mapBuffer(this.utb.currentBuffer(), false, true)) {
    view.data()
        .put((byte) 0)
        .put((byte) 0)
        .put((byte) 0);
}

// Create the render pass and pass in the buffers as part of the uniform
try (RenderPass pass = RenderSystem.getDevice().createCommandEncoder().createRenderPass(...)) {
    // ..
    pass.setUniform("ExampleUniform", this.ubo.currentBuffer());
    pass.setUniform("ExampleTexel", this.utb.currentBuffer());
    // ...
}
```

### Render Pass Scissoring now only OpenGL

The scissoring state has been removed from the generic pipeline code, now only accessible through the OpenGL implementation. Generic region handling has been delegated to `CommandEncoder#clearColorAndDepthTextures`. Note that this does not affect the existing `ScreenRectangle` system that handles the blit facing scissoring.

- `com.mojang.blaze3d.buffers`
    - `BufferType` enum is removed
    - `BufferUsage` enum is removed
    - `GpuBuffer` now takes two ints represent the size and usage, the type is no longer specified
        - `type` is removed
        - `usage` now returns an int
        - `slice` - Returns a record representing a slice of some buffer. The actual buffer is not modified in any way.
        - `$ReadView` -> `$MappedView`
    - `GpuBufferSlice` - A record that represents a slice of some buffer by holding a reference to the entire buffer, an offset index of the starting point, and the length of the slice.
    - `GpuFence` is now an interface, with its OpenGL implementation moved to `blaze3d.opengl.GlFence`
    - `Std140Builder` - A buffer inserter for an interface block laid out in the `std140` format. This is used for the uniform blocks within shaders.
    - `Std140SizeCalculator` - An object used for keeping track of the size of some interface block.
- `com.mojang.blaze3d.opengl`
    - `AbstractUniform` is removed, replaced by the `Std140*` classes
    - `BufferStorage` - A class that create buffers given its types. Its implementations define its general usage.
    - `DirectStateAccess`
        - `createBuffer` - Creates a buffer.
        - `bufferData` - Initializes a buffer's data store.
        - `bufferSubData` - Updates a subset of a buffer's data store.
        - `bufferStorage` - Creates the data store of a buffer.
        - `mapBufferRange` - Maps a section of a buffer's data store.
        - `unmapBuffer` - Relinguishes the mapping of a buffer and invalidates the pointer to its data store.
    - `GlBuffer` now takes in a `DirectStateAccess`, two ints, and a `ByteBuffer` instead of a `GlDebugLabel` and its `Buffer*` enums
        - `initialized` is removed
        - `persistentBuffer` - Holds a reference to an immutable section of some buffer.
        - `$ReadView` -> `$GlMappedView`
    - `GlCommandEncoder`
        - `executeDrawMultiple` now takes in a collection of strings that defines the list of required uniforms
        - `executeDraw` now takes in an `int` that represents the number of instances of the specified range of indices to be rendered
    - `GlConst#toGl` for `BufferType` and `BufferUsage` -> `bufferUsageToGlFlag`, `bufferUsageToGlEnum`
    - `GlDevice`
        - `USE_GL_ARB_buffer_storage` - Sets the extension flag for `GL_ARB_buffer_storage`.
        - `getBufferStorage` - Returns the storage responsible for creating buffers.
    - `GlProgram`
        - `Uniform` fields have been removed
        - `safeGetUniform`, `setDefaultUniforms` are removed
        - `bindSampler` is removed
        - `getSamplerLocations`, `getSamplers`, `getUniforms` -> `getUniforms` which returns a map of names to their `Uniform` objects
    - `GlRenderPass`
        - `uniforms` now is a `HashMap<String, GpuBufferSlice>`
        - `dirtSamplers` is removed
        - `isScissorEnabled` - Returns whether the scissoring state is enabled which crops the area that is rendered to the screen.
        - `getScissorX`, `getScissorY`, `getScissorWidth`, `getScissorHeight` - Returns the values representing the scissored rectangle.
        - `$ScissorState` - A class that holds a bounding rectangle to render within.
    - `Uniform` is now a sealed interface which is implemented as either a buffer object, texel buffer, or a sampler.
- `com.mojang.blaze3d.pipeline`
    - `BlendFunction#PANORAMA` is removed
    - `CompiledRenderPipeline#containsUniform` is removed
    - `RenderPipeline`
        - `$Builder#withUniform` now has an overload that can take in a `TextureFormat`
        - `$UniformDescription` now takes in a nullable `TextureFormat`
- `com.mojang.blaze3d.platform.Lightning` is now `AutoCloseable`
    - `setup*` -> `setupFor`, an instance method that takes in an `$Entry`
    - `updateLevel` - Updates the lighting buffer, taking in whether to use the nether diffused lighting.
- `com.mojang.blaze3d.shaders.UniformType` now only holds two types: `UNIFORM_BUFFER`, or `TEXEL_BUFFER`, and no longer takes in a count
- `com.mojang.blaze3d.systems`
    - `CommandEncoder`
        - `clearColorAndDepthTextures` now has an overload that takes in four `int`s representing the region to clear the texture information within
        - `writeToBuffer`, `mapBuffer(GpuBuffer, int, int)` now take in a `GpuBufferSlice` instead of a `GpuBuffer`
        - `createFence` - Creates a new sync fence.
    - `GpuDevice`
        - `createBuffer` no longer takes in a `BufferUsage`, with `BufferType` replaced by an `int`
        - `getUniformOffsetAlignment` - Returns the uniform buffer offset alignment.
    - `RenderPass`
        - `enableScissor` is removed
        - `bindSampler` can now take in a nullable `GpuTexture`
        - `setUniform` can either take in a `GpuBuffer` or `GpuBufferSlice` instead of the raw inputs
        - `drawIndexed` now has an overload that takes in an additional `int` that represents the number of instances of the specified range of indices to be rendered
        - `drawMultipleIndexed` now takes in a collection of strings that defines the list of required uniforms
        - `$UniformUploader#upload` now takes in a `GpuBufferSlice` instead of an array of `float`s
    - `RenderSystem`
        - `SCISSOR_STATE`, `enableScissor`, `disableScissor` are removed
        - `PROJECTION_MATRIX_UBO_SIZE` - Returns the size of the projection matrix uniform
        - `setShaderFog`, `getShaderFog` now deals with a `GpuBufferSlice`
        - `setShaderGlintAlpha`, `getShaderGlintAlpha` is removed
        - `setShaderLights`, `getShaderLights` now deals with a `GpuBufferSlice`
        - `setShaderColor`, `getShaderColor`
        - `setup*Lighting` methods are removed
        - `setProjectionMatrix`, `getProjectionMatrix` (now `getProjectionMatrixBuffer`) now deals with a `GpuBufferSlice`
        - `setShaderGameTime`, `getShaderGameTime` -> `setGlobalSettingsUniform`, `getGlobalSettingsUniform`; not one-to-one
        - `getDynamicUniforms` - Returns a list of uniforms to write for the shader.
        - `bindDefaultUniforms` - Binds the default uniforms to be used within a `RenderPass`
    - `ScissorState` class is removed
- `com.mojang.blaze3d.textures.TextureFormat#RED8I` - An 8-bit signed integer handling the red color channel.
- `com.mojang.blaze3d.vertex.DefaultVertexFormat#EMPTY` - A vertex format with no elements.
- `net.minecraft.client.renderer`
    - `CachedOrthoProjectionMatrixBuffer` - An object that caches the orthographic projection matrix, rebuilding if the width or height of the screen changes.
    - `CachedPerspectiveProjectionMatrixBuffer` - An object that caches the perspective projection matrix, rebuilding if the width, height, or field of view changes.
    - `CloudRenderer#endFrame` - Ends the current frame being rendered to by constructing a fence.
    - `CubeMap#render` no longer takes in the float for the partial tick.
    - `DynamicUniforms` - A class that writes the uniform interface blocks to the buffer for use in shaders.
    - `DynamicUniformStorage` - A class that holds the uniforms within a slice of a mappable ring buffer.
    - `FogParameters` record is removed, now stored in a general `GpuBufferSlice`
    - `FogRenderer` now implements `AutoCloseable`
        - `endFrame` - Ends the current frame being rendered to by constructing a fence.
        - `getBuffer` - Gets the buffer slice that holds the uniform for the current fog mode.
        - `setupFog` no longer takes in the `FogMode`, returning nothing
        - `$FogData#mode` is removed, replaced by `skyEnd`, `cloudEnd`
        - `$FogMode` enums have been replaced with `NONE` and `WORLD`, not one-to-one
    - `GameRenderer`
        - `getGlobalSettingsUniform` - Returns the uniform for the game settings.
        - `getLighting` - Gets the lighting renderer.
        - `setLevel` - Sets the current level the renderer is rendering.
    - `GlobalSettingsUniform` - An object fod handling the uniform holding the current game settings.
    - `LevelRenderer`
        - `endFrame` - Ends the current frame being rendered to by constructing a fence.
        - `getFogRenderer` - Returns the fog renderer.
    - `MappableRingBuffer` - An object that contains three buffers that are written to as required, then rotates to the next buffer to use on end. These are synced using fences.
    - `PanoramaRenderer#render` now takes in a boolean for whether to change the spin up the spin rather than two floats.
    - `PerspectiveProjectionMatrixBuffer` - An object that holds the projection matrix uniform.
    - `PostChain`
        - `load` now takes in the `CachedOrthoProjectionMatrixBuffer`
        - `addToFrame` no longer takes in the `Consumer<RenderPass>`
        - `process` no longer takes in the `Consumer<RenderPass>`
    - `PostChainConfig`
        - `$FixedSizedTarget` record is removed
        - `$FullScreenTarget` record is removed
        - `$InternalTarget` is now a record which can take in an optional width, height, whether the target is persistent, and the clear color to use
        - `$Pass#uniforms` is now a `Map<String, List<UniformValue>>`
        - `$Uniform` record is removed
    - `PostPass` is now `AutoCloseable`, taking in a `Map<String, List<UniformValue>>` for the uniforms along with a list of `PostPass$Input`s
        - `addToFrame` no longer takes in the `Consumer<RenderPass>`, with the `Matrix4f` passed in as a `GpuBufferSlice`
        - `$Input`
            - `bindTo` is removed
            - `texture` - Constructs a `GpuTexture` for the input given the map of resources.
            - `samplerName` - Returns the name of the sampler.
    - `SkyRenderer#renderSunMoonAndStars` no longer takes in the `FogParameters`
    - `UniformValue` - An interface that represents a uniform stored within an interface block.
- `net.minecraft.client.renderer.chunk.SectionRendererDispatcher$RenderSection#setDynamicTransformIndex`, `getDynamicTransformIndex` - Handles the index used to query the correct dynamic transforms for a given section.

## Minor Migrations

The following is a list of useful or interesting additions, changes, and removals that do not deserve their own section in the primer.

### Attribute Receivers

Living entities now implement an interface called `AttributeReceiver`, which is meant as a callback to perform some logic when an `AttributeInstance` is modified in some way. The receiver interacts with the stored attributes by passing it in when constructing the `AttributeMap`. Note that the modifier is only called if it changes the attribute value.

- `net.minecraft.world.entity.LivingEntity` now implements `AttributeReceiver`
- `net.minecraft.world.entity.ai.attributes`
    - `AttributeMap` now has an overload that takes in an `AttributeReceiver`
    - `AttributeReceiver` - An interface that performs an operation if some attribute is modified.

### Leashes

The leash system has been updated to support up to four on enabled entities. Additionally, the physics have been updated to more properly handle real-world elasicity of some stretchable object.

- `net.minecraft.client.renderer.entity.state.EntityRenderState`
    - `leashState` now returns a list of `$LeashState`s
    - `$LeashState#slack` - Whether the leash has any slack.
- `net.minecraft.world.entity`
    - `Entity`
        - `shearOffAllLeashConnections` - Handles when shears are used to removed a leash.
        - `dropAllLeashConnections` - Handles when all leashes should be removed from an entity.
        - `getQuadLeashHolderOffsets` - Gets the offsets when four leashes attached to an entity.
        - `supportQuadLeashAsHolder` - Returns whether the entity can be leashed up to four times.
        - `notifyLeashHolder`, `notifyLeasheeRemoved` - Handles when the leash is ticked on an entity and when its removed.
        - `setLeashOffset` is removed
        - `getLeashOffset` -> `Leashable#getLeashOffset`
    - `Leashable`
        - `MAXIMUM_ALLOWED_LEASHED_DIST` - The maximum distance a leash can be attached to an entity.
        - `AXIS_SPECIFIC_ELASTICITY` - The resistance of the leash along each axis.
        - `SPRING_DAMPENING` - The loss of energy when the leash is oscillating on stretch.
        - `TORSIONAL_ELASTICITY` - The resistance of the leash under torque.
        - `STIFFNESS` - The stiffness of the leash.
        - `ENTITY_ATTACHMENT_POINT` - Specifies the attachment point of the leash on the entity.
        - `LEASHER_ATTACHMENT_POINT` - Specifies the attachment point of the leash on the holder.
        - `SHARED_QUAD_ATTACHMENT_POINTS` - Specifies the attachment points of the leashes on an entity.
        - `canHaveALeashAttachedToIt` -> `canHaveALeashAttachedTo`, not one-to-one
        - `leashDistanceTo` - Returns the distance of the leash traveled between the holder and leashee.
        - `onElasticLeashPull` - Handles when the leash is being pulled upon.
        - `leashSnapDistance` - Returns when the leash will attempt to remove itself from the entity.
        - `leashElasticDistance` - Returns the max distance before the elasticity of the leash becomes a physics factor.
        - `handleLeashAtDistance` is removed
        - `angularFriction` - Returns the angular friction of the entity on a block or within a liquid.
        - `whenLeashedTo` - Notifies the entity that the leash is attached.
        - `elasticRangeLeashBehaviour`, `legacyElasticRangeLeashBehaviour` -> `checkElasticInteractions`, not one-to-one
        - `supportQuadLeash` - Whether the entity can be leashed up to four times.
        - `getQuadLeashOffsets` - Returns the offsets of each leash attached to the entity.
        - `createQuadLeashOffsets` - Creates the offsets for each of the four possible leash positions.
        - `leashableLeashedTo` - Gets all entities within a 32-block radius that are leashed to this holder.
        - `$LeashData#angularMomentum` - Returns the angular momentum of the entity when leashed.
        - `$Wrench` - A record which handles the force and torque applied to the leash.
- `net.minecraft.world.item.LeadItem#leashableInArea` -> `Leashable#leashableInArea`

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
- `net.minecraft.client.renderer.entity.layers.RopesLayer` - The render layer for the ropes used on a 'tamed' ghast.
- `net.mienecraft.client.renderer.entity.state.HappyGhastRenderState` - The state of a 'tamed' ghast.
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
    - `ExtraCodecs`
        - `VECTOR2F`
        - `VECTOR3I`
    - `Mth#smallestSquareSide` - Takes the ceiled square root of a number.
- `net.minecraft.world.entity`
    - `Entity`
        - `isInClouds` - Returns whether the entity's Y position is between the cloud height and four units above.
        - `teleportSpectators` - Teleports the spectators currently viewing from the player's perspective.
        - `isFlyingVehicle` - Returns whether the vehicle can fly.
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
- `net.minecraft.world.level.block`
    - `BaseRailBlock#rotate` - Rotates the current rail shape in the associated direction.
    - `DriedGhastBlock` - A block that represents a dried ghast.
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
- `net.minecraft.client.multiplayer.MultiPlayerGameMode#createPlayer` now takes in an `Input` instead of a `boolean`
- `net.minecraft.client.player.LocalPlayer` now takes in the last sent `Input` instead of a `boolean` for the shift key
    - `getLastSentInput` - Gets the last sent input from the server.
- `net.minecraft.client.renderer`
    - `DimensionSpecialEffects` no longer takes in the current cloud level and whether there is a ground
    - `LightTexture#getTarget` -> `getTexture`
- `net.minecraft.data.recipes.RecipeProvider#colorBlockWithDye` -> `colorItemWithDye`
- `net.minecraft.world.entity`
    - `AreaEffectCloud#setParticle` -> `setCustomParticle`
    - `Entity#checkSlowFallDistance` -> `checkFallDistanceAccumulation`
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
- `net.minecraft.world.item.ItemStack`
    - `forEachModifier` now takes in a `TriConsumer` that provides the modifier display
    - `hurtAndBreak` now has an overload which gets the `EquipmentSlot` from the `InteractionHand`
- `net.minecraft.world.level.block.sounds.AmbientDesertBlockSoundsPlayer#playAmbientBlockSounds` has been split into `playAmbientSandSounds`, `playAmbientDryGrassSounds`, `playAmbientDeadBushSounds`, `shouldPlayDesertDryVegetationBlockSounds`; not one-to-one
- `net.minecraft.world.level.dimension.DimensionType` now takes in an optional integer representing the cloud height level
- `net.minecraft.world.level.storage.DataVersion` is now a record

### List of Removals

- `net.minecraft.client.renderer.DimensionSpecialEffects#getCloudHeight`, `hasGround`
- `net.minecraft.client.renderer.texture.AbstractTexture`
    - `defaultBlur`
    - `setFilter`
- `net.minecraft.network.protocol.game.ServerboundPlayerCommandPacket$Action#*_SHIFT_KEY`
- `net.minecraft.world.entity.monster.Drowned#waterNavigation`, `groundNavigation`
- `net.minecraft.world.level.block.TerracottaBlock`
