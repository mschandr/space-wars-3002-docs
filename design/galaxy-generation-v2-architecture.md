# Galaxy Generation Analysis & V2 Architecture Plan

**Date**: 2026-03-09
**Status**: Proposal for User Review
**Objective**: Break galaxy creation into discrete, measurable phases with Big O analysis and timing estimates

---

## Executive Summary

The current galaxy generation uses two orchestrators:
1. **TieredGalaxyCreationService** (14-step monolithic approach)
2. **GalaxyGenerationOrchestrator** (9-pipeline approach)

### Key Findings
- Current system generates ~3,000 POIs synchronously in ~10-30 seconds
- Bottlenecks: warp gate generation (O(n²) worst case), trading hub stocking (O(n*m))
- Proposal: V2 architecture with discrete, user-callable phases for complete control and visibility

---

## Current System Analysis

### Phase 1: Galaxy Record Creation
**Code**: `TieredGalaxyCreationService::createGalaxyRecord()`

**Big O Analysis**:
- Time: O(1)
- Space: O(1)
- Cyclomatic Complexity: 3

**Query Count Estimate**: 1-2 queries
- INSERT into galaxies
- Optional: UPDATE if resuming

**Real-time Estimate**: ~5-10ms

---

### Phase 2: Core Region Star Generation
**Code**: `CoreSystemGenerator::generateCoreRegion()`

**Input**:
- Galaxy dimensions
- Core bounds (e.g., 81 stars for medium galaxy)
- Distribution method (Vogel spiral, Poisson, etc.)

**Big O Analysis**:
- Time: O(n log n) where n = number of core stars (Vogel: O(n), Poisson: O(n*attempts))
- Space: O(n)
- Cyclomatic Complexity: 8-12 (depends on generator algorithm)

**Query Count Estimate**:
- 1 query: batch INSERT all core POIs
- 1 query: UPDATE galaxy bounds

**Total**: ~2 queries

**Real-time Estimate**: ~500ms - 2 seconds
- Vogel spiral: fastest O(n)
- Poisson disk: slower O(n*30) attempts

---

### Phase 3: Fortress Defense Deployment
**Code**: `CoreSystemGenerator::deployFortressDefenses()`

**Big O Analysis**:
- Time: O(n) where n = inhabited core systems (~32 for medium tier)
- Space: O(n)
- Cyclomatic Complexity: 5

**Query Count Estimate**:
- 1 query: batch INSERT system_defenses rows
- 1 query: UPDATE points_of_interest with defense flags

**Total**: ~2 queries

**Real-time Estimate**: ~100-200ms

---

### Phase 4: Trading Post Creation
**Code**: `CoreSystemGenerator::createTradingPosts()`

**Big O Analysis**:
- Time: O(n) where n = inhabited core systems
- Space: O(n)
- Cyclomatic Complexity: 6

**Query Count Estimate**:
- 1 query: filter inhabited POIs
- 1 query: batch INSERT trading_hubs
- 1 query: batch INSERT trading_hub_inventories (27 minerals × hub count)

**Total**: ~3 queries

**Real-time Estimate**: ~200-500ms

---

### Phase 5: Core Warp Gate Network Generation
**Code**: `IncrementalWarpGateGenerator::generateGates()` → `TieredGalaxyCreationService::generateCoreWarpGates()`

**Big O Analysis**:
- Time: O(n) × O(m) where n = inhabited systems, m = adjacent systems per gate
- Worst case: O(n²) if every system connects to every other (prevented by adjacency threshold)
- Typical: O(n * 5-10) with configured max_gates
- Space: O(n * m)
- Cyclomatic Complexity: 12-15

**Query Count Estimate**:
- 1 query: SELECT inhabited POIs with coordinates
- N queries: for each system, SELECT nearby POIs (spatial query)
- 1 query: batch INSERT all warp_gates

**Total**: ~3-5 queries + N spatial queries

**Real-time Estimate**: ~2-5 seconds
- Heavily dependent on max_gates_per_system and adjacency threshold

---

### Phase 6: Outer Region Star Generation
**Code**: `OuterSystemGenerator::generateOuterRegion()`

**Big O Analysis**:
- Time: O(n log n) where n = outer stars (~2,919 for medium, 3000 - 81)
- Space: O(n)
- Cyclomatic Complexity: 8-10

**Query Count Estimate**:
- 1 query: batch INSERT all outer POIs
- 1 query: UPDATE galaxy bounds

**Total**: ~2 queries

**Real-time Estimate**: ~1-3 seconds (larger dataset than core)

---

### Phase 7: Outer Planetary System Generation
**Code**: `OuterSystemGenerator::generatePlanetarySystems()`

**Big O Analysis**:
- Time: O(n) where n = outer stars
- Per-star: 5-12 planets created
- Space: O(n * 8) [average planets per star]
- Cyclomatic Complexity: 7

**Query Count Estimate**:
- 1 query: batch INSERT all planets/moons (n * 8 rows ≈ 23,000 rows)

**Total**: ~1 query

**Real-time Estimate**: ~1-2 seconds (bulk insert)

---

### Phase 8: Mineral Deposit Population
**Code**: `OuterSystemGenerator::populateMineralDeposits()` → `MineralDepositGenerator`

**Big O Analysis**:
- Time: O(n) where n = outer POIs with mineral spawn chance (95% of ~2,919)
- Per-POI: 1-3 deposits based on rarity
- Space: O(n * deposits_per_poi)
- Cyclomatic Complexity: 6

**Query Count Estimate**:
- 1 query: SELECT all outer POIs
- 1 query: batch INSERT all resource_deposits (~2,800-3,300 deposits)

**Total**: ~2 queries

**Real-time Estimate**: ~500ms - 1 second

---

### Phase 9: Dormant Gate Placement
**Code**: `TieredGalaxyCreationService::placeDormantGates()`

**Big O Analysis**:
- Time: O(n) where n = uninhabited POIs in outer region
- Space: O(n)
- Cyclomatic Complexity: 5

**Query Count Estimate**:
- 1 query: SELECT uninhabited outer POIs
- 1 query: batch INSERT dormant warp_gates

**Total**: ~2 queries

**Real-time Estimate**: ~300-500ms

---

### Phase 10: Precursor Ship Placement
**Code**: Various precursor placement logic

**Big O Analysis**:
- Time: O(1) or O(n) depending on placement strategy
- Space: O(1)
- Cyclomatic Complexity: 3-4

**Query Count Estimate**: 1-3 queries

**Real-time Estimate**: ~50-100ms

---

### Phase 11: Mirror Universe Creation
**Code**: `MirrorUniverseGenerator`

**Big O Analysis**:
- Time: O(n) where n = all POIs in prime universe (copies template)
- Space: O(n)
- Cyclomatic Complexity: 7

**Query Count Estimate**:
- Multiple queries for copying POI structure

**Total**: ~5-10 queries

**Real-time Estimate**: ~2-3 seconds

---

### Phase 12: NPC Generation
**Code**: `NpcGenerationService::generate()`

**Big O Analysis**:
- Time: O(n) where n = desired NPC count (configurable, typically 50-200)
- Space: O(n)
- Cyclomatic Complexity: 8

**Query Count Estimate**:
- 1 query: batch INSERT NPCs
- 1 query: batch INSERT NPC ships

**Total**: ~2 queries

**Real-time Estimate**: ~200-500ms

---

### Phase 13: Market Event Generation
**Code**: `MarketEventGenerator::generate()`

**Big O Analysis**:
- Time: O(n * m) where n = trading hubs, m = minerals
- Space: O(n * m)
- Cyclomatic Complexity: 6

**Query Count Estimate**:
- 1 query: batch INSERT market_events

**Total**: ~1 query

**Real-time Estimate**: ~100-200ms

---

### Phase 14: Galaxy Finalization
**Code**: `TieredGalaxyCreationService::finalizeGalaxy()`

**Big O Analysis**:
- Time: O(1)
- Space: O(1)
- Cyclomatic Complexity: 2

**Query Count Estimate**: 1 query (UPDATE galaxy status)

**Real-time Estimate**: ~10ms

---

## Total Current System Analysis

| Phase | Queries | Time (ms) | O(n) |
|-------|---------|-----------|------|
| 1. Galaxy Record | 1-2 | 5-10 | O(1) |
| 2. Core Stars | 2 | 500-2000 | O(n log n) |
| 3. Fortress Defense | 2 | 100-200 | O(n) |
| 4. Trading Posts | 3 | 200-500 | O(n) |
| 5. Core Gates | 3-5 | 2000-5000 | O(n²) worst |
| 6. Outer Stars | 2 | 1000-3000 | O(n log n) |
| 7. Planetary Systems | 1 | 1000-2000 | O(n) |
| 8. Mineral Deposits | 2 | 500-1000 | O(n) |
| 9. Dormant Gates | 2 | 300-500 | O(n) |
| 10. Precursor | 1-3 | 50-100 | O(1) |
| 11. Mirror Universe | 5-10 | 2000-3000 | O(n) |
| 12. NPC Generation | 2 | 200-500 | O(n) |
| 13. Market Events | 1 | 100-200 | O(n*m) |
| 14. Finalization | 1 | 10 | O(1) |
| **TOTAL** | **~30-35** | **8-18 seconds** | **Mixed** |

---

## Current System Bottlenecks

1. **Warp Gate Generation** (Phase 5)
   - Worst case O(n²) with spatial queries
   - Takes 2-5 seconds alone
   - Mitigation: Batch queries, spatial indexing

2. **Outer Star Generation** (Phase 6)
   - 3,000 POI bulk insert is slow with large datasets
   - Time: 1-3 seconds

3. **Planetary System Generation** (Phase 7)
   - 23,000+ planet inserts
   - Time: 1-2 seconds

4. **Trading Hub Inventory Stocking** (within Phase 4)
   - TradingHubGenerator computes pricing for each hub/mineral combo
   - O(n * 27 minerals) pricing calculations

---

## Proposed V2 Architecture

### Design Principles

1. **Discrete Phases**: Each generation step is independently callable
2. **Measurable Progress**: User sees what's happening at each stage
3. **Resumable**: Stop and restart without duplication
4. **Versioned**: V2 lives alongside V1 (GalaxyGenerationV2Service)
5. **Admin-Configurable**: Size tier admin-only; gates/supernode config in game_config.php
6. **Distance-Aware**: Gate counts decrease with distance from core
7. **Supernode Config**: Definable requirements with fallback chains

---

## V2 Implementation Steps

### New Service: `GalaxyGenerationV2Service`

```php
class GalaxyGenerationV2Service {
    public function createGalaxyOnly(GalaxySizeTier $tier): Galaxy
    public function createSectors(Galaxy $galaxy): int
    public function createCoreStars(Galaxy $galaxy): int
    public function createOuterStars(Galaxy $galaxy): int
    public function createWarpLanes(Galaxy $galaxy): int
    public function createPlayer(Galaxy $galaxy, User $user): Player
    public function createSupernodeForPlayer(Player $player): PointOfInterest
    public function populateConnectedSystems(Galaxy $galaxy): int
}
```

### New Configuration: `config/game_config.php` additions

```php
'supernode' => [
    'requirements' => [
        'star_size' => ['large', 'giant'],  // Large or giant stars
        'services' => [
            'shipyard' => true,       // Must have shipyard
            'salvage_yard' => true,   // Must have salvage yard
            'cartography' => true,    // Must have star chart vendor
            'trading_hub' => true,    // Must have trading hub
            'bar' => false,           // Bar is optional (future)
        ],
        'habitable_worlds' => 2,      // Min 2 habitable planets
        'gas_giant_mining' => true,   // Must have gas giant with mining
        'moons' => 1,                 // Min 1 habitable moon with mining
        'defenses' => true,           // Orbital defenses around capital world
    ],
    'fallback_chain' => [
        // If supernode can't be created with all requirements, try reduced versions
        'strict',       // All requirements
        'standard',     // Remove bar requirement
        'minimal',      // Only core services + 1 habitable
    ],
],

'gate_distribution' => [
    'core_region' => [
        'max_gates' => 8,              // Max gates in core
        'probability_by_inhabitance' => [
            'inhabited' => 0.95,       // 95% inhabited systems have gates
            'uninhabited' => 0.0,      // No gates for uninhabited
        ],
    ],
    'middle_region' => [
        'max_gates' => 5,              // Fewer gates in middle
        'probability_by_inhabitance' => [
            'inhabited' => 0.70,       // 70% inhabited have gates
            'uninhabited' => 0.0,
        ],
    ],
    'outer_region' => [
        'max_gates' => 2,              // Few gates in frontier
        'probability_by_inhabitance' => [
            'inhabited' => 0.40,       // 40% inhabited have gates
            'uninhabited' => 0.0,      // Uninhabited are dead ends
        ],
    ],
    'random_gates' => [
        'enabled' => true,
        'min_per_system' => 1,         // At least 1 gate for connectivity
        'distance_based_decay' => 0.95, // 5% reduction per distance band
    ],
],
```

### Phase 1: Galaxy Only
**Scope**: Create empty galaxy record with sectors
**Input**: Size tier
**Output**: Galaxy with Sector grid, zero POIs

**Complexity**:
- Time: O(s) where s = sectors (16-100 depending on tier)
- Queries: ~2 (INSERT galaxy, batch INSERT sectors)
- Estimate: 10-50ms
- **Cyclomatic**: 2

---

### Phase 2: Core & Outer Stars
**Scope**: Generate all star POIs (core + outer)
**Input**: Galaxy
**Output**: ~3,000 POIs in appropriate regions

**Complexity**:
- Time: O(n log n) where n = total POIs
- Queries: ~2
- Estimate: 2-4 seconds
- **Cyclomatic**: 8

---

### Phase 3: Warp Lanes
**Scope**: Generate warp gates with distance-based logic
**Input**: Galaxy with all POIs
**Output**: ~1,500-2,000 warp gates

**Distance-Based Algorithm**:
```
For each inhabited POI:
  distance_band = calculate_distance_from_core(poi)

  if distance_band == 'core':
    max_gates = 8
    gate_probability = 0.95
  elif distance_band == 'middle':
    max_gates = 5
    gate_probability = 0.70
  else:  # outer
    max_gates = 2
    gate_probability = 0.40

  actual_gates = random(1, max_gates) with probability gate_probability
  connect_to_nearest_N(poi, actual_gates)
```

**Complexity**:
- Time: O(n * log n) with spatial indexing (KD-tree nearby search)
- Queries: ~3-5 (spatial selects + batch insert)
- Estimate: 3-6 seconds
- **Cyclomatic**: 14

---

### Phase 4: Player & Supernode
**Scope**: Create player and their starting supernode
**Input**: Galaxy
**Output**: Player with origin supernode

**Supernode Location Algorithm**:
1. Filter POIs by size (large/giant stars in core region)
2. Check requirements vs. fallback chain
3. Find best match meeting highest fallback level
4. If no match found, create one synthetically

**Complexity**:
- Time: O(n) to evaluate POIs, O(1) for player creation
- Queries: ~5-10
- Estimate: 500ms - 2 seconds
- **Cyclomatic**: 12

---

### Phase 5: System Population
**Scope**: Populate remaining systems with services/planets/minerals
**Input**: Galaxy
**Output**: Full star systems

**Complexity**:
- Time: O(n) where n = uninhabited POIs
- Queries: ~3-5 (per batch)
- Estimate: 3-5 seconds
- **Cyclomatic**: 10

---

## V2 Total Analysis

| Phase | Input | Output | Queries | Time (ms) | O(n) |
|-------|-------|--------|---------|-----------|------|
| 1. Galaxy + Sectors | Tier | Galaxy | 2 | 10-50 | O(1) |
| 2. Stars | Galaxy | 3,000 POIs | 2 | 2000-4000 | O(n log n) |
| 3. Warp Lanes | POIs | Gates | 3-5 | 3000-6000 | O(n log n) |
| 4. Player+Supernode | Galaxy | Player | 5-10 | 500-2000 | O(n) |
| 5. System Pop | Galaxy | Full systems | 3-5 | 3000-5000 | O(n) |
| **V2 TOTAL** | | | **15-25** | **8.5-17s** | **Mixed** |

---

## Command Line Interface (V2)

```bash
# Create galaxy with just structure
php artisan galaxy:create-v2 \
  --tier=medium \
  --name="Andromeda Prime"

# Add stars to existing galaxy
php artisan galaxy:populate-v2 \
  --galaxy=<uuid> \
  --step=stars

# Add warp lanes
php artisan galaxy:populate-v2 \
  --galaxy=<uuid> \
  --step=gates

# Create player starting point
php artisan galaxy:populate-v2 \
  --galaxy=<uuid> \
  --step=player \
  --user=<user_id>

# Complete population (all remaining steps)
php artisan galaxy:populate-v2 \
  --galaxy=<uuid> \
  --step=all

# With size restriction (admin only)
php artisan galaxy:create-v2 \
  --tier=medium \
  --as-admin  # Validates user is admin
```

---

## Size Tier Admin Restriction

Add to `CreateGalaxyCommand`:

```php
protected function authorize(): void
{
    if (! auth()->check() || ! auth()->user()->is_admin) {
        throw new UnauthorizedHttpException(null, 'Only admins can specify galaxy size tier');
    }
}
```

---

## Benefits of V2 Architecture

✅ **Observable Progress**: User/admin sees each step clearly
✅ **Resumable**: Crash at step 3? Restart from step 4
✅ **Testable**: Each phase is independently unit-testable
✅ **Versioned**: Old V1 still works, V2 is experimental
✅ **Configurable**: Gate counts, supernodes, services all in config
✅ **Performant**: Batch operations, optimized queries
✅ **Distance-Aware**: Core gets more gates, outer is sparse
✅ **Admin-Only Sizing**: No user-set tier bloat

---

## Recommendation

**Phase 1**: Create V2 scaffolding and GalaxyGenerationV2Service
**Phase 2**: Implement discrete phase methods in GalaxyGenerationV2Service
**Phase 3**: Create V2 console commands
**Phase 4**: Add supernode configuration to game_config.php
**Phase 5**: Implement distance-based warp lane generation
**Phase 6**: Testing & documentation

---

## Next Steps for User Approval

1. ✅ Do you approve the V2 architecture approach?
2. ✅ Should size tier selection be admin-only?
3. ✅ Are the supernode requirement categories correct?
4. ✅ Do the distance bands (core/middle/outer) match your vision?
5. ✅ Should gate probability be configurable per band?

