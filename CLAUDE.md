# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Development Commands

**Requirements:** JDK 17 (other versions will not work)

### Running and Building
- **Run desktop**: `./gradlew desktop:run` (Linux/Mac) or `gradlew desktop:run` (Windows)
- **Build desktop**: `./gradlew desktop:dist` → Output: `desktop/build/libs/Mindustry.jar`
- **Build server**: `./gradlew server:dist` → Output: `server/build/libs/server-release.jar`
- **Pack sprites**: `./gradlew tools:pack` (required for asset changes)

### Testing and Development
- **Run tests**: `./gradlew tests:test` (tests run in `core/assets` working directory)
- **Android debug**: `./gradlew android:assembleDebug` (requires Android SDK setup)
- **Clear cache**: `./gradlew clearCache` (clears `core/assets/cache`)

## Architecture Overview

Mindustry is a multi-platform tower defense RTS game written in Java using the Arc framework.

### Module Structure
- **core/**: Main game logic, shared across all platforms
- **desktop/**: Desktop-specific launcher and natives
- **server/**: Headless server implementation
- **android/**: Android platform-specific code
- **ios/**: iOS platform wrapper
- **tools/**: Build tools and sprite packing utilities
- **tests/**: JUnit test suite
- **annotations/**: Code generation annotations and processors

### Core Package Organization
- **mindustry.core**: Game state management, UI core, world control
- **mindustry.entities**: Entity component system, units, effects
- **mindustry.world**: Block system, tile management, world simulation
- **mindustry.content**: Game content definitions (blocks, units, items, etc.)
- **mindustry.ui**: User interface components and dialogs
- **mindustry.net**: Networking, multiplayer, packets
- **mindustry.logic**: Programming system (logic blocks)
- **mindustry.ai**: AI behavior for units and enemies
- **mindustry.game**: Game rules, objectives, campaign
- **mindustry.io**: Save/load system, map importing/exporting
- **mindustry.mod**: Modding system and Rhino scripting

### Code Generation
The `mindustry.gen` package is generated at build time from:
- `@Remote` annotated methods → `Call` and packet classes
- Component classes in `mindustry.entities.comp` → Entity classes
- Asset files → `Sounds`, `Musics`, `Tex`, `Icon` constants

### Key Global State
- **Vars.java**: Central hub for global variables and game state
- **Core** (from Arc): Application lifecycle, assets, input, graphics
- **content.** static imports: Access to game content (Blocks, Items, UnitTypes, etc.)

## Code Style Requirements

Follow the established formatting in CONTRIBUTING.md:
- No spaces around parentheses: `if(condition){`
- Same-line braces, 4-space indentation
- camelCase for everything (including constants/enums)
- No underscores, short variable names preferred
- Wildcard imports: `import some.package.*`
- Use `arc.struct` collections (`Seq`, `ObjectMap`) over Java standard collections
- Avoid allocations in main loop - use `Pools`, `Tmp` variables, or static reuse
- Avoid boxed types - use specialized collections (`IntSeq`, `IntMap`)
- No getters/setters unless necessary - prefer public fields

### Platform Compatibility
- No Java 8+ features (`java.util.function`, `java.util.stream`, forEach)
- Use `arc.func` functional interfaces instead
- No `java.awt` package (not supported on any platform)
- IntelliJ will warn about platform incompatibilities

## Testing
Tests use JUnit 5 and run with working directory `core/assets`. Each test forks separately to prevent mod interaction.