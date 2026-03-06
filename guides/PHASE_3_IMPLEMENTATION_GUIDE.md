# Phase 3: Construction System & Trade Locking — Implementation Guide

**Status:** ✅ Complete
**Date Completed:** 2026-03-04
**Commit:** 0343d70
**Version:** 2026.03.04.002

---

## Overview

Phase 3 closes two critical gaps in the economy overhaul:

1. **Part A — Trade Locking:** Fixes TOCTOU (Time-of-Check to Time-of-Use) race condition in concurrent trading
2. **Part B — Construction System:** Adds blueprint-based item crafting with ledger-backed input consumption and asynchronous job maturation

This guide explains the implementation, design decisions, and integration with the broader economy system.

---

## Part A: Trade Locking

### Problem: The Race Condition

Before Phase 3, `TradingService::buyMineral()` and `sellMineral()` had this flow:

```
Thread A: READ inventory (quantity=100)
Thread B: READ inventory (quantity=100)
Thread A: Validate: 100 >= 50? ✓ → DECREMENT to 50
Thread B: Validate: 100 >= 50? ✓ → DECREMENT to 50
Result: Net effect is -50 (should be -100)
```

Two concurrent requests could both read the same quantity, both pass validation, and both decrement—resulting in overselling inventory.

### Solution: Lock Inside Transaction

Phase 3 adds `lockForUpdate()` **inside** the transaction:

```php
return DB::transaction(function () use (...) {
    // Re-fetch with lock as first operation
    $inventory = TradingHubInventory::where('id', $inventory->id)
        ->lockForUpdate()
        ->firstOrFail();

    // Re-validate with locked copy
    if (! $inventory->hasStock($quantity)) {
        throw new \Exception('Insufficient stock available');
    }

    // Capture price from locked inventory
    $unitPrice = $inventory->sell_price;

    // Proceed with trade...
});
```

**Why inside the transaction?**
- Lock is released when transaction commits (not earlier)
- Other threads wait for the lock, preventing concurrent reads
- Re-validation ensures no race condition possible

**Files Modified:**
- `app/Services/TradingService.php` — Both `buyMineral()` and `sellMineral()`

### Verification

```bash
# Syntax check
php -l app/Services/TradingService.php

# Run existing trading tests (if available)
php artisan test tests/Unit/Services/TradingServiceTest.php
```

---

## Part B: Construction System

### Architecture

Construction is a 3-layer system:

```
┌─────────────────────────────────────┐
│   ConstructionController (API)      │ ← HTTP endpoints
├─────────────────────────────────────┤
│ ConstructionService (Business Logic)│ ← Job creation & maturation
├─────────────────────────────────────┤
│ LedgerService + InventoryService    │ ← Ledger-backed consumption
├─────────────────────────────────────┤
│   ConstructionJob (Model)           │ ← Status & timing tracking
├─────────────────────────────────────┤
│ Economy Tick (Async Processing)     │ ← Job maturation loop
└─────────────────────────────────────┘
```

### Key Components

#### 1. ConstructionJob Model
**File:** `app/Models/ConstructionJob.php`

Represents a construction job with:
- **Status**: PENDING → COMPLETE (or FAILED)
- **Timing**: started_at, completes_at, completed_at
- **Inputs**: inputs_consumed JSON (snapshot for audit trail)
- **Output**: output_item_code (denormalized from blueprint)

**Relationships:**
- `galaxy()` — The galaxy where hub is located
- `tradingHub()` — Where construction occurs
- `player()` — Who initiated the build
- `blueprint()` — What is being built

**Scopes:**
- `scopePending()` — Only PENDING jobs
- `scopeComplete()` — Only COMPLETE jobs
- `scopeByGalaxy($galaxy)` — Filter by galaxy

**Helpers:**
- `isPending()`, `isComplete()` — Status checks
- `isMatured()` — `completes_at <= now()`

#### 2. ConstructionService
**File:** `app/Services/Economy/ConstructionService.php`

**Method: build()**
```php
public function build(
    Player $player,
    PlayerShip $ship,
    TradingHub $hub,
    Blueprint $blueprint,
    int $quantity = 1
): array // {success, message, job_uuid, completes_at, shortages}
```

**Logic Flow:**
1. Verify ship is at hub (`ship->current_poi_id === hub->pointOfInterest->id`)
2. Resolve galaxy from `hub->pointOfInterest->galaxy`
3. Load blueprint inputs with commodities
4. **Lock** all input inventory rows in `mineral_id ASC` order (deadlock prevention)
5. Validate sufficient stock for each input
   - Collect ALL shortfalls (not just first)
   - Return error array with detailed shortages
6. If any shortfall → transaction rollback, return error
7. Record construction in ledger:
   - Call `LedgerService::recordConstruction()`
   - Returns array of negative-delta entries
8. Apply ledger entries atomically:
   - Call `InventoryService::applyLedgerBatch($entries)`
   - Each input qty deducted from hub inventory
9. Create ConstructionJob:
   - `completes_at = now()->addSeconds($blueprint->build_time_ticks)`
   - `inputs_consumed` = JSON snapshot of [{commodity_id, qty_each, total_qty}, ...]
10. Return success with job UUID and completion time

**Error Cases:**
- Ship not at hub → BEFORE transaction
- Insufficient stock → IN transaction (fully rolled back)
- Any exception → transaction rolls back, logged as error

**Method: completeJob()**
```php
public function completeJob(ConstructionJob $job): void
```

**Phase 3 Stub:**
- Update status → COMPLETE
- Set completed_at → now()
- Log "Item ready: {output_item_code}"
- **No item delivery** (Phase N+)

**Future Phases:**
```php
// Phase N+ will replace stub with:
$job->status = 'COMPLETE';
$job->completed_at = now();
$job->save();

ItemDeliveryService::deliver($job);  // Create player item
$job->player->notify(new ConstructionComplete($job));
```

#### 3. ConstructionTickService
**File:** `app/Services/Economy/ConstructionTickService.php`

**Method: processTick()**
```php
public function processTick(
    ?Galaxy $galaxy = null,
    bool $dryRun = false
): array // {checked, completed, errors}
```

**Logic:**
1. Query matured jobs: `ConstructionJob::pending()` where `completes_at <= now()`
2. Filter by galaxy if specified
3. For each job:
   - Try: call `ConstructionService::completeJob()`
   - Catch: record error with job UUID + message
   - Increment `completed` counter
4. If dry-run: count jobs but don't write
5. Return metrics: {checked, completed, errors[]}

**Integration with EconomyTickCommand:**
```php
// Step 1: Mining
$miningResults = $this->miningService->processTick($galaxy, $dryRun);

// Step 2: Construction (NEW)
$constructionResults = $this->constructionService->processTick($galaxy, $dryRun);

// Step 3: Shock decay
$shockResults = $this->shockService->processTick($galaxy, $dryRun);

// Step 4: Stats refresh
$statsResults = $this->statsService->recomputeGalaxyStats($galaxy);
```

#### 4. ConstructionController
**File:** `app/Http/Controllers/Api/ConstructionController.php`

Extends `BaseApiController`.

**Endpoint: listAvailableBlueprints()**
```
GET /api/trading-hubs/{uuid}/blueprints
```

Returns all blueprints with:
- Full input requirements
- `can_build: bool` (enough stock at hub)
- `shortages: []` (missing inputs, if any)

**Endpoint: startConstruction()**
```
POST /api/trading-hubs/{uuid}/build
{
  "player_uuid": "...",
  "ship_uuid": "...",
  "blueprint_uuid": "...",
  "quantity": 1
}
```

Calls `ConstructionService::build()`, returns job details or detailed error with shortages.

**Endpoint: listJobs()**
```
GET /api/players/{uuid}/construction-jobs?status=PENDING&page=1
```

Paginated job list with:
- All job details (uuid, blueprint, hub, timing, status)
- `time_remaining_seconds` (calculated at request time)
- Filterable by status (PENDING/COMPLETE/FAILED)

### Input Locking Order (Deadlock Prevention)

When multiple concurrent builds happen at the same hub, they might try to lock the same inputs. Without ordering, this causes deadlock:

```
Build A: Locks Iron, waits for Platinum
Build B: Locks Platinum, waits for Iron
→ Deadlock!
```

**Solution: Lock in ascending `mineral_id` order**

```php
$inventoryRows = [];
foreach ($inputs->sortBy('commodity_id') as $input) {
    $inventory = TradingHubInventory::where(...)
        ->where('mineral_id', $input->commodity_id)
        ->lockForUpdate()
        ->first();
    // ...
}
```

Now:
```
Build A: Locks Iron (1), then Platinum (5) ✓
Build B: Locks Iron (1), then Platinum (5) ✓
→ No deadlock!
```

### Database Schema

**Table: construction_jobs**

| Column | Type | Notes |
|--------|------|-------|
| id | BIGINT PK | Auto-increment |
| uuid | CHAR(36) UNIQUE | UUID v4 |
| galaxy_id | BIGINT FK | Cascades on delete |
| trading_hub_id | BIGINT FK | Cascades on delete |
| player_id | BIGINT FK | Cascades on delete |
| blueprint_id | BIGINT FK | Restricts on delete (preserve history) |
| quantity | INT | Number of units being built |
| status | ENUM(PENDING, COMPLETE, FAILED) | Default: PENDING |
| inputs_consumed | JSON | [{commodity_id, qty_each, total_qty}, ...] |
| output_item_code | VARCHAR(255) | Denormalized from blueprint |
| started_at | DATETIME | Creation time |
| completes_at | DATETIME | Maturation time |
| completed_at | DATETIME nullable | When job was marked complete |
| metadata | JSON nullable | Future use (extended info) |
| created_at | TIMESTAMP | Laravel timestamp |
| updated_at | TIMESTAMP | Laravel timestamp |

**Indexes:**
- `[status, completes_at]` — Fast tick queries
- `[player_id, status]` — Fast player job lookups
- `[galaxy_id]` — Fast galaxy queries

### Migration

**File:** `database/migrations/2026_03_04_000111_create_construction_jobs_table.php`

Runs successfully:
```bash
$ php artisan migrate
2026_03_04_000111_create_construction_jobs_table ... 89.48ms DONE
```

---

## Integration Points

### 1. Player Model
**File:** `app/Models/Player.php`

Added relationship:
```php
public function constructionJobs(): HasMany
{
    return $this->hasMany(ConstructionJob::class);
}
```

Allows:
```php
$player->constructionJobs()->where('status', 'PENDING')->count()
```

### 2. Ledger System
**File:** `app/Services/Economy/LedgerService.php`

Existing method used:
```php
public function recordConstruction(
    Galaxy $galaxy,
    TradingHub $hub,
    Blueprint $blueprint,
    int $actorId,
    ?string $correlationId = null,
    array $metadata = []
): array // Returns array of CommodityLedgerEntry objects
```

Creates one ledger entry per input commodity with:
- `reason_code = ReasonCode::CONSTRUCTION`
- `qty_delta = -qty_required` (negative = sink)
- `actor_type = ActorType::PLAYER`
- `actor_id = $player->id`
- Shared `correlation_id` linking all inputs

### 3. Inventory System
**File:** `app/Services/Economy/InventoryService.php`

Existing method used:
```php
public function applyLedgerBatch(array $entries): void
```

Applies all ledger entries in a single transaction with:
- Row locks to prevent concurrent mutation
- Conservation checks (can't go negative)
- Snapshot timestamp for audit trail

### 4. Economy Tick
**File:** `app/Console/Commands/EconomyTickCommand.php`

Updated to include construction step:
1. Mining extraction (existing)
2. **Construction job maturation (NEW)**
3. Shock decay (existing)
4. Stats cache refresh (existing)

Output now includes:
```
Construction Jobs:
  Checked: 5
  Completed: 3
```

### 5. Routes
**File:** `routes/api.php`

Three new routes:
```php
Route::get('trading-hubs/{uuid}/blueprints', [ConstructionController::class, 'listAvailableBlueprints']);
Route::post('trading-hubs/{uuid}/build', [ConstructionController::class, 'startConstruction']);
Route::get('players/{uuid}/construction-jobs', [ConstructionController::class, 'listJobs']);
```

Verified registered:
```bash
$ php artisan route:list | grep construction
GET|HEAD  api/trading-hubs/{uuid}/blueprints
POST      api/trading-hubs/{uuid}/build
GET|HEAD  api/players/{uuid}/construction-jobs
```

---

## Design Decisions

### Decision 1: Phase 3 Output Deferral

**Decision:** Item delivery is deferred to Phase N+; Phase 3 = consumption + job tracking only

**Rationale:**
- `output_item_code` is a bare string; no `player_items` table exists
- Full item system (storage, pickup deadlines, variants) is a separate design effort
- Deferring prevents scope explosion and allows parallel work on item storage layer
- Ledger-based consumption is the critical piece; delivery is orthogonal

**Consequences:**
- `completeJob()` is a stub (logs only)
- Players see "construction ready" but can't pick up items yet
- Migration path: Replace stub with ItemDeliveryService call (no schema changes needed)

**See:** [ADR 0001: Construction Output Deferral](../ADR/0001-construction-output-deferral.md)

### Decision 2: Lock Ordering for Deadlock Prevention

**Decision:** Lock all input inventory rows in ascending `mineral_id` order

**Rationale:**
- Prevents circular wait conditions in concurrent builds
- Simple to implement and understand
- Deterministic ordering ensures consistency

**Example:**
```
// Correct (ascending order)
Build A locks: Iron (ID 1) → Copper (ID 2) → Platinum (ID 5) ✓
Build B locks: Iron (ID 1) → Copper (ID 2) → Platinum (ID 5) ✓

// Dangerous (random order)
Build A locks: Copper (2) → Iron (1) → Platinum (5)
Build B locks: Iron (1) → Platinum (5) → Copper (2)
→ Can cause deadlock (A waits for 1, B waits for 2)
```

### Decision 3: Comprehensive Shortfall Reporting

**Decision:** Collect ALL input shortfalls before rejecting, not just the first

**Rationale:**
- Player gets complete picture of what's missing
- Reduces friction: one request shows all gaps instead of having to iterate

**Example:**
```json
// Phase 3: All shortfalls reported
{
  "shortages": [
    { "commodity": "Iron", "shortfall": 20 },
    { "commodity": "Copper", "shortfall": 45 },
    { "commodity": "Platinum", "shortfall": 15 }
  ]
}
// Player knows all three are missing, can restock once
```

### Decision 4: Atomic Ledger Application

**Decision:** Use `InventoryService::applyLedgerBatch()` for all inputs (not individual applies)

**Rationale:**
- Single transaction ensures atomicity (all inputs succeed or all fail)
- Reduces DB roundtrips compared to per-commodity applies
- Maintains conservation invariant: partial states are impossible

---

## Testing & Verification

### Syntax Verification
```bash
php -l app/Services/Economy/ConstructionService.php
php -l app/Services/Economy/ConstructionTickService.php
php -l app/Models/ConstructionJob.php
php -l app/Http/Controllers/Api/ConstructionController.php
```

All pass ✓

### Migration Test
```bash
php artisan migrate
# 2026_03_04_000111_create_construction_jobs_table ... 89.48ms DONE
```

### Economy Tick Test
```bash
php artisan economy:tick --dry-run

# Output includes:
# Construction Jobs:
#   Checked: 0
#   Completed: 0
```

### Route Registration
```bash
php artisan route:list | grep construction
# All 3 routes visible and registered
```

### Recommended Future Tests

Create unit tests:

**tests/Unit/Services/Economy/ConstructionServiceTest.php**
```php
class ConstructionServiceTest extends TestCase
{
    public function test_build_success_with_sufficient_stock() { }
    public function test_build_fails_with_insufficient_stock() { }
    public function test_build_fails_if_ship_not_at_hub() { }
    public function test_build_consumes_inputs_via_ledger() { }
    public function test_multi_quantity_multiplies_requirements() { }
}
```

**tests/Unit/Services/Economy/ConstructionTickServiceTest.php**
```php
class ConstructionTickServiceTest extends TestCase
{
    public function test_processTick_matures_pending_jobs() { }
    public function test_processTick_ignores_future_jobs() { }
    public function test_processTick_dry_run_writes_nothing() { }
    public function test_processTick_filters_by_galaxy() { }
}
```

---

## Common Tasks

### Create a Simple Blueprint (Manual)

```bash
php artisan tinker
```

```php
$bp = Blueprint::create([
    'uuid' => Str::uuid(),
    'code' => 'TEST_ITEM',
    'name' => 'Test Item',
    'description' => 'A test blueprint',
    'type' => 'module',
    'output_item_code' => 'test_output',
    'build_time_ticks' => 60,  // 60 seconds
]);

$commodity = Commodity::first();  // Use existing commodity

BlueprintInput::create([
    'blueprint_id' => $bp->id,
    'commodity_id' => $commodity->id,
    'qty_required' => 10,
]);

$bp->load('inputs');  // Reload
$bp;
```

### Test Construction Build (Manual)

```php
$player = Player::first();
$ship = $player->activeShip;
$hub = TradingHub::first();
$bp = Blueprint::first();

$service = app(ConstructionService::class);
$result = $service->build($player, $ship, $hub, $bp, 1);
$result;  // {success, message, job_uuid, ...}

// View the job
$job = ConstructionJob::where('uuid', $result['job_uuid'])->first();
$job->load('blueprint', 'player', 'tradingHub');
$job;
```

### Run Economy Tick (Manual)

```bash
# All galaxies
php artisan economy:tick

# Specific galaxy
php artisan economy:tick --galaxy=<uuid>

# Dry-run
php artisan economy:tick --dry-run
```

### Check Job Maturation (Manual)

```php
$job = ConstructionJob::first();
$job->isMatured();           // bool
$job->completes_at->diff(now());  // shows time remaining
```

---

## Related Documentation

- [ADR 0001: Construction Output Deferral](../ADR/0001-construction-output-deferral.md)
- [Construction API Reference](../api/construction.md)
- [Trading API Reference](../api/trading.md)
- [Economy System Guide](./ECONOMICS_GUIDE.md)
- [Ledger Economy Architecture](../guides/TEMPLATE_INSTANCE_ARCHITECTURE.md)

---

## Troubleshooting

### Issue: Migration fails with "column already exists"
**Cause:** Migration already ran
**Fix:** Check `php artisan migrate:status`; if table exists, it's safe to skip

### Issue: Construction job won't complete after waiting
**Cause:** Economy tick hasn't run since job matured
**Fix:** Manually run `php artisan economy:tick` or wait for scheduled task

### Issue: Build fails with "Insufficient resources" despite appearing available
**Cause:** Another concurrent build consumed the resources between endpoint check and transaction
**Fix:** Normal race condition; retry after hub inventory is restocked

### Issue: API returns 404 for trading hub
**Cause:** UUID is POI UUID, not TradingHub UUID
**Fix:** `findTradingHub()` supports both, but check `trading_hubs.uuid` column

### Issue: Deadlock detected in concurrent builds
**Cause:** Inputs locked in wrong order (should not happen with Phase 3 code)
**Fix:** Check `sortBy('commodity_id')` is applied; if using manual queries, ensure mineral_id ASC

---

## Future Evolution

### Phase 4: Reserves & Anti-Cornering
- Player reserve policies to prevent blocking supplies
- Anti-cornering price multipliers for bulk purchases
- Hub resilience minimums

### Phase N+: Item Delivery
- Create `player_items` table + `PlayerItem` model
- Implement `ItemDeliveryService`
- Replace `completeJob()` stub with real delivery
- Add pickup deadline & storage limit logic

**No schema changes required** to ConstructionJob—already designed to support delivery later.

---

## Summary

Phase 3 successfully implements:
- ✅ TOCTOU race condition fix in trade locking
- ✅ Ledger-backed construction system with atomic input consumption
- ✅ Asynchronous job maturation via economy tick
- ✅ 3 new API endpoints with comprehensive documentation
- ✅ Clean migration path for future item delivery

All code syntax-verified, migration tested, routes registered, and integrated with existing economy systems.
