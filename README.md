# GUI Maker Fabric (1.21.1)

A Java/Fabric implementation of GUI Maker concepts: server-managed GUI
profiles, captured pages, chest-style screens and configurable slot actions.
The checked-in mod does not use a datapack `mcfunction` backend. The source
points to the original [GUI Maker repository](https://github.com/officialbarden/guimaker);
see [LICENSE](LICENSE) for the included MIT license and attribution.

This guide describes the checked-in source, not a verified release or a
multiplayer-safety guarantee. No Gradle build, Minecraft server or game test was
run for this README. The previous README is retained below for continuity.

## Versions and build

`gradle.properties` declares Minecraft `1.21.1`, Fabric Loader `0.16.9`,
Fabric API `0.102.0+1.21.1` and mod version `1.0.0-alpha.1`;
`fabric.mod.json` requests Java 21, Fabric API and Minecraft `~1.21.1`.
The build uses Java 21, Fabric Loom `1.6-SNAPSHOT` and the Gradle 8.8 wrapper.
Building can download dependencies and a snapshot plugin, so use a trusted
network and review the configuration:

```bash
./gradlew build
```

On Windows use `gradlew.bat build`. The build config produces a sources JAR as
well as the mod build output under `build/libs/`; inspect and test artifacts
before deployment. The existing GitHub workflow runs on pull requests and
pushes to `main`, runs `clean build` and uploads JAR artifacts. It is not a
substitute for a tested Minecraft installation.

## State and permissions

The `/guimaker` command root requires permission level **2**. Several commands
also require a player because they use the player's held item or the inventory
in the block immediately below. GUI profiles and per-player cached data are
stored in the overworld's `guimaker_state` persistent state. **Back up the world
before experimenting.**

Cached holder/toggle values are keyed by player UUID. Uncached values are kept
on the shared slot definition, so do not assume isolation between players for
uncached slots. A button's configured command is executed with the clicking
player's command source; its actual permissions and side effects depend on that
source and the server configuration. Treat imported data and command actions
as trusted administrative input, not as a sandbox.

## Basic workflow

1. `/guimaker profile create` creates an empty profile and reports its ID.
   `/guimaker profile list` lists profiles; deletion removes the named profile.
2. Stand above a block with an inventory and run
   `/guimaker page capture <guiId> [name...]`. Capture copies its item stacks
   into a new numbered page. Sizes are normalized to rows of nine, from 9 to
   54 slots, and at most 54 inventory slots are copied.
3. `/guimaker page list <guiId>`, `page rename <guiId> <page> <name...>` and
   `page delete <guiId> <page>` inspect or modify pages. Create/capture a page
   before opening it; a new profile contains no pages.
4. `/guimaker open <targets> <guiId> [page]` opens page 1 by default. The
   screen is a virtual generic container, not a spawned minecart.
5. `/guimaker items` gives the caller holder, toggle and data-driven template
   items. If inventory insertion fails, the code drops the item for the player.

The profile and page deletion handlers do not offer confirmation; the code also
reports success even if the requested ID was absent. Check IDs and backups
before using destructive commands.

## Slot configuration

Slot commands take `<guiId> <page> <slot>`, with a zero-based slot argument
from `0` through `53`. A number beyond the captured page's actual size can be
stored but is not rendered by that page. Available operations include:

- `slot icon ...` copies the player's main-hand stack; `slot clear ...` removes
  the slot definition.
- `slot type ... <type>` accepts `holder`/`item_holder`,
  `toggle`/`toggle_button`, `data_button`/`data_driven_button`, and
  `data_page`/`data_driven_page_creator`/`page_creator`. Other strings fall
  back to a simple button; notably, the old guide's `data_page_creator` spelling
  is **not** one of the recognized aliases.
- `slot cached ... <true|false>` selects per-player caching for supported
  holder/toggle values; `slot togglelist ...` copies nonempty stacks from the
  inventory below the player as toggle states.
- `slot action clear|command|function|itemmodifier|sound|changemenu ...`
  configures actions. `function` refers to a registered Java callback ID,
  **not** a vanilla datapack function. `itemmodifier` also refers to the Java
  registry, not an arbitrary Minecraft item-modifier file. The complete command
  forms are retained in the historical section below.

Quick-move, swap, throw and clone actions are blocked for GUI slots; item-holder
pickup handling and player inventory behavior are implemented in
`GuiMakerScreenHandler`. Data-driven button/page-creator types are recognized,
but this source routes them through the same action execution branch as simple
buttons rather than implementing all original datapack behavior.

## Import, export and extension points

`/guimaker export <guiId> [targets]` gives a barrel containing profile data in
item `custom_data`; `/guimaker import` reads the caller's held item's custom
data. Import uses profile IDs from the item and **replaces an existing profile
with the same ID without a confirmation prompt**. Imported page size is
reconstructed from item slots with a minimum of 27, so an export/import is not
guaranteed to retain the original size of every captured page. Review the data,
back up the world, and test the round trip before relying on it.

Java integrations can register a `GuiButtonCallback` or `GuiItemModifier` via
`GuiMakerApi`. Built-ins are `guimaker:close`, `guimaker:refresh`,
`guimaker:message` and the `guimaker:noop` item modifier. Unknown callback or
modifier identifiers do not magically load datapack functions.

## Historical README (preserved)

The following is the previous README, retained verbatim. Its unqualified
multiplayer-safety language and `data_page_creator` command spelling should be
read in light of the source-specific limitations above.

~~~~~~text
# GUI Maker Fabric (1.21.1)

Java-only Fabric conversion of the original GUI Maker datapack:
- Target: Minecraft 1.21.1 + Fabric
- No internal `mcfunction` usage
- Multiplayer-safe persistent server state

## What is implemented

- Persistent GUI profiles/pages in Java (`PersistentState`)
- Virtual chest-style GUIs opened directly from Fabric code
- Multiplayer-safe per-player caching for:
  - Item Holders
  - Toggle Buttons
- GUI export/import as a custom barrel item
- Datapack-like slot metadata parsing from item `custom_data.gui`
- Server commands for creating, editing, deleting, exporting, importing and opening GUIs
- Java callback registry for button functions
- Java item-modifier registry for dynamic item rendering

## Core commands

### Create/list/delete
- `/guimaker profile create`
- `/guimaker profile list`
- `/guimaker profile delete <guiId>`

### Capture pages from a container block below the player
- `/guimaker page capture <guiId>`
- `/guimaker page capture <guiId> <name...>`
- `/guimaker page list <guiId>`
- `/guimaker page rename <guiId> <page> <name...>`
- `/guimaker page delete <guiId> <page>`

### Open GUI for players
- `/guimaker open <targets> <guiId>`
- `/guimaker open <targets> <guiId> <page>`

### Special items
- `/guimaker items`

This gives:
- Item Holder [UNCACHED]
- Item Holder [CACHED]
- Toggle Button [UNCACHED]
- Toggle Button [CACHED]
- Data Driven Item Button
- Data Driven Page Creator

### Edit slot config
- `/guimaker slot icon <guiId> <page> <slot>` (copies mainhand item)
- `/guimaker slot type <guiId> <page> <slot> <simple|holder|toggle|data_button|data_page_creator>`
- `/guimaker slot cached <guiId> <page> <slot> <true|false>`
- `/guimaker slot togglelist <guiId> <page> <slot>` (captures states from container below player)
- `/guimaker slot clear <guiId> <page> <slot>`

### Edit slot actions
- `/guimaker slot action clear <guiId> <page> <slot>`
- `/guimaker slot action command <guiId> <page> <slot> <command...>`
- `/guimaker slot action function <guiId> <page> <slot> <namespace:id>`
- `/guimaker slot action itemmodifier <guiId> <page> <slot> <namespace:id>`
- `/guimaker slot action sound <guiId> <page> <slot> <namespace:id>`
- `/guimaker slot action changemenu <guiId> <page> <slot> <targetGuiId> <targetPage>`

### Import / export
- `/guimaker export <guiId>`
- `/guimaker export <guiId> <targets>`
- `/guimaker import`

Hold the exported barrel in your main hand and run `/guimaker import`.

## Java extension points

### Button callback registry
Register callbacks through `GuiMakerApi.registerButtonCallback(...)`.

Built-ins:
- `guimaker:close`
- `guimaker:refresh`
- `guimaker:message`

### Item modifier registry
Register item modifiers through `GuiMakerApi.registerItemModifier(...)`.

Built-in:
- `guimaker:noop`

## Notes

- This mod intentionally replaces the datapack's command-function backend with Java logic.
- The chest GUI is virtual; it is not implemented through marker entities + tick polling.
- `data_driven_button` and `data_driven_page_creator` are recognized and preserved, but their advanced datapack behavior is left as extension points for future Java-side expansion.
- The workspace environment used to generate this project does not include Java 21, so the project was prepared but not built inside the sandbox.

## Build

Use Java 21.

```bash
./gradlew build
```
~~~~~~
