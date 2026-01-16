# Assets System

The Assets System is a core framework in TaleServer for defining, loading, managing, and synchronizing game data. It provides a type-safe, thread-safe, and extensible architecture for handling all game assets including items, blocks, sounds, particles, environments, and more.

## Table of Contents

- [Overview](#overview)
- [Core Concepts](#core-concepts)
  - [JsonAsset Interface](#jsonasset-interface)
  - [JsonAssetWithMap Interface](#jsonassetwithmap-interface)
  - [AssetStore](#assetstore)
  - [AssetMap](#assetmap)
  - [AssetRegistry](#assetregistry)
- [Asset Types](#asset-types)
- [Creating Custom Assets](#creating-custom-assets)
  - [Step 1: Define the Asset Class](#step-1-define-the-asset-class)
  - [Step 2: Create the Codec](#step-2-create-the-codec)
  - [Step 3: Register the Asset Store](#step-3-register-the-asset-store)
- [Asset Loading](#asset-loading)
  - [Loading from Files](#loading-from-files)
  - [Loading Programmatically](#loading-programmatically)
  - [Asset Inheritance](#asset-inheritance)
- [Asset Retrieval](#asset-retrieval)
- [Asset Removal](#asset-removal)
- [Asset Packs](#asset-packs)
- [Tagging System](#tagging-system)
- [Dependency Management](#dependency-management)
- [Asset Validation](#asset-validation)
- [Hot Reloading](#hot-reloading)
- [Events](#events)
- [Thread Safety](#thread-safety)
- [Best Practices](#best-practices)

---

## Overview

The Assets System serves as the central data management layer for TaleServer. Every piece of configurable game content - from items and blocks to sounds and particle effects - is defined as an asset. Assets are:

- **JSON-defined**: All assets are stored as JSON files that can be easily edited
- **Type-safe**: Strong typing ensures compile-time safety
- **Inheritable**: Assets can extend other assets to reduce duplication
- **Hot-reloadable**: Changes to asset files are detected and applied in real-time during development
- **Network-synchronized**: Assets are automatically sent to clients when they connect
- **Tagged**: Assets can be categorized using a flexible tagging system

## Core Concepts

### JsonAsset Interface

The foundation of all assets is the [`JsonAsset`](https://github.com/Savag3life/TaleServer/tree/main/hytale/main/src/com/hypixel/hytale/assetstore/JsonAsset.java) interface:

```java
public interface JsonAsset<K> {
    K getId();
}
```

This simple interface requires all assets to have a unique identifier of type `K` (typically `String`).

### JsonAssetWithMap Interface

Assets that need to be stored in an [`AssetMap`](https://github.com/Savag3life/TaleServer/tree/main/hytale/main/src/com/hypixel/hytale/assetstore/AssetMap.java) implement [`JsonAssetWithMap`](https://github.com/Savag3life/TaleServer/tree/main/hytale/main/src/com/hypixel/hytale/assetstore/map/JsonAssetWithMap.java):

```java
public interface JsonAssetWithMap<K, M extends AssetMap<K, ?>> extends JsonAsset<K> {
}
```

This interface links an asset type to its specific storage map implementation.

### AssetStore

The [`AssetStore`](https://github.com/Savag3life/TaleServer/tree/main/hytale/main/src/com/hypixel/hytale/assetstore/AssetStore.java) is the primary manager for a specific asset type. It handles:

- Loading assets from files or programmatically
- Storing assets in the associated `AssetMap`
- Managing asset dependencies
- Validating assets using codecs
- Firing events when assets are loaded/removed
- Supporting inheritance between assets

Key properties of an `AssetStore`:

| Property | Description |
|----------|-------------|
| `path` | The directory path where asset files are located (e.g., `"items"`) |
| `extension` | File extension for assets (default: `".json"`) |
| `codec` | The `AssetCodec` used to serialize/deserialize assets |
| `keyFunction` | Function to extract the key from an asset |
| `loadsAfter` | Set of asset types this store depends on |
| `loadsBefore` | Set of asset types that depend on this store |
| `replaceOnRemove` | Function to provide a replacement when an asset is removed |

### AssetMap

[`AssetMap`](https://github.com/Savag3life/TaleServer/tree/main/hytale/main/src/com/hypixel/hytale/assetstore/AssetMap.java) is the abstract storage layer for assets. Different implementations optimize for different use cases:

| Implementation | Use Case |
|----------------|----------|
| [`DefaultAssetMap`](https://github.com/Savag3life/TaleServer/tree/main/hytale/main/src/com/hypixel/hytale/assetstore/map/DefaultAssetMap.java) | General-purpose storage with case-insensitive keys |
| [`BlockTypeAssetMap`](https://github.com/Savag3life/TaleServer/tree/main/hytale/main/src/com/hypixel/hytale/assetstore/map/BlockTypeAssetMap.java) | Optimized for blocks with indexed array storage |
| [`IndexedAssetMap`](https://github.com/Savag3life/TaleServer/tree/main/hytale/main/src/com/hypixel/hytale/assetstore/map/IndexedAssetMap.java) | Generic indexed storage |
| [`IndexedLookupTableAssetMap`](https://github.com/Savag3life/TaleServer/tree/main/hytale/main/src/com/hypixel/hytale/assetstore/map/IndexedLookupTableAssetMap.java) | Fast lookup with lookup table optimization |

Key methods of `AssetMap`:

```java
// Retrieve an asset by key
T getAsset(K key);

// Get asset from a specific pack
T getAsset(String packKey, K key);

// Get the file path for an asset
Path getPath(K key);

// Get all keys for a file path
Set<K> getKeys(Path path);

// Get child assets (assets that inherit from a key)
Set<K> getChildren(K key);

// Get all assets with a specific tag
Set<K> getKeysForTag(int tagIndex);
```

### AssetRegistry

The [`AssetRegistry`](https://github.com/Savag3life/TaleServer/tree/main/hytale/main/src/com/hypixel/hytale/assetstore/AssetRegistry.java) is a static registry that holds all `AssetStore` instances:

```java
// Get an asset store by asset class
AssetStore<K, T, M> store = AssetRegistry.getAssetStore(MyAsset.class);

// Register a new asset store
AssetRegistry.register(myAssetStore);

// Unregister an asset store
AssetRegistry.unregister(myAssetStore);

// Get all registered stores
Map<Class<?>, AssetStore<?, ?, ?>> stores = AssetRegistry.getStoreMap();
```

The registry also manages the global tag system:

```java
// Get or create a tag index
int tagIndex = AssetRegistry.getOrCreateTagIndex("MyTag");

// Get existing tag index (returns TAG_NOT_FOUND if not exists)
int tagIndex = AssetRegistry.getTagIndex("MyTag");

// Register a client-synchronized tag
AssetRegistry.registerClientTag("ClientTag");
```

---

## Asset Types

TaleServer includes many built-in asset types. Here are the most common ones:

| Asset Type | Description | Path |
|------------|-------------|------|
| [`Item`](https://github.com/Savag3life/TaleServer/tree/main/hytale/main/src/com/hypixel/hytale/server/core/asset/type/item/config/Item.java) | Game items (weapons, tools, consumables) | `items` |
| [`BlockType`](https://github.com/Savag3life/TaleServer/tree/main/hytale/main/src/com/hypixel/hytale/server/core/asset/type/blocktype/config/BlockType.java) | Block definitions | `blocktypes` |
| [`SoundEvent`](https://github.com/Savag3life/TaleServer/tree/main/hytale/main/src/com/hypixel/hytale/server/core/asset/type/soundevent/config/SoundEvent.java) | Sound effect definitions | `sounds` |
| [`ParticleSystem`](https://github.com/Savag3life/TaleServer/tree/main/hytale/main/src/com/hypixel/hytale/server/core/asset/type/particle/config/ParticleSystem.java) | Particle effect systems | `particles` |
| [`Environment`](https://github.com/Savag3life/TaleServer/tree/main/hytale/main/src/com/hypixel/hytale/server/core/asset/type/environment/config/Environment.java) | World environment settings | `environments` |
| [`Weather`](https://github.com/Savag3life/TaleServer/tree/main/hytale/main/src/com/hypixel/hytale/server/core/asset/type/weather/config/Weather.java) | Weather type definitions | `weather` |
| [`ModelAsset`](https://github.com/Savag3life/TaleServer/tree/main/hytale/main/src/com/hypixel/hytale/server/core/asset/type/model/config/ModelAsset.java) | 3D model definitions | `models` |
| [`CraftingRecipe`](https://github.com/Savag3life/TaleServer/tree/main/hytale/main/src/com/hypixel/hytale/server/core/asset/type/item/config/CraftingRecipe.java) | Item crafting recipes | `recipes` |
| [`Projectile`](https://github.com/Savag3life/TaleServer/tree/main/hytale/main/src/com/hypixel/hytale/server/core/asset/type/projectile/config/Projectile.java) | Projectile configurations | `projectiles` |
| [`EntityEffect`](https://github.com/Savag3life/TaleServer/tree/main/hytale/main/src/com/hypixel/hytale/server/core/asset/type/entityeffect/config/EntityEffect.java) | Visual/audio effects on entities | `effects` |

---

## Creating Custom Assets

### Step 1: Define the Asset Class

Create a class that implements `JsonAssetWithMap`:

```java
public class MyCustomAsset implements JsonAssetWithMap<String, DefaultAssetMap<String, MyCustomAsset>> {

    // Required: ID field
    private String id;

    // Required: Extra data for the asset system
    private AssetExtraInfo.Data data;

    // Your custom fields
    private String name;
    private int value;
    private String[] tags;

    @Override
    public String getId() {
        return id;
    }

    // Getters and setters...
}
```

### Step 2: Create the Codec

Use [`AssetBuilderCodec`](https://github.com/Savag3life/TaleServer/tree/main/hytale/main/src/com/hypixel/hytale/assetstore/codec/AssetBuilderCodec.java) to define how your asset is serialized:

```java
public static final AssetBuilderCodec<String, MyCustomAsset> CODEC = AssetBuilderCodec.builder(
        MyCustomAsset.class,
        MyCustomAsset::new,                              // Constructor
        Codec.STRING,                                     // Key codec
        (asset, id) -> asset.id = id,                    // ID setter
        asset -> asset.id,                                // ID getter
        (asset, data) -> asset.data = data,              // Data setter
        asset -> asset.data                               // Data getter
    )
    // Add fields with inheritance support
    .<String>appendInherited(
        new KeyedCodec<>("Name", Codec.STRING),
        (asset, name) -> asset.name = name,
        asset -> asset.name,
        (asset, parent) -> asset.name = parent.name      // Inheritance behavior
    )
    .add()
    // Add fields without inheritance
    .<Integer>append(
        new KeyedCodec<>("Value", Codec.INTEGER),
        (asset, value) -> asset.value = value,
        asset -> asset.value
    )
    .addValidator(Validators.greaterThan(0))             // Add validation
    .documentation("The value must be positive")         // Add documentation
    .add()
    .build();
```

### Step 3: Register the Asset Store

Register your asset store during plugin initialization using [`HytaleAssetStore`](https://github.com/Savag3life/TaleServer/tree/main/hytale/main/src/com/hypixel/hytale/server/core/asset/HytaleAssetStore.java):

```java
public class MyPlugin extends JavaPlugin {

    @Override
    protected void setup() {
        // Register the asset store
        AssetRegistry.register(
            HytaleAssetStore.builder(MyCustomAsset.class, new DefaultAssetMap<>())
                .setPath("mycustomassets")           // Server/mycustomassets/
                .setCodec(MyCustomAsset.CODEC)
                .setKeyFunction(MyCustomAsset::getId)
                .loadsAfter(Item.class)              // Dependencies
                .build()
        );
    }
}
```

---

## Asset Loading

### Loading from Files

Assets are automatically loaded from JSON files during server startup. The file structure should match your asset store path:

```
Server/
  mycustomassets/
    MyAsset1.json
    MyAsset2.json
    subfolder/
      MyAsset3.json
```

Example JSON file (`MyAsset1.json`):

```json
{
    "Id": "MyAsset1",
    "Name": "My First Asset",
    "Value": 100,
    "Tags": {
        "Category": ["Combat", "Melee"],
        "Rarity": ["Common"]
    }
}
```

To manually trigger loading:

```java
AssetStore<String, MyCustomAsset, ?> store = AssetRegistry.getAssetStore(MyCustomAsset.class);

// Load from a directory
AssetLoadResult<String, MyCustomAsset> result = store.loadAssetsFromDirectory("PackName", assetsPath);

// Load from specific paths
List<Path> paths = List.of(path1, path2, path3);
AssetLoadResult<String, MyCustomAsset> result = store.loadAssetsFromPaths("PackName", paths);
```

### Loading Programmatically

You can create and load assets without JSON files:

```java
MyCustomAsset asset = new MyCustomAsset();
asset.setId("ProgrammaticAsset");
asset.setName("Created in Code");
asset.setValue(50);

AssetStore<String, MyCustomAsset, ?> store = AssetRegistry.getAssetStore(MyCustomAsset.class);
store.loadAssets("PackName", List.of(asset));
```

### Asset Inheritance

Assets can inherit from parent assets using the `Parent` field:

```json
{
    "Id": "ChildAsset",
    "Parent": "ParentAsset",
    "Value": 200
}
```

In this example, `ChildAsset` inherits all properties from `ParentAsset` but overrides `Value`.

Special inheritance keyword `"super"` inherits from the same asset in a different pack:

```json
{
    "Id": "ExistingAsset",
    "Parent": "super",
    "Value": 300
}
```

---

## Asset Retrieval

```java
// Get the asset store
AssetStore<String, MyCustomAsset, ?> store = AssetRegistry.getAssetStore(MyCustomAsset.class);

// Get the asset map
AssetMap<String, MyCustomAsset> assetMap = store.getAssetMap();

// Retrieve a single asset
MyCustomAsset asset = assetMap.getAsset("MyAsset1");

// Retrieve from a specific pack
MyCustomAsset asset = assetMap.getAsset("PackName", "MyAsset1");

// Get all assets
Map<String, MyCustomAsset> allAssets = assetMap.getAssetMap();

// Get assets by tag
int tagIndex = AssetRegistry.getTagIndex("Combat");
Set<String> combatAssets = assetMap.getKeysForTag(tagIndex);

// Get child assets (assets that inherit from this one)
Set<String> children = assetMap.getChildren("ParentAsset");

// Get asset file path
Path path = assetMap.getPath("MyAsset1");
```

---

## Asset Removal

```java
AssetStore<String, MyCustomAsset, ?> store = AssetRegistry.getAssetStore(MyCustomAsset.class);

// Remove specific assets
Set<String> removed = store.removeAssets(Set.of("Asset1", "Asset2"));

// Remove assets by file path
Set<String> removed = store.removeAssetWithPath(path);

// Remove all assets from a pack
store.removeAssetPack("PackName");
```

When an asset is removed:
1. All child assets (assets that inherit from it) are also removed
2. If `replaceOnRemove` is configured, a replacement asset is provided
3. `RemovedAssetsEvent` is fired
4. Connected clients receive removal packets

---

## Asset Packs

Assets can be organized into packs using [`AssetPack`](https://github.com/Savag3life/TaleServer/tree/main/hytale/main/src/com/hypixel/hytale/assetstore/AssetPack.java):

```java
// Create a pack from a directory
AssetPack pack = new AssetPack("MyMod", modPath);

// Create a pack from a ZIP file
AssetPack pack = new AssetPack("MyMod", zipPath, fileSystem);

// Check if pack is immutable (cannot write assets)
boolean immutable = pack.isImmutable();

// Get the pack root path
Path root = pack.getRoot();
```

Loading assets from a pack:

```java
AssetRegistryLoader.loadAssets(event, assetPack);
```

---

## Tagging System

Tags provide a flexible way to categorize and query assets. Tags are defined in the asset JSON:

```json
{
    "Id": "IronSword",
    "Tags": {
        "Material": ["Metal", "Iron"],
        "Category": ["Weapon", "Melee"],
        "Rarity": ["Common"]
    }
}
```

Tags are automatically expanded into a flat list. The example above generates:
- `Material`
- `Metal`
- `Iron`
- `Material=Metal`
- `Material=Iron`
- `Category`
- `Weapon`
- `Melee`
- `Category=Weapon`
- `Category=Melee`
- `Rarity`
- `Common`
- `Rarity=Common`

Querying by tag:

```java
// Get or create tag index
int tagIndex = AssetRegistry.getOrCreateTagIndex("Weapon");

// Query assets with tag
Set<String> weapons = assetMap.getKeysForTag(tagIndex);
```

---

## Dependency Management

Asset stores can declare dependencies on other asset types:

```java
HytaleAssetStore.builder(MyAsset.class, new DefaultAssetMap<>())
    .loadsAfter(Item.class, BlockType.class)   // Load after these types
    .loadsBefore(Recipe.class)                  // Load before this type
    .build();
```

This ensures assets are loaded in the correct order. The system:
1. Converts `loadsBefore` to `loadsAfter` on dependent stores
2. Detects circular dependencies
3. Loads assets in topological order

---

## Asset Validation

Assets are validated during loading using the codec's validators:

```java
AssetBuilderCodec.builder(...)
    .<Integer>append(
        new KeyedCodec<>("Value", Codec.INTEGER),
        (asset, value) -> asset.value = value,
        asset -> asset.value
    )
    .addValidator(Validators.greaterThan(0))
    .addValidator(Validators.lessThan(1000))
    .add()
```

Use [`AssetKeyValidator`](https://github.com/Savag3life/TaleServer/tree/main/hytale/main/src/com/hypixel/hytale/assetstore/AssetKeyValidator.java) to validate references to other assets:

```java
.addValidator(Item.VALIDATOR_CACHE.getValidator())
```

Validation results are collected in [`AssetValidationResults`](https://github.com/Savag3life/TaleServer/tree/main/hytale/main/src/com/hypixel/hytale/assetstore/AssetValidationResults.java):

```java
store.validate(key, results, extraInfo);
results.logOrThrowValidatorExceptions(logger);
```

---

## Hot Reloading

During development, the [`AssetMonitor`](https://github.com/Savag3life/TaleServer/tree/main/hytale/main/src/com/hypixel/hytale/server/core/asset/monitor/AssetMonitor.java) watches for file changes:

```java
// Add file monitoring for a directory
store.addFileMonitor("PackName", assetsPath);

// Remove file monitoring
store.removeFileMonitor(assetsPath);
```

When files change:
1. Modified assets are reloaded
2. Deleted assets are removed
3. New assets are loaded
4. Client notifications are sent
5. `LoadedAssetsEvent` and `RemovedAssetsEvent` are fired

Control cache rebuilding with [`AssetUpdateQuery`](https://github.com/Savag3life/TaleServer/tree/main/hytale/main/src/com/hypixel/hytale/assetstore/AssetUpdateQuery.java):

```java
AssetUpdateQuery query = new AssetUpdateQuery(
    false,  // disableAssetCompare
    new AssetUpdateQuery.RebuildCache(
        true,   // blockTextures
        true,   // models
        true,   // modelTextures
        false,  // mapGeometry
        true,   // itemIcons
        false   // commonAssetsRebuild
    )
);

store.loadAssetsFromPaths("PackName", paths, query);
```

---

## Events

The asset system fires events through the event bus:

### LoadedAssetsEvent

Fired when assets are successfully loaded:

```java
@EventHandler
public void onAssetsLoaded(LoadedAssetsEvent<String, Item> event) {
    Map<String, Item> loaded = event.getLoadedAssets();
    AssetMap<String, Item> assetMap = event.getAssetMap();
    // Process loaded assets...
}
```

### RemovedAssetsEvent

Fired when assets are removed:

```java
@EventHandler
public void onAssetsRemoved(RemovedAssetsEvent<String, Item> event) {
    Set<String> removed = event.getRemovedKeys();
    boolean hasReplacements = event.hasReplacements();
    // Handle removal...
}
```

### GenerateAssetsEvent

Fired during the asset generation phase:

```java
@EventHandler
public void onGenerateAssets(GenerateAssetsEvent<String, Item> event) {
    Map<String, Item> loaded = event.getLoadedAssets();
    // Generate derived data...
}
```

### RegisterAssetStoreEvent

Fired when a new asset store is registered:

```java
@EventHandler
public void onStoreRegistered(RegisterAssetStoreEvent event) {
    AssetStore<?, ?, ?> store = event.getAssetStore();
    // React to new store...
}
```

---

## Thread Safety

The asset system is designed for concurrent access:

- **AssetRegistry**: Uses `ReentrantReadWriteLock` for store registration
- **DefaultAssetMap**: Uses `StampedLock` for optimistic reads
- **BlockTypeAssetMap**: Uses `StampedLock` for indexed access
- **Tag storage**: Uses concurrent hash maps
- **Asset operations**: Synchronized during modifications

Lock ordering:
1. Acquire `AssetRegistry.ASSET_LOCK` write lock
2. Modify asset maps
3. Fire events
4. Release lock

```java
AssetRegistry.ASSET_LOCK.writeLock().lock();
try {
    // Modify assets safely
} finally {
    AssetRegistry.ASSET_LOCK.writeLock().unlock();
}
```

---

## Best Practices

### Asset ID Naming

Use PascalCase with underscores for asset IDs:
- `Iron_Sword` (correct)
- `iron_sword` (incorrect - will log warning)
- `IronSword` (acceptable but underscores preferred for readability)

### File Organization

```
Server/
  items/
    weapons/
      swords/
        Iron_Sword.json
        Steel_Sword.json
      axes/
        Iron_Axe.json
    consumables/
      Health_Potion.json
```

### Inheritance Hierarchy

Create base assets for common configurations:

```json
// Base_Weapon.json
{
    "Id": "Base_Weapon",
    "MaxStack": 1,
    "Categories": ["Weapons"]
}

// Iron_Sword.json
{
    "Id": "Iron_Sword",
    "Parent": "Base_Weapon",
    "Name": "Iron Sword",
    "Damage": 10
}
```

### Validation

Always validate asset references:

```java
.addValidator(ItemSoundSet.VALIDATOR_CACHE.getValidator())
```

### Documentation

Add documentation to codec fields:

```java
.<Integer>append(...)
.documentation("Maximum number of items in a stack (1-64)")
.add()
```

### Error Handling

Check load results:

```java
AssetLoadResult<String, MyAsset> result = store.loadAssetsFromDirectory(...);

if (!result.getFailedToLoadKeys().isEmpty()) {
    logger.warning("Failed to load: " + result.getFailedToLoadKeys());
}

if (!result.getFailedToLoadPaths().isEmpty()) {
    logger.warning("Failed files: " + result.getFailedToLoadPaths());
}
```

---

## See Also

- [Event System](Events.md) - Event bus architecture and registration patterns
- [World System](World.md) - Universe, World, and Instance APIs
- [Commands](Commands.md) - Chat commands and internal structure
