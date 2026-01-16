# Event System Architecture Overview

## 1. Core Event Interfaces

Located in [com/hypixel/hytale/event/](https://github.com/Savag3life/TaleServer/tree/main/hytale/main/src/com/hypixel/hytale/event/)

| Interface | Purpose |
|-----------|---------|
| `IBaseEvent<KeyType>` | Root marker interface for all events with generic routing key |
| `IEvent<KeyType>` | Synchronous events - blocks until all handlers complete |
| `IAsyncEvent<KeyType>` | Asynchronous events - returns `CompletableFuture` |
| `ICancellable` | Events that can be cancelled before processing |
| `IProcessedEvent` | Events that track post-processing status |

---

## 2. Event Type Hierarchies

### ECS Events (Entity Component System)

Located in [component/system/](https://github.com/Savag3life/TaleServer/tree/main/hytale/main/src/com/hypixel/hytale/component/system/)

```
EcsEvent (base)
├── CancellableEcsEvent implements ICancellableEcsEvent
│   ├── BreakBlockEvent
│   ├── PlaceBlockEvent
│   ├── DiscoverZoneEvent
│   ├── ChangeGameModeEvent
│   ├── CraftRecipeEvent
│   ├── DamageBlockEvent
│   ├── DropItemEvent
│   └── ... (many more)
└── UseBlockEvent
    ├── UseBlockEvent.Pre (cancellable)
    └── UseBlockEvent.Post (notification only)
```

### Player Events

Located in [server/core/event/events/](https://github.com/Savag3life/TaleServer/tree/main/hytale/main/src/com/hypixel/hytale/server/core/event/events/)

```
PlayerEvent<KeyType> implements IEvent<KeyType>
├── PlayerConnectEvent
├── PlayerDisconnectEvent
├── PlayerChatEvent implements ICancellable
├── PlayerInteractEvent implements ICancellable
├── PlayerMouseButtonEvent
├── PlayerMouseMotionEvent
├── PlayerCraftEvent
├── PlayerReadyEvent
└── AddPlayerToWorldEvent
```

### Entity Events

```
EntityEvent<EntityType, KeyType> implements IEvent<KeyType>
├── EntityRemoveEvent
├── LivingEntityInventoryChangeEvent
└── LivingEntityUseBlockEvent
```

### System Lifecycle Events

```
BootEvent implements IEvent<Void>
PrepareUniverseEvent implements IEvent<Void>
ShutdownEvent implements IEvent<Void>
```

### Asset Store Events

Located in [assetstore/event/](https://github.com/Savag3life/TaleServer/tree/main/hytale/main/src/com/hypixel/hytale/assetstore/event/)

```
AssetStoreEvent<KeyType>
├── RegisterAssetStoreEvent
└── RemoveAssetStoreEvent

AssetsEvent<K, T>
├── GenerateAssetsEvent
├── LoadedAssetsEvent
└── RemovedAssetsEvent
```

---

## 3. Event Bus Architecture

### Main Components

| Class | Purpose |
|-------|---------|
| `EventBus` | Main event bus implementation |
| `SyncEventBusRegistry` | Handles synchronous `IEvent` dispatch |
| `AsyncEventBusRegistry` | Handles async `IAsyncEvent` with `CompletableFuture` |
| `EventConsumerMap` | Stores listeners organized by priority |

### Registration Types

1. **Keyed Registration** - Context-specific (e.g., world-specific events)
   ```java
   eventBus.register(BreakBlockEvent.class, worldKey, event -> { ... });
   ```

2. **Global Registration** - Receives ALL events of that type
   ```java
   eventBus.registerGlobal(BreakBlockEvent.class, event -> { ... });
   ```

3. **Unhandled Registration** - Fallback when keyed handlers don't process
   ```java
   eventBus.registerUnhandled(SomeEvent.class, event -> { ... });
   ```

### Priority System

| Priority | Value | Use Case |
|----------|-------|----------|
| `FIRST` | -21844 | Must run before all others |
| `EARLY` | -10922 | Run early in chain |
| `NORMAL` | 0 | Default priority |
| `LATE` | 10922 | Run after most handlers |
| `LAST` | 21844 | Must run after all others |

---

## 4. Dispatch Flow

### Synchronous Events

```
eventBus.dispatchFor(EventClass.class, key)
  └── SyncEventBusRegistry.dispatchFor(key)
      └── SyncEventConsumerMap.dispatch(event)
          ├── Dispatch keyed listeners (by priority)
          ├── If handled → return event
          ├── Else → dispatch unhandled listeners
          └── Return modified event
```

### Asynchronous Events

```
eventBus.dispatchForAsync(EventClass.class, key)
  └── AsyncEventBusRegistry.dispatchFor(key)
      └── AsyncEventConsumerMap.dispatch(event)
          └── Returns CompletableFuture chain
```

---

## 5. What is a Query?

**Queries are part of the ECS (Entity Component System)**, not the event system directly. They filter which entities a system should operate on.

Located in [component/query/](https://github.com/Savag3life/TaleServer/tree/main/hytale/main/src/com/hypixel/hytale/component/query/)

### Query Types

| Query Type | Purpose |
|------------|---------|
| `AnyQuery` | Matches all archetypes (entities) |
| `ExactArchetypeQuery` | Matches specific archetype only |
| `NotQuery` | Negates another query |
| `AndQuery` | Logical AND of multiple queries |
| `OrQuery` | Logical OR of multiple queries |
| `ReadWriteArchetypeQuery` | For read/write component operations |

### Query Usage with Events

```java
public abstract class EntityEventSystem<ECS_TYPE, EventType extends EcsEvent>
    extends EventSystem<EventType> implements QuerySystem<ECS_TYPE> {

    @Override
    public Query<ECS_TYPE> getQuery() {
        // Define which entities receive this event
        return Query.and(hasHealthComponent, hasPositionComponent);
    }

    @Override
    public void handle(int index, ArchetypeChunk<ECS_TYPE> chunk,
                       Store<ECS_TYPE> store, CommandBuffer<ECS_TYPE> buffer,
                       EventType event) {
        // Only called for entities matching the query
    }
}
```

**Purpose**: When an `EcsEvent` is dispatched, the `EntityEventSystem` uses its `Query` to filter which entities should receive and process that event. This is how the ECS efficiently routes events only to relevant entities.

---

## 6. Event Implementation Patterns

### Simple Event

```java
public class BootEvent implements IEvent<Void> {
    // No additional fields needed
}
```

### Cancellable Event

```java
public class PlayerChatEvent extends PlayerEvent<Void> implements ICancellable {
    private boolean cancelled;

    @Override
    public boolean isCancelled() { return cancelled; }

    @Override
    public void setCancelled(boolean cancelled) { this.cancelled = cancelled; }
}
```

### Pre/Post Event Pattern

```java
public abstract class UseBlockEvent extends EcsEvent {
    // Common fields...

    public static final class Pre extends UseBlockEvent implements ICancellableEcsEvent {
        // Can be cancelled before action
    }

    public static final class Post extends UseBlockEvent {
        // Notification only, after action completed
    }
}
```

---

## 7. Registration & Handling Examples

### Basic Registration

```java
eventBus.register(PlayerConnectEvent.class, event -> {
    Player player = event.getPlayer();
    // Handle connection
});
```

### Priority Registration

```java
eventBus.register(EventPriority.EARLY, BreakBlockEvent.class, event -> {
    // Runs before NORMAL priority handlers
});
```

### Cancellation

```java
eventBus.register(BreakBlockEvent.class, event -> {
    if (!hasPermission(player)) {
        event.setCancelled(true);  // Prevent block break
    }
});
```

### Async Registration

```java
eventBus.registerAsync(SomeAsyncEvent.class, future ->
    future.thenApply(event -> {
        // Process asynchronously
        return event;
    }));
```

### Unregistration

```java
EventRegistration registration = eventBus.register(SomeEvent.class, handler);
// Later...
registration.unregister();
```

---

## 8. Summary

| Feature | Implementation |
|---------|----------------|
| **No annotations** | Uses functional `Consumer<T>` and `Function<>` patterns |
| **Type safety** | Generics throughout (`IEvent<KeyType>`) |
| **Thread safety** | `ConcurrentHashMap`, `CopyOnWriteArrayList`, `AtomicReference` |
| **Cancellation** | `ICancellable` and `ICancellableEcsEvent` interfaces |
| **Priorities** | 5 predefined + custom short values |
| **Sync/Async** | Separate registries for each |
| **ECS Integration** | `Query` system filters entities for event handling |
| **Total events** | ~107 concrete event implementations |

---

## 9. Complete Event Reference

This section provides a comprehensive list of all events in the codebase, organized by their handler/listener system.

### Server Core Events

Located in [server/core/event/](https://github.com/Savag3life/TaleServer/tree/main/hytale/main/src/com/hypixel/hytale/server/core/event/)

#### Lifecycle Events

| Event | Key Type | Cancellable | Description |
|-------|----------|-------------|-------------|
| `BootEvent` | `Void` | No | Fired when server boots |
| `ShutdownEvent` | `Void` | No | Fired when server shuts down |
| `PrepareUniverseEvent` | `Void` | No | Universe preparation (deprecated) |

#### Player Events

| Event | Key Type | Cancellable | Description |
|-------|----------|-------------|-------------|
| `PlayerConnectEvent` | `Void` | No | Player connects to server |
| `PlayerDisconnectEvent` | `Void` | No | Player disconnects from server |
| `PlayerReadyEvent` | `Void` | No | Player is fully loaded and ready |
| `PlayerSetupConnectEvent` | `Void` | Yes | Player setup phase connection |
| `PlayerSetupDisconnectEvent` | `Void` | No | Player setup phase disconnection |
| `PlayerChatEvent` | `String` | Yes | Player sends chat message |
| `PlayerCraftEvent` | `String` | No | Player crafts an item |
| `PlayerInteractEvent` | `String` | Yes | Player interacts with something |
| `PlayerMouseButtonEvent` | `Void` | No | Mouse button input |
| `PlayerMouseMotionEvent` | `Void` | No | Mouse motion input |
| `AddPlayerToWorldEvent` | `String` | No | Player added to world (key: world name) |
| `DrainPlayerFromWorldEvent` | `String` | No | Player removed from world (key: world name) |

#### Entity Events

| Event | Key Type | Cancellable | Description |
|-------|----------|-------------|-------------|
| `EntityRemoveEvent` | - | No | Entity is removed |
| `LivingEntityInventoryChangeEvent` | - | No | Entity inventory changed |
| `LivingEntityUseBlockEvent` | `String` | No | Entity uses a block |

#### World/Chunk Events

| Event | Key Type | Cancellable | Description |
|-------|----------|-------------|-------------|
| `AddWorldEvent` | `String` | Yes | World is being added |
| `RemoveWorldEvent` | `String` | No | World is being removed |
| `StartWorldEvent` | `String` | No | World has started |
| `AllWorldsLoadedEvent` | `String` | No | All worlds finished loading |
| `ChunkPreLoadProcessEvent` | - | No | Before chunk loading |
| `ChunkSaveEvent` | - | No | Chunk being saved (ECS) |
| `ChunkUnloadEvent` | - | No | Chunk being unloaded (ECS) |
| `MoonPhaseChangeEvent` | - | No | Moon phase changed (ECS) |
| `WorldPathChangedEvent` | `Void` | No | World path changed |

#### Permission Events

| Event | Key Type | Description |
|-------|----------|-------------|
| `PlayerPermissionChangeEvent` | `Void` | Player permission changed |
| `GroupPermissionChangeEvent` | `Void` | Group permission changed |
| `PlayerGroupEvent` | `Void` | Player group membership changed |

#### Plugin Events

| Event | Key Type | Description |
|-------|----------|-------------|
| `PluginSetupEvent` | `Class<? extends PluginBase>` | Plugin is being set up |

#### Prefab Events

| Event | Description |
|-------|-------------|
| `PrefabPasteEvent` | Prefab is being pasted |
| `PrefabPlaceEntityEvent` | Entity placed from prefab |

#### Asset Management Events

| Event | Key Type | Description |
|-------|----------|-------------|
| `AssetPackRegisterEvent` | `Void` | Asset pack registered |
| `AssetPackUnregisterEvent` | `Void` | Asset pack unregistered |
| `LoadAssetEvent` | `Void` | Asset loaded |
| `GenerateSchemaEvent` | `Void` | Schema generation |
| `CommonAssetMonitorEvent` | - | Common asset monitoring |
| `SendCommonAssetsEvent` | - | Sending common assets |

#### Module-Specific Events

| Event | Module | Description |
|-------|--------|-------------|
| `GenerateDefaultLanguageEvent` | i18n | Default language generation |
| `CombatTextUIComponentAnimationEvent` | Entity UI | Combat text animation |
| `CombatTextUIComponentOpacityAnimationEvent` | Entity UI | Combat text opacity |
| `CombatTextUIComponentPositionAnimationEvent` | Entity UI | Combat text position |
| `CombatTextUIComponentScaleAnimationEvent` | Entity UI | Combat text scale |
| `KillFeedEvent` | Entity Damage | Kill feed notification |
| `SingleplayerRequestAccessEvent` | Singleplayer | Singleplayer access request |

---

### ECS System Events

Located in [server/core/event/events/ecs/](https://github.com/Savag3life/TaleServer/tree/main/hytale/main/src/com/hypixel/hytale/server/core/event/events/ecs/)

These events extend `EcsEvent` or `CancellableEcsEvent` and are dispatched through the Entity Component System.

| Event | Cancellable | Description |
|-------|-------------|-------------|
| `BreakBlockEvent` | Yes | Block is being broken |
| `PlaceBlockEvent` | Yes | Block is being placed |
| `DamageBlockEvent` | Yes | Block is being damaged |
| `UseBlockEvent` | No | Block is being used |
| `UseBlockEvent.Pre` | Yes | Before block use (cancellable) |
| `DropItemEvent` | Yes | Item is being dropped |
| `InteractivelyPickupItemEvent` | Yes | Item is being picked up |
| `SwitchActiveSlotEvent` | Yes | Active slot is being switched |
| `ChangeGameModeEvent` | Yes | Game mode is being changed |
| `CraftRecipeEvent` | Yes | Recipe is being crafted |
| `DiscoverZoneEvent` | No | Zone discovered |
| `DiscoverZoneEvent.Display` | Yes | Zone discovery display |

---

### AssetStore Events

Located in [assetstore/event/](https://github.com/Savag3life/TaleServer/tree/main/hytale/main/src/com/hypixel/hytale/assetstore/event/)

| Event | Key Type | Description |
|-------|----------|-------------|
| `RegisterAssetStoreEvent` | `Void` | Asset store registered |
| `RemoveAssetStoreEvent` | `Void` | Asset store removed |
| `GenerateAssetsEvent<T>` | Asset type | Assets generated |
| `LoadedAssetsEvent<T>` | Asset type | Assets loaded |
| `RemovedAssetsEvent<T>` | Asset type | Assets removed |
| `AssetMonitorEvent` | - | Asset monitoring |
| `AssetStoreMonitorEvent` | - | Store monitoring |
| `AssetsEvent` | - | Generic assets event |

---

### Builtin Module Events

#### Asset Editor Events

Located in [builtin/asseteditor/event/](https://github.com/Savag3life/TaleServer/tree/main/hytale/main/src/com/hypixel/hytale/builtin/asseteditor/event/)

| Event | Key Type | Async | Description |
|-------|----------|-------|-------------|
| `AssetEditorActivateButtonEvent` | `String` (buttonId) | No | Button activated |
| `AssetEditorAssetCreatedEvent` | Asset type ID | No | Asset created |
| `AssetEditorClientDisconnectEvent` | - | No | Client disconnected |
| `AssetEditorFetchAutoCompleteDataEvent` | Dataset name | Yes | Fetch autocomplete |
| `AssetEditorRequestDataSetEvent` | Dataset name | Yes | Request dataset |
| `AssetEditorSelectAssetEvent` | - | No | Asset selected |
| `AssetEditorUpdateWeatherPreviewLockEvent` | - | No | Weather preview lock |

#### Instance/Adventure Events

Located in [builtin/instances/](https://github.com/Savag3life/TaleServer/tree/main/hytale/main/src/com/hypixel/hytale/builtin/instances/) and [builtin/adventure/](https://github.com/Savag3life/TaleServer/tree/main/hytale/main/src/com/hypixel/hytale/builtin/adventure/)

| Event | Key Type | Description |
|-------|----------|-------------|
| `DiscoverInstanceEvent` | - | Instance discovered (ECS) |
| `DiscoverInstanceEvent.Display` | - | Instance discovery display (cancellable) |
| `TreasureChestOpeningEvent` | `String` (world name) | Treasure chest opening |
| `VoidEvent` | - | Void event (ECS) |

---

### Protocol Events

Located in [protocol/](https://github.com/Savag3life/TaleServer/tree/main/hytale/main/src/com/hypixel/hytale/protocol/)

These events handle client-server communication.

| Event | Description |
|-------|-------------|
| `SoundEvent` | Sound playback |
| `BlockSoundEvent` | Block-related sound |
| `ItemSoundEvent` | Item-related sound |
| `BlockParticleEvent` | Block particle effects |
| `MouseButtonEvent` | Mouse button input packet |
| `MouseMotionEvent` | Mouse motion input packet |
| `CombatTextEntityUIComponentAnimationEvent` | Combat text animation packet |
| `ItemReticleClientEvent` | Item reticle update |
| `CustomPageEvent` | Custom UI page event |
| `ReticleEvent` | Reticle state event |

---

### NPC System Events

Located in [server/npc/](https://github.com/Savag3life/TaleServer/tree/main/hytale/main/src/com/hypixel/hytale/server/npc/)

| Event | Description |
|-------|-------------|
| `SensorEvent` | Base sensor event |
| `SensorEntityEvent` | Entity-based sensor event |
| `BuilderSensorEvent` | Builder-pattern sensor config |
| `BuilderSensorEntityEvent` | Builder-pattern entity sensor |
| `AllNPCsLoadedEvent` | All NPCs finished loading |
| `LoadedNPCEvent` | Single NPC loaded |

**Sensor Event Types:**

| Type | Description |
|------|-------------|
| `PlayerFirst` | Player takes priority |
| `PlayerOnly` | Only players trigger |
| `NpcFirst` | NPC takes priority |
| `NpcOnly` | Only NPCs trigger |

---

### Component/ECS Base Types

Located in [component/system/](https://github.com/Savag3life/TaleServer/tree/main/hytale/main/src/com/hypixel/hytale/component/system/)

| Class | Purpose |
|-------|---------|
| `EcsEvent` | Base for all ECS events |
| `CancellableEcsEvent` | Cancellable ECS events |
| `ICancellableEcsEvent` | Interface for cancellable ECS events |
| `EventSystemType<ECS_TYPE, Event, SYSTEM_TYPE>` | Event system type registration |
| `WorldEventType<ECS_TYPE, Event>` | World-scoped event system |
| `EntityEventType<ECS_TYPE, Event>` | Entity-scoped event system |

---

## 10. Event Handler Reference

### Known Plugin Handlers

| Handler | Events |
|---------|--------|
| `ObjectivePlugin` | `PlayerDisconnectEvent` |
| `BuilderToolsPlugin` | `PlayerConnectEvent`, `PlayerDisconnectEvent` |
| `CreativeHubPlugin` | `PlayerConnectEvent` (global) |
| `InstancesPlugin` | `PlayerConnectEvent` |
| `MountPlugin` | `PlayerDisconnectEvent` |
| `ServerPlayerListModule` | `PlayerConnectEvent`, `PlayerDisconnectEvent` |
| `AssetEditorPlugin` | All `AssetEditor*` events |
| `CraftingManager` | `PlayerCraftEvent` (world-keyed) |

### Known Module Handlers

| Module | Events |
|--------|--------|
| `AssetModule` | `BootEvent` |
| `MigrationModule` | `BootEvent` |
| `PermissionsModule` | Permission events |
| `InteractionModule` | Block interaction events |
| `I18nModule` | `GenerateDefaultLanguageEvent` |
| `SingleplayerModule` | `SingleplayerRequestAccessEvent` |

### Packet Handlers

| Handler | Events |
|---------|--------|
| `AssetEditorPacketHandler` | `AssetEditor*` events (sync/async) |
| `GamePacketHandler` | Input events |
| `SetupPacketHandler` | Setup events |
