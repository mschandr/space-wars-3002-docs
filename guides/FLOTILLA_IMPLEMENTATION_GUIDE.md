# Flotilla & Fleet Mechanics - Technical Implementation Guide

**Source Document**: `/docs/design/flotilla.md` (v1.0, 2026-02-21)
**Status**: Design Ready for Implementation
**Complexity**: High (combat, cargo, fuel systems interaction)

---

## Quick Summary

**Flotilla**: A temporary formation of 2-4 ships owned by the same player that move and operate together as a unit.

**Fleet**: All ships a player owns (existing concept, no changes needed).

**Key Mechanics**:
- Slowest ship determines movement speed for entire flotilla
- Fuel penalty: +10% per additional ship (2 ships = 1.1×, 3 ships = 1.2×, 4 ships = 1.3×)
- All ships attack/defend together in combat
- Pirates target weakest ship first (creates natural escort dynamic)
- Salvage is XOR choice: recover cargo (70%) OR components (with escalating loss) — not both

---

## Implementation Phases

### **Phase 1: Database & Models** (Foundation)
- [x] Design database schema
- [ ] Create migration: `flotillas` table
- [ ] Create migration: modify `player_ships` add `flotilla_id`
- [ ] Create `Flotilla` model with relationships
- [ ] Extend `PlayerShip` model with flotilla relationship
- [ ] Add configuration to `config/game_config.php`

### **Phase 2: Core Services** (Business Logic)
- [ ] `FlotillaService` - CRUD operations
- [ ] `FlotillaMovementService` - Movement with fuel penalties
- [ ] `FlotillaCombatService` - Combat calculations
- [ ] `FlotillaSalvageService` - Post-battle recovery logic

### **Phase 3: API Controller & Routes** (REST Endpoints)
- [ ] Create `FlotillaController` with 6 endpoints
- [ ] Register routes in `routes/api.php`

### **Phase 4: Combat Integration** (Existing System)
- [ ] Modify `CombatService` to handle flotillas
- [ ] Integrate pirate targeting logic
- [ ] Implement ship destruction mid-combat

### **Phase 5: Testing** (Quality Assurance)
- [ ] Unit tests for services
- [ ] Feature tests for API endpoints
- [ ] Combat simulation tests

### **Phase 6: Documentation** (Developer Guides)
- [ ] Implementation summary
- [ ] API reference
- [ ] Testing manual

---

## Database Schema

### New Table: `flotillas`

```php
Schema::create('flotillas', function (Blueprint $table) {
    $table->id();
    $table->uuid()->unique();
    $table->foreignId('player_id')->constrained('players')->onDelete('cascade');
    $table->string('name')->nullable();
    $table->foreignId('flagship_ship_id')->constrained('player_ships')->onDelete('cascade');
    $table->timestamps();

    // Indexes for common queries
    $table->index(['player_id']);
    $table->index(['flagship_ship_id']);
});
```

### Modified Table: `player_ships`

```php
// In migration, add to existing player_ships table:
Schema::table('player_ships', function (Blueprint $table) {
    $table->foreignId('flotilla_id')
        ->nullable()
        ->constrained('flotillas')
        ->onDelete('set null');  // If flotilla dissolves, ship becomes independent

    $table->index(['flotilla_id']);
});
```

---

## Configuration

Add to `config/game_config.php`:

```php
'flotilla' => [
    'max_ships' => 4,                        // Max ships per flotilla
    'fuel_penalty_per_ship' => 0.10,         // +10% per additional ship
    'form_turn_cost' => 1,                   // Future: turns to form flotilla
    'cargo_recovery_rate' => 0.70,           // 70% cargo recoverable on win
    'pirate_loot_recovery_rate' => 0.50,     // 50% of pirate cargo
    'component_recovery_loss' => [           // Escalating loss per component
        1 => 0.10,  // 1st: 10% loss
        2 => 0.20,  // 2nd: 20% loss
        3 => 0.30,  // 3rd: 30% loss
        // ... pattern continues
    ],
],
```

---

## Models

### `Flotilla` Model

```php
class Flotilla extends Model
{
    use HasUuid;

    protected $fillable = ['player_id', 'name', 'flagship_ship_id'];

    // Relationships
    public function player() { return $this->belongsTo(Player::class); }
    public function flagship() { return $this->belongsTo(PlayerShip::class, 'flagship_ship_id'); }
    public function ships() { return $this->hasMany(PlayerShip::class, 'flotilla_id'); }

    // Scopes
    public function scopeActive() { return $this->whereNotNull('id'); }

    // Helper methods
    public function shipCount(): int { return $this->ships()->count(); }
    public function isFull(): bool { return $this->shipCount() >= config('game_config.flotilla.max_ships'); }
    public function canAddShip(): bool { return !$this->isFull(); }
    public function totalHull(): int { return $this->ships()->sum('hull'); }
    public function totalCargoHold(): int { return $this->ships()->sum('cargo_hold'); }
    public function lowestHull(): PlayerShip { return $this->ships()->orderBy('hull')->first(); }
    public function slowestShip(): PlayerShip { return $this->ships()->orderBy('warp_drive')->first(); }

    // Combat helpers
    public function getTotalWeaponDamage(): int { /* calculated from all ships */ }
    public function getWeakestShip(): PlayerShip { return $this->lowestHull(); }
    public function areAllShipsAtSamePoi(): bool { /* all ship current_poi_id match */ }
}
```

### `PlayerShip` Model Extension

```php
// Add to existing PlayerShip model:

public function flotilla() { return $this->belongsTo(Flotilla::class); }
public function isInFlotilla(): bool { return !is_null($this->flotilla_id); }
public function isFlagship(): bool { return $this->flotilla?->flagship_ship_id === $this->id; }
```

---

## Services

### `FlotillaService.php` (~200 LOC)

```php
class FlotillaService
{
    public function createFlotilla(Player $player, PlayerShip $flagship, string $name = null): Flotilla
        // Validates: flagship ship exists, player owns ship
        // Creates: Flotilla with flagship
        // Atomic: transaction wrapped

    public function addShipToFlotilla(Flotilla $flotilla, PlayerShip $ship): void
        // Validates: ship not already in flotilla
        // Validates: flotilla not full (max 4 ships)
        // Validates: all ships at same POI
        // Updates: ship->flotilla_id = flotilla->id
        // Atomic: transaction wrapped

    public function removeShipFromFlotilla(Flotilla $flotilla, PlayerShip $ship): void
        // Validates: ship belongs to this flotilla
        // Validates: cannot remove flagship (must change flagship first)
        // Updates: ship->flotilla_id = null
        // Atomic: transaction wrapped

    public function setFlagship(Flotilla $flotilla, PlayerShip $newFlagship): void
        // Validates: new flagship in this flotilla
        // Updates: flotilla->flagship_ship_id
        // Atomic: transaction wrapped

    public function dissolveFlotilla(Flotilla $flotilla): void
        // Updates: all ships set flotilla_id = null
        // Deletes: flotilla record
        // Atomic: transaction wrapped

    public function getFlotillaStatus(Flotilla $flotilla): array
        // Returns: {name, flagship, ships: [...], fuel_status, cargo_status}
}
```

### `FlotillaMovementService.php` (~150 LOC)

```php
class FlotillaMovementService
{
    public function canMoveFlotilla(Flotilla $flotilla): array
        // Validates: all ships have sufficient fuel
        // Returns: ['can_move' => bool, 'reason' => string]

    public function moveFlotilla(Flotilla $flotilla, PointOfInterest $destination): void
        // Validates: can move (fuel check)
        // For each ship:
        //   - Calculate fuel cost with penalty
        //   - Deduct fuel
        //   - Update current_poi_id
        // Atomic: transaction wraps all ships

    public function calculateFuelCost(Flotilla $flotilla, int $distance): array
        // Returns: per-ship fuel costs accounting for:
        //   - Individual warp drive efficiency
        //   - Formation penalty (+10% per extra ship)
        // Returns: {'ship_id' => cost, ...}

    private function getFormationFuelPenalty(int $shipCount): float
        // 1 ship = 1.0× (no penalty)
        // 2 ships = 1.1×
        // 3 ships = 1.2×
        // 4 ships = 1.3×
}
```

### `FlotillaCombatService.php` (~200 LOC)

```php
class FlotillaCombatService
{
    public function getTotalFlotillaWeaponDamage(Flotilla $flotilla): int
        // Sums damage from all ships
        // Each ship: (base_damage * weapon_level + random_variance)

    public function selectPirateFocusTarget(Flotilla $flotilla): PlayerShip
        // Pirates target ship with LOWEST hull first
        // Returns: weakest ship in flotilla

    public function applyDamageToFlotilla(Flotilla $flotilla, int $damageAmount): array
        // Applies damage to pirate-selected target ship
        // If ship destroyed: remove from flotilla, redistribute damage?
        // Returns: {damage_applied, ship_destroyed: bool, flotilla_destroyed: bool}

    public function handleShipDestructionInCombat(Flotilla $flotilla, PlayerShip $destroyedShip): void
        // If destroyed ship is flagship: promote next largest ship to flagship
        // Remove ship from flotilla
        // Continue combat with remaining ships
}
```

### `FlotillaSalvageService.php` (~200 LOC)

```php
class FlotillaSalvageService
{
    public function getSalvageOptions(Flotilla $flotilla, array $destroyedShips): array
        // Returns: {cargo_available: int, components_available: [...]}
        // Calculates what can be salvaged before player chooses

    public function recoverCargo(Flotilla $flotilla, array $destroyedShips): void
        // Recovers 70% of destroyed ships' cargo
        // Distributes to surviving ships by available hold space
        // Excess cargo is lost (insufficient space)
        // Atomic: transaction wrapped

    public function recoverComponents(Flotilla $flotilla, array $destroyedShips): array
        // Recovers components with escalating loss:
        //   1st: 10% loss (level 10 → 9)
        //   2nd: 20% loss (level 8 → 6.4)
        //   3rd: 30% loss, etc.
        // Returns: recovered components with adjusted levels
        // Atomic: transaction wrapped

    public function recoverPirateLoot(array $pirateShips): array
        // Always available (separate from XOR choice)
        // Random percentage of pirate cargo (50%+ loss on components)
        // Added to surviving ships
}
```

---

## API Endpoints

### `FlotillaController.php` (~300 LOC)

```
6 REST Endpoints:

1. POST /api/players/{uuid}/flotilla
   - Create flotilla
   - Request: { "name": "...", "flagship_ship_id": uuid }
   - Validates: player owns ship, ship at valid location
   - Returns: 201 with created flotilla, or 422 with error

2. GET /api/players/{uuid}/flotilla
   - Get flotilla status
   - Returns: { flotilla, ships, fuel_status, cargo_status }
   - Returns: 404 if player has no flotilla

3. POST /api/players/{uuid}/flotilla/add-ship
   - Add ship to flotilla
   - Request: { "ship_id": uuid }
   - Validates: flotilla not full, ship at same POI
   - Returns: 200 with updated flotilla, or 422 with error

4. POST /api/players/{uuid}/flotilla/remove-ship
   - Remove ship from flotilla
   - Request: { "ship_id": uuid }
   - Validates: ship not flagship
   - Returns: 200 with updated flotilla, or 422 with error

5. POST /api/players/{uuid}/flotilla/set-flagship
   - Change flagship designation
   - Request: { "ship_id": uuid }
   - Validates: ship in this flotilla
   - Returns: 200 with updated flotilla, or 422 with error

6. DELETE /api/players/{uuid}/flotilla
   - Dissolve flotilla
   - Returns: 200 with confirmation, ships become independent
   - Cascades: all ships set flotilla_id = null
```

---

## Combat Integration Points

### Existing `CombatService` Modifications

```php
class CombatService
{
    // NEW: Handle flotillas in combat
    public function resolveCombatFlotilla(Flotilla $flotilla, array $pirates): CombatResult
        // Uses FlotillaCombatService for:
        //   - Total damage calculation (all ships)
        //   - Pirate targeting (lowest hull)
        //   - Mid-combat ship destruction
        // Existing result handling (XP, credits, pirate loot)

    // MODIFIED: Existing combat initialization
    // If player has flotilla, pass to flotilla combat instead of single-ship combat
}
```

### Movement System Integration

```php
// In existing movement/warp system:
if ($ship->isInFlotilla()) {
    // Use FlotillaMovementService for validation & execution
    return $this->flotilaMovementService->moveFlotilla(
        $ship->flotilla,
        $destination
    );
} else {
    // Use existing single-ship movement
    return $this->shipMovementService->move($ship, $destination);
}
```

---

## Business Logic Rules

### Formation Rules
- **Location**: All ships must be at same POI to form flotilla
- **Max size**: 4 ships per flotilla
- **Flagship**: Required, gates trading/mining actions
- **Removal**: Cannot remove flagship without changing flagship first

### Movement Rules
- **Slowest ship sets pace**: Flotilla speed = slowest ship's warp drive
- **Individual fuel burn**: Each ship burns fuel independently
- **Formation penalty**: +10% fuel per additional ship
  ```
  Total cost = (ship1_cost × 1.1) + (ship2_cost × 1.1) + ...
  ```
- **Atomic movement**: All ships move or none move (no partial moves)
- **Fuel requirement**: Every ship must have sufficient fuel before move executes

### Cargo Rules
- **Per-ship isolation**: Cargo stays on its ship always
- **No transfer during flotilla**: Cannot move cargo between ships while in formation
- **On dissolution**: Cargo stays on each ship; default overflow goes to ship with most space

### Combat Rules
- **Combined attack**: All ships fire together, total damage = sum
- **Pirate targeting**: Pirates attack weakest ship first (lowest hull)
- **Mid-combat destruction**: If ship dies, remove from formation, continue with remaining ships
- **Flagship loss**: If flagship destroyed, next largest ship becomes new flagship

### Salvage Rules
- **XOR choice**: Recover cargo (70%) OR recover components (with loss) — not both
- **Cargo distribution**: By surviving ships' available hold space
- **Component loss escalation**:
  ```
  1st component: 10% loss
  2nd component: 20% loss
  3rd component: 30% loss
  ... continues per component recovered
  ```
- **Pirate loot**: Always available (separate), 50%+ loss on components, random % of cargo

### Flagship Rules
- **Actions gate**: Only flagship can trade, mine, dock
- **Combat role**: Target for priority selection (doesn't make flagship more durable)
- **Succession**: If destroyed, next largest ship by hull becomes flagship
- **Change requirement**: To trade with different ship, change flagship or remove from flotilla

---

## Error Handling

### Validation Errors (422)
- Ship not at same POI
- Flotilla full (max 4 ships)
- Ship already in flotilla
- Insufficient fuel
- Cannot remove flagship
- Flotilla doesn't exist

### Not Found Errors (404)
- Player has no flotilla
- Ship doesn't exist
- Flotilla doesn't exist

### State Errors (409)
- Cannot move while damaged/refueling
- Cannot dock while in combat

---

## Testing Strategy

### Unit Tests
- **FlotillaServiceTest** (8 tests)
  - Create, add ship, remove ship, set flagship, dissolve
  - Validation logic (full, location, ownership)

- **FlotillaMovementServiceTest** (6 tests)
  - Fuel calculation with formation penalties
  - Can move validation
  - Atomic movement

- **FlotillaCombatServiceTest** (6 tests)
  - Total damage calculation
  - Target selection (weakest ship)
  - Mid-combat destruction

- **FlotillaSalvageServiceTest** (8 tests)
  - Cargo recovery (70%)
  - Component recovery (escalating loss)
  - Pirate loot distribution

### Feature Tests
- **FlotillaControllerTest** (14 tests)
  - All 6 endpoints
  - Validation, error responses
  - Full workflow (create → add ships → dissolve)

### Integration Tests
- **CombatIntegrationTest** (10 tests)
  - Flotilla vs pirates
  - Mid-combat destruction
  - Post-battle salvage flow

---

## Implementation Checklist

- [ ] **Phase 1: Database**
  - [ ] Migration: create `flotillas` table
  - [ ] Migration: modify `player_ships` add `flotilla_id`
  - [ ] Flotilla model created
  - [ ] PlayerShip model extended
  - [ ] Config added

- [ ] **Phase 2: Services**
  - [ ] FlotillaService (~200 LOC)
  - [ ] FlotillaMovementService (~150 LOC)
  - [ ] FlotillaCombatService (~200 LOC)
  - [ ] FlotillaSalvageService (~200 LOC)
  - [ ] Service provider bindings

- [ ] **Phase 3: API**
  - [ ] FlotillaController (~300 LOC)
  - [ ] Routes registered (6 endpoints)
  - [ ] Request validation
  - [ ] Response formatting

- [ ] **Phase 4: Combat Integration**
  - [ ] CombatService modified for flotillas
  - [ ] Pirate targeting logic
  - [ ] Mid-combat destruction handling

- [ ] **Phase 5: Testing**
  - [ ] 28 unit tests (4 test files)
  - [ ] 14 feature tests (1 test file)
  - [ ] 10 integration tests (1 test file)
  - [ ] Total: 52 tests

- [ ] **Phase 6: Documentation**
  - [ ] Implementation summary
  - [ ] API reference
  - [ ] Testing manual

---

## Future Enhancements (Not V1)

- **NPC Escort Hire**: Hire mercenary ships at trading hubs with trustability ratings
- **Colony Ship Escort**: Flotillas will be required to safely transport colony ships
- **Inter-ship Cargo Transfer**: Future feature to move cargo between docked ships
- **Flotilla Combat Orders**: Allow players to set formation patterns (defensive, aggressive, balanced)
- **Multi-player Flotillas**: Cooperative fleet management (high complexity, deferred)

---

## Notes

- **Smart Pirate Targeting**: Pirates in this system are not foolish — they target based on cargo value and ship hull. A well-composed escort (tough combat ships + fragile cargo ships) will naturally protect the vulnerable vessels.

- **Flagship as Gateway**: Only the flagship can interact with the world (trade, mine, dock). This creates interesting logistics puzzles — do you keep a cargo ship as flagship (vulnerable but flexible) or a combat ship (safe but limited)?

- **Atomic Operations**: All flotilla operations (movement, combat, salvage) are atomic transactions. A ship cannot partially move, a battle cannot partially resolve, and salvage choices commit immediately.

- **No Shared Cargo Pool**: Unlike some games, cargo stays on its ship. This forces players to think about cargo distribution and load balancing when composing their flotilla.

---

## Code Metrics (Estimated)

| Component | LOC | Files |
|-----------|-----|-------|
| Migrations | 40 | 1 |
| Models | 80 | 2 |
| Services | 750 | 4 |
| Controller | 300 | 1 |
| Tests | 900 | 6 |
| Configuration | 20 | 1 |
| **Total** | **2,090** | **15** |

---

**Status**: Ready for Phase 1 implementation (database & models)
