# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

Space Wars 3002 is a modern re-imagining of the classic BBS door game Trade Wars 2002. It's a turn-based space trading and conquest game built with Laravel (PHP 8.3+) featuring procedurally generated galaxies, trading mechanics, colonies, combat, and multiple victory paths.

## Commands

### Development
```bash
# Install dependencies
composer install

# Database setup
php artisan migrate

# Run tests (PHPUnit)
php artisan test
# or
vendor/bin/phpunit

# Run specific test
vendor/bin/phpunit tests/Unit/Services/TravelServiceTest.php

# Code linting (Laravel Pint)
vendor/bin/pint

# Static analysis (PHPStan)
vendor/bin/phpstan analyse

# Tinker (REPL for testing)
php artisan tinker
```

### Frontend
```bash
npm install
npm run dev    # Development with Vite
npm run build  # Production build
```

### Galaxy Management
```bash
# Initialize complete galaxy (all-in-one)
# Auto-calculates warp gate adjacency based on dimensions (max_dimension / 15)
php artisan galaxy:initialize {name?} --width=300 --height=300 --stars=3000 --grid-size=10

# Skip optional steps
php artisan galaxy:initialize --skip-gates --skip-pirates --skip-mirror

# Individual galaxy commands
php artisan galaxy:expand {galaxy} --stars=3000
php artisan galaxy:generate-sectors {galaxy} --grid-size=10
php artisan galaxy:generate-gates {galaxy} --adjacency=20  # Auto-calculated in galaxy:initialize
php artisan galaxy:distribute-pirates {galaxy}
php artisan galaxy:create-mirror {galaxy}

# Designate inhabited systems (33-50% recommended)
php artisan galaxy:designate-inhabited {galaxy} --percentage=0.40 --min-spacing=50

# Visualization
php artisan galaxy:view {galaxy}

# Trading & economy
php artisan trading:assign-production {galaxy}
php artisan trading:hub:generate {galaxy}
php artisan trading:hub:populate-inventory {galaxy}

# Cartography (star chart vendors)
php artisan cartography:generate-shops {galaxy} --spawn-rate=0.3 --regenerate
```

### Player Management
```bash
# Initialize player in galaxy
php artisan player:initialize {galaxy_id} {user_id} --call-sign="PlayerName"

# Launch player interface (Console Interface)
php artisan player:interface {player_id}
# Available commands in player interface:
#   [v] - View star map (shows nearby systems within sensor range)
#   [j] - Jump to coordinates (direct travel to any system)
#   [w] - Warp gate travel (use existing gates)
#   [t] - Trade minerals (at trading hubs)
#   [s] - Ship information
#   [c] - Cargo manifest

# Upgrade ship
php artisan ship:upgrade {player_ship_id} {component} {level}
```

### Game Loop
```bash
# Process market events
php artisan market:process-events

# Process colony cycles
php artisan colonies:process-cycles
```

## Architecture

### Core Game Loop
The game is turn-based with players spending limited turns per day. Each action (travel, trade, combat, mining) consumes turns.

### Domain Model Structure

**Galaxy Generation**: Multi-stage procedural generation system
1. Galaxy creation with configurable dimensions and parameters
2. Point of Interest (POI) distribution using pluggable generators
3. Sector grid overlay for region-based mechanics
4. Inhabited system designation (33-50% of stars)
5. Warp gate network generation for inhabited systems only (uninhabited systems remain isolated)
6. Trading hub distribution (50-80% of inhabited systems)
7. Pirate distribution on warp lanes
8. Resource/mineral assignment

**Point Generators** (`app/Generators/Points/`): Pluggable system via `PointGeneratorInterface`
- `PoissonDisk`: Blue noise distribution with minimum spacing
- `RandomScatter`: Pure random placement
- `HaltonSequence`: Low-discrepancy quasi-random
- `VogelsSpiral`: Sunflower-pattern spiral
- `StratifiedGrid`: Grid-based with jitter
- `LatinHypercube`: Stratified random sampling
- `R2Sequence`: R2 low-discrepancy sequence
- `UniformRandom`: Baseline random distribution

Selected via `config/game_config.php` → `galaxy.generator` and instantiated via `PointGeneratorFactory` using reflection.

**Core Models**:
- `Galaxy`: Top-level container with snapshotted configuration
- `PointOfInterest` (POI): Generic spatial entity (stars, planets, belts, nebulae, stations)
  - Hierarchical: POIs can have parent POI (e.g., planet orbiting star)
  - Supports trading hubs, ship shops, repair shops, plan shops
  - `is_inhabited` flag for civilized systems (40% of stars by default, range: 33-50%)
  - Only inhabited systems receive warp gates and trading hubs
  - Uninhabited systems remain isolated to encourage exploration and colonization
- `Sector`: Grid-based regions overlaying POIs (e.g., 10x10 sectors)
- `WarpGate`: Connections between POIs (can be hidden, dead-ends, or mirror universe gates)
- `Player`: Player state including credits, location (`current_poi_id`), level, XP
  - Star charts tracking: `player_star_charts` pivot table for revealed systems
- `PlayerShip`: Ship instances with upgrades (weapons, hull, warp_drive, cargo, sensors, shields)
  - Fuel regeneration: `current_fuel`, `max_fuel`, `fuel_last_updated_at`
- `Ship`: Ship blueprints/templates (scout, freighter, battleship, etc.)
- `Colony`: Player-controlled colonies on planets with population, buildings, mineral production
- `Mineral`: Tradable resources with dynamic pricing and scarcity
- `TradingHub`: Markets for buying/selling minerals
- `StellarCartographer`: Star chart vendors at inhabited trading hubs (spawn rate ~30%)
- `PirateFaction`/`PirateCaptain`/`PirateFleet`: NPC hostile entities

**Services** (`app/Services/`): Business logic layer
- `TravelService`: Movement through warp gates, fuel calculations, XP rewards
  - Formula: Fuel cost = ceil(distance / efficiency), efficiency = 1 + (warp_drive - 1) * 0.2
  - XP = max(10, distance * 5)
- `FuelRegenerationService`: Time-based fuel regeneration (10 units/hour base, scaled by warp drive)
  - Formula: Regen rate = BASE_RATE * (1 + (warp_drive - 1) * 0.3) units/hour
- `TradingService`: Buy/sell minerals at trading hubs
- `CombatResolutionService`: Combat between player and pirates
- `MiningService`: Extract minerals from belts/planets
- `ColonyCycleService`: Population growth, resource production
- `MirrorUniverseService`: High-risk parallel dimension with 2x resources, 2x pirate difficulty
- `PirateEncounterService`: Random pirate encounters based on sector danger
- `MarketEventService`/`MarketEventGenerator`: Dynamic market price fluctuations
- `ShipUpgradeService`/`ShipRepairService`: Ship management
- `StarChartService`: Star chart purchasing, coverage calculation via BFS (2-hop default)
  - Exponential pricing: basePrice * (multiplier ^ unknownCount)
  - Pirate detection with probabilistic accuracy (70-95% based on sensors)
- `InhabitedSystemGenerator`: Distributes inhabited systems with minimum spacing constraints

**Console Commands** (`app/Console/Commands/`):
- Galaxy generation commands (see Commands section above)
- Player management commands
- Market/economy processing
- Console-based player interface (`PlayerInterfaceCommand`)

**Shop Handlers** (`app/Console/Shops/`): Interactive console interfaces for trading and services
- `MineralTradingHandler`: Buy/sell minerals at trading hubs
- `ComponentShopHandler`: Purchase ship component upgrades
- `RepairShopHandler`: Repair ship hull damage
- `ShipShopHandler`: Purchase or switch ships
- `PlansShopHandler`: Buy rare upgrade plans
- `PirateEncounterHandler`: Handle combat encounters during travel

**Player Interface Features**:
- Inhabited status displayed for all systems (inhabited vs uninhabited)
- Warp gate count shown in location view (always visible)
- Star map viewer: Shows nearby systems within sensor range (sensors × 100 units)
- Coordinate jump: Direct travel to any system using coordinates
- Sensor upgrades increase visibility range (100 units per level)

### Configuration System

**`config/game_config.php`**: Single source of truth for game balance and mechanics
- Galaxy dimensions, point count, generator selection
- Warp gate probabilities (hidden, dead-end, jackpot)
- Economy: ore types, price fluctuation, scarcity bias
- Ships: starting credits, ship classes (cargo, speed, combat)
- Victory conditions: merchant credits, colonization %, conquest %, pirate power
- Mirror universe: sensor requirements, cooldowns, risk/reward multipliers
- Turn limits per day

**Galaxy Config Snapshotting**: Each Galaxy model snapshots the config at creation time, allowing different galaxies to have different rules.

### Database Patterns

**Migrations** (`database/migrations/`):
- Follow Laravel conventions
- Relationships: Players → PlayerShips → Ships (blueprint), Players → current_poi_id (PointOfInterest)
- Pivot tables: `player_plans`, `player_cargos`, `trading_hub_inventories`

**Eloquent Relationships**:
- Player `belongsTo` PointOfInterest (current location)
- Player `hasMany` PlayerShips, `hasOne` activeShip
- PointOfInterest `hasMany` children (orbital bodies), `belongsTo` parent
- PointOfInterest polymorphic relations to TradingHub, Colony, etc.
- WarpGate connects two POIs (`from_poi_id`, `to_poi_id`)

### Victory Paths

Four distinct win conditions (configured in `config/game_config.php`):
1. **Merchant Empire**: Accumulate 1 billion credits
2. **Colonization**: Control >50% of galactic population
3. **Conquest**: Control >60% of star systems
4. **Pirate King**: Seize >70% of outlaw network

### Mirror Universe

Ultra-rare high-risk/high-reward parallel dimension:
- 1 mirror gate per galaxy, requires sensor level 5 to detect
- 2x resource spawns, 1.5x trading prices, 3x rare mineral spawn rate
- 2x pirate difficulty, 1.5x pirate fleet sizes
- 24-hour cooldown before returning to prevent abuse

## Testing

**Unit Tests** (`tests/Unit/`):
- Services: TravelService, TradingService, CombatResolutionService, etc.
- Models: Player, Colony, PlayerShip, ColonyBuilding
- Point generators: PointGeneratorsTest (count, bounds, spacing, determinism)

**Feature Tests** (`tests/Feature/`):
- End-to-end galaxy generation
- Integration between services and models

**Test Conventions**:
- Use in-memory database or test database (see `phpunit.xml`)
- Factory pattern for model creation (`database/factories/`)
- Mock external dependencies where appropriate

## Key Patterns & Conventions

**Service Pattern**: All game logic in dedicated service classes, not in models or controllers
- Models = data + Eloquent relationships
- Services = business rules + calculations
- Commands = orchestration + user interaction

**Point Generator Factory**: Uses reflection to instantiate generators by name from config
- Interface: `PointGeneratorInterface` with `sample(Galaxy): array` method
- Abstract base class: `AbstractPointGenerator` with shared utilities
- DensityGuard: Enforces MAX_DENSITY = 0.65 to prevent impossible configurations

**Console Interface Pattern**: Interactive console-based gameplay
- ANSI-based terminal interface with color support
- Keyboard-driven menus and navigation
- Shop handlers for specialized interactions
- Non-blocking input for real-time gameplay

**Weighted Random**: Custom package `mschandr/weighted-random` for probability-based selections (pirate encounters, market events, etc.)

## Important Notes

- The game uses **UUIDs** for galaxies and players (not auto-increment IDs)
- **Turns are precious**: Every action should consider turn cost
- **Mirror universe gates** are intentionally rare (1 per galaxy) and require high sensor level
- **Point generators** are deterministic when using the same seed (important for testing)
- **Inhabited systems** are 33-50% of all stars (default 40%), distributed with minimum spacing (default 50 units)
- **Warp gates** connect only inhabited systems - uninhabited systems must be reached by coordinates
  - Adjacency threshold auto-calculated: max(width, height) / 15
- **Trading hubs** spawn at 50-80% of inhabited systems (default 65%)
- **Star charts** use BFS traversal through warp gates to calculate coverage (2 hops default)
  - Players start with 3 free charts to nearest inhabited systems
  - Charts sold at Stellar Cartographer shops (~30% of trading hubs)
  - Exponential pricing discourages repeated small purchases
- **Fuel regeneration** is passive and time-based (better warp drives = faster regen)
- **Laravel Telescope** is available in dev for debugging (disabled in tests)
- Uses **PHP 8.3** features (typed properties, constructor promotion, etc.)
- Frontend uses **Vue 3 + Inertia.js** for modern SPA experience (visualization tools)
