# Minecraft 1.21.10 -> 1.22 Mod Migration Primer

This is a high level, non-exhaustive overview on how to migrate your mod from 1.21.10 to 1.22. This does not look at any specific mod loader, just the changes to the vanilla classes. All provided names use the official mojang mappings.

This primer is licensed under the [Creative Commons Attribution 4.0 International](http://creativecommons.org/licenses/by/4.0/), so feel free to use it as a reference and leave a link so that other readers can consume the primer.

If there's any incorrect or missing information, please file an issue on this repository or ping @ChampionAsh5357 in the Neoforged Discord server.

Thank you to:

- @xfacthd for some educated guesses regarding the usage annotations

## Pack Changes

There are a number of user-facing changes that are part of vanilla which are not discussed below that may be relevant to modders. You can find a list of them on [Misode's version changelog](https://misode.github.io/versions/?id=1.22&tab=changelog).

## The Separation of Samplers

Blaze3d has separated setting the `AddressMode`s and `FilterMode`s when reading texture data into `GpuSampler`. As the name implies, a `GpuSampler` defines how to sample data from a buffer, such as a texture. `GpuSampler` contains four methods: `getAddressModeU` / `getAddressModeV` for determining how the sampler should behave when reading the UV positions (either repeat or clamp), and `getMinFilter` / `getMagFilter` for determining how to minify or magnify the texture respectively (either nearest neighbor or linear).

Samplers can be created via `GpuDevice#createSampler`, but that is not necessary in this case. As there are only 16 possible combinations, vanilla creates all `GpuSampler`s and stores them in a cache, accessible via `RenderSystem#getSamplerCache`, followed by `SamplerCache#getSampler`:

```java
GpuSampler sampler = RenderSystem.getSamplerCache().getSampler(
    // U address mode
    AddressMode.CLAMP_TO_EDGE,
    // V address mode
    AddressMode.CLAMP_TO_EDGE,
    // Minification filter
    FilterMode.LINEAR,
    // Magnification filter
    FilterMode.NEAREST
)
```

To make user of the sampler for a texture, when binding the texture in a render pass (via `RenderPass#bindTexture`), you must now specify the sampler to use in addition to the texture view:

```java
try (RenderPass pass = RenderSystem.getDevice().createCommandEncoder().createRenderPass(...)) {
    // Set other parameters
    pass.bindTexture(
        // The name of the sampler2D uniform, usually in the fragment shader
        "Sampler0",
        // The texture to sample
        ...,
        // The sampler to use
        sampler
    );

    // Draw buffer
}
```

Setting up the post processor has not changed from the user perspective as only clamp to edge address modes may be selected.

- `com.mojang.blaze3d.opengl`
    - `GlRenderPass`
        - `samplers` now is a hash map of strings to `GlRenderPass$TextureViewAndSampler`s
        - `$TextureViewAndSampler` - A record that defines a sampler with its sampled texture.
    - `GlSampler` - The OpenGL implementation of a gpu sampler.
    - `GlTexture#modesDirty`, `flushModeChanges` are removed
- `com.mojang.blaze3d.pipeline.RenderTarget#filterMode`, `setFilterMode` are removed
- `com.mojang.blaze3d.systems`
    - `GpuDevice#createSampler` - Creates a sampler for some source to destination with the desired address and filter modes.
    - `RenderPass#bindTexture` now takes in a `GpuSampler`
    - `RenderSystem`
        - `samplerCache` - Returns a cache of samples containing all possible combinations.
        - `setupOverlayColor` now takes in a `GpuSampler`
        - `setShaderTexture` now takes in a `GpuSampler`
        - `getShaderTexture` now returns a `$TextureAndSampler` instead of a `GpuTextureView`
        - `$TextureAndSampler` - A record that defines a sampler with its sampled texture.
    - `SamplerCache` - A cache of all possible samplers that may be used by the renderer.
- `com.mojang.blaze3d.textures`
    - `GpuSampler` - A buffer sampler with the specified UV address modes and minification and magnification filters.
    - `GpuTexture`
        - `addressModeU` -> `GpuSampler#getAddressModeU`
        - `addressModeV` -> `GpuSampler#getAddressModeV`
        - `minFilter` -> `GpuSampler#getMinFilter`
        - `magFilter` -> `GpuSampler#getMagFilter`
        - `setAddressMode`, `setTextureFilter` have been replaced by `SamplerCache#getSampler`, not one-to-one
- `net.minecraft.client.gui.render.TextureSetup` now takes in the `GpuSampler`s for each of the textures
    - This also includes the static constructors
- `net.minecraft.client.renderer.PostPass`
    - `$input#bilinear` - Whether to use a bilinear filter.
    - `$TextureInput` now takes in a `boolean` representing whether to use a bilinear filter

## Gizmos

Gizmos are the newest iteration in the submission and rendering decoupling, this time for debug renderers. However, the underlying structure to submit gizmos for rendering is much more complex since debug renderers can submit objects at nearly any point during the client's process.

### What is a Gizmo?

A `Gizmo` is basically an object that submits some object primitives -- specifically points, lines, triangle fans, quads, and text -- for rendering. Each gizmo essentially strings together these primitives via `emit` into its desired shape. During the render process, these make use of the `RenderPipelines#DEBUG_*` pipelines to render their primitives to the screen. Creating a new gizmo is as simple as extending the interface:

```java
// Store some parameters like a render state to submit the element primitives
public record ExampleGizmo(Vec3 start, Vec3 end) implements Gizmo {

    @Override
    public void emit(GizmoPrimitives gizmos) {
        // Submit any elements here
        gizmos.addLine(this.start, this.end, 0, 3f);
    }
}
```

Actually submitting the elements happens through `Gizmos#addGizmo`. This stores the gizmo to be emitted and drawn to the screen, as long as it is called during `Minecraft#tick` or `GameRenderer#render`, which is how the debug renderers emit theirs. All of the methods in `Gizmos` call `addGizmo` internally, which is why the method is typically absent outside it's class:

```java
// Somewhere in GameRenderer#render

Gizmos.addGizmo(new ExampleGizmo(Vec3.ZERO, Vec3.X_AXIS));

// Calls addGizmo internally
Gizmos.point(Vec3.ZERO, 0, 5f);
```

Calling `addGizmo` returns a `GizmoProperties`, which sets some properties for when the element is drawn. Currently, this is only `setAlwaysOnTop`, which tells the renderer to clear the depth texture before rendering the gizmo.

### Putting it Together

With that, how can you submit gizmos for rendering pretty much anywhere in the client pipeline? Well, this all starts from `Gizmos#withCollector` and `SimpleGizmoCollector`.

`SimpleGizmoCollector` is basically just a list that holds the collected gizmos to render. During the rendering process, `SimpleGizmoCollector#drainGizmos` is called, copying the gizmos over to a separate list for the renderer to `Gizmo#emit` them, before drawing them to the screen through the familiar frame pass and buffer source. `drainGizmos` clears the internal list it holds for the next frame.

To actually collect these elements, there is a rather convoluted process. Both `Minecraft` and `LevelRenderer` have their own `SimpleGizmoCollector`. `Minecraft` collects gizmos during the ticking process, while `LevelRenderer` collects gizmos during the submission/render phases. This is done by setting the collector using `Gizmo#withCollector`, which returns a `Gizmos$TemporaryCollection`. The collection is `AutoCloseable` that, when closed, releases the held collector on the local thread. So, the collectors are wrapped in a try-with-resources to facilitate the submission during these periods. Then, during the debug pass, the per tick gizmos are merged with the per frame gizmos and drawn to the screen, quite literally at the last moment in `LevelRenderer#renderLevel`.

- `net.minecraft.client.Minecraft`
    - `collectPerTickGizmos` - Returns a collection of all gizmos to emit.
    - `getPerTickGizmos` - Gets the gizmos to draw to the screen.
- `net.minecraft.client.renderer`
    - `LevelRenderer#collectPerFrameGizmos` - Returns a collection of all gizmos to emit.
    - `OrderedSubmitNodeCollector#submitHitbox` is removed
    - `ShapeRenderer`
        - `renderShape` now takes in a line width `float`
        - `renderLineBox` -> `Gizmos#cuboid`, not one-to-one
        - `addChainedFilledBoxVertices` -> `Gizmos#cuboid`, not one-to-one
        - `renderFace` -> `Gizmos#rect`, not one-to-one
        - `renderVector` -> `Gizmos#line`, not one-to-one
    - `SubmitNodeCollection#getHitboxSubmits` is removed
    - `SubmitNodeStorage$HitboxSubmit` record is removed
- `net.minecraft.client.renderer.debug`
    - `DebugRenderer`
        - `render` -> `emitGizmos`, no longer takes in the `PoseStack`, `BufferSource` or `boolean`, now taking in the partial tick `float`
        - `renderFilledUnitCube` -> `Gizmos#cuboid`, not one-to-one
        - `renderFilledBox` -> `Gizmos#cuboid`, not one-to-one
        - `renderTextOverBlock` -> `Gizmos#billboardTextOverBlock`, not one-to-one
        - `renderTextOverMob` -> `Gizmos#billboardTextOverMob`, not one-to-one
        - `renderFloatingText` -> `Gizmos#billboardText`, not one-to-one
        - `renderVoxelShape` -> `LevelRenderer#renderHitOutline`, now private, not one-to-one
        - `SimpleDebugRenderer$render` -> `emitGizmos`, no longer takes in the `PoseStack`, `BufferSource` or `boolean`, now taking in the partial tick `float`
    - `GameTestBlockHighlightRenderer`
        - `render` -> `emitGizmos`, taking in no parameters
        - `renderMarker` no longer takes in the `PoseStack` or buffer source
        - `$Marker#get*` are removed
    - `LightDebugRenderer` now takes in two flags determining whether to show the block or sky light
    - `PathfindingRenderer#renderPath`, `renderPathLine` no longer take in the `PoseStack` or buffer source
- `net.minecraft.client.renderer.entity.EntityRenderer#extractAdditionalHitboxes` is removed
- `net.minecraft.client.renderer.entity.state`
    - `EntityRenderState#hitboxesRenderState`, `serverHitboxesRenderState` are removed
    - `HitboxesRenderState` class is removed
    - `ServerHitboxesRenderState` class is removed
- `net.minecraft.client.renderer.feature.HitboxFeatureRenderer` -> `EntityHitboxDebugRenderer`, not one-to-one
- `net.minecraft.client.renderer.gizmos.DrawableGizmoPrimitives` - A storage and renderer for primitive shapes. or gizmos.
- `net.minecraft.client.renderer.texture`
    - `AbstractTexture`
        - `sampler`, `getSampler` - Returns the `GpuSampler` used by the texture.
        - `setClamp` -> `GpuSampler#getAddressMode*`, not one-to-one
        - `setFilter` -> `GpuSampler#get*Filter`, not one-to-one
    - `ReloadableTexture` no longer takes in the address mode and filter `boolean`s
- `net.minecraft.client.server.IntegratedServer#getPerTickGizmos` - Gets the gizmos to draw to the screen for the current tick.
- `net.minecraft.gizmos`
    - `ArrowGizmo` - A gizmo that draws an arrow.
    - `CircleGizmo` - A gizmo that draws an approximate circle with twenty vertices.
    - `CuboidGizmo` - A gizmo that draws a rectangular prism.
    - `Gizmo` - An object that can emit simple shape primitives to draw.
    - `GizmoCollector` - An add-only collector.
    - `GizmoPrimitives` - Shape primitives that can be drawn.
    - `GizmoProperties` - Properties that apply to how the gizmo should be drawn.
    - `Gizmos` - A collection of static methods for creating gizmos and collecting them.
    - `GizmoStyle` - A property holder to define how a gizmo should be drawn. These are used by the gizmos themselves and not the actual primitives.
    - `LineGizmo` - A gizmo that draws a line.
    - `PointGizmo` - A gizmo that draws a point.
    - `RectGizmo` - A gizmo that draws a rectangle.
    - `SimpleGizmoCollector` - A collector implementation for adding gizmos before sending them off for rendering.
    - `TextGizmo` - A gizmo that draws text.

## Permission Overhaul

The permission level integer has been expanded into an entirely new system that is both simple yet complicated. There are three main parts: `Permission`s, `PermissionSet`s, and `PermissionCheck`s.

### Permissions

`Permission`s are functionally data objects that define some sort of state. Vanilla provides two types of permissions: `Permission$Atom`, which just is a unique unit object; and `Permission$HasCommandLevel`, which holds the desired `PermissionLevel` for a command. Both of the data objects are registered as map codecs for dumping the command report.

```java
// Attempts to query moderator level permissions.
public static final Permission COMMANDS_MODERATOR = new Permission.HasCommandLevel(PermissionLevel.MODERATORS);
```

A custom permission can be created through some class or record that extends `Permission` with an associated `MapCodec` registered to its static registry:

```java
// This does not check whether the user has the given permission
// It is literally just holding data representing the permission state
public record HasExamplePermission(int state) implements Permission {
    public static final MapCodec<HasExamplePermission> MAP_CODEC = Codec.INT.fieldOf("state")
        .xmap(HasExamplePermission::new, HasExamplePermission::state);
    
    @Override
    public MapCodec<HasExamplePermission> codec() {
        return HasExamplePermission.MAP_CODEC;
    }
}

// In some registration handler
Registry.register(
    BuiltInRegistries.PERMISSION_TYPE
    ResourceLocation.withNamespaceAndPath("examplemod", "has_example_permission"),
    HasExamplePermission.MAP_CODEC
);

// Storing the permission for use
public static final Permission HAS_STATE_ONE = new HasExamplePermission(1);
```

### Permission Sets

If `Permission`s define the queryable state, then `PermissionSet`s are the permissions the user actually has. `PermissionSet` is a functional interface that checks whether the user has the desired permission state (via `hasPermission`). Vanilla uses a `LevelBasedPermissionSet` for checking whether the permission queried matches the current `PermissionLevel`. As a permission set is usually checked against some swathe of permissions, multiple permission sets can be combined into one via `PermissionSet#union`. It functionally performs an OR operation, meaning that if a set does not check a permission, it should default to `false`.

```java
public interface ExamplePermissionSet extends PermissionSet {

    // Keep track of the user's state for our permissions
    int state();

    @Override
    default boolean hasPermission(Permission permission) {
        // Check our permission
        if (permission instanceof HasExamplePermission example) {
            return this.state() >= example.state();
        }

        // Otherwise ignore
        return false;
    }
}

// Storing a permission set
// Could also be implemented or stored on the desired target
public static ExamplePermissionSet STATE_ONE = () -> 1;

// Check whether the permission set has the desired permission
STATE_ONE.hasPermission(HAS_STATE_ONE);
```

Currently, there is no simple method to store custom permission sets on the desired user. `CommandSourceStack` does have a method to union other permission sets via `withMaximumPermission`, but that would need to be handled within the associated `createCommandSourceStack` method. The normal `LevelBasedPermissionSet` can typically be queried through a `permissions` method, though there is no common interface across objects, even though `PermissionSetSupplier` seems to exist for that purpose.

### Permission Checks

Now, a `PermissionSet` never directly checks a `Permission` within the codebase. That would require having the object accessible at all times. Instead, a `PermissionCheck` object is created, which takes in the `PermissionSet` and `check`s whether the user has the desired data to continue execution. Vanilla provides two types of checks: `$AlwaysPass`, which means it will always return true; and `$Require`, which requires the set to have the desired `Permission`. The checks also have a map codec for dumping the command report.

```java
// Requires the permission set has acccess to moderator commands
public static final PermissionCheck LEVEL_MODERATORS = new PermissionCheck.Require(COMMANDS_MODERATOR);
```

Custom permission checks can be created through some class or record that implements `check` and registers the map codec to its static registry:

```java
public static record AnyOf(List<Permission> permissions) implements PermissionCheck {
    public static final MapCodec<AnyOf> MAP_CODEC = Permission.CODEC.listOf().fieldOf("permissions")
        .xmap(AnyOf::new, AnyOf::permissions);

    @Override
    public boolean check(PermissionSet permissionSet) {
        return this.permissions.stream().filter(perm -> permissionSet.hasPermission(perm)).findAny().isPresent();
    }

    @Override
    public MapCodec<AnyOf> codec() {
        return MAP_CODEC;
    }
}

// In some registration handler
Registry.register(
    BuiltInRegistries.PERMISSION_CHECK_TYPE
    ResourceLocation.withNamespaceAndPath("examplemod", "any_of"),
    AnyOf.MAP_CODEC
);

// Storing the check for use in a command
public static final PermissionCheck CHECK_STATE_ONE = new AnyOf(List.of(
    HAS_STATE_ONE,
    Permissions.COMMANDS_GAMEMASTER
));

// For some command
Commands.literal("example").requires(Commands.hasPermission(CHECK_STATE_ONE));
```

- `net.minecraft.client.multiplayer.ClientSuggestionProvider` no longer implements `PermissionSource`
    - The constructor now takes in a `PermissionSet` instead of a `boolean`
    - `allowsRestrictedCommands` -> `ClientPacketListener#ALLOW_RESTRICTED_COMMANDS`, now private, not one-to-one
- `net.minecraft.client.player.LocalPlayer#setPermissionLevel` -> `setPermissions`, not one-to-one
- `net.minecraft.commands`
    - `Commands`
        - `LEVEL_*` are now `PermissionCheck`s instead of `int`s
        - `hasPermission`  now takes in a `PermissionCheck` instead of an `int`, and returns a `PermissionProviderCheck` instead of a `PermissionCheck`
    - `CommandSourceStack` no longer implements `PermissionSource`
        - The constructor now takes in a `PermissionSet` instead of an `int`
        - The `protected` constructor is now `private`
        - `withPermission` now takes in a `PermissionSet` instead of an `int`
        - `withMaximumPermission` now takes in a `PermissionSet` instead of an `int`
    - `ExecutionCommandSource` now extends `PermissionSetSupplier` instead of `PermissionSource`
    - `PermissionSource` interface is removed
    - `SharedSuggestionProvider` now extends `PermissionSetSupplier`
- `net.minecraft.commands.arguments.selector.EntitySelectorParser#allowSelectors` now has an overload that takes in a `PermissionSetSupplier`
- `net.minecraft.core.registries`
    - `BuiltInRegistries#PERMISSION_TYPE`, `Registries#PERMISSION_TYPE` - An object that defines some data requirement.
    - `BuiltInRegistries#PERMISSION_CHECK_TYPE`, `Registries#PERMISSION_CHECK_TYPE` - A predicate that checks whether the set has the required data.
- `net.minecraft.server`
    - `MinecraftServer`
        - `operatorUserPermissionLevel` -> `operatorUserPermissions`, not one-to-one
        - `getFunctionCompilationLevel` -> `getFunctionCompilationPermissions`, not one-to-one
        - `getProfilePermissions` now returns a `LevelBasedPermissionSet`
    - `ReloadableServerResources#loadResources` now takes in a `PermissionSet` instead of an `int`
    - `ServerFunctionLibrary` now takes in a `PermissionSet` instead of an `int`
    - `WorldLoader$InitConfig` now takes in a `PermissionSet` instead of an `int`
- `net.minecraft.server.commands.PermissionCheck` -> `.server.permissions.PermissionCheck`, not one-to-one
- `net.minecraft.server.dedicated.DedicatedServerProperties`
    - `opPermissionLevel` -> `opPermissions`, not one-to-one
    - `functionPermissionLevel` -> `functionPermissions`, not one-to-one
    - `deserializePermissions`, `serializePermission` - Reads and writes the chosen level permission set. 
- `net.minecraft.server.jsonrpc.internalapi`
    - `MinecraftOperatorListService#op` now takes in an optional `PermissionLevel` instead of an `int`
    - `MinecraftServerSettingsService`
        - `getOperatorUserPermissionLevel` -> `getOperatorUserPermissions`, not one-to-one
        - `setOperatorUserPermissionLevel` -> `setOperatorUserPermissions`, not one-to-one
- `net.minecraft.server.jsonrpc.methods`
    - `OperatorService$OperatorDto#permissionLevel` now takes in an optional `PermissionLevel` instead of an `int`
    - `ServerSettingsService`
        - `operatorUserPermissionLevel` now returns a `PermissionLevel` instead of an `int`
        - `setOperatorUserPermissionLevel` now takes in and returns a `PermissionLevel` instead of an `int`
- `net.minecraft.server.permissions`
    - `LevelBasedPermissionSet` - A set of permissions that checks whether the user has an equal or higher command permission level.
    - `Permission` - Data related to the user's permissions, such as command level.
    - `PermissionCheckTypes` - The types of permission checks vanilla provides.
    - `PermissionLevel` - Defines a level sequence for a permission.
    - `PermissionProviderCheck` - A predicate that checks that tests the supplier's permission set against a check.
    - `Permissions` - The permissions vanilla provides.
    - `PermissionSet` - A set of a permissions to user has, but mostly defines a method to determine whether a user has the desired permission.
    - `PermissionSetSupplier` - An object that supplies a `PermissionSet`.
    - `PermissionTypes` - The types of permissions vanilla provides.
- `net.minecraft.server.players`
    - `PlayersList#op` now takes in an optional `LevelBasedPermissionSet` instead of an `int`
    - `ServerOpListEntry` now takes in a `LevelBasedPermissionSet` instead of an `int`
        - `getLevel` -> `permissions`, not one-to-one
- `net.minecraft.world.entity.player.Player#getPermissionLevel`, `hasPermissions` -> `permissions`, not one-to-one
- `net.minecraft.world.entity.projectile`
    - `AbstractArrow#findHitEntities` - Gets all entities hit by the vector.
    - `ProjectileUtils`
        - `getHitEntitiesAlong` - Gets the entities hit along the provided path.
        - `getManyEntityHitResult` - Gets all entities hit along the path of the two points within the bounding box.

## New Data Components

With the addition of the spear, a number of data components have been added to provide the associated functionality. The following is an brief overview of these components.

### Use Effects

`DataComponents#USE_EFFECTS` defines some effects to apply to the player that is using (e.g., right-clicking) an item. Currently, there are only two types of effects: whether the player can sprint when using the item, and the scalar that is applied to the player's horizontal movement.

```java
// For some item registration
new Item(new Item.Properties.component(
    DataComponents.USE_EFFECTS,
    new UseEffects(
        // Whether the player can sprint while using the item
        true,
        // The scalar applied to the player's horizontal movement
        0.5f
    )
));
```

### Damage Type

`DataComponents#DAMAGE_TYPE` defines the damage type applied to an entity when hit with this item. It takes in either the `ResourceKey` of the damage type or the `DamageType` object itself.

```java
// For some item registration
new Item(new Item.Properties.component(
    DataComponents.DAMAGE_TYPE,
    new EitherHolder<>(
        // The damage type this item applies to the attacked entity
        DamageTypes.FALLING_ANVIL
    )
));
```

### Swing Animation

`DataComponents#SWING_ANIMATION` defines the animation to play when swinging or attacking (e.g., left-clicking) with the item. There are three types of animation to play: `SwingAnimationType#NONE`, which does nothing; `WHACK`, which plays the standard swing animation; and `STAB`, which plays the spear thrust animation. The length of the animation can also be specified.

```java
// For some item registration
new Item(new Item.Properties.component(
    DataComponents.SWING_ANIMATION,
    new SwingAnimation(
        // The animation to play
        SwingAnimationType.NONE,
        // The amount of time to play the animation for, in ticks
        20
    )
));
```

### Minimum Attack Charge

`DataComponents#MINIMUM_ATTACK_CHARGE` determines how long the player must wait before making another attack with the item. The charge is a value between 0 and 1, which determines percentage of the delay to wait before making another attack. The delay is determined by the player's attack speed. This is checked twice if the player's action is a stab.

```java
// For some item registration
new Item(new Item.Properties.component(
    DataComponents.MINIMUM_ATTACK_CHARGE,
    // The percentage of time the player must wait before making another attack with this item
    0.5f
));
```

### Piercing Weapon

`DataComponents#PIERCING_WEAPON` sets the player's attack as not an attack, but as a stab or piercing attack. This is a separate action than swinging, which either attacks the entity or breaks a block. A piercing weapon can attack an entity, but is unable to break blocks. It also applies any enchantment effects for lunging. Piercing weapons are only applied to the player.

The logic pipeline flows like so:

- If `Player#cannotAttackWithItem` returns true, then the pipeline is terminated
- Piercing attack is handled via:
    - Client - `MultiPlayerGameMode#piercingAttack`
    - Server - `PiercingWeapon#attack`
- Server-only:
    - Get all entities that:
        - Are between the min and max reach of the weapon, in blocks
        - Are within the hitbox constructed from the reach starting at the player's eye position
        - If `PiercingWeapon#canHitEntity` returns true:
            - Entity can be hit by a projectile
            - If both players, that this player can harm the other player
            - If living, the living entity was not stabbed in the last ten ticks
            - Is not a passenger of the same vehicle
    - Call `LivingEntity#stabAttack` on each entity
- `LivingEntity#onAttack` is fired
- `LivingEntity#lungeForwardMaybe` is fired
- Server-only: `PiercingWeapon#makeHitSound` and `makeSound` is played
- `LivingEntity#swing` is fired

```java
// For some item registration
new Item(new Item.Properties.component(
    DataComponents.PIERCING_WEAPON,
    new PiercingWeapon(
        // The minimum reach in blocks for this item to hit the entity
        2f,
        // The maximum reach in blocks for this item to hit the entity
        4.5,
        // The margin to increase the hitbox by in blocks
        // This is to compensate for potential precision issues
        0.25f,
        // Whether being hit by this item deals knockback to the entity
        true,
        // Whether being hit by this item dismounts the entity from its vehicle
        true,
        // The sound to play when attacking with this item
        SoundEvents.LLAMA_SWAG,
        // The sound to play when this item hits an entity
        SoundEvents.ITEM_BREAK
    )
));
```

### Kinetic Weapon

`DataComponents#KINETIC_WEAPON` affects an entity's use (e.g., right-click) behavior. On right-click, if an item has the component, then `KineticWeapon#damageEntities` is called every tick instead of `Item#onUseTick`, only on the server. The kinetic weapon also calls `LivingEntity#stabAttack` to damage its entities similar to piercing attack. In fact, the component itself is roughly a copy of `PiercingWeapon`, except with a few additional fields to handle the kinetic damage applied and to make it accessible to all living entities instead of only the player.

```java
// For some item registration
new Item(new Item.Properties.component(
    DataComponents.KINETIC_WEAPON,
    new KineticWeapon(
        // The minimum reach in blocks for this item to hit the entity
        2f,
        // The maximum reach in blocks for this item to hit the entity
        4.5,
        // The margin to increase the hitbox by in blocks
        // This is to compensate for potential precision issues
        0.25f,
        // The number of ticks to wait before attempting to stab any entities in range
        20,
        // The condition to check whether an attack from this item will dismount
        // an entity in a vehicle
        new KineticWeapon.Condition(
            // The maximum number of ticks from first use plus delay that this
            // condition may return true
            100,
            // The minimum speed the entity must be traveling for this condition
            // to succeed
            // The speed is calculated as the dot product of the delta movement
            // and the view vector multiplied by 20
            // Vanilla spears use values from 7-14 for dismount and 5.1 for knockback
            9f,
            // The minimum speed relative to the attacking entity that this entity
            // must be traveling for this condition to succeed
            // Vanilla spears use 4.6 for damage
            5f
        ),
        // The condition to check whether an attack from this item will cause knockback
        // to an entity
        KineticWeapon.Condition.ofAttackerSpeed(
            // Maximum ticks
            100,
            // Entity traveling speed
            5.1f
        ),
        // The condition to check whether an attack from this item will damage an
        // entity
        KineticWeapon.Condition.ofRelativeSpeed(
            // Maximum ticks
            100,
            // Relative traveling speed
            4.6f
        ),
        // The movement of the item during the third person attack animation
        // Vanilla spears use 0.38
        0.38f,
        // A multiplier to apply to the damage of an entity
        // The damage is calculated as the relative traveling speed of this entity
        // to its target
        4f,
        // The sound to play when first using this item
        SoundEvents.LLAMA_SWAG,
        // The sound to play when this item hits an entity
        SoundEvents.ITEM_BREAK
    )
));
```

- `net.minecraft.core.components`
    - `DataComponents`
        - `USE_EFFECTS` - The effects to apply to the entity when using the item.
        - `MINIMUM_ATTACK_CHARGE` - The minimum amount of time to attack with the item.
        - `DAMAGE_TYPE` - The `DamageType` the item deals
        - `PIERCING_WEAPON` - A weapon with some hitbox range that lunges towards the entity.
        - `KINETIC_WEAPON` - A weapon with some hitbox range that requires some amount of forward momentum.
        - `SWING_ANIMATION` - The animation applied when swinging an item.
- `net.minecraft.core.component.DataComponentType#ignoreSwapAnimation`, `$Builder#ignoreSwapAnimation` - When true, the swap animation does not affect the data component 'usage'.
- `net.minecraft.core.component.predicates`
    - `AnyValue` - A predicate that checks if the component exists on the getter.
    - `DataComponentPredicate`
        - `$Type` is now an interface
            - Its original implementation has been replaced by `$TypeBase`
        - `$AnyValueType` - A type that uses the `AnyValue` predicate.
        - `$ConcreteType` - A type that defines a specific predicate.
- `net.minecraft.world.entity.LivingEntity#SWING_DURATION` -> `SwingAnimation#duration`, not one-to-one
- `net.minecraft.world.item`
    - `ItemStack`
        - `getSwingAnimation` - Returns the swing animation of the item.
        - `getDamageSource` - Returns the damage source the item provides when hit.
    - `SwingAnimationType` - The type of animation played when swinging the item.
- `net.minecraft.world.item.component`
    - `KineticWeapon` - A weapon with some hitbox range that requires some amount of forward momentum. 
    - `PiercingWeapon` - A weapon with some hitbox range that lunges towards the entity.
    - `SwingAnimation` - The animation applied when swinging an item.
    - `UseEffects` - The effects to apply to the entity when using the item.

## Minor Migrations

The following is a list of useful or interesting additions, changes, and removals that do not deserve their own section in the primer.

### Usage Annotations

Mojang has recently given some integer values and flags an annotation, marking its intended usage. This does not affect modders in any way, as it likely seems to be a way to perform static analysis on the values passed around, probably for some kind of validation.

- `com.mojang.blaze3d.buffers.GpuBuffer$Usage` - An annotation that marks whether a given integer defines the usage flags of the particular buffer.
- `com.mojang.blaze3d.platform.InputConstants$Value` - An annotation that marks whether a given integer defines the input of a device.
- `com.mojang.blaze3d.buffers.GpuTexture$Usage` - An annotation that marks whether a given integer defines the usage flags of the particular texture.
- `net.minecraft.client.input`
    - `InputWithModifiers$Modifiers` - An annotation that marks whether a given integer defines the modifiers of an input.
    - `KeyEvent$Action` - An annotation that marks whether a given integer defines the action being performed by the input (i.e., press, release, repeat).
    - `MouseButtonInfo`
        - `$Action` - An annotation that marks whether a given integer defines the action being performed by the mouse (i.e., press, release, repeat).
        - `$MouseButton` - An annotation that marks whether a given integer defines the input of a mouse.
- `net.minecraft.server.level.TicketType$Flags` - An annotation that marks whether a given integer defines the flags of a ticket type.
- `net.minecraft.world.level.block.Block$UpdateFlags` - An annotation that marks whether a given integer defines the flags for a block update.

### Tag Changes

- `minecraft:block`
    - `can_glide_through`
- `minecraft:enchantment`
    - `exclusive_set/lunge`
- `minecraft:entity_type`
    - `burn_in_daylight`
    - `can_wear_nautilus_armor`
    - `nautilus_hostiles`
- `minecraft:item`
    - `zombie_horse_food`
    - `nautilus_bucket_food`
    - `nautilus_food`
    - `nautilus_taming_items`
    - `spears`
    - `enchantable/lunge`
    - `enchantable/sword` -> `enchantable/melee_weapon`, `enchantable/sweeping`

### List of Additions

- `com.mojang.blaze3d.opengl.GlConst#GL_POINTS` - Defines the points primitive as the type to render.
- `com.mojang.blaze3d.platform`
    - `InputConstants#MOUSE_BUTTON_*` - The inputs of a mouse click, represented by numbers as they may have different intended purposes.
    - `TextureUtil#solidify` - Modifies the texture by packing and unpacking the pixels to better help with non-darkened interiors within mipmaps.
- `com.mojang.blaze3d.vertex`
    - `DefaultVertexFormat`
        - `POSITION_COLOR_LINE_WIDTH` - A vertex format that specifies the position, color, and line width.
        - `POSITION_COLOR_NORMAL_LINE_WIDTH` - A vertex format that specifies the position, color, normal, and line width.
    - `VertexFormat$Mode#POINTS` - A vertex mode that draws points.
    - `VertexFormatElement#LINE_WIDTH` - A vertex element that takes in one float representing the width.
- `com.mojang.math`
    - `OctahedralGroup#permutation` - Returns the symmetric group.
    - `SymmetricGroup3#inverse` - Returns the inverse group.
- `net.minecraft.advancements.critereon.DataComponentMatchers$Builder#any` - Matches whether there exists some data for the component.
- `net.minecraft.client`
    - `Camera#cubicBiomeSampler` - Returns the sampler used to handle biome fog rendering.
    - `KeyMapping$Category#DEBUG` - The debug keyboard category.
    - `OptionInstance`
        - `$IntRangeBase`
            - `next` - Gets the next value.
            - `previous` - Gets the previous value.
        - `$SliderableEnum` - A slider that selects between enum options.
        - `$SliderableValueSet`
            - `next` - Gets the next value.
            - `previous` - Gets the previous value.
    - `Options`
        - `keyToggleGui` - A key mapping that toggles the in-game gui.
        - `keyToggleSpectatorShaderEffects` - A key mapping that toggles the shader effects tied to a camera entity.
        - `keyDebug*`, `debugKeys` - Key mappings for the debug renderers.
        - `weatherRadius` - Returns the radius of the weather particles to render in an area.
        - `cutoutLeaves` - Whether leaves should render in cutout or solid.
        - `vignette` - Whether a vignette should be applied to the screen.
        - `improvedTransparency` - Whether to use the transparency post processor.
- `net.minecraft.client.animation.definitions.NautilusAnimation` - The animation definitions for the nautilus.
- `net.minecraft.client.data.models.ItemModelGenerators#generateSpear` - Generates the spear item model.
- `net.minecraft.client.data.models.model.ModelTemplates#SPEAR_IN_HAND` - A template for the spear in hand model.
- `net.minecraft.client.gui.components`
    - `EditBox#setInvertHighlightedTextColor` - Sets whether to invert the highlighted text color.
    - `FocusableTextWidget`
        - `getPadding` - Returns the text padding.
        - `updateWidth` - Updates the width the text can take up.
        - `updateHeight` - Update the height the text can take up.
        - `$Builder` - Builds the component.
    - `MultiLineTextWidget#getTextX`, `getTextY` - Gets the text position.
    - `OptionsList`
        - `addHeader` - Adds a header entry.
        - `resetOption` - Resets the option value.
        - `$AbstractEntry` - Defines the element within the selection list.
        - `$HeaderEntry` - An entry that represents the header of a section.
    - `ResettableOptionWidget` - A widget that can reset its value to a default.
- `net.minecraft.client.gui.screens.inventory.EffectsInInventory`
    - `SPACING` - The spacing between effects.
    - `SPRITE_SQUARE_SIZE` - The size of the effect icon.
- `net.minecraft.client.gui.screens.options`
    - `OptionsSubScreen#resetOption` - Resets the option value to its default.
    - `VideoSettingsScreen#updateTransparencyButton` - Sets the transparency button to the current option value.
- `net.minecraft.client.input.InputQuirks#EDIT_SHORTCUT_KEY_LEFT`, `EDIT_SHORTCUT_KEY_RIGHT` -> `InputWithModifiers#hasControlDownWithQuirk`, not one-to-one
- `net.minecraft.client.model`
    - `HumanoidModel$ArmPose`
        - `SPEAR` - The spear third person arm pose.
        - `animateUseItem` - Modifies the `PoseStack` given the use time, arm, and stack.
    - `NautilusArmorModel` - The armor model for a nautilus.
    - `NautilusModel` - The model for a nautilus.
    - `NautilusSaddleModel` - The saddle model for a nautilus
    - `SpearAnimations` - The animations performed when using a spear.
- `net.minecraft.client.model.geom`
    - `ModelLayers`
        - `*NAUTILUS*` - The model layers for the nautilus.
        - `UNDEAD_HORSE*_ARMOR` - The armor model layers for the undead horse.
    - `PartName`
        - `INNER_MOUTH`, `LOWER_MOUTH` - Part names for a mouth.
        - `SHELL` - Part name for a shell.
- `net.minecraft.client.multiplayer.MultiPlayerGameMode#piercingAttack` - Initiates a lunging attack.
- `net.minecraft.client.renderer.Sheets#CELESTIAL_SHEET` - The atlas for the celestial textures.
- `net.minecraft.client.renderer.blockentity.BlockEntityWithBoundingBoxRenderer#STRUCTURE_VOIDS_COLOR` - The void color for a structure.
- `net.minecraft.client.renderer.entity.NautilusRenderer` - The entity renderer for a nautilus.
- `net.minecraft.client.renderer.entity.state`
    - `ArmedEntityRenderState`
        - `swingAnimationType` - The animation to play when swinging their hand.
        - `ticksUsingItem` - How many ticks the item has been used for.
        - `getUseItemStackForArm` - Returns the held item stack based on the arm.
    - `NautilusRenderState` - The entity render state of a nautilus.
    - `UndeadRenderState` - The entity render state for an undead humanoid.
- `net.minecraft.client.renderer.item.ItemModelResolver#swapAnimationScale` - Gets the scale of the swap animation for the stack.
- `net.minecraft.client.resources.model`
    - `BlockModelRotation#IDENTITY` - The identity rotation.
    - `EquipmentClientInfo#NAUTILUS_*` - The layers for the nautilus.
- `net.minecraft.core.Vec3i`
    - `multiply` - Multiplies each component with a provided scalar.
    - `toMutable` - Returns a mutable `Vector3i`.
- `net.minecraft.data.AtlasIds#CELESTIAL_SHEET` - The atlas for the celestial textures.
- `net.minecraft.network.protocol.game.ServerboundPlayerActionPacket$Action#STAB` - The player performed the stab action.
- `net.minecraft.server`
    - `MinecraftServer`
        - `getServerActivityMonitor` - Returns the monitor that sends the server activity notification.
        - `getStopwatches` - Returns a map of ids to timers.
    - `ServerScoreboard#storeToSaveDataIfDirty` - Writes the data if dirty.
- `net.minecraft.server.commands.StopwatchCommand` - A command that starts or stops a stopwatch.
- `net.minecraft.server.jsonrpc.OPutgoingRpcMethods#SERVER_ACTIVITY_OCCURRED` - A request made from the minecraft server about server activity occurring.
- `net.minecraft.server.level.ServerLevel#getDayCount` - Gets the number of days that has passed.
- `net.minecraft.server.network.ServerGamePacketListenerImpl#resetFlyingTicks` - Resets how long the player has been flying.
- `net.minecraft.server.notifications`
    - `NotificationService#serverActivityOccurred` - Notifies the management server that activity has occurred.
    - `ServerActivityMonitor` - The monitor that sends the server activity notification
- `net.minecraft.util`
    - `CubicSampler#update` - Updates the current biome manager and position.
    - `Ease` - A utility full of easing functions.
    - `ExtraCodecs`
        - `NON_NEGATIVE_LONG`, `POSITIVE_LONG` - Longs with the listed constraints.
        - `longRange` - A long codec that validates whether it is between the provided range.
    - `Mth#Cube` - Cubes a number.
- `net.minecraft.util.profiling.jfr.JvmProfiler#onClientTick` - Runs on client tick, taking in the current FPS.
- `net.minecraft.util.profiling.jfr.event.ClientFpsEvent` - An event that keeps track of the client FPS.
- `net.minecraft.util.profiling.jfr.stats.FpsStat` - A record containing the client FPS.
- `net.minecraft.world`
    - `Stopwatch` - A record that holds the creation time and amount of time that has elapsed.
    - `Stopwatches` - A tracker for starting, managing, and stopping stopwatches.
- `net.minecraft.world.effect.MobEffects#BREATH_OF_THE_NAUTILUS` - Prevents the user from losing air underwater.
- `net.minecraft.world.entity`
    - `Entity#getHeadLookAngle` - Calculates the view vector of the head rotation.
    - `LivingEntity`
        - `DEFAULT_KNOCKBACK` - The default knockback applied to an entity on hit.
        - `itemSwapTicker` - The amount of time taken when swapping items.
        - `recentKineticEnemies` - The attackers that have recently attacked with a kinetic weapon.
        - `lungeForwardMaybe` - Apply the lunge effects.
        - `causeExtraKnockback` - Applies an multiplicative force to the knockback.
        - `wasRecentlyStabbed`, `rememberStabbedEntity` - Handles enemies that were stabbed with a kinetic weapon.
        - `stabAttack` - Handles when a mob is stabbed by this entity.
        - `onAttack` - Handles when this entity has attacked another entity.
        - `getTicksUsingItem` - Returns the number of ticks this item has been used for.
    - `Mob#sunProtectionSlot` - The equipment slot that protects the entity from the sun.
    - `NeutralMob#level` - Returns the level the entity is in.
    - `PlayerRideableJumping#getPlayerJumpPendingScale` - Returns the scalar to apply to the entity on player jump.
- `net.minecraft.world.entity.ai.attributes.Attributes#DEFAULT_ATTACK_SPEED` - The default attack speed.
- `net.minecraft.world.entity.ai.behavior.ChargeAttack` - Handles a charge attack performed by a mob.
- `net.minecraft.world.entity.ai.goal.SpearUseGoal` - Handles a mob using a spear.
- `net.minecraft.world.entity.ai.memory.MemoryModuleType`
    - `CHARGE_COOLDOWN_TICKS` - The number of cooldown ticks after a charge attack.
    - `ATTACK_TARGET_COOLDOWN` - The number of cooldown ticks before attacking a target.
- `net.minecraft.world.entity.ai.sensing.SensorType#NAUTILUS_TEMPTATIONS` - A sensor for the items that tempt a nautilus.
- `net.minecraft.world.entity.animal.horse.AbstractHorse#isMobControlled` - Whether a mob can control this horse.
- `net.minecraft.world.entity.animal.nautilus`
    - `AbstractNautilus` - The core of the nautilus entity.
    - `Nautilus` - The nautilus entity.
    - `NautilusAi` - The brain of a nautilus.
    - `ZombieNautilus` - The zombie nautilus entity.
    - `ZombieNautilusAi` - The brain of a zombie nautilus.
- `net.minecraft.world.entity.player.Player`
    - `cannotAttackWithItem` - Checks whether the player cannot attack with the item.
    - `getItemSwapScale` - Returns the scalar to use for the item swap animation.
    - `resetOnlyAttackStrengthTicker` - Resets the attack strength ticker.
- `net.minecraft.world.item`
    - `Item$Properties`
        - `spear` - Adds the spear components.
        - `nautilusArmor` - Adds the nautilus armor components.
    - `ItemStack#matchesIgnoringComponents` - Whether the stack matches ignoring all components that match the predicate.
    - `ItemUseAnimation`
        - `TRIDENT` - The trident use animation.
        - `hasCustomArmTransform` - Whether the animation provides a custom transform to the arm.
- `net.minecraft.world.item.enchantment`
    - `Enchantment#doLunge` - Applies the post piercing attack effect.
    - `EnchantmentEffectComponents#POST_PIERCING_ATTACK` - The effect to apply after a piercing attack.
    - `EnchantmentHelper#doLungeEffects` - Applies the effect on lunge.
    - `LevelBasedValue$Exponent` - Applies an exponent given the base and power.
- `net.minecraft.world.item.enchantment.effects`
    - `ApplyEntityImpulse` - An entity effect that adds an impulse in the direction of the look angle.
    - `ScaleExponentially` - A value effect that multiplies the value by a number raised to some exponent.
- `net.minecraft.world.level.MoonPhase` - An enum representing the phases of the moon.
- `net.minecraft.world.phys.Vec3`
    - `offsetRandomXZ` - Offsets the point by a random amount in the XZ direction.
    - `rotation` - Computes the rotation of the vector.
    - `applyLocalCoordinatesToRotation` - Adds the components relative to the current rotation of the vector.
- `net.minecraft.world.scores`
    - `Scoreboard`
        - `packPlayerTeams` - Packs the player teams into a serializable format.
        - `packObjectives` - Packs the objectives into a serializable format.
        - `packDisplaySlots` - Packs the display slots into a serializable format.
    - `ScoreboardSaveData`
        - `getData`, `setData` - Handles the packed scoreboard.
        - `Packed$EMPTY` - Represents an empty scoreboard.

### List of Changes

- `com.mojang.blaze3d.systems`
    - `GpuDevice#createTexture` now has an overload that takes in a supplied label instead of the raw string
    - `RenderSystem`
        - `lineWidth` -> `VertexConsumer#setLineWidth`
        - `getShaderLineWidth` -> `Window#getAppropriateLineWidth`
- `com.mojang.math`
    - `OctahedralGroup#permute` -> `SymmetricGroup3#permuteAxis`
    - `SymmetricGroup3`
        - `permutation` -> `permute`
        - `permuteVector` -> `OctahedralGroup#rotate`
    - `Transformation` now takes in the interface, 'read only' variants of its arguments (e.g., `Vector3f` -> `Vector3fc`)
        - This also applies to the argument getter methods
- `net.minecraft.SharedConstants`
    - `DEBUG_WATER` -> `DebugScreenEntries#VISUALIZE_WATER_LEVELS`, not one-to-one
    - `DEBUG_HEIGHTMAP` -> `DebugScreenEntries#VISUALIZE_HEIGHTMAP`, not one-to-one
    - `DEBUG_COLLISION` -> `DebugScreenEntries#VISUALIZE_COLLISION_BOXES`, not one-to-one
    - `DEBUG_SUPPORT_BLOCKS` -> `DebugScreenEntries#VISUALIZE_ENTITY_SUPPORTING_BLOCKS`, not one-to-one
    - `DEBUG_LIGHT` -> `DebugScreenEntries#VISUALIZE_BLOCK_LIGHT_LEVELS`, `VISUALIZE_SKY_LIGHT_LEVELS`; not one-to-one
    - `DEBUG_SKY_LIGHT_SECTIONS` -> `DebugScreenEntries#VISUALIZE_SKY_LIGHT_SECTIONS`, not one-to-one
    - `DEBUG_SOLID_FACE` -> `DebugScreenEntries#VISUALIZE_SOLID_FACES`, not one-to-one
    - `DEBUG_CHUNKS` -> `DebugScreenEntries#VISUALIZE_CHUNKS_ON_SERVER`, not one-to-one
- `net.minecraft.advancements.critereon.EntityFlagsPredicate` now takes in optional booleans for if the entity is in water or fall flying
    - The associated `$Builder` methods have also been added
- `net.minecraft.client`
    - `Camera`
        - `setup` now takes in a `Level` instead of a `BlockGetter`
        - `get*` has been replaced by their record alternatives (e.g. `getEntity` -> `entity`)
        - `Vector3f` return values are replaced with `Vector3fc`
    - `GraphicsStatus` -> `GraphicsPreset`, not one-to-one
    - `KeyMapping` now has an overload that takes in the sort order
    - `OptionInstance$OptionInstanceSliderButton` now implements `ResettableOptionWidget`
    - `Options#graphicsMode` -> `graphicsPreset`, `applyGraphicsPreset`
- `net.minecraft.client.data.models`
    - `EquipmentAssetProvider#humanoidAndHorse` -> `humanoidAndMountArmor`
    - `ItemModelOutput#accept` now has an overload that takes in the `ClientItem$Properties`
- `net.minecraft.client.gui.GuiGraphics`
    - `textHighlight` now takes in a `boolean` of whether to render the background rectangle
    - `submitOutline` -> `renderOutline`
- `net.minecraft.client.gui.components`
    - `AbstractSliderButton`
        - `HANDLE_WIDTH` is now protected
        - `canChangeValue`, `setValue` are now protected
    - `AbstractWidget#message` is now protected
    - `CycleButton` now implements `ResettableOptionWidget`
        - `builder` now has an overload to take in a supplied default value
        - `booleanBuilder` now takes in a boolean to choose which component to default to
        - `$Builder` now takes in a supplied default value
    - `FocusableTextWidget` constructor is now package private, use `builder` instead
    - `OptionsList` now passes in an `$AbstractEntry` to the generic rather than an `$Entry`
        - `$Entry` now extends `$AbstractEntry`
    - `StringWidget#clipText` is now public static, taking in the `Font`
- `net.minecraft.client.gui.components.debug`
    - `DebugScreenEntryList`
        - `toggleF3Visible` -> `toggleDebugOverlay`
        - `setF3Visible` -> `setOverlayVisible`
        - `isF3Visible` -> `isOverlayVisible`
    - `DebugScreenEntryStatus#IN_F3` -> `IN_OVERLAY`
- `net.minecraft.client.gui.components.toasts.NowPlayingToast#renderToast` now takes in the `Minecraft` instance
- `net.minecraft.client.screens.Screen` now has an overload that takes in the `Minecraft` instance and `Font` to use
    - `minecraft` is now final
    - `font` is now final
    - `init(Minecraft, int, int)` -> `init(int, int)`
    - `resize(Minecraft, int, int)` -> `init(int, int)`
- `net.minecraft.client.gui.screens.inventory`
    - `AbstractCommandBlockEditScreen#populateAndSendPacket` no longer takes in the `BaseCommandBlock`
    - `EffectsInInventory#renderEffects` -> `render`
    - `MinecartCommandBlockEditScreen` now takes in a `MinecartCommandBlock` instead of a `BaseCommandBlock`
- `net.minecraft.client.model`
    - `AnimationUtils`
        - `animateCrossbowCharge` now takes in a `float` instead of an `int`
        - `animateZombieArms` now takes in an `UndeadRenderState` instead of two `float`s
    - `HumanoidModel#setupAttackAnimation` no longer takes in a `float`
- `net.minecraft.client.multiplayer.ClientLevel#getSkyColor` now takes in the `Camera` instead of the camera position
- `net.minecraft.client.renderer`
    - `DimensionSpecialEffects` no longer takes in the `boolean` for end flashes
    - `DynamicUniforms#writeTransform`, `$Transform` no longer take in the line width `float`
    - `ItemBlockRenderTypes#setFancy` -> `setCutoutLeaves`
    - `RenderPipelines`
        - `LINE_STRIP` -> `LINES`, not one-to-one
        - `DEBUG_LINE_STRIP` -> `DEBUG_POINTS`, not one-to-one
    - `RenderStateShard`
        - `$MultiTextureStateShard$Builder#add` no longer takes in a `boolean` whether to use mipmaps
        - `$TextureStateShard` no longer takes in a `boolean` whether to use mipmaps
    - `RenderType`
        - `LINE_STRIP`, `lineStrip` -> `LINES`, not one-to-one
        - `debugLineStrip` -> `debugPoint`, not one-to-one
    - `SkyRenderer` now takes in the `TextureManager` and `AtlasManager`
        - `extractRenderState` now takes in a `Camera` instead of the camera position
        - `renderSunMoonAndStars` now takes in a `MoonPhase` instead of an `int`
    - `UniformValue`
        - `$IVec3Uniform` now takes in a `Vector3ic` instead of a `Vector3i`
        - `$Vec2Uniform` now takes in a `Vector2fc` instead of a `Vector2f`
        - `$Vec3Uniform` now takes in a `Vector3fc` instead of a `Vector3f`
        - `$Vec4Uniform` now takes in a `Vector4fc` instead of a `Vector4f`
    - `WeatherEffectRenderer#tickRainParticles` now takes in an `int` for the weather radius
- `net.minecraft.client.renderer.block.model.BlockElementRotation` now takes in a `Vector3fc` for the origin and a `Matrix4fc` transform
- `net.minecraft.client.renderer.blockentity.TestInstanceRenderer` no longer takes in the `BlockEntityRendererProvider$Context`
- `net.minecraft.client.renderer.blockentity.state.BlockEntityWithBoundingBoxRenderState$InvisibleBlockType$STRUCUTRE_VOID` -> `STRUCTURE_VOID`
- `net.minecraft.client.renderer.chunk.ChunkSectionLayer` no longer takes in whether to use mipmaps
    - `textureView` -> `texture`, not one-to-one
- `net.minecraft.client.renderer.entity.layers.ItemInHandLayer#submitArmWithItem` now takes in the held `ItemStack`
- `net.minecraft.client.renderer.entity.state`
    - `ArmedEntityRenderState`
        - `*HandItem` -> `*HandItemState`, `*HandItemStack`; not one-to-one
        - `extractArmedRenderState` now takes in the partial tick `float`
    - `HorseRenderState#bodyArmorItem` -> `EquineRenderState#bodyArmorItem`
    - `HumanoidRenderState`
        - `attackTime` -> `ArmedEntityRenderState#attackTime`
        - `ticksUsingItem` is now a float
    - `IllagerRenderState` now extends `UndeadRenderState`
        - `ticksUsingItem` is now a float
    - `ZombieRenderState` now extends `UndeadRenderState`
    - `ZombifiedPiglinRenderState` now extends `UndeadRenderState`
- `net.minecraft.client.renderer.fog.environment.FogEnvironment#setupFog` no longer takes in the `Entity` and `BlockPos`, instead the `Camera`
- `net.minecraft.client.renderer.item.ClientItem$Properties` now takes in a float for changing the scale of the swap animation
- `net.minecraft.client.renderer.state.SkyRenderState#moonPhase` is now a `MoonPhase` instead of an `int`
- `net.minecraft.client.renderer.texture`
    - `MipmapGenerator#generateMipLevels` now takes in a `boolean` of whether the mipmaps should be darkened to emulate the darker interior of the block
    - `SpriteContents` now takes in whether the mipmaps should be darkened to emulate the darker interior of the block
    - `TextureMetadataSection` now takes in a `boolean` of whether the mipmaps should be darkened to emulate the darker interior of the block
- `net.minecraft.client.resources.model.BlockModelRotation` is now a class
    - `by` -> `get`, not one-to-one
- `net.minecraft.gametest.framework.GameTestHelper`
    - `spawn` now has an overload that takes in the `EntitySpawnReason` or three `int`s for the position
    - `assetTrue`, `assetFalse` now has an overload that takes in a `String` instead of a `Component`
- `net.minecraft.network.FriendlyByteBuf`
    - `writeVector3f` now takes in a `Vector3fc` instead of a `Vector3f`
    - `writeQuaternion` now takes in a `Quaternionfc` instead of a `Quaternionf`
- `net.minecraft.network.codec`
    - `ByteBufCodecs`
        - `VECTOR3F` now uses a `Vector3fc` instead of a `Vector3f`
        - `QUATERNIONF` now uses a `Quaternionfc` instead of a `Quaternionf`
    - `StreamCodec#composite` now has ten and twelve parameter variants
- `net.minecraft.network.syncher.EntityDataSerializers`
    - `VECTOR3` now uses a `Vector3fc` instead of a `Vector3f`
    - `QUATERNION` now uses a `Quaternionfc` instead of a `Quaternionf`
- `net.minecraft.server.MinecraftServer`
    - `isAllowedToEnterPortal` -> `ServerLevel#isAllowedToEnterPortal`
    - `isSpawningMonsters` -> `ServerLevel#isSpawningMonsters`
    - `isPvpAllowed` -> `ServerLevel#isPvpAllowed`
    - `isCommandBlockEnabled` -> `ServerLevel#isCommandBlockEnabled`
    - `isSpawnerBlockEnabled` -> `ServerLevel#isSpawnerBlockEnabled`
    - `getGameRules` -> `ServerLevel#getGameRules`
- `net.minecraft.server.ServerScoreboard` no longer implements its own saved data type, instead using the packed `ScoreboardSaveData`
    - `TYPE` -> `ScoreboardSavedData#TYPE`
- `net.minecraft.util`
    - `CubicSampler` is now used as an instance
        - `gaussianSampleVec3` -> `sampleVec3`, not one-to-one
    - `ExtraCodecs` now use the interface, 'read only' variants for its generic (e.g., `Vector3f` -> `Vector3fc`)
    - `Mth#easeInOutSine` -> `Ease#inOutSine`
- `net.minecraft.util.profiling.jfr.Percentiles#evaluate` now has an overload that takes in an `int[]`
- `net.minecraft.util.profiling.jfr.parse.JfrStatsResult` now takes in an FPS stat
    - `tickTimes` -> `serverTickTimes`
- `net.minecraft.util.profiling.jfr.stats.TimedStatSummary#summary` now returns an optional of the `TimeStatSummary`
- `net.minecraft.world.RandomSequences` no longer takes in the world seed
    - `codec` -> `CODEC`
    - `get`, `reset` now takes in the world seed
- `net.minecraft.world.entity`
    - `LivingEntity#invulnerableDuration` -> `INVULNERABLE_DURATION`
    - `Mob#playAttackSound` -> `LivingEntity#playAttackSound`
    - `NeutralMob`
        - `TAG_ANGER_TIME` -> `TAG_ANGER_END_TIME`, not one-to-one
        - `getRemainingPersistentAngerTime` -> `getPersistentAngerEndTime`, not one-to-one
        - `setRemainingPersistentAngerTime` -> `setTimeToRemainAngry`, `setPersistentAngerEndTime`; second is not one-to-one
        - `getPersistentAngerTarget`, `setPersistentAngerTarget` now deal with `EntityReference`s
- `net.minecraft.world.entity.ai.util`
    - `GoalUtils#mobRestricted` now takes in a `double` instead of an `int`
    - `LandRandomPos`
        - `getPosAway` now has an overload that takes in an additional `double` for the start/end radians
        - `generateRandomPosTowardDirection` now takes in a `double` instead of an `int`
    - `RandomPos`
        - `generateRandomDirectionWithinRadians` now takes in `double`s for the start/end radians
        - `generateRandomPosTowardDirection` now takes in a `double` instead of an `int`
- `net.minecraft.world.entity.monster`
    - `Monster#checkMonsterSpawnRules` now expanded its type generic to extends `Mob` instead of `Monster`
    - `Zombie`
        - `doUnderWaterConversion` now takes in the `ServerLevel`
        - `convertToZombieType` now takes in the `ServerLevel`
- `net.minecraft.world.entity.npc`
    - `AbstractVillager`
        - `updateTrades` now takes in the `ServerLevel`
        - `addOffersFromItemListings` now takes in the `ServerLevel`
    - `Villager#shouldRestock` now takes in the `ServerLevel`
    - `VillagerTrades$ItemListing#getOffer` now takes in the `ServerLevel`
- `net.minecraft.world.entity.player.Player`
    - `openMinecartCommandBlock` now takes in a `MinecartCommandBlock` instead of a `BaseCommandBlock`
    - `sweepAttack` -> `doSweepAttack`, now private, not one-to-one
- `net.minecraft.world.entity.vehicle`
    - `AbstractMinecart` now takes in the `ServerLevel`
    - `MinecartCommandBlock$MinecartCommandBase` is now package-private
- `net.minecraft.world.item.Item#getDamageSource` -> `getItemDamageSource`, now deprecated
- `net.minecraft.world.item.enchantment.effects.PlaySoundEffect` now takes in a list of sound events instead of a single
- `net.minecraft.world.level`
    - `BaseCommandBlock`
        - `performCommand` now takes in a `ServerLevel` instead of a `Level`
        - `onUpdated` now takes in a `ServerLevel`
        - `createCommandSourceStack` now takes in a `ServerLevel`
        - `$CloseableCommandBlockSource` now takes in a `ServerLevel`, with its constructor protected
    - `Level#getGameTime` -> `LevelAccessor#getGameTime`
    - `LevelAccessor#getCurrentDifficultyAt` -> `ServerLevelAccessor#getCurrentDifficultyAt`
    - `LevelTimeAccess#getMoonPhase` now returns a `MoonPhase` instead of an `int`
- `net.minecraft.world.level.biome`
    - `AmbientAdditionsSettings` is now a record
    - `AmbientMoodSettings` is now a record
    - `AmbientParticleSettings` is now a record
- `net.minecraft.world.level.dimension.DimensionType`
    - `MOON_PHASES` is now an array of `MoonPhase`s and private
    - `moonPhase` now returns a `MoonPhase` instead of an `int`
- `net.minecraft.world.level.saveddata.SavedDataType` no longer takes in a `SavedData$Context`, removing the function argument constructor
- `net.minecraft.world.level.storage.DimensionDataStorage` no longer takes in a `SavedData$Context`
- `net.minecraft.world.phys.Vec3` now takes in a `Vector3fc` instead of a `Vector3f`
- `net.minecraft.world.scores.ScoreboardSavedData` now takes in a `ScoreboardSaveData$Packed` instead of a `Scoreboard`
    - `FILE_ID` merged into type
    - `loadFrom` -> `ServerScoreboard#load`
    - `pack` -> `ServerScoreboard#store`, now private, not one-to-one

### List of Removals

- `com.mojang.blaze3d.textures.GpuTexture#useMipmaps`, `setUseMipmaps`
- `com.mojang.blaze3d.vertex.VertexFormat$Mode#LINE_STRIP`
- `net.minecraft.client.Minecraft#useFancyGraphics`
- `net.minecraft.client.gui.components.CycleButton`
    - `onOffBuilder()`
    - `$Builder#withInitialValue`
- `net.minecraft.client.gui.screens.inventory.EffectsInInventory#renderTooltip`
- `net.minecraft.client.renderer`
    - `GpuWarnlistManager#dismissWarningAndSkipFabulous`, `isSkippingFabulous`
    - `RenderPipelines`
        - `CUTOUT_MIPPED`
        - `DEBUG_STRUCTURE_QUADS`, `DEBUG_SECTION_QUADS`
    - `RenderStateShard`
        - `BLOCK_SHEET_MIPPED`
        - `DEFAULT_LINE`, `$LineStateShard`
    - `RenderType`
        - `cutoutMipped`
        - `debugStructureQuads`, `debugSectionQuads`
        - `$CompositeState$CompositeStateBuilder#setLineState`
    - `SkyRenderer#initTextures`
- `net.minecraft.client.renderer.chunk.ChunkSectionLayer#CUTOUT_MIPPED`
- `net.minecraft.client.renderer.fog.environment.FogEnvironment#onNotApplicable`
- `net.minecraft.client.renderer.texture.AbstractTexture#setUseMipmaps`
- `net.minecraft.client.resources.model.BlockModelRotation#actualRotation`
- `net.minecraft.gametest.framework.GameTestHelper#setNight`, `setDayTime`
- `net.minecraft.server.ServerScoreboard#createData`, `addDirtyListener`
- `net.minecraft.util.CubicSampler$Vec3Fetcher`
- `net.minecraft.world.entity.Mob#isSunBurnTick`
- `net.minecraft.world.entity.animal.horse.ZombieHorse#checkZombieHorseSpawnRules`
    - Use `Monster#checkMonsterSpawnRules` instead
- `net.minecraft.world.entity.raid.Raid#TICKS_PER_DAY`
- `net.minecraft.world.level.BaseCommandBlock`
    - `getLevel`
    - `getUsedBy`, `getPosition`
- `net.minecraft.world.level.Level#TICKS_PER_DAY`
- `net.minecraft.world.level.saveddata.SavedData$Context`

CONTINUE: `net.minecraft.world.entity`