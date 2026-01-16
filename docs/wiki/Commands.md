# Command System Architecture Overview

The codebase uses a **fluent, type-safe command system** built on inheritance patterns with automatic permission generation and flexible argument handling.

---

## 1. Core Command Classes

Located in [server/core/command/system/](https://github.com/Savag3life/TaleServer/tree/main/hytale/main/src/com/hypixel/hytale/server/core/command/system/)

| Class | Purpose |
|-------|---------|
| `AbstractCommand` | Base class for all commands with argument registration and permission handling |
| `CommandManager` | Central registry for command registration and execution dispatch |
| `CommandContext` | Holds execution context including sender, parsed arguments, and input |
| `CommandSender` | Interface for entities that can send commands (extends `PermissionHolder`) |
| `ParseResult` | Tracks errors during command parsing |
| `ParserContext` | Categorizes input tokens into required/optional arguments |

---

## 2. Command Base Class Hierarchy

Located in [basecommands/](https://github.com/Savag3life/TaleServer/tree/main/hytale/main/src/com/hypixel/hytale/server/core/command/system/basecommands/)

```
AbstractCommand (base)
├── CommandBase (synchronous execution)
├── AbstractAsyncCommand (asynchronous execution)
│   ├── AbstractPlayerCommand (player-context commands)
│   ├── AbstractWorldCommand (world-context commands)
│   └── AbstractTargetPlayerCommand (target other players)
└── AbstractCommandCollection (subcommand containers/menus)
```

### CommandBase - Synchronous Commands

For commands that execute immediately and block until complete.

```java
public class MyCommand extends CommandBase {
    public MyCommand() {
        super("mycommand", "server.commands.mycommand.desc");
    }

    @Override
    protected void executeSync(@Nonnull CommandContext context) {
        // Your synchronous logic here
        context.sendMessage(Message.raw("Command executed!"));
    }
}
```

### AbstractAsyncCommand - Asynchronous Commands

For commands that return a `CompletableFuture<Void>` and don't block.

```java
public class MyAsyncCommand extends AbstractAsyncCommand {
    public MyAsyncCommand() {
        super("myasync", "server.commands.myasync.desc");
    }

    @Nonnull
    @Override
    protected CompletableFuture<Void> executeAsync(@Nonnull CommandContext context) {
        return CompletableFuture.runAsync(() -> {
            // Your async logic here
        });
    }
}
```

### AbstractPlayerCommand - Player-Context Commands

For commands that require a player to execute and need access to player/world context.

```java
public class MyPlayerCommand extends AbstractPlayerCommand {
    public MyPlayerCommand() {
        super("myplayercmd", "server.commands.myplayercmd.desc");
    }

    @Override
    protected void execute(
        @Nonnull CommandContext context,
        @Nonnull Store<EntityStore> store,
        @Nonnull Ref<EntityStore> ref,
        @Nonnull PlayerRef playerRef,
        @Nonnull World world
    ) {
        // Access to player store, reference, and world
        Player player = store.getComponent(ref, Player.getComponentType());
        // Your player-specific logic here
    }
}
```

### AbstractCommandCollection - Subcommand Containers

For commands that act as menus containing subcommands.

```java
public class PlayerCommand extends AbstractCommandCollection {
    public PlayerCommand() {
        super("player", "server.commands.player.desc");
        this.addSubCommand(new PlayerResetCommand());
        this.addSubCommand(new PlayerStatsSubCommand());
        this.addSubCommand(new PlayerEffectSubCommand());
    }
}
```

---

## 3. Command Registration

### System Commands (CommandManager)

System commands are registered directly with the `CommandManager`:

```java
// In CommandManager.registerCommands()
this.registerSystemCommand(new GameModeCommand());
this.registerSystemCommand(new KillCommand());
this.registerSystemCommand(new GiveCommand());
```

Permission format: `hytale.system.command.<commandname>`

### Plugin Commands (CommandRegistry)

Plugins register commands through their `CommandRegistry`:

```java
public class MyPlugin extends PluginBase {
    @Override
    protected void setup() {
        CommandRegistry commandRegistry = this.getCommandRegistry();
        commandRegistry.registerCommand(new MyCommand());
        commandRegistry.registerCommand(new AnotherCommand());
    }
}
```

Permission format: `<plugin.basePermission>.command.<commandname>`

### Registration Flow

```
registerCommand(AbstractCommand)
  └── command.setOwner(plugin/manager)
      └── Auto-generate permission if not set
  └── CommandManager.register(command)
      └── Add to commandRegistration map
      └── command.completeRegistration()
          └── Validate arguments and variants
      └── Register aliases
  └── Return CommandRegistration (for unregistration)
```

---

## 4. Argument System

Located in [arguments/system/](https://github.com/Savag3life/TaleServer/tree/main/hytale/main/src/com/hypixel/hytale/server/core/command/system/arguments/system/)

### Argument Types Overview

| Type | Description | Usage Pattern |
|------|-------------|---------------|
| `RequiredArg<T>` | Must be provided by user | Positional: `/cmd <value>` |
| `DefaultArg<T>` | Optional with default value | `--name value` or uses default |
| `OptionalArg<T>` | Optional, null if not provided | `--name value` |
| `FlagArg` | Boolean flag (true if present) | `--flagname` |

### Defining Arguments

Arguments are defined as fields in your command class using fluent methods:

```java
public class GiveCommand extends AbstractPlayerCommand {
    // Required argument - must be provided
    private final RequiredArg<Item> itemArg =
        this.withRequiredArg("item", "server.commands.give.item.desc", ArgTypes.ITEM_ASSET);

    // Default argument - has fallback value
    private final DefaultArg<Integer> quantityArg =
        this.withDefaultArg("quantity", "server.commands.give.quantity.desc", ArgTypes.INTEGER, 1, "1");

    // Optional argument - null if not provided
    private final OptionalArg<String> metadataArg =
        this.withOptionalArg("metadata", "server.commands.give.metadata.desc", ArgTypes.STRING);

    // Flag argument - boolean presence check
    private final FlagArg verboseFlag =
        this.withFlagArg("verbose", "server.commands.give.verbose.desc");
}
```

### List Arguments

For arguments that accept multiple values:

```java
// Required list argument
private final RequiredArg<List<String>> namesArg =
    this.withListRequiredArg("names", "description", ArgTypes.STRING);

// Default list argument
private final DefaultArg<List<Integer>> idsArg =
    this.withListDefaultArg("ids", "description", ArgTypes.INTEGER, List.of(1, 2, 3), "1, 2, 3");

// Optional list argument
private final OptionalArg<List<UUID>> uuidsArg =
    this.withListOptionalArg("uuids", "description", ArgTypes.UUID);
```

### Accessing Argument Values

```java
@Override
protected void execute(CommandContext context, ...) {
    // Get required argument value
    Item item = this.itemArg.get(context);

    // Get default argument (returns default if not provided)
    Integer quantity = this.quantityArg.get(context);

    // Check if optional argument was provided
    if (this.metadataArg.provided(context)) {
        String metadata = this.metadataArg.get(context);
    }

    // Check flag presence
    boolean isVerbose = this.verboseFlag.get(context);
}
```

### Argument Dependencies

Optional arguments can have dependencies on other arguments:

```java
private final OptionalArg<Integer> radiusArg = this.withOptionalArg("radius", "desc", ArgTypes.INTEGER)
    .requiredIf(this.positionArg)           // Required when position is provided
    .requiredIfAbsent(this.targetArg)       // Required when target is NOT provided
    .availableOnlyIfAll(this.enabledFlag)   // Only available when enabled flag is set
    .availableOnlyIfAllAbsent(this.simpleFlag); // Only available when simple flag is NOT set
```

### Argument Aliases

Optional arguments can have aliases:

```java
private final OptionalArg<Integer> countArg = this.withOptionalArg("count", "desc", ArgTypes.INTEGER)
    .addAlias("c")
    .addAlias("num");

// All of these work: --count 5, --c 5, --num 5
```

### Per-Argument Permissions

Optional arguments can require specific permissions:

```java
private final OptionalArg<Boolean> forceArg = this.withOptionalArg("force", "desc", ArgTypes.BOOLEAN);
// Set permission in constructor:
this.forceArg.setPermission("mycommand.force");
```

---

## 5. Built-in Argument Types

Located in [ArgTypes.java](https://github.com/Savag3life/TaleServer/tree/main/hytale/main/src/com/hypixel/hytale/server/core/command/system/arguments/types/ArgTypes.java)

### Primitive Types

| Type | Description | Examples |
|------|-------------|----------|
| `ArgTypes.BOOLEAN` | true/false values | `true`, `false` |
| `ArgTypes.INTEGER` | Whole numbers | `-27432`, `0`, `56346` |
| `ArgTypes.FLOAT` | Decimal numbers (single precision) | `3.14159`, `-2.5` |
| `ArgTypes.DOUBLE` | Decimal numbers (double precision) | `-3.14`, `3.141596` |
| `ArgTypes.STRING` | Text values | `"Hytale is cool!"` |
| `ArgTypes.UUID` | UUID identifiers | `550e8400-e29b-41d4-a716-446655440000` |

### Player/Entity Types

| Type | Description | Examples |
|------|-------------|----------|
| `ArgTypes.PLAYER_UUID` | Player by UUID or username | `john_doe`, `<uuid>` |
| `ArgTypes.PLAYER_REF` | Player reference | `john_doe`, `user123` |
| `ArgTypes.ENTITY_ID` | Entity by UUID | `<uuid>` |

### World/Position Types

| Type | Description | Examples |
|------|-------------|----------|
| `ArgTypes.WORLD` | World by name | `default` |
| `ArgTypes.RELATIVE_POSITION` | 3D double coordinates (relative) | `~ ~ ~`, `~5.5 ~ ~` |
| `ArgTypes.RELATIVE_BLOCK_POSITION` | 3D int coordinates (relative) | `~ ~ ~`, `~5 ~ ~-3` |
| `ArgTypes.RELATIVE_CHUNK_POSITION` | Chunk coordinates | `5 10`, `~c2 ~c-3` |
| `ArgTypes.VECTOR3I` | 3D integer vector | `124 232 234` |
| `ArgTypes.VECTOR2I` | 2D integer vector | `124 232` |
| `ArgTypes.ROTATION` | Pitch/yaw/roll | `124.63 232.27 234.22` |

### Asset Types

| Type | Description |
|------|-------------|
| `ArgTypes.ITEM_ASSET` | Item asset reference |
| `ArgTypes.BLOCK_TYPE_ASSET` | Block type asset reference |
| `ArgTypes.MODEL_ASSET` | Model asset reference |
| `ArgTypes.PARTICLE_SYSTEM` | Particle system asset |
| `ArgTypes.SOUND_EVENT_ASSET` | Sound event asset |
| `ArgTypes.EFFECT_ASSET` | Entity effect asset |
| `ArgTypes.WEATHER_ASSET` | Weather asset |
| `ArgTypes.ENVIRONMENT_ASSET` | Environment asset |

### Game Types

| Type | Description | Examples |
|------|-------------|----------|
| `ArgTypes.GAME_MODE` | Game mode enum | `SURVIVAL`, `CREATIVE`, `BUILDER` |
| `ArgTypes.SOUND_CATEGORY` | Sound category | `MASTER`, `MUSIC`, `AMBIENT` |
| `ArgTypes.COLOR` | Color value | `#FF0000`, `16711680`, `0xFF0000` |
| `ArgTypes.TICK_RATE` | Tick rate | `30tps`, `33ms`, `60` |

### Ranges and Patterns

| Type | Description | Examples |
|------|-------------|----------|
| `ArgTypes.INT_RANGE` | Min/max integer range | `-2 8`, `1 5` |
| `ArgTypes.BLOCK_PATTERN` | Block pattern with weights | `[20%Rock_Stone, 80%Rock_Shale]` |
| `ArgTypes.BLOCK_MASK` | Block mask filters | `[!Fluid_Water, !^Fluid_Lava]` |

### Creating Enum Argument Types

```java
// For any enum type
SingleArgumentType<MyEnum> MY_ENUM = ArgTypes.forEnum("my.enum.name", MyEnum.class);
```

---

## 6. Subcommands and Variants

### Subcommands

Add named subcommands to create command hierarchies:

```java
public class EntityCommand extends AbstractCommandCollection {
    public EntityCommand() {
        super("entity", "server.commands.entity.desc");
        this.addSubCommand(new EntitySpawnCommand());    // /entity spawn
        this.addSubCommand(new EntityKillCommand());     // /entity kill
        this.addSubCommand(new EntityListCommand());     // /entity list
    }
}
```

**Rules:**
- Subcommand must have a name
- Cannot reuse subcommand instances (each needs unique parent)
- Subcommand names and aliases must be unique within parent

### Usage Variants

Add alternate syntaxes with different argument counts:

```java
public class GameModeCommand extends AbstractPlayerCommand {
    private final RequiredArg<GameMode> gameModeArg =
        this.withRequiredArg("gamemode", "desc", ArgTypes.GAME_MODE);

    public GameModeCommand() {
        super("gamemode", "server.commands.gamemode.desc");
        this.addAliases("gm");
        this.requirePermission(HytalePermissions.fromCommand("gamemode.self"));

        // Add variant for targeting other players
        this.addUsageVariant(new GameModeOtherCommand());
    }

    // Self-targeting: /gamemode creative
    @Override
    protected void execute(...) {
        GameMode gameMode = this.gameModeArg.get(context);
        // Set sender's gamemode
    }

    // Variant for others: /gamemode creative PlayerName
    private static class GameModeOtherCommand extends CommandBase {
        private final RequiredArg<GameMode> gameModeArg =
            this.withRequiredArg("gamemode", "desc", ArgTypes.GAME_MODE);
        private final RequiredArg<PlayerRef> playerArg =
            this.withRequiredArg("player", "desc", ArgTypes.PLAYER_REF);

        GameModeOtherCommand() {
            super("server.commands.gamemode.other.desc");  // No name for variants!
            this.requirePermission(HytalePermissions.fromCommand("gamemode.other"));
        }

        @Override
        protected void executeSync(CommandContext context) {
            GameMode gameMode = this.gameModeArg.get(context);
            PlayerRef target = this.playerArg.get(context);
            // Set target's gamemode
        }
    }
}
```

**Rules:**
- Variants must NOT have a name (use description-only constructor)
- Variants are matched by number of required parameters
- Cannot have two variants with the same parameter count

---

## 7. Permissions

### Automatic Permission Generation

Permissions are automatically generated based on the command hierarchy:

```
System command: hytale.system.command.<name>
Plugin command: <plugin.basePermission>.command.<name>
Subcommand:     <parent.permission>.<subcommand.name>
```

### Manual Permission Assignment

```java
public MyCommand() {
    super("mycommand", "description");
    this.requirePermission("custom.permission.node");
}
```

### Permission Groups

Group commands by game mode or category:

```java
public MyCommand() {
    super("mycommand", "description");
    this.setPermissionGroup(GameMode.BUILDER);  // Only for builder mode
    // OR
    this.setPermissionGroups("admin", "moderator");  // Multiple groups
}
```

### Permission Checking Flow

```
hasPermission(sender)
  └── Check if sender has this command's permission
  └── If yes, recursively check parent command permissions
  └── Return true only if all permissions pass
```

---

## 8. Command Execution Flow

### Complete Execution Chain

```
1. User Input: "/gamemode creative PlayerName"

2. CommandManager.handleCommand()
   └── Parse command name from input ("gamemode")
   └── Find command in registry or aliases
   └── Call runCommand()

3. Tokenization
   └── Tokenizer.parseArguments() - Split into tokens
   └── ParserContext.of() - Categorize tokens

4. AbstractCommand.acceptCall()
   └── Check for subcommand/variant match
   └── Verify permissions
   └── Check for --help flag
   └── Check confirmation requirement (if any)
   └── Validate required parameter count

5. Argument Processing
   └── processRequiredArguments() - Parse positional args
   └── processOptionalArguments() - Parse --name value pairs
   └── Verify argument dependencies

6. Execution
   └── execute(CommandContext)
       ├── CommandBase: executeSync()
       ├── AbstractAsyncCommand: executeAsync()
       └── AbstractPlayerCommand: execute(context, store, ref, playerRef, world)
```

### Token Categories

| Category | Description | Example |
|----------|-------------|---------|
| Pre-optional single | Positional arguments | `creative PlayerName` |
| Pre-optional list | Multi-value between `{|` and `|}` | `{| value1 ; value2 |}` |
| Optional args | Key-value pairs | `--count 5 --verbose` |

---

## 9. Special Features

### Command Aliases

```java
public MyCommand() {
    super("mycommand", "description");
    this.addAliases("mc", "mycmd");  // /mc and /mycmd also work
}
```

### Confirmation Requirement

For dangerous commands that need explicit confirmation:

```java
public DangerousCommand() {
    super("dangerous", "description", true);  // requiresConfirmation = true
}
// User must run: /dangerous --confirm
```

### Singleplayer Unavailability

Mark commands as unavailable in singleplayer:

```java
public MyCommand() {
    super("mycommand", "description");
    this.setUnavailableInSingleplayer(true);
}
```

### Extra Arguments

Allow commands to accept more arguments than defined:

```java
public MyCommand() {
    super("mycommand", "description");
    this.setAllowsExtraArguments(true);
}
```

---

## 10. Complete Command Example

```java
package com.example.plugin.commands;

import com.hypixel.hytale.component.Ref;
import com.hypixel.hytale.component.Store;
import com.hypixel.hytale.server.core.Message;
import com.hypixel.hytale.server.core.command.system.CommandContext;
import com.hypixel.hytale.server.core.command.system.arguments.system.*;
import com.hypixel.hytale.server.core.command.system.arguments.types.ArgTypes;
import com.hypixel.hytale.server.core.command.system.basecommands.AbstractPlayerCommand;
import com.hypixel.hytale.server.core.command.system.basecommands.CommandBase;
import com.hypixel.hytale.server.core.entity.entities.Player;
import com.hypixel.hytale.server.core.universe.PlayerRef;
import com.hypixel.hytale.server.core.universe.world.World;
import com.hypixel.hytale.server.core.universe.world.storage.EntityStore;

public class HealCommand extends AbstractPlayerCommand {

    // Required argument
    private final RequiredArg<Float> amountArg =
        this.withRequiredArg("amount", "server.commands.heal.amount.desc", ArgTypes.FLOAT);

    // Optional flag
    private final FlagArg maxFlag =
        this.withFlagArg("max", "server.commands.heal.max.desc");

    public HealCommand() {
        super("heal", "server.commands.heal.desc");
        this.addAliases("hp", "health");
        this.requirePermission("myplugin.command.heal.self");

        // Add variant for healing other players
        this.addUsageVariant(new HealOtherCommand());
    }

    @Override
    protected void execute(
        @Nonnull CommandContext context,
        @Nonnull Store<EntityStore> store,
        @Nonnull Ref<EntityStore> ref,
        @Nonnull PlayerRef playerRef,
        @Nonnull World world
    ) {
        Player player = store.getComponent(ref, Player.getComponentType());
        float amount = this.amountArg.get(context);

        if (this.maxFlag.get(context)) {
            // Heal to max
            context.sendMessage(Message.translation("server.commands.heal.maxed"));
        } else {
            // Heal by amount
            context.sendMessage(
                Message.translation("server.commands.heal.success")
                    .param("amount", amount)
            );
        }
    }

    // Variant for healing other players
    private static class HealOtherCommand extends CommandBase {
        private final RequiredArg<PlayerRef> targetArg =
            this.withRequiredArg("target", "server.commands.heal.target.desc", ArgTypes.PLAYER_REF);
        private final RequiredArg<Float> amountArg =
            this.withRequiredArg("amount", "server.commands.heal.amount.desc", ArgTypes.FLOAT);

        HealOtherCommand() {
            super("server.commands.heal.other.desc");
            this.requirePermission("myplugin.command.heal.other");
        }

        @Override
        protected void executeSync(@Nonnull CommandContext context) {
            PlayerRef target = this.targetArg.get(context);
            float amount = this.amountArg.get(context);

            context.sendMessage(
                Message.translation("server.commands.heal.other.success")
                    .param("target", target.getUsername())
                    .param("amount", amount)
            );
        }
    }
}
```

### Registering in a Plugin

```java
public class MyPlugin extends PluginBase {
    @Override
    protected void setup() {
        this.getCommandRegistry().registerCommand(new HealCommand());
    }
}
```

---

## 11. Built-in Commands Reference

The server includes many built-in commands registered in [CommandManager](https://github.com/Savag3life/TaleServer/tree/main/hytale/main/src/com/hypixel/hytale/server/core/command/system/CommandManager.java):

### Player Commands
- `gamemode` / `gm` - Change game mode
- `kill` - Kill self or other player
- `damage` - Apply damage
- `give` - Give items
- `inventory` - Inventory management
- `player` - Player subcommands (stats, effects, camera, etc.)
- `hide` - Hide player
- `sudo` - Execute command as another player
- `whereami` - Show current location
- `whoami` - Show player info

### World Commands
- `chunk` - Chunk operations
- `entity` - Entity management
- `spawnblock` - Block spawning
- `worldgen` - World generation

### Server Commands
- `auth` - Authentication
- `kick` - Kick player
- `maxplayers` - Set max players
- `stop` - Stop server
- `who` - List online players

### Utility Commands
- `help` - Show help
- `backup` - Create backups
- `notify` - Send notifications
- `sound` - Play sounds
- `worldmap` - World map operations
- `lighting` - Lighting commands
- `sleep` - Sleep/pause
- `network` - Network commands
- `commands` - List commands

### Debug Commands
- `ping` - Check latency
- `version` - Show version
- `log` - Logging control
- `server` - Server stats (CPU, memory, GC)
- `assets` - Asset information
- `packs` - Pack management
- `stresstest` - Stress testing
- And many more...

---

## 12. Summary

| Feature | Implementation |
|---------|----------------|
| **Type Safety** | Generics throughout (`RequiredArg<T>`, `ArgumentType<T>`) |
| **Fluent API** | Chain methods for argument definition |
| **Auto Permissions** | Generated hierarchically from command structure |
| **Sync/Async** | Choose `CommandBase` or `AbstractAsyncCommand` |
| **Player Context** | `AbstractPlayerCommand` provides store/ref/world access |
| **Subcommands** | `addSubCommand()` for hierarchical commands |
| **Variants** | `addUsageVariant()` for alternate syntaxes |
| **Validation** | Arguments validate automatically during parsing |
| **Tab Completion** | Built into `ArgumentType.suggest()` |
| **Localization** | Description keys support i18n |
