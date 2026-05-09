# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Repository Overview

**Warhammer Foundry Monorepo** is an NPM workspace containing four related Foundry VTT modules—three game systems and one shared library. All packages target Foundry VTT 14.x and use ES modules with Rollup for bundling.

- **Repository Type**: Monorepo with NPM workspaces and git submodules
- **Main Entry Points**: Each package has its own `src/` directory with a main `.js` file
- **Build Output**: Rollup bundles to the local Foundry VTT installation directory (resolved via `foundry-path.js/mjs`)

## Package Architecture

### Package Relationships

```
warhammer-library (shared library)
    ↑
    └── Required by: wfrp4e, impmal, oldworld
    └── Provides: Base classes, UI components, utilities
```

**warhammer-library** is a Foundry VTT library module (not a playable system) that provides:
- Base document classes: `WarhammerActor`, `WarhammerItem`, `WarhammerActiveEffect`, `WarhammerChatMessage`
- Data model base classes: `BaseWarhammerActorModel`, `BaseWarhammerItemModel`, `WarhammerActiveEffectModel`
- Shared UI components: Dialogs, forms, compendium browser, effect editors
- Utilities: Socket handlers, sheet helpers, combat helpers, area templates, zone effects
- Hooks: System-wide event handlers for combat, journals, tables, targeting, etc.

The library exports a global `warhammer` namespace that child systems extend/override.

### The Four Packages

| Package | ID | Type | Purpose |
|---------|----|----|---------|
| `packages/warhammer-library` | `warhammer-lib` | Library Module | Shared base classes and utilities |
| `packages/wfrp4e` | `wfrp4e` | Game System | Warhammer Fantasy Roleplay 4th Edition |
| `packages/impmal` | `impmal` | Game System | Warhammer 40k: Imperium Maledictum |
| `packages/oldworld` | `oldworld-foundryvtt` | Game System | Warhammer: The Old World |

Each package is a git submodule pointing to a separate GitHub repository (Tiamanti's account).

## Build System

### Key Build Concepts

**Local Foundry Path Resolution**: Each package uses `foundry-path.js/mjs` to resolve the local Foundry VTT installation directory. This is critical for watch mode to output files in the correct location. The path is typically:
```
Windows: %AppData%\FoundryVTT\Data\modules/{id}
```

**Build Process**:
1. `npm run build` (per package) → Rollup watch mode for development
2. `npm run release` (per package) → Rollup production build
3. `npm run build:all` (root) → Production build all packages

### Building

**Single package development** (with watch mode):
```bash
cd packages/{package-name}
npm install  # if needed
npm run build
```

**Single package production release**:
```bash
cd packages/{package-name}
npm run release
```

**All packages at once** (production, no watch):
```bash
npm run build:all
```

**Specific package from root**:
```bash
npm run build:warhammer-library
npm run build:wfrp4e
npm run build:impmal
npm run build:oldworld
```

### Rollup Configuration

- **Input**: Main entry file in `src/` (e.g., `src/wfrp4e.js`)
- **Output**: Bundled JS + CSS to local Foundry directory
- **Plugins**:
  - `rollup-plugin-copy`/`rollup-plugin-copy-watch` – Copies `static/` assets (Handlebars templates, images, JSON configs)
  - `rollup-plugin-postcss` – Processes CSS/SCSS and extracts to separate file
  - `rollup-plugin-baked-env` – Injects environment variables
- **Environment**: Controlled via `cross-env NODE_ENV=development|production`
- **Custom Packers**: wfrp4e and impmal use `scriptPacker.js` (and wfrp4e also `dbpacker.js`) to prepare additional data files

## Linting & Code Style

**ESLint is configured** for warhammer-library and impmal:
- Config format: Flat config (ESLint 9+) in `eslint.config.js`
- Rules enforced:
  - Indentation (error)
  - Semicolons (error)
  - Braces: Allman style with single-line allowed
  - No unused vars turned off (allow unused), but no undefined is a warning
  - JSDoc validation enabled but permissive on types
- **Globals**: All Foundry VTT APIs pre-declared (foundry, game, CONFIG, ui, canvas, Hooks, Actor, Item, etc.)

**Lint commands**:
```bash
# warhammer-library
cd packages/warhammer-library
npm run lint

# impmal
cd packages/impmal
npm run lint
npm run fix-lint  # auto-fix
```

wfrp4e and oldworld do not currently have lint configs set up.

## Critical Files

| File | Purpose |
|------|---------|
| `packages/{package}/foundry-path.js(mjs)` | Resolves local Foundry VTT installation path for watch output |
| `packages/{package}/rollup.config.mjs(.js)` | Bundler configuration |
| `packages/{package}/system.json` (systems) or `module.json` (library) | Foundry manifest (entry point list, version, compatibility) |
| `packages/{package}/src/{id}.js` | Main entry point that imports and registers document classes, sheets, hooks |
| `static/templates/` | Handlebars templates (auto-copied to dist) |
| `static/assets/` | Images and other assets |

## Module Dependencies

### warhammer-library internals

```
src/warhammer-lib.js (entry point)
├── imports system/config.js, document classes, hooks, utilities
├── defines global `warhammer` namespace with:
│   ├── warhammer.utility (helper functions)
│   ├── warhammer.apps (UI classes)
│   ├── warhammer.models (data model classes)
│   └── warhammer.documents (document class overrides)
└── registers custom elements (FilterStateElement, CheckboxElement)
```

Key subdirectories:
- `src/model/` – Data models inheriting from Foundry's DataModel
- `src/apps/` – UI applications (dialogs, forms, browsers)
- `src/document/` – Document class overrides (Actor, Item, ActiveEffect, etc.)
- `src/sheets/` – Actor and Item sheet implementations
- `src/hooks/` – Registered Foundry event listeners
- `src/util/` – Helper utilities (socket handlers, sheet helpers, etc.)
- `src/system/` – Configuration, commands, tests

### Game Systems (wfrp4e, impmal, oldworld)

Each system's entry point (`wfrp4e.js`, etc.) typically:
1. Imports actor/item sheets from `sheets/`
2. Imports document classes from `documents/`
3. Imports data models from `model/`
4. Imports system config and hooks
5. Registers all classes with Foundry's CONFIG object
6. Sets up compatibility/migrations if needed

Data models inherit from library base classes:
- Actor models → `BaseWarhammerActorModel`
- Item models → `BaseWarhammerItemModel`
- Active Effect models → `WarhammerActiveEffectModel`

## Known Issues & TODOs

See `TODO.md` for wfrp4e-specific issues:
- Bleeding roll endurance calculation
- Edit test permission checks
- /pay command messaging
- Alcohol consumption auto-add on fails
- Integration with UiA module for Pursuit and Group Advantage mechanics

## Testing

**No automated tests configured** (root `package.json` has placeholder `npm run test`). Testing is manual through Foundry VTT.

## Setup & Development

### First-time clone

```bash
git clone --recurse-submodules <repo-url>
cd warhammer-foundry-monorepo
npm install
```

### If cloned without submodules

```bash
git submodule update --init --recursive
npm install
```

### Configure Local Foundry Path

Each package looks for `foundry-path.js/mjs` to resolve your Foundry installation. Edit or create this file to point to your local Foundry VTT Data directory:

```javascript
// foundry-path.js example
export default function() {
  return "C:\\Users\\YourName\\AppData\\Roaming\\FoundryVTT\\Data\\modules\\{module-id}";
}
```

Alternatively, create `foundryconfig.json` in the package root (copy from `example.foundryconfig.json`).

### Update git submodules

```bash
# Pull latest from all submodules
git submodule update --remote --merge

# Pull latest from single submodule
git submodule update --remote --merge packages/wfrp4e
```

## Debugging Tips

1. **Watch mode not outputting to Foundry?** Check that `foundry-path.js/mjs` returns the correct path. Print debug output or manually test the path exists.

2. **Import resolution issues?** Remember all packages use `"type": "module"` (ES modules). Use `.js` extension in imports, not `.mjs`.

3. **Submodule out of sync?** Run `git submodule update --init --recursive` and verify each package's `.git` directory exists.

4. **CSS/Templates not updating?** Rollup's copy watch should catch `static/` changes. If not, restart watch mode.

5. **Foundry not picking up changes?** Check module.json/system.json is copied to the output directory. The `rollup-plugin-copy` task should handle this, but verify `static/` contains the manifest file.

## Notes for Future Work

- **wfrp4e** and **oldworld** lack ESLint configs; consider adding them for consistency
- Test framework is not set up; consider adding vitest or similar if unit tests become needed
- Documentation for extending the library is in `packages/warhammer-library/README.md` (integration checklist)
