# Minecraft 1.21.11 -> 26.1 Mod Migration Primer

This is a high level, non-exhaustive overview on how to migrate your mod from 1.21.11 to 26.1. This does not look at any specific mod loader, just the changes to the vanilla classes.

This primer is licensed under the [Creative Commons Attribution 4.0 International](http://creativecommons.org/licenses/by/4.0/), so feel free to use it as a reference and leave a link so that other readers can consume the primer.

If there's any incorrect or missing information, please file an issue on this repository or ping @ChampionAsh5357 in the Neoforged Discord server.

## Pack Changes

There are a number of user-facing changes that are part of vanilla which are not discussed below that may be relevant to modders. You can find a list of them on [Misode's version changelog](https://misode.github.io/versions/?id=26.1&tab=changelog).

## Java 25 and Deobfuscation

26.1 introduces two new changes into the general pipeline.

First, the Java Development Kit has been upgraded from 21 to 25. Vanilla makes use of these new features, such as [JEP 447](https://openjdk.org/jeps/447), which allows statements before `super` within constructors. For users within the modding scene, please make sure to update accordingly, or take advantage of your IDE or build tool features. Microsoft's OpenJDK can be found [here](https://learn.microsoft.com/en-us/java/openjdk/download#openjdk-25).

Vanilla has also returned to being deobfuscated, meaning that all value types now have the official names provided by Mojang. There are still some things that are not captured due to the Java compilation process, such as inlining primitive and string constants, but the majority are now provided. This will only have a change for users or mod loaders who used a different value type mapping set from the official mappings.

## Loot Type Unrolling

Loot pool entires, item functions, item conditions, nbt providers, number providers, and score providers no longer use a wrapping object type to act as the registered instance. Now, the registries directly take in the `MapCodec` used for the serialization and deserialization process. As such, `Loot*Type` classes that held the codec have been removed. `getType` is now renamed to `codec`, taking in the registered `MapCodec`.

```java
// The following is an example with LootItemFunctions, but can roughly apply to the other instances as well

public record NoopItemFunction() implements LootItemFunction {
    public static final NoopItemFunction INSTANCE = new NoopItemFunction();
    // The map codec used as the registry object
    public static final MapCodec<NoopItemFunction> MAP_CODEC = MapCodec.unit(INSTANCE);

    // Replaces getType
    @Override
    public MapCodec<NoopItemFunction> codec() {
        // Return the registry object
        return MAP_CODEC;
    }
}

// Register the map codec to the appropriate registry
Registry.register(BuiltInRegistries.LOOT_FUNCTION_TYPE, Identifier.fromNamespaceAndPath("examplemod", "noop"), NoopItemFunction.MAP_CODEC);
```

- `net.minecraft.core.registries.BuiltInRegistries`, `Registries`
    - `LOOT_POOL_ENTRY_TYPE` now holds a `LootPoolEntryContainer` map codec instead of a `LootPoolEntryType`
    - `LOOT_FUNCTION_TYPE` now holds a `LootItemFunction` map codec instead of a `LootItemFunctionType`
    - `LOOT_CONDITION_TYPE` now holds a `LootItemCondition` map codec instead of a `LootItemConditionType`
    - `LOOT_NUMBER_PROVIDER_TYPE` now holds a `NumberProvider` map codec instead of a `LootNumberProviderType`
    - `LOOT_NBT_PROVIDER_TYPE` now holds a `NbtProvider` map codec instead of a `LootNbtProviderType`
    - `LOOT_SCORE_PROVIDER_TYPE` now holds a `ScoreboardNameProvider` map codec instead of a `LootScoreProviderType`
- `net.minecraft.world.level.storage.loot.entries` now renames `CODEC` fields to `MAP_CODEC`
    - `LootPoolEntries` instance fields have all been removed
        - The map codecs in each class should be used instead
        - `bootstrap` - Registers the loot pool entries.
    - `LootPoolEntryContainer#getType` -> `codec`, not one-to-one
    - `LootPoolEntryType` record is removed
- `net.minecraft.world.level.storage.loot.functions` now renames `CODEC` fields to `MAP_CODEC`
    - `LootItemFunction#getType` -> `codec`, not one-to-one
    - `LootItemFunctions` instance fields have all been removed
        - The map codecs in each class should be used instead
        - `bootstrap` - Registers the loot item functions.
    - `LootItemFunctionType` record is removed
- `net.minecraft.world.level.storage.loot.predicates` now renames `CODEC` fields to `MAP_CODEC`
    - `LootItemCondition#getType` -> `codec`, not one-to-one
    - `LootItemConditions` instance fields have all been removed
        - The map codecs in each class should be used instead
        - `bootstrap` - Registers the loot item conditions.
    - `LootItemConditionType` record is removed
- `net.minecraft.world.level.storage.loot.providers.nbt` now renames `CODEC` fields to `MAP_CODEC`
    - `LootNbtProviderType` record is removed
    - `NbtProvider` now implements `LootContextUser`
        - `getType` -> `codec`, not one-to-one
    - `NbtProviders` instance fields have all been removed
        - The map codecs in each class should be used instead
        - `bootstrap` - Registers the nbt providers.
    - `LootItemConditionType` record is removed
- `net.minecraft.world.level.storage.loot.providers.number` now renames `CODEC` fields to `MAP_CODEC`
    - `LootNumberProviderType` record is removed
    - `NumberProvider#getType` -> `codec`, not one-to-one
    - `NumberProviders` instance fields have all been removed
        - The map codecs in each class should be used instead
        - `bootstrap` - Registers the number providers.
- `net.minecraft.world.level.storage.loot.providers.score` now renames `CODEC` fields to `MAP_CODEC`
    - `LootScoreProviderType` record is removed
    - `ScoreProvider` now implements `LootContextUser`
        - `getType` -> `codec`, not one-to-one
    - `ScoreProviders` instance fields have all been removed
        - The map codecs in each class should be used instead
        - `bootstrap` - Registers the score providers.

## Validation Overhaul

The validation handler used to collect and then report problems with data-generated content has been overhauled. This is not, and has never been a replacement for field validation. The validation handler is specifically for more easily handling validation between multiple pieces of information that may not be exposed to a single field, such as whether a loot provider can be used for the given context params.

All validated objects implement either `Validatable`, or `CriterionTriggerInstance` specifically for advancement criteria. Both of these methods provide one method: `validate`, which is used to check the validity of an object. `validate` takes in a `ValidationContext`, which essentially holds the `ProblemReporter` used to collect issues, the current context params, and a reference resolver. `CriterionTriggerInstance` provides a `ValidationContextSource`, but this can be transformed to a `ValidationContext` using one of the `context` methods, providing the context params to check against. If the specific object cannot be validated, then `ValidationContext#reportProblem` is called, detailing the specific issue.

```java
// For some object that implements Validatable

@Override
public void validate(ValidationContext ctx) {
    // Check if a specific condition is validated.
    if (this.foo() != this.bar()) {
        // If not, report that there is an issue.
        ctx.reportProblem(() -> "'Foo' does not equal 'bar'.");
    }
}
```

If the object itself does not have any issues, but rather the specific fields, then the reporter can follow the stack trace while checking the individual elements using one of the `ValidationContext#for*` methods, performing something similar.

```java
// For some object that implements Validatable
// Let's assume it has a list of children objects.

@Override
public void validate(ValidationContext ctx) {
    for (int i = 0; i < this.children.size(); i++) {
        // Get specific context for child in list
        var childCtx = ctx.forIndexedField("children", i);
        // Check if a specific condition is validated.
        if (this.foo() != this.bar()) {
            // If not, report that there is an issue.
            childCtx.reportProblem(() -> "'Foo' does not equal 'bar'.");
        }
    }
}
```

`Validatable` also provides some static utilities for checking other `Validatable` fields.

```java
// For some object that implements Validatable
// Assume some child object also implements Validatable

@Override
public void validate(ValidationContext ctx) {
    Validatable.validate(ctx, "child", this.child);
}
```

After `Validatable` is implemented on all required objects, the validation can then be called depending on where it is used (typically after deserialization or before serialization). The call stack for these is typically the following:

```java
// For some Validatable validatable
// Let's assume we have access to the HolderGetter.Provider provider.
// The parameter itself is optional if not available.

// Create the problem collector and validation context.
// The context params should only include the ones that are being provided.
ProblemReporter reporter = new ProblemCollector.Collector();
ValidationContext ctx = new ValidationContext(reporter, LootContextParamSets.ALL_PARAMS, provider);

// Call the validator
validatable.validate(ctx);
```

This can also be appended to the end of codecs via `Codec#validate`:

```java
public record ExampleObject() implements Validatable {
    public static final Codec<ExampleObject> CODEC = MapCodec.unitCodec(
        ExampleObject::new
    ).validate(
        // Supply the validator along with the context params to check against.
        // This method does not have access to the registry provider.
        Validatable.validatorForContext(LootContextParamSets.ALL_PARAMS)
    );

    @Override
    public void validate(ValidationContext ctx) {
        // ...
    }
}
```

- `net.minecraft.advancements.CriterionTriggerInstance#validate` now takes in a `ValidationContextSource` instead of a `CriterionValidator`
- `net.minecraft.advancements.criterion`
    - `ContextAwarePredicate` now implements `Validatable`
    - `CriterionValidator` -> `ValidationContextSource` and `Validatable`
        - `Validatable` contains the `validate*` methods
        - `ValidationContextSource` holds the context and reporters
- `net.minecraft.world.item.enchantment`
    - `ConditionalEffect` now implements `Validatable`
        - `conditionCodec` is replaced by calling `validate` after load
    - `TargetedConditionalEffect` now implements `Validatable`
- `net.minecraft.world.level.storage.loot`
    - `IntRange` now implements `LootContextUser`
        - `getReferencedContextParams` replaced by `validate`
    - `LootContext$VisitedEntry` generic must now extend `Validatable`
    - `LootContextUser` now implements `Validatable`
    - `LootDataType` generic must now extend `Validatable`
        - The constructor now takes in a `$ContextGetter` instead of a `$Validator`
        - `runValidation` now takes in the `ValidationContextSource` instead of a `ValidationContext`
            - Also has an overload taking in a `HolderLookup` instead of a key-value pair
        - `createSimpleValidator`, `createLootTableValidator`, `$Validator` replaced by `Validatable`
        - `$ContextGetter` - Gets the `ContextKeySet` for some value.
    - `LootPool` now implements `Validatable`
    - `LootTable` now implements `Validatable`
    - `Validatable` - An interface which handles the validation of its instance within the given context.
    - `ValidationContext`
        - `forField` - Creates a context for a given field.
        - `forIndexedField` - Creates a context for a given entry in an list.
        - `forMapField` - Creates a context for a given key in a map.
        - `setContextKeySet` is removed
    - `ValidationContextSource` - The source for the defined context where the validation is taking place.
- `net.minecraft.world.level.storage.loot.entries.LootPoolEntryContainer` now implements `Validatable`
- `net.minecraft.world.level.storage.loot.functions`
    - `SetAttributesFunction$Modifier` now implements `LootContextUser`
    - `SetStewEffectFunction$EffectEntry` now implements `LootContextUser`

## Datapack Villager Trades

Villager trades has now become a data-generated registry from its original map-based setup. While basic impressions make it seem like that system is more limited, it is actually just as extensible, though in a rather convoluted way because the trades are functionally a loot table itself to determine what `MerchantOffer`s a trader can provide. For the purposes of understanding, this section will go over the basics of the trade rewrite, along with how each previous item listing can be converted to a `VillagerTrade`.

### Understanding the Trade Format

All trades are expressed as `VillagerTrade`s, which at its core determines what a trader `wants` and what it `gives` in return. Each trade can also provide modifiers to the item itself or its cost, or whether the specific trade can be made at all with the given condition. Each trade also specifies the number of trades that can be made, how much xp to give, or the price multiple that is multiplied with the user's reputation. A `VillagerTrade` is then turned into a `MerchantOffer` via `getOffer`, taking in the `LootContext`, typically with a context param of `LootContextParamSets#VILLAGER_TRADE`, providing the trader itself (`THIS_ENTITY`) and its position (`ORIGIN`).

The trades themselves are within `data/<namespace>/villager_trade/<path>`. Typically the path contains the profession and level the trade is for, such as `examplemod:example_profession/1/example_trade`

```json5
// For some villager trade 'examplemod:example_profession/1/example_trade'
// JSON at 'data/examplemod/villager_trade/example_profession/1/example_trade.json'
{
    // The stack the trader wants.
    "wants": {
        // The item of the stack.
        "id": "minecraft:apple",
        // A number provider used to determine how much
        // of the item that it wants. Once the count is
        // determined, any additional cost on the `gives`
        // stack (via `ADDITIONAL_TRADE_COST` component)
        // is added to the count before being clamped to
        // the max stack size.
        // If not specified, defaults to 1.
        "count": {
            "type": "minecraft:uniform",
            "min": 1,
            "max": 5
        },
        // Any components that stack should have. The stack
        // must have the exact components specified.
        // If not specified, then no components will be
        // checked, meaning that this is ignored.
        "components": {
            // Map of registry key to component value.
            "minecraft:custom_name": "Apple...?"
        }
    },
    // An additional stack the trader wants.
    // If not specified, the trader only checks `wants`.
    "additional_wants": {
        "id": "minecraft:emerald"
    },
    // The stack the trader gives in return.
    "gives": {
        // The item of the stack.
        "id": "minecraft:golden_apple",
        // A number [1, 99].
        // If not specified, defaults to 1.
        "count": 1,
        // The components to apply to the stack.
        // If not specified, just applies the default
        // item components.
        "components": {
            "minecraft:custom_name": "Not an Apple"
        }
    },
    // A number provider to determine how many times
    // the trade can be performed by the player before
    // the trader restocks. The value will always be at
    // least 1. 
    // If not specified, defaults to 4.
    "max_uses": {
        "type": "minecraft:uniform",
        "min": 1,
        "max": 20
    },
    // A number provider to determine the price multiplier
    // to apply to the cost of the item based on the player's
    // reputation with the trader and the item demand,
    // calculated from how many times the player has performed
    // the trade. This should generally be a small number as
    // the maximum reputation a user can have per trader is 150,
    // though it's more likely to have a reputation of 25 at most.
    // However, the trade must always have a minimum of one item,
    // even if the reputation discount multiplied with the reputation
    // indicates 100% or more off.
    // If not specified, defaults to 0.
    "reputation_discount": {
        "type": "minecraft:uniform",
        "min": 0,
        "max": 0.05
    },
    // A number provider to determine the amount of experience
    // the player obtains from performing the trade with the trader.
    // This is typically around 5-30 xp for vanilla trades.
    // If not specified, defaults to 1.
    "xp": {
        "type": "minecraft:uniform",
        "min": 10,
        "max": 20
    },
    // A loot item condition that determines whether the
    // trader can provide the trade to the player.
    // If not specified, defaults to always true.
    "merchant_predicate": {
        // This trade can only be performed by villagers
        // from a desert or snow village.
        "condition": "minecraft:entity_properties",
        "entity": "this",
        "predicate": {
            "predicates": {
                "minecraft:villager/variant": [
                    "minecraft:desert",
                    "minecraft:snow"
                ]
            }
        }
    },
    // A list of loot item function that modifies the item
    // offered from `gives` to the player.
    // If not specified, `gives` is not modifier.
    "given_item_modifiers": [
        {
            // Chooses a random enchantment from the provided tag
            "function": "minecraft:enchant_randomly",
            // If this is true, the trade cost increases the
            // number of `wants` items required.
            "include_additional_cost_component": true,
            "only_compatible": false,
            "options": "#minecraft:trades/desert_common"
        }
    ],
    // Can either be an enchantment id, such as "minecraft:protection",
    // or a list of enchantment ids, such as ["minecraft:protection", "minecraft:smite", ...],
    // or an enchantment tag, such as "#minecraft:trades/desert_common".
    // When provided, if the `gives` item after modifiers contains an
    // enchantment in this list, then the number of `wants` items required
    // is multiplied by 2.
    // If not specified, does nothing.
    "double_trade_price_enchantments": "#minecraft:trades/desert_common"
}
```

### The Trades of a Trader

Every trader can make many trades, typically choosing from a specified pool known as a `TradeSet`. Trade sets themselves are a separate datapack registry that consumes `VillagerTrade`s. Each set of trades determines how many trades can be offered and if the same trade can be chosen more than once. The trades that are offered are computed within `AbstractVillager#addOffersFromTradeSet`, first calling `TradeSet#calculateNumberOfTrades` to get the number of offers, and then using either `AbstractVillager#addOffersFromItemListings` or `AbstractVillager#addOffersFromItemListingsWithoutDuplicates` to choose the offers to use.

Note that if duplicate trades are allowed, there is a potential race condition where if all trades' `merchant_predicate`s fail`, then the offers will loop forever. This is because the method always assumes that there will be one trade that can be made.

The trade sets are within `data/<namespace>/trade_set/<path>`. Typically the path contains the profession and level the trade is for, such as `examplemod:example_profession/level_1.json`

```json5
// For some trade set 'examplemod:example_profession/level_1'
// JSON at 'data/examplemod/villager_trade/trade_set/level_1.json'
{
    // Can either be a villager trade id, such as "examplemod:example_profession/1/example_trade",
    // or a list of trade ids, such as ["examplemod:example_profession/1/example_trade", "minecraft:farmer/1/wheat_emerald", ...],
    // or an trade tag, such as "#examplemod:example_profession/level_1".
    // This is the set of trades that can be offered by the trader.
    // This should always be a villager trade tag so allow other users
    // to easily add their own trades to a trader.
    "trades": "#examplemod:example_profession/level_1",
    // A number provider that determines the number of offers that can be
    // made by the trader.
    "amount": {
        "type": "minecraft:uniform",
        "min": 1,
        "max": 5
    },
    // Whether the same trade can be used to make multiple offers.
    // If not specified, defaults to false.
    "allow_duplicates": true,
    // An identifier that determines the unique random instance to
    // user when determining the offers.
    // If not specified, uses the level random.
    "random_sequence": "examplemod:example_profession/level_1"
}
```

Where the villager trade tag could be:

```json5
// For some tag 'examplemod:example_profession/level_1'
// JSON at 'data/examplemod/tags/villager_trade/example_profession/level_1.json'
{
    "values": [
        "examplemod:example_profession/1/example_trade"
    ]
}
```

This also means trades can be easily added to existing trade sets by adding to the associated tag:

```json5
// For some tag 'minecraft:farmer/level_1'
// JSON at 'data/minecraft/tags/villager_trade/farmer/level_1.json'
{
    "values": [
        "examplemod:example_profession/1/example_trade"
    ]
}
```

Meanwhile, adding to a new `VillagerProfession` is done by mapping the level int to the trade set key in `tradeSetsByLevel`:

```java
public static final VillagerProfession EXAMPLE = Registry.register(
    BuiltInRegistries.VILLAGER_PROFESSION,
    Identifier.fromNamespaceAndPath("examplemod", "example_profession"),
    new VillagerProfession(
        Component.literal(""),
        p -> true,
        p -> true,
        ImmutableSet.of(),
        ImmutableSet.of(),
        null,
        // A map of profession level to trade set keys
        Int2ObjectMap.ofEntries(
            Int2ObjectMap.entry(
                // The profession level
                1,
                // The trade set id
                ResourceKey.create(Registries.TRADE_SET, Identifier.fromNamespaceAndPath("examplemod", "example_profession/level_1"))
            )
        )
    )
);
```

### Item Listing Conversions

With all this in mind, we can now convert item listings to their new data-generated villager trades.

#### Emeralds <-> Items

For the following trades:

```java
public static final VillagerTrades.ItemListing ITEM_TO_EMERALD = new VillagerTrades.EmeraldForItems(
    // The item the trader wants.
    Items.WHEAT,
    // The number of items the trader wants.
    20,
    // The maximum number of times the trade can be made
    // before restock.
    16,
    // The amount of experience given for the trade.
    2,
    // The number of emeralds given in return.
    1
);

public static final VillagerTrades.ItemListing EMERALD_TO_ITEM = new VillagerTrades.ItemsForEmeralds(
    // The item the trader will give in return.
    Items.BREAD,
    // The number of emeralds the trader wants.
    1,
    // The number of items the trader will give.
    6,
    // The maximum number of times the trade can be made
    // before restock.
    16,
    // The amount of experience given for the trade.
    1,
    // The price multiplier to apply to the offer, given
    // reputation and demand.
    0.05f
);
```

Their equivalent would be:

```json5
// For some villager trade 'examplemod:item_to_emerald'
// JSON at 'data/examplemod/villager_trade/item_to_emerald.json'
{
    "gives": {
        // The number of emeralds given in return.
        "count": 1,
        "id": "minecraft:emerald"
    },
    // The maximum number of times the trade can be made
    // before restock.
    "max_uses": 16,
    // The price multiplier to apply to the offer, given
    // reputation and demand.
    // `EmeraldForItems` hardcoded this to 0.05
    "reputation_discount": 0.05,
    "wants": {
        // The number of items the trader wants.
        "count": 20,
        // The item the trader wants.
        "id": "minecraft:wheat"
    },
    // The amount of experience given for the trade.
    "xp": 2
}


// For some villager trade 'examplemod:emerald_to_item'
// JSON at 'data/examplemod/villager_trade/emerald_to_item.json'
{
    "gives": {
        // The number of items the trader will give.
        "count": 6,
        // The item the trader will give in return.
        "id": "minecraft:bread"
    },
    // The maximum number of times the trade can be made
    // before restock.
    "max_uses": 16,
    // The price multiplier to apply to the offer, given
    // reputation and demand.
    "reputation_discount": 0.05,
    "wants": {
        "id": "minecraft:emerald",
        // The number of emeralds the trader wants.
        "count": 1
    },
    // The amount of experience given for the trade.
    "xp": 1
}
```

#### Items and Emeralds -> Items

For the following trade:

```java
public static final VillagerTrades.ItemListing ITEM_EMERALD_TO_ITEM = new VillagerTrades.ItemsAndEmeraldsToItems(
    // The item the trader wants.
    Items.COD,
    // The number of items the trader wants. 
    6,
    // The number of emeralds the trader additionally wants.
    1,
    // The item the trader will give in return.
    Items.COOKED_COD,
    // The number of items the trader will give.
    6,
    // The maximum number of times the trade can be made
    // before restock.
    16,
    // The amount of experience given for the trade.
    1,
    // The price multiplier to apply to the offer, given
    // reputation and demand.
    0.05f
);
```

The equivalent would be:

```json5
// For some villager trade 'examplemod:item_emerald_to_item'
// JSON at 'data/examplemod/villager_trade/item_emerald_to_item.json'
{
    // The emeralds the trader additionally wants.
    "additional_wants": {
        "id": "minecraft:emerald",
    },
    "gives": {
        // The number of items the trader will give.
        "count": 6,
        // The item the trader will give in return.
        "id": "minecraft:cooked_cod"
    },
    // The maximum number of times the trade can be made
    // before restock.
    "max_uses": 16,
    // The price multiplier to apply to the offer, given
    // reputation and demand.
    "reputation_discount": 0.05,
    "wants": {
        // The number of items the trader wants. 
        "count": 6,
        // The item the trader wants.
        "id": "minecraft:cod"
    },
    // The amount of experience given for the trade.
    "xp": 1
}
```

#### Emeralds -> Dyed Armor

For the following trade:

```java
public static final VillagerTrades.ItemListing EMERALD_TO_DYED_ARMOR = new VillagerTrades.DyedArmorForEmeralds(
    // The item the trader will give in return and dye.
    Items.LEATHER_HELMET,
    // The number of emeralds the trader wants.
    5,
    // The maximum number of times the trade can be made
    // before restock.
    12,
    // The amount of experience given for the trade.
    5
);
```

The equivalent would be:

```json5
// For some villager trade 'examplemod:emerald_to_dyed_armor'
// JSON at 'data/examplemod/villager_trade/emerald_to_dyed_armor.json'
{
    "given_item_modifiers": [
        {
            // Sets the random dye(s) on the armor.
            "function": "minecraft:set_random_dyes",
            "number_of_dyes": {
                "type": "minecraft:sum",
                "summands": [
                    1.0,
                    {
                        "type": "minecraft:binomial",
                        "n": 2.0,
                        "p": 0.75
                    }
                ]
            }
        },
        {
            // Checks that the dye was successfully applied
            // to the item.
            "function": "minecraft:filtered",
            "item_filter": {
                "items": "minecraft:leather_helmet",
                "predicates": {
                    "minecraft:dyed_color": {}
                }
            },
            // If it fails, discards the offer.
            "on_fail": {
                "function": "minecraft:discard"
            }
        }
    ],
    "gives": {
        "count": 1,
        // The item the trader will give in return and dye.
        "id": "minecraft:leather_helmet"
    },
    // The maximum number of times the trade can be made
    // before restock.
    "max_uses": 12,
    // The price multiplier to apply to the offer, given
    // reputation and demand.
    "reputation_discount": 0.05,
    "wants": {
        // The number of emeralds the trader wants.
        "count": 5,
        "id": "minecraft:emerald"
    },
    // The amount of experience given for the trade.
    "xp": 5
}
```

#### Emeralds -> Enchanted Item

For the following trade:

```java
public static final VillagerTrades.ItemListing EMERALD_TO_ENCHANTED_BOOK = new VillagerTrades.EnchantBookForEmeralds(
    // The amount of experience given for the trade.
    30,
    // The minimum level used when selecting the stored enchantments on the book.
    3,
    // The maximum level used when selecting the stored enchantments on the book.
    3,
    // A tag containing the list of available enchantments to select from for the book.
    EnchantmentTags.TRADES_DESERT_SPECIAL
);

public static final VillagerTrades.ItemListing EMERALD_TO_ENCHANTED_ITEM = new VillagerTrades.EnchantedItemForEmeralds(
    // The item the trader will give in return and try to enchant.
    Items.FISHING_ROD,
    // The base number of emeralds the trader wants.
    3,
    // The maximum number of times the trade can be made
    // before restock.
    3,
    // The amount of experience given for the trade.
    10,
    // The price multiplier to apply to the offer, given
    // reputation and demand.
    0.2f
);
```

The equivalent would be:

```json5
// For some villager trade 'examplemod:emerald_to_enchanted_book'
// JSON at 'data/examplemod/villager_trade/emerald_to_enchanted_book.json'
{
    // The trader expects a book to write the enchantment to.
    "additional_wants": {
        "id": "minecraft:book"
    },
    // Trade cost for emeralds increased when in the double trade price
    // tag.
    "double_trade_price_enchantments": "#minecraft:double_trade_price",
    "given_item_modifiers": [
        {
            "function": "minecraft:enchant_with_levels",
            "include_additional_cost_component": true,
            "levels": {
                "type": "minecraft:uniform",
                // The minimum level used when selecting the stored enchantments on the book.
                "min": 3,
                // The maximum level used when selecting the stored enchantments on the book.
                "max": 3
            },
            // The list of available enchantments to select from for the book.
            "options": [
                "minecraft:efficiency"
            ]
        },
        {
            // Make sure the enchantment was added successfully wih the given level.
            "function": "minecraft:filtered",
            "item_filter": {
                "items": "minecraft:enchanted_book",
                "predicates": {
                    "minecraft:stored_enchantments": [
                        {
                            "levels": {
                                // The minimum level used when selecting the stored enchantments on the book.
                                "min": 3,
                                // The maximum level used when selecting the stored enchantments on the book.
                                "max": 3
                            }
                        }
                    ]
                }
            },
            // Discard on failure
            "on_fail": {
                "function": "minecraft:discard"
            }
        }
    ],
    // The trader gives the enchanted book.
    "gives": {
        "count": 1,
        "id": "minecraft:enchanted_book"
    },
    // The maximum number of times the trade can be made
    // before restock was hardcoded to 12.
    "max_uses": 12,
    // The price multiplier to apply to the offer, given
    // reputation and demand, hardcoded to 0.2.
    "reputation_discount": 0.2,
    "wants": {
        "count": {
            "type": "minecraft:sum",
            "summands": [
                // A hardcoded computation based on the min and max
                // level of the enchantment.
                11.0,
                {
                    "type": "minecraft:uniform",
                    "max": 35.0,
                    "min": 0.0
                }
            ]
        },
        "id": "minecraft:emerald"
    },
    // The amount of experience given for the trade.
    "xp": 30
}

// For some villager trade 'examplemod:emerald_to_enchanted_item'
// JSON at 'data/examplemod/villager_trade/emerald_to_enchanted_item.json'
{
    "given_item_modifiers": [
        {
            // Applies the enchantment to the given equipment.
            "function": "minecraft:enchant_with_levels",
            "include_additional_cost_component": true,
            "levels": {
                "type": "minecraft:uniform",
                "max": 20,
                "min": 5
            },
            "options": "#minecraft:on_traded_equipment"
        },
        {
            // Checks to make sure the enchantment was applied.
            "function": "minecraft:filtered",
            "item_filter": {
                "items": "minecraft:fishing_rod",
                "predicates": {
                    "minecraft:enchantments": [
                        {}
                    ]
                }
            },
            // On fail, give nothing.
            "on_fail": {
                "function": "minecraft:discard"
            }
        }
    ],
    "gives": {
        "count": 1,
        // The item the trader will give in return and try to enchant.
        "id": "minecraft:fishing_rod"
    },
    // The maximum number of times the trade can be made
    // before restock.
    "max_uses": 3,
    // The price multiplier to apply to the offer, given
    // reputation and demand.
    "reputation_discount": 0.2,
    "wants": {
        "count": {
            "type": "minecraft:sum",
            "summands": [
                // The base number of emeralds the trader wants.
                3,
                {
                    // The variation based on the enchantment level.
                    // Originally, this would be the value used in the
                    // item function, but since they are now isolated,
                    // these values could differ.
                    "type": "minecraft:uniform",
                    "max": 20,
                    "min": 5
                }
            ]
        },
        "id": "minecraft:emerald"
    },
    // The amount of experience given for the trade.
    "xp": 10
}
```

#### Items and Emeralds -> Potion Effect Item

For the following trade:

```java
public static final VillagerTrades.ItemListing EMERALD_TO_SUSPICIOUS_STEW = new VillagerTrades.SuspiciousStewForEmerald(
    // The effect applied by the suspicious stew.
    MobEffects.NIGHT_VISION,
    // The number of ticks the effect should be active for.
    100,
    // The amount of experience given for the trade.
    15
);

public static final VillagerTrades.ItemListing ITEM_EMERALD_TO_TIPPED_ARROW = new VillagerTrades.TippedArrowForItemsAndEmeralds(
    // The item the trader additionally wants.
    Items.ARROW,
    // The number of items the trader additionally wants.
    5,
    // The item the trader will give in return.
    Items.TIPPED_ARROW,
    // The number of items the trader will give.
    5,
    // The number of emeralds the trader wants.
    2,
    // The maximum number of times the trade can be made
    // before restock.
    12,
    // The amount of experience given for the trade.
    30
);
```

The equivalent would be:

```json5
// For some villager trade 'examplemod:emerald_to_suspicious_stew'
// JSON at 'data/examplemod/villager_trade/emerald_to_suspicious_stew.json'
{
    "given_item_modifiers": [
        {
            "effects": [
                {
                    // The effect applied by the suspicious stew.
                    "type": "minecraft:night_vision",
                    // The number of ticks the effect should be active for.
                    "duration": 100
                }
                // Vanilla merges all suspicious stew offers
                // into one since this function picks one
                // stew effect at random.
            ],
            "function": "minecraft:set_stew_effect"
        }
    ],
    "gives": {
        "count": 1,
        "id": "minecraft:suspicious_stew"
    },
    // The maximum number of times the trade can be made
    // before restock, hardcoded to 12.
    "max_uses": 12,
    // The price multiplier to apply to the offer, given
    // reputation and demand, hardcoded to 0.05.
    "reputation_discount": 0.05,
    "wants": {
        "id": "minecraft:emerald"
    },
    // The amount of experience given for the trade.
    "xp": 15
}

// For some villager trade 'examplemod:item_emerald_to_tipped_arrow'
// JSON at 'data/examplemod/villager_trade/item_emerald_to_tipped_arrow.json'
{
    "additional_wants": {
        // The number of items the trader additionally wants.
        "count": 5,
        // The item the trader additionally wants.
        "id": "minecraft:arrow"
    },
    "given_item_modifiers": [
        {
            // Applies a random potion effect from the tradable potions.
            // Original implementation just picked any potion at random.
            "function": "minecraft:set_random_potion",
            "options": "#minecraft:tradeable"
        }
    ],
    "gives": {
        // The number of items the trader will give.
        "count": 5,
        // The item the trader will give in return.
        "id": "minecraft:tipped_arrow"
    },
    // The maximum number of times the trade can be made
    // before restock.
    "max_uses": 12,
    // The price multiplier to apply to the offer, given
    // reputation and demand, hardcoded to 0.05.
    "reputation_discount": 0.05,
    "wants": {
        // The number of emeralds the trader wants.
        "count": 2,
        "id": "minecraft:emerald"
    },
    // The amount of experience given for the trade.
    "xp": 30
}
```

#### Emeralds -> Treasure Map

For the following trade:

```java
public static final VillagerTrades.ItemListing EMERALD_TO_TREASURE_MAP = new VillagerTrades.TreasureMapForEmeralds(
    // The number of emeralds the trader wants.
    8,
    // A tag containing a list of treasure structures to find the nearest of.
    StructureTags.ON_TAIGA_VILLAGE_MAPS,
    // The translation key of the map name.
    "filled_map.village_taiga",
    // The icon used to decorate the treasure location found on the map.
    MapDecorationTypes.TAIGA_VILLAGE,
    // The maximum number of times the trade can be made
    // before restock.
    12,
    // The amount of experience given for the trade.
    5
);
```

The equivalent would be:

```json5
// For some villager trade 'examplemod:emerald_to_treasure_map'
// JSON at 'data/examplemod/villager_trade/emerald_to_treasure_map.json'
{
    "additional_wants": {
        // An item the trader additionally wants, hardcoded to compass.
        "id": "minecraft:compass"
    },
    "given_item_modifiers": [
        {
            // Finds a treasure structure to display on the map.

            // The icon used to decorate the treasure location found on the map.
            "decoration": "minecraft:village_taiga",
            // A tag containing a list of treasure structures to find the nearest of.
            "destination": "minecraft:on_taiga_village_maps",
            "function": "minecraft:exploration_map",
            "search_radius": 100
        },
        {
            // Sets the name of the map.

            "function": "minecraft:set_name",
            "name": {
                // The translation key of the map name.
                "translate": "filled_map.village_taiga"
            },
            "target": "item_name"
        },
        {
            // Check to make sure a structure was found.

            "function": "minecraft:filtered",
            "item_filter": {
                "items": "minecraft:filled_map",
                "predicates": {
                    "minecraft:map_id": {}
                }
            },
            "on_fail": {
                "function": "minecraft:discard"
            }
        }
    ],
    "gives": {
        "count": 1,
        // Returns a map filled with the treasure location.
        "id": "minecraft:map"
    },
    // The maximum number of times the trade can be made
    // before restock.
    "max_uses": 12,
    // The price multiplier to apply to the offer, given
    // reputation and demand, hardcoded to 0.05.
    "reputation_discount": 0.05,
    "wants": {
        // The number of emeralds the trader wants.
        "count": 8,
        "id": "minecraft:emerald"
    },
    // The amount of experience given for the trade.
    "xp": 5
}
```

#### Villager Variants

Some traders would provide different options depending on the villager type it was as one giant map of item listings. Now, each type has its own individual villager trade, using a `merchant_predicate` to check whether the offer can be made to the specific villager type:

```json5
// For some villager trade 'examplemod:villager_type_item'
// JSON at 'data/examplemod/villager_trade/villager_type_item.json'
{
    // ...
    "merchant_predicate": {
        // Check the entity.
        "condition": "minecraft:entity_properties",
        "entity": "this",
        "predicate": {
            "predicates": {
                // Villager type must be desert for this trade to be chosen.
                "minecraft:villager/variant": "minecraft:desert"
            }
        }
    },
    // ...
}
```

- `net.minecraft.core.registries.Registries`
    - `TRADE_SET` - A key to the registry holding a list of trades.
    - `VILLAGER_TRADE` - A key to the registry holding a single trade.
- `net.minecraft.tags.VillagerTradeTags` - Tags for villager trades.
- `net.minecraft.world.entity.npc.villager`
    - `AbstractVillager#addOffersFromItemListings` -> `addOffersFromTradeSet`, taking in a key for the trade set rather than the `$ItemListing`s and number of offers; not one-to-one
    - `VillagerProfession` now takes in a map of trader level to `TradeSet`
        - `getTrades` - Returns the trades set key for the given level.
    - `VillagerTrades` has been split into many different classes and implementations
        - `TRADES`, `EXPERIMENTAL_TRADES`, `WANDERING_TRADER_TRADES` is now represented by the `VillagerProfession#tradeSetsByLevel`
            - As they are now datapack entries, they are stored in their respective datapacks
            - `TRADES`, `EXPERIMENTAL_TRADES` are represented by `data/minecraft/trade_set/<profession>/*`
            - `WANDERING_TRADER_TRADES` are represented by `data/minecraft/trade_set/wandering_trader/*`
        - `$DyedArmorForEmeralds` -> `VillagerTrades#dyedItem`, `addRandomDye`; not one-to-one
        - `$EmeraldForItems` -> `VillagerTrade`, not one-to-one
        - `$EmeraldsForVillagerTypeItem` -> `VillagerTrades#registerBoatTrades`, see usage, not one-to-one
        - `$EnchantBookForEmeralds` -> `VillagerTrades#enchantedBook`, not one-to-one
        - `$EnchantedItemForEmeralds` -> `VillagerTrades#enchantedItem`, not one-to-one
        - `$ItemListing` -> `VillagerTrade`
        - `$ItemsAndEmeraldsToItems` -> `VillagerTrade` with `additionalWants`
        - `$ItemsForEmeralds` -> `VillagerTrade`, not one-to-one
        - `$SuspiciousStewForEmerald` -> `VillagerTrade` with `SetStewEffectFunction`
        - `$TippedArrowForItemsAndEmeralds` -> `VillagerTrade` with `SetRandomPotionFunction`
        - `$TreasureMapForEmeralds` -> `VillagerTrades$VillagerExplorerMapEntry`, see usage, not one-to-one
        - `$TypeSpecificTrade` -> `VillagerTrades#villagerTypeRestriction`, `villagerTypeHolderSet`; not one-to-one
- `net.minecraft.world.item.enchantment.providers.TradeRebalanceEnchantmentProviders` interface is removed
    - Replaced by `TradeRebalanceVillagerTrades`, `TradeRebalanceRegistries`
- `net.minecraft.world.item.trading`
    - `TradeCost` - An `ItemStack` that is being traded to some trader.
    - `TradeRebalanceVillagerTrades` - All trades part of the trade rebalance datapack.
    - `TradeSet` - A set of trades that can be performed by a trader at some level.
    - `TradeSets` - All vanilla trades sets for some trader at some level.
    - `VillagerTrade` - A trade between the trader and the player.
    - `VillagerTrades` - All vanilla trades.
- `net.minecraft.world.level.storage.loot.parameters.LootContextParamSets#VILLAGER_TRADE` - A loot context when a villager trade is taking place, containing the trade origin and the trading entity.

## `Level#random` field now protected

The `Level#random` field is now `protected` instead of `public`. As such, uses should transition to using the public `getRandom` method.

```java
// For some Level level
RandomSource random = level.getRandom();
```

- `net.minecraft.world.level.Level#random` field is now `protected` instead of `public`
    - Use `getRandom` method instead

## Minor Migrations

The following is a list of useful or interesting additions, changes, and removals that do not deserve their own section in the primer.

### Container Screen Changes

The usage of `AbstractContainerScreen`s has changed slightly, requiring some minor changes. First `imageWidth` and `imageHeight` are now final, settable as parameters within the constructor. If these two are not specified, they default to the original 176 x 166 background image.

```java
// Assume some AbstractContainerMenu subclass exists
public class ExampleContainerScreen extends AbstractContainerScreen<ExampleContainerMenu> {

    // Constructor
    public ExampleContainerScreen(ExampleContainerMenu meu, Inventory playerInventory, Component title) {
        // Specify image width and height as the last two parameters in the constructor
        super(menu, playerInventory, title, 256, 256);
    }
}
```

Additionally, the `AbstractContainerScreen#render` override now calls `renderTooltip` at the end of the call stack. This means that, in most cases, you should not override `render` in subtypes of the `AbstractContainerScreen`. Everything can be done in one of the other methods provided by the class.

- `net.minecraft.client.gui.screens.inventory.AbstractContainerScreen` now optionally takes in the background image width and height
    - `imageWidth`, `imageHeight` are now final
    - `DEFAULT_IMAGE_WIDTH`, `DEFAULT_IMAGE_HEIGHT` - The default width and height of the container background image.
    - `slotClicked` now takes is a `ContainerInput` instead of a `ClickType`
    - `render` override now calls `renderTooltip` by default
- `net.minecraft.world.inventory`
    - `AbstractContainerMenu#clicked` now takes in a `ContainerInput` instead of a `ClickType`
    - `ClickType` -> `ContainerInput`

### New Tag Providers

A new `TagsProvider` has been added that provides a utility for working with `Holder$Reference`s, known as `HolderTagProvider`. This is only used by the `PotionTagsProvider`.

Additionally, the `TagBuilder` now provides a method of setting the `replace` field on a tag, which removes all previous read entries during deserialization.

- `net.minecraft.data.tags`
    - `HolderTagProvider` - A tag provider with a utility for appending tags by their reference holder.
    - `KeyTagProvider#tag` now has an overload of whether to replace the entries in the tag.
    - `PotionTagsProvider` - A provider for potion tags.
    - `TradeRebalanceTradeTagsProvider` - A provider for villager trade tags for the trade rebalance.
    - `VillagerTradesTagsProvider` - A provider for villager trade tags.
- `net.minecraft.tags.TagBuilder#shouldReplace`, `setReplace` - Handles the `replace` field that removes all previously read entries during deserialization.

### Test Environment State Tracking

`TestEnvironmentDefinition`s can now keep track of the original state of the world when being created, such that it can be properly restored on run. This is done through a generic known as the 'SavedDataType'. On `setup`, each environment will return the generic data representing the original state of what was modified. Then, on `teardown`, the original state will be restored to the level for the next test case.

```java
// The generic should represent the original data stored on the level
public record RespawnEnvironment(LevelData.RespawnData respawn) implements TestEnvironmentDefinition<LevelData.RespawnData> {

    @Override
    public LevelData.RespawnData setup(ServerLevel level) {
        // Modify the level while logging the original state.
        var original = level.getRespawnData();
        level.setRespawnData(this.respawn);

        // Return the original state.
        return original;
    }

    @Override
    public void teardown(ServerLevel level, LevelData.RespawnData original) {
        // Reset the state of the level to the original values.
        level.setRespawnData(original);
    }
    
    @Override
    public MapCodec<RespawnEnvironment> codec() {
        // Return the registered MapCodec here.
        // ...
    }
}
```

- `net.minecraft.gametest.framework.TestEnvironmentDefinition` now has a generic representing the original state of the given modification performed by the test environment
    - `setup` now returns the generic representing the original state
    - `teardown` is no longer default, taking in the original state to restore
    - `activate`, `$Activation` - Handles an active test environment. 

### Typed Instance

`TypedInstance` is an interface that is attached to certain objects that represent an instance of some other 'type' object. For example, and `Entity` is a typed instance of `EntityType`, or a `BlockState` is a typed instance of `Block`. This interface is meant as a standard method to provide access to the type `Holder` and check whether the backing type, and therefore the instance, is equivalent to some identifier, tag, or raw object via `is`.

```java
// For some Entity entity, check the EntityType
entity.is(EntityType.PLAYER);

// For some ItemStack itemStack, check the Item
itemStack.is(ItemTags.BUTTONS);

// For some BlockState blockState, check the Block
blockState.is(Blocks.DIRT);

// For some FluidState fluidState, check the Fluid
fluidState.is(FluidTags.WATER);
```

- `net.minecraft.core.TypedInstance` - An interface that represents that this object is an instance for some other 'type' object.
- `net.minecraft.world.entity`
    - `Entity` now implements `TypedInstance<EntityType<?>>`
    - `EntityType#is` -> `TypedInstance#is`
        - Now on the `Entity` instance
- `net.minecraft.world.item.ItemStack` now implements `TypedInstance<Item>`
    - `getItemHolder` -> `typeHolder`
    - `getTags` -> `tags`
- `net.minecraft.world.level.block.state.BlockBehaviour$BlockStateBase` now implements `TypedInstance<Block>`
    - `getBlockHolder` -> `typeHolder`
    - `getTags` -> `tags`
- `net.minecraft.world.level.material.FluidState` now implements `TypedInstance<Fluid>`
    - `holder` -> `typeHolder`
    - `getTags` -> `tags`

### Entity Textures and Adult/Baby Models

Entity textures within `assets/minecraft/textures/entity/*` have now been sorted into subdirectories (e.g., `entity/panda` for panda textures, or `entity/pig` for pig textures). Most textures have been named with the entity type starting followed by an underscore along with its variant (e.g., `arrow_tipped` for tipped arrow, `pig_cold` for the cold pig variant, or `panda_brown` for the brown panda variant).

Additionally, some animal models have been split into separate classes for the baby and adult variant. These models either directly extend an abstract model implementation (e.g., `AbstractFelineModel`) or the original model class (e.g., `PigModel`).

- `net.minecraft.client.model.QuadrupedModel` now has a constructor that takes in a function for the `RenderType`
- `net.minecraft.client.model.animal.chicken`
    - `AdultChickenModel` - Entity model for the adult chicken.
    - `BabyChickenModel` - Entity model for the baby chicken.
    - `ChickenModel` is now abstract
        - `RED_THING` -> `AdultChickenModel#RED_THING`
        - `BABY_TRANSFORMER` has been directly merged into the layer definition for the `BabyChickenModel`
        - `createBodyLayer` -> `AdultChickenModel#REDcreateBodyLayer_THING`
        - `createBaseChickenModel` -> `AdultChickenModel#createBaseChickenModel`
    - `ColdChickenModel` now extends `AdultChickenModel`
- `net.minecraft.client.model.animal.cow.BabyCowModel` - Entity model for the baby cow.
- `net.minecraft.client.model.animal.feline`
    - `CatModel` -> `AdultCatModel`, `BabyCatModel`; not one-to-one
    - `FelineModel` -> `AbstractFelineModel`, not one-to-one
        - Implementations in `AdultFelineModel` and `BabyFelineModel`
    - `OcelotModel` -> `AdultOcelotModel`, `BabyOcelotModel`; not one-to-one
- `net.minecraft.client.model.animal.pig.BabyPigModel` - Entity model for the baby pig.
- `net.minecraft.client.model.animal.rabbit`
    - `AdultRabbitModel` - Entity model for the adult rabbit.
    - `BabyRabbitModel` - Entity model for the baby rabbit.
    - `RabbitModel` is now abstract
        - The constructor now takes in two animation definitions for the hop and idle head tilt
        - `FRONT_LEGS`, `BACK_LEGS` - The child name for the entity legs.
        - `LEFT_HAUNCH`, `RIGHT_HAUNCH` are now `protected`
        - `createBodyLayer` -> `AdultRabbitModel#createBodyLayer`, `BabyRabbitModel#createBodyLayer`; not one-to-one
- `net.minecraft.client.model.animal.sheep`
    - `BabySheepModel` - Entity model for the baby sheep.
    - `SheepModel#BABY_TRANSFORMER` has been directly merged into the layer definition for the `BabySheepModel`
- `net.minecraft.client.model.animal.wolf`
    - `AdultWolfModel` - Entity model for the adult wolf.
    - `BabyWolfModel` - Entity model for the baby wolf.
    - `WolfModel` is now abstract
        - `ModelPart` fields are now all protected
        - `createMeshDefinition` -> `AdultWolfModel#createBodyLayer`, `BabyWolfModel#createBodyLayer`; not one-to-one
        - `shakeOffWater` - Sets the body rotation when shaking off water.
        - `setSittingPose` - Sets the sitting pose of the wolf.
- `net.minecraft.client.model.geom.ModelLayers`
    - `COLD_CHICKEN_BABY` is removed
    - `COLD_PIG_BABY` is removed
    - `PIG_BABY_SADDLE` is removed
    - `SHEEP_BABY_WOOL_UNDERCOAT` is removed
    - `WOLF_BABY_ARMOR` is removed
- `net.minecraft.client.renderer.entity`
    - `CatRenderer` now takes in an `AbstractFelineModel` for its generic
    - `OcelotRenderer` now takes in an `AbstractFelineModel` for its generic
- `net.minecraft.client.renderer.entity.layers.CatCollarLayer` now takes in an `AbstractFelineModel` for its generic
- `net.minecraft.client.renderer.entity.state.RabbitRenderState`
    - `hopAnimationState` - The state of the hop the entity is performing.
    - `idleHeadTiltAnimationState` - The state of the head tilt when performing the idle animation.
- `net.minecraft.world.entity.animal.chicken.ChickenVariant` now takes in a resource for the baby texture
- `net.minecraft.world.entity.animal.cow.CowVariant` now takes in a resource for the baby texture
- `net.minecraft.world.entity.animal.cat.CatVariant` now takes in a resource for the baby texture
    - `CatVariant#assetInfo` - Gets the entity texture based on whether the entity is a baby.
- `net.minecraft.world.entity.animal.pig.PigVariant` now takes in a resource for the baby texture
- `net.minecraft.world.entity.animal.rabbit.Rabbit`
    - `hopAnimationState` - The state of the hop the entity is performing.
    - `idleHeadTiltAnimationState` - The state of the head tilt when performing the idle animation.
- `net.minecraft.world.entity.animal.wolf`
    - `WolfSoundVariant` -> `WolfSoundVariant$WolfSoundSet`
        - The class itself now holds two sound sets for the adult wolf and baby wolf.
    - `WolfSoundVariants$SoundSet#getSoundEventSuffix` -> `getSoundEventIdentifier`
- `net.minecraft.world.entity.animal.wolf.WolfVariant` now takes in an asset info for the baby variant

### ChunkPos, now a record

`ChunkPos` is now a record. The `BlockPos` constructor has been replaced by `ChunkPos#containing` while the packed `long` constructor has been replaced by `ChunkPose#unpack`.

- `net.minecraft.world.level.ChunkPos` is now a record
    - `ChunkPos(BlockPos)` -> `containing`
    - `ChunkPos(long)` -> `unpack`
    - `toLong`, `asLong` -> `pack`

### Environment Attribute Additions

- `visual/block_light_tint` - Tints the color of the light emitted by a block.
- `visual/night_vision_color` - The color when night vision is active.
- `visual/ambient_light_color` - The color of the ambient light in an environment.

### Tag Changes

- `minecraft:enchantment`
    - `trades/desert_special` is removed
    - `trades/jungle_special` is removed
    - `trades/plains_special` is removed
    - `trades/savanna_special` is removed
    - `trades/snow_special` is removed
    - `trades/swamp_special` is removed
    - `trades/taiga_special` is removed
- `minecraft:item`
    - `metal_nuggets`
- `minecraft:potion`
    - `tradable`

### List of Additions

- `assets/minecraft/shaders/include/smooth_lighting.glsl` - A utility for smooth lightmap sampling.
- `com.mojang.blaze3d`
    - `GLFWErrorCapture` - Captures errors during a GL process. 
    - `GLFWErrorScope` - A closable that defines the scope of the GL errors to capture.
- `com.mojang.blaze3d.opengl.GlBackend` - A GPU backend for OpenGL.
- `com.mojang.blaze3d.platform.NativeImage#isClosed` - Whether the image is closed or deallocated.
- `com.mojang.blaze3d.shaders.GpuDebugOptions` - The debug options for the GPU pipeline.
- `com.mojang.blaze3d.systems`
    - `BackendCreationException` - An exception throw when the GPU backend couldn't be created.
    - `GpuBackend` - An interface responsible for creating the used GPU device and window to display to.
    - `GpuDevice`
        - `setVsync` - Sets whether VSync is enabled.
        - `presentFrame` - Swaps the front and back buffers of the window to display the present frame.
        - `isZZeroToOne` - Whether the 0 to 1 Z range is used instead of -1 to 1.
    - `WindowAndDevice` - A record containing the window handle and the `GpuDevice`
- `net.minecraft.advancements.criterion.FoodPredicate` - A criterion predicate that can check the food level and saturation.
- `net.minecraft.client`
    - `Minecraft#sendLowDiskSpaceWarning` - Sends a system toast for low disk space.
    - `Options#keyDebugLightmapTexture` - A key mapping to show the lightmap texture.
- `net.minecraft.client.animation.definitions`
    - `BabyRabbitAnimation` - Animations for the baby rabbit.
    - `RabbitAnimation` - Animations for the rabbit.
- `net.minecraft.client.gui.components`
    - `AbstractScrollArea`
        - `scrollbarWidth` - The width of the scrollbar.
        - `defaultSettings` - Constructs the default scrollbar settings given the scroll rate.
        - `$ScrollbarSettings` - A record containing the metadata for the scrollbar.
    - `DebugScreenOverlay`
        - `showLightmapTexture` - Whether to render the lightmap texture on the overlay.
        - `toggleLightmapTexture` - Toggles whether the lightmap texture should be rendered.
    - `ScrollableLayout`
        - `setMinHeight` - Sets the minimum height of the container layout.
        - `$ReserveStrategy` - What to use when reserving the width of the scrollbar within the layout.
    - `Tooltip`
        - `component` - The component of a tooltip to display.
        - `style` - The identifier used to compute the tooltip background and frame.
- `net.minecraft.client.gui.components.debug`
    - `DebugEntryLookingAtEntityTags` - A debug entry for displaying an entity's tags.
    - `DebugScreenEntries#LOOKING_AT_ENTITY_TAGS` - The identifier for the entity tags debug entry.
- `net.minecraft.client.gui.navigation.FocusNavigationEvent$ArrowNavigation#with` - Sets the previous focus of the navigation.
- `net.minecraft.client.renderer`
    - `LightmapRenderStateExtractor` - Extracts the render state for the lightmap.
    - `UiLightmap` - The lightmap when in an user interface.
- `net.minecraft.client.renderer.state`
    - `LevelRenderState#lastEntityRenderStateCount` - The number of entities being rendered to the screen.
    - `LightmapRenderState` - The render state for the lightmap.
- `net.minecraft.core.component.DataComponents#ADDITIONAL_TRADE_COST` - A modifier that offsets the trade cost by the specified amount.
- `net.minecraft.core.component.predicates`
    - `DataComponentPredicates#VILLAGER_VARIANT` - A predicate that checks a villager type.
    - `VillagerTypePredicate` - A predicate that checks a villager's type.
- `net.minecraft.core.registries.ConcurrentHolderGetter` - A getter that reads references from a local cache, synchronizing to the original when necessary.
- `net.minecraft.gametest.framework`
    - `GameTestHelper#getBoundsWithPadding` - Gets the bounding box of the test area with the specified padding.
    - `GameTestInstance#padding` - The number of blocks spaced around each game test.
- `net.minecraft.network.protocol.game`
    - `ClientboundLowDiskSpaceWarningPacket` - A packet sent to the client warning about low disk space on the machine.
    - `ClientGamePacketListener#handleLowDiskSpaceWarning` - Handles the warning packet about low disk space.
- `net.minecraft.server.MinecraftServer`
    - `warnOnLowDiskSpace` - Sends a warning if the disk space is below 64 MiB.
    - `sendLowDiskSpaceWarning` - Sends a warning for low disk space.
- `net.minecraft.server.commands.SwingCommand` - A command that calls `LivingEntity#swing` for all targets.
- `net.minecraft.server.packs.AbstractPackResources#loadMetadata` - Loads the root `pack.mcmeta`.
- `net.minecraft.server.packs.resources.ResourceMetadata$MapBased` - A resource metadata that stores the sections in a map.
- `net.minecraft.tags.TagLoader$ElementLookup#fromGetters` - Constructs an element lookup from the given `HolderGetter`s depending on whether the element is required or not.
- `net.minecraft.util`
    - `LightCoordsUtil` - A utility for determining the light coordinates from light values.
    - `ProblemReporter$MapEntryPathElement` - A path element for some entry key in a map.
- `net.minecraft.world.entity`
    - `Entity$Flags` - An annotation that marks a particular value as a bitset of flags for an entity.
    - `Mob#asValidTarget` - Checks whether an entity is a valid target (can attack) for this entity.
    - `TamableAnimal#feed` - The player gives the stack as food, healing them either based on the stored component or some default value.
- `net.minecraft.world.entity.monster.piglin`
    - `Piglin`
        - `INVENTORY_SLOT_OFFSET` - The slot index offset for the inventory.
        - `INVENTORY_SIZE` - The size of a piglin's inventory. 
    - `PiglinAi#findNearbyAdultPiglins` - Returns a list of all adult piglins in this piglin's memory.
- `net.minecraft.world.item.DyeColor#VALUES` - A list of all dye colors.
- `net.minecraft.world.item.enchantment.EnchantmentTarget#NON_DAMAGE_CODEC` - A codec that only allows the attacker and victim enchantment targets.
- `net.minecraft.world.level.dimension.DimensionDefaults`
    - `BLOCK_LIGHT_TINT` - The default tint for the block light.
    - `NIGHT_VISION_COLOR` - The default color when in night vision.
- `net.minecraft.world.level.material.LavaFluid#LIGHT_EMISSION` - The amount of light the lava fluid emits.
- `net.minecraft.world.level.storage.loot.functions`
    - `EnchantRandomlyFunction$Builder#withOptions` - Specifies the enchantments that can be used to randomly enchant this item.
    - `SetRandomDyesFunction` - An item function that applies a random dye (if the item is in the `dyeable` tag) to the `DYED_COLOR` component from the provided list.
    - `SetRandomPotionFunction` - An item function that applies a random potion to the `POTION_CONTENTS` component from the provided list.
- `net.minecraft.world.level.storage.loot.providers.number.Sum` - A provider that sums the value of all provided number providers.

### List of Changes

- `com.mojang.blaze3d.opengl`
    - `GlDevice` now takes in a `GpuDebugOptions` containing the log level, whether to use synchronous logs, and whether to use debug labels instead of those parameters being passed in directly
    - `GlProgram#BUILT_IN_UNIFORMS`, `INVALID_PROGRAM` are now final
- `com.mojang.blaze3d.platform.Window` now takes in a list of `GpuBackend`s, the default `ShaderSource`, and the `GpuDebugOptions` instead of the `ScreenManager`
- `com.mojang.blaze3d.systems.RenderSystem`
    - `pollEvents` is now public
    - `flipFrame` no longer takes in the `Window`
    - `initRenderer` now only takes in the `GpuDevice`
- `net.minecraft.advancements.criterion`
    - `EntityTypePredicate#matches` now takes in a holder-wrapped `EntityType` instead of the raw type itself
    - `KilledTrigger$TriggerInstance#entityPredicate` -> `entity`
    - `PlayerPredicate` now takes in a `FoodPredicate`
        - Can be set using `PlayerPredicate$Builder#setFood`
- `net.minecraft.client.gui`
    - `GuiGraphics`
        - `blit` now has an overload that takes in the `GpuTextureView` and the `GpuSampler`
        - `setTooltipForNextFrame` now has an overload that takes in a list of `FormattedCharSequence`s for the tooltip and whether to replace any existing tooltip
    - `ItemSlotMouseAction#onSlotClicked` now takes in a `ContainerInput` instead of a `ClickType`
- `net.minecraft.client.gui.components`
    - `AbstractContainerWidget` now takes in an `AbstractScrollArea$ScrollbarSettings`
    - `AbstractScrollArea` now takes in an `AbstractScrollArea$ScrollbarSettings`
        - `scrollbarVisible` -> `scrollable`
        - `scrollBarY` is now public
        - `scrollRate` is now longer abstract
    - `AbstractTextAreaWidget` now takes in an `AbstractScrollArea$ScrollbarSettings`
    - `ScrollableLayout$Container` now takes in an `AbstractScrollArea$ScrollbarSettings`
    - `Tooltip#create` now has an overload that takes in an optional `TooltipComponent` and style `Identifier`
- `net.minecraft.client.gui.components.debug`
    - `DebugEntryLookingAtBlock`, `DebugEntryLookingAtFluid` -> `DebugEntryLookingAt`
        - More specifically, `$BlockStateInfo`, `$BlockTagInfo`, `$FluidStateInfo`, `$FluidTagInfo`
    - `DebugEntryLookingAtEntity#GROUP` is now public
    - `DebugScreenEntries`
        - `LOOKING_AT_BLOCK` -> `LOOKING_AT_BLOCK_STATE`, `LOOKING_AT_BLOCK_TAGS`
        - `LOOKING_AT_FLUID` -> `LOOKING_AT_FLUID_STATE`, `LOOKING_AT_FLUID_TAGS`
- `net.minecraft.client.gui.components.tabs.TabNavigationBar#setWidth` -> `updateWidth`, not one-to-one
- `net.minecraft.client.gui.navigation.FocusNavigationEvent$ArrowNavigation` now takes in a nullable `ScreenRectangle` for the previous focus
- `net.minecraft.client.gui.screens.ConfirmScreen#layout` is now final
- `net.minecraft.client.gui.screens.inventory.AbstractMountInventoryScreen#mount` is now final
- `net.minecraft.client.multiplayer.MultiPlayerGameMode#handleInventoryMouseClick` now takes in a `ContainerInput` instead of a `ClickType`
- `net.minecraft.client.particle.Particle#getLightColor` -> `getLightCoords`
- `net.minecraft.client.renderer`
    - `GameRenderer`
        - `getDarkenWorldAmount` -> `getBossOverlayWorldDarkening`
        - `lightTexture` -> `lightmap`, `levelLightmap`; not one-to-one
    - `LevelRenderer#getLightColor` -> `getLightCoords`
    - `LightTexture` -> `Lightmap`
        - `tick` -> `LightmapRenderStateExtractor#tick`
        - `updateLightTexture` -> `update`
        - `pack` -> `LightCoordsUtil#pack`
        - `block` -> `LightCoordsUtil#block`
        - `sky` -> `LightCoordsUtil#sky`
        - `lightCoordsWithEmission` -> `LightCoordsUtil#lightCoordsWithEmission`
    - `VirtualScreen` replaced by `GpuBackend`
- `net.minecraft.client.renderer.block.ModelBlockRenderer$Cache#getLightColor` -> `getLightCoords`
- `net.minecraft.client.renderer.blockentity.state`
    - `BrushableBlockRenderState#itemState` is now final
    - `EndPortalRenderState` is now a final set of `Direction`s rather than an `EnumSet`
    - `ShelfRenderState#items` is now final
- `net.minecraft.client.renderer.entity.state`
    - `FallingBlockRenderState#movingBlockRenderState` is now final
    - `WitherRenderState#xHeadRots`, `yHeadRots` are now final
- `net.minecraft.client.renderer.state.BlockBreakingRenderState#progress` is now final
- `net.minecraft.client.resources.sounds.AbstractSoundInstance#random` is now final
- `net.minecraft.core.WritableRegistry#bindTag` -> `bindTags`, now taking in a map of keys to holder lists instead of one mapping
- `net.minecraft.data.loot.BlockLootSubProvider`
    - `explosionResistant` is now private
    - `enabledFeatures` is now private
    - `map` is now private
- `net.minecraft.gametest.framework`
    - `GameTestHelper#assertBlockPresent` now has an overload that only takes in the block to check for
    - `GameTestServer#create` now takes in an `int` for the number of times to run all matching tests
    - `TestData` now takes in an `int` for the number of blocks padding around the test
- `net.minecraft.network.chat.LastSeenMessages#EMPTY` is now final
- `net.minecraft.network.protocol.game.ServerboundContainerClickPacket` now takes in a `ContainerInput` instead of a `ClickType`
- `net.minecraft.resources.RegistryDataLoader#load` now returns a `CompletableFuture` of the frozen registry access
- `net.minecraft.server.commands.ChaseCommand#DIMENSION_NAMES` is now final
- `net.minecraft.server.dedicated.DedicatedServerProperties#acceptsTransfers` is now final
- `net.minecraft.server.packs`
    - `BuiltInMetadata` has been merged in `ResourceMetadata`
        - `get` -> `ResourceMetadata#getSection`
        - `of` -> `ResourceMetadata#of`
    - `VanillaPackResourcesBuilder#setMetadata` now takes in a `ResourceMetadata` instead of a `BuiltInMetadata`
- `net.minecraft.tags.TagLoader`
    - `loadTagsFromNetwork` now takes in a `Registry` instead of a `WritableRegistry`, returning a map of tag keys to a list of holder entries
    - `loadTagsForRegistry` now has an overload which takes in registry key along with a element lookup, returning a map of tag keys to a list of holder entries
- `net.minecraft.util.Brightness`
    - `pack` -> `LightCoordsUtil#pack`
    - `block` -> `LightCoordsUtil#block`
    - `sky` -> `LightCoordsUtil#sky`
- `net.minecraft.util.worldupdate.WorldUpgrader` no longer takes in the `WorldData`
    - `STATUS_*` messages have been combined in `UpgradeStatusTranslator`
        - This also contains the specific datafix type
    - `running`, `finished`, `progress`, `totalChunks`, `totalFiles`, `converted`, `skipped`, `progressMap`, `status` have all been moved into `UpgradeProgress`, stored in `upgradeProgress`
    - `getProgress` -> `getTotalProgress`, not one-to-one
    - `$AbstractUpgrader`, `$SimpleRegionStorageUpgrader` -> `RegionStorageUpgrader`, not one-to-one
        - `$ChunkUpgrader`, `$EntityUpgrader`, `$PoiUpgrader` are now just constructed within `WorldUpgrader#work`
    - `$ChunkUpgrader#tryProcessOnePosition` has been partially abstracted into `getDataFixContentTag`, `verifyChunkPosAndEraseCache`, `verifyChunkPos`
    - `$FileToUpgrade` -> `FileToUpgrade`
- `net.minecraft.world.InteractionResult$ItemContext#NONE`, `DEFAULT` are now final
- `net.minecraft.world.entity`
    - `Entity`
        - `fluidHeight` is now final
        - `getTags` -> `entityTags`
    - `Leashable$Wrench#ZERO` is now final
    - `LivingEntity`
        - `interpolation` is now final
        - `canAttackType` -> `canAttack`, not one-to-one, taking in the `Entity` instead of the `EntityType`
        - `lungeForwardMaybe` -> `postPiercingAttack`
- `net.minecraft.world.entity.ai.behavior.SpearAttack` no longer takes in the approach distance `float`
- `net.minecraft.world.entity.ai.goal`
    - `BoatGoals` -> `FollowPlayerRiddenEntityGoal$FollowEntityGoal`
        - `BOAT` is replaced by `ENTITY`
    - `FollowBoatGoal` -> `FollowPlayerRiddenEntityGoal`, not one-to-one
- `net.minecraft.world.entity.ai.goal.target.NearestAttackableTargetGoal#targetConditions` is now final
- `net.minecraft.world.entity.monster.breeze.Breeze#idle`, `slide`, `slideBack`, `longJump`, `shoot`, `inhale` are now final
- `net.minecraft.world.entity.monster.piglin.PiglinAi#isZombified` now takes in the `Entity` instead of the `EntityType`
- `net.minecraft.world.entity.monster.warden.Warden#roarAnimationState`, `sniffAnimationState`, `emergeAnimationState`, `diggingAnimationState`, `attackAnimationState`, `sonicBoomAnimationState` are now final
- `net.minecraft.world.entity.monster.zombie.Zombie#handleAttributes` now takes in the `EntitySpawnReason`
- `net.minecraft.world.entity.player`
    - `Input#EMPTY` is now final
    - `Player`
        - `currentImpulseImpactPos` -> `LivingEntity#currentImpulseImpactPos`
        - `currentExplosionCause` -> `LivingEntity#currentExplosionCause`
        - `setIgnoreFallDamageFromCurrentImpulse` -> `LivingEntity#setIgnoreFallDamageFromCurrentImpulse`
        - `applyPostImpulseGraceTime` -> `LivingEntity#applyPostImpulseGraceTime`
        - `isIgnoringFallDamageFromCurrentImpulse` -> `LivingEntity#isIgnoringFallDamageFromCurrentImpulse`
        - `tryResetCurrentImpulseContext` -> `LivingEntity#tryResetCurrentImpulseContext`
        - `resetCurrentImpulseContext` -> `LivingEntity#resetCurrentImpulseContext`
        - `isInPostImpulseGraceTime` -> `LivingEntity#isInPostImpulseGraceTime`
- `net.minecraft.world.entity.vehicle.minecart.NewMinecartBehavior$MinecartStep#EMPTY` is now final
- `net.minecraft.world.inventory.AbstractContainerMenu#getQuickCraftPlaceCount` now takes in the number of quick craft slots instead of the set of slots itself
- `net.minecraft.world.item`
    - `CreativeModeTab$Output` is now `protected` from `public`
    - `EnderpearlItem#PROJECTILE_SHOOT_POWER` is now final
    - `Items#register*` methods are now `private` from `public`
    - `SnowballItem#PROJECTILE_SHOOT_POWER` is now final
    - `ThrowablePotionItem#PROJECTILE_SHOOT_POWER` is now final
    - `WindChargeItem#PROJECTILE_SHOOT_POWER` is now final
- `net.minecraft.world.item.alchemy.Potions` now deal with `Holder$Reference`s instead of the super `Holder`
- `net.minecraft.world.item.component.AttackRange`
    - `minRange` -> `minReach`
    - `maxRange` -> `maxReach`
    - `minCreativeRange` -> `minCreativeReach`
    - `maxCreativeRange` -> `maxCreativeReach`
- `net.minecraft.world.item.enchantment`
    - `ConditionalEffect#codec` no longer takes in the `ContextKeySet`
    - `Enchantment#doLunge` -> `doPostPiercingAttack`
    - `EnchantmentHelper#doLungeEffects` -> `doPostPiercingAttackEffects`
    - `TargetedConditionalEffect#codec`, `equipmentDropsCodec` no longer take in the `ContextKeySet`
- `net.minecraft.world.item.enchantment.effects.EnchantmentAttributeEffect#CODEC` -> `MAP_CODEC`
- `net.minecraft.world.item.equipment.Equippable#canBeEquippedBy` now takes in a holder-wrapped `EntityType` instead of the raw type itself
- `net.minecraft.world.level.LevelAccessor` no longer implements `LevelReader`
- `net.minecraft.world.level.block`
    - `FireBlock#setFlammable` is now `private` from `public`
    - `MultifaceSpreader$DefaultSpreaderConfig#block` is now final
- `net.minecraft.world.level.block.entity.TestInstanceBlockEntity`  
    - `getTestBoundingBox` - The bounding box of the test inflated by its padding.
    - `getTestBounds` -  The axis aligned bounding box of the test inflated by its padding.
- `net.minecraft.world.level.block.entity.vault`
    - `VaultConfig#DEFAULT`, `CODEC` are now final
    - `VaultServerData#CODEC` is now final
    - `VaultSharedData#CODEC` is now final
- `net.minecraft.world.level.chunk.ChunkAccess#blendingData` is now final
- `net.minecraft.world.level.chunk.storage.SimpleRegionStorage#upgradeChunkTag` now takes in an `int` for the target version
- `net.minecraft.world.level.gameevent.vibrations.VibrationSystem$Data#CODEC` is now final
- `net.minecraft.world.level.levelgen.blockpredicates`
    - `TrueBlockPredicate#INSTANCE` is now final
    - `UnobstructedPredicate#INSTANCE` is now final
- `net.minecraft.world.level.levelgen.placement.BiomeFilter#CODEC` is now final
- `net.minecraft.world.level.levelgen.structure.TemplateStructurePiece#template`, `placeSettings` are now final
- `net.minecraft.world.level.levelgen.structure.pools.alias`
    - `DirectPoolAlias#CODEC` is now final
    - `RandomGroupPoolAlias#CODEC` is now final
    - `RandomPoolAlias#CODEC` is now final
- `net.minecraft.world.level.levelgen.structure.templatesystem.LiquidSettings#CODEC` is now final
- `net.minecraft.world.level.storage.loot.functions`
    - `EnchantRandomlyFunction` now takes in a `boolean` of whether to include the additional trade cost component from the stack being enchanted
        - Set via `$Builder#includeAdditionalCostComponent`
    - `EnchantWithLevelsFunction` now takes in a `boolean` of whether to include the additional trade cost component from the stack being enchanted
        - Set via `$Builder#includeAdditionalCostComponent`
        - `$Builder#fromOptions` -> `withOptions`, now with overload to take in an optional `HolderSet`

### List of Removals

- `net.minecraft.SharedConstants#USE_WORKFLOWS_HOOKS`
- `net.minecraft.client.data.models.BlockModelGenerators#createGenericCube`
- `net.minecraft.client.gui.components.EditBox#setFilter`
- `net.minecraft.data.loot.BlockLootSubProvider(Set, FeatureFlagSet, Map, HolderLookup$Provider)`
- `net.minecraft.server.packs.AbstractPackResources#getMetadataFromStream`
- `net.minecraft.world`
    - `ContainerListener`
    - `SimpleContainer#addListener`, `removeListener`
- `net.minecraft.world.level.storage.loot.functions.SetOminousBottleAmplifierFunction#amplifier`
