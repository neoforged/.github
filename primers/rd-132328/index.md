# Minecraft rd-132211 -> rd-132328 Mod Migration Primer

This is a high level, non-exhaustive overview on how to migrate your mod from rd-132211 to rd-132328. This does not look at any specific mod loader, just the changes to the vanilla classes. All provided names use the official mojang mappings.

This primer is licensed under the [Creative Commons Attribution 4.0 International](http://creativecommons.org/licenses/by/4.0/), so feel free to use it as a reference and leave a link so that other readers can consume the primer.

If there's any incorrect or missing information, please file an issue on this repository or ping @ChampionAsh5357 in the Neoforged Discord server.

## Entities!

The `Player` character has now been abstracted into a generic `Entity`, which represents a dynamic object within the world. As such, most fields and methods on the player class was moved onto the `Entity`, with the player overriding the `tick` method to handle keyboard input. With this abstraction, a new entity called `Zombie` has also been added, which currently randomly flails their limbs around.

To create a new entity, you need to extend the `Entity` class. Any logic should be handled within the `tick` method:

```java
public class ExampleEntity extends Entity {

    // The level must be passed in, but any other parameters
    // are for setting up the entity in the world.
    public ExampleEntity(Level level, float x, float y, float z) {
        super(level);
        // Place the entity in some location.
        this.x = x;
        this.y = y;
        this.z = z;
    }

    @Override
    public void tick() {
        // Handle how the entity should move or interact with the world
        // and its objects.
    }
}
```

As there is no generic implementation for creating or updating `Entity`s, it is up to the implementer to determine how this should occur. For example:

```java
// We will create an array to manage our entities.
private static final List<ExampleEntity> ENTITIES = new ArrayList<>();

// Assume we can inject into the end of `RubyDung#init`
public void init() throws LWJGLException, IOException {
    // ...

    for (int i = 0; i < 10; i++) {
        ENTITIES.add(new ExampleEntity(this.level, 128f, 0f, 128f));
    }
}
```

As for updating the `Entity`, you will need to inject the call into `RubyDung#tick`:

```java
// Assume we can inject into the beginning of `RubyDung#tick`
public void tick() {
    for (ExampleEntity entity : ENTITIES) {
        entity.tick();
    }

    // ...
}
```

### Models

With this `Entity` framework came a partial framework to render models for the entity. The model handler allows for rectangular prisms to be defined that render appropriately in the world, assuming the initial positioning and texture binding is handled by some custom entity render method.

All models are made up of `Vertex`s, which define a point relative to the entity's position with some UV offset for the bound texture.`Polygon`s hold these vertices and uploads them to the GPU. The polygon scales the UV based on the assumption that the texture is exactly 64x32.

`Cube`s represent a part of the model. Unlike the name implies, a `Cube` actually stores a rectangular prism via `addBox`, which construct the `Vertex` and `Polygon` according to the given position and dimensions, determining what part of the texture to use based on the offsets passed into the constructor.

These `Cube`s are then directly stored on the entity itself, constructed within the constructor:

```java
public class ExampleEntity extends Entity {

    private final Cube body;

    public ExampleEntity(Level level, float x, float y, float z) {
        super(level);
        // ...

        // Create the model to use.
        // Each cube represents one box.
        this.body = new Cube(
            // The X texture offset.
            0,
            // The Y texture offset.
            0
        );
        this.body.addBox(
            // The starting XYZ relative to the entity's position.
            -4f, -8f, -4f
            // The width, height, and depth of the box.
            8, 8, 8
        );
    }
}
```

To render the `Cube`, the entity should provide a render method taking in the delta time, a value between 0 and 1. Then, within the render method, call `Cube#render`. The render method is responsible for setting up everything else about the entity.

```java
public class ExampleEntity extends Entity {

    private final Cube body;

    // ...

    public void render(float partialTick) {
        // Setup the render state for the entity
        GL11.glEnable(GL11.GL_TEXTURE_2D);
        // We will assume we managed to inject a texture into the root
        // of the JAR at the desired location.
        GL11.glBindTexture(GL11.GL_TEXTURE_2D, Textures.loadTexture("/examplemod/example_texture.png", GL11.NEAREST));
        GL11.glPushMatrix();

        // Handle model translation and interpolation
        GL11.glTranslatef(
            this.xo + (this.x - this.xo) * partialTick,
            this.yo + (this.y - this.yo) * partialTick,
            this.zo + (this.z - this.zo) * partialTick
        );
        GL11.glScalef(1.0f, -1.0f, 1.0f);
        GL11.glScalef(0.058333334f, 0.058333334f, 0.058333334f);
        GL11.glRotatef(this.rot * 57.29578f + 180.0f, 0.0f, 1.0f, 0.0f);

        // Update the model state
        this.body.yRot = (float) Math.sin(
            (double) System.nanoTime() / 1.0E9d * 10.0d * (double) this.speed + (double) this.timeOffs
        ) * 0.8f;

        // Render the model
        this.body.render();

        // Teardown the render state
        GL11.glPopMatrix();
        GL11.glDisable(GL11.GL_TEXTURE_2D);
    }
}
```

Similar to entity updating, it is up to the implementer to determine where the render method should be called within `RubyDung#render`, though it is recommended to do so around the same place as `Zombie`s:

```java
// Assume we can inject `RubyDung#render` right after the zombie render calls:
public void render(float a) {
    // ...

    for (ExampleEntity entity : ENTITIES) {
        entity.render(a);
    }

    // ...
}
```

- `com.mojang.rubydung`
    - `Entity` - A dynamic object within the world.
    - `Player` now extends `Entity`
        - `level` -> `Entity#level`
        - `xo`, `yo`, `zo` -> `Entity#xo`, `yo`, `zo`
        - `x`, `y`, `z` -> `Entity#x`, `y`, `z`
        - `xd`, `yd`, `zd` -> `Entity#xd`, `yd`, `zd`
        - `yRot`, `xRot` -> `Entity#yRot`, `xRot`
        - `bb` -> `Entity#bb`
        - `onGround` -> `Entity#onGround`
        - `resetPos` -> `Entity#resetPos`, now `protected` from `private`
        - `turn` -> `Entity#turn`
        - `tick` -> `Entity#tick`
            - `Player` override this method now
        - `move`, `moveRelative`, `Entity#move`, `moveRelative`
- `com.mojang.rubydung.character`
    - `Cube` - An object that contains a box of six polygons and eight vertices.
    - `Polygon` - A face defined by some vertices, though it often assumes that the face is a square with a 64x32 texture.
    - `Vec3` - A three-dimensional float point.
    - `Vertex` - A point with UV coordinates for the applied texture.
    - `Zombie` - An entity that runs randomly while flailing their limbs.

## Minor Migrations

The following is a list of useful or interesting additions, changes, and removals that do not deserve their own section in the primer.

### List of Removals

- `com.mojang.rubydung.Textures#bind`
