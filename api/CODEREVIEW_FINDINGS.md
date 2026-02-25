# Code Quality Audit: Duplication & Architectural Issues

Date: 2026-02-24
Scope: Full codebase search for duplication, dead code, and anti-patterns

---

## CRITICAL FINDINGS

### 1. MAJOR DUPLICATION: Player Creation Flow (High Priority)

#### Issue
Two conflicting implementations of player creation logic across three entry points:

**Location A: GalaxyController.join()** (COMPLETE IMPLEMENTATION)
- File: `/home/mdhas/workspace/space-wars-3002/app/Http/Controllers/Api/GalaxyController.php`
- Lines: 565-670
- Pattern: Inline player creation with full spawn discovery logic
- Implements:
  1. Find random inhabited starting location (prefer with trading hub) - lines 566-588
  2. Create Player with starting credits - lines 594-603
  3. Create PlayerShip with starter attributes - lines 606-640
  4. Ensure spawn system has a name - line 645
  5. Create minimum scan for spawn location (level 1) - lines 648-649
  6. Discover outgoing lanes from spawn - lines 652-653
  7. Grant spawn location star chart (free) - lines 656-664
  8. Grant starting charts for nearby inhabited systems - lines 667-668

**Location B: PlayerController.store()** (INCOMPLETE IMPLEMENTATION)
- File: `/home/mdhas/workspace/space-wars-3002/app/Http/Controllers/Api/PlayerController.php`
- Lines: 36-115
- Pattern: Delegates to PlayerSpawnService but MISSING spawn discovery
- Missing:
  - System name generation
  - System scan creation
  - Lane knowledge discovery
  - Star chart discovery (TODO comment at line 98)

**Location C: InitializePlayerCommand** (COMPLETE IMPLEMENTATION)
- File: `/home/mdhas/workspace/space-wars-3002/app/Console/Commands/InitializePlayerCommand.php`
- Lines: 35-131
- Pattern: Uses PlayerSpawnService + StarChartService correctly
- Implements all spawn discovery steps (via services)

#### Root Cause
- GalaxyController implements spawn logic inline instead of delegating to PlayerSpawnService
- PlayerController and InitializePlayerCommand use services but inconsistently
- PlayerSpawnService exists but GalaxyController bypasses it entirely

#### Impact
- Two different player creation flows produce different results
- GalaxyController players get full spawn discovery; PlayerController players get partial setup
- Risk of bugs if one implementation is fixed and the other isn't
- Maintenance nightmare: changes must be made in 3 places

#### Remediation
Extract unified player creation to PlayerSpawnService with complete flow:
1. Move lines 565-588 logic from GalaxyController to service method
2. Create `PlayerSpawnService::createPlayerAtSpawnLocation()` encapsulating:
   - Spawn location selection
   - Player creation
   - Ship creation
   - System name generation
   - Scan creation
   - Lane discovery
   - Star chart grants
3. Update both GalaxyController.join() and PlayerController.store() to use this method
4. InitializePlayerCommand already has the pattern - just reuse

**Code Block Duplication:**
```php
// Lines 566-588 GalaxyController.join() - DUPLICATES PlayerSpawnService::findOptimalSpawnLocation()
$startingLocation = $galaxy->pointsOfInterest()
    ->where('type', PointOfInterestType::STAR)
    ->where('is_inhabited', true)
    ->whereHas('tradingHub')
    ->inRandomOrder()
    ->first();
```

---

### 2. LEGACY METHODS IN PlanetarySystemGenerator (Medium Priority)

#### Issue
Four private methods exist in duplicate with different implementations:

**File:** `/home/mdhas/workspace/space-wars-3002/app/Services/GalaxyGeneration/Generators/PlanetarySystemGenerator.php`

| Method | Optimized Version | Legacy Version | Lines | Status |
|--------|------------------|-----------------|-------|--------|
| `determinePlanetType()` | `getPlanetTypeForStar()` | `determinePlanetType()` | 577-603 vs 372-431 | LEGACY |
| `determineMoonCount()` | `getMoonCount()` | `determineMoonCount()` | 608-619 vs 479-492 | LEGACY |
| `getPlanetSize()` | `getPlanetSizeIndex()` | `getPlanetSize()` | 624-632 vs 497-505 | LEGACY |
| `generatePlanetarySystem()` | `generateStarSystem()` | `generatePlanetarySystem()` | 510-572 vs 263-347 | LEGACY |

#### Details

**Legacy method: generatePlanetarySystem()** (Lines 510-572)
```php
private function generatePlanetarySystem(PointOfInterest $star, int $planetCount, $now): array
{
    for ($i = 1; $i <= $planetCount; $i++) {
        $type = $this->determinePlanetType($i, $planetCount);  // LEGACY CALL
        $moonCount = $this->determineMoonCount($type);        // LEGACY CALL
        // ... creates planets with getPlanetSize($type)       // LEGACY CALL
    }
    // ...
}
```

**Optimized method: generateStarSystem()** (Lines 263-347)
- Uses `getPlanetTypeForStar()` instead of `determinePlanetType()`
- Uses `getMoonCount()` instead of `determineMoonCount()`
- Uses `getPlanetSizeIndex()` instead of `getPlanetSize()`
- Batches UUID generation for efficiency
- Uses `BulkInserter` for better performance

**Legacy method: determinePlanetType()** (Lines 577-603)
```php
private function determinePlanetType(int $orbitalIndex, int $totalPlanets): PointOfInterestType
{
    // Returns enum value directly, not int
    // Less efficient than optimized version
}
```

**Optimized method: getPlanetTypeForStar()** (Lines 372-431)
```php
private function getPlanetTypeForStar(int $orbitalIndex, int $totalPlanets, string $stellarClass): int
{
    // Returns int (raw enum value)
    // Considers stellar class for better variety
    // More sophisticated weighting
}
```

#### Impact
- Both versions are callable, creating confusion about which to use
- Memory waste storing duplicate logic
- Risk of regression if one is updated but not the other
- `generatePlanetarySystem()` is never called (checked with grep - no callers found)

#### Remediation
1. Search entire codebase for callers of `generatePlanetarySystem()`, `determinePlanetType()` (with PointOfInterestType return), `determineMoonCount()` (PointOfInterestType param), `getPlanetSize()` (PointOfInterestType param)
2. If none found: Delete all four legacy methods
3. If found: Migrate callers to optimized versions and delete legacy
4. Add PHPStan rule to prevent private method duplication

**Callers Found:**
```
generatePlanetarySystem: 0 callers found
determinePlanetType (PointOfInterestType version): 0 callers found
determineMoonCount (PointOfInterestType version): 0 callers found
getPlanetSize (PointOfInterestType version): 0 callers found
```

---

### 3. ARTISAN CALLS IN SERVICES & JOBS (Medium Priority)

#### Issue
Services and Jobs invoke console commands via `Artisan::call()` instead of calling services directly

**Files Using Artisan::call():**
1. `/home/mdhas/workspace/space-wars-3002/app/Jobs/CompleteGalaxyCreationJob.php` (8 calls)
2. `/home/mdhas/workspace/space-wars-3002/app/Jobs/CompleteTieredGalaxyCreationJob.php` (8 calls)
3. `/home/mdhas/workspace/space-wars-3002/app/Services/TieredGalaxyCreationService.php` (7 calls)
4. `/home/mdhas/workspace/space-wars-3002/app/Services/GalaxyGeneration/Generators/MirrorUniverseGenerator.php` (6 calls)
5. `/home/mdhas/workspace/space-wars-3002/app/Services/GalaxyGeneration/GalaxyGenerationOrchestrator.php` (3 calls)
6. `/home/mdhas/workspace/space-wars-3002/app/Services/GalaxyCreationService.php` (6 calls)
7. `/home/mdhas/workspace/space-wars-3002/app/Console/Commands/GalaxyInitialize.php` (6 calls)

#### Details

**Example 1: TieredGalaxyCreationService.php** (Lines 120, 341, 351, 361)
```php
// Line 120
Artisan::call('galaxy:create-mirror', ['galaxy' => $galaxy->id]);

// Lines 341, 351, 361 - Database seeding via Artisan
Artisan::call('db:seed', ['--class' => 'MineralSeeder']);
Artisan::call('db:seed', ['--class' => 'PlansSeeder']);
Artisan::call('db:seed', ['--class' => 'PirateCaptainSeeder']);
```

**Example 2: MirrorUniverseGenerator.php** (Lines 102-147)
```php
Artisan::call('galaxy:expand', ['galaxy' => $galaxy->id]);
Artisan::call('galaxy:designate-inhabited', ['galaxy' => $galaxy->id]);
Artisan::call('galaxy:generate-gates', ['galaxy' => $galaxy->id]);
Artisan::call('trading:generate-hubs', ['galaxy' => $galaxy->id]);
Artisan::call('galaxy:distribute-pirates', ['galaxy' => $galaxy->id]);
```

**Example 3: CompleteGalaxyCreationJob.php** (Lines 64, 73, 100, 110)
```php
Artisan::call('trading-hub:populate-inventory', ['galaxy' => $galaxy->uuid]);
Artisan::call('cartography:generate-shops', ['galaxy' => $galaxy->uuid, 'regenerate']);
Artisan::call('galaxy:distribute-pirates', ['galaxy' => $galaxy->id]);
Artisan::call('galaxy:create-mirror', ['galaxy' => $galaxy->id]);
```

#### Impact
- **Anti-pattern**: Violates Single Responsibility Principle
- **Testability**: Services become hard to test (require command setup)
- **Discoverability**: Business logic hidden inside commands, not in services
- **Coupling**: Tight coupling to console command interface
- **Error Handling**: Less control over error handling and return values
- **Performance**: Extra overhead of command parsing and routing

#### Root Cause
Original implementation extracted logic into commands, then later refactored to create services, but never updated the job/service callers to use services directly

#### Remediation
For each `Artisan::call()`:
1. Extract the command's business logic into a dedicated service
2. Replace `Artisan::call()` with direct service method call
3. Update the command to delegate to the service (not the reverse)
4. Add type hints and exception handling

**Priority Order:**
1. Database seeding (MineralSeeder, PlansSeeder, PirateCaptainSeeder) - Lines 341, 351, 361
2. Mirror universe creation - Line 120 (TieredGalaxyCreationService)
3. Pirate distribution - Lines 100, 110, 147
4. Trading hub operations - Lines 64, 73, 112, 127, 138
5. Galaxy expansion commands - Lines 102, 106, 117, 126

---

### 4. UNUSED IMPORTS (Low Priority)

#### Issue
Multiple controllers import resources/services they don't use

**File:** `/home/mdhas/workspace/space-wars-3002/app/Http/Controllers/Api/GalaxyController.php`

Unused imports detected:
- Line 7: `SystemNameGenerator` - imported but never used
- Line 10: `PlayerResource` - imported but never used
- Line 11: `Colony` model - imported but only used in one query without needing the import
- Line 12: `CombatSession` - imported but only used in one query

#### Details

**Line 7 - SystemNameGenerator**
```php
use App\Http\Controllers\Api\Builders\SystemNameGenerator;
// Imported but never referenced in GalaxyController
// Is used in GalaxyController.join() at line 645
```

**Actually Used - Ignore Above**
Line 645 calls: `SystemNameGenerator::ensureName($startingLocation);`

So SystemNameGenerator IS used. Let me verify remaining:
- `PlayerResource` - used at line 10 in the return (line 678)
- `Colony` - used at line 333 in statistics query
- `CombatSession` - used at line 347

**Correction: All imports in GalaxyController are used.**

---

## SUPPORTING FINDINGS

### 5. Player Creation Flow Diagram

```
Current State (CONFLICTING):

┌─────────────────────────────────────────────────────────────────┐
│                  PLAYER CREATION ENTRY POINTS                   │
└─────────────────────────────────────────────────────────────────┘

Route 1: POST /api/galaxies/{uuid}/join
└─> GalaxyController::join()
    ├─ Find spawn location (INLINE, lines 566-588)
    ├─ Create Player (INLINE, lines 594-603)
    ├─ Create PlayerShip (INLINE, lines 606-640)
    ├─ Generate system name (line 645)
    ├─ Create scan (lines 648-649)
    ├─ Discover lanes (lines 652-653)
    └─ Grant star charts (lines 656-668)

Route 2: POST /api/players
└─> PlayerController::store()
    ├─ Use PlayerSpawnService::findOptimalSpawnLocation()
    ├─ Create Player (INLINE, lines 84-93)
    ├─ Use PlayerSpawnService::ensureStarterShipAvailable()
    └─ TODO: Grant star charts (missing, line 98)

Route 3: CLI: php artisan player:init
└─> InitializePlayerCommand::handle()
    ├─ Use PlayerSpawnService::findOptimalSpawnLocation()
    ├─ Use PlayerSpawnService::getSpawnLocationReport()
    ├─ Create Player (INLINE, lines 88-97)
    ├─ Use PlayerSpawnService::ensureStarterShipAvailable()
    └─ Use StarChartService::grantStartingCharts()

Desired State (UNIFIED):

┌─────────────────────────────────────────────────────────────────┐
│         PlayerSpawnService::createPlayerAtSpawnLocation()       │
├─────────────────────────────────────────────────────────────────┤
│ 1. Find optimal spawn location                                  │
│ 2. Create player with starting credits                          │
│ 3. Ensure starter ship available                                │
│ 4. Generate system name                                         │
│ 5. Create minimum scan (level 1)                                │
│ 6. Discover outgoing lanes                                      │
│ 7. Grant spawn location star chart (free)                       │
│ 8. Grant starting charts for nearby systems                     │
│ Returns: Player instance                                        │
└─────────────────────────────────────────────────────────────────┘
     ↑
     └─ Used by: GalaxyController, PlayerController, InitializePlayerCommand
```

---

## MINOR ISSUES

### 6. PlayerController.store() Incomplete Implementation

**File:** `/home/mdhas/workspace/space-wars-3002/app/Http/Controllers/Api/PlayerController.php`
**Lines:** 36-115

Issues:
1. Line 98: TODO comment indicates missing star chart grant logic
2. No system name generation (unlike GalaxyController)
3. No scan/lane knowledge discovery (unlike GalaxyController)
4. Uses Player::findByUuid() at line 69 - inefficient after creation, could use $player directly

**TODO Comment (Line 98):**
```php
// TODO: Give player starter star charts (3 nearest systems)
```

This feature exists in GalaxyController (lines 667-668) and InitializePlayerCommand (line 104) but not here.

---

### 7. Database Seeding in Request Handlers (Anti-pattern)

**File:** `/home/mdhas/workspace/space-wars-3002/app/Http/Controllers/Api/GalaxyController.php`
**Lines:** 607-615

```php
$starterShip = Ship::where('class', 'starter')->first();
if (! $starterShip) {
    // TODO: TECH DEBT - Remove runtime seeder fallback
    // Issue: Running seeders in request handlers is an anti-pattern
    // Current: Seeds ships if missing (unsafe for production)
    // Desired: Return 500 error if starter ship missing
    (new \Database\Seeders\ShipTypesSeeder)->run();
    $starterShip = Ship::where('class', 'starter')->first();
}
```

This is a runtime fallback seeder and should be removed. The issue is already documented in the code itself.

---

## SUMMARY TABLE

| Issue | Severity | Files | Type | Action |
|-------|----------|-------|------|--------|
| Player creation duplication | **CRITICAL** | GalaxyController, PlayerController, InitializePlayerCommand | Duplication | Consolidate to PlayerSpawnService |
| PlanetarySystemGenerator legacy methods | **MEDIUM** | PlanetarySystemGenerator.php | Dead Code | Delete 4 private methods |
| Artisan::call() in services | **MEDIUM** | 7 files (services/jobs) | Anti-pattern | Replace with service calls |
| Runtime seeder fallback | **LOW** | GalaxyController.php:607 | Anti-pattern | Remove, enforce seeding at deploy |
| PlayerController incomplete | **LOW** | PlayerController.php:98 | Missing Feature | Implement via unified service |

---

## RECOMMENDED IMPLEMENTATION ORDER

1. **Phase 1 (Critical):** Consolidate player creation
   - Create `PlayerSpawnService::createPlayerAtSpawnLocation()`
   - Update GalaxyController.join()
   - Update PlayerController.store()
   - Verify InitializePlayerCommand still works

2. **Phase 2 (Medium):** Remove legacy PlanetarySystemGenerator methods
   - Delete `generatePlanetarySystem()`, `determinePlanetType()`, `determineMoonCount()`, `getPlanetSize()`
   - Run full test suite
   - Verify no callers

3. **Phase 3 (Medium):** Replace Artisan::call() with services
   - Create dedicated services for seeding, pirate distribution, etc.
   - Update all 7 files
   - Remove Artisan facade imports

4. **Phase 4 (Low):** Clean up remaining issues
   - Remove runtime seeder fallback
   - Complete PlayerController features
   - Add integration tests for unified flows

---

## VERIFICATION COMMANDS

```bash
# Check for all Artisan::call() instances
grep -r "Artisan::call" app --include="*.php" -n

# Check for all uses of legacy PlanetarySystemGenerator methods
grep -r "determinePlanetType\|determineMoonCount\|getPlanetSize\|generatePlanetarySystem" app --include="*.php" | grep -v "private function"

# Find unused imports in a file
php artisan make:command FindUnusedImports
```

---

## Documentation References

- **CLAUDE.md** - Architecture and service patterns
- **app/MEMORY.md** - Recent refactoring history (SOLID/DRY phases)
- **Trait: HasUuid** - Used by 23+ models
- **Pattern: BaseApiController** - Common API response helpers
