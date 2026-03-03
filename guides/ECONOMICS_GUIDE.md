# Space Wars 3002 - Economics System Guide

**Last Updated:** March 3, 2026

## Overview

The economics system in Space Wars 3002 is a data-driven, service-oriented architecture that powers all trading, pricing, and commodity management. This guide explains the architecture, key concepts, and how systems interact.

## Table of Contents

1. [Core Concepts](#core-concepts)
2. [Architecture](#architecture)
3. [Pricing System](#pricing-system)
4. [Commodity Categories](#commodity-categories)
5. [Supply & Demand](#supply--demand)
6. [Crew Personas](#crew-personas)
7. [Vendor Profiles](#vendor-profiles)
8. [Customs System](#customs-system)
9. [Black Market](#black-market)
10. [Configuration](#configuration)

---

## Core Concepts

### Single Source of Truth
All economic parameters are stored in `config/economy.php`, ensuring consistent behavior across all services.

### Template + Instance Pattern
Rather than hardcoding vendor behavior, the system uses:
- **Templates** (TradingPost): Predefined vendor profiles with names, criminality baseline, and dialogue
- **Instances** (VendorProfile): Per-POI realizations of templates, allowing galaxy-specific variations

### Deterministic Pricing
The `PricingService` is a pure function service:
- No database writes
- No lazy loading
- Consistent output for same inputs
- Fixed-point integer arithmetic (credits)

### Single Save Per Trade
The `HubInventoryMutationService` ensures all supply/demand mutations and price recalculations happen in **one database write** per trade, preventing partial updates and ensuring atomicity.

---

## Architecture

### Service Layer

```
┌─────────────────────────────────────────┐
│         Controller/API Layer            │
├─────────────────────────────────────────┤
│   TradingController                     │
│   CrewController                        │
│   VendorController                      │
│   CustomsController                     │
├─────────────────────────────────────────┤
│         Service Layer                   │
├─────────────────────────────────────────┤
│ PricingService           (pure math)     │
│ HubInventoryMutationService (mutations) │
│ CommodityAccessService   (filtering)    │
│ ShipPersonaService       (crew alignment)│
│ VendorProfileService     (relationships) │
│ CustomsService           (arrival checks)│
├─────────────────────────────────────────┤
│         Data Objects                    │
├─────────────────────────────────────────┤
│ PricingContext (immutable config)       │
├─────────────────────────────────────────┤
│         Models                          │
├─────────────────────────────────────────┤
│ TradingPost, VendorProfile              │
│ TradingHubInventory, Mineral            │
│ CrewMember, CustomsOfficial             │
│ PlayerVendorRelationship                │
└─────────────────────────────────────────┘
```

### Data Flow: Buy Mineral

```
1. Player initiates buy → TradingController::buyMineral()
2. Fetch inventory + mineral
3. Create PricingContext
4. Compute prices via PricingService
5. Create ShipPersona from crew
6. Apply vendor markup via VendorProfileService
7. Verify player can afford
8. HubInventoryMutationService::applyTrade('buy')
   → Mutate supply/demand
   → Recompute prices
   → Save [ONE DB WRITE]
9. Return trade result
```

---

## Pricing System

### Price Formula

**Demand Multiplier:**
```
demandMultiplier = 1 + ((demand_level - 50) / 100)
```
- demand_level=0 → 0.5x (lowest demand)
- demand_level=50 → 1.0x (neutral)
- demand_level=100 → 1.5x (highest demand)

**Supply Multiplier:**
```
supplyMultiplier = 1 - ((supply_level - 50) / 100)
```
- supply_level=0 → 1.5x (shortage)
- supply_level=50 → 1.0x (neutral)
- supply_level=100 → 0.5x (oversupply)

**Raw Price:**
```
rawPrice = mineral.base_value * demandMultiplier * supplyMultiplier
```

**With Event Boost:**
```
priceWithEvents = rawPrice * ctx.eventMultiplier
```

**With Mirror Universe:**
```
priceWithMirror = ctx.isMirrorUniverse ? priceWithEvents * ctx.mirrorBoost : priceWithEvents
```

**Clamped to Bounds:**
```
minPrice = base_value * config('economy.pricing.min_multiplier', 0.10)
maxPrice = base_value * config('economy.pricing.max_multiplier', 10.00)
clamped = clamp(priceWithMirror, minPrice, maxPrice)
```

**Final Mid-Price (integer):**
```
midPrice = round(clamped)
```

**Buy/Sell Spread:**
```
spread = config('economy.pricing.spread_per_side', 0.08)
buyPrice = round(midPrice * (1 - spread))
sellPrice = round(midPrice * (1 + spread))
```

### Price Determinism

**Same inputs ALWAYS produce same prices:**
- Mineral base_value ✓
- Supply level ✓
- Demand level ✓
- Event multiplier ✓
- Mirror universe flag ✓

**Example:** Mineral worth 1000 credits, demand=50, supply=50, no events:
- demandMultiplier = 1.0
- supplyMultiplier = 1.0
- rawPrice = 1000
- midPrice = 1000
- buyPrice = 1000 * (1 - 0.08) = 920
- sellPrice = 1000 * (1 + 0.08) = 1080

---

## Commodity Categories

### Three-Tier System

**CIVILIAN** (Always Visible)
- Default category for most minerals
- No access restrictions
- Examples: Iron, Copper, Water Ice

**INDUSTRIAL** (Reputation-Gated)
- Requires `min_reputation` threshold
- Not visible to new players
- Examples: Rare Earth Elements, Processed Minerals

**BLACK** (Threshold + Multi-Gate)
- Invisible until shady interaction threshold crossed
- Requires `min_reputation` after threshold
- Requires sector security <= `min_sector_security`
- Never mentioned in UI if not visible
- Examples: Spice, Exotic Compounds, Illegal Substances

### Visibility Logic

```php
// CommodityAccessService::filterForPlayer()
1. Civilian → Always visible
2. Industrial:
   - No min_reputation set → visible
   - min_reputation set AND player.reputation >= min → visible
   - Otherwise → hidden
3. Black Market:
   - player.shady_interactions < threshold → HIDDEN (silent)
   - player.shady_interactions >= threshold:
     - Check min_reputation if set
     - Check min_sector_security if set
     - If all pass → visible
     - Otherwise → hidden
```

---

## Supply & Demand

### Bidirectional Mutation

**OLD BUG (Fixed):** Supply and demand would both converge to 50% over time, causing price convergence to ~75% of base value.

**NEW SOLUTION:** Both supply AND demand move in opposite directions per trade:

**On Buy:**
```
quantity -= amount
demand_level += step  (player wants it more)
supply_level -= step  (less available)
```

**On Sell:**
```
quantity += amount
supply_level += step  (more available)
demand_level -= step  (less wanted)
```

### Step Units

To prevent excessive price volatility from single-unit trades:
```
unitsPerStep = config('economy.pricing.units_per_step', 10)
step = max(1, intdiv(amount, unitsPerStep))
```

**Example:** Trade 45 units
- step = max(1, intdiv(45, 10)) = max(1, 4) = 4
- Supply/demand shifts by 4 points (not 45)

### Bounds

Both supply and demand are clamped:
```
demand_level: [0, 100]
supply_level: [0, 100]
```

---

## Crew Personas

### Five Roles

| Role | Description | Typical Bonus |
|------|-------------|---|
| SCIENCE_OFFICER | Research, exploration | Rare item access |
| TACTICAL_OFFICER | Combat, negotiation | Combat gear discount |
| CHIEF_ENGINEER | Ship repair, maintenance | Repair discount |
| LOGISTICS_OFFICER | Trading, commerce | Trading discount |
| HELMS_OFFICER | Navigation, piloting | Travel cost reduction |

### Three Alignments

| Alignment | Shady Score | Vendor Bonuses | Black Market Access |
|-----------|---|---|---|
| LAWFUL | -1 | Legitimate vendors discount | Never has access |
| NEUTRAL | 0 | Mixed bonuses | Access after threshold |
| SHADY | +1 | Black market vendors discount | Preferred by criminals |

### Persona Computation

**ShipPersonaService** aggregates crew to compute ship-wide traits:

```php
$persona = [
    'overall_alignment' => 'lawful'|'neutral'|'shady',
    'shady_score' => sum(crew.shady_actions),
    'total_shady_interactions' => crew.sum('shady_actions'),
    'black_market_visible' => threshold_check,
    'vendor_bonuses' => [
        'trading_discount' => 0.0-0.15,
        'repair_discount' => 0.0-0.10,
        // ... more bonuses
    ]
]
```

### Black Market Visibility

- **Never exposed in API response if false** (silent filtering)
- Only computed internally
- Threshold: `config('economy.black_market.visibility_threshold', 10)` shady interactions
- Players never see "forbidden" messages—items simply don't exist

---

## Vendor Profiles

### Trading Post Templates (Global)

32 predefined templates created at app initialization:

- **12 Trading Hubs** (0.02–0.35 criminality)
  - "Kovac's Emporium", "Chen's Exchange", etc.
- **8 Salvage Yards** (0.35–0.60 criminality)
  - "The Rusty Bolt", "Junk Paradise", etc.
- **8 Shipyards** (0.02–0.10 criminality)
  - "Titan Yards", "Nova Shipworks", etc.
- **8 Markets** (0.10–0.28 criminality)
  - "Central Market", "Trade Floor", etc.

### Vendor Instances (Per-POI)

Each trading hub gets one **VendorProfile** instance:
- References a TradingPost template
- Criminality = template base ± 5% variance
- Inherits personality, dialogue pool, markup base
- Tracks per-player relationship data

### Effective Markup

```php
VendorProfileService::getEffectiveMarkup($vendor, $player)
```

**Formula:**
```
baseMarkup = vendor.markup_base
goodwillDiscount = player_relationship.goodwill * 0.001
crewBonus = ShipPersonaService::getVendorBonuses()
alignmentPenalty = misaligned_crew_penalty()

effectiveMarkup = baseMarkup - goodwillDiscount + crewBonus - alignmentPenalty
```

### Dialogue System

Fixed pools per vendor archetype + context:
```php
$dialogue = $vendor->dialogue_pool[$context][$index]
// Contexts: 'greeting', 'deal_accepted', 'deal_refused', 'farewell', etc.
```

Architecture supports future LLM integration without changing API.

---

## Customs System

### Workflow

**On Arrival at Inhabited POI:**

1. **Check for Customs Official**
   - If none → CLEARED (no customs authority)

2. **Scan Cargo**
   - Detect `is_illegal=true` or `category=BLACK` items
   - If none → CLEARED

3. **Compute Detection Chance**
   ```
   detectionChance = official.detection_skill
   detectionChance -= (hiddenCargoHold.level * 0.15) per hold
   detectionChance = clamp(detectionChance, 0.0, 1.0)
   ```

4. **Detection Roll**
   ```
   detected = random() < detectionChance
   ```

5. **Determine Outcome**
   - If not detected → CLEARED
   - If detected:
     - Assess cargo value
     - Check official honesty for bribe opportunity
     - Apply severity to determine fine or seizure
     - Extreme violations → IMPOUNDED

### Outcomes

```
CustomsOutcome::CLEARED      # No issues
CustomsOutcome::FINED        # Minor penalty
CustomsOutcome::CARGO_SEIZED # Illegal goods removed
CustomsOutcome::BRIBED       # Player successfully bribed official
CustomsOutcome::IMPOUNDED    # Ship seized for serious violations
```

### Hidden Cargo Holds

Ship component type that reduces customs detection:
- Per level: -15% detection chance
- Installation: Via shipyard system
- Removal: Returns to salvage inventory at 70% base price

---

## Black Market

### Visibility Threshold

**Requirement:** `shady_interactions >= config('economy.black_market.visibility_threshold', 10)`

### Gating Criteria

Even visible black market items require:
1. **Reputation Check** (if mineral.min_reputation set)
2. **Sector Security Check** (if mineral.min_sector_security set)
3. **Availability** (vendor criminality, player alignment)

### API Hygiene

- **Never return 403 Forbidden** for black market items
- **Silent filtering:** Items below visibility simply don't exist in responses
- **No UI hints** about black market until threshold crossed
- Response field `black_market_visible` only included if `true`

### Shady Interactions

Count per crew member, cumulative across all ships:
```
Player.shady_interactions = activeShip.crew().sum('shady_actions')
```

Incremented by:
- Trading with black market vendors
- Purchasing illegal commodities
- Customs bribery
- Faction sabotage missions

---

## Configuration

### config/economy.php

```php
return [
    'pricing' => [
        'spread_per_side'   => 0.08,          // 8% buy/sell spread
        'min_multiplier'    => 0.10,           // Floor: 10% of base
        'max_multiplier'    => 10.00,          // Ceiling: 10x base
        'price_impact'      => 0.01,           // Demand shift per unit step
        'units_per_step'    => 10,             // Integer step size
    ],
    'events' => [
        'enabled'           => true,
        'chance_per_tick'   => 0.15,           // 15% event chance per tick
        'magnitude_range'   => [0.5, 3.0],     // 50%-300% price swing
        'duration_minutes'  => [30, 240],      // 30min-4hr duration
    ],
    'black_market' => [
        'enabled'           => true,
        'visibility_threshold' => 10,          // 10 shady interactions
        'access_rules'      => [
            'min_reputation'      => -20,      // Can be negative reputation
            'min_crew_alignment'  => 'neutral',// At least one shady crew member
        ],
    ],
    'stock_by_rarity' => [
        'abundant'   => [15000, 30000],
        'common'     => [5000,  15000],
        'uncommon'   => [2000,   8000],
        'rare'       => [500,    3000],
        'epic'       => [200,    1000],
        'very_rare'  => [100,    1000],
        'legendary'  => [10,      200],
        'mythic'     => [5,        50],
    ],
];
```

### config/features.php

Feature flags for gradual rollout:

```php
return [
    'black_market'    => true,   // Black market system enabled
    'crew_profiles'   => true,   // Crew assignment system enabled
    'vendor_profiles' => true,   // Vendor relationship system enabled
    'customs'         => true,   // Customs checks on arrival
    'npc_traders'     => false,  // NPC trading (off by default)
    'pirates'         => true,   // Pirate encounters
    'colonies'        => true,   // Colony system
];
```

---

## Key Changes from Previous System

| Aspect | Before | After |
|--------|--------|-------|
| **Pricing Config** | Hard-coded in services | `config/economy.php` |
| **Spread Mismatch** | 0.15 vs 0.08 divergence | Single source of truth |
| **Vendor Behavior** | Simple archetypes | Template + instance pattern |
| **Supply/Demand** | One-directional (convergence bug) | Bidirectional (stable) |
| **Black Market** | Exposed in UI | Silent filtering |
| **Crew System** | None | Full persona-based system |
| **Customs** | None | Per-POI officials with detection |
| **Commodity Access** | None | Reputation + threshold gating |

---

## Testing Coverage

**46/52 tests passing (88%)**

Critical path tests verify:
- ✅ Pricing determinism and clamping
- ✅ Supply/demand bidirectional mutations
- ✅ Single-save transaction pattern
- ✅ Black market visibility gating
- ✅ Crew alignment computation
- ✅ Customs logic structure

See `/tests/Unit/Services/` for full test suite.

---

## Integration Points

### Controllers
- `CrewController` - Hire/dismiss crew
- `VendorController` - Vendor profiles and interactions
- `CustomsController` - Customs outcomes
- `TradingController` - Enhanced with commodity filtering

### Models
- `Player` - Active ship, shady interaction count
- `PlayerShip` - Crew relationships
- `CrewMember` - Per-player traits
- `VendorProfile` - Per-POI instances
- `CustomsOfficial` - Per-POI enforcement
- `Mineral` - Category and access restrictions

### Services
- `PricingService` - Pure pricing computation
- `HubInventoryMutationService` - Trade mutations
- `CommodityAccessService` - Visibility filtering
- `ShipPersonaService` - Crew alignment
- `VendorProfileService` - Markup and dialogue
- `CustomsService` - Arrival checks

---

## Performance Characteristics

| Operation | Complexity | Notes |
|-----------|-----------|-------|
| **Price Computation** | O(1) | Pure math, no DB |
| **Inventory Mutation** | O(1) | Single DB write |
| **Commodity Filtering** | O(n) | n = inventory items |
| **Crew Persona** | O(m) | m = crew members |
| **Vendor Lookup** | O(1) | Indexed by POI |
| **Customs Check** | O(k) | k = cargo items |

---

## Future Enhancements

1. **NPC Traders** - Enable `npc_traders` flag
2. **LLM Dialogue** - Replace static pools with Ollama/Claude integration
3. **Dynamic Events** - Market crashes, frenzies, shortages
4. **Faction Economics** - Pirate/colony pricing dynamics
5. **Player Economies** - Trading between players
6. **Commodity Futures** - Speculation system

---

## Glossary

- **Criminality** - 0.0 (legitimate) to 1.0 (black market)
- **Goodwill** - Relationship score with vendor (-100 to +100)
- **Shady Interactions** - Count of black market/illegal transactions
- **Visibility Threshold** - Number of shady interactions needed to see black market
- **Markup** - Percentage markup on base price applied by vendor
- **Mid-Price** - Base price after supply/demand adjustment
- **Spread** - Difference between buy and sell prices

---

**Version:** 2.0
**Status:** Production
**Next Review:** Q2 2026
