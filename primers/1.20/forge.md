# Minecraft 1.19.4 -> 1.20 Forge Mod Migration Primer

This is a high level, non-exhaustive overview on how to migrate your mod from 1.19.4 to 1.20 using Forge.

This primer is licensed under the [Creative Commons Attribution 4.0 International](http://creativecommons.org/licenses/by/4.0/), so feel free to use it as a reference and leave a link so that other readers can consume the primer.

If there's any incorrect or missing information, please leave a comment below. Thanks!

## Vanilla Changes

Vanilla changes are listed [here](./index.md).

## Creative Tab Registry

Nw tabs can be registered using one of the register methods while adding to existing tabs is done via `BuildCreativeModeTabContentsEvent` on the mod event bus. For new tabs, Forge patches in the ability to order tabs via `CreativeModeTab$Builder#withTabsBefore` and `#withTabsAfter`.

```java
// Registered on the MOD event bus
// Assume we have RegistryObject<Item> and RegistryObject<Block> called ITEM and BLOCK
@SubscribeEvent
public void buildContents(BuildCreativeModeTabContentsEvent event) {
    // Add to ingredients tab
    if (event.getTabKey() == CreativeModeTabs.INGREDIENTS) {
        event.accept(ITEM);
        event.accept(BLOCK); // Takes in an ItemLike, assumes block has registered item
    }
}
```

## Forge Config: Resource Pack Caching Removal

All entries related to resource pack caching were removed from the Forge common config. Vanilla now handles this by default.

## Forge Capability Class Removals

The `Capability*` which originally held the Forge capabilities have been removed. All calls should be delegated to `ForgeCapabilities` instead.

## Forge Network Message Consumers

Within the message builder for Forge networks, the `#consumer`, which handles the message logic, has been removed. Now, it has been broken into `#consumerNetworkThread` and `#consumerMainThread`. These methods determine whether the logic should be handled directly on the network thread or delegated to the consumer itself.

`#consumerMainThread` handles the packet for you, so it only accepts a `BiConsumer`.

```java
public void mainThreadMessage(MSG message, Supplier<NetworkEvent.Context> ctx) { /**/ }
```

`#consumerNetworkThread` can take in a `BiConsumer` or `ToBooleanBiFunction` depending on if you want to handle the packet via `#setPacketHandled` or using the return statement, respectively.

```java
public void networkThreadMessageC(MSG message, Supplier<NetworkEvent.Context> ctx) { 
    // ...
    ctx.get().setPacketHandled(true);
}

public boolean networkThreadMessageF(MSG message, Supplier<NetworkEvent.Context> ctx) { 
    // ...
    return true;
}
```

## Removal of Dummy Entries

Forge dummy entries handled by Forge registries have been completely removed.

## New Tags

* Item Tag `forge:tools/swords` -> `minecraft:swords`
* Item Tag `forge:tools/axes` -> `minecraft:axes`
* Item Tag `forge:tools/pickaxes` -> `minecraft:pickaxes`
* Item Tag `forge:tools/shovels` -> `minecraft:shovels`
* Item Tag `forge:tools/hoes` -> `minecraft:hoes`

## Minor Changes and Renames

* `net.minecraftforge.event.entity.player.PlayerEvent$BreakSpeed#getPos` -> `#getPosition` which now takes in an `Optional<BlockPos>`
* `net.minecraftforge.client.event.RenderLevelLastEvent` -> `RenderLevelStageEvent`
    * There is no direct translation between the events. You need to choose which stage suits your logic best.
* `net.miencraftforge.common.world.ModifiableBiomeInfo$BiomeInfo$Builder#getEffects` -> `#getSpecialEffects`
* `net.miencraftforge.client.extensions.IForgeTransformation#push` -> `IForgePoseStack#pushTransformation`
* `net.minecraftforge.client.event.RegisterParticleProvidersEvent#register` -> `#registerSpecial`, `#registerSprite`, `#registerSpriteSet` respectively
* `net.minecraftforge.common.extensions.IForgePlayer#getAttackRange` -> `#getEntityReach`
    * `ForgeMod#REACH_DISTANCE` -> `#BLOCK_REACH`
* `net.minecraftforge.common.extensions.IForgePlayer#getReachDistance` -> `#getBlockReach`
    * `ForgeMod#ATTACK_RANGE` -> `#ENTITY_REACH`
* `net.minecraftforge.common.extensions.IForgePlayer#canHit` and `#canInteractWith` -> `#canReach`
* `net.minecraftforge.event.entity.living.LivingSetAttackTargetEvent` -> `LivingChangeTargetEvent`
* `net.minecraftforge.client.model.generators.BlockModelBuilder#rootTransform` -> `#rootTransforms`
    * All root transformation logic has been moved to `ModelBuilder` and `TransformationHelper`
* `net.minecraftforge.common.extensions.IForgeItem#onUsingTick` -> `Item#onUseTick`
* `net.minecraftforge.event.level.SaplingGrowTreeEvent` -> `BlockGrowFeatureEvent`
* `net.minecraftforge.client.model.IQuadTransformer#empty`, `#applying`, and `#applyingLightmap` moved to `QuadTransformers`
* `net.minecraftforge.client.gui.ScreenUtils` has all been moved to extension methods on `GuiGraphics`
    * `#drawTexturedModalRect` and `#drawGradientRect` have been replaced with `#blit` and `#fillGradient`, respectively
