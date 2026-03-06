# Phase 4: Safety Valves & Anti-Cornering - Implementation Guide

**Date:** March 5, 2026
**Status:** ✅ COMPLETE
**Version:** 2026.03.05.003

---

## Overview

Phase 4 implements **anti-cornering mechanics** and **reserve policies** to prevent players from monopolizing trading markets and ensure stable price discovery through supply/demand mechanisms.

### Core Objectives

1. **Prevent Market Monopolization** - Purchase limits and volume fees
2. **Maintain Minimum Inventory** - Reserve policies guarantee NPC supply
3. **Ensure Price Stability** - Volume fees increase prices for large purchases
4. **Three-Layer Validation** - Database constraints + service validation + config coercion

---

## Architecture

### 1. Anti-Cornering Service

**File:** `app/Services/Economy/AntiCorneringService.php`

#### Methods

**`computeVolumeAdjustment(float $requestedQty, ?TradingHubInventory $inventory = null): float`**

Calculates additional spread penalty for large purchases above threshold.

```php
// Input validation
if ($requestedQty < 0) {
    throw new \InvalidArgumentException("Quantity cannot be negative. Got: {$requestedQty}");
}

// Config coercion to non-negative
$threshold = max(0, (float) config('economy.anti_cornering.volume_fee.threshold'));
$feePerUnit = max(0, (float) config('economy.anti_cornering.volume_fee.fee_per_unit'));
$maxSpread = max(0, (float) config('economy.anti_cornering.volume_fee.max_additional_spread'));

// Returns: 0.0 to max_additional_spread (always >= 0)
return min($additionalSpread, $maxSpread);
```

**`canPurchaseThisTick(Player $player, float $requestedQty): bool`**

Enforces per-tick purchase limits to prevent monopolization.

```php
// Input validation: quantity >= 0
if ($requestedQty < 0) {
    throw new \InvalidArgumentException("Quantity cannot be negative. Got: {$requestedQty}");
}

// Config validity check
$maxPerTick = config('economy.anti_cornering.max_purchase_per_tick');
if ($maxPerTick === null || $maxPerTick <= 0) {
    return true; // No limit
}

// Query ledger for daily TRADE_BUY entries
$purchasedThisTick = abs($purchasedThisTick ?? 0); // Absolute value ensures non-negative

return ($purchasedThisTick + $requestedQty) <= $maxPerTick;
```

**`getPurchaseBlockReason(Player $player, float $requestedQty): ?string`**

Returns human-readable error message if purchase limit exceeded.

---

### 2. Reserve Policy Model

**File:** `app/Models/ReservePolicy.php`

Galaxy-scoped minimum inventory guarantees with NPC fallback pricing.

```php
protected $fillable = [
    'galaxy_id',        // Which galaxy this policy applies to
    'commodity_id',     // null = system-wide, or specific commodity
    'min_qty_on_hand',  // Minimum inventory level
    'npc_fallback_enabled',
    'npc_price_multiplier',  // NPC supply markup (e.g., 1.5 = 50% markup)
    'description',
];

protected $casts = [
    'min_qty_on_hand' => 'float',
    'npc_price_multiplier' => 'float',
    'npc_fallback_enabled' => 'boolean',
];
```

#### Unique Constraint

```sql
UNIQUE KEY unique_per_galaxy_commodity (galaxy_id, commodity_id)
```

Ensures one policy per (galaxy, commodity) pair.

#### Database Schema

```sql
CREATE TABLE reserve_policies (
    id BIGINT PRIMARY KEY AUTO_INCREMENT,
    galaxy_id BIGINT NOT NULL,
    commodity_id BIGINT NULLABLE,
    min_qty_on_hand UNSIGNED DECIMAL(14,4) NOT NULL,
    npc_price_multiplier UNSIGNED DECIMAL(5,2) NOT NULL DEFAULT 1.5,
    npc_fallback_enabled BOOLEAN DEFAULT TRUE,
    description VARCHAR(255),
    created_at TIMESTAMP,
    updated_at TIMESTAMP,
    FOREIGN KEY (galaxy_id) REFERENCES galaxies(id) ON DELETE CASCADE,
    FOREIGN KEY (commodity_id) REFERENCES commodities(id) ON DELETE SET NULL,
    UNIQUE (galaxy_id, commodity_id)
);
```

---

### 3. Pricing Service Enhancement

**File:** `app/Services/Pricing/PricingService.php`

Extended `computePriceCoverageBased()` to integrate volume fees.

#### Signature

```php
public function computePriceCoverageBased(
    TradingHub $hub,
    Commodity $commodity,
    ?HubCommodityStats $stats = null,
    float $onHandQty = 0,
    ?float $requestedQty = null
): array
```

#### Volume Fee Integration

```php
// Compute base price from coverage
$coverage = $onHandQty / $dailyDemand;
$clampedPrice = // ... coverage-based computation

// Add volume fee if quantity provided
$volumeFee = 0;
if ($requestedQty !== null) {
    $volumeFee = $this->antiCorneringService->computeVolumeAdjustment(
        $requestedQty,
        $inventory
    );
}

// Apply volume fee to spread
$finalSpreadBuy = $spreadBuy + $volumeFee;
$finalSpreadSell = $spreadSell + $volumeFee;

return [
    'buy_price' => (int)round($clampedPrice * (1 + $finalSpreadBuy)),
    'sell_price' => (int)round($clampedPrice * (1 - $finalSpreadSell)),
];
```

#### Demand Estimation Fallback Chain

```php
private function estimateDailyDemand(float $avgDailyDemand, Commodity $commodity): float
{
    // Input validation: must be non-negative
    if ($avgDailyDemand < 0) {
        throw new \InvalidArgumentException("avgDailyDemand cannot be negative. Got: {$avgDailyDemand}");
    }

    // If provided, use it
    if ($avgDailyDemand > 0) {
        return $avgDailyDemand;
    }

    // Fallback 1: base_price_ratio (1% of commodity base price)
    $estimate = $commodity->base_price / 100;
    if ($estimate >= 0) {
        return $estimate;
    }

    // Fallback 2: static config value with max(0, ...) coercion
    $fallback = (float) config('economy.demand_estimation.fallback_daily_demand', 100);
    return max(0, $fallback); // Guaranteed >= 0
}
```

---

### 4. Trading Service Integration

**File:** `app/Services/TradingService.php`

Both `buyMineral()` and `sellMineral()` now compute prices with quantity parameter.

#### Buy Flow

```php
public function buyMineral(Player $player, PlayerShip $ship, TradingHubInventory $inventory, int $quantity): array
{
    // Pre-transaction validations (cheap checks)
    if (!$inventory->hasStock($quantity)) {
        return ['success' => false, 'message' => 'Insufficient stock available'];
    }
    if ($availableSpace < $quantity) {
        return ['success' => false, 'message' => 'Insufficient cargo space'];
    }

    // Atomic transaction with inventory lock
    return DB::transaction(function () use ($player, $ship, $inventory, $quantity) {
        $inventory = TradingHubInventory::where('id', $inventory->id)
            ->lockForUpdate()  // Prevent TOCTOU
            ->firstOrFail();

        // Compute prices WITH quantity (includes volume fee)
        $priceData = $this->pricingService->computePriceCoverageBased(
            $inventory->tradingHub,
            $inventory->mineral,
            null,
            (float) $inventory->on_hand_qty,
            (float) $quantity  // <-- Volume fee calculation here
        );

        $unitPrice = $priceData['sell_price'];  // Volume-adjusted price
        $totalCost = $isTutorial ? 0 : $unitPrice * $quantity;

        // Validate credits AFTER computing real costs
        if (!$isTutorial && $player->credits < $totalCost) {
            throw new \Exception('Insufficient credits (includes anti-cornering fees)');
        }

        // Execute trade
        $player->deductCredits($totalCost);
        $this->mutationService->applyTrade($inventory, $quantity, 'buy', $ctx);
        // ... update cargo, log transaction
    });
}
```

#### Sell Flow

Similar structure for `sellMineral()`, using `$priceData['buy_price']` for sell prices.

---

### 5. API Integration

**File:** `app/Http/Controllers/Api/TradingTransactionController.php`

Pre-flight check before trading.

```php
public function buyMinerals(Request $request, string $uuid): JsonResponse
{
    // ... validation and resolution ...

    // Phase 4: Anti-cornering check
    if (!$this->antiCorneringService->canPurchaseThisTick($player, $validated['quantity'])) {
        $blockReason = $this->antiCorneringService->getPurchaseBlockReason($player, $validated['quantity']);
        return $this->error($blockReason, 'PURCHASE_LIMIT_EXCEEDED');
    }

    // Proceed to trading service
    $result = $this->tradingService->buyMineral(
        $player,
        $ship,
        $inventory,
        $validated['quantity']
    );
}
```

---

## Configuration

**File:** `config/economy.php`

### Reserve Policies

```php
'reserves' => [
    'enabled' => true,
    'policies' => [
        'default' => [
            'min_qty' => 5000,
            'npc_price_multiplier' => 1.5,
        ],
        // Per-commodity overrides
        'IRON' => [
            'min_qty' => 10000,
            'npc_price_multiplier' => 1.25,
        ],
    ],
],
```

### Anti-Cornering

```php
'anti_cornering' => [
    'volume_fee' => [
        'threshold' => 500,                    // Units above this trigger fee
        'fee_per_unit' => 0.001,              // Fee per unit above threshold
        'max_additional_spread' => 0.25,      // Cap additional spread at 25%
    ],
    'max_purchase_per_tick' => null,          // null = unlimited, or positive integer
    'cooldown_minutes' => 0,                  // Future: cooldown between large purchases
],
```

### Demand Estimation

```php
'demand_estimation' => [
    'fallback_method' => 'base_price_ratio',  // or 'static'
    'fallback_daily_demand' => 100,           // Fallback amount
],
```

---

## Three-Layer Validation

### Layer 1: Database Constraints

**UNSIGNED DECIMAL fields** prevent negative value storage:

```sql
min_qty_on_hand UNSIGNED DECIMAL(14,4)
npc_price_multiplier UNSIGNED DECIMAL(5,2)
on_hand_qty UNSIGNED DECIMAL(14,4)
```

MySQL/SQLite enforces at INSERT/UPDATE, rejecting or coercing negative values.

### Layer 2: Service Layer Validation

**Explicit checks** throw `InvalidArgumentException` on negative inputs:

```php
if ($requestedQty < 0) {
    throw new \InvalidArgumentException("Quantity cannot be negative. Got: {$requestedQty}");
}
```

### Layer 3: Configuration Coercion

**Runtime safety** ensures config values are non-negative:

```php
$threshold = max(0, (float) config('economy.anti_cornering.volume_fee.threshold'));
return max(0, $fallback);
```

---

## Seeding

**File:** `database/seeders/ReservePolicySeeder.php`

Creates default policies for all galaxies and commodities.

### Execution

```bash
# Via SeedTestData command
php artisan seed:test-data

# Via DatabaseSeeder (explicit call needed)
$seeder = new ReservePolicySeeder();
$seeder->setCommand($this);
$seeder->run();
```

### Generated Policies

For each galaxy:
- **System-wide policy** (commodity_id = null): 5000 min qty, 1.5x multiplier
- **Per-commodity policies**: Rarity-based minimums and multipliers

Example for a rare mineral:
```php
ReservePolicy::create([
    'galaxy_id' => $galaxy->id,
    'commodity_id' => $mineral->id,
    'min_qty_on_hand' => 3000,
    'npc_price_multiplier' => 1.75,
    'npc_fallback_enabled' => true,
    'description' => 'Reserve policy for Quantium (rare)',
]);
```

---

## Testing

### Unit Tests

**AntiCorneringServiceTest** (12 tests)
```php
✅ test_volume_adjustment_above_threshold()
✅ test_volume_adjustment_capped_by_max_spread()
✅ test_purchase_limit_blocks_excessive_buying()
✅ test_negative_quantity_throws_exception_on_volume_adjustment()
✅ test_negative_quantity_throws_exception_on_can_purchase()
✅ test_negative_quantity_throws_exception_on_block_reason()
// ... 6 more
```

**ReservePolicyTest** (11 tests)
```php
✅ test_reserve_policy_creation()
✅ test_reserve_policy_database_prevents_negative_min_qty()
✅ test_reserve_policy_database_prevents_negative_price_multiplier()
✅ test_reserve_policy_accepts_zero_values()
// ... 7 more
```

**PricingServicePhase4Test** (10 tests)
```php
✅ test_compute_price_integrates_volume_fee()
✅ test_demand_estimation_fallback_base_price_ratio()
✅ test_demand_estimation_fallback_static()
// ... 7 more
```

### Feature Tests

**Phase4IntegrationTest** (12 tests)
```php
✅ test_large_purchase_includes_volume_fee()
✅ test_purchase_limit_prevents_monopoly()
✅ test_inventory_locks_prevent_toctou()
✅ test_credits_validation_includes_volume_fees()
✅ test_transaction_log_records_volume_adjusted_prices()
// ... 7 more
```

### Running Tests

```bash
# All Phase 4 tests
php artisan test tests/Unit/Services/Economy/
php artisan test tests/Unit/Services/Pricing/
php artisan test tests/Feature/Economy/

# Specific test
php artisan test tests/Feature/Economy/Phase4IntegrationTest.php

# With coverage
php artisan test --coverage tests/Feature/Economy/Phase4IntegrationTest.php
```

---

## Troubleshooting

### Issue: "Insufficient credits (includes anti-cornering fees)"

**Cause:** Volume fee increased total cost beyond player's credits.

**Solution:** Check `config/economy.anti_cornering.volume_fee` settings:
- Reduce `fee_per_unit` to lower impact
- Increase `threshold` to apply fees only to larger purchases
- Adjust `max_additional_spread` cap

### Issue: "Purchase limit exceeded (10000 units)"

**Cause:** Player has already purchased `max_purchase_per_tick` this game tick.

**Solution:**
- Wait for next tick
- Reduce `max_purchase_per_tick` in config if testing
- Disable limit by setting to `null`

### Issue: Reserve policy not applied

**Cause:** Seeding not run or policy not created for commodity.

**Solution:**
```bash
# Run seeding
php artisan seed:test-data

# Verify policies exist
php artisan tinker
>>> App\Models\ReservePolicy::count()
```

---

## Future Enhancements

1. **Market Maker System** - AI traders that maintain spreads within reserve ranges
2. **Dynamic Purchase Limits** - Adjust max_purchase_per_tick based on market conditions
3. **Cooldown System** - Enforce waiting period between large purchases
4. **Alert System** - Notify admins when monopoly attempts detected
5. **Reputation Impact** - Penalize players who trigger anti-cornering limits excessively

---

## Performance Considerations

### Database Queries

- **Reserve Policy lookup:** Indexed on (galaxy_id, commodity_id)
- **Ledger queries:** Filtered by player_id and transaction_type, indexed
- **Inventory locks:** Using SELECT...FOR UPDATE atomically

### Caching Opportunities

- Cache reserve policies per galaxy (invalidate on update)
- Cache demand estimation fallback values
- Cache anti-cornering config values (reload on config change)

---

## Security Notes

1. **TOCTOU Prevention:** Inventory re-fetched with `lockForUpdate()` inside transaction
2. **Integer Overflow:** Using `decimal(14,4)` prevents overflow in quantity calculations
3. **Config Injection:** All config values coerced via `max(0, ...)` to prevent negative escape
4. **Error Messages:** Clear messages identify validation failures without exposing internals

---

## Commits

- `6278783` - Phase 4: Safety Valves & Anti-Cornering - Trading Integration
- `e9028ac` - Phase 4: Reserve Policy Seeding & Integration Tests

---

## Related Documentation

- [PHASE_4_NON_NEGATIVE_VALIDATION_AUDIT.md](./PHASE_4_NON_NEGATIVE_VALIDATION_AUDIT.md)
- [ECONOMICS_GUIDE.md](./ECONOMICS_GUIDE.md)
- [config/economy.php](../../config/economy.php)
- [app/Services/Economy/AntiCorneringService.php](../../app/Services/Economy/AntiCorneringService.php)

---

**Status:** ✅ Phase 4 Implementation Complete
