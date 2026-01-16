# Universe, World & Instance API Overview

## 1. Architectural Hierarchy

```
Universe (Singleton)
├── Player Management
│   └── PlayerRef (network connection bridge)
│
├── World Management
│   ├── Map<String, World> (by name)
│   └── Map<UUID, World> (by UUID)
│       └── World (single-threaded tick thread)
│           ├── ChunkStore (chunk data)
│           ├── EntityStore (entity/player data)
│           ├── ChunkLightingManager
│           ├── WorldMapManager
│           ├── EventRegistry
│           └── WorldConfig
│               └── InstanceWorldConfig (for instances)
│
└── Instance Management (InstancesPlugin)
    └── Ephemeral worlds from templates
```

---

## 2. Universe

Located in [server/core/universe/Universe.java](https://github.com/Savag3life/TaleServer/tree/main/hytale/main/src/com/hypixel/hytale/server/core/universe/Universe.java)

The Universe is the **singleton root container** that manages all worlds and players. It extends `JavaPlugin` and provides thread-safe access to game state.

### Singleton Access

```java
Universe universe = Universe.get();
```

### World Management API

| Method | Return Type | Description |
|--------|-------------|-------------|
| `addWorld(String name)` | `CompletableFuture<World>` | Create world with default config |
| `addWorld(String name, String generatorType, String chunkStorageType)` | `CompletableFuture<World>` | Create world with specified providers |
| `makeWorld(String name, Path savePath, WorldConfig config)` | `CompletableFuture<World>` | Create world from explicit config |
| `loadWorld(String name)` | `CompletableFuture<World>` | Load existing world from disk |
| `removeWorld(String name)` | `boolean` | Remove world (drains players first) |
| `removeWorldExceptionally(String name)` | `void` | Remove world without waiting |
| `getWorld(String name)` | `World` | Get by name (case-insensitive) |
| `getWorld(UUID uuid)` | `World` | Get by UUID |
| `getDefaultWorld()` | `World` | Get fallback/default world |
| `getWorlds()` | `Map<String, World>` | Unmodifiable map of all worlds |

### Player Management API

| Method | Return Type | Description |
|--------|-------------|-------------|
| `addPlayer(Channel, language, protocol, uuid, username, auth, viewRadius, skin)` | `CompletableFuture<PlayerRef>` | Add new player connection |
| `removePlayer(PlayerRef)` | `void` | Disconnect player |
| `resetPlayer(PlayerRef)` | `CompletableFuture<PlayerRef>` | Reset player state |
| `getPlayers()` | `List<PlayerRef>` | All connected players |
| `getPlayer(UUID)` | `PlayerRef` | Get by UUID |
| `getPlayer(String, NameMatching)` | `PlayerRef` | Get by name with matching mode |
| `getPlayerCount()` | `int` | Current online count |

### Broadcasting API

| Method | Description |
|--------|-------------|
| `sendMessage(Message)` | Send chat message to all players |
| `broadcastPacket(Packet)` | Send packet to all players |
| `broadcastPacketNoCache(Packet)` | Send packet without caching |
| `broadcastPacket(Packet...)` | Send multiple packets |

### Lifecycle API

| Method | Return Type | Description |
|--------|-------------|-------------|
| `getUniverseReady()` | `CompletableFuture<Void>` | Completes when universe initialized |
| `runBackup()` | `CompletableFuture<Void>` | Trigger manual backup |
| `getPath()` | `Path` | Universe root directory |
| `getPlayerStorage()` | `PlayerStorage` | Persistent player data store |
| `getWorldConfigProvider()` | `WorldConfigProvider` | World config factory |

---

## 3. World

Located in [server/core/universe/world/World.java](https://github.com/Savag3life/TaleServer/tree/main/hytale/main/src/com/hypixel/hytale/server/core/universe/world/World.java)

Each World runs on its own **dedicated tick thread** and acts as an `Executor` for task scheduling. Worlds are isolated containers with their own chunk storage, entity store, and configuration.

### Initialization & Lifecycle

| Method | Return Type | Description |
|--------|-------------|-------------|
| `init()` | `CompletableFuture<World>` | Initialize world systems |
| `isAlive()` | `boolean` | Check if world is running |
| `drainPlayersTo(World)` | `CompletableFuture<Void>` | Move all players to another world |

### Player Management

| Method | Return Type | Description |
|--------|-------------|-------------|
| `addPlayer(PlayerRef)` | `CompletableFuture<PlayerRef>` | Add player at spawn |
| `addPlayer(PlayerRef, Transform)` | `CompletableFuture<PlayerRef>` | Add player at position |
| `addPlayer(PlayerRef, Transform, clearOverride, fadeOverride)` | `CompletableFuture<PlayerRef>` | Add with options |
| `trackPlayerRef(PlayerRef)` | `void` | Register player tracking |
| `untrackPlayerRef(PlayerRef)` | `void` | Remove player tracking |
| `getPlayerCount()` | `int` | Players in this world |
| `getPlayerRefs()` | `Collection<PlayerRef>` | All player refs |

### Chunk Access (Synchronous)

> [!WARNING]
> Synchronous chunk methods **block the calling thread**. Only use from the world's tick thread or with caution.

| Method | Return Type | Description |
|--------|-------------|-------------|
| `getChunk(long index)` | `WorldChunk` | Get or load chunk (blocking) |
| `getNonTickingChunk(long index)` | `WorldChunk` | Get without marking as ticking |
| `getChunkIfInMemory(long index)` | `WorldChunk` | Get only if cached |
| `getChunkIfLoaded(long index)` | `WorldChunk` | Get only if fully loaded |
| `getChunkIfNonTicking(long index)` | `WorldChunk` | Get if loaded but not ticking |

### Chunk Access (Asynchronous)

| Method | Return Type | Description |
|--------|-------------|-------------|
| `getChunkAsync(long index)` | `CompletableFuture<WorldChunk>` | Load chunk async |
| `getNonTickingChunkAsync(long index)` | `CompletableFuture<WorldChunk>` | Load non-ticking async |

### Entity Management

| Method | Return Type | Description |
|--------|-------------|-------------|
| `getEntityRef(UUID)` | `Ref<EntityStore>` | Get entity reference by UUID |
| `getEntity(UUID)` | `Entity` | *Deprecated* - use `getEntityRef` |
| `addEntity(T, position, rotation, reason)` | `T` | *Deprecated* - use EntityStore |
| `getPlayers()` | `List<Player>` | *Deprecated* - use `getPlayerRefs` |

### Configuration & Status

| Method | Return Type | Description |
|--------|-------------|-------------|
| `getName()` | `String` | World name |
| `getWorldConfig()` | `WorldConfig` | Full configuration |
| `getTick()` | `long` | Current tick counter |
| `isTicking()` | `boolean` | Whether chunk updates run |
| `setTicking(boolean)` | `void` | Enable/disable chunk ticking |
| `isPaused()` | `boolean` | Whether world is paused |
| `setPaused(boolean)` | `void` | Pause/unpause world |
| `setTps(int)` | `void` | Set ticks per second |
| `getGameplayConfig()` | `GameplayConfig` | Gameplay settings |
| `getDeathConfig()` | `DeathConfig` | Death/respawn settings |

### Storage Accessors

| Method | Return Type | Description |
|--------|-------------|-------------|
| `getChunkStore()` | `ChunkStore` | Chunk data manager |
| `getEntityStore()` | `EntityStore` | Entity data manager |
| `getChunkLighting()` | `ChunkLightingManager` | Lighting calculations |
| `getWorldMapManager()` | `WorldMapManager` | World map system |
| `getEventRegistry()` | `EventRegistry` | World-local events |
| `getSavePath()` | `Path` | World save directory |

### Task Execution (Executor Pattern)

| Method | Return Type | Description |
|--------|-------------|-------------|
| `execute(Runnable)` | `void` | Queue task for world thread |
| `isInThread()` | `boolean` | Check if on world thread |

### Feature Flags

| Method | Return Type | Description |
|--------|-------------|-------------|
| `registerFeature(ClientFeature, enabled)` | `void` | Set feature state |
| `getFeatures()` | `Map<ClientFeature, Boolean>` | All feature flags |
| `isFeatureEnabled(ClientFeature)` | `boolean` | Check feature |
| `broadcastFeatures()` | `void` | Sync features to clients |

---

## 4. WorldConfig

Located in [server/core/universe/world/WorldConfig.java](https://github.com/Savag3life/TaleServer/tree/main/hytale/main/src/com/hypixel/hytale/server/core/universe/world/WorldConfig.java)

Persistent configuration for a world, serialized to disk.

### Core Properties

| Property | Type | Description |
|----------|------|-------------|
| `uuid` | `UUID` | Unique world identifier |
| `displayName` | `String` | Player-facing name |
| `seed` | `long` | World generation seed |
| `gameMode` | `GameMode` | Default player game mode |
| `gameTime` | `Instant` | World time-of-day |
| `forcedWeather` | `String` | Weather override (or null) |

### Behavioral Flags

| Property | Type | Default | Description |
|----------|------|---------|-------------|
| `isTicking` | `boolean` | `true` | Enable chunk updates |
| `isBlockTicking` | `boolean` | `true` | Enable block updates |
| `isPvpEnabled` | `boolean` | `true` | Allow PvP combat |
| `isFallDamageEnabled` | `boolean` | `true` | Apply fall damage |
| `isSpawningNPC` | `boolean` | `true` | Allow NPC spawning |
| `isCompassUpdating` | `boolean` | `true` | Update compass |
| `deleteOnUniverseStart` | `boolean` | `false` | Auto-delete on startup |
| `deleteOnRemove` | `boolean` | `false` | Delete files on removal |

### Provider Interfaces

| Property | Type | Description |
|----------|------|-------------|
| `spawnProvider` | `ISpawnProvider` | Player spawn location logic |
| `worldGenProvider` | `IWorldGenProvider` | World generation strategy |
| `worldMapProvider` | `IWorldMapProvider` | World map generation |
| `chunkStorageProvider` | `IChunkStorageProvider` | Chunk persistence |
| `resourceStorageProvider` | `IResourceStorageProvider` | Resource persistence |

### Plugin Configuration

| Property | Type | Description |
|----------|------|-------------|
| `requiredPlugins` | `Map<PluginIdentifier, SemverRange>` | Plugin dependencies |
| `pluginConfig` | `TypeMap<Object>` | Plugin-specific settings |

---

## 5. Instance System

Located in [builtin/instances/](https://github.com/Savag3life/TaleServer/tree/main/hytale/main/src/com/hypixel/hytale/builtin/instances/)

Instances are **ephemeral world copies** created from templates. They have automatic lifecycle management and return-point handling.

### InstancesPlugin

```java
InstancesPlugin instances = InstancesPlugin.get();
```

| Constant | Value | Description |
|----------|-------|-------------|
| `INSTANCE_PREFIX` | `"instance-"` | World name prefix |
| `CONFIG_FILENAME` | `"instance.bson"` | Config file name |

### Spawning Instances

| Method | Return Type | Description |
|--------|-------------|-------------|
| `spawnInstance(String name, World forWorld, Transform returnPoint)` | `CompletableFuture<World>` | Create instance from template |
| `spawnInstance(String name, String worldName, World forWorld, Transform returnPoint)` | `CompletableFuture<World>` | Create with custom name |

**Instance creation flow:**
1. Copy instance template files
2. Generate new UUID
3. Inject `InstanceWorldConfig` with removal conditions
4. Set `WorldReturnPoint` for exit
5. Register with Universe

### InstanceWorldConfig

Located in [builtin/instances/config/InstanceWorldConfig.java](https://github.com/Savag3life/TaleServer/tree/main/hytale/main/src/com/hypixel/hytale/builtin/instances/config/InstanceWorldConfig.java)

| Method | Return Type | Description |
|--------|-------------|-------------|
| `get(WorldConfig)` | `InstanceWorldConfig` | Get if exists, null otherwise |
| `ensureAndGet(WorldConfig)` | `InstanceWorldConfig` | Get or create |
| `getRemovalConditions()` | `RemovalCondition[]` | Auto-removal triggers |
| `setRemovalConditions(RemovalCondition...)` | `void` | Set removal triggers |
| `getReturnPoint()` | `WorldReturnPoint` | Exit destination |
| `setReturnPoint(WorldReturnPoint)` | `void` | Set exit destination |
| `getDiscovery()` | `InstanceDiscoveryConfig` | First-entry display |
| `shouldPreventReconnection()` | `boolean` | Block reconnection |

### Removal Conditions

Located in [builtin/instances/removal/](https://github.com/Savag3life/TaleServer/tree/main/hytale/main/src/com/hypixel/hytale/builtin/instances/removal/)

| Condition | Trigger |
|-----------|---------|
| `WorldEmptyCondition` | No players in instance |
| `IdleTimeoutCondition` | No activity for duration |
| `TimeoutCondition` | Absolute time elapsed |

```java
interface RemovalCondition {
    boolean shouldRemoveWorld(Store<ChunkStore> store);
}
```

### WorldReturnPoint

| Property | Type | Description |
|----------|------|-------------|
| `worldUuid` | `UUID` | Target world UUID |
| `transform` | `Transform` | Exit coordinates |
| `preventReconnection` | `boolean` | Block instance reconnection |

### Instance Lifecycle

```
1. spawnInstance() called
   ├── Template copied to worlds/
   ├── New UUID assigned
   └── World registered with Universe

2. Players teleport in
   └── DiscoverInstanceEvent fires (first entry)

3. RemovalSystem monitors each tick
   └── Checks all RemovalCondition instances

4. When all conditions met:
   ├── Players drained to WorldReturnPoint
   ├── World removed from Universe
   └── Files deleted (if deleteOnRemove=true)
```

---

## 6. Safety Considerations

### Thread Safety

> [!IMPORTANT]
> Each World runs on its own **single-threaded tick thread**. Cross-thread access requires careful synchronization.

#### Thread-Safe Collections

| Class | Field | Type |
|-------|-------|------|
| `Universe` | `players` | `ConcurrentHashMap<UUID, PlayerRef>` |
| `Universe` | `worlds` | `ConcurrentHashMap<String, World>` |
| `Universe` | `worldsByUuid` | `ConcurrentHashMap<UUID, World>` |
| `World` | `players` | `ConcurrentHashMap<UUID, PlayerRef>` |
| `World` | `features` | `Collections.synchronizedMap` |

#### Atomic State

| Class | Field | Type | Purpose |
|-------|-------|------|---------|
| `World` | `alive` | `AtomicBoolean` | Shutdown state |
| `World` | `acceptingTasks` | `AtomicBoolean` | Task queue gate |
| `World` | `entitySeed` | `AtomicInteger` | Entity ID generation |
| `WorldConfig` | `hasChanged` | `AtomicBoolean` | Change tracking |

### Cross-Thread Execution Pattern

```java
// WRONG - Direct access from wrong thread
world.getChunk(index); // May deadlock or corrupt state

// CORRECT - Use executor for off-thread code
world.execute(() -> {
    WorldChunk chunk = world.getChunk(index);
    // Safe - runs on world thread
});

// CORRECT - Use async API
world.getChunkAsync(index).thenAccept(chunk -> {
    // Runs on world thread when complete
});
```

### Thread Check Pattern

```java
if (!world.isInThread()) {
    // Queue for world thread
    return CompletableFuture.supplyAsync(() -> {
        return doWork();
    }, world);
} else {
    // Already on world thread
    return CompletableFuture.completedFuture(doWork());
}
```

### Player State Transitions

```
PlayerRef states:
├── Holder<EntityStore> - Between worlds (safe to read)
└── Ref<EntityStore>    - In world (requires world thread)

Transitions:
  addToStore()      → Holder becomes Ref (world thread)
  removeFromStore() → Ref becomes Holder (world thread)
```

### Timeout Patterns

```java
// Prevent deadlocks with timeouts
CompletableFuture.runAsync(() -> operation(), world)
    .orTimeout(5L, TimeUnit.SECONDS)
    .whenComplete((result, error) -> {
        if (error instanceof TimeoutException) {
            // Handle timeout
        }
    });
```

### World Shutdown Safety

1. `acceptingTasks.set(false)` - Stop new task submissions
2. Wait for loading chunks with timeout
3. Drain all players to fallback world
4. Save configuration atomically
5. `alive.set(false)` - Mark as dead

---

## 7. Performance Considerations

### Save Intervals

```java
public static final float SAVE_INTERVAL = 10.0F; // seconds
```

### Task Monitoring

The world thread logs warnings for slow tasks:
```java
if (taskDuration > tickStepNanos) {
    logger.warning("Task took %s ns: %s", duration, runnable);
}
```

### Chunk Loading Backoff

```java
public static final long MAX_FAILURE_BACKOFF_NANOS = TimeUnit.SECONDS.toNanos(10L);
public static final long FAILURE_BACKOFF_NANOS = TimeUnit.MILLISECONDS.toNanos(1L);
```

### Metrics Registry

The server exposes metrics for monitoring:

| Metric | Source | Description |
|--------|--------|-------------|
| `Worlds` | Universe | World count |
| `PlayerCount` | Universe | Online players |
| `Name` | World | World identifier |
| `Alive` | World | Running state |
| `TickLength` | World | Tick duration metrics |
| `EntityStore` | World | Entity metrics |
| `ChunkStore` | World | Chunk metrics |
| `TotalGeneratedChunkCount` | ChunkStore | Generation stats |
| `TotalLoadedChunkCount` | ChunkStore | Load stats |
| `QueuedPacketsCount` | PlayerRef | Network queue depth |
| `PingInfo` | PlayerRef | Connection latency |

### Performance Best Practices

| Do | Don't |
|----|-------|
| Use async chunk APIs (`getChunkAsync`) | Block on synchronous chunk access |
| Batch entity operations | Process entities one-by-one |
| Use `LocalCachedChunkAccessor` for bulk reads | Repeatedly call `getChunk` |
| Check `isIdle()` for optional work | Run heavy tasks every tick |
| Use `isInThread()` before direct access | Assume thread context |

### Idle Detection

```java
protected boolean isIdle() {
    return this.players.isEmpty();
}
// When idle, world can reduce tick rate or skip optional work
```

### Memory Management

- `ChunkUnloadingSystem` automatically unloads distant chunks
- `WorldConfig.isUnloadingChunks` controls unloading behavior
- `WorldConfig.saveNewChunks` controls disk I/O
- GC tracking: `markGCHasRun()` / `consumeGCHasRun()`

### Configuration Tuning

| Setting | Impact | Recommendation |
|---------|--------|----------------|
| `isTicking = false` | Disables chunk updates | Use for static worlds |
| `isBlockTicking = false` | Disables block updates | Reduces CPU for display-only |
| `isUnloadingChunks = false` | Prevents unloading | Use for small, active worlds |
| `saveNewChunks = false` | Disables persistence | Use for temporary instances |
| `setTps(int)` | Dynamic tick rate | Lower for less-active worlds |

---

## 8. Events

### Universe Events

| Event | When | Location |
|-------|------|----------|
| `AddWorldEvent` | World added to universe | EventBus |
| `RemoveWorldEvent` | World removed from universe | EventBus |
| `PrepareUniverseEvent` | Universe initialization | EventBus |

### World Events

| Event | When | Location |
|-------|------|----------|
| `AddPlayerToWorldEvent` | Player enters world | World EventRegistry |
| `PlayerConnectEvent` | Player connection established | EventBus |
| `PlayerDisconnectEvent` | Player disconnected | EventBus |

### Instance Events

| Event | When | Location |
|-------|------|----------|
| `DiscoverInstanceEvent` | First entry into instance | EventBus |

---

## 9. Common Patterns

### Creating a World

```java
Universe.get().addWorld("my-world", "default", "file")
    .thenAccept(world -> {
        world.getWorldConfig().setPvpEnabled(false);
        world.setTicking(true);
    });
```

### Teleporting a Player

```java
World targetWorld = Universe.get().getWorld("target");
Transform position = new Transform(x, y, z, yaw, pitch);

targetWorld.addPlayer(playerRef, position)
    .thenAccept(player -> {
        player.sendMessage("Welcome!");
    });
```

### Creating an Instance

```java
InstancesPlugin.get()
    .spawnInstance("dungeon-template", returnWorld, returnTransform)
    .thenAccept(instance -> {
        InstanceWorldConfig config = InstanceWorldConfig.ensureAndGet(
            instance.getWorldConfig()
        );
        config.setRemovalConditions(new WorldEmptyCondition());
    });
```

### Running Code on World Thread

```java
// From any thread
world.execute(() -> {
    // This runs safely on the world thread
    WorldChunk chunk = world.getChunk(chunkIndex);
    // Modify chunk...
});
```

### Async Chunk Operation

```java
world.getChunkAsync(chunkIndex)
    .thenCompose(chunk -> {
        // Process chunk
        return CompletableFuture.completedFuture(result);
    })
    .exceptionally(error -> {
        logger.error("Chunk load failed", error);
        return fallback;
    });
```

---

## 10. Summary

| Component | Responsibility | Thread Model |
|-----------|----------------|--------------|
| **Universe** | Global container, player connections | Multi-threaded (safe) |
| **World** | Game state, chunks, entities | Single-threaded (tick thread) |
| **WorldConfig** | Persistent configuration | Immutable (mostly) |
| **InstancesPlugin** | Ephemeral world copies | Async creation |
| **InstanceWorldConfig** | Instance lifecycle | Stored in WorldConfig |
| **RemovalSystem** | Auto-cleanup instances | Per-tick evaluation |

| API Pattern | When to Use |
|-------------|-------------|
| `CompletableFuture<T>` | Cross-thread operations |
| `world.execute(Runnable)` | Queue work for world thread |
| `isInThread()` check | Before direct state access |
| `orTimeout()` | Prevent deadlocks |
| `ConcurrentHashMap` | Shared collections |
