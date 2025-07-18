# Minecraft 1.20.5 -> 1.20.6 Mod Migration Primer

This is a high level, non-exhaustive overview on how to migrate your mod from 1.20.5 to 1.20.6. This does not look at any specific mod loader, just the changes to the vanilla classes. All provided names use the official mojang mappings.

This primer is licensed under the [Creative Commons Attribution 4.0 International](http://creativecommons.org/licenses/by/4.0/), so feel free to use it as a reference and leave a link so that other readers can consume the primer.

If there's any incorrect or missing information, please file an issue on this repository or ping @ChampionAsh5357 in the Neoforged Discord server.

## Pack Changes

There are a number of user-facing changes that are part of vanilla which are not discussed below that may be relevant to modders. You can find a list of them on [Misode's version changelog](https://misode.github.io/versions/?id=1.20.6&tab=changelog).

## Additions

- `net.minecraft.world.level.block.entity.BlockEntity#parseCustomNameSafe` - Attempts to serialize a string into a `Component`. On failure, `null` is returned
