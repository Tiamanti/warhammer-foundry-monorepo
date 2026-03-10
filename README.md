# Warhammer Foundry Monorepo

Monorepo for Warhammer Foundry VTT modules. Uses npm workspaces with git submodules.

## Packages

| Directory | Module | GitHub |
|-----------|--------|--------|
| `packages/warhammer-library` | Warhammer Library (shared base) | [WarhammerLibrary-FVTT](https://github.com/Tiamanti/WarhammerLibrary-FVTT) |
| `packages/wfrp4e` | Warhammer Fantasy Roleplay 4e | [WFRP4e-FoundryVTT](https://github.com/Tiamanti/WFRP4e-FoundryVTT) |
| `packages/impmal` | Warhammer 40k: Imperium Maledictum | [ImpMal-FoundryVTT](https://github.com/Tiamanti/ImpMal-FoundryVTT) |
| `packages/oldworld` | Warhammer: The Old World | [OldWorld-FoundryVTT](https://github.com/Tiamanti/OldWorld-FoundryVTT) |

The game modules (`wfrp4e`, `impmal`, `oldworld`) depend on `warhammer-library` being loaded first in Foundry VTT. They do not depend on each other.

## Setup

### Clone (first time)

```bash
git clone --recurse-submodules <repo-url>
cd warhammer-foundry-monorepo
npm install
```

### Clone (if already cloned without submodules)

```bash
git submodule update --init --recursive
npm install
```

## Development

Each package has its own `build` script that runs Rollup in watch mode.

### Build a single package

```bash
cd packages/warhammer-library   # or wfrp4e, impmal, oldworld
npm install
npm run build
```

### Build all packages (one-shot, no watch)

```bash
npm run build:all
```

> Note: `build:all` runs `release` mode. For watch mode during active development, run `npm run build` inside the individual package directory.

## Scripts

| Script | Description |
|--------|-------------|
| `npm run build` (per package) | Rollup watch mode (development) |
| `npm run release` (per package) | Rollup production build |
| `npm run build:all` (root) | Production build across all packages |
| `npm run lint` (warhammer-library, impmal) | Run ESLint |

## Updating Submodules

To pull the latest commits for all submodules:

```bash
git submodule update --remote --merge
```

To update a single submodule:

```bash
git submodule update --remote --merge packages/warhammer-library
```

## Resources

- [Foundry VTT Types](https://github.com/League-of-Foundry-Developers/foundry-vtt-types) — IDE type support (installed as root devDependency)
