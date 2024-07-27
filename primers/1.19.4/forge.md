# Minecraft 1.19.3 -> 1.19.4 Forge Mod Migration Primer

This is a high level, non-exhaustive overview on how to migrate your mod from 1.19.3 to 1.19.4 using Forge.

This primer is licensed under the [Creative Commons Attribution 4.0 International](http://creativecommons.org/licenses/by/4.0/), so feel free to use it as a reference and leave a link so that other readers can consume the primer.

If there's any incorrect or missing information, please leave a comment below. Thanks!

## Vanilla Changes

Vanilla changes are listed [here](./index.md).

## Creative Tabs

Custom creative tabs from the previous primer are now slightly modified to take in the two parameters: 

```java
// Registered on the MOD event bus
// Assume we have RegistryObject<Item> and RegistryObject<Block> called ITEM and BLOCK
@SubscribeEvent
public void buildContents(CreativeModeTabEvent.Register event) {
  event.registerCreativeModeTab(new ResourceLocation(MOD_ID, "example"), builder ->
    // Set name of tab to display
    builder.title(Component.translatable("item_group." + MOD_ID + ".example"))
    // Set icon of creative tab
    .icon(() -> new ItemStack(ITEM.get()))
    // Add default items to tab
    .displayItems((params, output) -> {
      output.accept(ITEM.get());
      output.accept(BLOCK.get());
    })
  );
}
```

Additionally, `net.minecraftforge.event.CreativeModeTabEvent$BuildContents` can access the parameters via `#getParameters`. The other methods now delegate to the parameters for better compatibility when updating from 1.19.3.

## Spawn Events Refactor

As of 45.0.23, spawn events have been completely refactored. For starters, `LivingSpawnEvent` has been renamed to `MobSpawnEvent`. Even further `CheckSpawn` and `SpecialSpawn` hae been merged into a single event: `FinalizeSpawn`. `FinalizeSpawn` can be canceled to prevent `Mob#finalizeSpawn` from being called while the entity itself can be prevent using` FinalizeSpawn#setSpawnCancelled`.

If you want to learn more about this event and the technical changes, see the [blog post](https://blog.minecraftforge.net/breaking/spawnevents/).

## New Registries

Forge has added a new static registry for `ItemDisplayContext`s aptly named `forge:display_contexts` for registering perspectives an item may be rendered within, replacing custom `TransformType`s. There can only be at most 256 display contexts.

## Sprite Registration Refactoring

As of 45.0.25, all `net.minecraftforge.client.event.RegisterParticleProvidersEvent#register` methods have been deprecated for removal, opting to switch to method names which better specify their usecase:

* `#register(ParticleType, ParticleProvider)` -> `#registerSpecial`
* `#register(ParticleType, ParticleProvider$Sprite)` -> `#registerSprite`
* `#register(ParticleType, ParticleEngine$SpriteParticleRegistration)` -> `#registerSpriteSet`

## Minor Changes

* `net.minecraftforge.fml.CrashReportCallables` can now be supplied a callable which will append to the system report when the boolean supplier returns `true`.
