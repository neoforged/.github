# Minecraft 1.21.11 -> 26.1 Mod Migration Primer

This is a high level, non-exhaustive overview on how to migrate your mod from 1.21.11 to 26.1. This does not look at any specific mod loader, just the changes to the vanilla classes.

This primer is licensed under the [Creative Commons Attribution 4.0 International](http://creativecommons.org/licenses/by/4.0/), so feel free to use it as a reference and leave a link so that other readers can consume the primer.

If there's any incorrect or missing information, please file an issue on this repository or ping @ChampionAsh5357 in the Neoforged Discord server.

Thank you to:

- @Shnupbups for some grammatical fixes

## Pack Changes

There are a number of user-facing changes that are part of vanilla which are not discussed below that may be relevant to modders. You can find a list of them on [Misode's version changelog](https://misode.github.io/versions/?id=26.1&tab=changelog).

## Java 25 and Deobfuscation

26.1 introduces two new changes into the general pipeline.

First, the Java Development Kit has been upgraded from 21 to 25. Vanilla makes use of these new features, such as [JEP 447](https://openjdk.org/jeps/447), which allows statements before `super` within constructors. For users within the modding scene, please make sure to update accordingly, or take advantage of your IDE or build tool features. Microsoft's OpenJDK can be found [here](https://learn.microsoft.com/en-us/java/openjdk/download#openjdk-25).

Vanilla has also returned to being deobfuscated, meaning that all value types now have the official names provided by Mojang. There are still some things that are not captured due to the Java compilation process, such as inlining primitive and string constants, but the majority are now provided. This will only have a change for users or mod loaders who used a different value type mapping set from the official mappings.

## Loot Type Unrolling

Loot pool entries, item functions, item conditions, nbt providers, number providers, and score providers no longer use a wrapping object type to act as the registered instance. Now, the registries directly take in the `MapCodec` used for the serialization and deserialization process. As such, `Loot*Type` classes that held the codec have been removed. `getType` is now renamed to `codec`, taking in the registered `MapCodec`.

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
        - `forIndexedField` - Creates a context for a given entry in a list.
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
    // The stack template the trader gives in return.
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
    // A list of loot item functions that modify the item
    // offered from `gives` to the player.
    // If not specified, `gives` is not modified.
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

Note that if duplicate trades are allowed, there is a potential race condition where if all trades' `merchant_predicate`s fail, then the offers will loop forever. This is because the method always assumes that there will be one trade that can be made.

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
            // Make sure the enchantment was added successfully with the given level.
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

## Data Component Initializers

Data components have begun their transition from being moved off the raw object and onto the `Holder` itself. This is to properly handle objects that are not available during construction, such as datapack registry objects for items. Currently, this is only implemented for `Item`s, but the system allows for any registry object, given its wrapped-holder, to store some defined data components.

Data components are attached to their holders through the `DataComponentInitializers`. During object construction, `DataComponentInitializers#add` is called, providing the `ResourceKey` identifier of the registry object, along with a `DataComponentInitializers$Initializer`. The initializer takes in three parameters: the builder for the component map, the full registries `HolderLookup$Provider`, and the key passed to `DataComponentInitializers#add`. The initializer functions like a consumer, also allowing for additional initializers to be chained via `$Initializer#andThen` or to easily add a component via `$Initializer#add`.

```java
// For some custom registry object
public ExampleObject(ResourceKey<ExampleObject> id) {
    // Register the data component initializer
    BuiltInRegistries.DATA_COMPONENT_INITIALIZERS.add(
        // The identifier for the registry object
        id,
        // The initializer function, taking in a component builder,
        // the registries context, and the id
        (components, context, key) -> components
            .set(DataComponents.MAX_DAMAGE, 1)
            .set(DataComponents.DAMAGE_TYPE, context.getOrThrow(DamageTypes.SPEAR))
    );
}
```

From there, the data components are initialized or reinitialized whenever `ReloadableServerResources#loadResources` is called (on datapack reload). The datapack objects are loaded first, then the components are set on the `Holder$Reference`s. From there, the components can be gathered via `Holder#components`.

```java
// For some Holder<ExampleObject> EXAMPLE_HOLDER
DataComponentMap components = EXAMPLE_HOLDER.components();
```

### Items

Since components are now initialized during resource reload, `Item$Properties` provides two methods to properly delay component initialization until such data components are loaded: `delayedComponent` and `delayedHolderComponent`. `delayedComponent` takes in the component type and a function that takes in the `HolderLookup$Provider` and returns the component value. `delayedHolderCOmponent` delegates to `delayedComponent`, taking in a `ResourceKey` and setting the registry object as the component value.

```java
public static final Item EXAMPLE_ITEM = new Item(
    new Item.Properties()
        .delayedComponent(
            // The component type whose construction
            // should be lazily initialized.
            DataComponents.JUKEBOX_PLAYABLE,
            // A function that takes in the registries
            // and returns the component value.
            context -> new JukeboxPlayable(context.getOrThrow(
                JukeboxSongs.THIRTEEN
            ))
        )
        .delayedHolderComponent(
            // The component type which has a
            // holder value type.
            DataComponents.DAMAGE_TYPE
            // The resource key for a registry
            // object of the associated holder
            // generic type.
            DamageTypes.SPEAR
        )
        // ...
);
```

### Recipes

Since data components now hold the true values from being lazily initialized after resource reload, `Recipe#assemble` no longer takes in the `HolderLookup$Provider`. Instead, it assumes that the recipe has all the required data stored passed into the recipe, either on a stack or directly.

- `net.minecraft.core`
    - `Holder`
        - `areComponentsBound` - Whether the components have been bound to the holder.
        - `components` - The components of the object stored on the holder.
        - `direct`, `$Direct` now can take in the `DataComponentMap`
        - `$Reference#bindComponents` - Stores the components on the holder reference.
    - `Registry#componentLookup` - Gets the lookup of component to holders.
- `net.minecraft.core.component`
    - `DataComponentInitializers` - A class that handles initializing the data components for component-attached objects.
    - `DataComponentLookup` - A lookup that maps the component type to the holders that use it.
    - `DataComponentMap$Builder#addValidator` - Adds a validator for the components on the object.
    - `DataComponentPatch`
        - `get` now takes in a `DataComponentGetter` and returns the raw component value
        - `$Builder#set` now has an overload that takes in an iterable of `TypedDataComponent`s
    - `DataComponents`
        - `DAMAGE_TYPE` now is a holder-wrapped `DamageType` instead of `EitherHolder`-wrapped
        - `PROVIDES_TRIM_MATERIAL` now is a holder-wrapped `TrimMaterial` instead of `ProvidesTrimMaterial`
        - `CHICKEN_VARIANT` now is a holder-wrapped `ChickenVariant` instead of `EitherHolder`-wrapped
        - `ZOMBIE_NAUTILUS_VARIANT` now is a holder-wrapped `ZombieNautilusVariant` instead of `EitherHolder`-wrapped
- `net.minecraft.core.registries`
    - `BuiltInRegistries#DATA_COMPONENT_INITIALIZERS` - A list of data components attached to registry entries.
    - `Registries#componentsDirPath` - The path directory for the components in a registry.
- `net.minecraft.data.PackOutput#createRegistryComponentPathProvider` - The path provider for the components registry report.
- `net.minecraft.data.info.ItemListReport` -> `RegistryComponentsReport`, not one-to-one
- `net.minecraft.resources`
    - `NetworkRegistryLoadTask` - A load task that handles registering registry objects and tags from the network, or else from a resource.
    - `RegistryDataLoader`
        - `$PendingRegistration` -> `RegistryLoadTask$PendingRegistration`
        - `$RegistryData` now takes in a `RegistryValidator` instead of a `boolean` 
        - `$RegistryLoadTask` -> `RegistryLoadTask`, not one-to-one
    - `RegistryValidator` - An interface that validates the entries in a registry, storing all errors in a map.
    - `ResourceManagerRegistryLoadTask` - A load task that handles registering registry objects and tags from the local resource manager.
- `net.minecraft.server.ReloadableServerResources#updateStaticRegistryTags` -> `updateComponentsAndStaticRegistryTags`, not one-to-one
- `net.minecraft.world.item`
    - `EitherHolder` class is removed
    - `Item`
        - `CODEC_WITH_BOUND_COMPONENTS` - An item codec that validates that the components are bound.
        - `$Properties`
            - `delayedComponent` - Sets the component lazily, providing the `HolderLookup$Provider` to get any dynamic elements.
            - `delayedHolderComponent` - Sets the component lazily for some holder-wrapped registry object.
    - `ItemStack#validateComponents` is now `private` from `public`
    - `JukeboxPlayable` now holds a holder-wrapped `JukeboxSong` instead of an `EitherHolder`-wrapped variant
    - `JukeboxSong#fromStack` no longer takes in the `HolderLookup$Provider`
    - `SpawnEggItem`
        - `spawnEntity` is now `static`
        - `byId` now returns an optional holder-wrapped `Item` instead of a `SpawnEggItem`
        - `eggs` is removed
        - `getType` is now `static`
        - `spawnOffspringFromSpawnEgg` is now `static`
- `net.minecraft.world.item.component`
    - `InstrumentComponent` now holds a holder-wrapped `Instrument` instead of an `EitherHolder`-wrapped variant
        - `unwrap` is removed
    - `ProvidesTrimMaterial` now holds a holder-wrapped `TrimMaterial` instead of an `EitherHolder`-wrapped variant
- `net.minecraft.world.item.crafting`
    - `Recipe#assemble` no longer takes in the `HolderLookup$Provider`
    - `SmithingTrimRecipe#applyTrim` no longer takes in the `HolderLookup$Provider`
- `net.minecraft.world.item.equipment.trim.TrimMaterials#getFromIngredient` is removed
- `net.minecraft.world.timeline.Timeline#validateRegistry` - Validates that each time marker was only defined once.

## Item Instances and Stack Templates

`ItemStack`s now have an immutable instance known as an `ItemStackTemplate`. Similar to the `ItemStack`, it contains the holder-wrapped `Item`, the number of items it represents, and a `DataComponentPatch` of the components to apply to the stack. The template can be transformed to a stack via `create`, or `apply` if adding additional data components. An `ItemStack` can likewise be turned into a template via `ItemStackTemplate#fromNonEmptyStack`. Templates are now used in place of `ItemStack`s were immutability is required (e.g., advancements, recipes, etc.). They provide a regular `CODEC`, a `MAP_CODEC`, and a `STREAM_CODEC` for network communication.

```java
ItemStackTemplate apple = new ItemStackTemplate(
    // The item of the stack
    Items.APPLE.builtInRegistryHolder(),
    // The number of items held
    5,
    // The components applied to the stack
    DataComponentPatch.builder()
        .set(DataComponents.ITEM_NAME, Component.literal("Apple?"))
        .build()
);

// Turn the template into a stack
ItemStack stack = apple.create();

// Creating a template from a non-empty stack
ItemStackTemplate fromStack = ItemStackTemplate.fromNonEmptyStack(stack);
```

To help standardize accessing the general components between the two, both the template and the stack implement `ItemInstance`, which provides access to the standard holder method checks, the count, and the `DataComponentGetter`. The common interface also means that if it doesn't matter whether the stack is mutable or not, then the `ItemInstance` can be used as the type.

```java
// Both are item instances
ItemInstance stack = new ItemStack(Items.APPLE);
ItemInstance template = new ItemStackTemplate(Items.APPLE);

// Get the holder or check something
Holder<Item> item = stack.typeHolder();
template.is(Items.APPLE);

// Get the number of items in the stack or template
int stackCount = stack.count();
int templateCount = template.count();

// Get the component values
Identifier stackModel = stack.get(DataComponents.ITEM_MODEL);
Identifier templateModel = template.get(DataComponents.ITEM_MODEL);
```

### Recipe Builders

Due to the addition of `ItemStackTemplate`s, `RecipeBuilder`s have changed slightly in their implementation. First, the builder no longer stores an `Item` for the result, instead defining the `defaultId` as the resource key recipe. As such, this allows for a more clear method of defining custom recipes that do not export an `Item`.

Of course, the change to this new format just has `defaultId` return `RecipeBuilder#getDefaultRecipeId` with the `ItemInstance` (either a stack or template), like so:

```java
public class ExampleRecipeBuilder implements RecipeBuilder {

    // The result of the recipe
    private final ItemStackTemplate result;

    public ExampleRecipeBuilder(ItemStackTemplate result) {
        this.result = result;
    }

    @Override
    public ResourceKey<Recipe<?>> defaultId() {
        // Get the default recipe id from the result
        return RecipeBuilder.getDefaultRecipeId(this.result);
    }

    // Implement everything else below
    // ...
}
```

- `net.minecraft.advancements`
    - `Advancement$Builder#display` now takes in an `ItemStackTemplate` instead of an `ItemStack`
    - `DisplayInfo` now takes in an `ItemStackTemplate` instead of an `ItemStack`
        - `getIcon` now returns an `ItemStackTemplate` instead of an `ItemStack`
- `net.minecraft.advancements.criterion`
    - `AnyBlockInteractionTrigger#trigger` now takes in an `ItemInstance` instead of an `ItemStack`
    - `ItemPredicate` now implements a predicate of `ItemInstance` instead of `ItemStack`
    - `ItemUsedOnLocationTrigger#trigger` now takes in an `ItemInstance` instead of an `ItemStack`
- `net.minecraft.client.particle.BreakingItemParticle$ItemParticleProvider#getSprite` now takes in an `ItemStackTemplate` instead of an `ItemStack`
- `net.minecraft.commands.arguments.item`
    - `ItemInput` is now a record
        - `serialize` is removed
        - `createItemStack` no longer takes in the `boolean` to check the size
    - `ItemParser#parse` now returns an `ItemResult` instead of an `ItemParser$ItemResult`
        - `$ItemResult` merged into `ItemInput`
- `net.minecraft.core.component.predicates`
    - `BundlePredicate` now deals with an iterable of `ItemInstance`s rather than `ItemStack`s
    - `ContainerPredicate` now deals with an iterable of `ItemInstance`s rather than `ItemStack`s
- `net.minecraft.core.particles.ItemParticleOption` now takes in an `ItemStackTemplate` instead of an `ItemStack`
    - There is also an overload for a regular `Item`
    - `getItem` now returns an `ItemStackTemplate` instead of an `ItemStack`
- `net.minecraft.data.recipes`
    - `RecipeBuilder`
        - `getResult` is removed
        - `defaultId` - The default identifiers for the recipe made with this builder.
        - `getDefaultRecipeId` now takes in an `ItemInstance` instead of an `ItemLike`
    - `RecipeProvider`
        - `oreSmelting`, `oreBlasting` now take in a `CookingBookCategory`
        - `oreCooking` no longer takes in the `RecipeSerializer` and now takes in a `CookingBookCategory`
        - `cookRecipes`, `simpleCookingRecipe` no longer take in the `RecipeSerializer`
        - `shapeless` now takes in an `ItemStackTemplate` instead of an `ItemStack`
    - `ShapedRecipeBuilder` now takes in an `ItemStackTemplate` for the result, with the `ItemLike` moved to an overload
        - Both constructors are `private`
    - `ShapelessRecipeBuilder` now takes in an `ItemStackTemplate` for the result instead of an `ItemStack`
    - `SimpleCookingRecipeBuilder` now takes in an `ItemStackTemplate` for the result, with the `ItemLike` moved to an overload
        - `generic` no longer takes in the `RecipeSerializer` and now takes in a `CookingBookCategory`
        - `blasting`, `smelting` now take in a `CookingBookCategory`
    - `SingleItemRecipeBuilder` now takes in an `ItemStackTemplate` for the result, with the `ItemLike` moved to an overload
        - The `ItemStackTemplate` constructor is made `private`
    - `SmithingTransformRecipeBuilder` now takes in an `ItemStackTemplate` for the result instead of an `Item`
    - `TransmuteRecipeBuilder` now takes in an `ItemStackTemplate` for the result instead of a `Holder<Item>`
        - The constructor is `private`
        - `transmute` now has an overload that takes in the `ItemStackTemplate` for the result
        - `addMaterialCountToOutput`, `setMaterialCount` - Handles the size of the result stack based on the number of materials used.
- `net.minecraft.network.chat.HoverEvent$ShowItem` now takes in an `ItemStackTemplate` instead of an `ItemStack`
- `net.minecraft.server.dialog.body.ItemBody` now takes in an `ItemStackTemplate` instead of an `ItemStack`
- `net.minecraft.world.entity.LivingEntity`
    - `dropFromEntityInteractLootTable` now takes in an `ItemInstance` instead of an `ItemStack` for the tool
    - `dropFromShearingLootTable` now takes in an `ItemInstance` instead of an `ItemStack` for the tool
- `net.minecraft.world.item`
    - `BundleItem#getSelectedItemStack` -> `getSelectedItem`, now returning an `ItemStackTemplate` instead of an `ItemStack`
    - `Item`
        - `getCraftingRemainder` now returns an `ItemStackTemplate` instead of an `ItemStack`
        - `$Properties#craftRemainder` now has an overload that takes in the `ItemStackTemplate`
    - `ItemInstance` - A typed item instance that can query the item, the size, and its components.
    - `ItemStack` now implements `ItemInstance`
        - `SINGLE_ITEM_CODEC`, `STRICT_CODEC`, `STRING_SINGLE_ITEM_CODEC`, `SIMPLE_ITEM_CODEC` are removed
        - `getMaxStackSize` -> `ItemInstance#getMaxStackSize`
    - `ItemStackTemplate` - A record containing the immutable components of a stack: the item, count, and components.
- `net.minecraft.world.item.component`
    - `BundleContents` now takes in a list of `ItemStackTemplate`s instead of `ItemStack`s
        - `items` now returns a list of `ItemStackTemplate`s instead of an iterable of `ItemStack`s
        - `itemsCopy` is removed
        - `getSelectedItem` - Returns the stack template of the selected item, or `null` if no item is selected.
    - `ChargedProjectile` is now a record
        - The constructor takes in a list of `ItemStackTemplate`s instead of `ItemStack`s
        - `of` -> `ofNonEmpty`
        - `getItems` -> `itemCopies`
    - `ItemContainerContents`
        - `stream` -> `allItemsCopyStream`
        - `nonEmptyStream` -> `nonEmptyItemCopyStream`
        - `nonEmptyItems`, `nonEmptyItemsCopy` -> `nonEmptyItems`, now returning an iterable of `ItemStackTemplate`s instead of `ItemStack`s
    - `UseRemainder` now takes in an `ItemStackTemplate` instead of an `ItemStack`
- `net.minecraft.world.item.crafting`
    - `AbstractCookingRecipe` now takes in an `ItemStackTemplate` instead of an `ItemStack` for the result
        - `$Factory#create` now takes in an `ItemStackTemplate` instead of an `ItemStack` for the result
    - `BlastingRecipe` now takes in an `ItemStackTemplate` instead of an `ItemStack` for the result
    - `CampfireCookingRecipe` now takes in an `ItemStackTemplate` instead of an `ItemStack` for the result
    - `ShapedRecipe` now takes in an `ItemStackTemplate` instead of an `ItemStack` for the result
    - `ShapelessRecipe` now takes in an `ItemStackTemplate` instead of an `ItemStack` for the result
    - `SingleItemRecipe` now takes in an `ItemStackTemplate` instead of an `ItemStack` for the result
        - `result` now returns an `ItemStackTemplate` instead of an `ItemStack`
        - `$Factory#create` now takes in an `ItemStackTemplate` instead of an `ItemStack` for the result
    - `SmeltingRecipe` now takes in an `ItemStackTemplate` instead of an `ItemStack` for the result
    - `SmithingTransformRecipe` now takes in an `ItemStackTemplate` instead of an `ItemStack` for the result
    - `SmokingRecipe` now takes in an `ItemStackTemplate` instead of an `ItemStack` for the result
    - `StonecutterRecipe` now takes in an `ItemStackTemplate` instead of an `ItemStack` for the result
    - `TransmuteRecipe` now takes in an `ItemStackTemplate` instead of an `ItemStack` for the result
    - `TransmuteResult` -> `ItemStackTemplate`, not one-to-one
        - `isResultUnchanged` is removed
        - `apply` -> `TransmuteRecipe#createWithOriginalComponents`, not one-to-one
- `net.minecraft.world.item.crafting.display.SlotDisplay$ItemStackSlotDisplay` now takes in an `ItemStackTemplate` instead of an `ItemStack`
- `net.minecraft.world.item.enchantment.EnchantmentHelper#getItemEnchantmentLevel` now takes in an `ItemInstance` instead of an `ItemStack`
- `net.minecraft.world.level.block.Block`
    - `dropFromBlockInteractLootTable` now takes in an `ItemInstance` instead of an `ItemStack`
    - `getDrops` now takes in an `ItemInstance` instead of an `ItemStack`
- `net.minecraft.world.level.block.entity.DecoratedPotBlockEntity#createdDecoratedPotItem` -> `createdDecoratedPotInstance`
    - `createDecoratedPotTemplate` creates the `ItemStackTemplate` instead of the `ItemStack`
- `net.minecraft.world.level.storage.loot`
    - `LootContext$ItemStackTarget` now implements an `ItemInstance` generic for the argument getter instead of the `ItemStack`
    - `LootContextArg$ArgCodecBuilder#anyItemStack` now requires a function taking in an `ItemInstance` context key instead of an `ItemStack` context key
- `net.minecraft.world.level.storage.loot.parameters.LootContextParams#TOOL` is now an `ItemInstance` context key instead of an `ItemStack` context key

## Serializer Records and Recipe Info

Recipes have been slightly reworked in their implementation. First, `RecipeSerializer` is now a record taking in the `MapCodec` and `StreamCodec` used to serialize and deserialize the recipe. As such, `Serializer` classes have been removed in their entirety, replaced with providing the codecs to the record during registration:

```java
// Assume some ExampleRecipe implements Recipe
// We'll say there's also only one INSTANCE
public static final RecipeSerializer<ExampleRecipe> EXAMPLE_RECIPE = new RecipeSerializer<>(
    // The map codec for reading the recipe to/from disk.
    MapCodec.unit(INSTANCE),
    // The stream codec for reading the recipe to/from the network.
    StreamCodec.unit(INSTANCE)
);
```

Second, some common data regarding the recipe settings and book information have been sectioned into separate objects. These objects are passed into the recipe as part of the constructor and used to more cleanly handle similar implementations across all recipes.

Vanilla provides four of these common object classes, split into two separate categories. `Recipe$CommonInfo` is used for general recipe settings. Meanwhile, `Recipe$BookInfo` is used for recipe book information, with `CraftingRecipe$CraftingBookInfo` for crafting recipes, and `AbstractCookingRecipe$CookingBookInfo` for cooking recipes (e.g., smelting, blasting, etc.). The common object classes provide methods for constructing the codecs as required, which can then be passed into the relevant recipe codec.

These classes are typically passed through the `Recipe` subclasses, used as boilerplate to implement abstract methods. None of the data in these objects are directly available outside the implementation itself, only through the methods defined in the `Recipe` interface. Because of this, classes like `NormalCraftingRecipe`, `CustomRecipe`, `SimpleSmithingRecipe`, and `SingleItemRecipe`, and `AbstractCookingRecipe` can be used to create a new recipe implementation by implementing a few methods.

Note that these common info classes are a design philosophy, which you can choose to implement if desired. It's only when building off existing recipe subtypes that you are required to make use of them.

- `net.minecraft.data.recipes`
    - `CustomCraftingRecipeBuilder` - A recipe builder that creates an arbitrary crafting recipe from some common and crafting book information.
    - `RecipeBuilder`
        - `determineBookCategory` -> `determineCraftingBookCategory`
        - `createCraftingCommonInfo` - Creates the common recipe info.
        - `createCraftingBookInfo` - Creates the crafting book info.
    - `RecipeUnlockAdvancementBuilder` - An advancement builder for unlocking a recipe.
    - `SpecialRecipeBuilder`, `special` now takes in a supplied `Recipe` instead of a function of `CraftingBookCategory` to `Recipe`
        - `unlockedBy` - The criteria required to unlock the recipe advancement.
- `net.minecraft.world.item.crafting`
    - `AbstractCookingRecipe` now takes in the `Recipe$CommonInfo` and `$CookingBookInfo` instead of the group and `CookingBookCategory`
        - `$Factory#create` now takes in the `Recipe$CommonInfo` and `$CookingBookInfo` instead of the group and `CookingBookCategory`
        - `$Serializer` replaced by `cookingMapCodec`, `cookingStreamCodec`
        - `$CookingBookInfo` - A record containing the common cooking information for the recipe book.
    - `BannerDuplicateRecipe` now takes in the banner `Ingredient` and the `ItemStackTemplate` result instead of the `CraftingBookCategory`
        - `MAP_CODEC`, `STREAM_CODEC`, `SERIALIZER` - Serializers for the recipe.
    - `BlastingRecipe` now takes in the `Recipe$CommonInfo` and `$AbstractCookingRecipeCookingBookInfo` instead of the group and `CookingBookCategory`
        - `MAP_CODEC`, `STREAM_CODEC`, `SERIALIZER` - Serializers for the recipe.
    - `BookCloningRecipe` now takes in the `Ingredient` source and material, the `MinMaxBounds$Int`s defining the generations that can be copied, and the `ItemStackTemplate` result instead of the `CraftingBookCategory`
        - `ALLOWED_BOOK_GENERATION_RANGES`, `DEFAULT_BOOK_GENERATION_RANGES` - Ranges for the book generation cloning.
        - `MAP_CODEC`, `STREAM_CODEC`, `SERIALIZER` - Serializers for the recipe.
    - `CampfireCookingRecipe` now takes in the `Recipe$CommonInfo` and `AbstractCookingRecipe$CookingBookInfo` instead of the group and `CookingBookCategory`
        - `MAP_CODEC`, `STREAM_CODEC`, `SERIALIZER` - Serializers for the recipe.
    - `CraftingRecipe$CraftingBookInfo` - A record containing the common crafting information for the recipe book.
    - `CustomRecipe` no longer takes in anything to its constructor
        - `$Serializer` is removed, replaced by its implementation's codecs
    - `DecoratedPotRecipe` now takes in the `Ingredient` patterns for each side along with the `ItemStackTemplate` result instead of the `CraftingBookCategory`
        - `MAP_CODEC`, `STREAM_CODEC`, `SERIALIZER` - Serializers for the recipe.
    - `DyeRecipe` now takes in the `Recipe$CommonInfo` and `CraftingRecipe$CraftingBookInfo` along with the `Ingredient` target and dye and the `ItemStackTemplate` result instead of the `CraftingBookCategory`
        - `MAP_CODEC`, `STREAM_CODEC`, `SERIALIZER` - Serializers for the recipe.
    - `FireworkRocketRecipe` now takes in the `Ingredient` shell, fuel, and star along with the `ItemStackTemplate` result instead of the `CraftingBookCategory`
        - `MAP_CODEC`, `STREAM_CODEC`, `SERIALIZER` - Serializers for the recipe.
    - `FireworkStarFadeRecipe` now takes in the `Ingredient` target and dye along with the `ItemStackTemplate` result instead of the `CraftingBookCategory`
        - `MAP_CODEC`, `STREAM_CODEC`, `SERIALIZER` - Serializers for the recipe.
    - `FireworkStarRecipe` now takes in the `$Shape` to `Ingredient` map; the `Ingredient` trail, twinkle, fuel, and dye; along with the `ItemStackTemplate` result instead of the `CraftingBookCategory`
        - `MAP_CODEC`, `STREAM_CODEC`, `SERIALIZER` - Serializers for the recipe.
    - `MapCloningRecipe` replaced with `TransmuteRecipe`
    - `MapExtendingRecipe` now extends `CustomRecipe` instead of `ShapedRecipe`
        - The constructor now takes in the `Ingredient` map and material along with the `ItemStackTemplate` result instead of the `CraftingBookCategory`
        - `MAP_CODEC`, `STREAM_CODEC`, `SERIALIZER` - Serializers for the recipe.
    - `NormalCraftingRecipe` - A class that defines the standard implementation for a crafting recipe.
    - `Recipe`
        - `showNotification`, `group` are no longer default
        - `$BookInfo` - The information for the recipe book.
        - `$CommonInfo` - The common information across all recipes.
    - `RecipeSerializer` is now a record containing the `MapCodec` and `StreamCodec`
        - The registered entries have been moved to `RecipeSerializers`
        - `register` is removed
    - `RecipeSerializers` - All vanilla serializers for recipes.
    - `RepairItemRecipe` no longer takes in anything
        - `INSTANCE` - The recipe serializer singleton.
        - `MAP_CODEC`, `STREAM_CODEC`, `SERIALIZER` - Serializers for the recipe.
    - `ShapedRecipe` now extends `NormalCraftingRecipe` instead of implementing `CraftingRecipe`
        - The constructor now takes in the `Recipe$CommonInfo` and `CraftingRecipe$CraftingBookInfo` instead of the group and `CraftingBookCategory`
        - `$Serializer` -> `MAP_CODEC`, `STREAM_CODEC`, `SERIALIZER`; not one-to-one
    - `ShapelessRecipe` now extends `NormalCraftingRecipe` instead of implementing `CraftingRecipe`
        - The constructor now takes in the `Recipe$CommonInfo` and `CraftingRecipe$CraftingBookInfo` instead of the group and `CraftingBookCategory`
        - `$Serializer` -> `MAP_CODEC`, `STREAM_CODEC`, `SERIALIZER`; not one-to-one
    - `ShieldDecorationRecipe` now takes in the `Ingredient` banner and target along with the `ItemStackTemplate` result instead of the `CraftingBookCategory`
        - `MAP_CODEC`, `STREAM_CODEC`, `SERIALIZER` - Serializers for the recipe.
    - `SimpleSmithingRecipe` - A class that defines the standard implementation of a smithing recipe.
    - `SingleItemRecipe` now takes in the `Recipe$CommonInfo` instead of the group
        - `commonInfo` - The common information for the recipe.
        - `$Factory#create` now takes in the `Recipe$CommonInfo` instead of the group
        - `$Serializer` -> `simpleMapCodec`, `simpleStreamCodec`; not one-to-one
    - `SmeltingRecipe` now takes in the `Recipe$CommonInfo` and `AbstractCookingRecipe$CookingBookInfo` instead of the group and `CookingBookCategory`
        - `MAP_CODEC`, `STREAM_CODEC`, `SERIALIZER` - Serializers for the recipe.
    - `SmithingTransformRecipe` now extends `SimpleSmithingRecipe` instead of implementing `SmithingRecipe`
        - The constructor now takes in the `Recipe$CommonInfo`
        - `$Serializer` -> `MAP_CODEC`, `STREAM_CODEC`, `SERIALIZER`; not one-to-one
    - `SmithingTrimRecipe` now extends `SimpleSmithingRecipe` instead of implementing `SmithingRecipe`
        - The constructor now takes in the `Recipe$CommonInfo`
        - `$Serializer` -> `MAP_CODEC`, `STREAM_CODEC`, `SERIALIZER`; not one-to-one
    - `SmokingRecipe` now takes in the `Recipe$CommonInfo` and `AbstractCookingRecipe$CookingBookInfo` instead of the group and `CookingBookCategory`
        - `MAP_CODEC`, `STREAM_CODEC`, `SERIALIZER` - Serializers for the recipe.
    - `StonecutterRecipe` now takes in the `Recipe$CommonInfo` instead of the group
        - `MAP_CODEC`, `STREAM_CODEC`, `SERIALIZER` - Serializers for the recipe.
    - `TippedArrowRecipe` -> `ImbueRecipe`, not one-to-one
        - The constructor now takes in the `Recipe$CommonInfo` and `CraftingRecipe$CraftingBookInfo` along with the `Ingredient` source and material and the `ItemStackTemplate` result instead of the `CraftingBookCategory`
        - `MAP_CODEC`, `STREAM_CODEC`, `SERIALIZER` - Serializers for the recipe.
    - `TransmuteRecipe` now extends `NormalCraftingRecipe` instead of implementing `CraftingRecipe`
        - The constructor now takes in the `Recipe$CommonInfo` and `CraftingRecipe$CraftingBookInfo` along with the `MinMaxBounds$Ints` and `boolean` handling the material count and adding it to the result instead of the group and `CraftingBookCategory`
        - `$Serializer` -> `MAP_CODEC`, `STREAM_CODEC`, `SERIALIZER`; not one-to-one
- `net.minecraft.world.item.crafting.display.SlotDisplay`
    - `$OnlyWithComponent` - A display only with the contents having the desired components.
    - `$WithAnyPotion` - A display with the contents having any potion contents component.
- `net.minecraft.world.level.storage.loot.functions.SmeltItemFunction#smelted` now can take in whether to use the input material count

## Dye Component

Specifying whether an item can be used as a dye material is now handled through the `DYE` data component. The component specifies a `DyeColor`, which can be set via `Item$Properties#component`:

```java
public static final Item EXAMPLE_DYE = new Item(new Item.Properties().component(
    DataComponents.DYE, DyeColor.WHITE
));
```

However, a dye material's behavior is not fully encompassed by the component alone. In most cases, the component is used in conjunction with some other tag or subclass to get the desired behavior.

### Entities and Signs

Dying sheeps and signs are handled purely through the `DyeItem` subclass, checking if the item has the `DYE` component. 

Dying the collars of wolfs and cats, on the other hand, can be any item, assuming it has the `DYE` component. Additionally, the item must be in the `ItemTags#WOLF_COLLAR_DYES` or `ItemTags#CAT_COLLAR_DYES` tag, respectively.

### Dye Recipes

The `DyeRecipe`, formerly named `ArmorDyeRecipe`, can take any target ingredient and apply the colors of the dye ingredients to obtain the desired result with the associated `DYED_COLOR` component. Any item can be considered a dye; however, those without the `DYE` component will default to `DyeColor#WHITE`. Armor makes used of `RecipeProvider#dyedItem` to allow any item in the `ItemTags#DYES` tag to dye armor. However, bundles and shulker boxes have their color components as different items, meaning instead the default recipes are tied directly to the vanilla `DyeItem`, meaning a separate recipe will need to be generated for applying dyes to those items.

The loom, firework star, and firework star face recipes on the other hand expect any dye material to have the `DYE` component. The loom has an additional requirement of the item being in the `ItemTags#LOOM_DYES` tag.

- `net.minecraft.core.component.DataComponents#DYE` - Represents that an item can act as a dye for the specific color.
- `net.minecraft.data.recipes.RecipeProvider`
    - `dyedItem` - Creates a dyed item recipe.
    - `dyedShulkerBoxRecipe` - Creates a dyed shulker box recipe.
    - `dyedBundleRecipe` - Creates a dyed bundle recipe.
- `net.minecraft.world.item`
    - `BundleItem`
        - `getAllBundleItemColors`, `getByColor` are removed
    - `DyeColor#VALUES` - A list of all dye colors.
    - `DyeItem` no longer takes in the `DyeColor`
        - `getDyeColor`, `byColor` are removed
- `net.minecraft.world.item.component.DyedItemColor#applyDyes` now takes in a list of `DyeColor`s instead of `DyeItem`s
    - An overload can also take in a `DyedItemColor` component instead of the `ItemStack`
- `net.minecraft.world.item.crafting.ArmorDyeRecipe` -> `DyeRecipe`, not one-to-one
- `net.minecraft.world.item.crafting.display.SlotDisplay$DyedSlotDemo` - A display for demoing dying an item.
- `net.minecraft.world.level.block`
    - `BannerBlock#byColor` is removed
    - `ShulkerBoxBlock#getBlockByColor`, `getColoredItemStack` are removed

## World Clocks and Time Markers

World clocks are objects that represent some time that increases every tick from when the world first loads. These clocks are used as timers to properly handle timing-based events (e.g., the current day, sleeping). Vanilla provides two world clocks: one for `minecraft:overworld` and one for `minecraft:the_end`.

Creating a clock is rather simple: just an empty, datapack registry object in `world_clock`.

```json5
// For some world clock 'examplemod:EXAMPLE_CLOCK'
// JSON at 'data/examplemod/world_clock/EXAMPLE_CLOCK.json'
{}
```

From there, you can query the clock state through the `ClockManager` via `Level#clockManager` or `MinecraftServer#clockManager`:

```java
// For some Level level
// Assume we have some ResourceKey<WorldClock> EXAMPLE_CLOCK

// Get the clock reference
Holder.Reference<WorldClock> clock = level.registryAccess().getOrThrow(EXAMPLE_CLOCK);

// Query the clock time
long ticksPassed = level.clockManager().getTotalTicks(clock);
```

If accessing the clock from the server, you can also modify the state of the clock:

```java
// For some ServerLevel level
// Assume we have some ResourceKey<WorldClock> EXAMPLE_CLOCK

// Get the clock reference
Holder.Reference<WorldClock> clock = level.registryAccess().getOrThrow(EXAMPLE_CLOCK);

// Get the server clock manager
ServerClockManager clockManager = level.clockManager();

// Set the total number of ticks that have passed
clockManager.setTotalTicks(clock, 0L);

// Add the a certain value to the number of ticks
clockManager.addTicks(clock, 10L);

// Pause the clock
clockManager.setPaused(
    clock,
    // `true` to pause.
    // `false` to unpause.
    true
);
```

### Marking Timelines

On its own, world clocks are rather limited in scope as you have to keep track of the number of ticks that have passed. However, when used with timelines, specific recurring times can be marked to handle timing-based events.

As a refresher, `Timeline`s are a method of modifying attributes based on some `WorldClock`. Currently, all vanilla timelines use the `minecraft:overworld` world clock to keep track of their timing, via the `clock` field. All timelines must define a clock it uses; however, be aware that clocks to not support any synchronization by default, meaning that updating the time on one clock will not affect the other.

```json5
// For some timeline 'examplemod:example_timeline'
// In `data/examplemod/timeline/example_timeline.json
{
    // Uses the custom clock.
    // Any changes to the clock time will only affect
    // timelines using this clock.
    // e.g., `minecraft:overworld` will not be changed.
    "clock": "examplemod:example_clock",
    // Perform actions on a 24,000 tick interval.
    "period_ticks": "24000",
    // ...
}


// For some timeline 'examplemod:example_timeline_2'
// In `data/examplemod/timeline/example_timeline_2.json
{
    // Uses the same clock as the one above.
    // Changes to the clock time will affect both
    // timelines.
    "clock": "examplemod:example_clock",
    // Perform actions on a 1,000 tick interval.
    "period_ticks": "1000",
    // ...
}

// For some timeline 'examplemod:example_overworld'
// In `data/examplemod/timeline/example_overworld.json
{
    // Uses the vanilla overworld clock.
    // Changes to the clock time will not affect
    // the above timelines as they use a different
    // clock.
    "clock": "minecraft:overworld",
    // Perform actions on a 6,000 tick interval.
    "period_ticks": "6000",
    // ...
}
```

Timelines can also define time markers, which is just some identifier for a time within the timeline period for a world clock. These are commonly used within commands (e.g. `/time set`), to check if a certain time has passed (e.g. village sieges), or to skip to the given time (e.g. waking up from sleep). Time markers are defined in `time_markers`, specifying the `ticks` within the period and whether it can be `show_in_commands`. As time markers are identified by their world clock, timelines using the same world clock cannot define the same time markers.

```json5
// For some timeline 'examplemod:example_timeline'
// In `data/examplemod/timeline/example_timeline.json
{
    // The identifier of the clock.
    "clock": "examplemod:example_clock",
    // Perform actions on a 24,000 tick interval.
    "period_ticks": "24000",
    // The markers within the period specified by the timeline
    "time_markers": {
        // A marker
        "examplemod:example_marker": {
            // The number of ticks within the period
            // that this marker represents.
            // e.g., 5000, 29000, 53000, etc.
            "ticks": 5000,
            // When true, allows the time marker to be
            // suggested in the command. If false,
            // the time marker can still be used in the
            // command, it just won't be suggested.
            "show_in_commands": true
        }
    }
    // ...
}
```

Once the time markers are defined, they are registered for a world clock in the `ServerClockManager`, allowing them to be used given the `ResourceKey`.

```java
// For some ServerLevel level
// Assume we have some ResourceKey<WorldClock> EXAMPLE_CLOCK
// Assume we have some ResourceKey<ClockTimerMarker> EXAMPLE_MARKER

// Get the clock reference
Holder.Reference<WorldClock> clock = level.registryAccess().getOrThrow(EXAMPLE_CLOCK);

// Get the server clock manager
ServerClockManager clockManager = level.clockManager();

// Check if the time is at the specified marker
// This should be used when checking for a specific time event every tick
boolean atMarker = clockManager.isAtTimeMarker(clock, EXAMPLE_MARKER);

// Skip to the time specified by the marker
// If the world clock is at 3000, sets the clock time to 5000
// If the world clock is at 6000, sets the clock time to 29000
// Returns whether the time was set, which is always true if the marker
// exists for the clock.
boolean timeSet = clockManager.skipToTimeMarker(clock, EXAMPLE_MARKER);
```

- `net.minecraft.client.ClientClockManager` - Manages the ticking of the world clocks on the client side.
- `net.minecraft.client.gui.components.debug.DebugEntryDayCount` - Displays the current day of the Minecraft world.
- `net.minecraft.client.multiplayer`
    - `ClientLevel`
        - `setTimeFromServer` no longer takes in the `long` day time nor the `boolean` of whether to tick the day time
        - `$ClientLevelData#setDayTime` is removed
    - `ClientPacketListener#clockManager` - Gets the client clock manager.
- `net.minecraft.client.renderer.EndFlashState#tick` now takes in the end clock time instead of the game time
- `net.minecraft.commands.arguments.ResourceArgument`
    - `getClock` - Gets a reference to the world clock given the string resource identifier.
    - `getTimeline` - Gets a reference to the timeline given the string resource identifier.
- `net.minecraft.core.registries.Registries#WORLD_CLOCK` - The registry identifier for the world clock.
- `net.minecraft.gametest.framework.TestEnvironmentDefinition$TimeOfDay` -> `$ClockTime`, not one-to-one
- `net.minecraft.network.protocol.game.ClientboundSetTimePacket` now takes in a map of clocks to their states instead of the day time `long` and `boolean`
- `net.minecraft.server`
    - `MinecraftServer`
        - `forceTimeSynchronization` -> `forceGameTimeSynchronization`, not one-to-one
        - `clockManager` - The server clock manager.
- `net.minecraft.server.level.ServerLevel#setDayTime`, `getDayCount` are removed
- `net.minecraft.world.attribute.EnvironmentAttributeSystem$Builder#addTimelineLayer` now takes in the `ClockManager` instead of a `LongSupplier`
- `net.minecraft.world.clock`
    - `ClockManager` - A manager that gets the total number of ticks that have passed for a world clock.
    - `ClockState` - The current state of the clock, including the total number of ticks and whether the clock is paused.
    - `ClockTimeMarker` - A marker that keeps track of a certain period of time for a world clock.
    - `ClockTimeMarkers` - All vanilla time markers.
    - `PackedClockStates` - A map of clocks to their states, compressed for storage or other use.
    - `ServerClockManager` - Manages the ticking of the world clocks on the server side.
    - `WorldClock` - An empty record that represents a key for timing some location.
    - `WorldClocks` - All vanilla world clocks.
- `net.minecraft.world.level.Level`
    - `getDayTime` -> `getOverworldClockTime`, not one-to-one
    - `clockManager` - The clock manager.
    - `getDefaultClockTime` - Gets the clock time for the current dimension.
- `net.minecraft.world.level.dimension.DimensionType` now takes in a default, holder-wrapped `WorldClock`
- `net.minecraft.world.level.storage`
    - `LevelData#getDayTime` is removed
    - `ServerLevelData`
        - `setDayTime` is removed
        - `setClockStates`, `clockStates` - Handles the current states of the world clocks.
- `net.minecraft.world.level.storage.loot.predicates.TimeCheck` now takes in a holder-wrapped `WorldClock`
    - `time` now takes in a holder-wrapped `WorldClock`
    - `$Builder` now takes in a holder-wrapped `WorldClock`
- `net.minecraft.world.timeline`
    - `AttributeTrack#bakeSampler` now takes in a holder-wrapped `WorldClock`
    - `AttributeTrackSampler` now takes in a holder-wrapped `WorldClock`, and a `ClockManager` instead of a `LongSupplier` for the day time getter
    - `Timeline` now takes in a holder-wrapped `WorldClock` along with a map of time markers to their infos
        - `#builder` now takes in a holder-wrapped `WorldClock`
        - `getPeriodCount` - Gets the number of times a period has occurred within the given timeframe.
        - `getCurrentTicks` now takes in the `ClockManager` instead of the `Level`
        - `getTotalTicks` now takes in the `ClockManager` instead of the `Level`
        - `clock` - Returns the holder-wrapped `WorldClock`.
        - `registerTimeMarkers` - Registers all `ClockTimeMarker` defined in this timeline.
        - `createTrackSampler` now takes in a `ClockManager` instead of a `LongSupplier` for the day time getter
        - `$Builder#addTimeMarker` - Adds a time markers at the given tick, along with whether the marker can be suggested in commands.
    - `Timelines#DAY` -> `OVERWORLD_DAY`

## Minor Migrations

The following is a list of useful or interesting additions, changes, and removals that do not deserve their own section in the primer.

### Plantable Tags

The blocks to determine whether a plantable can survive or be placed on has been moved to block and fluid tags. Each of these tags starts with `support_*` along with the block (e.g., `bamboo`, `cactus`) or group (e.g., `crops`, `dry_vegetation`). This is handled by the relevant block subclasses overriding either `Block#canSurvive`, or for vegetation `VegetationBlock#mayPlaceOn`.

- `net.minecraft.world.level.block`
    - `AttachedStemBlock` now takes in a `TagKey` for the blocks it can be placed on
    - `FarmBlock` -> `FarmlandBlock`
    - `FungusBlock` -> `NetherFungusBlock`, not one-to-one
    - `RootsBlock` -> `NetherRootsBlock`, not one-to-one
    - `WaterlilyBlock` -> `LilyPadBlock`, not one-to-one
    - `StemBlock` now takes in a `TagKey` for the blocks it can be placed on, and a `TagKey` for the blocks its fruit can be placed on

### Container Screen Changes

The usage of `AbstractContainerScreen`s has changed slightly, requiring some minor changes. First `imageWidth` and `imageHeight` are now final, settable as parameters within the constructor. If these two are not specified, they default to the original 176 x 166 background image.

```java
// Assume some AbstractContainerMenu subclass exists
public class ExampleContainerScreen extends AbstractContainerScreen<ExampleContainerMenu> {

    // Constructor
    public ExampleContainerScreen(ExampleContainerMenu menu, Inventory playerInventory, Component title) {
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

// For some BlockEntity blockEntity, check the BlockEntityType
blockEntity.is(BlockEntityType.CHEST);

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
- `net.minecraft.world.level.block.entity`
    - `BlockEntity` now implements `TypedInstance<BlockEntityType>`
    - `BlockEntityType#getKey` is removed
- `net.minecraft.world.level.block.state.BlockBehaviour$BlockStateBase` now implements `TypedInstance<Block>`
    - `getBlockHolder` -> `typeHolder`
    - `getTags` -> `tags`
- `net.minecraft.world.level.material.FluidState` now implements `TypedInstance<Fluid>`
    - `holder` -> `typeHolder`
    - `getTags` -> `tags`

### Entity Textures and Adult/Baby Models

Entity textures within `assets/minecraft/textures/entity/*` have now been sorted into subdirectories (e.g., `entity/panda` for panda textures, or `entity/pig` for pig textures). Most textures have been named with the entity type starting followed by an underscore along with its variant (e.g., `arrow_tipped` for tipped arrow, `pig_cold` for the cold pig variant, or `panda_brown` for the brown panda variant).

Additionally, some animal models have been split into separate classes for the baby and adult variants. These models either directly extend an abstract model implementation (e.g., `AbstractFelineModel`) or the original model class (e.g., `PigModel`).

- `net.minecraft.client.animation.definitions.BabyAxolotlAnimation` - Animations for the baby axolotl.
- `net.minecraft.client.model.QuadrupedModel` now has a constructor that takes in a function for the `RenderType`
- `net.minecraft.client.model.animal.axolotl.AxolotlModel` -> `AdultAxolotlModel`, `BabyAxolotlModel`
- `net.minecraft.client.model.animal.chicken`
    - `AdultChickenModel` - Entity model for the adult chicken.
    - `BabyChickenModel` - Entity model for the baby chicken.
    - `ChickenModel` is now abstract
        - `RED_THING` -> `AdultChickenModel#RED_THING`
        - `BABY_TRANSFORMER` has been directly merged into the layer definition for the `BabyChickenModel`
        - `createBodyLayer` -> `AdultChickenModel#createBodyLayer`
        - `createBaseChickenModel` -> `AdultChickenModel#createBaseChickenModel`
    - `ColdChickenModel` now extends `AdultChickenModel`
- `net.minecraft.client.model.animal.cow.BabyCowModel` - Entity model for the baby cow.
- `net.minecraft.client.model.animal.dolphin.BabyDolphinModel` - Entity model for the baby dolphin.
- `net.minecraft.client.model.animal.equine`
    - `AbstractEquineModel#BABY_TRANSFORMER` has been directly merged into the layer definition for the `BabyDonkeyModel`
    - `BabyDonkeyModel` - Entity model for the baby donkey.
    - `DonkeyModel#createBabyLayer` -> `BabyDonkeyModel#createBabyLayer`
    - `EquineSaddleModel#createFullScaleSaddleLayer` is removed
        Merged into `createSaddleLayer`, with the baby variant removed
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
- `net.minecraft.client.model.animal.squid`
    - `BabySquidModel` - Entity model for the baby squid.
    - `SquidModel#createTentacleName` is now protected from private
- `net.minecraft.client.model.animal.turtle`
    - `AdultTurtleModel` - Entity model for the adult turtle.
    - `BabyTurtleModel` - Entity model for the baby turtle.
    - `TurtleModel` is now abstract
        - `BABY_TRANSFORMER` has been directly merged into the layer definition for the `BabyTurtleModel`
        - `createBodyLayer` -> `AdultTurtleModel#createBodyLayer`, `BabyTurtleModel#createBodyLayer`
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
    - `DONKEY_BABY_SADDLE` is removed
    - `HORSE_BABY_ARMOR` is removed
    - `HORSE_BABY_SADDLE` is removed
    - `MULE_BABY_SADDLE` is removed
    - `SKELETON_HORSE_BABY_SADDLE` is removed
    - `UNDEAD_HORSE_BABY_ARMOR` is removed
    - `ZOMBIE_HORSE_BABY_SADDLE` is removed
- `net.minecraft.client.renderer.entity`
    - `AxolotlRenderer` now takes in an `EntityModel<AxolotlRenderState>` for its generic
    - `CatRenderer` now takes in an `AbstractFelineModel` for its generic
    - `DonkeyRenderer` now takes in an `EquipmentClientInfo$LayerType` and `ModelLayerLocation` for the saddle layer and model, and splits the `DonkeyRenderer$Type` into the adult type and baby type
        - `$Type`
            - `DONKEY_BABY` - A baby variant of the donkey.
            - `MULE_BABY` - A baby variant of the mule.
    - `OcelotRenderer` now takes in an `AbstractFelineModel` for its generic
    - `UndeadHorseRenderer` now takes in an `EquipmentClientInfo$LayerType` and `ModelLayerLocation` for the saddle layer and model, and splits the `UndeadHorseRenderer$Type` into the adult type and baby type
        - `$Type`
            - `SKELETON_BABY` - A baby variant of the skeleton horse.
            - `ZOMBIE_BABY` - A baby variant of the zombie horse.
- `net.minecraft.client.renderer.entity.layers.CatCollarLayer` now takes in an `AbstractFelineModel` for its generic
- `net.minecraft.client.renderer.entity.state`
    - `AxolotlRenderState`
        - `swimAnimation` - The state of swimming.
        - `walkAnimationState` - The state of walking on the ground, not underwater.
        - `walkUnderWaterAnimationState` - The state of walking underwater.
        - `idleUnderWaterAnimationState` - The state of idling underwater but not on the ground.
        - `idleUnderWaterOnGroundAnimationState` - The state of idling underwater while touching the seafloor.
        - `idleOnGroundAnimationState` - The state of idling on the ground, not underwater.
    - `RabbitRenderState`
        - `hopAnimationState` - The state of the hop the entity is performing.
        - `idleHeadTiltAnimationState` - The state of the head tilt when performing the idle animation.
- `net.minecraft.world.entity.animal.axolotl.Axolotl`
    - `swimAnimation` - The state of swimming.
    - `walkAnimationState` - The state of walking on the ground, not underwater.
    - `walkUnderWaterAnimationState` - The state of walking underwater.
    - `idleUnderWaterAnimationState` - The state of idling underwater but not on the ground.
    - `idleUnderWaterOnGroundAnimationState` - The state of idling underwater while touching the seafloor.
    - `idleOnGroundAnimationState` - The state of idling on the ground, not underwater.
    - `$AnimationState` -> `$AxolotlAnimationState`
- `net.minecraft.world.entity.animal.chicken.ChickenVariant` now takes in a resource for the baby texture
- `net.minecraft.world.entity.animal.cow.CowVariant` now takes in a resource for the baby texture
- `net.minecraft.world.entity.animal.cat.CatVariant` now takes in a resource for the baby texture
    - `CatVariant#assetInfo` - Gets the entity texture based on whether the entity is a baby.
- `net.minecraft.world.entity.animal.equine.AbstractHorse#BABY_SCALE` - The scale of the baby size compared to an adult.
- `net.minecraft.world.entity.animal.pig.PigVariant` now takes in a resource for the baby texture
- `net.minecraft.world.entity.animal.rabbit.Rabbit`
    - `hopAnimationState` - The state of the hop the entity is performing.
    - `idleHeadTiltAnimationState` - The state of the head tilt when performing the idle animation.
- `net.minecraft.world.entity.animal.wolf`
    - `WolfSoundVariant` -> `WolfSoundVariant$WolfSoundSet`
        - The class itself now holds two sound sets for the adult wolf and baby wolf.
    - `WolfSoundVariants$SoundSet#getSoundEventSuffix` -> `getSoundEventIdentifier`
- `net.minecraft.world.entity.animal.wolf.WolfVariant` now takes in an asset info for the baby variant

### The Removal of interactAt

`Entity#interactAt` has been removed, with all further invocation merged with `Entity#interact`. Originally, `Entity#interactAt` was called if the hit result was within the entity's interaction range. If `interactAt` did not consume the action (i.e., `InteractionResult#consumesAction` returns false), then `Entity#interact` is called. Now, `Entity#interact` is called if within the entity's interaction range, taking in the `Vec3` location of interaction.

```java
// In some Entity subclass

@Override
public InteractionResult interact(Player player, InteractionHand hand, Vec3 location) {
    // Handle the interaction
    super.interact(player, hand, location);
}
```

`interactAt` checks if result does not consume the interaction, and if not calls `interact`
Now, `interact` is called with the entity hit

- `net.minecraft.client.multiplayer.MultiPlayerGameMode`
    - `interact` now takes in the `EntityHitResult`
    - `interactAt` is removed
- `net.minecraft.world.entity.Entity`
    - `interact` now takes in a `Vec3` for the location of the interaction
    - `interactAt` is removed
- `net.minecraft.world.entity.player.Player#interactOn` now takes in a `Vec3` for the location of the interaction

### Solid and Translucent Features

Feature rendering has been further split into two passes: one for solid render types, and one for translucent render types. As such, most `render` methods now have a `renderSolid` and `renderTranslucent` method, respectively. Those which only render solid or translucent data only have one of the methods.

- `net.minecraft.client.renderer.SubmitNodeCollector$ParticleGroupRenderer`
    - `isEmpty` - Whether there are no particles to render in this group.
    - `prepare` now takes whether the particles are being prepared for the translucent layer
    - `render` no longer takes in the translucent `boolean`
- `net.minecraft.client.renderer.features`
    - Feature `render` methods have been split into `renderSolid` for solid render types, and `renderTranslucent` for see-through render types
    - `FeatureRenderDispatcher`
        - `renderAllFeatures` has been split into `renderSolidFeatures` and `renderTranslucentFeatures`
            - The original method now calls both of these methods, first solid then translucent 
        - `clearSubmitNodes` - Clears the submit node storage.
        - `renderTranslucentParticles` - Renders collected translucent particles.
    - `FlameFeatureRenderer#render` -> `renderSolid`
    - `LeashFeatureRenderer#render` -> `renderSolid`
    - `NameTagFeatureRenderer#render` -> `renderTranslucent`
    - `ShadowFeatureRenderer#render` -> `renderTranslucent`
    - `TextFeatureRenderer#render` -> `renderTranslucent`

### ChunkPos, now a record

`ChunkPos` is now a record. The `BlockPos` constructor has been replaced by `ChunkPos#containing` while the packed `long` constructor has been replaced by `ChunkPose#unpack`.

- `net.minecraft.world.level.ChunkPos` is now a record
    - `ChunkPos(BlockPos)` -> `containing`
    - `ChunkPos(long)` -> `unpack`
    - `toLong`, `asLong` -> `pack`

### No More Tripwire Pipelines

The tripwire render pipeline has been completely removed. Now, tripwires make use of `cutout` with an `alpha_cutoff_bias` of 0.1 for the texture.

- `net.minecraft.client.renderer.RenderPipelines#TRIPWIRE_BLOCK`, `TRIPWIRE_TERRAIN` are removed
- `net.minecraft.client.renderer.chunk`
    - `ChunkSectionLayer#TRIPWIRE` is removed
    - `ChunkSectionLayerGroup#TRIPWIRE` is removed
- `net.minecraft.client.renderer.rendertype.RenderTypes#tripwireMovingBlock` is removed

### Activities and Brains

Activities, which define how a living entity behaves during a certain phase, now are stored and passed around through `ActivityData`. This contains the activity type, the behaviors to perform along with their priorities, the conditions on when this activity activates, and the memories to erase when the activity has stopped. `Brain#provider` now takes in an `$ActivitySupplier` to construct a list of `ActivityData` the entity performs. 

Brains have also changed slightly. First, the `Brain` itself is not directly serializable. Instead, the brain is `$Packed` into the record, holding a map of its current memories. To get the memory types used, they are typically extracted from those that the `Sensor#requires`. Additionally, `LivingEntity#brainProvider` no longer exists, instead opting to store the provider in a static constant on the entity itself. This means that brains are now completely constructed through the `makeBrain` method, taking in the previous `$Packed` data:

```java
// For some ExampleEntity extends LivingEntity
// Assume extends Mob subclass
private static final Brain.Provider<ExampleEntity> BRAIN_PROVIDER = Brain.provider(
    // The list of sensors the entity uses.
    ImmutableList.of(),
    // A function that takes in the entity and returns a list of activities.
    entity -> List.of(
        new ActivityData(
            // The activity type
            Activity.CORE,
            // A list of priority and behavior pairs
            ImmutableList.of(Pair.of(0, new MoveToTargetSink())),
            // A set of memory conditions for the activity to run
            // For example, this memory value must be present
            ImmutableSet.of(Pair.of(MemoryModuleType.ATTACK_TARGET, MemoryStatus.VALUE_PRESENT)),
            // The set of memories to erase when the activity has stopped
            ImmutableSet.of(MemoryModuleType.ATTACK_TARGET)
        )
    )
);

@Override
protected Brain.Provider<ExampleEntity> makeBrain(Brain.Packed packedBrain) {
    // Make the brain, populating any previous memories
    return BRAIN_PROVIDER.makeBrain(this, packedBrain);
}
```

- `net.minecraft.world.entity.LivingEntity`
    - `brainProvider` is removed
    - `makeBrain` now takes in a `Brain$Packed` instead of a `Dynamic`
- `net.minecraft.world.entity.ai`
    - `ActivityData` - A record containing the activity being performed, the behaviors to perform during that activity, any memory conditions, and what memories to erase once stopped.
    - `Brain` is now protected, taking in a list of `ActivityData`, a `MemoryMap` instead of an immutable list of memories, and not the supplied `Codec`
        - The public constructor no longer takes in anything
        - `provider` now has an overload that only takes in the sensor types, defaulting the memory types to an empty list
            - Some `provider` methods also take in the `Brain$ActivitySupplier` to perform
        - `codec`, `serializeStart` are replaced by `pack`, `Brain$Packed`
        - `addActivityAndRemoveMemoryWhenStopped`, `addActivityWithConditions` merged into `addActivity`
            - Alternatively use `ActivityData#create`
        - `coptWithoutBehaviors` is removed
        - `$ActivitySupplier` - Creates a list of activities for the entity.
        - `$MemoryValue` is removed
        - `$Packed` - A record containing the data to serialize the brain to disk.
        - `$Provider#makeBrain` now takes in the entity and the `Brain$Packed` to deserialize the memories
- `net.minecraft.world.entity.ai.behavior.VillagerGoalPackages#get*Package` no longer take in the `VillagerProfession`
- `net.minecraft.world.entity.ai.memory.MemoryMap` - A map linking the memory type to its stored value.
- `net.minecraft.world.entity.animal.allay.AllayAi`
    - `SENSOR_TYPES`, `MEMORY_TYPES` -> `BRAIN_PROVIDER`, now private; not one-to-one
    - `makeBrain` -> `getActivities`, not one-to-one
- `net.minecraft.world.entity.animal.armadillo.ArmadilloAi#makeBrain`, `brainProvider` -> `getActivities`, now `protected`, not one-to-one
- `net.minecraft.world.entity.animal.axolotl`
    - `AxolotlSENSOR_TYPES` -> `BRAIN_PROVIDER`, now private; not one-to-one
    - `AxolotlAi`
        - `makeBrain` -> `getActivities`, not one-to-one
        - `initPlayDeadActivity`, now `protected`, no longer taking in anything
        - `initFightActivity`, now `protected`, no longer taking in anything
        - `initCoreActivity`, now `protected`, no longer taking in anything
        - `initIdleActivity`, now `protected`, no longer taking in anything
- `net.minecraft.world.entity.animal.camel.CamelAi#makeBrain`, `brainProvider` -> `getActivities`, now `protected`, not one-to-one
- `net.minecraft.world.entity.animal.frog`
    - `Frog#SENSOR_TYPES` -> `BRAIN_PROVIDER`, now private; not one-to-one
    - `FrogAi#makeBrain` -> `getActivities`, not one-to-one
    - `Tadpole#SENSOR_TYPES` -> `BRAIN_PROVIDER`, now private; not one-to-one
- `net.minecraft.world.entity.animal.frog.TadpoleAi#makeBrain` -> `getActivities`, now `public`, not one-to-one
- `net.minecraft.world.entity.animal.goat`
    - `Goat#SENSOR_TYPES` -> `BRAIN_PROVIDER`, now private; not one-to-one
    - `GoatAi#makeBrain` -> `getActivities`, not one-to-one
- `net.minecraft.world.entity.animal.golem.CopperGolemAi#makeBrain`, `brainProvider` -> `getActivities`, now `protected`, not one-to-one
- `net.minecraft.world.entity.animal.happyghast.HappyGhastAi#makeBrain`, `brainProvider` -> `getActivities`, now `protected`, not one-to-one
- `net.minecraft.world.entity.animal.nautilus`
    - `NautilusAi`
        - `SENSOR_TYPES`,`MEMORY_TYPES` -> `Nautilus#BRAIN_PROVIDER`, now private, not one-to-one
        - `makeBrain`, `brainProvider` -> `getActivities`, now `public`, not one-to-one
    - `ZombieNautilusAi`
        - `SENSOR_TYPES`,`MEMORY_TYPES` -> `ZombieNautilus#BRAIN_PROVIDER`, now private, not one-to-one
        - `makeBrain`, `brainProvider` -> `getActivities`, now `public`, not one-to-one
- `net.minecraft.world.entity.animal.sniffer.SnifferAi#makeBrain` -> `getActivities`, now `public`, not one-to-one
- `net.minecraft.world.entity.monster.Zoglin`
    - `SENSOR_TYPES` -> `BRAIN_PROVIDER`, now private, not one-to-one
    - `getActivities` - The activities the zoglin performs.
- `net.minecraft.world.entity.monster.breeze.BreezeAi#makeBrain` -> `getActivities`, not one-to-one
- `net.minecraft.world.entity.monster.creaking.CreakingAi#makeBrain`, `brainProvider` -> `getActivities`, now `protected`, not one-to-one
- `net.minecraft.world.entity.monster.hoglin`
    - `Hoglin#SENSOR_TYPES` -> `BRAIN_PROVIDER`, now private; not one-to-one
    - `HoglinAi#makeBrain` -> `getActivities`, not one-to-one
- `net.minecraft.world.entity.monster.piglin`
    - `Piglin#SENSOR_TYPES`,`MEMORY_TYPES` -> `BRAIN_PROVIDER`, now private, not one-to-one
    - `PiglinAi#makeBrain` -> `getActivities`, now `public`, not one-to-one
    - `PiglinBrute#SENSOR_TYPES`,`MEMORY_TYPES` -> `BRAIN_PROVIDER`, now private, not one-to-one
    - `PiglinBruteAi#makeBrain` -> `getActivities`, now `public`, not one-to-one
- `net.minecraft.world.entity.monster.warden.WardenAi#makeBrain` -> `getActivities`, not one-to-one

### Blaze3d Backends

`CommandEncoder`, `GpuDevice`, and `RenderPassBackend` has been split into the `*Backend` interface, which functions similarly to the previous interface, and the wrapper class, which holds the backend and provides delegate calls, performing any validation necessary. The `*Backend` interfaces now explicitly perform the operation without checking whether the operation is valid.

- `com.mojang.blaze3d.opengl`
    - `GlCommandEncoder` now implements `CommandEncoderBackend` instead of `CommandEncoder`, the class now package-private
        - `getDevice` is removed
    - `GlDevice` now implements `GpuDeviceBackend` instead of `GpuDevice`, the class now package-private
    - `GlRenderPass` now implements `RenderPassBackend` instead of `RenderPass`, the class now package-private
        - The constructor now takes in the `GlDevice`
- `com.mojang.blaze3d.systems`
    - `CommandEncoder` -> `CommandEncoderBackend`
        - The original interface is now a class wrapper around the interface, delegating to the backend after performing validation checks
     - `GpuDevice` -> `GpuDeviceBackend`
        - The original interface is now a class wrapper around the interface, delegating to the backend after performing validation checks
     - `RenderPass` -> `RenderPassBackend`
        - The original interface is now a class wrapper around the interface, delegating to the backend after performing validation checks

### Specific Logic Changes

- Some shaders within `assets/minecraft/shaders/core` now use `texture` over `texelFetch`.
    - `entity.vsh`
    - `rendertype_entity_decal.vsh`
    - `rendertype_item_entity_translucent_cull.vsh`
    - `rendertype_leash.vsh`
    - `rendertype_text.vsh`
    - `rendertype_text_background.vsh`
    - `rendertype_text_intensity.vsh`
    - `rendertype_translucent_moving_block.vsh`
- Picture-In-Picture submission calls are now taking in `0xF000F0` instead of `0x000000` for the light coordinates.
- `net.minecraft.client.multiplayer.RegistryDataCollector#collectGameRegistries` `boolean` parameter now handles only updating components from synchronized registries along with tags.
- `net.minecraft.client.renderer.RenderPipelines#VIGNETTE` now blends the alpha with a source of zero and a destination of one.
- `net.minecraft.world.entity.EntitySelector#CAN_BE_PICKED` can now find entities in spectator mode, assuming `Entity#isPickable` is true.
- `net.minecraft.world.entity.ai.sensing.NearestVisibleLivingEntitySensor#requires` is no longer implemented by default.

### Environment Attribute Additions

- `visual/block_light_tint` - Tints the color of the light emitted by a block.
- `visual/night_vision_color` - The color when night vision is active.
- `visual/ambient_light_color` - The color of the ambient light in an environment.

### Tag Changes

- `minecraft:block`
    - `bamboo_plantable_on` -> `supports_bamboo`
    - `mushroom_grow_block` -> `overrides_mushroom_light_requirement`
    - `small_dripleaf_placeable` -> `supports_small_dripleaf`
    - `big_dripleaf_placeable` -> `supports_big_dripleaf`
    - `dry_vegetation_may_place_on` -> `supports_dry_vegetation`
    - `snow_layer_cannot_survive_on` -> `cannot_support_snow_layer`
    - `snow_layer_can_survive_on` -> `support_override_snow_layer`
    - `enables_bubble_column_drag_down`
    - `enables_bubble_column_push_up`
    - `supports_vegetation`
    - `supports_crops`
    - `supports_stem_crops`
    - `supports_stem_fruit`
    - `supports_pumpkin_stem`
    - `supports_melon_stem`
    - `supports_pumpkin_stem_fruit`
    - `supports_melon_stem_fruit`
    - `supports_sugar_cane`
    - `supports_sugar_cane_adjacently`
    - `supports_cactus`
    - `supports_chorus_plant`
    - `supports_chorus_flower`
    - `supports_nether_sprouts`
    - `supports_azalea`
    - `supports_warped_fungus`
    - `supports_crimson_fungus`
    - `supports_mangrove_propagule`
    - `supports_hanging_mangrove_propagule`
    - `supports_nether_wart`
    - `supports_crimson_roots`
    - `supports_warped_roots`
    - `supports_wither_rose`
    - `supports_cocoa`
    - `supports_lily_pad`
    - `supports_frogspawn`
    - `support_override_cactus_flower`
    - `cannot_support_seagrass`
    - `cannot_support_kelp`
    - `grows_crops`
- `minecraft:enchantment`
    - `trades/desert_special` is removed
    - `trades/jungle_special` is removed
    - `trades/plains_special` is removed
    - `trades/savanna_special` is removed
    - `trades/snow_special` is removed
    - `trades/swamp_special` is removed
    - `trades/taiga_special` is removed
- `minecraft:entity_type`
    - `cannot_be_age_locked`
- `minecraft:fluid`
    - `supports_sugar_cane_adjacently`
    - `supports_lily_pad`
    - `supports_frogspawn`
    - `bubble_column_can_occupy`
- `minecraft:item`
    - `metal_nuggets`
    - `dyeable` is removed, split between:
        - `dyes`
        - `loom_dyes`
        - `loom_patterns`
        - `cauldron_can_remove_due`
        - `cat_collar_dyes`
        - `wolf_collar_dyes`
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
    - `BackendCreationException` - An exception thrown when the GPU backend couldn't be created.
    - `GpuBackend` - An interface responsible for creating the used GPU device and window to display to.
    - `GpuDevice`
        - `setVsync` - Sets whether VSync is enabled.
        - `presentFrame` - Swaps the front and back buffers of the window to display the present frame.
        - `isZZeroToOne` - Whether the 0 to 1 Z range is used instead of -1 to 1.
    - `WindowAndDevice` - A record containing the window handle and the `GpuDevice`
- `net.minecraft.advancements.criterion`
    - `FoodPredicate` - A criterion predicate that can check the food level and saturation.
    - `MinMaxBounds`
        - `validateContainedInRange` - Returns a function which validates the target range and returns as a data result.
        - `$Bounds#asRange` - Converts the bounds into a `Range`.
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
    - `DebugEntryLookingAt#getHitResult` - Gets the hit result of the camera entity.
    - `DebugEntryLookingAtEntityTags` - A debug entry for displaying an entity's tags.
    - `DebugScreenEntries#LOOKING_AT_ENTITY_TAGS` - The identifier for the entity tags debug entry.
- `net.minecraft.client.gui.navigation.FocusNavigationEvent$ArrowNavigation#with` - Sets the previous focus of the navigation.
- `net.minecraft.client.gui.screens.options
    - `DifficultyButtons` - A class containing the layout element to create the difficulty buttons.
    - `HasGamemasterPermissionReaction` - An interface marking an option screen as able to respond to changing gamemaster permissions.
    - `OptionsScreen#getLastScreen` - Returns the previous screen that navigated to this one.
    - `WorldOptionsScreen` - A screen containing the options for the current world the player is in.
- `net.minecraft.client.multiplayer.MultiPlayerGameMode#spectate` - Sends a packet to the server that the player will spectate the given entity.
- `net.minecraft.client.renderer`
    - `LightmapRenderStateExtractor` - Extracts the render state for the lightmap.
    - `UiLightmap` - The lightmap when in a user interface.
    - `RenderPipelines#LINES_DEPTH_BIAS` - A render pipeline that sets the polygon depth offset factor to -1 and the units to -1.
- `net.minecraft.client.renderer.rendertype.RenderType#hasBlending` - Whether the pipeline has a defined blend function.
- `net.minecraft.client.renderer.state`
    - `LevelRenderState#lastEntityRenderStateCount` - The number of entities being rendered to the screen.
    - `LightmapRenderState` - The render state for the lightmap.
- `net.minecraft.core.component.DataComponents#ADDITIONAL_TRADE_COST` - A modifier that offsets the trade cost by the specified amount.
- `net.minecraft.core.component.predicates`
    - `DataComponentPredicates#VILLAGER_VARIANT` - A predicate that checks a villager type.
    - `VillagerTypePredicate` - A predicate that checks a villager's type.
- `net.minecraft.core.dispenser.SpawnEggItemBehavior` - The dispenser behavior for spawn eggs.
- `net.minecraft.core.registries.ConcurrentHolderGetter` - A getter that reads references from a local cache, synchronizing to the original when necessary.
- `net.minecraft.data`
    - `BlockFamilies`
        - `END_STONE` - A family for end stone variants.
        - `getFamily` - Gets the family for the base block, when present.
    - `BlockFamily`
        - `$Builder#tiles`, `$Variant#TILES` - The block that acts as the tiles variant for some base block.
        - `$Builder#bricks`, `$Variant#BRICKS` - The block that acts as the bricks variant for some base block.
    - `DataGenerator$Uncached` - A data generator which does not cache any information about the files generated.
- `net.minecraft.data.recipes.RecipeProvider`
    - `bricksBuilder`, `tilesBuilder` - Builders for the bricks and tiles block variants.
    - `generateCraftingRecipe`, `generateStonecutterRecipe` - Generates the appropriate recipes for the given block family.
    - `getBaseBlock` -> `getBaseBlockForCrafting`
    - `bredAnimal` - Unlocks the recipe if the player has bred two animals together.
- `net.minecraft.gametest.framework`
    - `GameTestEvent#createWithMinimumDelay` - Creates a test event with some minimum delay.
    - `GameTestHelper`
        - `getBoundsWithPadding` - Gets the bounding box of the test area with the specified padding.
        - `runBeforeTestEnd` - Runs the runnable at one tick before the test ends.
    - `GameTestInstance#padding` - The number of blocks spaced around each game test.
    - `GameTestSequence#thenWaitAtLeast` - Waits for at least the specified number of ticks before running the runnable.
- `net.minecraft.network.protocol.game`
    - `ClientboundGameRuleValuesPacket` - A packet that sends the game rule values in string form to the client.
    - `ClientboundGamePacketListener#handleGameRuleValues` - Handles the game rule values sent from the server.
    - `ClientboundLowDiskSpaceWarningPacket` - A packet sent to the client warning about low disk space on the machine.
    - `ClientGamePacketListener#handleLowDiskSpaceWarning` - Handles the warning packet about low disk space.
    - `ServerboundAttackPacket` - A packet sent to the server about what entity the player attacked.
    - `ServerboundClientCommandPacket$Action#REQUEST_GAMERULE_VALUES` - Requests the game rule values from the server.
    - `ServerboundSetGameRulePacket` - A packet that sends the game rules entries to set from the client.
    - `ServerboundSpectateEntityPacket` - A packet sent to the server about what entity the player wants to spectate.
    - `ServerGamePacketListener`
        - `handleAttack` - Handles the player's attack on an entity.
        - `handleSpectateEntity` - Handles the player wanting to spectate an entity.
        - `handleSetGameRule` - Handles setting the game rules from the client.
- `net.minecraft.server.MinecraftServer`
    - `warnOnLowDiskSpace` - Sends a warning if the disk space is below 64 MiB.
    - `sendLowDiskSpaceWarning` - Sends a warning for low disk space.
    - `getGlobalGameRules` - Gets the game rules for the overworld dimension.
- `net.minecraft.server.commands.SwingCommand` - A command that calls `LivingEntity#swing` for all targets.
- `net.minecraft.server.packs.AbstractPackResources#loadMetadata` - Loads the root `pack.mcmeta`.
- `net.minecraft.server.packs.resources.ResourceMetadata$MapBased` - A resource metadata that stores the sections in a map.
- `net.minecraft.tags.TagLoader$ElementLookup#fromGetters` - Constructs an element lookup from the given `HolderGetter`s depending on whether the element is required or not.
- `net.minecraft.util`
    - `LightCoordsUtil` - A utility for determining the light coordinates from light values.
    - `ProblemReporter$MapEntryPathElement` - A path element for some entry key in a map.
- `net.minecraft.world.InteractionHand#STREAM_CODEC` - The network codec for the interaction hand.
- `net.minecraft.world.entity`
    - `AgeableMob`
        - `AGE_LOCK_COOLDOWN_TICKS` - The number of ticks to wait before the entity can be age locked/unlocked.
        - `ageLockParticleTimer` - The time for displaying particles while age locking/unlocking.
    - `Entity$Flags` - An annotation that marks a particular value as a bitset of flags for an entity.
    - `LivingEntity#getLiquidCollisionShape` - Return's the bounds of the entity when attempting to collide with a liquid.
    - `Mob`
        - `asValidTarget` - Checks whether an entity is a valid target (can attack) for this entity.
        - `getTargetUnchecked` - Gets the raw target without checking if its valid.
        - `canAgeUp` - If the entity can grow up into a more mature variant.
        - `setAgeLocked`, `isAgeLocked` - Whether the entity cannot grow up.
    - `NeutralMob#getTargetUnchecked` - Gets the raw target without checking if its valid.
    - `TamableAnimal#feed` - The player gives the stack as food, healing them either based on the stored component or some default value.
- `net.minecraft.world.entity.ai.behavior`
    - `BehaviorControl#getRequiredMemories` - The list of memories required by the behavior.
    - `GoAndGiveItemsToTarget$ItemThrower` - An interface that handles what should occur if an item is thrown because of this entity's behavior.
- `net.minecraft.world.entity.ai.behavior.declarative.BehaviorBuilder$TriggerWithResult#memories` - The list of memories required by the behavior.
- `net.minecraft.world.entity.ai.memory.NearestVisibleLivingEntities#nearbyEntities` - The list of entities within the follow range of this one.
- `net.minecraft.world.entity.monster.piglin`
    - `PiglinAi`
        - `MAX_TIME_BETWEEN_HUNTS` - The maximum number of seconds before a piglin begins to hunt again.
        - `findNearbyAdultPiglins` - Returns a list of all adult piglins in this piglin's memory.
- `net.minecraft.world.entity.raid.Raid`
    - `getBannerComponentPatch` - Gets the components of the banner pattern.
    - `getOminousBannerTemplate` - Gets the stack template for the omnious banner.
- `net.minecraft.world.inventory.SlotRanges`
    - `MOB_INVENTORY_SLOT_OFFSET` - The start index of a mob's inventory.
    - `MOB_INVENTORY_SIZE` - The size of a mob's inventory.
- `net.minecraft.world.item.component.BundleContents#BEEHIVE_WEIGHT` - The weight of a beehive.
- `net.minecraft.world.item.enchantment.EnchantmentTarget#NON_DAMAGE_CODEC` - A codec that only allows the attacker and victim enchantment targets.
- `net.minecraft.world.level.dimension.DimensionDefaults`
    - `BLOCK_LIGHT_TINT` - The default tint for the block light.
    - `NIGHT_VISION_COLOR` - The default color when in night vision.
    - `TURTLE_EGG_HATCH_CHANCE` - The chance of a turtle egg hatching on a random tick.
- `net.minecraft.world.level.material`
    - `FluidState#isFull` - If the fluid amount is eight.
    - `LavaFluid#LIGHT_EMISSION` - The amount of light the lava fluid emits.
- `net.minecraft.world.level.storage.loot.functions`
    - `EnchantRandomlyFunction$Builder#withOptions` - Specifies the enchantments that can be used to randomly enchant this item.
    - `SetRandomDyesFunction` - An item function that applies a random dye (if the item is in the `dyeable` tag) to the `DYED_COLOR` component from the provided list.
    - `SetRandomPotionFunction` - An item function that applies a random potion to the `POTION_CONTENTS` component from the provided list.
- `net.minecraft.world.level.storage.loot.parameters.LootContextParams#ADDITIONAL_COST_COMPONENT_ALLOWED` - Enables a trade to incur an additional cost if desired by the wanted stack or modifiers.
- `net.minecraft.world.level.storage.loot.providers.number.Sum` - A provider that sums the value of all provided number providers.

### List of Changes

- `com.mojang.blaze3d.opengl`
    - `GlDevice` now takes in a `GpuDebugOptions` containing the log level, whether to use synchronous logs, and whether to use debug labels instead of those parameters being passed in directly
    - `GlProgram#BUILT_IN_UNIFORMS`, `INVALID_PROGRAM` are now final
- `com.mojang.blaze3d.platform`
    - `ClientShutdownWatchdog` now takes in the `Minecraft` instance
    - `Window` now takes in a list of `GpuBackend`s, the default `ShaderSource`, and the `GpuDebugOptions` instead of the `ScreenManager`
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
    - `PopupScreen$Builder#setMessage` -> `addMessage`, not one-to-one
    - `ScrollableLayout$Container` now takes in an `AbstractScrollArea$ScrollbarSettings`
    - `Tooltip#create` now has an overload that takes in an optional `TooltipComponent` and style `Identifier`
- `net.minecraft.client.gui.components.debug`
    - `DebugEntryLookingAtBlock`, `DebugEntryLookingAtFluid` -> `DebugEntryLookingAt`
        - More specifically, `$BlockStateInfo`, `$BlockTagInfo`, `$FluidStateInfo`, `$FluidTagInfo`
    - `DebugEntryLookingAtEntity#GROUP` is now public
    - `DebugScreenEntries`
        - `LOOKING_AT_BLOCK` -> `LOOKING_AT_BLOCK_STATE`, `LOOKING_AT_BLOCK_TAGS`
        - `LOOKING_AT_FLUID` -> `LOOKING_AT_FLUID_STATE`, `LOOKING_AT_FLUID_TAGS`
    - `DebugScreenEntryList` now takes in a `DataFixer`
- `net.minecraft.client.gui.components.tabs.TabNavigationBar#setWidth` -> `updateWidth`, not one-to-one
- `net.minecraft.client.gui.navigation.FocusNavigationEvent$ArrowNavigation` now takes in a nullable `ScreenRectangle` for the previous focus
- `net.minecraft.client.gui.render.state.GuiItemRenderState` no longer takes in the `String` name
- `net.minecraft.client.gui.screens`
    - `ConfirmScreen#layout` is now final
    - `DemoIntroScreen` replaced by `ClientPacketListener#openDemoIntroScreen`, now private
- `net.minecraft.client.gui.screens.inventory.AbstractMountInventoryScreen#mount` is now final
- `net.minecraft.client.gui.screens.options`
    - `OptionsScreen` now implements `HasGamemasterPermissionReaction`
        - The constructor now takes in a `boolean` for if the player is currently in a world
        - `createDifficultyButton` now handles within `WorldOptionsScreen#createDifficultyButtons`, not one-to-one
    - `WorldOptionsScreen` now implements `HasGamemasterPermissionReaction`
        - `createDifficultyButtons` -> `DifficultyButtons#create`, now public
- `net.minecraft.client.gui.screens.worldselection.EditGameRulesScreen` -> `AbstractGameRulesScreen`
    - Implementations in `.screens.options.InWorldGameRulesScreen` and `WorldCreationGameRulesScreen`
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
    - `WeatherEffectRenderer#render` no longer takes in the `MultiBufferSource`
- `net.minecraft.client.renderer.block.ModelBlockRenderer$Cache#getLightColor` -> `getLightCoords`
- `net.minecraft.client.renderer.blockentity.state`
    - `BrushableBlockRenderState#itemState` is now final
    - `EndPortalRenderState` is now a final set of `Direction`s rather than an `EnumSet`
    - `ShelfRenderState#items` is now final
- `net.minecraft.client.renderer.entity.state`
    - `FallingBlockRenderState#movingBlockRenderState` is now final
    - `HumanoidRenderState#attackArm` -> `ArmedEntityRenderState#attackArm`
    - `WitherRenderState#xHeadRots`, `yHeadRots` are now final
- `net.minecraft.client.renderer.state.BlockBreakingRenderState#progress` is now final
- `net.minecraft.client.resources.sounds.AbstractSoundInstance#random` is now final
- `net.minecraft.core.WritableRegistry#bindTag` -> `bindTags`, now taking in a map of keys to holder lists instead of one mapping
- `net.minecraft.data`
    - `BlockFamily`
        - `shouldGenerateRecipe` -> `shouldGenerateCraftingRecipe`, `shouldGenerateStonecutterRecipe`
        - `$Builder#dontGenerateRecipe` -> `dontGenerateCraftingRecipe`, `generateStonecutterRecipe`
    - `DataGenerator` is now abstract
        - The constructor now only takes in the `Path` output, not the `WorldVersion` or whether to always generate
            - The original implementation can be founded in `DataGenerator$Cached`
        - `run` is now abstract
    - `Main#addServerProviders` -> `addServerDefinitionProviders`, no longer taking in the dev `boolean`, not one-to-one
    - The remaining logic has been put into `addServerConverters`, taking in the dev `boolean` but not the report `boolean`
- `net.minecraft.data.loot.BlockLootSubProvider`
    - `explosionResistant` is now private
    - `enabledFeatures` is now private
    - `map` is now private
- `net.minecraft.data.recipes`
    - `RecipeProvider$FamilyRecipeProvider` -> `$FamilyCraftingRecipeProvider`, `$FamilyStonecutterRecipeProvider`
    - `SingleItemRecipeBuilder#stonecutting` moved to a parameter on the `BlockFamily`
- `net.minecraft.gametest.framework`
    - `GameTestHelper#assertBlockPresent` now has an overload that only takes in the block to check for
    - `GameTestServer#create` now takes in an `int` for the number of times to run all matching tests
    - `TestData` now takes in an `int` for the number of blocks padding around the test
- `net.minecraft.network.FriendlyByteBuf`
    - `readVec3`, `writeVec3` replaced with `Vec3#STREAM_CODEC`
    - `readLpVec3`, `writeLpVec3` replaced with `Vec3#LP_STREAM_CODEC`
- `net.minecraft.network.chat.LastSeenMessages#EMPTY` is now final
- `net.minecraft.network.protocol.game`
    - `ClientboundSetEntityMotionPacket` is now a record
    - `ServerboundContainerClickPacket` now takes in a `ContainerInput` instead of a `ClickType`
    - `ServerboundInteractPacket` is now a record, now taking in the `Vec3` interaction location
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
    - `loadTagsForRegistry` now has an overload which takes in registry key along with an element lookup, returning a map of tag keys to a list of holder entries
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
        - `canAttackType` -> `canAttack`, not one-to-one, taking in the `LivingEntity` instead of the `EntityType`
        - `lungeForwardMaybe` -> `postPiercingAttack`
        - `entityAttackRange` -> `getAttackRangeWith`, now taking in the `ItemStack` used to attack
- `net.minecraft.world.entity.ai.behavior`
    - `GoAndGiveItemsToTarget` now takes in the `$ItemThrower`
        - `throwItem` -> `BehaviorUtils#throwItem`, not one-to-one
    - `SpearAttack` no longer takes in the approach distance `float`
    - `TryLaySpawnOnWaterNearLand` -> `TryLaySpawnOnFluidNearLand`, not one-to-one
- `net.minecraft.world.entity.ai.behavior.declarative.BehaviorBuilder#sequence` now takes in a `Oneshot` for the second entry instead of a `Trigger`
- `net.minecraft.world.entity.ai.goal`
    - `BoatGoals` -> `FollowPlayerRiddenEntityGoal$FollowEntityGoal`
        - `BOAT` is replaced by `ENTITY`
    - `FollowBoatGoal` -> `FollowPlayerRiddenEntityGoal`, not one-to-one
- `net.minecraft.world.entity.ai.goal.target.NearestAttackableTargetGoal#targetConditions` is now final
- `net.minecraft.world.entity.ai.sensing.NearestVisibleLivingEntitySensor#getMemory` -> `getMemoryToSet`
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
        - `isWithinAttackRange` now takes in the `ItemStack` attacked with
- `net.minecraft.world.entity.vehicle.minecart.NewMinecartBehavior$MinecartStep#EMPTY` is now final
- `net.minecraft.world.inventory.AbstractContainerMenu#getQuickCraftPlaceCount` now takes in the number of quick craft slots instead of the set of slots itself
- `net.minecraft.world.item`
    - `BundleItem#getSelectedItem` -> `getSelectedItemIndex`
    - `CreativeModeTab$Output` is now `protected` from `public`
    - `EnderpearlItem#PROJECTILE_SHOOT_POWER` is now final
    - `Items#register*` methods are now `private` from `public`
    - `ItemUtils#onContainerDestroyed` now takes in a `Stream` of `ItemStack`s instead of an `Iterable`
    - `SignApplicator#tryApplyToSign`, `canApplyToSign` now take in the `ItemStack` being used
    - `SnowballItem#PROJECTILE_SHOOT_POWER` is now final
    - `ThrowablePotionItem#PROJECTILE_SHOOT_POWER` is now final
    - `WindChargeItem#PROJECTILE_SHOOT_POWER` is now final
- `net.minecraft.world.item.alchemy.Potions` now deal with `Holder$Reference`s instead of the super `Holder`
- `net.minecraft.world.item.component`
    - `AttackRange`
        - `minRange` -> `minReach`
        - `maxRange` -> `maxReach`
        - `minCreativeRange` -> `minCreativeReach`
        - `maxCreativeRange` -> `maxCreativeReach`
    - `BundleContents`
        - `weight` now returns a `DataResult`-wrapped `Fraction` instead of the raw object
        - `getSelectedItem` -> `getSelectedItemIndex`
    - `WrittenBookContent#tryCraftCopy` -> `craftCopy`
- `net.minecraft.world.item.enchantment`
    - `ConditionalEffect#codec` no longer takes in the `ContextKeySet`
    - `Enchantment#doLunge` -> `doPostPiercingAttack`
    - `EnchantmentHelper#doLungeEffects` -> `doPostPiercingAttackEffects`
    - `TargetedConditionalEffect#codec`, `equipmentDropsCodec` no longer take in the `ContextKeySet`
- `net.minecraft.world.item.enchantment.effects.EnchantmentAttributeEffect#CODEC` -> `MAP_CODEC`
- `net.minecraft.world.item.equipment.Equippable#canBeEquippedBy` now takes in a holder-wrapped `EntityType` instead of the raw type itself
- `net.minecraft.world.level.LevelAccessor` no longer implements `LevelReader`
- `net.minecraft.world.level.block`
    - `BubbleColumnBlock#updateColumn` now takes in the bubble column `Block`
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
- `net.minecraft.world.level.gamerules.GameRules` now has an overload that takes in a list of `GameRule`s
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
- `net.minecraft.client.gui.render.state.GuiItemRenderState#name`
- `net.minecraft.client.renderer.rendertype.RenderTypes#weather`
- `net.minecraft.data.loot.BlockLootSubProvider(Set, FeatureFlagSet, Map, HolderLookup$Provider)`
- `net.minecraft.server.packs.AbstractPackResources#getMetadataFromStream`
- `net.minecraft.util.LightCoordsUtil#UI_FULL_BRIGHT`
- `net.minecraft.world`
    - `ContainerListener`
    - `SimpleContainer#addListener`, `removeListener`
- `net.minecraft.world.entity.ai.memory.MemoryModuleType#INTERACTABLE_DOORS`
- `net.minecraft.world.entity.monster.Zoglin#MEMORY_TYPES`
- `net.minecraft.world.entity.monster.creaking.Creaking#MEMORY_TYPES`
- `net.minecraft.world.entity.monster.hogling.Hoglin#MEMORY_TYPES`
- `net.minecraft.world.item`
    - `BundleItem#hasSelectedItem`
    - `Item#getName()`
- `net.minecraft.world.item.component`
    - `BundleContents`
        - `getItemUnsafe`
        - `hasSelectedItem`
    - `WrittenBookContent#MAX_CRAFTABLE_GENERATION`
- `net.minecraft.world.level.block.LiquidBlock#SHAPE_STABLE`
- `net.minecraft.world.level.storage.loot.functions.SetOminousBottleAmplifierFunction#amplifier`
