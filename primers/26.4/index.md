# Minecraft 26.3 -> 26.4 Mod Migration Primer

This is a high level, non-exhaustive overview on how to migrate your mod from 26.3 to 26.4. This does not look at any specific mod loader, just the changes to the vanilla classes. All provided names use the official mojang mappings. You can find the [changelog here](./changelog.md).

This primer is licensed under the [Creative Commons Attribution 4.0 International](http://creativecommons.org/licenses/by/4.0/), so feel free to use it as a reference and leave a link so that other readers can consume the primer.

If there's any incorrect or missing information, please file an issue on this repository or ping @ChampionAsh5357 in the Neoforged Discord server.

Thank you to:

- @bl4ckscor3 for `BY_ID` within enums now replaced by `EnumStreamCodec#byId`

## Pack Changes

There are a number of user-facing changes that are part of vanilla which are not discussed below that may be relevant to modders. You can find a list of them on [Misode's version changelog](https://misode.github.io/versions/?id=26.4&tab=changelog).

## Debug Facts

The debug screen has been rewritten to better organize information into 'groups' with understandable contents, or 'facts'.

On its most basic level, `DebugScreenEntry`s add their elements to the `DebugScreenDisplayer` via `addToGroup` (replaces `addLine`) or `addFactToGroup`, which takes in a `DebugGroup` to display under, along with the information to display. For a given group, debug elements are displayed in the following order: any `DebugFact`s added via `addFactToGroup`, then lines added through `addToGroup`, and then finally the `DebugCustomRenderer` for non-text elements. As lines remain the same from previous versions, this will focus on `DebugFact`s and `DebugCustomRenderer`s.

`DebugFact` is essentially a `Component` wrapper, where a call to any of its methods to add data appends a sibling `Component`. There are many methods of doing so (e.g., `DebugFact#text`, `value`) which all delegate to `MutableComponent#append`. `DebugScreenDisplayer#addFactToGroup` constructs a fact by taking the `String` name of the fact, followed by a consumer to build the backing `Component`:

```java
// A debug entry example.
public class DebugExample implements DebugScreenEntry {

    @Override
    public void display(DebugScreenDisplayer displayer, @Nullable Level serverOrClientLevel, @Nullable LevelChunk clientChunk, @Nullable LevelChunk serverChunk) {
        // Adds a fact to display.
        // Will show up as 'Example Fact: Hello world!'.
        displayer.addFactToGroup(
            // The group to display the fact under in the debug menu.
            DebugGroups.MISC,
            // The name of the fact being displayed.
            // Will show up before the fact followed by a ':'.
            "Example Fact",
            // A builder to construct the debug fact component.
            fact -> fact.value("Hello world!")
        );
    }
}
```

`DebugCustomRenderer` displays last, used adding debug elements that cannot be represented as text. `extract` submits the elements to render to the screen. `height` and `width` meanwhile are used to properly align the element on screen (especially on the right side) such that no other debug information is drawn underneath, along with draw the background box for the group:


```java
// A debug entry example.
public class DebugExample implements DebugScreenEntry {

    @Override
    public void display(DebugScreenDisplayer displayer, @Nullable Level serverOrClientLevel, @Nullable LevelChunk clientChunk, @Nullable LevelChunk serverChunk) {
        // Adds a custom renderer to display items.
        displayer.addFactToGroup(
            // The group to display the fact under in the debug menu.
            DebugGroups.MISC,
            // The debug renderer to display in the group.
            new DebugCustomRenderer() {

                @Override
                public void extract(GuiGraphicsExtractor graphics, int left, int top, DebugColumn.Side side) {
                    // Submit the items to display.
                    // 'left' represents the starting X of the group.
                    // 'top' represents the starting Y of the element in the group.
                    graphics.item(new ItemStack(Items.DIRT), left, top);
                }

                @Override
                public int height() {
                    // The height of the element to render.
                    // This is technically oversimplified as it should be using the bounds
                    // initialized in `GuiItemRenderState`, which is not easily accessible.
                    return 16;
                }

                @Override
                public int width(int groupWidth) {
                    // The width of the element to render.
                    // This is technically oversimplified as it should be using the bounds
                    // initialized in `GuiItemRenderState`, which is not easily accessible.
                    return 16;
                }
            }
        );
    }
}
```

Custom `DebugGroup`s can be created quite easily via `DebugGroup$Builder`, calling either `titled` if the group should have a title header, or `titleless` if no header is given. Then, along with any configurations, the group is built by `build`, which can then be added to the `DebugScreenDisplayer`:


```java
// A debug group example.
// Note that the display order of debug groups is just whichever is encountered
// first in the debug entries.
public static final DebugGroup EXAMPLE_GROUP = DebugGroup.Builder.titled(
    // The name of the group.
    // Will be displayed left-aligned before any lines, facts, or renderers.
    "Example Group"
).withAccentColor(
    // The RGB color to act as a window sidebar spanning the group height.
    // Will be displayed on the left or right depending on which side its on.
    0xFF00FF
).withPreferredColumn(
    // The side the group will be displayed on, as long as the column is not full.
    // If the column is full, then the opposite column will be used first, then alternating
    // depending on fullness.
    DebugColumn.Side.RIGHT
).build();

// A debug entry example.
public class DebugExample implements DebugScreenEntry {

    @Override
    public void display(DebugScreenDisplayer displayer, @Nullable Level serverOrClientLevel, @Nullable LevelChunk clientChunk, @Nullable LevelChunk serverChunk) {
        displayer.addToGroup(
            // The group to display the fact under in the debug menu.
            DebugGroups.EXAMPLE_GROUP,
            "Example text"
        );
    }
}
```

## Placement Domains

`PlacementModifier`s must now specify the horizontal domain where a feature can be placed. This is done through `modifyXzDomain`, which takes in the previous domain (either modified by a different `PlacementModifier` or `0` for the original domain) and returns the new domain. Vanilla uses the domains as a validity check to make sure features can only start with `[-16, 31]` of the origin for each chunk.

```java
// An example `PlacementModifier`.
public record CenteredOffsetPlacement(IntProvider xzSize, int center) implements PlacementModifier {
    // The type used to register the placement.
    public static final MapCodec<CenteredOffsetPlacement> CODEC = RecordCodecBuilder.mapCodec(
        instance -> instance.group(
            IntProviders.codec(1, 16).fieldOf("xz_size").forGetter(CenteredOffsetPlacement::xzSize),
            ExtraCodecs.intRange(0, 15).fieldOf("center").forGetter(CenteredOffsetPlacement::center)
        ).apply(instance, CenteredOffsetPlacement::new)
    );

    @Override
    public MapCodec<CenteredOffsetPlacement> codec() {
        return CODEC;
    }

    @Override
    public void modify(PlacementContext context, RandomSource random, BlockPos origin, Consumer<BlockPos> output) {
        // The modifier offsets the placement by the provided XZ size, subtracting the center value.
        int x = this.xzSize.sample(random);
        int z = this.xzSize.sample(random);
        output.accept(origin.offset(x - this.center, 0, z - this.center));
    }

    @Override
    public InclusiveRange<Integer> modifyXzDomain(InclusiveRange<Integer> inputDomain) {
        // Must match the horizontal domain.
        return new InclusiveRange<>(
            // The minimum possible value that can be outputted relative to the origin.
            inputDomain.minInclusive() + this.xzSize.minInclusive() - this.center,
            // The maximum possible value that can be outputted relative to the origin.
            inputDomain.maxInclusive() + this.xzSize.maxInclusive() - this.center
        );
    }
}
```

## Not a `RuleTestType`, but a `MapCodec`

Rule test types have joined the unrolling club, as `RuleTestType` is removed and replaced with a `MapCodec<RuleTest>`. As such, `RuleTest#getType` is now `codec` to return the registered `MapCodec`. Additionally, `RuleTest` is now an interface from an abstract class, meaning that the `codec` method is now `public`. All other signatures remain the same.

```java
// A Rule test example.
public record RandomTest(float probability) implements RuleTest {
    // The map codec used as the registry object.
    public static final MapCodec<RandomTest> CODEC = Codec.FLOAT.fieldOf("probability")
        .xmap(RandomTest::new, RandomTest::probability);

    @Override
    public boolean test(BlockState state, BlockPos pos, RandomSource random) {
        return random.nextFloat() < this.probability;
    }

    // Replaces `getType`.
    @Override
    public MapCodec<RandomTest> codec() {
        // Return the registry object.
        return CODEC;
    }
}

// Register the map codec to the appropriate registry.
Registry.register(
    BuiltInRegistries.RULE_TEST_TYPE,
    Identifier.fromNamespaceAndPath("examplemod", "random"),
    RandomTest.CODEC
);
```

## Biomes are Noise Biomes, but also Biomes

A large portion of the codebase around chunk generators resolving biomes has been further specified. Most of the 'biome resolver' classes and methods were renamed to 'noise biome resolvers' (e.g., `BiomeResolver` -> `NoiseBiomeResolver`). However, the original biome resolver methods and classes still remain, just under the connotation that the backing data used to resolve the biome may not specifically be directly from the generated noise.
