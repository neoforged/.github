# Minecraft 26.3 -> 26.4 Mod Migration Primer

This is a high level, non-exhaustive overview on how to migrate your mod from 26.3 to 26.4. This does not look at any specific mod loader, just the changes to the vanilla classes. All provided names use the official mojang mappings. You can find the [changelog here](./changelog.md).

This primer is licensed under the [Creative Commons Attribution 4.0 International](http://creativecommons.org/licenses/by/4.0/), so feel free to use it as a reference and leave a link so that other readers can consume the primer.

If there's any incorrect or missing information, please file an issue on this repository or ping @ChampionAsh5357 in the Neoforged Discord server.

## Pack Changes

There are a number of user-facing changes that are part of vanilla which are not discussed below that may be relevant to modders. You can find a list of them on [Misode's version changelog](https://misode.github.io/versions/?id=26.4&tab=changelog).

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

## Biomes are Noise Biomes, but also Biomes

A large portion of the codebase around chunk generators resolving biomes has been further specified. Most of the 'biome resolver' classes and methods were renamed to 'noise biome resolvers' (e.g., `BiomeResolver` -> `NoiseBiomeResolver`). However, the original biome resolver methods and classes still remain, just under the connotation that the backing data used to resolve the biome may not specifically be directly from the generated noise.
