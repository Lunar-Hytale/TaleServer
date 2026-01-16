# UI System

The UI System is the core framework in TaleServer for creating, managing, and interacting with in-game user interfaces. This includes full-screen pages, HUD elements, entity-attached UI components, and interactive forms. This document covers the complete API for working with custom UIs, from basic pages to complex interactive menus with event handling.

## Table of Contents

- [Overview](#overview)
- [Core Concepts](#core-concepts)
  - [Page Lifetimes](#page-lifetimes)
  - [UI Command Types](#ui-command-types)
  - [Event Binding Types](#event-binding-types)
- [Page System](#page-system)
  - [CustomUIPage](#customuipage)
  - [InteractiveCustomUIPage](#interactivecustomuipage)
  - [BasicCustomUIPage](#basiccustomuipage)
  - [PageManager](#pagemanager)
- [UI Builders](#ui-builders)
  - [UICommandBuilder](#uicommandbuilder)
  - [UIEventBuilder](#uieventbuilder)
  - [EventData](#eventdata)
- [UI Data Types](#ui-data-types)
  - [Value](#value)
  - [Area](#area)
  - [Anchor](#anchor)
  - [PatchStyle](#patchstyle)
  - [ItemGridSlot](#itemgridslot)
  - [LocalizableString](#localizablestring)
  - [DropdownEntryInfo](#dropdownentryinfo)
- [Entity UI Components](#entity-ui-components)
  - [EntityUIComponent](#entityuicomponent)
  - [EntityStatUIComponent](#entitystatuicomponent)
  - [CombatTextUIComponent](#combattextuicomponent)
  - [UIComponentList](#uicomponentlist)
- [UI Assets](#ui-assets)
  - [Asset Structure](#asset-structure)
  - [Selectors](#selectors)
- [Creating Custom UIs](#creating-custom-uis)
  - [Simple Page Example](#simple-page-example)
  - [Interactive Page Example](#interactive-page-example)
  - [Dynamic Updates](#dynamic-updates)
- [Full Example: Shop Menu](#full-example-shop-menu)
- [Best Practices](#best-practices)

---

## Overview

The UI System provides:

- **Custom Pages**: Full-screen modal interfaces for shops, dialogs, settings, and menus
- **HUD Elements**: Persistent on-screen displays for health, mana, and custom information
- **Entity UI**: Floating UI attached to entities (health bars, damage numbers, nameplates)
- **Event Handling**: Interactive forms with buttons, inputs, checkboxes, and dropdowns
- **Localization**: Built-in support for translated strings and message formatting
- **Real-time Updates**: Incremental UI updates without full page rebuilds

```
UI System Architecture
├── Page System
│   ├── CustomUIPage (base class for all custom pages)
│   ├── InteractiveCustomUIPage<T> (pages with event handling)
│   └── BasicCustomUIPage (simplified non-interactive pages)
│
├── UI Builders
│   ├── UICommandBuilder (manipulate UI elements)
│   ├── UIEventBuilder (bind interactive events)
│   └── EventData (event payload configuration)
│
├── Entity UI
│   ├── EntityUIComponent (base asset type)
│   ├── EntityStatUIComponent (stat displays)
│   └── CombatTextUIComponent (floating text)
│
└── Network Protocol
    ├── CustomPage (page packet)
    ├── CustomHud (HUD packet)
    └── CustomUICommand[] (UI manipulation commands)
```

---

## Core Concepts

### Page Lifetimes

[`CustomPageLifetime`](https://github.com/Savag3life/TaleServer/tree/main/hytale/main/src/com/hypixel/hytale/protocol/packets/interface_/CustomPageLifetime.java) controls when and how players can close a UI page.

| Lifetime | Value | Description |
|----------|-------|-------------|
| `CantClose` | 0 | Player cannot dismiss the page (must be closed programmatically) |
| `CanDismiss` | 1 | Player can dismiss with ESC key |
| `CanDismissOrCloseThroughInteraction` | 2 | Player can dismiss or close via UI button interaction |

```java
// Page that cannot be dismissed by the player
new MyPage(playerRef, CustomPageLifetime.CantClose);

// Page that can be closed with ESC
new MyPage(playerRef, CustomPageLifetime.CanDismiss);

// Page with close button or ESC
new MyPage(playerRef, CustomPageLifetime.CanDismissOrCloseThroughInteraction);
```

### UI Command Types

[`CustomUICommandType`](https://github.com/Savag3life/TaleServer/tree/main/hytale/main/src/com/hypixel/hytale/protocol/packets/interface_/CustomUICommandType.java) defines operations for manipulating UI elements.

| Command Type | Description |
|--------------|-------------|
| `Append` | Add element from file to selector (or root) |
| `AppendInline` | Add element from inline JSON to selector |
| `InsertBefore` | Insert element from file before selector |
| `InsertBeforeInline` | Insert element from inline JSON before selector |
| `Remove` | Remove element matching selector |
| `Set` | Set property value on selector |
| `Clear` | Clear all children from selector |

### Event Binding Types

[`CustomUIEventBindingType`](https://github.com/Savag3life/TaleServer/tree/main/hytale/main/src/com/hypixel/hytale/protocol/packets/interface_/CustomUIEventBindingType.java) defines interactive event types (24 total).

**Activation Events:**

| Event Type | Description |
|------------|-------------|
| `Activating` | Primary activation (left click, enter key) |
| `RightClicking` | Right mouse button click |
| `DoubleClicking` | Double left click |

**Value Events:**

| Event Type | Description |
|------------|-------------|
| `ValueChanged` | Input value changed (text fields, checkboxes, sliders) |
| `ElementReordered` | Element position changed in list |
| `Validating` | Input validation triggered |

**Focus Events:**

| Event Type | Description |
|------------|-------------|
| `FocusGained` | Element received focus |
| `FocusLost` | Element lost focus |

**Mouse Events:**

| Event Type | Description |
|------------|-------------|
| `MouseEntered` | Mouse cursor entered element |
| `MouseExited` | Mouse cursor left element |
| `MouseButtonReleased` | Mouse button released |

**Slot Events (for grids/inventories):**

| Event Type | Description |
|------------|-------------|
| `SlotClicking` | Slot left clicked |
| `SlotDoubleClicking` | Slot double clicked |
| `SlotMouseEntered` | Mouse entered slot |
| `SlotMouseExited` | Mouse exited slot |
| `SlotMouseDragCompleted` | Drag operation completed on slot |
| `SlotMouseDragExited` | Drag exited slot |
| `SlotClickReleaseWhileDragging` | Click released while dragging |
| `SlotClickPressWhileDragging` | Click pressed while dragging |

**Other Events:**

| Event Type | Description |
|------------|-------------|
| `KeyDown` | Key pressed while element focused |
| `DragCancelled` | Drag operation cancelled |
| `Dropped` | Item dropped on element |
| `Dismissing` | Page being dismissed |
| `SelectedTabChanged` | Tab selection changed |

---

## Page System

### CustomUIPage

[`CustomUIPage`](https://github.com/Savag3life/TaleServer/tree/main/hytale/main/src/com/hypixel/hytale/server/core/entity/entities/player/pages/CustomUIPage.java) is the abstract base class for all custom UI pages.

**Constructor:**

```java
public CustomUIPage(@Nonnull PlayerRef playerRef, @Nonnull CustomPageLifetime lifetime)
```

**Abstract Methods:**

| Method | Description |
|--------|-------------|
| `build(ref, commandBuilder, eventBuilder, store)` | Build the initial UI structure |

**Protected Methods:**

| Method | Description |
|--------|-------------|
| `rebuild()` | Completely rebuild the page (clears and rebuilds) |
| `sendUpdate()` | Send empty update (triggers acknowledge) |
| `sendUpdate(commandBuilder)` | Send incremental UI update |
| `sendUpdate(commandBuilder, clear)` | Send update with optional clear |
| `close()` | Close the page and return to game |

**Lifecycle Methods:**

| Method | Description |
|--------|-------------|
| `onDismiss(ref, store)` | Called when page is dismissed by player |
| `setLifetime(lifetime)` | Change the page lifetime |
| `getLifetime()` | Get current lifetime |

### InteractiveCustomUIPage

[`InteractiveCustomUIPage<T>`](https://github.com/Savag3life/TaleServer/tree/main/hytale/main/src/com/hypixel/hytale/server/core/entity/entities/player/pages/InteractiveCustomUIPage.java) extends `CustomUIPage` with typed event handling.

**Constructor:**

```java
public InteractiveCustomUIPage(
    @Nonnull PlayerRef playerRef,
    @Nonnull CustomPageLifetime lifetime,
    @Nonnull BuilderCodec<T> eventDataCodec
)
```

**Event Handling:**

```java
// Override to handle events with your typed data class
@Override
public void handleDataEvent(
    @Nonnull Ref<EntityStore> ref,
    @Nonnull Store<EntityStore> store,
    @Nonnull T data
) {
    // Process the event data
}
```

**Update Methods:**

```java
// Send update with both commands and new event bindings
protected void sendUpdate(
    @Nullable UICommandBuilder commandBuilder,
    @Nullable UIEventBuilder eventBuilder,
    boolean clear
)
```

### BasicCustomUIPage

`BasicCustomUIPage` is a simplified abstract class for pages that don't need event handling.

```java
public abstract class BasicCustomUIPage extends CustomUIPage {
    public BasicCustomUIPage(PlayerRef playerRef, CustomPageLifetime lifetime) {
        super(playerRef, lifetime);
    }

    // Simplified build method - no event builder
    public abstract void build(UICommandBuilder commandBuilder);
}
```

### PageManager

[`PageManager`](https://github.com/Savag3life/TaleServer/tree/main/hytale/main/src/com/hypixel/hytale/server/core/entity/entities/player/pages/PageManager.java) controls all page operations for a player.

**Key Methods:**

| Method | Description |
|--------|-------------|
| `openCustomPage(ref, store, page)` | Open a custom UI page |
| `updateCustomPage(customPage)` | Send incremental page update |
| `setPage(ref, store, page)` | Switch to a standard page (or `Page.None`) |
| `openCustomPageWithWindows(page, windows...)` | Open page with window overlays |
| `handleEvent(ref, store, event)` | Process incoming UI event |

**Usage:**

```java
// Get the page manager from a player
Player player = store.getComponent(ref, Player.getComponentType());
PageManager pageManager = player.getPageManager();

// Open a custom page
MyCustomPage page = new MyCustomPage(playerRef);
pageManager.openCustomPage(ref, store, page);

// Close to game view
pageManager.setPage(ref, store, Page.None);
```

---

## UI Builders

### UICommandBuilder

[`UICommandBuilder`](https://github.com/Savag3life/TaleServer/tree/main/hytale/main/src/com/hypixel/hytale/server/core/ui/builder/UICommandBuilder.java) provides a fluent API for manipulating UI elements.

**Append Operations:**

```java
UICommandBuilder builder = new UICommandBuilder();

// Append UI document to root
builder.append("Pages/MyPage.ui");

// Append to specific container
builder.append("#Container", "Pages/ListItem.ui");

// Append inline JSON
builder.appendInline("#Container", "{\"Type\":\"Label\",\"Text\":\"Hello\"}");
```

**Insert Operations:**

```java
// Insert before an element
builder.insertBefore("#ExistingElement", "Pages/NewElement.ui");

// Insert inline before element
builder.insertBeforeInline("#ExistingElement", "{\"Type\":\"Label\"}");
```

**Removal Operations:**

```java
// Remove an element
builder.remove("#ElementToRemove");

// Clear all children from container
builder.clear("#Container");
```

**Set Operations:**

```java
// Set text
builder.set("#Title.Text", "Hello World");

// Set boolean
builder.set("#CheckBox.Value", true);

// Set number
builder.set("#ProgressBar.Value", 0.75f);
builder.set("#Counter.Text", 42);

// Set visibility
builder.set("#Panel.Visible", false);

// Set Message (with formatting/localization)
builder.set("#Label.TextSpans", Message.translation("my.translation.key"));
builder.set("#Label.TextSpans", Message.translation("my.key").param("count", 5));

// Set null
builder.setNull("#OptionalField.Value");
```

**Complex Types:**

```java
// Set Area
builder.setObject("#Element.Area", new Area(10, 20, 100, 50));

// Set ItemStack
builder.setObject("#Slot.Item", new ItemStack("Iron_Sword"));

// Set ItemGridSlot
builder.setObject("#Slot.Slot", itemGridSlot);

// Set arrays
builder.set("#Dropdown.Entries", dropdownEntries);

// Set Value reference
builder.set("#Element.Style", Value.ref("Pages/Styles.ui", "HighlightedStyle"));
```

**Supported Object Types for `setObject()`:**

| Type | Description |
|------|-------------|
| `Area` | 2D rectangular region |
| `ItemGridSlot` | Inventory slot styling |
| `ItemStack` | Item with quantity/metadata |
| `LocalizableString` | String or translation key |
| `PatchStyle` | Textured borders/backgrounds |
| `DropdownEntryInfo` | Dropdown menu entry |
| `Anchor` | Layout positioning |

### UIEventBuilder

[`UIEventBuilder`](https://github.com/Savag3life/TaleServer/tree/main/hytale/main/src/com/hypixel/hytale/server/core/ui/builder/UIEventBuilder.java) binds interactive events to UI elements.

**Basic Binding:**

```java
UIEventBuilder eventBuilder = new UIEventBuilder();

// Simple button click
eventBuilder.addEventBinding(CustomUIEventBindingType.Activating, "#MyButton");

// With custom data
eventBuilder.addEventBinding(
    CustomUIEventBindingType.Activating,
    "#SaveButton",
    new EventData().append("Action", "Save")
);
```

**With Interface Lock:**

```java
// Lock interface during event processing (default: true)
eventBuilder.addEventBinding(
    CustomUIEventBindingType.Activating,
    "#Button",
    new EventData().append("Action", "Process"),
    true  // locksInterface
);

// Non-blocking event (interface stays responsive)
eventBuilder.addEventBinding(
    CustomUIEventBindingType.ValueChanged,
    "#SearchInput",
    new EventData().append("Action", "Search"),
    false  // don't lock interface
);
```

**Data References:**

Use `@` prefix to reference UI element values in event data:

```java
eventBuilder.addEventBinding(
    CustomUIEventBindingType.Activating,
    "#SubmitButton",
    new EventData()
        .append("Action", "Submit")
        .append("@Username", "#UsernameInput.Value")
        .append("@Password", "#PasswordInput.Value")
        .append("@RememberMe", "#RememberCheckbox.Value")
);
```

### EventData

[`EventData`](https://github.com/Savag3life/TaleServer/tree/main/hytale/main/src/com/hypixel/hytale/server/core/ui/builder/EventData.java) configures event payload data.

```java
// Create event data
EventData data = new EventData();

// Append static values
data.append("Action", "Purchase");
data.append("ItemId", "Iron_Sword");

// Append UI value references (@ prefix)
data.append("@Quantity", "#QuantitySlider.Value");
data.append("@SelectedSlot", "#ItemGrid.SelectedIndex");

// Static helper
EventData data = EventData.of("Action", "Click");
```

---

## UI Data Types

### Value

[`Value<T>`](https://github.com/Savag3life/TaleServer/tree/main/hytale/main/src/com/hypixel/hytale/server/core/ui/Value.java) allows properties to reference either direct values or document references.

```java
// Direct value
Value<String> directValue = Value.of("Hello");
Value<Integer> intValue = Value.of(42);

// Document reference
Value<String> styleRef = Value.ref("Pages/Styles.ui", "HighlightedStyle");
Value<String> colorRef = Value.ref("Pages/Colors.ui", "PrimaryColor");
```

**Usage in UICommandBuilder:**

```java
// Reference a style from another document
private static final Value<String> BUTTON_SELECTED =
    Value.ref("Pages/BasicTextButton.ui", "SelectedLabelStyle");

// Apply the reference
commandBuilder.set("#Button.Style", BUTTON_SELECTED);
```

### Area

[`Area`](https://github.com/Savag3life/TaleServer/tree/main/hytale/main/src/com/hypixel/hytale/server/core/ui/Area.java) defines a 2D rectangular region.

```java
// Create area: X, Y, Width, Height
Area area = new Area(10, 20, 100, 50);

// Use in command builder
commandBuilder.setObject("#Panel.Area", area);
```

### Anchor

[`Anchor`](https://github.com/Savag3life/TaleServer/tree/main/hytale/main/src/com/hypixel/hytale/server/core/ui/Anchor.java) defines layout positioning.

```java
// Create anchor with positioning values
Anchor anchor = new Anchor(left, right, top, bottom, width, height);

// Use in command builder
commandBuilder.setObject("#Element.Anchor", anchor);
```

### PatchStyle

[`PatchStyle`](https://github.com/Savag3life/TaleServer/tree/main/hytale/main/src/com/hypixel/hytale/server/core/ui/PatchStyle.java) defines textured borders and backgrounds (9-slice).

### ItemGridSlot

[`ItemGridSlot`](https://github.com/Savag3life/TaleServer/tree/main/hytale/main/src/com/hypixel/hytale/server/core/ui/ItemGridSlot.java) defines inventory grid cell styling.

### LocalizableString

[`LocalizableString`](https://github.com/Savag3life/TaleServer/tree/main/hytale/main/src/com/hypixel/hytale/server/core/ui/LocalizableString.java) wraps a string or localization key with optional parameters.

### DropdownEntryInfo

[`DropdownEntryInfo`](https://github.com/Savag3life/TaleServer/tree/main/hytale/main/src/com/hypixel/hytale/server/core/ui/DropdownEntryInfo.java) defines a dropdown menu entry with label and value.

```java
// Create dropdown entries
DropdownEntryInfo[] entries = new DropdownEntryInfo[] {
    new DropdownEntryInfo("Option 1", "value1"),
    new DropdownEntryInfo("Option 2", "value2"),
    new DropdownEntryInfo("Option 3", "value3")
};

// Set on dropdown
commandBuilder.set("#Dropdown.Entries", entries);
```

---

## Entity UI Components

### EntityUIComponent

[`EntityUIComponent`](https://github.com/Savag3life/TaleServer/tree/main/hytale/main/src/com/hypixel/hytale/server/core/modules/entityui/asset/EntityUIComponent.java) is the base asset type for entity-attached UI elements.

**Properties:**

| Property | Type | Description |
|----------|------|-------------|
| `Id` | `String` | Unique identifier |
| `HitboxOffset` | `Vector2f` | Offset from entity center |

### EntityStatUIComponent

[`EntityStatUIComponent`](https://github.com/Savag3life/TaleServer/tree/main/hytale/main/src/com/hypixel/hytale/server/core/modules/entityui/asset/EntityStatUIComponent.java) displays entity stats (health bars, mana, etc.).

**JSON Definition:**

```json
{
    "Id": "health_bar",
    "Type": "EntityStat",
    "EntityStat": "health",
    "HitboxOffset": { "X": 0.0, "Y": 2.5 }
}
```

### CombatTextUIComponent

[`CombatTextUIComponent`](https://github.com/Savag3life/TaleServer/tree/main/hytale/main/src/com/hypixel/hytale/server/core/modules/entityui/asset/CombatTextUIComponent.java) displays floating damage/healing numbers.

**Properties:**

| Property | Type | Description |
|----------|------|-------------|
| `RandomPositionOffsetRange` | `Range` | Random position variation |
| `ViewportMargin` | `float` | Distance from screen edges |
| `Duration` | `float` | Display time in seconds |
| `FontSize` | `float` | Text size |
| `TextColor` | `Color` | Text color |
| `AnimationEvents` | `AnimationEvent[]` | Scale, position, opacity animations |

**JSON Definition:**

```json
{
    "Id": "damage_text",
    "Type": "CombatText",
    "HitboxOffset": { "X": 0.0, "Y": 2.0 },
    "Duration": 1.5,
    "FontSize": 24,
    "TextColor": "#FF0000",
    "RandomPositionOffsetRange": { "Min": -0.5, "Max": 0.5 },
    "AnimationEvents": [
        {
            "Type": "Scale",
            "StartTime": 0.0,
            "EndTime": 0.2,
            "StartValue": 0.5,
            "EndValue": 1.2
        },
        {
            "Type": "Position",
            "StartTime": 0.0,
            "EndTime": 1.5,
            "StartY": 0.0,
            "EndY": 1.0
        },
        {
            "Type": "Opacity",
            "StartTime": 1.0,
            "EndTime": 1.5,
            "StartValue": 1.0,
            "EndValue": 0.0
        }
    ]
}
```

### UIComponentList

[`UIComponentList`](https://github.com/Savag3life/TaleServer/tree/main/hytale/main/src/com/hypixel/hytale/server/core/modules/entityui/UIComponentList.java) is an ECS component that lists UI components attached to an entity.

```java
// Get UI components for an entity
UIComponentList uiList = store.getComponent(entityRef, UIComponentList.getComponentType());
if (uiList != null) {
    int[] componentIds = uiList.getComponentIds();
    // Process component IDs
}
```

---

## UI Assets

### Asset Structure

UI documents are stored as `.ui` files in the `Pages/` directory. They define the structure, styling, and layout of UI elements.

**Common Elements:**

- `Panel` - Container element
- `Label` - Text display
- `Button` - Clickable button
- `TextInput` - Text input field
- `CheckBox` - Boolean toggle
- `Slider` - Value slider
- `Dropdown` - Selection dropdown
- `ItemGrid` - Inventory grid
- `Image` - Image display

### Selectors

Selectors identify UI elements for commands and events:

```java
// ID selector
"#MyElement"

// Nested element
"#ParentPanel #ChildElement"

// Property access
"#Element.Text"
"#Element.Visible"
"#Element.Value"

// Array index
"#List[0]"
"#List[5].Text"

// Combined
"#MainPanel #ItemList[3] #Label.Text"
```

---

## Creating Custom UIs

### Simple Page Example

A basic page that displays information without event handling:

```java
package com.example.ui;

import com.hypixel.hytale.component.Ref;
import com.hypixel.hytale.component.Store;
import com.hypixel.hytale.protocol.packets.interface_.CustomPageLifetime;
import com.hypixel.hytale.server.core.Message;
import com.hypixel.hytale.server.core.entity.entities.player.pages.CustomUIPage;
import com.hypixel.hytale.server.core.ui.builder.UICommandBuilder;
import com.hypixel.hytale.server.core.ui.builder.UIEventBuilder;
import com.hypixel.hytale.server.core.universe.PlayerRef;
import com.hypixel.hytale.server.core.universe.world.storage.EntityStore;
import javax.annotation.Nonnull;

public class WelcomePage extends CustomUIPage {
    private final String playerName;
    private final int playerLevel;

    public WelcomePage(@Nonnull PlayerRef playerRef, String playerName, int playerLevel) {
        super(playerRef, CustomPageLifetime.CanDismiss);
        this.playerName = playerName;
        this.playerLevel = playerLevel;
    }

    @Override
    public void build(
        @Nonnull Ref<EntityStore> ref,
        @Nonnull UICommandBuilder commandBuilder,
        @Nonnull UIEventBuilder eventBuilder,
        @Nonnull Store<EntityStore> store
    ) {
        // Load UI document
        commandBuilder.append("Pages/WelcomePage.ui");

        // Set title
        commandBuilder.set("#Title.TextSpans",
            Message.translation("welcome.title").param("name", playerName));

        // Set player info
        commandBuilder.set("#LevelLabel.Text", "Level: " + playerLevel);

        // Show/hide elements based on level
        commandBuilder.set("#VIPBadge.Visible", playerLevel >= 50);
    }
}
```

**Opening the page:**

```java
// Get player components
Player player = store.getComponent(playerRef.getReference(), Player.getComponentType());

// Create and open page
WelcomePage page = new WelcomePage(playerRef, "Steve", 25);
player.getPageManager().openCustomPage(playerRef.getReference(), store, page);
```

### Interactive Page Example

A page with buttons and input handling:

```java
package com.example.ui;

import com.hypixel.hytale.codec.Codec;
import com.hypixel.hytale.codec.KeyedCodec;
import com.hypixel.hytale.codec.builder.BuilderCodec;
import com.hypixel.hytale.codec.codecs.EnumCodec;
import com.hypixel.hytale.component.Ref;
import com.hypixel.hytale.component.Store;
import com.hypixel.hytale.protocol.packets.interface_.CustomPageLifetime;
import com.hypixel.hytale.protocol.packets.interface_.CustomUIEventBindingType;
import com.hypixel.hytale.protocol.packets.interface_.Page;
import com.hypixel.hytale.server.core.Message;
import com.hypixel.hytale.server.core.entity.entities.Player;
import com.hypixel.hytale.server.core.entity.entities.player.pages.InteractiveCustomUIPage;
import com.hypixel.hytale.server.core.ui.builder.EventData;
import com.hypixel.hytale.server.core.ui.builder.UICommandBuilder;
import com.hypixel.hytale.server.core.ui.builder.UIEventBuilder;
import com.hypixel.hytale.server.core.universe.PlayerRef;
import com.hypixel.hytale.server.core.universe.world.storage.EntityStore;
import javax.annotation.Nonnull;

public class SettingsPage extends InteractiveCustomUIPage<SettingsPage.PageData> {
    private boolean musicEnabled = true;
    private boolean sfxEnabled = true;
    private float volume = 0.8f;

    public SettingsPage(@Nonnull PlayerRef playerRef) {
        super(playerRef, CustomPageLifetime.CanDismissOrCloseThroughInteraction, PageData.CODEC);
    }

    @Override
    public void build(
        @Nonnull Ref<EntityStore> ref,
        @Nonnull UICommandBuilder commandBuilder,
        @Nonnull UIEventBuilder eventBuilder,
        @Nonnull Store<EntityStore> store
    ) {
        // Load base UI
        commandBuilder.append("Pages/SettingsPage.ui");

        // Set initial values
        commandBuilder.set("#MusicCheckbox.Value", musicEnabled);
        commandBuilder.set("#SfxCheckbox.Value", sfxEnabled);
        commandBuilder.set("#VolumeSlider.Value", volume);

        // Bind events
        eventBuilder.addEventBinding(
            CustomUIEventBindingType.ValueChanged,
            "#MusicCheckbox",
            new EventData()
                .append("Action", Action.ToggleMusic.name())
                .append("@Enabled", "#MusicCheckbox.Value"),
            false  // Don't lock interface for checkbox
        );

        eventBuilder.addEventBinding(
            CustomUIEventBindingType.ValueChanged,
            "#SfxCheckbox",
            new EventData()
                .append("Action", Action.ToggleSfx.name())
                .append("@Enabled", "#SfxCheckbox.Value"),
            false
        );

        eventBuilder.addEventBinding(
            CustomUIEventBindingType.ValueChanged,
            "#VolumeSlider",
            new EventData()
                .append("Action", Action.SetVolume.name())
                .append("@Volume", "#VolumeSlider.Value"),
            false
        );

        eventBuilder.addEventBinding(
            CustomUIEventBindingType.Activating,
            "#SaveButton",
            new EventData().append("Action", Action.Save.name())
        );

        eventBuilder.addEventBinding(
            CustomUIEventBindingType.Activating,
            "#CancelButton",
            new EventData().append("Action", Action.Cancel.name())
        );
    }

    @Override
    public void handleDataEvent(
        @Nonnull Ref<EntityStore> ref,
        @Nonnull Store<EntityStore> store,
        @Nonnull PageData data
    ) {
        Player player = store.getComponent(ref, Player.getComponentType());

        switch (data.action) {
            case ToggleMusic:
                musicEnabled = data.enabled;
                break;

            case ToggleSfx:
                sfxEnabled = data.enabled;
                break;

            case SetVolume:
                volume = data.volume;
                // Update volume label
                UICommandBuilder builder = new UICommandBuilder();
                builder.set("#VolumeLabel.Text", String.format("%.0f%%", volume * 100));
                sendUpdate(builder);
                break;

            case Save:
                // Save settings
                saveSettings();
                playerRef.sendMessage(Message.translation("settings.saved"));
                player.getPageManager().setPage(ref, store, Page.None);
                break;

            case Cancel:
                player.getPageManager().setPage(ref, store, Page.None);
                break;
        }
    }

    private void saveSettings() {
        // Save to player data
    }

    // Action enum
    public enum Action {
        ToggleMusic,
        ToggleSfx,
        SetVolume,
        Save,
        Cancel
    }

    // Event data class
    public static class PageData {
        public static final BuilderCodec<PageData> CODEC = BuilderCodec.builder(PageData.class, PageData::new)
            .append(
                new KeyedCodec<>("Action", new EnumCodec<>(Action.class, EnumCodec.EnumStyle.LEGACY)),
                (o, v) -> o.action = v,
                o -> o.action
            ).add()
            .append(
                new KeyedCodec<>("@Enabled", Codec.BOOLEAN),
                (o, v) -> o.enabled = v,
                o -> o.enabled
            ).add()
            .append(
                new KeyedCodec<>("@Volume", Codec.FLOAT),
                (o, v) -> o.volume = v,
                o -> o.volume
            ).add()
            .build();

        public Action action;
        public boolean enabled;
        public float volume;
    }
}
```

### Dynamic Updates

Update UI without rebuilding the entire page:

```java
// In your page class
private void updateProgressBar(float progress) {
    UICommandBuilder builder = new UICommandBuilder();
    builder.set("#ProgressBar.Value", progress);
    builder.set("#ProgressLabel.Text", String.format("%.0f%%", progress * 100));
    sendUpdate(builder);
}

// Add items to a list dynamically
private void addListItem(String itemText, int index) {
    UICommandBuilder builder = new UICommandBuilder();
    UIEventBuilder eventBuilder = new UIEventBuilder();

    // Append new item
    builder.append("#ItemList", "Pages/ListItem.ui");
    builder.set("#ItemList[" + index + "] #Label.Text", itemText);

    // Bind click event
    eventBuilder.addEventBinding(
        CustomUIEventBindingType.Activating,
        "#ItemList[" + index + "]",
        new EventData()
            .append("Action", "SelectItem")
            .append("Index", String.valueOf(index))
    );

    sendUpdate(builder, eventBuilder, false);
}

// Clear and rebuild a section
private void refreshList() {
    UICommandBuilder builder = new UICommandBuilder();
    UIEventBuilder eventBuilder = new UIEventBuilder();

    // Clear existing items
    builder.clear("#ItemList");

    // Add new items
    for (int i = 0; i < items.size(); i++) {
        builder.append("#ItemList", "Pages/ListItem.ui");
        builder.set("#ItemList[" + i + "] #Label.Text", items.get(i).getName());

        eventBuilder.addEventBinding(
            CustomUIEventBindingType.Activating,
            "#ItemList[" + i + "]",
            new EventData()
                .append("Action", "SelectItem")
                .append("Index", String.valueOf(i))
        );
    }

    sendUpdate(builder, eventBuilder, false);
}
```

---

## Full Example: Shop Menu

A complete implementation of an interactive shop menu with categories, items, and purchasing:

```java
package com.example.shop;

import com.hypixel.hytale.codec.Codec;
import com.hypixel.hytale.codec.KeyedCodec;
import com.hypixel.hytale.codec.builder.BuilderCodec;
import com.hypixel.hytale.codec.codecs.EnumCodec;
import com.hypixel.hytale.component.Ref;
import com.hypixel.hytale.component.Store;
import com.hypixel.hytale.protocol.packets.interface_.CustomPageLifetime;
import com.hypixel.hytale.protocol.packets.interface_.CustomUIEventBindingType;
import com.hypixel.hytale.protocol.packets.interface_.Page;
import com.hypixel.hytale.server.core.Message;
import com.hypixel.hytale.server.core.entity.entities.Player;
import com.hypixel.hytale.server.core.entity.entities.player.pages.InteractiveCustomUIPage;
import com.hypixel.hytale.server.core.inventory.ItemStack;
import com.hypixel.hytale.server.core.inventory.container.ItemContainer;
import com.hypixel.hytale.server.core.ui.Value;
import com.hypixel.hytale.server.core.ui.builder.EventData;
import com.hypixel.hytale.server.core.ui.builder.UICommandBuilder;
import com.hypixel.hytale.server.core.ui.builder.UIEventBuilder;
import com.hypixel.hytale.server.core.universe.PlayerRef;
import com.hypixel.hytale.server.core.universe.world.storage.EntityStore;
import javax.annotation.Nonnull;
import java.util.List;
import java.util.ArrayList;

public class ShopPage extends InteractiveCustomUIPage<ShopPage.PageData> {

    // Style references
    private static final Value<String> TAB_NORMAL = Value.ref("Pages/ShopStyles.ui", "TabNormal");
    private static final Value<String> TAB_SELECTED = Value.ref("Pages/ShopStyles.ui", "TabSelected");

    // Shop data
    private final List<ShopCategory> categories;
    private int selectedCategoryIndex = 0;
    private int selectedItemIndex = -1;
    private int purchaseQuantity = 1;

    public ShopPage(@Nonnull PlayerRef playerRef, List<ShopCategory> categories) {
        super(playerRef, CustomPageLifetime.CanDismissOrCloseThroughInteraction, PageData.CODEC);
        this.categories = categories;
    }

    @Override
    public void build(
        @Nonnull Ref<EntityStore> ref,
        @Nonnull UICommandBuilder commandBuilder,
        @Nonnull UIEventBuilder eventBuilder,
        @Nonnull Store<EntityStore> store
    ) {
        // Load base UI
        commandBuilder.append("Pages/ShopPage.ui");

        // Set shop title
        commandBuilder.set("#ShopTitle.TextSpans",
            Message.translation("shop.title").color("#FFD700"));

        // Get player's gold balance
        Player player = store.getComponent(ref, Player.getComponentType());
        int gold = getPlayerGold(player);
        commandBuilder.set("#GoldDisplay.Text", formatGold(gold));

        // Build category tabs
        buildCategoryTabs(commandBuilder, eventBuilder);

        // Build item list for selected category
        buildItemList(commandBuilder, eventBuilder);

        // Setup purchase panel
        commandBuilder.set("#PurchasePanel.Visible", false);

        // Bind close button
        eventBuilder.addEventBinding(
            CustomUIEventBindingType.Activating,
            "#CloseButton",
            new EventData().append("Action", Action.Close.name())
        );
    }

    private void buildCategoryTabs(UICommandBuilder commandBuilder, UIEventBuilder eventBuilder) {
        commandBuilder.clear("#CategoryTabs");

        for (int i = 0; i < categories.size(); i++) {
            ShopCategory category = categories.get(i);

            // Add tab button
            commandBuilder.append("#CategoryTabs", "Pages/ShopCategoryTab.ui");
            commandBuilder.set("#CategoryTabs[" + i + "] #TabLabel.Text", category.getName());
            commandBuilder.set("#CategoryTabs[" + i + "] #TabIcon.Texture", category.getIcon());

            // Highlight selected tab
            if (i == selectedCategoryIndex) {
                commandBuilder.set("#CategoryTabs[" + i + "].Style", TAB_SELECTED);
            } else {
                commandBuilder.set("#CategoryTabs[" + i + "].Style", TAB_NORMAL);
            }

            // Bind tab click
            eventBuilder.addEventBinding(
                CustomUIEventBindingType.Activating,
                "#CategoryTabs[" + i + "]",
                new EventData()
                    .append("Action", Action.SelectCategory.name())
                    .append("Index", String.valueOf(i))
            );
        }
    }

    private void buildItemList(UICommandBuilder commandBuilder, UIEventBuilder eventBuilder) {
        commandBuilder.clear("#ItemList");

        if (selectedCategoryIndex < 0 || selectedCategoryIndex >= categories.size()) {
            return;
        }

        ShopCategory category = categories.get(selectedCategoryIndex);
        List<ShopItem> items = category.getItems();

        for (int i = 0; i < items.size(); i++) {
            ShopItem item = items.get(i);

            // Add item row
            commandBuilder.append("#ItemList", "Pages/ShopItemRow.ui");

            String selector = "#ItemList[" + i + "]";

            // Set item details
            commandBuilder.set(selector + " #ItemIcon.Item", new ItemStack(item.getItemId()));
            commandBuilder.set(selector + " #ItemName.TextSpans",
                Message.translation(item.getTranslationKey()));
            commandBuilder.set(selector + " #ItemPrice.Text", formatGold(item.getPrice()));

            // Show stock if limited
            if (item.hasLimitedStock()) {
                commandBuilder.set(selector + " #StockLabel.Visible", true);
                commandBuilder.set(selector + " #StockLabel.Text",
                    "Stock: " + item.getStock());
            } else {
                commandBuilder.set(selector + " #StockLabel.Visible", false);
            }

            // Highlight selected item
            if (i == selectedItemIndex) {
                commandBuilder.set(selector + " #SelectionHighlight.Visible", true);
            } else {
                commandBuilder.set(selector + " #SelectionHighlight.Visible", false);
            }

            // Bind item click
            eventBuilder.addEventBinding(
                CustomUIEventBindingType.Activating,
                selector,
                new EventData()
                    .append("Action", Action.SelectItem.name())
                    .append("Index", String.valueOf(i))
            );

            // Bind hover events
            eventBuilder.addEventBinding(
                CustomUIEventBindingType.MouseEntered,
                selector,
                new EventData()
                    .append("Action", Action.HoverItem.name())
                    .append("Index", String.valueOf(i)),
                false
            );

            eventBuilder.addEventBinding(
                CustomUIEventBindingType.MouseExited,
                selector,
                new EventData().append("Action", Action.UnhoverItem.name()),
                false
            );
        }
    }

    private void showPurchasePanel(ShopItem item) {
        UICommandBuilder builder = new UICommandBuilder();
        UIEventBuilder eventBuilder = new UIEventBuilder();

        builder.set("#PurchasePanel.Visible", true);
        builder.set("#PurchasePanel #ItemName.TextSpans",
            Message.translation(item.getTranslationKey()));
        builder.set("#PurchasePanel #ItemIcon.Item", new ItemStack(item.getItemId()));
        builder.set("#PurchasePanel #UnitPrice.Text", formatGold(item.getPrice()));
        builder.set("#PurchasePanel #QuantityInput.Value", purchaseQuantity);
        builder.set("#PurchasePanel #TotalPrice.Text",
            formatGold(item.getPrice() * purchaseQuantity));

        // Quantity controls
        eventBuilder.addEventBinding(
            CustomUIEventBindingType.Activating,
            "#PurchasePanel #DecreaseBtn",
            new EventData().append("Action", Action.DecreaseQuantity.name())
        );

        eventBuilder.addEventBinding(
            CustomUIEventBindingType.Activating,
            "#PurchasePanel #IncreaseBtn",
            new EventData().append("Action", Action.IncreaseQuantity.name())
        );

        eventBuilder.addEventBinding(
            CustomUIEventBindingType.ValueChanged,
            "#PurchasePanel #QuantityInput",
            new EventData()
                .append("Action", Action.SetQuantity.name())
                .append("@Quantity", "#PurchasePanel #QuantityInput.Value"),
            false
        );

        // Purchase and cancel buttons
        eventBuilder.addEventBinding(
            CustomUIEventBindingType.Activating,
            "#PurchasePanel #ConfirmButton",
            new EventData().append("Action", Action.ConfirmPurchase.name())
        );

        eventBuilder.addEventBinding(
            CustomUIEventBindingType.Activating,
            "#PurchasePanel #CancelPurchaseBtn",
            new EventData().append("Action", Action.CancelPurchase.name())
        );

        sendUpdate(builder, eventBuilder, false);
    }

    private void hidePurchasePanel() {
        UICommandBuilder builder = new UICommandBuilder();
        builder.set("#PurchasePanel.Visible", false);
        sendUpdate(builder);
    }

    private void updateTotalPrice(ShopItem item) {
        UICommandBuilder builder = new UICommandBuilder();
        builder.set("#PurchasePanel #TotalPrice.Text",
            formatGold(item.getPrice() * purchaseQuantity));
        builder.set("#PurchasePanel #QuantityInput.Value", purchaseQuantity);
        sendUpdate(builder);
    }

    @Override
    public void handleDataEvent(
        @Nonnull Ref<EntityStore> ref,
        @Nonnull Store<EntityStore> store,
        @Nonnull PageData data
    ) {
        Player player = store.getComponent(ref, Player.getComponentType());

        switch (data.action) {
            case SelectCategory: {
                int index = parseIntSafe(data.index, 0);
                if (index >= 0 && index < categories.size() && index != selectedCategoryIndex) {
                    selectedCategoryIndex = index;
                    selectedItemIndex = -1;
                    purchaseQuantity = 1;
                    hidePurchasePanel();

                    // Rebuild category tabs and item list
                    UICommandBuilder builder = new UICommandBuilder();
                    UIEventBuilder eventBuilder = new UIEventBuilder();
                    buildCategoryTabs(builder, eventBuilder);
                    buildItemList(builder, eventBuilder);
                    sendUpdate(builder, eventBuilder, false);
                }
                break;
            }

            case SelectItem: {
                int index = parseIntSafe(data.index, -1);
                ShopCategory category = categories.get(selectedCategoryIndex);

                if (index >= 0 && index < category.getItems().size()) {
                    selectedItemIndex = index;
                    purchaseQuantity = 1;

                    // Update selection highlight
                    UICommandBuilder builder = new UICommandBuilder();
                    UIEventBuilder eventBuilder = new UIEventBuilder();
                    buildItemList(builder, eventBuilder);
                    sendUpdate(builder, eventBuilder, false);

                    // Show purchase panel
                    showPurchasePanel(category.getItems().get(index));
                }
                break;
            }

            case HoverItem: {
                int index = parseIntSafe(data.index, -1);
                ShopCategory category = categories.get(selectedCategoryIndex);

                if (index >= 0 && index < category.getItems().size()) {
                    ShopItem item = category.getItems().get(index);

                    // Show tooltip
                    UICommandBuilder builder = new UICommandBuilder();
                    builder.set("#Tooltip.Visible", true);
                    builder.set("#Tooltip #Description.TextSpans",
                        Message.translation(item.getDescriptionKey()));
                    sendUpdate(builder);
                }
                break;
            }

            case UnhoverItem: {
                UICommandBuilder builder = new UICommandBuilder();
                builder.set("#Tooltip.Visible", false);
                sendUpdate(builder);
                break;
            }

            case DecreaseQuantity: {
                if (purchaseQuantity > 1) {
                    purchaseQuantity--;
                    ShopItem item = getSelectedItem();
                    if (item != null) {
                        updateTotalPrice(item);
                    }
                }
                break;
            }

            case IncreaseQuantity: {
                ShopItem item = getSelectedItem();
                if (item != null) {
                    int maxQuantity = item.hasLimitedStock() ? item.getStock() : 99;
                    if (purchaseQuantity < maxQuantity) {
                        purchaseQuantity++;
                        updateTotalPrice(item);
                    }
                }
                break;
            }

            case SetQuantity: {
                int qty = parseIntSafe(data.quantity, 1);
                ShopItem item = getSelectedItem();
                if (item != null) {
                    int maxQuantity = item.hasLimitedStock() ? item.getStock() : 99;
                    purchaseQuantity = Math.max(1, Math.min(qty, maxQuantity));
                    updateTotalPrice(item);
                }
                break;
            }

            case ConfirmPurchase: {
                ShopItem item = getSelectedItem();
                if (item != null) {
                    int totalCost = item.getPrice() * purchaseQuantity;
                    int playerGold = getPlayerGold(player);

                    if (playerGold >= totalCost) {
                        // Deduct gold
                        setPlayerGold(player, playerGold - totalCost);

                        // Give items
                        ItemStack itemStack = new ItemStack(item.getItemId(), purchaseQuantity);
                        ItemContainer inventory = player.getInventory().getCombinedHotbarFirst();
                        inventory.addItemStack(itemStack);

                        // Update stock if limited
                        if (item.hasLimitedStock()) {
                            item.setStock(item.getStock() - purchaseQuantity);
                        }

                        // Send success message
                        playerRef.sendMessage(Message.translation("shop.purchase.success")
                            .param("quantity", purchaseQuantity)
                            .param("item", Message.translation(item.getTranslationKey())));

                        // Update UI
                        UICommandBuilder builder = new UICommandBuilder();
                        UIEventBuilder eventBuilder = new UIEventBuilder();
                        builder.set("#GoldDisplay.Text", formatGold(playerGold - totalCost));
                        buildItemList(builder, eventBuilder);
                        sendUpdate(builder, eventBuilder, false);

                        // Reset purchase panel
                        purchaseQuantity = 1;
                        hidePurchasePanel();
                    } else {
                        // Not enough gold
                        playerRef.sendMessage(Message.translation("shop.purchase.insufficient_gold")
                            .color("#FF0000"));
                    }
                }
                break;
            }

            case CancelPurchase: {
                selectedItemIndex = -1;
                purchaseQuantity = 1;
                hidePurchasePanel();

                // Clear selection highlight
                UICommandBuilder builder = new UICommandBuilder();
                UIEventBuilder eventBuilder = new UIEventBuilder();
                buildItemList(builder, eventBuilder);
                sendUpdate(builder, eventBuilder, false);
                break;
            }

            case Close: {
                player.getPageManager().setPage(ref, store, Page.None);
                break;
            }
        }
    }

    private ShopItem getSelectedItem() {
        if (selectedCategoryIndex >= 0 && selectedCategoryIndex < categories.size()) {
            ShopCategory category = categories.get(selectedCategoryIndex);
            if (selectedItemIndex >= 0 && selectedItemIndex < category.getItems().size()) {
                return category.getItems().get(selectedItemIndex);
            }
        }
        return null;
    }

    private int getPlayerGold(Player player) {
        // Implementation: get gold from player data
        return 1000; // Placeholder
    }

    private void setPlayerGold(Player player, int amount) {
        // Implementation: set gold in player data
    }

    private String formatGold(int amount) {
        return String.format("%,d G", amount);
    }

    private int parseIntSafe(String value, int defaultValue) {
        try {
            return Integer.parseInt(value);
        } catch (NumberFormatException e) {
            return defaultValue;
        }
    }

    // Action enum
    public enum Action {
        SelectCategory,
        SelectItem,
        HoverItem,
        UnhoverItem,
        DecreaseQuantity,
        IncreaseQuantity,
        SetQuantity,
        ConfirmPurchase,
        CancelPurchase,
        Close
    }

    // Event data class
    public static class PageData {
        public static final BuilderCodec<PageData> CODEC = BuilderCodec.builder(PageData.class, PageData::new)
            .append(
                new KeyedCodec<>("Action", new EnumCodec<>(Action.class, EnumCodec.EnumStyle.LEGACY)),
                (o, v) -> o.action = v,
                o -> o.action
            ).add()
            .append(
                new KeyedCodec<>("Index", Codec.STRING),
                (o, v) -> o.index = v,
                o -> o.index
            ).add()
            .append(
                new KeyedCodec<>("@Quantity", Codec.STRING),
                (o, v) -> o.quantity = v,
                o -> o.quantity
            ).add()
            .build();

        public Action action;
        public String index;
        public String quantity;
    }
}

// Supporting classes
class ShopCategory {
    private final String name;
    private final String icon;
    private final List<ShopItem> items;

    public ShopCategory(String name, String icon, List<ShopItem> items) {
        this.name = name;
        this.icon = icon;
        this.items = items;
    }

    public String getName() { return name; }
    public String getIcon() { return icon; }
    public List<ShopItem> getItems() { return items; }
}

class ShopItem {
    private final String itemId;
    private final String translationKey;
    private final String descriptionKey;
    private final int price;
    private int stock;
    private final boolean limitedStock;

    public ShopItem(String itemId, String translationKey, String descriptionKey,
                    int price, int stock, boolean limitedStock) {
        this.itemId = itemId;
        this.translationKey = translationKey;
        this.descriptionKey = descriptionKey;
        this.price = price;
        this.stock = stock;
        this.limitedStock = limitedStock;
    }

    public String getItemId() { return itemId; }
    public String getTranslationKey() { return translationKey; }
    public String getDescriptionKey() { return descriptionKey; }
    public int getPrice() { return price; }
    public int getStock() { return stock; }
    public void setStock(int stock) { this.stock = stock; }
    public boolean hasLimitedStock() { return limitedStock; }
}
```

**Opening the shop:**

```java
// Create shop categories and items
List<ShopCategory> categories = new ArrayList<>();

// Weapons category
List<ShopItem> weapons = List.of(
    new ShopItem("Iron_Sword", "shop.items.iron_sword", "shop.items.iron_sword.desc", 100, 0, false),
    new ShopItem("Diamond_Sword", "shop.items.diamond_sword", "shop.items.diamond_sword.desc", 500, 5, true)
);
categories.add(new ShopCategory("Weapons", "Icons/Shop/Weapons.png", weapons));

// Armor category
List<ShopItem> armor = List.of(
    new ShopItem("Iron_Helmet", "shop.items.iron_helmet", "shop.items.iron_helmet.desc", 75, 0, false),
    new ShopItem("Iron_Chestplate", "shop.items.iron_chestplate", "shop.items.iron_chestplate.desc", 150, 0, false)
);
categories.add(new ShopCategory("Armor", "Icons/Shop/Armor.png", armor));

// Open the shop
ShopPage shopPage = new ShopPage(playerRef, categories);
player.getPageManager().openCustomPage(ref, store, shopPage);
```

---

## Best Practices

### Page Design

**1. Use Descriptive Selectors:**

```java
// Good - clear hierarchy
"#MainPanel #ItemList[0] #NameLabel.Text"

// Bad - ambiguous
"#Label.Text"
```

**2. Organize Event Actions:**

```java
// Use enums for action types
public enum Action {
    Save,
    Cancel,
    SelectItem,
    // ...
}

// Reference in EventData
new EventData().append("Action", Action.Save.name())
```

**3. Use Non-blocking Events for Real-time Updates:**

```java
// Good - non-blocking for immediate feedback
eventBuilder.addEventBinding(
    CustomUIEventBindingType.ValueChanged,
    "#SearchInput",
    new EventData().append("Action", Action.Search.name()),
    false  // Don't lock interface
);

// Good - blocking for important actions
eventBuilder.addEventBinding(
    CustomUIEventBindingType.Activating,
    "#PurchaseButton",
    new EventData().append("Action", Action.Purchase.name()),
    true  // Lock interface while processing
);
```

### Performance

**1. Use Incremental Updates:**

```java
// Good - update only what changed
UICommandBuilder builder = new UICommandBuilder();
builder.set("#Counter.Text", newValue);
sendUpdate(builder);

// Bad - rebuild everything
rebuild();
```

**2. Batch Related Commands:**

```java
// Good - single update with multiple commands
UICommandBuilder builder = new UICommandBuilder();
builder.set("#Name.Text", name);
builder.set("#Level.Text", level);
builder.set("#Health.Value", health);
sendUpdate(builder);

// Bad - multiple updates
sendUpdate(new UICommandBuilder().set("#Name.Text", name));
sendUpdate(new UICommandBuilder().set("#Level.Text", level));
sendUpdate(new UICommandBuilder().set("#Health.Value", health));
```

**3. Clear Before Rebuilding Lists:**

```java
// Good - clear then rebuild
builder.clear("#ItemList");
for (int i = 0; i < items.size(); i++) {
    builder.append("#ItemList", "Pages/ListItem.ui");
    builder.set("#ItemList[" + i + "] #Label.Text", items.get(i).getName());
}
```

### Event Data

**1. Use Value References for Form Data:**

```java
// Good - capture values at event time
new EventData()
    .append("Action", "Submit")
    .append("@Username", "#UsernameInput.Value")
    .append("@Password", "#PasswordInput.Value")

// Bad - hardcoded values
new EventData()
    .append("Action", "Submit")
    .append("Username", "")  // Always empty!
```

**2. Include Context in Event Data:**

```java
// Good - include identifying information
new EventData()
    .append("Action", "SelectItem")
    .append("ItemId", item.getId())
    .append("Index", String.valueOf(index))
```

### Error Handling

**1. Validate Event Data:**

```java
@Override
public void handleDataEvent(Ref<EntityStore> ref, Store<EntityStore> store, PageData data) {
    if (data.action == null) {
        return; // Ignore invalid events
    }

    switch (data.action) {
        case SelectItem:
            int index = parseIntSafe(data.index, -1);
            if (index < 0 || index >= items.size()) {
                return; // Invalid index
            }
            // Process valid selection
            break;
    }
}
```

**2. Check Page State Before Updates:**

```java
private void updateUI() {
    Ref<EntityStore> ref = playerRef.getReference();
    if (ref == null || !ref.isValid()) {
        return; // Player disconnected
    }

    // Safe to update
    UICommandBuilder builder = new UICommandBuilder();
    builder.set("#Label.Text", "Updated");
    sendUpdate(builder);
}
```

---

## Summary Reference

### CustomUIPage Methods

| Method | Description |
|--------|-------------|
| `build(ref, commandBuilder, eventBuilder, store)` | Build initial UI |
| `rebuild()` | Completely rebuild the page |
| `sendUpdate()` | Send empty update |
| `sendUpdate(commandBuilder)` | Send incremental update |
| `sendUpdate(commandBuilder, clear)` | Send update with optional clear |
| `close()` | Close the page |
| `onDismiss(ref, store)` | Handle page dismissal |

### UICommandBuilder Methods

| Method | Description |
|--------|-------------|
| `append(path)` | Append document to root |
| `append(selector, path)` | Append to container |
| `appendInline(selector, json)` | Append inline JSON |
| `insertBefore(selector, path)` | Insert before element |
| `remove(selector)` | Remove element |
| `clear(selector)` | Clear children |
| `set(selector, value)` | Set property value |
| `setObject(selector, object)` | Set complex object |
| `setNull(selector)` | Set property to null |

### UIEventBuilder Methods

| Method | Description |
|--------|-------------|
| `addEventBinding(type, selector)` | Bind simple event |
| `addEventBinding(type, selector, data)` | Bind with data |
| `addEventBinding(type, selector, data, locksInterface)` | Bind with lock control |

### Page Lifecycle

```
1. openCustomPage() called
   ↓
2. build() called - create initial UI
   ↓
3. Page displayed to player
   ↓
4. Event occurs → handleDataEvent() called
   ↓
5. sendUpdate() - update UI as needed
   ↓
6. close() or setPage(Page.None) - close page
   ↓
7. onDismiss() called if player dismissed
```
