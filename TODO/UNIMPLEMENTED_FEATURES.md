# Unimplemented Features - Space Wars 3002

**Last Updated**: 2026-03-06
**Review Method**: Comprehensive documentation audit across all design docs, API references, and implementation guides

---

## Overview

This document tracks all features that have been documented in design specs and API references but have not yet been implemented in code. Items are organized by priority tier and include references to source documentation.

**Summary Counts**:
- ✅ Fully Implemented: 3 major systems
- 📋 Documented but Unimplemented: 11 major feature areas
- 🔄 Partially Implemented: 3 systems
- ⏳ Planned (Design Only): 8+ future enhancements

---

## ✅ Fully Implemented Systems

### 1. Flotilla & Fleet Mechanics
**Status**: PRODUCTION READY
**Completion**: Phase 1-6 (100%)
**Documentation**: `docs/guides/FLOTILLA_IMPLEMENTATION_GUIDE.md`, `docs/guides/FLOTILLA_PHASE_1_IMPLEMENTATION_SUMMARY.md`
**Code**: `app/Models/Flotilla.php`, `app/Services/Flotilla/*`, `app/Http/Controllers/Api/FlotillaController.php`

- ✅ Database schema (2 migrations)
- ✅ 4 services: FlotillaService, FlotillaMovementService, FlotillaCombatService, FlotillaSalvageService
- ✅ 6 REST API endpoints
- ✅ 52 unit & feature tests
- ✅ Combat system integration
- ✅ Movement with fuel penalties
- ✅ Multi-ship combat mechanics
- ✅ Post-battle salvage (cargo/components XOR choice)

---

### 2. Job Board Contract System
**Status**: PRODUCTION READY
**Completion**: Phase 1a-1e (100%)
**Documentation**: `docs/guides/JOB_BOARD_IMPLEMENTATION_GUIDE.md`, `docs/guides/JOB_BOARD_PHASE_1_IMPLEMENTATION_SUMMARY.md`
**Code**: `app/Models/Contract.php`, `app/Services/*Contract*`, `app/Http/Controllers/Api/ContractController.php`

- ✅ Database schema (3 migrations, audit trail)
- ✅ 4 services: ContractService, ContractGenerationService, ReputationService, ContractExpiryService
- ✅ 5 REST API endpoints
- ✅ 41 unit & feature tests
- ✅ Reputation system (0-100 scale with tiers)
- ✅ Contract lifecycle (POSTED → ACCEPTED → COMPLETED → FAILED)
- ✅ Scheduled expiry processing
- ✅ Transport & supply contract types

---

### 3. Economy Overhaul
**Status**: PRODUCTION READY
**Completion**: Phase 0-3 (100%)
**Documentation**: `docs/guides/ECONOMICS_GUIDE.md`, `docs/ECONOMY_OVERHAUL_IMPLEMENTATION_PLAN.md`
**Code**: `app/Models/Commodity*`, `app/Services/Economy/*`, `app/Console/Commands/EconomyTickCommand.php`

- ✅ Immutable commodity ledger
- ✅ Supply/demand-based pricing
- ✅ Mining extraction with resource deposits
- ✅ Economic shock system with exponential decay
- ✅ Construction job system
- ✅ Trade locking (TOCTOU fix)
- ✅ Tick-based processing
- ✅ Economy tick command coordination

---

## 🔄 Partially Implemented Systems

### 1. Combat System
**Status**: PARTIAL (basic combat works, flotilla combat partially integrated)
**Documentation**: `docs/api/combat.md`
**Code**: `app/Services/Combat/CombatService.php`, `app/Http/Controllers/Api/CombatController.php`

**Implemented**:
- ✅ Single-ship vs pirate combat
- ✅ Basic damage calculation
- ✅ XP rewards
- ✅ Cargo loss on defeat
- ✅ Pirate encounter generation

**Missing**:
- [ ] Complete flotilla combat integration (routing works, but full simulation incomplete)
- [ ] Mid-combat ship destruction with flagship succession
- [ ] Smart pirate targeting (weakest ship first)
- [ ] Escape attempts based on slowest ship speed
- [ ] Surrender handling for multi-ship flotillas
- [ ] Post-battle salvage mechanics (documented but not fully implemented)

---

### 2. Pirate Faction System
**Status**: PARTIAL (data structures exist, features incomplete)
**Documentation**: `docs/api/pirate-factions.md`
**Code**: `app/Models/PirateFaction.php`, `app/Http/Controllers/Api/PirateFactionController.php`

**Implemented**:
- ✅ PirateFaction, PirateCaptain, PirateFleet models
- ✅ 3-4 API endpoints for listing factions/captains
- ✅ Global reputation tracking (based on kills)

**Missing**:
- [ ] Per-faction reputation tracking (currently all factions share single reputation)
- [ ] Faction-specific reputation modifiers
- [ ] Dynamic pirate encounter difficulty scaling by faction
- [ ] Reputation improvement mechanics (only degradation via kills)
- [ ] Diplomatic/faction interaction endpoints
- [ ] Faction-specific combat bonuses/penalties
- [ ] Territory control mechanics

---

### 3. NPC Traders
**Status**: SCAFFOLDED (command exists, not integrated)
**Documentation**: `docs/guides/ECONOMICS_GUIDE.md` (Phase 9)
**Code**: `app/Console/Commands/NpcTraderTickCommand.php`

**Implemented**:
- ✅ Tick command structure
- ✅ Basic inventory mutation via HubInventoryMutationService

**Missing**:
- [ ] Continuous NPC trading cycles integration
- [ ] NPC fleet management
- [ ] NPC reputation dynamics
- [ ] Trade route learning (NPCs learning profitable routes)
- [ ] NPC buying/selling decisions
- [ ] Inter-NPC negotiations

---

## 📋 Critical Priority (Blocks Other Features)

### 1. Colonies System
**Status**: NOT STARTED
**Documentation**: `docs/api/colonies.md` (63KB comprehensive spec)
**Estimated LOC**: 2,500-3,000
**Estimated Hours**: 40-50

**What's Needed**:
- [ ] **Database Schema** (6+ tables)
  - `colonies` - primary colony records
  - `colony_buildings` - individual building instances
  - `building_types` - building templates
  - `production_queues` - ship/resource production
  - `colony_population` - population tracking
  - `colony_defenses` - orbital/ground defenses

- [ ] **Models**
  - `Colony` with relationships to buildings, production, defenses
  - `ColonyBuilding` with upgrade/repair mechanics
  - `BuildingType` with production formulas
  - `ProductionQueue` for ship construction
  - `ColonyDefense` for fortification

- [ ] **Services** (4-5 services)
  - `ColonyService` - establishment, abandonment, management
  - `ColonyProductionService` - queue management, completion
  - `ColonyBuildingService` - construction, upgrade, demolition
  - `ColonyDefenseService` - defense calculations
  - `ColonyMiningService` - automated resource extraction

- [ ] **API Endpoints** (15+ documented)
  - `GET /api/players/{uuid}/colonies` - list colonies
  - `POST /api/players/{uuid}/colonies` - establish
  - `GET /api/colonies/{uuid}` - get details
  - `PUT /api/colonies/{uuid}` - update
  - `DELETE /api/colonies/{uuid}` - abandon
  - Building management endpoints (5+)
  - Production queue endpoints (3+)
  - Defense/fortification endpoints (3+)
  - Mining operations endpoints (3+)

- [ ] **Business Logic**
  - Population growth calculations
  - Resource production formulas
  - Development tier progression
  - Building construction/upgrade time
  - Defense effectiveness calculations
  - Assault/conquest mechanics

- [ ] **Integration Points**
  - Mining system (automated extraction)
  - Ship construction (production queue)
  - Economy system (commodity flow)
  - Combat system (base defense)
  - Trading hubs (colony markets)

**Source Docs**: `docs/api/colonies.md`

---

### 2. Fog-of-War & Knowledge System
**Status**: NOT STARTED (API endpoints documented, backend not implemented)
**Documentation**: `docs/design/fog-of-war.md` (1000+ lines)
**Estimated LOC**: 1,500-2,000
**Estimated Hours**: 25-30

**What's Needed**:
- [ ] **Database Schema** (2-3 new tables)
  - `player_knowledge_map` - knowledge state per player per system
  - `knowledge_sources` - track source of knowledge (visit, chart, rumor, etc.)
  - `knowledge_decay_schedule` - expiration tracking

- [ ] **Models**
  - `PlayerKnowledge` with knowledge_level (UNKNOWN=0 → VISITED=4)
  - Source tracking (visit, spawn, warp_lane, sensor, baseline, chart, rumor, scan)

- [ ] **Services**
  - `KnowledgeService` - track player discoveries
  - `KnowledgeFreshnessService` - manage 7-day decay
  - `SensorRangeService` - calculate detection ranges

- [ ] **API Endpoints** (10 documented)
  - `GET /api/players/{uuid}/knowledge-map` - full fog-of-war map
  - `GET /api/players/{uuid}/location` - current location details
  - `GET /api/players/{uuid}/nearby-systems` - sensor range systems
  - `GET /api/players/{uuid}/scan-local` - all POIs in range
  - `GET /api/players/{uuid}/local-bodies` - orbital bodies + defenses
  - `POST /api/players/{uuid}/scan-system` - progressive scan
  - `GET /api/players/{uuid}/scan-results/{poiUuid}` - cached scan data
  - `GET /api/players/{uuid}/exploration-log` - scan history
  - `POST /api/players/{uuid}/bulk-scan-levels` - batch query
  - `GET /api/players/{uuid}/system-data/{poiUuid}` - filtered by level

- [ ] **Knowledge Levels & Progression**
  - UNKNOWN (0) - hidden in fog
  - DETECTED (1) - coordinates only, 0.3 opacity
  - BASIC (2) - name, star type, inhabited, planet count, 0.5 opacity
  - SURVEYED (3) - full services, 0.8 opacity
  - VISITED (4) - complete knowledge, 1.0 opacity (permanent)

- [ ] **Knowledge Decay System**
  - Freshness field (0.0-1.0) for time-based degradation
  - Sources that never decay: visit, spawn, warp_lane, scan
  - Sources that decay (7 days): chart, rumor
  - Sensor range source: real-time (no persistence)

- [ ] **UI Response Fields**
  - Visibility rules per knowledge level
  - Freshness multipliers
  - Pirate warning data
  - Scan level information

**Source Docs**: `docs/design/fog-of-war.md`

---

### 3. Advanced Combat Features (for Flotillas)
**Status**: PARTIAL (needs completion to work with flotillas)
**Documentation**: `docs/design/flotilla.md` (Combat Integration section)
**Estimated LOC**: 300-500
**Estimated Hours**: 5-8

**What's Needed**:
- [ ] Mid-combat ship destruction mechanics
  - Remove destroyed ship from flotilla
  - Redistribute remaining damage?
  - Update combat totals

- [ ] Flagship succession on destruction
  - Promote next largest ship by hull
  - Update flotilla flagship_ship_id
  - Atomic transaction

- [ ] Smart pirate targeting
  - Select weakest ship (lowest hull) for damage
  - Update selection as ships are destroyed
  - Configuration in game_config.php

- [ ] Escape attempts
  - Formula: 40% base × (slowest_warp / avg_pirate_speed)
  - Per-ship escape probability
  - Success/failure handling

- [ ] Surrender handling (multi-ship)
  - Cargo lost from all ships (70%)
  - Atomic update to all ships
  - Proper XP/credit handling

- [ ] Post-battle salvage (XOR choice)
  - Cargo recovery (70% of destroyed ships)
  - Component recovery (with escalating loss)
  - Pirate loot (always available, separate)
  - Distribution logic to surviving ships

**Source Docs**: `docs/design/flotilla.md` (sections 14-15)

---

## 🔶 High Priority (Core Gameplay)

### 4. Enhanced Pirate Faction System
**Status**: PARTIAL (basic structure exists)
**Documentation**: `docs/api/pirate-factions.md`
**Estimated LOC**: 800-1,200
**Estimated Hours**: 12-18

**What's Needed**:
- [ ] **Per-Faction Reputation Tracking**
  - New table: `player_faction_reputations` with per-faction scores
  - Replace global reputation calculation
  - Track: completed bounties, kills, pirate loot recovered

- [ ] **Reputation Tiers & Effects**
  - ALLIED (≥100): Safe passage, trade opportunities, intelligence sharing
  - FRIENDLY (50-99): Reduced encounter rate, better prices
  - NEUTRAL (0-49): Standard pirate behavior
  - UNFRIENDLY (-1 to -49): Increased encounter rate, higher difficulty
  - HOSTILE (-50 to -99): Frequent ambushes, reinforcements called
  - HATED (≤-100): Elite fleets, bounty on head, no escape

- [ ] **Reputation Mechanics**
  - Base reputation: 0 (neutral)
  - Killing pirate captain: varies by captain reputation
  - Completing pirate bounties: +10-50
  - Joining faction: special ops access
  - Reputation improvement paths (not just degradation)

- [ ] **API Endpoints** (4 documented, expand as needed)
  - `GET /api/galaxies/{uuid}/pirate-factions` - list factions
  - `GET /api/pirate-factions/{uuid}` - faction detail
  - `GET /api/pirate-factions/{uuid}/captains` - list captains
  - `GET /api/players/{uuid}/pirate-reputation` - player reputation
  - NEW: `POST /api/pirate-factions/{uuid}/join` - faction membership
  - NEW: `GET /api/pirate-factions/{uuid}/bounties` - available contracts

- [ ] **Faction-Specific Mechanics**
  - Different encounter difficulties per faction
  - Faction-based pirate fleet compositions
  - Territory control mechanics
  - Faction wars (future)

**Source Docs**: `docs/api/pirate-factions.md`

---

### 5. Scanning & Exploration System
**Status**: NOT STARTED (API endpoints documented, implementation missing)
**Documentation**: `docs/design/fog-of-war.md` (sections 6-10)
**Estimated LOC**: 1,000-1,500
**Estimated Hours**: 15-20

**What's Needed**:
- [ ] **Progressive Scan Levels** (0-4)
  - Level 0: Unscanned (nothing revealed)
  - Level 1: Basic composition (star class, planet count)
  - Level 2: Detailed data (star specifics, anomalies)
  - Level 3: Mineral composition (mining values)
  - Level 4: Hidden structures (pirate bases, artifacts)

- [ ] **Scan Mechanics**
  - Time required per scan level (increases exponentially)
  - Sensor level requirements per level
  - Scan failure probability
  - Cached results (prevent re-scanning overhead)

- [ ] **Anomaly Detection**
  - Hidden pirate bases
  - Artifact sites
  - Unstable phenomena
  - Resource anomalies

- [ ] **Services**
  - `ScanService::performScan($player, $poi, $force)` - execute scan
  - `ScanService::cacheResults($poi, $level, $data)` - store results
  - `ScanService::getScanLevel($player, $poi)` - current progress
  - `ScanService::revealNext($player, $poi)` - advance to next level

- [ ] **API Endpoints**
  - `POST /api/players/{uuid}/scan-system` - start/continue scan
  - `GET /api/players/{uuid}/scan-results/{poiUuid}` - retrieve cached data
  - `GET /api/players/{uuid}/exploration-log` - scan history
  - `POST /api/players/{uuid}/bulk-scan-levels` - batch query
  - `GET /api/players/{uuid}/system-data/{poiUuid}` - filtered by scan level

- [ ] **Data Filtering by Level**
  - Level 1+: Temperature ranges, star classes
  - Level 2+: Exact temperatures, luminosity, goldilocks zones
  - Level 3+: Mineral deposits, extraction rates
  - Level 4+: Hidden structures, anomaly details

**Source Docs**: `docs/design/fog-of-war.md` (sections 5-10)

---

### 6. Enhanced Star Chart System
**Status**: PARTIAL (basic chart purchase exists, advanced features missing)
**Estimated LOC**: 400-600
**Estimated Hours**: 6-10

**What's Needed**:
- [ ] **Chart Types**
  - Local charts (current sector)
  - Regional charts (multi-sector coverage)
  - Galactic charts (complete galaxy map)

- [ ] **Dynamic Pricing**
  - Base price: varies by chart type
  - Coverage area multiplier: exponential
  - Freshness multiplier: older charts cost less
  - Demand-based pricing: scarce systems cost more

- [ ] **Chart Expiration & Decay**
  - Age tracking (creation_date)
  - Freshness degradation (similar to knowledge decay)
  - Refresh mechanics (re-purchase to get current data)
  - Knowledge ceiling: always at least DETECTED level

- [ ] **Bulk Chart Purchases**
  - Nearby systems bundle
  - Regional coverage packs
  - Discount pricing for bundles

- [ ] **Chart Sourcing**
  - Stellar cartographers generate charts
  - NPC traders sell charts
  - Exploration discovery grants free charts
  - Player-to-player chart trading (future)

**Source Docs**: `docs/api/galaxies.md`, `docs/guides/learning-guide.md`

---

## 🟡 Medium Priority (Quality of Life)

### 7. Living NPC Trader Economy
**Status**: SCAFFOLDED (tick command exists, integration missing)
**Documentation**: `docs/guides/ECONOMICS_GUIDE.md` (Phase 9)
**Estimated LOC**: 600-800
**Estimated Hours**: 10-12

**What's Needed**:
- [ ] **NPC Trading Mechanics**
  - NPC decision-making: when/what to buy-sell
  - Profit-seeking behavior: follow price arbitrage
  - Route learning: NPCs remember profitable paths
  - Cargo management: per-NPC fleet inventory

- [ ] **Integration with Tick System**
  - Run NPC trading cycle during EconomyTickCommand
  - Update hub inventories via HubInventoryMutationService
  - Record transactions in ledger (optional)
  - Generate market volatility from NPC activity

- [ ] **NPC Fleet Management**
  - Assignment of NPC ships to trading routes
  - Movement scheduling
  - Fuel management
  - Cargo handling

- [ ] **Reputation Dynamics**
  - NPC reputation with trading hubs
  - Preferred suppliers/buyers
  - Blacklisting mechanics
  - Relationship-based pricing

**Source Docs**: `docs/guides/ECONOMICS_GUIDE.md` (Phase 9)

---

### 8. Mirror Universe / High-Risk Zones
**Status**: PARTIAL (configuration exists, mechanics incomplete)
**Documentation**: `docs/guides/CLAUDE.md` (Mirror Universe section)
**Estimated LOC**: 400-600
**Estimated Hours**: 6-10

**What's Needed**:
- [ ] **Mirror Universe System**
  - One mirror gate per galaxy (already placed)
  - Sensor level 5+ requirement for detection
  - 2x resource spawns in mirror dimension
  - 3x rare mineral spawn rate
  - 2x pirate difficulty scaling
  - 3x pirate fleet sizes
  - 24-hour cooldown before return

- [ ] **Services**
  - `MirrorUniverseService::detectGate($player)` - sensor-gated detection
  - `MirrorUniverseService::enterMirror($player)` - crossing gates
  - `MirrorUniverseService::canReturn($player)` - cooldown checking
  - `MirrorUniverseService::exitMirror($player)` - return logic

- [ ] **Mechanics**
  - Automatic difficulty scaling for encounters
  - Resource mutation formulas
  - Cooldown tracking per player
  - Risk/reward communication to player

- [ ] **API Endpoints**
  - `POST /api/players/{uuid}/warp-gates/{gateUuid}/enter-mirror` - crossing
  - `GET /api/players/{uuid}/mirror-status` - cooldown/eligibility
  - `POST /api/players/{uuid}/mirror-exit` - forced return

**Source Docs**: `docs/guides/CLAUDE.md`

---

### 9. Colony Defenses & Orbital Systems
**Status**: NOT STARTED (complex integration needed)
**Documentation**: `docs/design/fog-of-war.md` (local-bodies endpoint)
**Estimated LOC**: 1,200-1,800
**Estimated Hours**: 18-25

**What's Needed**:
- [ ] **Defense Structure Types**
  - Orbital defense platforms (player-built)
  - System defense turrets (cannons, lasers, missiles)
  - Fighter squadrons (hangar-based)
  - Planetary shields
  - Magnetic mines
  - Colony garrison units

- [ ] **Models & Database**
  - `OrbitDefenseStructure` - orbital platforms
  - `SystemDefenseUnit` - turrets/cannons
  - `DefenseConfiguration` - per-colony settings
  - Relationships to Colony, PointOfInterest

- [ ] **Calculation Services**
  - `DefenseCalculationService::calculateTotalDamage($colony)` - per-round damage
  - `DefenseCalculationService::calculateThreatLevel($colony)` - threat rating
  - `DefenseCalculationService::simulateAttack($attacker, $colony)` - combat prediction

- [ ] **Threat Level System**
  - None: 0 damage per round
  - Minimal: 1-50 damage
  - Moderate: 51-150 damage
  - Heavy: 151-400 damage
  - Fortress: 401+ damage

- [ ] **API Response Integration**
  - `orbital_presence` (always visible) - physical structures
  - `defensive_capability` (sensor 5+ only) - detailed breakdown
  - Breakdown includes:
    - Orbital defense platforms count
    - System defenses count
    - Fighter squadrons size
    - Colony garrison size
    - Defense building count
    - Magnetic mines count
    - Planetary shield HP
    - Total damage per round
    - Threat level label

- [ ] **Combat Integration**
  - Colony assault mechanics
  - Defense effectiveness in combat
  - Damage application to shields/buildings
  - Repair mechanics

**Source Docs**: `docs/design/fog-of-war.md` (local-bodies endpoint sections)

---

## 🔵 Low Priority (Future Expansion)

### 10. Victory Condition Tracking
**Status**: CONFIG ONLY (defined in code, tracking not implemented)
**Estimated LOC**: 300-400
**Estimated Hours**: 4-6

**What's Needed**:
- [ ] **Victory Paths** (4 conditions defined in `config/game_config.php`)
  - Merchant Empire: Accumulate 1 billion credits
  - Colonization: Control >50% galactic population
  - Conquest: Control >60% star systems
  - Pirate King: Seize >70% outlaw network

- [ ] **Services**
  - `VictoryService::checkVictoryCondition($player, $condition)`
  - `VictoryService::getPlayerProgress($player)` - all paths
  - `VictoryService::isVictory($player)` - boolean check

- [ ] **Tracking**
  - Player victory status in database
  - Progress metrics per player
  - Leaderboard positions

- [ ] **API Endpoints**
  - `GET /api/players/{uuid}/victory-progress` - status on all paths
  - `GET /api/leaderboards/victory` - victory leaderboard

**Source Docs**: `docs/guides/CLAUDE.md`, `config/game_config.php`

---

### 11. Advanced Job Board Contracts (Phase 2+)
**Status**: DESIGN ONLY (Phase 1 complete, Phase 2+ not started)
**Documentation**: `docs/design/job_board_contract_system.md` (Future expansions section)
**Estimated LOC per Phase**: 1,000-1,500
**Estimated Hours per Phase**: 15-20

**Phase 2 Contract Types**:
- [ ] Escort Contracts
  - Protect convoy from point A to B
  - Defend against random encounters
  - Reputation with convoy owner

- [ ] Bounty Contracts
  - Hunt specific pirate captains
  - Proof of kill requirement
  - Faction reputation gains

- [ ] Exploration Contracts
  - Scout unexplored systems
  - Report back findings
  - Chart generation rewards

- [ ] Player-Posted Contracts
  - Players posting contracts for other players
  - Commodity hauling between players
  - Mission reputation tracking

**Advanced Features (Phase 3+)**:
- [ ] Black Market Job Boards
  - Illegal contract types
  - Higher risks/rewards
  - Reputation with underworld

- [ ] Pirate Trap Contracts
  - Deceptive job board listings
  - Ambush scenarios
  - Faction-specific outcomes

- [ ] Contract Insurance
  - Payment upfront for coverage
  - Refund on mission failure
  - Risk assessment

**Source Docs**: `docs/design/job_board_contract_system.md`

---

### 12. Planned Fleet & Expansion Features
**Status**: DESIGN ONLY (future enhancements)
**Documentation**: `docs/design/flotilla.md` (Future Enhancements section)

**What's Documented**:
- [ ] **NPC Escort Hiring**
  - Mercenary ship recruitment
  - Trustability ratings
  - Escort composition management
  - Contract terms

- [ ] **Colony Ship Escorts**
  - Flotillas required for safe transport
  - Deployment mechanics
  - Protection mechanics

- [ ] **Inter-ship Cargo Transfer**
  - Docked ship cargo movement
  - Load balancing mechanics
  - Atomic transfers

- [ ] **Flotilla Combat Orders**
  - Formation patterns (defensive, aggressive, balanced)
  - Per-ship role assignment
  - Tactical depth

- [ ] **Multi-Player Flotillas**
  - Player-to-player fleet cooperation
  - Shared objectives
  - High complexity (deferred)

**Source Docs**: `docs/design/flotilla.md` (section: Future Enhancements)

---

## 📊 Implementation Roadmap Summary

### Phase A: Critical Path (Must Have)
**Estimated Total**: ~120-150 hours

1. **Colonies System** (40-50h)
2. **Fog-of-War / Knowledge System** (25-30h)
3. **Advanced Combat Features** (5-8h)
4. **Enhanced Pirate Faction System** (12-18h)
5. **Scanning & Exploration** (15-20h)

### Phase B: Core Gameplay (Should Have)
**Estimated Total**: ~40-50 hours

6. **Enhanced Star Chart System** (6-10h)
7. **Living NPC Trader Economy** (10-12h)
8. **Mirror Universe Mechanics** (6-10h)
9. **Colony Defenses & Orbital Systems** (18-25h)

### Phase C: Polish & Expansion (Nice to Have)
**Estimated Total**: ~15-20 hours

10. **Victory Condition Tracking** (4-6h)
11. **Advanced Job Board Contracts** (Phase 2+, 15-20h per phase)
12. **Future Fleet Features** (deferred, multi-phase)

---

## Dependencies & Blocking

```
Colonies System
  ├─ blocks: Mining operations (automated)
  ├─ blocks: Production queues (ship building)
  ├─ blocks: Victory condition (colonization path)
  └─ requires: Economy system (✅ done)

Fog-of-War / Knowledge System
  ├─ blocks: Frontend star map rendering
  ├─ blocks: Exploration gameplay
  └─ requires: Basic travel (✅ done)

Advanced Combat Features
  ├─ blocks: Flotilla combat completeness
  ├─ requires: Flotilla system (✅ done)
  └─ requires: Basic combat (✅ done)

Scanning & Exploration System
  ├─ requires: Fog-of-war system
  └─ blocks: Progressive discovery gameplay

Enhanced Pirate Faction System
  ├─ requires: Pirate system (partially ✅)
  └─ blocks: Faction reputation gameplay

Colony Defenses & Orbital Systems
  ├─ requires: Colonies system
  └─ blocks: Colony assault/PvP

Victory Condition Tracking
  ├─ requires: Colonies (colonization path)
  ├─ requires: Pirate system (pirate king path)
  ├─ requires: Trading (merchant path)
  └─ requires: Star system control (conquest path)
```

---

## Tracking

Use this file as a reference for:
- Feature status updates (mark with ✅ when completed)
- Time estimation for sprint planning
- Dependency checking before starting new features
- Reference documentation per feature

When implementing a feature, update the relevant section status from NOT STARTED → IN PROGRESS → COMPLETED.

