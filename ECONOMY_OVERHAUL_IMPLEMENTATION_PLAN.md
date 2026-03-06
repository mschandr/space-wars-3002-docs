# Economy Overhaul Implementation Plan

**Status:** Ready for review and handoff
**Created:** March 3, 2026
**Scope:** Full ledger-backed commodity economy with conservation

---

## 0. Executive Summary

Transform the current "abstract supply/demand" economy into a **stock-and-flow, ledger-backed system** where:
- Every mineral movement is tracked (source → ledger → inventory)
- Prices derive from coverage (days of stock) rather than abstract metrics
- Mining, construction, and upkeep create a closed-loop ecosystem
- Discoveries and shocks provide dynamic events with decay

**Timeline:** ~4 phases over 4-8 weeks
**Complexity:** Medium-High (new ledger + tick job + 6+ migrations)
**Risk:** Moderate (concurrency + backward compat with existing inventory)

---

## 1. Phase 0: Foundation & Schema (2-3 days)

**Goal:** Create all required tables; prepare for ledger enforcement.

### 1.1 New Tables

#### `commodities` (Reference)
```sql
CREATE TABLE commodities (
    id BIGINT PRIMARY KEY,
    uuid UUID UNIQUE NOT NULL,
    code VARCHAR(50) UNIQUE NOT NULL,           -- IRON, TITANIUM, RARE_EARTH, etc.
    name VARCHAR(100) NOT NULL,
    category ENUM('MINERAL', 'EXOTIC', 'SOFT') DEFAULT 'MINERAL',
    base_price DECIMAL(12,2) NOT NULL,          -- Credits per unit
    is_conserved BOOLEAN DEFAULT TRUE,          -- If false, no ledger enforcement

    -- Pricing clamps (per commodity)
    price_min_multiplier DECIMAL(3,2) DEFAULT 0.5,
    price_max_multiplier DECIMAL(3,2) DEFAULT 3.0,

    created_at TIMESTAMP,
    updated_at TIMESTAMP,

    INDEX idx_code (code),
    INDEX idx_category (category)
);
```

#### `commodity_ledger_entries` (Immutable log)
```sql
CREATE TABLE commodity_ledger_entries (
    id BIGINT PRIMARY KEY AUTO_INCREMENT,
    uuid UUID UNIQUE NOT NULL,
    timestamp DATETIME NOT NULL,
    system_id BIGINT NOT NULL,                  -- FK to systems/galaxies
    hub_id BIGINT,                              -- FK to trading_hubs (nullable for system-level entries)
    commodity_id BIGINT NOT NULL,               -- FK to commodities

    qty_delta DECIMAL(14,4) NOT NULL,           -- Positive=source, negative=sink
    reason_code VARCHAR(50) NOT NULL,           -- MINING, CONSTRUCTION, UPKEEP, TRADE_BUY, TRADE_SELL, SALVAGE, NPC_INJECT, NPC_CONSUME, GENESIS
    actor_type ENUM('PLAYER', 'NPC', 'SYSTEM') NOT NULL,
    actor_id BIGINT,                            -- Player ID, NPC ID, or null for SYSTEM

    correlation_id UUID,                        -- Ties multiple entries to one action (e.g., one trade)
    metadata JSON,                              -- {ship_id, blueprint_id, discovery_id, ...}

    created_at TIMESTAMP NOT NULL,

    FOREIGN KEY (hub_id) REFERENCES trading_hubs (id) ON DELETE SET NULL,
    FOREIGN KEY (commodity_id) REFERENCES commodities (id),

    INDEX idx_timestamp (timestamp),
    INDEX idx_hub_commodity_timestamp (hub_id, commodity_id, timestamp),
    INDEX idx_system_commodity_timestamp (system_id, commodity_id, timestamp),
    INDEX idx_correlation (correlation_id),
    INDEX idx_reason (reason_code)
);
```

#### `hub_commodity_stats` (Derived metrics, updated each tick)
```sql
CREATE TABLE hub_commodity_stats (
    id BIGINT PRIMARY KEY AUTO_INCREMENT,
    hub_id BIGINT NOT NULL,
    commodity_id BIGINT NOT NULL,

    -- Rolling averages (over last N ticks/days)
    avg_daily_demand DECIMAL(12,4) NOT NULL DEFAULT 0,
    avg_daily_supply DECIMAL(12,4) NOT NULL DEFAULT 0,

    -- Derived prices (cached from PricingService for perf)
    cached_buy_price DECIMAL(12,2),
    cached_sell_price DECIMAL(12,2),

    last_computed_at DATETIME NOT NULL,

    FOREIGN KEY (hub_id) REFERENCES trading_hubs (id) ON DELETE CASCADE,
    FOREIGN KEY (commodity_id) REFERENCES commodities (id) ON DELETE CASCADE,

    UNIQUE KEY unique_hub_commodity (hub_id, commodity_id),
    INDEX idx_hub (hub_id),
    INDEX idx_commodity (commodity_id)
);
```

#### `resource_deposits` (Mining sources)
```sql
CREATE TABLE resource_deposits (
    id BIGINT PRIMARY KEY AUTO_INCREMENT,
    uuid UUID UNIQUE NOT NULL,
    system_id BIGINT NOT NULL,                  -- FK to systems
    commodity_id BIGINT NOT NULL,               -- FK to commodities

    quality INT DEFAULT 100,                    -- 0-100, affects extraction rate
    max_extraction_per_tick DECIMAL(12,4) NOT NULL,  -- Units per economic tick

    discovered_at DATETIME,
    discovered_by_actor_id BIGINT,              -- Player or NPC who found it
    discovered_by_actor_type ENUM('PLAYER', 'NPC'),

    status ENUM('ACTIVE', 'DEPLETED', 'BLOCKED') DEFAULT 'ACTIVE',
    total_extracted DECIMAL(14,4) DEFAULT 0,    -- For tracking depletion
    max_total_qty DECIMAL(14,4),                -- Optional: total available before depletion

    metadata JSON,                              -- {sector_id, quality_notes, ...}
    created_at TIMESTAMP,
    updated_at TIMESTAMP,

    FOREIGN KEY (system_id) REFERENCES systems (id) ON DELETE CASCADE,
    FOREIGN KEY (commodity_id) REFERENCES commodities (id) ON DELETE RESTRICT,

    INDEX idx_system_commodity (system_id, commodity_id),
    INDEX idx_status (status),
    INDEX idx_discovered_at (discovered_at)
);
```

#### `economic_shocks` (Discovery booms, blockades, disasters)
```sql
CREATE TABLE economic_shocks (
    id BIGINT PRIMARY KEY AUTO_INCREMENT,
    uuid UUID UNIQUE NOT NULL,
    system_id BIGINT NOT NULL,
    commodity_id BIGINT,                        -- Nullable = system-wide shock

    shock_type VARCHAR(50) NOT NULL,            -- DISCOVERY, BLOCKADE, DISASTER, BOOM
    magnitude DECIMAL(5,3) NOT NULL,            -- e.g., +0.25 = +25% price multiplier

    -- Exponential decay: effective_mag = magnitude * exp(-decay_rate * elapsed_ticks)
    decay_half_life_ticks INT DEFAULT 100,      -- Ticks until magnitude = 50% of initial

    starts_at DATETIME NOT NULL,
    ends_at DATETIME,                           -- Explicit end, or computed from decay

    triggered_by_actor_id BIGINT,               -- Who triggered it (e.g., discovered deposit)
    triggered_by_actor_type ENUM('PLAYER', 'NPC', 'SYSTEM'),

    metadata JSON,                              -- {deposit_id, reason, ...}
    is_active BOOLEAN DEFAULT TRUE,

    created_at TIMESTAMP,
    updated_at TIMESTAMP,

    FOREIGN KEY (system_id) REFERENCES systems (id) ON DELETE CASCADE,
    FOREIGN KEY (commodity_id) REFERENCES commodities (id) ON DELETE SET NULL,

    INDEX idx_system_active (system_id, is_active),
    INDEX idx_commodity_active (commodity_id, is_active),
    INDEX idx_starts_at (starts_at)
);
```

#### `blueprints` (Construction recipes)
```sql
CREATE TABLE blueprints (
    id BIGINT PRIMARY KEY AUTO_INCREMENT,
    uuid UUID UNIQUE NOT NULL,
    code VARCHAR(100) UNIQUE NOT NULL,          -- FRIGATE_MK1, STATION_BASIC, etc.
    name VARCHAR(100) NOT NULL,
    description TEXT,

    type ENUM('SHIP', 'HABITAT', 'MODULE', 'FACILITY') NOT NULL,
    output_item_code VARCHAR(100),              -- What gets created (ship_id, habitat_id, etc.)

    build_time_ticks INT DEFAULT 10,            -- How long to build (in ticks)

    created_at TIMESTAMP,
    updated_at TIMESTAMP,

    INDEX idx_type (type),
    INDEX idx_code (code)
);
```

#### `blueprint_inputs` (Recipe ingredients)
```sql
CREATE TABLE blueprint_inputs (
    id BIGINT PRIMARY KEY AUTO_INCREMENT,
    blueprint_id BIGINT NOT NULL,
    commodity_id BIGINT NOT NULL,
    qty_required DECIMAL(12,4) NOT NULL,        -- Units needed

    FOREIGN KEY (blueprint_id) REFERENCES blueprints (id) ON DELETE CASCADE,
    FOREIGN KEY (commodity_id) REFERENCES commodities (id) ON DELETE RESTRICT,

    UNIQUE KEY unique_blueprint_commodity (blueprint_id, commodity_id)
);
```

### 1.2 Modify Existing Tables

#### `trading_hubs` - Add spread config
```sql
ALTER TABLE trading_hubs ADD COLUMN (
    spread_buy DECIMAL(5,4) DEFAULT 0.08 COMMENT 'Buy spread (0.08 = 8%)',
    spread_sell DECIMAL(5,4) DEFAULT 0.08 COMMENT 'Sell spread (0.08 = 8%)',
    reserve_policy_id BIGINT COMMENT 'FK to optional reserve policies'
);
```

#### `trading_hub_inventories` - Add on-hand tracking
```sql
ALTER TABLE trading_hub_inventories ADD COLUMN (
    on_hand_qty DECIMAL(14,4) DEFAULT 0 COMMENT 'Actual physical quantity (units)',
    reserved_qty DECIMAL(14,4) DEFAULT 0 COMMENT 'Reserved for pending orders (optional)',
    last_snapshot_at DATETIME COMMENT 'When this was last reconciled with ledger'
);

-- Backfill existing quantity into on_hand_qty
UPDATE trading_hub_inventories SET on_hand_qty = COALESCE(quantity, 0);
```

### 1.3 Migrations

Create files:
- `2026_03_03_000101_create_commodities_table.php`
- `2026_03_03_000102_create_commodity_ledger_entries_table.php`
- `2026_03_03_000103_create_hub_commodity_stats_table.php`
- `2026_03_03_000104_create_resource_deposits_table.php`
- `2026_03_03_000105_create_economic_shocks_table.php`
- `2026_03_03_000106_create_blueprints_table.php`
- `2026_03_03_000107_create_blueprint_inputs_table.php`
- `2026_03_03_000108_alter_trading_hubs_add_spread_and_policy.php`
- `2026_03_03_000109_alter_trading_hub_inventories_add_onhand.php`

### 1.4 Models

Create:
- `app/Models/Commodity.php`
- `app/Models/CommodityLedgerEntry.php`
- `app/Models/HubCommodityStats.php`
- `app/Models/ResourceDeposit.php`
- `app/Models/EconomicShock.php`
- `app/Models/Blueprint.php`
- `app/Models/BlueprintInput.php`

Update:
- `app/Models/TradingHub.php` - Add relationships

### 1.5 Enums

Create:
- `app/Enums/Economy/ReasonCode.php` - MINING, CONSTRUCTION, UPKEEP, TRADE_BUY, TRADE_SELL, SALVAGE, NPC_INJECT, NPC_CONSUME, GENESIS
- `app/Enums/Economy/ShockType.php` - DISCOVERY, BLOCKADE, DISASTER, BOOM
- `app/Enums/Economy/ActorType.php` - PLAYER, NPC, SYSTEM

---

## 2. Phase 1: Ledger & Inventory Services (3-4 days)

**Goal:** Enforce "no inventory mutation without ledger"; build core services.

### 2.1 LedgerService

**File:** `app/Services/Economy/LedgerService.php`

```php
class LedgerService {
    // Record mining output
    public function recordMiningOutput(
        int $systemId,
        int $hubId,
        int $commodityId,
        float $qty,
        int $actorId,
        ActorType $actorType,
        string $correlationId = null
    ): CommodityLedgerEntry

    // Record construction consumption
    public function recordConstruction(
        int $systemId,
        int $hubId,
        int $blueprintId,
        int $actorId,
        string $correlationId = null
    ): array // Returns array of CommodityLedgerEntry

    // Record trade
    public function recordTrade(
        int $systemId,
        int $hubId,
        int $commodityId,
        float $qty,
        float $pricePerUnit,
        int $buyerId,
        int $sellerId,
        string $correlationId = null
    ): CommodityLedgerEntry

    // Record upkeep consumption
    public function recordUpkeep(
        int $systemId,
        int $hubId,
        int $commodityId,
        float $qty,
        int $actorId,
        string $correlationId = null
    ): CommodityLedgerEntry

    // Batch create (for tick jobs)
    public function recordBatch(array $entries): array

    // Query ledger for analysis
    public function getLedgerHistory(
        int $systemId,
        ?int $hubId = null,
        ?int $commodityId = null,
        ?DateTimeInterface $since = null
    ): Collection
}
```

**Responsibilities:**
- Create immutable ledger entries
- Enforce that **only** these methods can create TRADE_BUY/TRADE_SELL entries
- Validate actor exists
- Return entries for downstream processing

### 2.2 InventoryService

**File:** `app/Services/Economy/InventoryService.php`

```php
class InventoryService {
    // Apply a single ledger entry to inventory
    public function applyLedgerEntry(
        CommodityLedgerEntry $entry
    ): void

    // Apply multiple entries (for tick jobs)
    public function applyLedgerBatch(array $entries): void

    // Get current on-hand for a commodity at a hub
    public function getOnHand(
        int $hubId,
        int $commodityId
    ): float

    // Check if inventory would go negative
    public function canApply(
        CommodityLedgerEntry $entry
    ): bool

    // Reconcile: verify ledger total matches on-hand
    public function reconcile(
        int $hubId,
        int $commodityId
    ): array // {ledger_total, on_hand, variance}
}
```

**Responsibilities:**
- Apply ledger entries to `hub_inventories.on_hand_qty`
- Prevent negative inventory for conserved commodities
- Transactional consistency
- Audit trail (snapshots)

### 2.3 Pricing Service Updates

**Update:** `app/Services/Pricing/PricingService.php`

Replace the current supply/demand formula with coverage-based:

```php
class PricingService {
    // Compute price from coverage + shocks + clamps
    public function computePrice(
        TradingHub $hub,
        Commodity $commodity,
        ?PricingContext $ctx = null
    ): PriceQuote // {buyPrice, sellPrice, components: {...}}

    private function computeScarcityMultiplier(float $coverageDays): float {
        // coverage <= 1 day   => max multiplier (near 3.0)
        // coverage ~7 days    => 1.0
        // coverage >= 30 days => min multiplier (near 0.5)
        // Smooth curve between
    }

    private function computeShockMultiplier(
        int $systemId,
        ?int $commodityId = null
    ): float {
        // Active shocks with decay
        // Clamp within reasonable global bounds
    }
}
```

**Formula:**
```
coverage_days = on_hand_qty / (avg_daily_demand + epsilon)
scarcity_mult = f(coverage_days)  // Smooth curve
shock_mult = ∏(1 + shock_magnitude * decay_factor)  // Clamped
raw_price = base_price * scarcity_mult * shock_mult
buy_price = raw_price * (1 + spread_buy)
sell_price = raw_price * (1 - spread_sell)

Final: clamp within [base * min_mult, base * max_mult]
```

### 2.4 Stats Service

**File:** `app/Services/Economy/HubCommodityStatsService.php`

```php
class HubCommodityStatsService {
    // Compute rolling averages from ledger over window
    public function computeStats(
        int $hubId,
        int $commodityId,
        int $windowTicks = 7  // 7-day window
    ): void  // Updates hub_commodity_stats

    // Batch recompute all stats (tick job)
    public function recomputeAllStats(int $windowTicks = 7): void
}
```

**Method:**
- Query ledger for TRADE_BUY, TRADE_SELL, MINING, etc. over window
- Aggregate by reason_code to separate demand from supply
- Store in `hub_commodity_stats`

### 2.5 Tests

Create:
- `tests/Unit/Services/Economy/LedgerServiceTest.php` - 12+ tests
- `tests/Unit/Services/Economy/InventoryServiceTest.php` - 15+ tests
- `tests/Unit/Services/Economy/PricingServiceCoverageTest.php` - 10+ tests

---

## 3. Phase 2: Mining & Tick Job (3-4 days)

**Goal:** Source minerals via mining; establish tick simulation loop.

### 3.1 Mining Service

**File:** `app/Services/Economy/MiningService.php`

```php
class MiningService {
    // Process all active deposits in a system
    public function mineTick(
        int $systemId,
        int $tickNumber = null
    ): array  // {processed: int, totalQty: float, newDeposits: int}

    // Extract from a specific deposit
    private function extractFromDeposit(
        ResourceDeposit $deposit,
        TradingHub $destinationHub  // Where to credit the minerals
    ): float  // Qty extracted this tick
}
```

**Logic per tick:**
1. For each active deposit in system:
   - Compute extraction = min(max_extraction_per_tick, remaining_capacity)
   - Record ledger entry (reason=MINING)
   - Credit to hub inventory

### 3.2 Discovery Service

**File:** `app/Services/Economy/DiscoveryService.php`

```php
class DiscoveryService {
    // Discover a new deposit (triggered by player/NPC event)
    public function discoverDeposit(
        int $systemId,
        int $commodityId,
        array $params,  // {max_extraction_per_tick, quality, ...}
        int $actorId,
        ActorType $actorType
    ): ResourceDeposit

    // Create a shock (DISCOVERY boom)
    private function createShock(
        ResourceDeposit $deposit,
        ShockType $type = ShockType::DISCOVERY
    ): EconomicShock
}
```

**On discovery:**
1. Create resource_deposit (status=ACTIVE)
2. Create economic_shock (type=DISCOVERY, magnitude=+0.25, half_life=100 ticks)
3. Optional: system-wide BOOM shock if discovered by player

### 3.3 Shock Decay

**File:** `app/Services/Economy/ShockDecayService.php`

```php
class ShockDecayService {
    // Compute effective magnitude of a shock at current tick
    public function getEffectiveMagnitude(EconomicShock $shock, int $atTick): float {
        $elapsedTicks = $atTick - $shock->starts_at_tick;
        $decayRate = log(2) / $shock->decay_half_life_ticks;
        return $shock->magnitude * exp(-$decayRate * $elapsedTicks);
    }

    // Mark shocks as inactive if fully decayed
    public function pruneDecayedShocks(): int
}
```

### 3.4 Tick Economy Job

**File:** `app/Console/Commands/TickEconomyCommand.php`

```php
class TickEconomyCommand extends Command {
    protected $signature = 'economy:tick {--system-id= : Optional specific system} {--ticks=1}';

    public function handle(): int {
        // 1. Decay shocks
        // 2. Mine: for each system, extract minerals
        // 3. (v1.5) Upkeep: consume minerals from habitats
        // 4. Recompute stats
        // 5. Cache prices (optional, for perf)

        return 0;
    }
}
```

**Sequence per tick:**
```
1. ShockDecayService::decayAll()
2. For each active system:
   a. MiningService::mineTick($systemId)
   b. ConstructionService::processBuildQueue($systemId) [v1.5]
   c. UpkeepService::processSinks($systemId) [v2]
3. HubCommodityStatsService::recomputeAllStats()
4. (Optional) PricingService::cacheAllPrices()
```

### 3.5 Tests

- `tests/Unit/Services/Economy/MiningServiceTest.php` - 8+ tests
- `tests/Unit/Services/Economy/DiscoveryServiceTest.php` - 6+ tests
- `tests/Unit/Services/Economy/ShockDecayServiceTest.php` - 5+ tests
- `tests/Feature/Economy/TickEconomyTest.php` - 10+ tests (concurrency, conservation, etc.)

---

## 4. Phase 3: Construction & Trade Enforcement (3-4 days)

**Goal:** Consumption sink via construction; enforce ledger in all trade paths.

### 4.1 Construction Service

**File:** `app/Services/Economy/ConstructionService.php`

```php
class ConstructionService {
    // Request to build something
    public function startConstruction(
        int $hubId,
        int $blueprintId,
        int $playerId
    ): ConstructionJob  // New model

    // Process construction queue each tick
    public function processBuildQueue(int $systemId): array

    // Complete a construction job
    private function completeConstruction(ConstructionJob $job): void {
        // 1. Record ledger entries for all inputs (reason=CONSTRUCTION)
        // 2. Create the output item (ship, habitat, etc.)
    }
}
```

### 4.2 Update Trade Flow

**File:** `app/Services/Trading/TradeExecutionService.php` (new)

Refactor existing trade code to:
```php
class TradeExecutionService {
    public function buyMineral(
        Player $player,
        TradingHub $hub,
        Commodity $commodity,
        float $quantity
    ): TradeResult {
        // 1. Begin DB transaction
        // 2. Lock hub_inventories row (FOR UPDATE)
        // 3. Compute current price
        // 4. Validate inventory sufficient
        // 5. Record ledger (reason=TRADE_BUY)
        // 6. Apply to inventory (InventoryService)
        // 7. Deduct credits from player
        // 8. Commit
        // 9. Return result with components for UX
    }

    public function sellMineral(
        Player $player,
        TradingHub $hub,
        Commodity $commodity,
        float $quantity
    ): TradeResult {
        // Similar, inverse
    }
}
```

**Concurrency:**
- DB transaction wrapping
- `SELECT ... FOR UPDATE` on hub_inventories
- Prevents double-spend

### 4.3 New Model: ConstructionJob

```php
class ConstructionJob extends Model {
    use HasUuid;

    public $fillable = [
        'player_id', 'hub_id', 'blueprint_id',
        'started_at', 'completed_at', 'progress_ticks',
        'status'  // QUEUED, IN_PROGRESS, COMPLETED
    ];
}
```

### 4.4 Tests

- `tests/Unit/Services/Economy/ConstructionServiceTest.php` - 10+ tests
- `tests/Feature/Economy/TradeWithLedgerTest.php` - 12+ tests (concurrency, conservation, etc.)

---

## 5. Phase 4: Refinement & Safety Valves (2-3 days)

**Goal:** Anti-cornering, reserves, and market maker.

### 5.1 Reserve Policy

**File:** `app/Models/ReservePolicy.php`

```php
class ReservePolicy {
    // Define minimum inventory + fallback NPC supply
    // {commodity_id, min_qty_on_hand, npc_fallback_price_multiplier}
}
```

**Logic:**
- If hub inventory < min_qty, enable NPC sales at elevated price
- Prevents progression blocking

### 5.2 Anti-Cornering

Add to config:
```php
'economy.anti_cornering' => [
    'max_purchase_per_tick' => 1000,  // Units
    'volume_fee_threshold' => 500,     // Above this, spread increases
    'volume_fee_per_unit' => 0.001,    // Additional spread per unit
]
```

### 5.3 Demand Estimation

If avg_daily_demand is 0 (new commodity):
- Use fallback: `assumed_daily_demand = (base_price / 100)` or config value
- Prevents division by zero

### 5.4 Tests

- `tests/Feature/Economy/AntiCorneringTest.php` - 6+ tests
- `tests/Feature/Economy/ReservePolicyTest.php` - 6+ tests

---

## 6. Integration Checklist

### Backward Compatibility
- [ ] Existing inventory data backfilled to ledger (GENESIS entries)
- [ ] Existing TradingHubInventory.quantity → on_hand_qty
- [ ] Phase 5-9 crew/vendor/customs systems still work
- [ ] Pricing still respects vendor markup (Vendor × Commodity = final price)

### Concurrency
- [ ] All trade paths use DB transactions
- [ ] Row-level locks (SELECT FOR UPDATE) on hub_inventories
- [ ] Ledger entries are immutable (no updates)

### Auditability
- [ ] Every inventory change has a ledger entry
- [ ] Ledger query tools for support/debugging
- [ ] Price quotes include component breakdown (base, scarcity, shock, spread)

### Performance
- [ ] hub_commodity_stats cached to avoid expensive ledger queries
- [ ] Indexes on (hub_id, commodity_id, timestamp)
- [ ] Tick job parallelizes per-system mining

---

## 7. Configuration (config/economy.php updates)

```php
return [
    'ledger' => [
        'enabled' => true,
        'enforce_conservation' => true,
    ],

    'mining' => [
        'enabled' => true,
        'base_max_extraction_per_tick' => 100,  // Default if deposit doesn't specify
    ],

    'construction' => [
        'enabled' => false,  // Enable in v1
        'build_time_per_blueprint' => [
            'FRIGATE_MK1' => 10,  // ticks
        ],
    ],

    'shocks' => [
        'enabled' => true,
        'default_discovery_magnitude' => 0.25,
        'default_boom_magnitude' => 0.15,
        'default_decay_half_life_ticks' => 100,
    ],

    'stats' => [
        'window_ticks' => 7,
        'recompute_frequency' => 'every_tick',  // or 'every_10_ticks'
    ],

    'pricing' => [
        'coverage_model' => true,  // Use new coverage-based pricing
        'scarcity' => [
            'min_coverage_days' => 0.5,
            'max_coverage_days' => 30,
            'neutral_coverage_days' => 7,
        ],
        'spread_per_side' => 0.08,
        'price_min_multiplier' => 0.5,
        'price_max_multiplier' => 3.0,
    ],

    'reserves' => [
        'enabled' => false,  // Enable in v2
        'core_commodities' => ['IRON', 'TITANIUM'],
        'min_qty' => 10000,
        'npc_fallback_price_multiplier' => 1.5,
    ],

    'anti_cornering' => [
        'max_purchase_per_tick' => 1000,
        'volume_fee_threshold' => 500,
        'volume_fee_per_unit' => 0.001,
    ],
];
```

---

## 8. Migration & Rollout Strategy

### Step 1: Parallel Runs
- Keep old pricing logic alongside new
- Run both for N ticks, compare results
- Verify conservation (ledger total = inventory total)

### Step 2: Gradual Cutover
- Enable new ledger for new trades
- Backfill historic trades with TRADE_BUY/TRADE_SELL entries
- Switch PricingService to coverage model

### Step 3: Tick Job Activation
- Start TickEconomyCommand on schedule (e.g., daily)
- Mine commodities into hubs
- Monitor prices, alert if extreme oscillations

### Step 4: Feature Rollout
- v1: Ledger + Mining (no construction yet)
- v1.5: Construction recipes + shocks + discoveries
- v2: Upkeep + reserves + anti-cornering

---

## 9. Open Questions (For You)

1. **Commodity seed data?**
   - Should we seeder IRON, TITANIUM, etc.? Or user-defined?

2. **Existing inventory backfill?**
   - How many units is current "quantity" in trading_hub_inventories?
   - Establish unit scale (e.g., 1 = 1 ton)?

3. **Mining rate?**
   - Should discovery auto-generate reasonable extraction rates, or manual config?

4. **Construction v1?**
   - Include in v1, or defer to v1.5?

5. **NPC actors?**
   - Should NPCs also participate in mining/construction ledger?
   - Or only players for v1?

---

## 10. Effort Estimate

| Phase | Tasks | Days | Risk |
|-------|-------|------|------|
| Phase 0 | Migrations + Models | 2-3 | Low |
| Phase 1 | Ledger/Inventory/Pricing | 3-4 | Medium |
| Phase 2 | Mining + Tick Job | 3-4 | Medium |
| Phase 3 | Construction + Trade Lock | 3-4 | High (concurrency) |
| Phase 4 | Reserves + Anti-Corner | 2-3 | Low |
| **Total** | | **13-18 days** | |

**Parallel Work:**
- Phases can overlap; Phase 1 doesn't block Phase 2 start
- Estimated real-world: 4-6 weeks with 2 devs or 1 AI agent

---

## 11. Success Criteria

✅ **Ledger complete** - All inventory mutations have ledger entries
✅ **Conservation** - Minerals enter only via mining, exit only via construction/upkeep
✅ **Price transparency** - Quotes show component breakdown
✅ **Auditability** - Can reconstruct any inventory state from ledger
✅ **Concurrency** - No race conditions under concurrent trade load
✅ **Performance** - Tick job completes in <1 second for 10-system galaxy
✅ **Gameplay** - New players can always progress (reserve policies in place)

---

## 12. Files to Create/Modify

### New Files (Phase 0-4)
- 9 migrations
- 7 models
- 3 enums
- 7 services
- 1 command
- 12+ test files
- 1 config update

### Modified Files
- `app/Models/Galaxy.php` - Add relationships
- `app/Models/TradingHub.php` - Add relationships
- `app/Models/Player.php` - Ledger/trade interactions
- `app/Services/Pricing/PricingService.php` - Coverage formula
- Trade execution paths (TradingController, TradingService, etc.)

---

**Status:** Ready for Phase 0 implementation
**Next Step:** User clarifies open questions (Section 9)
**Handoff:** Full implementation plan with clear acceptance criteria
