# Event System Architecture Overview

## 1. Core Event Interfaces

Located in [com/hypixel/hytale/event/](https://github.com/Savag3life/HytaleServer/tree/main/hytale/main/src/com/hypixel/hytale/event/)

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

Located in [component/system/](https://github.com/Savag3life/HytaleServer/tree/main/hytale/main/src/com/hypixel/hytale/component/system/)

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

Located in [server/core/event/events/](https://github.com/Savag3life/HytaleServer/tree/main/hytale/main/src/com/hypixel/hytale/server/core/event/events/)

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

Located in [assetstore/event/](https://github.com/Savag3life/HytaleServer/tree/main/hytale/main/src/com/hypixel/hytale/assetstore/event/)

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

Located in [component/query/](https://github.com/Savag3life/HytaleServer/tree/main/hytale/main/src/com/hypixel/hytale/component/query/)

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
| **Total events** | ~50+ concrete event implementations |
