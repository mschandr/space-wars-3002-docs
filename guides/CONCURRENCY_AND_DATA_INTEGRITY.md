# Concurrency & Data Integrity Guide

## Overview

Space Wars 3002 implements multiple layers of protection against race conditions, lost updates, and data inconsistencies. This guide documents the patterns and implementation details.

---

## Pessimistic Locking (Pessimistic Lock-First)

### Pattern: lockForUpdate()

Used when multiple concurrent requests might modify the same row.

**When to use:**
- Trading operations (inventory mutations)
- Construction jobs (resource deduction)
- Player inventory changes

**Implementation:**

```php
// Inside a transaction:
$inventory = TradingHubInventory::where('id', $inventoryId)
    ->lockForUpdate()
    ->firstOrFail();

// Safe to read and modify $inventory
$inventory->quantity -= $amount;
$inventory->save();
```

**Files using this pattern:**
- `app/Services/TradingService.php` (buyMineral, sellMineral)
- `app/Console/Shops/MineralTradingHandler.php` (buyMineralFlow, sellMineralFlow)
- `app/Models/ResourceDeposit.php` (recordExtraction with transaction wrapper)

**Trade-off:**
- ✅ Prevents dirty reads and lost updates
- ✅ Serializes concurrent access naturally
- ❌ May reduce throughput under high contention
- ❌ Risk of deadlocks if multiple locks taken in different order

**Deadlock Prevention:** Always acquire locks in consistent order (e.g., mineral_id ASC)

---

## Atomic Updates (DB::raw)

### Pattern: Using DB::raw for Increments

Used for simple field increments that don't depend on current value.

**When to use:**
- Counters (times_bribed, total_checks, visit_count)
- Running totals (total_bribes_paid)
- Pure increments/decrements without conditional logic

**Implementation:**

```php
DB::table('galaxy_customs_records')
    ->where('id', $recordId)
    ->update([
        'times_bribed' => DB::raw('times_bribed + 1'),
        'total_bribes_paid' => DB::raw('total_bribes_paid + ' . (int)$amount),
        'updated_at' => now(),
    ]);
```

**Files using this pattern:**
- `app/Models/GalaxyCustomsRecord.php` (recordBribe, recordFine, recordSuccessfulCheck)

**SQL Generated:**
```sql
UPDATE galaxy_customs_records
SET times_bribed = times_bribed + 1,
    total_bribes_paid = total_bribes_paid + 100,
    updated_at = '2026-03-04 10:42:24'
WHERE id = 42;
```

**Benefits:**
- ✅ Single atomic SQL statement
- ✅ No read-modify-write cycle
- ✅ Works seamlessly with ACID transactions
- ✅ No stale model instances (use refresh() after)

---

## Transaction + Lock Combination

### Pattern: Serialized Read-Modify-Write

Used when the update depends on current values or has conditional logic.

**When to use:**
- Updates with CASE/WHEN logic
- Multiple interdependent field updates
- Operations needing consistency across multiple fields

**Implementation:**

```php
DB::transaction(function () use ($amount) {
    // Lock row first
    $deposit = DB::table('resource_deposits')
        ->where('id', $depositId)
        ->lockForUpdate()
        ->first();

    if (!$deposit) return;

    // Calculate new state
    $newTotal = floatval($deposit->total_extracted) + $amount;
    $newStatus = $newTotal >= floatval($deposit->max_total_qty)
        ? 'DEPLETED'
        : 'ACTIVE';

    // Update atomically
    DB::table('resource_deposits')
        ->where('id', $depositId)
        ->update([
            'total_extracted' => $newTotal,
            'status' => $newStatus,
            'updated_at' => now(),
        ]);
});
```

**Files using this pattern:**
- `app/Models/ResourceDeposit.php` (recordExtraction)

**Execution:**
1. Transaction begins
2. Row locked (other processes wait or fail)
3. Calculate derived values
4. Update atomic
5. Transaction commits, lock released

**Result:** Serialized access prevents all race conditions for this row

---

## Fail-Closed Security

### Pattern: Default-Deny Access Control

When access requirements cannot be met, deny access rather than defaulting to permissive.

**Before (Permissive Default):**
```php
$security = $player->currentPoi?->security_level ?? 0;  // Missing POI = security 0!
if ($security > $mineral->min_sector_security) {
    return false; // Only denies if security is higher
}
```

**Problem:** Player without POI (null) is treated as having zero security, allowing access to restricted items.

**After (Fail-Closed):**
```php
// Explicit null check
if ($player->currentPoi === null || $player->currentPoi->security_level === null) {
    return false;  // Deny if can't verify security
}
if ($player->currentPoi->security_level > $mineral->min_sector_security) {
    return false;
}
```

**File:** `app/Services/Trading/CommodityAccessService.php`

**Principle:** In security decisions, assume the most restrictive default unless explicitly verified.

---

## N+1 Prevention

### Pattern: relationLoaded() Guard

Only compute expensive properties when their dependencies are eager-loaded.

**Before (N+1 Risk):**
```php
'ship_persona' => $this->when(
    config('features.crew_profiles'),
    fn() => $this->getCrewPersona()  // Loads crew if not already loaded!
)
```

**After (N+1 Safe):**
```php
'ship_persona' => $this->when(
    config('features.crew_profiles') && $this->relationLoaded('crew'),
    fn() => $this->getCrewPersona()  // Only if crew was eager-loaded
)
```

**File:** `app/Http/Resources/ShipResource.php`

**Result:** If crew relation not loaded, persona is omitted from response (no query). Client learns to eager-load crew when needed.

### Pattern: Read-Only Relationship Lookup

Distinguish between "find existing" and "get or create" operations.

**Mutating (Creates Side Effect):**
```php
$relationship = $vendorService->getOrCreateRelationship($player, $vendor);
// Creates PlayerVendorRelationship if not exists
```

**Read-Only (No Side Effect):**
```php
$relationship = $vendorService->findRelationship($player, $vendor);
// Returns null if not exists; doesn't create
```

**File:** `app/Services/VendorProfileService.php` + `app/Http/Controllers/Api/VendorController.php`

**Usage:**
- GET /api/vendors/{uuid} uses `findRelationship()` (read-only view)
- Trade operations use `getOrCreateRelationship()` (creates tracking)

**Result:** Viewing vendor details doesn't pollute relationship table with false records.

---

## Database Constraints

### Unsigned Types

Prevent negative values at database level.

**Applied to:**
- `honesty`, `severity`, `detection_skill` (0.0–1.0 range)
- `bribe_threshold`, `total_bribes_paid` (non-negative counts)
- `qty_required` (extraction quantities)
- `max_extraction_per_tick`, `total_extracted`, `max_total_qty` (resource tracking)

**Example:**
```sql
ALTER TABLE customs_officials
    ADD CONSTRAINT honesty_range
    CHECK (honesty >= 0 AND honesty <= 1);
```

**Benefit:** Application errors cannot insert invalid data; database enforces boundaries.

### Unique Constraints

Prevent duplicates for identity fields.

**Applied to:**
- `uuid` columns (all external references)
- Example: `crew_assignments.uuid` unique

**Query Pattern:**
```php
// UUID is the external key:
CrewAssignment::where('uuid', $externalUuid)->first();

// Database prevents:
INSERT INTO crew_assignments (uuid, ...) VALUES ('xxxx', ...);
INSERT INTO crew_assignments (uuid, ...) VALUES ('xxxx', ...);  // UNIQUE violation
```

**Benefit:** API clients can safely assume UUID uniqueness.

---

## Immutability Enforcement

### Pattern: Override All Mutation Methods

Ensure ledger entries cannot be modified after creation.

**File:** `app/Models/CommodityLedgerEntry.php`

```php
public function update(array $attributes = [], array $options = []): bool
{
    throw new \Exception('Ledger entries are immutable');
}

public function save(array $options = []): bool
{
    throw new \Exception('Ledger entries are immutable');
}

public function delete(): ?bool
{
    throw new \Exception('Ledger entries are immutable');
}
```

**Coverage:**
- ✅ Direct update: `$entry->update([...])` → throws
- ✅ Bulk update: `$entry->save()` → throws
- ✅ Deletion: `$entry->delete()` → throws
- ✅ Eloquent mass updates: `CommodityLedgerEntry::where(...)->update([...])` still works (bypasses model)

**Note:** Mass updates via query builder bypass model methods. To prevent, use a service or repository layer.

---

## Preview/Accuracy

### Problem: Dry-Run Underreporting

When fetching preview counts, use the same scope as actual deletion.

**Before:**
```php
// Count (single galaxy)
$count = DB::table('table')->where('galaxy_id', $galaxyId)->count();

// Delete (including mirrors)
DB::table('table')->whereIn('galaxy_id', $allGalaxyIds)->delete();
```

Result: Dry-run reports 100 rows, but deletion removes 300 (mirrors included).

**After:**
```php
// Build mirror list
$allGalaxyIds = array_merge(
    [$galaxyId],
    DB::table('galaxies')->where('mirror_galaxy_id', $galaxyId)->pluck('id')->toArray()
);

// Count using same scope
$count = DB::table('table')->whereIn('galaxy_id', $allGalaxyIds)->count();

// Delete using same scope
DB::table('table')->whereIn('galaxy_id', $allGalaxyIds)->delete();
```

**File:** `app/Console/Commands/GalaxyFlushCommand.php`

**Result:** Dry-run accurately predicts what will be deleted.

---

## Summary Table

| Pattern | Use Case | Files | Atomicity | Performance |
|---------|----------|-------|-----------|-------------|
| **lockForUpdate()** | Trading, inventory mutations | TradingService, MineralTradingHandler | ✅ Full | Fair (serialized) |
| **DB::raw()** | Simple increments/counters | GalaxyCustomsRecord | ✅ Full | Excellent |
| **Transaction + Lock** | Conditional logic + state sync | ResourceDeposit | ✅ Full | Fair (serialized) |
| **Fail-Closed** | Access control | CommodityAccessService | ✅ Logic | Excellent |
| **relationLoaded()** | N+1 prevention | ShipResource | N/A | Excellent |
| **Read-Only Lookup** | Side-effect prevention | VendorProfileService | N/A | Excellent |
| **Unsigned Types** | Boundary enforcement | Migrations | ✅ DB-level | Excellent |
| **Unique Constraints** | Deduplication | Migrations | ✅ DB-level | Excellent |
| **Immutability** | Ledger integrity | CommodityLedgerEntry | ✅ Full | Excellent |

---

## Testing Concurrency

### Unit Test Pattern

```php
public function testConcurrentBribesDoNotLoseUpdates()
{
    $record = GalaxyCustomsRecord::create([...]);

    // Simulate two concurrent updates
    GalaxyCustomsRecord::find($record->id)->recordBribe(100);
    GalaxyCustomsRecord::find($record->id)->recordBribe(200);

    $record->refresh();

    // Both increments must be present
    $this->assertEquals(2, $record->times_bribed);
    $this->assertEquals(300, $record->total_bribes_paid);
}
```

### Integration Test Pattern

```php
public function testConcurrentExtractionDoesNotOverdraw()
{
    $deposit = ResourceDeposit::factory()->create([
        'total_extracted' => 0,
        'max_total_qty' => 100,
    ]);

    // Simulate two concurrent extractions
    $deposit->recordExtraction(60);
    $deposit->recordExtraction(50);  // Should not exceed max

    $deposit->refresh();

    // Total should be clamped
    $this->assertLessThanOrEqual(100, $deposit->total_extracted);
    $this->assertEquals('DEPLETED', $deposit->status);
}
```

---

## Production Deployment Checklist

- [ ] All migrations have proper rollback logic (DROP CONSTRAINT IF EXISTS)
- [ ] lockForUpdate() used for all inventory-mutating operations
- [ ] DB::raw() used for simple increments (not PHP loops)
- [ ] resourceDeposit->recordExtraction() uses transaction + lock
- [ ] Fail-closed security checks applied to commodity access
- [ ] relationLoaded() guards prevent N+1 in resource serialization
- [ ] Read-only lookups used for GET endpoints
- [ ] Ledger entries locked down (all mutation methods throw)
- [ ] UUID constraints applied to all identity fields
- [ ] CHECK constraints prevent out-of-bounds data
- [ ] Load tests verify no race conditions under concurrency

---

## References

- [Laravel Transactions](https://laravel.com/docs/9.x/queries#transactions)
- [Pessimistic Locking](https://laravel.com/docs/9.x/eloquent#pessimistic-locking)
- [ACID Compliance](https://en.wikipedia.org/wiki/ACID)
- [Deadlock Prevention](https://dev.mysql.com/doc/refman/8.0/en/innodb-deadlock-prevention.html)
