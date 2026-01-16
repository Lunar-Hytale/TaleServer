# Plugin System Architecture Overview

The Hytale server uses a **modular plugin architecture** with JAR-based distribution, isolated class loading, semantic versioning for dependencies, and a comprehensive lifecycle management system.

---

## 1. Core Plugin Classes

Located in [common/plugin/](https://github.com/Savag3life/TaleServer/tree/main/hytale/main/src/com/hypixel/hytale/common/plugin/)

| Class | Purpose |
|-------|---------|
| `PluginManager` | Singleton orchestrator for loading, lifecycle, and registry management |
| `PluginBase` | Abstract foundation for all plugins with registry accessors |
| `JavaPlugin` | Concrete implementation for JAR-based plugins |
| `PluginManifest` | Immutable metadata holder parsed from `manifest.json` |
| `PluginIdentifier` | Value object for `Group:Name` plugin identification |
| `PluginState` | Enum defining the 6 lifecycle states |
| `PluginClassLoader` | Isolated `URLClassLoader` per plugin |

---

## 2. Plugin Lifecycle

Plugins transition through 6 distinct states managed by `PluginManager`:

```
NONE → SETUP → START → ENABLED → SHUTDOWN → DISABLED
```

### Lifecycle States and Methods

| State | Method Called | Description |
|-------|---------------|-------------|
| `NONE` | - | Initial state after creation |
| `SETUP` | `preLoad()`, `setup()` | Registries initialized, configs loaded |
| `START` | `start()` | Transitional state before enabled |
| `ENABLED` | - | Fully operational |
| `SHUTDOWN` | `shutdown()` | Cleanup in progress |
| `DISABLED` | - | Final state, resources released |

### Lifecycle Flow

```
1. Plugin Discovery
   └── Scan JAR files for manifest.json
   └── Parse manifest and validate dependencies

2. preLoad() Phase (Async)
   └── Configuration loading via CompletableFuture
   └── Called before dependencies are fully satisfied

3. setup() Phase
   └── Register commands, events, components
   └── Dependencies must reach SETUP state first
   └── State transitions to SETUP on success

4. start() Phase
   └── Enable functionality
   └── All dependencies must be ENABLED
   └── State transitions to ENABLED on success

5. shutdown() Phase
   └── Called on disable or server shutdown
   └── Cleanup executed in reverse load order
   └── All registries auto-cleaned
```

### Implementing Lifecycle Methods

```java
public class MyPlugin extends JavaPlugin {
    public MyPlugin(@Nonnull JavaPluginInit init) {
        super(init);
    }

    @Override
    protected void preLoad() {
        // Async config loading (optional)
        this.withConfig(MyConfigCodec.CODEC);
    }

    @Override
    protected void setup() {
        // Register commands
        this.getCommandRegistry().registerCommand(new MyCommand());

        // Register event listeners
        this.getEventRegistry().listen(PlayerConnectEvent.class, this::onPlayerConnect);

        // Register components
        this.getEntityStoreRegistry().registerComponent(MyComponent.class);
    }

    @Override
    protected void start() {
        // Enable functionality
        this.getLogger().info("Plugin started!");
    }

    @Override
    protected void shutdown() {
        // Cleanup resources
        this.getLogger().info("Plugin shutting down!");
    }
}
```

---

## 3. Plugin Manifest

Every plugin requires a `manifest.json` file embedded in the JAR root.

### Manifest Structure

```json
{
  "Group": "MyOrganization",
  "Name": "MyPlugin",
  "Version": "1.0.0",
  "Description": "A description of what this plugin does",
  "Authors": [
    {
      "Name": "Developer Name",
      "Email": "dev@example.com",
      "Website": "https://example.com"
    }
  ],
  "Website": "https://plugin-website.com",
  "Main": "com.example.myplugin.MyPlugin",
  "ServerVersion": ">=1.0.0",
  "Dependencies": {
    "Hytale:EventSystem": ">=1.0.0"
  },
  "OptionalDependencies": {
    "Hytale:LoggingSystem": ">=1.0.0"
  },
  "LoadBefore": {
    "OtherOrg:OtherPlugin": ">=1.0.0"
  },
  "DisabledByDefault": false,
  "IncludesAssetPack": false,
  "SubPlugins": []
}
```

### Manifest Fields Reference

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `Group` | String | Yes | Organization/author identifier |
| `Name` | String | Yes | Plugin name (unique within group) |
| `Version` | String | No | Semantic version (e.g., `1.2.3`, `1.0.0-beta`) |
| `Description` | String | No | Human-readable description |
| `Authors` | AuthorInfo[] | No | Array of author information |
| `Website` | String | No | Plugin website URL |
| `Main` | String | No | Fully qualified main class name |
| `ServerVersion` | SemverRange | No | Required server version range |
| `Dependencies` | Map | No | Required plugin dependencies |
| `OptionalDependencies` | Map | No | Optional plugin dependencies |
| `LoadBefore` | Map | No | Plugins that should load after this one |
| `DisabledByDefault` | Boolean | No | If true, plugin starts disabled |
| `IncludesAssetPack` | Boolean | No | If true, registers embedded asset pack |
| `SubPlugins` | PluginManifest[] | No | Nested sub-plugin definitions |

### Plugin Identifier Format

Dependencies use the `Group:Name` format:

```json
{
  "Dependencies": {
    "Hytale:BlockPhysics": ">=1.0.0",
    "MyOrg:CoreLib": "^2.0.0"
  }
}
```

### Semver Range Syntax

| Syntax | Meaning |
|--------|---------|
| `1.2.3` | Exact version |
| `>=1.0.0` | Version 1.0.0 or higher |
| `^1.2.0` | Compatible with 1.2.0 (same major) |
| `~1.2.0` | Approximately 1.2.0 (same major.minor) |
| `1.0.0 - 2.0.0` | Range between versions |

---

## 4. Plugin Dependencies

### Dependency Types

| Type | Behavior |
|------|----------|
| **Required** (`Dependencies`) | Must exist and match version; plugin fails if not satisfied |
| **Optional** (`OptionalDependencies`) | Can be missing; participates in load order if present |
| **LoadBefore** | Creates reverse dependency; target loads after this plugin |

### Dependency Resolution Process

```
1. Validation Phase
   └── Check server version compatibility
   └── Verify all required dependencies exist
   └── Validate version ranges match
   └── Throw MissingPluginDependencyException on failure

2. Load Order Calculation
   └── Build dependency graph (DAG)
   └── Topological sort for load order
   └── Detect and report cyclic dependencies
   └── Classpath plugins load before external plugins

3. Runtime State Checking
   └── At each lifecycle stage, verify dependency states
   └── Dependencies must reach required state before dependent advances
```

### Dependency Example

```java
// Plugin A (no dependencies)
// manifest.json: { "Group": "MyOrg", "Name": "CoreLib", "Version": "1.0.0" }

// Plugin B (depends on A)
// manifest.json:
{
  "Group": "MyOrg",
  "Name": "FeaturePlugin",
  "Version": "1.0.0",
  "Dependencies": {
    "MyOrg:CoreLib": ">=1.0.0"
  }
}

// Load order: CoreLib → FeaturePlugin
// Setup order: CoreLib.setup() → FeaturePlugin.setup()
// Shutdown order: FeaturePlugin.shutdown() → CoreLib.shutdown()
```

### Accessing Dependencies at Runtime

```java
@Override
protected void setup() {
    PluginManager pm = PluginManager.get();

    // Check if optional dependency is loaded
    Optional<PluginBase> optionalPlugin = pm.getPlugin(
        PluginIdentifier.of("MyOrg", "OptionalFeature")
    );

    if (optionalPlugin.isPresent()) {
        // Use optional feature
    }
}
```

---

## 5. Class Loading and Isolation

### Class Loader Architecture

```
PluginClassLoader (extends URLClassLoader)
├── BuiltinPlugin (classpath plugins)
└── ThirdPartyPlugin (external JAR plugins)

PluginBridgeClassLoader
└── Manages cross-plugin class visibility
└── Checks manifest dependencies before allowing access
```

### Class Visibility Rules

1. Plugin can access its own classes
2. Plugin can access server/library classes
3. Plugin can access classes from declared dependencies
4. Plugin can access classes from optional dependencies (if present)
5. Plugin **cannot** access classes from unrelated plugins

### Example Class Loading

```java
// Plugin A exports: com.myorg.corelib.api.*
// Plugin B depends on A

// In Plugin B:
import com.myorg.corelib.api.CoreService;  // Works - declared dependency

// In Plugin C (no dependency on A):
import com.myorg.corelib.api.CoreService;  // Fails - ClassNotFoundException
```

---

## 6. Plugin Registries

Each plugin has access to multiple registries through `PluginBase`:

| Registry | Purpose | Access Method |
|----------|---------|---------------|
| `CommandRegistry` | Register commands | `getCommandRegistry()` |
| `EventRegistry` | Register event listeners | `getEventRegistry()` |
| `TaskRegistry` | Schedule tasks | `getTaskRegistry()` |
| `AssetRegistry` | Register assets | `getAssetRegistry()` |
| `CodecRegistry` | Register codecs | `getCodecRegistry()` |
| `ClientFeatureRegistry` | Client features | `getClientFeatureRegistry()` |
| `BlockStateRegistry` | Custom block states | `getBlockStateRegistry()` |
| `EntityRegistry` | Custom entity types | `getEntityRegistry()` |
| `EntityStoreRegistry` | Entity store components | `getEntityStoreRegistry()` |
| `ChunkStoreRegistry` | Chunk store components | `getChunkStoreRegistry()` |

### Registry Auto-Cleanup

All registrations are automatically cleaned up when the plugin shuts down:

```java
@Override
protected void setup() {
    // These are automatically unregistered on shutdown
    this.getCommandRegistry().registerCommand(new MyCommand());
    this.getEventRegistry().listen(SomeEvent.class, this::handleEvent);
}
// No manual cleanup needed in shutdown()
```

---

## 7. Plugin Configuration

### Defining Configuration

```java
public class MyPluginConfig {
    public static final BuilderCodec<MyPluginConfig> CODEC = BuilderCodec.of(MyPluginConfig.class)
        .with("enabled", Validators.nonNull(), c -> c.enabled, true)
        .with("maxPlayers", Validators.range(1, 100), c -> c.maxPlayers, 20)
        .with("welcomeMessage", c -> c.welcomeMessage, "Welcome!")
        .build();

    public boolean enabled;
    public int maxPlayers;
    public String welcomeMessage;
}
```

### Loading Configuration

```java
public class MyPlugin extends JavaPlugin {
    private Config<MyPluginConfig> config;

    @Override
    protected void preLoad() {
        this.config = this.withConfig(MyPluginConfig.CODEC);
    }

    @Override
    protected void setup() {
        MyPluginConfig cfg = this.config.get();
        if (cfg.enabled) {
            this.getLogger().info("Max players: " + cfg.maxPlayers);
        }
    }
}
```

### Configuration File Location

Configs are stored in the plugin's data directory:
```
mods/<Group>_<Name>/config.json
```

---

## 8. Plugin Discovery Paths

Plugins are discovered from multiple locations in this order:

| Source | Location | Priority |
|--------|----------|----------|
| Core Plugins | Registered via `registerCorePlugin()` | Highest |
| Classpath Plugins | `manifest.json` in classpath | High |
| Builtin Plugins | `<server>/builtin/` directory | Medium |
| Mod Plugins | `mods/` directory | Normal |
| Custom Directories | Via `--mods-directories` flag | Normal |

### Directory Structure

```
server/
├── builtin/
│   ├── builtin-plugin-1.jar
│   └── builtin-plugin-2.jar
├── mods/
│   ├── my-plugin.jar
│   └── another-plugin.jar
└── hytale-server.jar
```

---

## 9. Dynamic Plugin Management

### Load/Unload at Runtime

```java
PluginManager pm = PluginManager.get();
PluginIdentifier id = PluginIdentifier.of("MyOrg", "MyPlugin");

// Load a disabled plugin
boolean loaded = pm.load(id);

// Unload an enabled plugin
boolean unloaded = pm.unload(id);

// Reload a plugin (unload + load)
boolean reloaded = pm.reload(id);
```

### Plugin Command

Built-in `/plugin` command provides:
- List all plugins and their states
- Enable/disable plugins dynamically
- View plugin information

---

## 10. Core Plugin Registration

For plugins bundled with the server (not in JARs):

```java
public class MyCorePlugin extends JavaPlugin {
    public static final PluginManifest MANIFEST = PluginManifest.corePlugin(MyCorePlugin.class)
        .description("A core plugin")
        .version("1.0.0")
        .depends(OtherCorePlugin.MANIFEST)
        .build();

    private static MyCorePlugin instance;

    public MyCorePlugin(@Nonnull JavaPluginInit init) {
        super(init);
        instance = this;
    }

    public static MyCorePlugin getInstance() {
        return instance;
    }

    @Override
    protected void setup() {
        // Setup logic
    }
}
```

---

## 11. Sub-Plugins

Plugins can define nested sub-plugins in their manifest:

```json
{
  "Group": "MyOrg",
  "Name": "MainPlugin",
  "Version": "1.0.0",
  "Main": "com.myorg.MainPlugin",
  "SubPlugins": [
    {
      "Group": "MyOrg",
      "Name": "SubFeatureA",
      "Main": "com.myorg.features.FeatureA"
    },
    {
      "Group": "MyOrg",
      "Name": "SubFeatureB",
      "Main": "com.myorg.features.FeatureB"
    }
  ]
}
```

Sub-plugins:
- Share the same classloader as parent
- Loaded as separate plugin instances
- Inherit parent manifest values
- Useful for modular plugin architectures

---

## 12. Asset Pack Integration

Plugins can include asset packs:

```json
{
  "Group": "MyOrg",
  "Name": "MyPlugin",
  "IncludesAssetPack": true
}
```

The asset pack is automatically registered during `start()` if:
- `IncludesAssetPack` is `true`
- JAR contains valid asset pack structure

---

## 13. Error Handling

### Common Exceptions

| Exception | Cause |
|-----------|-------|
| `MissingPluginDependencyException` | Required dependency not found or version mismatch |
| `ClassNotFoundException` | Main class not found in JAR |
| `NoSuchMethodException` | Missing `JavaPluginInit` constructor |

### Lifecycle Error Behavior

- Exceptions in `setup()`, `start()`, or `shutdown()` move plugin to `DISABLED`
- Full stack trace logged with context
- Dependent plugins may also fail

### Validation Errors

```
[ERROR] Plugin validation failed for MyOrg:MyPlugin
  - Missing required dependency: OtherOrg:RequiredPlugin
  - Version mismatch: Hytale:CoreLib requires >=2.0.0, found 1.5.0
```

---

## 14. Thread Safety

### Synchronization in PluginManager

- `ReentrantReadWriteLock` protects plugin registry
- `ConcurrentHashMap` for classloader caching
- `CopyOnWriteArrayList` for safe iteration

### Safe Plugin Access Pattern

```java
// Reading plugin list
PluginManager pm = PluginManager.get();
List<PluginBase> plugins = pm.getPlugins();  // Returns defensive copy

// Getting specific plugin
Optional<PluginBase> plugin = pm.getPlugin(identifier);
plugin.ifPresent(p -> {
    // Safe to use
});
```

---

## 15. Complete Plugin Example

### manifest.json

```json
{
  "Group": "ExampleOrg",
  "Name": "WelcomePlugin",
  "Version": "1.0.0",
  "Description": "Welcomes players when they join",
  "Authors": [
    { "Name": "Developer", "Email": "dev@example.org" }
  ],
  "Main": "org.example.welcome.WelcomePlugin",
  "Dependencies": {
    "Hytale:EventSystem": ">=1.0.0"
  }
}
```

### WelcomePlugin.java

```java
package org.example.welcome;

import com.hypixel.hytale.common.plugin.JavaPlugin;
import com.hypixel.hytale.common.plugin.JavaPluginInit;
import com.hypixel.hytale.server.core.Message;
import com.hypixel.hytale.server.core.event.events.PlayerConnectEvent;
import javax.annotation.Nonnull;

public class WelcomePlugin extends JavaPlugin {

    public WelcomePlugin(@Nonnull JavaPluginInit init) {
        super(init);
    }

    @Override
    protected void setup() {
        this.getEventRegistry().listen(PlayerConnectEvent.class, this::onPlayerConnect);
        this.getCommandRegistry().registerCommand(new WelcomeCommand());

        this.getLogger().info("WelcomePlugin setup complete");
    }

    @Override
    protected void start() {
        this.getLogger().info("WelcomePlugin enabled");
    }

    @Override
    protected void shutdown() {
        this.getLogger().info("WelcomePlugin disabled");
    }

    private void onPlayerConnect(PlayerConnectEvent event) {
        event.getPlayer().sendMessage(
            Message.translation("welcome.message")
                .param("player", event.getPlayer().getUsername())
        );
    }
}
```

### Project Structure

```
welcome-plugin/
├── src/
│   └── org/example/welcome/
│       ├── WelcomePlugin.java
│       └── WelcomeCommand.java
├── resources/
│   └── manifest.json
└── build.gradle
```

---

## 16. Summary

| Feature | Implementation |
|---------|----------------|
| **Distribution** | JAR files with embedded `manifest.json` |
| **Lifecycle** | 6 states (NONE → SETUP → START → ENABLED → SHUTDOWN → DISABLED) |
| **Dependencies** | Semantic versioning with topological sort |
| **Class Loading** | Isolated `URLClassLoader` per plugin with bridge for visibility |
| **Discovery** | Core, classpath, builtin/, mods/, custom directories |
| **Configuration** | Async loading via `BuilderCodec` in `preLoad()` |
| **Registries** | 10+ auto-cleaned registries (commands, events, tasks, etc.) |
| **Thread Safety** | `ReentrantReadWriteLock` + `ConcurrentHashMap` |
| **Dynamic Management** | Runtime load/unload/reload via `PluginManager` |
| **Sub-Plugins** | Nested plugin definitions sharing parent classloader |
