# Item System

The Item System is the core framework in TaleServer for defining, managing, and manipulating game items. Items represent everything players can hold, use, equip, or store - from weapons and armor to consumables and building blocks. This document covers the complete API for working with items, item stacks, metadata, containers, and related systems.

## Table of Contents

- [Overview](#overview)
- [Core Classes](#core-classes)
  - [Item](#item)
  - [ItemStack](#itemstack)
  - [ItemContext](#itemcontext)
- [Item Properties](#item-properties)
  - [Basic Properties](#basic-properties)
  - [Rendering Properties](#rendering-properties)
  - [Functionality Properties](#functionality-properties)
- [Item Subtypes](#item-subtypes)
  - [ItemWeapon](#itemweapon)
  - [ItemArmor](#itemarmor)
  - [ItemTool](#itemtool)
  - [ItemGlider](#itemglider)
  - [ItemUtility](#itemutility)
- [Item Quality System](#item-quality-system)
- [Item Categories](#item-categories)
- [Item Containers](#item-containers)
  - [Container Types](#container-types)
  - [Container Operations](#container-operations)
  - [Transactions](#transactions)
- [Item Metadata](#item-metadata)
- [Item Drop System](#item-drop-system)
- [Translation & Lore](#translation--lore)
- [Creating Items](#creating-items)
  - [JSON Definition](#json-definition)
  - [Programmatic Creation](#programmatic-creation)
- [Working with Items](#working-with-items)
  - [Retrieving Items](#retrieving-items)
  - [Spawning Item Entities](#spawning-item-entities)
  - [Giving Items to Players](#giving-items-to-players)
- [Best Practices](#best-practices)

---

## Overview

The Item System provides:

- **Type Definitions**: Items define the base properties and behaviors of game objects via JSON assets
- **Instance Management**: ItemStacks represent actual instances with quantity, durability, and metadata
- **Container System**: ItemContainers manage collections of ItemStacks with transactional operations
- **Subtype Specialization**: Weapons, Armor, Tools, and Utilities have specialized configuration
- **Quality & Categories**: Hierarchical organization and visual theming for items
- **Translation Support**: Localized names and descriptions via translation keys

```
Item (Asset Definition)
├── Rendering: Model, Texture, Animation, Particles
├── Properties: MaxStack, Durability, Categories
├── Subtypes: Weapon, Armor, Tool, Glider, Utility
├── Interactions: Per-interaction-type behavior mappings
└── Translation: Name, Description localization keys

ItemStack (Runtime Instance)
├── itemId → references Item asset
├── quantity → stack count (1 to MaxStack)
├── durability → current durability value
├── maxDurability → maximum durability
└── metadata → arbitrary BSON data
```

---

## Core Classes

### Item

[`Item`](https://github.com/Savag3life/TaleServer/tree/main/hytale/main/src/com/hypixel/hytale/server/core/asset/type/item/config/Item.java) is the asset definition class that defines an item's properties, appearance, and behavior. Items are loaded from JSON files in the `items/` directory.

**Key Methods:**

| Method | Return Type | Description |
|--------|-------------|-------------|
| `getId()` | `String` | Get the unique item identifier |
| `getTranslationKey()` | `String` | Get the localization key for the item name |
| `getDescriptionTranslationKey()` | `String` | Get the localization key for description |
| `getMaxStack()` | `int` | Maximum quantity per stack |
| `getMaxDurability()` | `double` | Maximum durability (0 = unbreakable) |
| `getItem()` | `Item` | Get Item from asset registry by ID |
| `hasBlockType()` | `boolean` | Whether this item has an associated block |
| `getBlockId()` | `String` | Get the associated BlockType ID |
| `getWeapon()` | `ItemWeapon` | Get weapon configuration (nullable) |
| `getArmor()` | `ItemArmor` | Get armor configuration (nullable) |
| `getTool()` | `ItemTool` | Get tool configuration (nullable) |
| `getGlider()` | `ItemGlider` | Get glider configuration (nullable) |
| `getUtility()` | `ItemUtility` | Get utility configuration |
| `getCategories()` | `String[]` | Get creative menu categories |
| `getQualityIndex()` | `int` | Get quality tier index |
| `getInteractions()` | `Map<InteractionType, String>` | Get interaction mappings |

**Static Access:**

```java
// Get the item asset store
AssetStore<String, Item, DefaultAssetMap<String, Item>> store = Item.getAssetStore();

// Get the item asset map for direct lookups
DefaultAssetMap<String, Item> assetMap = Item.getAssetMap();

// Retrieve a specific item
Item ironSword = Item.getAssetMap().getAsset("Iron_Sword");
```

### ItemStack

[`ItemStack`](https://github.com/Savag3life/TaleServer/tree/main/hytale/main/src/com/hypixel/hytale/server/core/inventory/ItemStack.java) represents an actual instance of an item with quantity, durability, and custom metadata. ItemStacks are immutable - all modification methods return new instances.

**Constructors:**

```java
// Create a stack of 1
ItemStack stack = new ItemStack("Iron_Sword");

// Create a stack with quantity
ItemStack stack = new ItemStack("Wood_Planks", 64);

// Create with quantity and metadata
ItemStack stack = new ItemStack("Iron_Sword", 1, metadata);

// Create with full parameters
ItemStack stack = new ItemStack("Iron_Sword", 1, durability, maxDurability, metadata);
```

**Key Methods:**

| Method | Return Type | Description |
|--------|-------------|-------------|
| `getItemId()` | `String` | Get the item asset ID |
| `getItem()` | `Item` | Get the Item asset (never null, returns UNKNOWN if invalid) |
| `getQuantity()` | `int` | Get current stack quantity |
| `getDurability()` | `double` | Get current durability |
| `getMaxDurability()` | `double` | Get max durability |
| `isUnbreakable()` | `boolean` | Whether durability is infinite |
| `isBroken()` | `boolean` | Whether durability is depleted |
| `isEmpty()` | `boolean` | Whether this is the empty stack |
| `isValid()` | `boolean` | Whether the item ID resolves to a valid Item |
| `isStackableWith(ItemStack)` | `boolean` | Whether stacks can merge (same ID, durability, metadata) |
| `isEquivalentType(ItemStack)` | `boolean` | Whether same type (ignores durability) |
| `getMetadata()` | `BsonDocument` | Get custom metadata (cloned) |
| `getBlockKey()` | `String` | Get associated block ID if placeable |

**Immutable Modification Methods:**

```java
ItemStack original = new ItemStack("Iron_Sword", 1);

// Modify quantity (returns null if quantity becomes 0)
ItemStack doubled = original.withQuantity(2);
ItemStack half = original.withQuantity(original.getQuantity() / 2);

// Modify durability
ItemStack damaged = original.withDurability(50.0);
ItemStack repaired = original.withIncreasedDurability(10.0);
ItemStack fullyRepaired = original.withRestoredDurability(100.0);
ItemStack newMax = original.withMaxDurability(200.0);

// Modify metadata
ItemStack withMeta = original.withMetadata(new BsonDocument());
ItemStack withTypedMeta = original.withMetadata("CustomKey", Codec.STRING, "CustomValue");
ItemStack withRawMeta = original.withMetadata("Key", new BsonString("value"));

// Change state (for items with state variants)
ItemStack stateVariant = original.withState("Charged");
```

**Static Utility Methods:**

```java
// Check if stack is null or empty
boolean empty = ItemStack.isEmpty(stack);

// Check stackability between two stacks
boolean canStack = ItemStack.isStackableWith(stackA, stackB);

// Check type equivalence
boolean sameType = ItemStack.isEquivalentType(stackA, stackB);

// Check if same item type (ignores metadata/durability)
boolean sameItem = ItemStack.isSameItemType(stackA, stackB);

// Create from network packet
ItemStack fromPacket = ItemStack.fromPacket(itemQuantityPacket);
```

### ItemContext

[`ItemContext`](https://github.com/Savag3life/TaleServer/tree/main/hytale/main/src/com/hypixel/hytale/server/core/inventory/ItemContext.java) provides context about where an ItemStack is located within a container.

```java
public class ItemContext {
    public ItemContainer getContainer();  // The containing inventory
    public short getSlot();               // The slot index
    public ItemStack getItemStack();      // The item stack
}
```

---

## Item Properties

### Basic Properties

| Property | Type | Default | Description |
|----------|------|---------|-------------|
| `Id` | `String` | Required | Unique identifier for the item |
| `Parent` | `String` | null | Parent item to inherit from |
| `MaxStack` | `int` | 100 (or 1 for tools/weapons) | Maximum stack size |
| `MaxDurability` | `double` | 0 | Max durability (0 = unbreakable) |
| `DurabilityLossOnHit` | `double` | 0 | Durability lost per hit |
| `Consumable` | `boolean` | false | Whether item is consumed on use |
| `Variant` | `boolean` | false | Whether to hide from creative library |
| `DropOnDeath` | `boolean` | false | Whether dropped when player dies |
| `FuelQuality` | `double` | 1.0 | Fuel efficiency multiplier |
| `Categories` | `String[]` | null | Creative menu categories |
| `Quality` | `String` | "Default" | Quality tier ID |
| `Set` | `String` | null | Equipment set identifier |
| `ItemLevel` | `int` | 0 | Item power level |

### Rendering Properties

| Property | Type | Description |
|----------|------|-------------|
| `Icon` | `String` | Icon texture path |
| `Model` | `String` | 3D model path (`.blockymodel`) |
| `Texture` | `String` | Item texture path |
| `Animation` | `String` | Item animation path |
| `Scale` | `float` | Render scale (default 1.0) |
| `PlayerAnimationsId` | `String` | Player animation set ID |
| `UsePlayerAnimations` | `boolean` | Whether to use player animations |
| `DroppedItemAnimation` | `String` | Animation when dropped |
| `Particles` | `ModelParticle[]` | Particle effects |
| `FirstPersonParticles` | `ModelParticle[]` | First-person particle effects |
| `Trails` | `ModelTrail[]` | Trail effects |
| `Light` | `ColorLight` | Light emission configuration |
| `Reticle` | `String` | Custom reticle configuration |
| `ClipsGeometry` | `boolean` | Whether item clips through geometry |
| `RenderDeployablePreview` | `boolean` | Show placement preview |

### Functionality Properties

| Property | Type | Description |
|----------|------|-------------|
| `Interactions` | `Map<InteractionType, String>` | Interaction handlers per type |
| `InteractionConfig` | `InteractionConfiguration` | Interaction configuration |
| `InteractionVars` | `Map<String, String>` | Interaction variables |
| `ItemAppearanceConditions` | `Map<String, ItemAppearanceCondition[]>` | Dynamic appearance changes |
| `DisplayEntityStatsHUD` | `String[]` | Stats to display on HUD |
| `PullbackConfig` | `ItemPullbackConfig` | First-person arm pullback config |
| `ResourceTypes` | `ItemResourceType[]` | Resource type mappings |
| `Recipe` | `CraftingRecipe` | Auto-generated crafting recipe |

---

## Item Subtypes

### ItemWeapon

[`ItemWeapon`](https://github.com/Savag3life/TaleServer/tree/main/hytale/main/src/com/hypixel/hytale/server/core/asset/type/item/config/ItemWeapon.java) defines weapon-specific properties.

```json
{
    "Weapon": {
        "StatModifiers": {
            "AttackDamage": [
                { "Target": "Base", "CalculationType": "Add", "Amount": 15.0 }
            ],
            "AttackSpeed": [
                { "Target": "Base", "CalculationType": "Multiply", "Amount": 1.2 }
            ]
        },
        "EntityStatsToClear": ["Poison", "Burn"],
        "RenderDualWielded": true
    }
}
```

| Property | Type | Description |
|----------|------|-------------|
| `StatModifiers` | `Map<String, StaticModifier[]>` | Stat modifiers when equipped |
| `EntityStatsToClear` | `String[]` | Stats to clear on hit |
| `RenderDualWielded` | `boolean` | Render in both hands |

### ItemArmor

[`ItemArmor`](https://github.com/Savag3life/TaleServer/tree/main/hytale/main/src/com/hypixel/hytale/server/core/asset/type/item/config/ItemArmor.java) defines armor-specific properties.

```json
{
    "Armor": {
        "ArmorSlot": "Chest",
        "BaseDamageResistance": 10.0,
        "DamageResistance": {
            "Physical": [
                { "Target": "Base", "CalculationType": "Add", "Amount": 5.0 }
            ],
            "Fire": [
                { "Target": "Base", "CalculationType": "Add", "Amount": 10.0 }
            ]
        },
        "DamageEnhancement": {
            "Magic": [
                { "Target": "Base", "CalculationType": "Multiply", "Amount": 1.1 }
            ]
        },
        "KnockbackResistances": {
            "Physical": 0.2
        },
        "StatModifiers": {
            "MaxHealth": [
                { "Target": "Base", "CalculationType": "Add", "Amount": 20.0 }
            ]
        },
        "Regenerating": {
            "Health": [
                { "Value": 1.0, "Interval": 5.0 }
            ]
        },
        "CosmeticsToHide": ["Cape", "Hair"]
    }
}
```

| Property | Type | Description |
|----------|------|-------------|
| `ArmorSlot` | `ItemArmorSlot` | Slot: `Head`, `Chest`, `Legs`, `Feet` |
| `BaseDamageResistance` | `double` | Flat damage reduction |
| `DamageResistance` | `Map<DamageCause, StaticModifier[]>` | Per-damage-type resistance |
| `DamageEnhancement` | `Map<DamageCause, StaticModifier[]>` | Per-damage-type bonus |
| `DamageClassEnhancement` | `Map<DamageClass, StaticModifier[]>` | Per-damage-class bonus |
| `KnockbackResistances` | `Map<DamageCause, Float>` | Knockback reduction |
| `KnockbackEnhancements` | `Map<DamageCause, Float>` | Knockback bonus |
| `StatModifiers` | `Map<String, StaticModifier[]>` | Stat modifiers when equipped |
| `Regenerating` | `Map<String, Regenerating[]>` | Passive regeneration |
| `InteractionModifiers` | `Map<String, Map<String, StaticModifier>>` | Per-interaction stat modifiers |
| `CosmeticsToHide` | `Cosmetic[]` | Cosmetics to hide when worn |

### ItemTool

[`ItemTool`](https://github.com/Savag3life/TaleServer/tree/main/hytale/main/src/com/hypixel/hytale/server/core/asset/type/item/config/ItemTool.java) defines tool-specific properties for block breaking.

```json
{
    "Tool": {
        "Speed": 2.0,
        "Specs": [
            {
                "BlockSets": ["Wood"],
                "SpeedMultiplier": 4.0,
                "DamageMultiplier": 1.0
            },
            {
                "BlockTypes": ["Stone", "Cobblestone"],
                "SpeedMultiplier": 2.0,
                "DamageMultiplier": 0.5
            }
        ],
        "DurabilityLossBlockTypes": [
            {
                "BlockSets": ["Stone"],
                "DurabilityLossOnHit": 2.0
            }
        ],
        "HitSoundLayer": "SE_Tool_Axe_Hit",
        "IncorrectMaterialSoundLayer": "SE_Tool_Wrong_Material"
    }
}
```

| Property | Type | Description |
|----------|------|-------------|
| `Speed` | `float` | Base mining speed |
| `Specs` | `ItemToolSpec[]` | Per-block-type specifications |
| `DurabilityLossBlockTypes` | `DurabilityLossBlockTypes[]` | Custom durability loss |
| `HitSoundLayer` | `String` | Sound when hitting correct material |
| `IncorrectMaterialSoundLayer` | `String` | Sound when hitting wrong material |

### ItemGlider

[`ItemGlider`](https://github.com/Savag3life/TaleServer/tree/main/hytale/main/src/com/hypixel/hytale/server/core/asset/type/item/config/ItemGlider.java) defines glider-specific properties.

```json
{
    "Glider": {
        "TerminalVelocity": 8.0,
        "FallSpeedMultiplier": 0.5,
        "HorizontalSpeedMultiplier": 1.5,
        "Speed": 12.0
    }
}
```

| Property | Type | Description |
|----------|------|-------------|
| `TerminalVelocity` | `float` | Maximum fall speed |
| `FallSpeedMultiplier` | `float` | Fall acceleration rate |
| `HorizontalSpeedMultiplier` | `float` | Horizontal acceleration rate |
| `Speed` | `float` | Horizontal movement speed |

### ItemUtility

[`ItemUtility`](https://github.com/Savag3life/TaleServer/tree/main/hytale/main/src/com/hypixel/hytale/server/core/asset/type/item/config/ItemUtility.java) defines utility item properties.

```json
{
    "Utility": {
        "Usable": true,
        "Compatible": true,
        "StatModifiers": {
            "MoveSpeed": [
                { "Target": "Base", "CalculationType": "Multiply", "Amount": 1.2 }
            ]
        },
        "EntityStatsToClear": ["Slow"]
    }
}
```

| Property | Type | Description |
|----------|------|-------------|
| `Usable` | `boolean` | Whether item can be used |
| `Compatible` | `boolean` | Whether compatible with other items |
| `StatModifiers` | `Map<String, StaticModifier[]>` | Stat modifiers when active |
| `EntityStatsToClear` | `String[]` | Stats to clear on use |

---

## Item Quality System

[`ItemQuality`](https://github.com/Savag3life/TaleServer/tree/main/hytale/main/src/com/hypixel/hytale/server/core/asset/type/item/config/ItemQuality.java) defines visual tiers for items (Common, Rare, Legendary, etc.).

```json
{
    "Id": "Legendary",
    "QualityValue": 4,
    "ItemTooltipTexture": "UI/ItemQualities/Tooltips/ItemTooltipLegendary.png",
    "ItemTooltipArrowTexture": "UI/ItemQualities/Tooltips/ItemTooltipLegendaryArrow.png",
    "SlotTexture": "UI/ItemQualities/Slots/SlotLegendary.png",
    "BlockSlotTexture": "UI/ItemQualities/Slots/SlotLegendary.png",
    "SpecialSlotTexture": "UI/ItemQualities/Slots/SpecialSlotLegendary.png",
    "TextColor": "#FFD700",
    "LocalizationKey": "server.general.qualities.Legendary",
    "VisibleQualityLabel": true,
    "RenderSpecialSlot": true,
    "HideFromSearch": false,
    "ItemEntityConfig": {
        "ParticleSystemId": "LegendaryItem",
        "ShowItemParticles": true
    }
}
```

**API Usage:**

```java
// Get quality asset map
IndexedLookupTableAssetMap<String, ItemQuality> qualityMap = ItemQuality.getAssetMap();

// Get quality by ID
ItemQuality legendary = qualityMap.getAsset("Legendary");

// Get quality from item
Item item = Item.getAssetMap().getAsset("Legendary_Sword");
int qualityIndex = item.getQualityIndex();
ItemQuality quality = qualityMap.getAsset(qualityIndex);

// Quality properties
String name = quality.getLocalizationKey();
Color textColor = quality.getTextColor();
int value = quality.getQualityValue(); // For sorting
```

---

## Item Categories

[`ItemCategory`](https://github.com/Savag3life/TaleServer/tree/main/hytale/main/src/com/hypixel/hytale/server/core/asset/type/item/config/ItemCategory.java) organizes items in the creative menu.

```json
{
    "Id": "Weapons",
    "Name": "server.categories.weapons",
    "Icon": "Icons/Categories/Weapons.png",
    "Order": 1,
    "InfoDisplayMode": "Tooltip",
    "Children": [
        {
            "Id": "Swords",
            "Name": "server.categories.swords",
            "Icon": "Icons/Categories/Swords.png",
            "Order": 1
        },
        {
            "Id": "Bows",
            "Name": "server.categories.bows",
            "Icon": "Icons/Categories/Bows.png",
            "Order": 2
        }
    ]
}
```

**Assign items to categories:**

```json
{
    "Id": "Iron_Sword",
    "Categories": ["Weapons", "Swords", "Metal"]
}
```

---

## Item Containers

### Container Types

[`ItemContainer`](https://github.com/Savag3life/TaleServer/tree/main/hytale/main/src/com/hypixel/hytale/server/core/inventory/container/ItemContainer.java) is the abstract base for all item storage.

| Implementation | Use Case |
|----------------|----------|
| `SimpleItemContainer` | Basic fixed-size storage |
| `ItemStackItemContainer` | Single-item storage |
| `CombinedItemContainer` | Multiple containers as one |
| `DelegateItemContainer` | Wrapper/proxy container |
| `EmptyItemContainer` | Zero-capacity container |

**Creating Containers:**

```java
// Simple container with 36 slots
ItemContainer inventory = new SimpleItemContainer((short) 36);

// Combined container (hotbar + main inventory)
ItemContainer combined = new CombinedItemContainer(hotbar, mainInventory);

// Get container from player
Player player = ...;
ItemContainer playerInventory = player.getInventory().getCombinedHotbarFirst();
```

### Container Operations

**Adding Items:**

```java
// Add to any available slot
ItemStackTransaction result = container.addItemStack(itemStack);
if (result.succeeded()) {
    ItemStack remainder = result.getRemainder(); // null if fully added
}

// Add to specific slot
ItemStackSlotTransaction result = container.addItemStackToSlot((short) 0, itemStack);

// Add multiple items
List<ItemStack> items = List.of(item1, item2, item3);
ListTransaction<ItemStackTransaction> results = container.addItemStacks(items);

// Check if items can be added first
if (container.canAddItemStack(itemStack)) {
    container.addItemStack(itemStack);
}
```

**Removing Items:**

```java
// Remove from specific slot
SlotTransaction result = container.removeItemStackFromSlot((short) 5);

// Remove specific quantity from slot
ItemStackSlotTransaction result = container.removeItemStackFromSlot((short) 5, 10);

// Remove matching item from anywhere
ItemStackTransaction result = container.removeItemStack(targetStack);

// Remove by tag
TagTransaction result = container.removeTag(tagIndex, quantity);

// Remove by material
MaterialTransaction result = container.removeMaterial(materialQuantity);

// Remove by resource type
ResourceTransaction result = container.removeResource(resourceQuantity);
```

**Moving Items:**

```java
// Move from slot to another container
MoveTransaction<ItemStackTransaction> result = container.moveItemStackFromSlot(
    (short) 0,
    targetContainer
);

// Move to specific slot
MoveTransaction<SlotTransaction> result = container.moveItemStackFromSlotToSlot(
    (short) 0,
    quantity,
    targetContainer,
    (short) 5
);

// Move all items
ListTransaction<MoveTransaction<ItemStackTransaction>> result = container.moveAllItemStacksTo(
    targetContainer
);

// Quick stack (move only items that match existing stacks)
ListTransaction<MoveTransaction<ItemStackTransaction>> result = container.quickStackTo(
    targetContainer
);
```

**Other Operations:**

```java
// Get item at slot
ItemStack item = container.getItemStack((short) 0);

// Set item at slot
ItemStackSlotTransaction result = container.setItemStackForSlot((short) 0, itemStack);

// Replace item in slot
ItemStackSlotTransaction result = container.replaceItemStackInSlot(
    (short) 0,
    expectedOldStack,
    newStack
);

// Clear container
ClearTransaction result = container.clear();

// Sort items
ListTransaction<SlotTransaction> result = container.sortItems(SortType.NAME);

// Check if empty
boolean isEmpty = container.isEmpty();

// Count matching items
int count = container.countItemStacks(stack -> stack.getItemId().equals("Iron_Ore"));

// Iterate all items
container.forEach((slot, itemStack) -> {
    System.out.println("Slot " + slot + ": " + itemStack);
});
```

### Transactions

All container operations return transaction objects with operation results:

```java
// Basic transaction
public interface Transaction {
    boolean succeeded();
}

// Item stack transaction
ItemStackTransaction transaction = container.addItemStack(itemStack);
boolean success = transaction.succeeded();
ItemStack added = transaction.getOutput();
ItemStack remainder = transaction.getRemainder();

// Slot transaction
SlotTransaction transaction = container.removeItemStackFromSlot((short) 0);
short slot = transaction.getSlot();
ItemStack before = transaction.getSlotBefore();
ItemStack after = transaction.getSlotAfter();
ItemStack output = transaction.getOutput();

// List transaction
ListTransaction<ItemStackTransaction> results = container.addItemStacks(items);
List<ItemStackTransaction> list = results.getList();
for (ItemStackTransaction t : list) {
    // Process each result
}
```

**Change Events:**

```java
// Register for container changes
EventRegistration registration = container.registerChangeEvent(event -> {
    ItemContainer changed = event.container();
    Transaction transaction = event.transaction();
    // Handle change
});

// Unregister when done
registration.unregister();
```

---

## Item Metadata

ItemStacks can store arbitrary metadata using BSON documents:

```java
// Add metadata
ItemStack enchanted = itemStack.withMetadata("Enchantments", Codec.STRING_ARRAY,
    new String[]{"Sharpness", "Unbreaking"});

// Add typed metadata with KeyedCodec
KeyedCodec<Integer> LEVEL_CODEC = new KeyedCodec<>("Level", Codec.INTEGER);
ItemStack leveled = itemStack.withMetadata(LEVEL_CODEC, 5);

// Read metadata
String[] enchantments = itemStack.getFromMetadataOrNull("Enchantments",
    new ArrayCodec<>(Codec.STRING, String[]::new));

// Read with default
Integer level = itemStack.getFromMetadataOrDefault("Level",
    BuilderCodec.builder(Integer.class, () -> 1).build());

// Check and read
BsonDocument meta = itemStack.getMetadata();
if (meta != null && meta.containsKey("CustomData")) {
    BsonValue value = meta.get("CustomData");
}
```

**Built-in Metadata Keys:**

```java
public static class Metadata {
    public static final String BLOCK_STATE = "BlockState";
}

// Example: Block state for placed blocks
ItemStack blockItem = itemStack.withMetadata(ItemStack.Metadata.BLOCK_STATE,
    Codec.STRING, "waterlogged=true");
```

---

## Item Drop System

[`ItemDropList`](https://github.com/Savag3life/TaleServer/tree/main/hytale/main/src/com/hypixel/hytale/server/core/asset/type/item/config/ItemDropList.java) defines loot tables for drops.

```json
{
    "Id": "Zombie_Drops",
    "Container": {
        "Type": "Choice",
        "Probability": 0.8,
        "Containers": [
            {
                "Type": "Single",
                "Item": {
                    "Id": "Rotten_Flesh",
                    "Min": 1,
                    "Max": 3
                }
            },
            {
                "Type": "Multiple",
                "Items": [
                    {
                        "Id": "Iron_Ingot",
                        "Min": 0,
                        "Max": 1,
                        "Probability": 0.05
                    },
                    {
                        "Id": "Gold_Nugget",
                        "Min": 0,
                        "Max": 2,
                        "Probability": 0.02
                    }
                ]
            }
        ]
    }
}
```

**Container Types:**

| Type | Description |
|------|-------------|
| `Single` | Single item with quantity range |
| `Multiple` | Multiple items, each with probability |
| `Choice` | Random selection from sub-containers |
| `Droplist` | Reference another ItemDropList |
| `Empty` | No drops |

**ItemEntityConfig:**

[`ItemEntityConfig`](https://github.com/Savag3life/TaleServer/tree/main/hytale/main/src/com/hypixel/hytale/server/core/asset/type/item/config/ItemEntityConfig.java) controls dropped item entity behavior:

```json
{
    "ItemEntity": {
        "Physics": {
            "Gravity": 5.0,
            "Friction": 0.5,
            "Buoyant": false
        },
        "PickupRadius": 1.75,
        "Lifetime": 300.0,
        "ParticleSystemId": "Item",
        "ParticleColor": "#FFFFFF",
        "ShowItemParticles": true
    }
}
```

---

## Translation & Lore

[`ItemTranslationProperties`](https://github.com/Savag3life/TaleServer/tree/main/hytale/main/src/com/hypixel/hytale/server/core/asset/type/item/config/ItemTranslationProperties.java) handles localization:

```json
{
    "Id": "Legendary_Sword",
    "TranslationProperties": {
        "Name": "server.items.legendary_sword.name",
        "Description": "server.items.legendary_sword.description"
    }
}
```

**Default Translation Keys:**

If not specified, default keys are generated:
- Name: `server.items.<itemId>.name`
- Description: `server.items.<itemId>.description`

**API Usage:**

```java
Item item = Item.getAssetMap().getAsset("Legendary_Sword");

// Get translation keys
String nameKey = item.getTranslationKey();
String descKey = item.getDescriptionTranslationKey();

// Get translation properties
ItemTranslationProperties props = item.getTranslationProperties();
if (props != null) {
    String customNameKey = props.getName();
    String customDescKey = props.getDescription();
}
```

---

## Creating Items

### JSON Definition

Create a JSON file in the `Server/items/` directory:

```json
{
    "Id": "Magic_Staff",
    "Parent": "Base_Weapon",

    "TranslationProperties": {
        "Name": "server.items.magic_staff.name",
        "Description": "server.items.magic_staff.description"
    },

    "Icon": "Icons/ItemsGenerated/Magic_Staff.png",
    "Model": "Items/Weapons/Magic_Staff.blockymodel",
    "Texture": "Items/Weapons/Magic_Staff.png",
    "Scale": 1.0,

    "MaxStack": 1,
    "MaxDurability": 500.0,
    "DurabilityLossOnHit": 1.0,

    "Quality": "Rare",
    "Categories": ["Weapons", "Magic"],
    "ItemLevel": 10,

    "Weapon": {
        "StatModifiers": {
            "MagicDamage": [
                { "Target": "Base", "CalculationType": "Add", "Amount": 25.0 }
            ]
        }
    },

    "Interactions": {
        "Primary": "RI_Magic_Staff_Attack",
        "Secondary": "RI_Magic_Staff_Special"
    },

    "Light": {
        "R": 100,
        "G": 50,
        "B": 200
    },

    "Tags": {
        "Element": ["Magic", "Arcane"],
        "Tier": ["Rare"]
    }
}
```

### Programmatic Creation

```java
// Items are typically created via JSON, but can be created programmatically:

// Create a simple item stack
ItemStack magicStaff = new ItemStack("Magic_Staff");

// Create with custom durability
ItemStack damagedStaff = new ItemStack("Magic_Staff", 1, 250.0, 500.0, null);

// Create with metadata
BsonDocument metadata = new BsonDocument();
metadata.put("Enchantment", new BsonString("Fire"));
ItemStack enchantedStaff = new ItemStack("Magic_Staff", 1, metadata);
```

---

## Working with Items

### Retrieving Items

```java
// Get item definition
Item item = Item.getAssetMap().getAsset("Iron_Sword");

// Get all items with a tag
int tagIndex = AssetRegistry.getTagIndex("Weapon");
Set<String> weaponIds = Item.getAssetMap().getKeysForTag(tagIndex);

// Get items by category
Item item = Item.getAssetMap().getAsset("Iron_Sword");
String[] categories = item.getCategories(); // ["Weapons", "Swords"]

// Check item properties
if (item.getWeapon() != null) {
    // It's a weapon
}
if (item.hasBlockType()) {
    String blockId = item.getBlockId();
}
```

### Spawning Item Entities

```java
// Using ItemUtils to drop items from an entity
ItemStack itemToDrop = new ItemStack("Iron_Sword");
Ref<EntityStore> itemEntity = ItemUtils.dropItem(entityRef, itemToDrop, componentAccessor);

// Throw item with custom speed
Ref<EntityStore> thrownItem = ItemUtils.throwItem(
    entityRef,
    itemToDrop,
    5.0f, // throw speed
    componentAccessor
);

// Throw in specific direction
Vector3d direction = new Vector3d(1, 0.5, 0).normalize();
Ref<EntityStore> thrownItem = ItemUtils.throwItem(
    entityRef,
    componentAccessor,
    itemToDrop,
    direction,
    5.0f
);

// Interactive pickup (with event firing)
ItemUtils.interactivelyPickupItem(
    playerRef,
    itemStack,
    origin, // where the item came from
    componentAccessor
);
```

### Giving Items to Players

```java
// Get player's inventory
Player player = componentAccessor.getComponent(playerRef, Player.getComponentType());
ItemContainer inventory = player.getInventory().getCombinedHotbarFirst();

// Give item
ItemStack gift = new ItemStack("Diamond", 64);
ItemStackTransaction result = inventory.addItemStack(gift);

if (!result.succeeded() || result.getRemainder() != null) {
    // Inventory full, drop remainder
    ItemStack remainder = result.getRemainder();
    if (remainder != null) {
        ItemUtils.dropItem(playerRef, remainder, componentAccessor);
    }
}

// Give to specific slot
ItemStackSlotTransaction slotResult = inventory.addItemStackToSlot((short) 0, gift);

// Check if player can receive items first
if (inventory.canAddItemStack(gift)) {
    inventory.addItemStack(gift);
} else {
    // Handle full inventory
}
```

---

## Best Practices

### Item ID Naming

Use PascalCase with underscores:
- `Iron_Sword` (correct)
- `iron_sword` (will log warning)
- `IronSword` (acceptable)

### File Organization

```
Server/
  items/
    weapons/
      swords/
        Iron_Sword.json
        Diamond_Sword.json
      axes/
        Iron_Axe.json
    armor/
      Iron_Helmet.json
      Iron_Chestplate.json
    consumables/
      Health_Potion.json
    blocks/
      Stone.json
```

### Inheritance Hierarchy

Create base items for common configurations:

```json
// Base_Sword.json
{
    "Id": "Base_Sword",
    "MaxStack": 1,
    "MaxDurability": 100.0,
    "Categories": ["Weapons", "Swords"],
    "PlayerAnimationsId": "Sword",
    "Weapon": {
        "StatModifiers": {
            "AttackDamage": [
                { "Target": "Base", "CalculationType": "Add", "Amount": 5.0 }
            ]
        }
    }
}

// Iron_Sword.json
{
    "Id": "Iron_Sword",
    "Parent": "Base_Sword",
    "Quality": "Common",
    "MaxDurability": 250.0,
    "Weapon": {
        "StatModifiers": {
            "AttackDamage": [
                { "Target": "Base", "CalculationType": "Add", "Amount": 10.0 }
            ]
        }
    }
}
```

### Transaction Handling

Always check transaction results:

```java
// Good - check result
ItemStackTransaction result = container.addItemStack(itemStack);
if (result.succeeded()) {
    ItemStack remainder = result.getRemainder();
    if (remainder != null) {
        // Handle items that couldn't be added
        ItemUtils.dropItem(entityRef, remainder, accessor);
    }
} else {
    // Handle complete failure
}

// Bad - ignoring result
container.addItemStack(itemStack); // Items might be lost!
```

### Check Before Modify

```java
// Good - check first for important operations
if (container.canRemoveMaterials(materials)) {
    container.removeMaterials(materials);
    // Safe to proceed with crafting
}

// Also good - use allOrNothing flag
ListTransaction<MaterialTransaction> result = container.removeMaterials(
    materials,
    true,  // allOrNothing - fails if can't remove all
    true   // filter
);
if (result.succeeded()) {
    // All materials removed
}
```

### Metadata Keys

Use consistent, namespaced keys for custom metadata:

```java
// Good - namespaced keys
public static final String META_ENCHANTMENTS = "MyPlugin:Enchantments";
public static final String META_CUSTOM_NAME = "MyPlugin:CustomName";
public static final String META_BOUND_PLAYER = "MyPlugin:BoundPlayer";

// Bad - generic keys that might conflict
public static final String META_DATA = "Data";
public static final String META_LEVEL = "Level";
```

---

## Summary Reference

### Item Asset Methods

| Method | Description |
|--------|-------------|
| `Item.getAssetStore()` | Get the item asset store |
| `Item.getAssetMap()` | Get the item asset map |
| `item.getId()` | Get item ID |
| `item.getTranslationKey()` | Get name localization key |
| `item.getMaxStack()` | Get max stack size |
| `item.getMaxDurability()` | Get max durability |
| `item.getWeapon()` | Get weapon config |
| `item.getArmor()` | Get armor config |
| `item.getTool()` | Get tool config |
| `item.getCategories()` | Get category list |

### ItemStack Methods

| Method | Description |
|--------|-------------|
| `new ItemStack(id)` | Create stack of 1 |
| `new ItemStack(id, qty)` | Create stack with quantity |
| `stack.getItemId()` | Get item ID |
| `stack.getItem()` | Get Item asset |
| `stack.getQuantity()` | Get quantity |
| `stack.withQuantity(qty)` | Clone with new quantity |
| `stack.withDurability(dur)` | Clone with new durability |
| `stack.withMetadata(doc)` | Clone with new metadata |
| `stack.isStackableWith(other)` | Check if stackable |
| `ItemStack.isEmpty(stack)` | Check if null or empty |

### Container Methods

| Method | Description |
|--------|-------------|
| `container.addItemStack(stack)` | Add item anywhere |
| `container.addItemStackToSlot(slot, stack)` | Add to specific slot |
| `container.removeItemStackFromSlot(slot)` | Remove from slot |
| `container.getItemStack(slot)` | Get item at slot |
| `container.setItemStackForSlot(slot, stack)` | Set slot contents |
| `container.clear()` | Clear all items |
| `container.isEmpty()` | Check if empty |
| `container.canAddItemStack(stack)` | Check if addable |
| `container.moveItemStackFromSlot(slot, target)` | Move between containers |
