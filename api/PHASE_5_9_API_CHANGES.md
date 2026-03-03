# Phase 5-9 Implementation - API Changes & New Endpoints

**Implementation Date:** March 2-3, 2026
**Status:** Production Ready
**Version:** 2.0.0

## Summary

This document details all new API endpoints and changes introduced in Phases 5-9 of the "Single Commodity Market + Personas" implementation.

## Table of Contents

1. [New Endpoints by Phase](#new-endpoints-by-phase)
2. [Modified Endpoints](#modified-endpoints)
3. [Data Models](#data-models)
4. [Error Handling](#error-handling)
5. [Feature Flags](#feature-flags)
6. [Migration Guide](#migration-guide)

---

## New Endpoints by Phase

### Phase 5: Crew Profiles

#### GET /api/galaxies/{uuid}/crew/available

Get list of available crew members at current POI.

**Params:**
- `poi_uuid` (required, query) - Point of interest where crew is available

**Response:**
```json
{
  "success": true,
  "available_crew": [
    {
      "uuid": "550e8400-e29b-41d4-a716-446655440000",
      "name": "Lt. Kovacs",
      "role": "tactical_officer",
      "alignment": "neutral",
      "reputation": 25,
      "shady_actions": 5,
      "traits": {
        "negotiation": 0.75,
        "intimidation": 0.68,
        "charm": 0.55,
        "technical_skill": 0.42
      },
      "backstory": "Former military officer seeking redemption...",
      "hire_cost": 5000
    }
  ],
  "total_available": 24
}
```

**Availability:** Feature flag `crew_profiles`
**Added:** March 2, 2026

---

#### POST /api/ships/{uuid}/crew/hire

Hire a crew member to active ship.

**Body:**
```json
{
  "crew_uuid": "550e8400-e29b-41d4-a716-446655440000",
  "negotiate_salary": false
}
```

**Response:**
```json
{
  "success": true,
  "message": "Lt. Kovacs hired successfully",
  "crew_member": {
    "uuid": "550e8400-e29b-41d4-a716-446655440000",
    "name": "Lt. Kovacs",
    "role": "tactical_officer",
    "alignment": "neutral"
  },
  "ship_persona_updated": true,
  "credits_spent": 5000
}
```

**Errors:**
- 400 - Insufficient credits
- 404 - Crew not found
- 409 - Ship at max crew capacity

**Availability:** Feature flag `crew_profiles`
**Added:** March 2, 2026

---

#### POST /api/ships/{uuid}/crew/dismiss/{crewUuid}

Dismiss crew member from ship.

**Response:**
```json
{
  "success": true,
  "message": "Lt. Kovacs dismissed",
  "credits_returned": 2500
}
```

**Availability:** Feature flag `crew_profiles`
**Added:** March 2, 2026

---

#### POST /api/ships/{uuid}/crew/transfer

Transfer crew member to another ship.

**Body:**
```json
{
  "crew_uuid": "550e8400-e29b-41d4-a716-446655440000",
  "target_ship_uuid": "660e8400-e29b-41d4-a716-446655440001"
}
```

**Response:**
```json
{
  "success": true,
  "message": "Crew transferred to target ship",
  "from_ship_persona": { /* updated persona */ },
  "to_ship_persona": { /* updated persona */ }
}
```

**Availability:** Feature flag `crew_profiles`
**Added:** March 2, 2026

---

### Phase 6: Vendor Profiles

#### GET /api/vendors/{uuid}

Get vendor profile and player relationship data.

**Response:**
```json
{
  "success": true,
  "vendor": {
    "uuid": "750e8400-e29b-41d4-a716-446655440000",
    "name": "Kovac's Emporium",
    "poi_name": "Kepler Station",
    "service_type": "trading_hub",
    "criminality": 0.15,
    "personality": {
      "honesty": 0.75,
      "greed": 0.45,
      "charm": 0.80,
      "risk_tolerance": 0.60
    }
  },
  "relationship": {
    "goodwill": 25,
    "shady_dealings": 0,
    "visit_count": 8,
    "effective_markup": 0.04,
    "last_interaction": "2026-03-02T14:30:00Z"
  },
  "vendor_bonuses": {
    "trading_discount": 0.03
  }
}
```

**Availability:** Feature flag `vendor_profiles`
**Added:** March 2, 2026

---

#### POST /api/vendors/{uuid}/interact

Record player interaction with vendor.

**Body:**
```json
{
  "interaction_type": "trade|browse|negotiate|shady_trade"
}
```

**Response:**
```json
{
  "success": true,
  "message": "Interaction recorded",
  "relationship_updated": {
    "goodwill": 35,
    "visit_count": 9
  },
  "dialogue": "Pleasure doing business with you again..."
}
```

**Availability:** Feature flag `vendor_profiles`
**Added:** March 2, 2026

---

### Phase 7: Customs System

#### POST /api/travel/{destination}/customs/check

Perform customs check on arrival (called automatically).

**Response:**
```json
{
  "success": true,
  "outcome": "cleared|fined|cargo_seized|bribed|impounded",
  "message": "Cargo scan complete. You're cleared to dock.",
  "fine_amount": 0,
  "seized_items": [],
  "can_bribe": false,
  "bribe_amount": null
}
```

**Outcomes:**
- `cleared` - No issues found
- `fined` - Minor penalty applied
- `cargo_seized` - Illegal goods confiscated
- `bribed` - Successfully paid off official
- `impounded` - Ship seized for serious violations

**Availability:** Feature flag `customs`
**Added:** March 2, 2026

---

#### POST /api/travel/{destination}/customs/bribe

Attempt to bribe customs official.

**Body:**
```json
{
  "bribe_amount": 5000
}
```

**Response:**
```json
{
  "success": true,
  "outcome": "bribed",
  "message": "Official accepts your generous gift...",
  "credits_spent": 5000,
  "ship_reputation_change": -10
}
```

**Errors:**
- 400 - Insufficient credits
- 403 - Official cannot be bribed (honest)
- 409 - No illegal cargo detected

**Availability:** Feature flag `customs`
**Added:** March 2, 2026

---

#### POST /api/travel/{destination}/customs/accept

Accept fine/seizure outcome.

**Body:**
```json
{
  "outcome": "fined|cargo_seized"
}
```

**Response:**
```json
{
  "success": true,
  "message": "Fine accepted. You're cleared to dock.",
  "credits_deducted": 2500,
  "cargo_removed": ["Spice x50", "Exotic Compounds x30"],
  "remaining_balance": 47500
}
```

**Availability:** Feature flag `customs`
**Added:** March 2, 2026

---

## Modified Endpoints

### GET /api/my-ship

**NEW RESPONSE FIELDS** (Added March 2, 2026):

```json
{
  "ship": {
    // ... existing fields ...
    "crew": [
      {
        "uuid": "550e8400-e29b-41d4-a716-446655440000",
        "name": "Lt. Kovacs",
        "role": "tactical_officer",
        "alignment": "neutral",
        "traits": {
          "negotiation": 0.75,
          "intimidation": 0.68,
          "charm": 0.55
        },
        "vendor_bonuses": {
          "combat_gear_discount": 0.05,
          "negotiation_power": 0.10
        }
      }
    ],
    "ship_persona": {
      "overall_alignment": "neutral",
      "shady_score": 15,
      "total_shady_interactions": 15,
      "vendor_bonuses": {
        "trading_discount": 0.03,
        "repair_discount": 0.02
      },
      // NOTE: black_market_visible only included if true
      "black_market_visible": true
    }
  }
}
```

**Feature Flag:** `crew_profiles` (hides crew and ship_persona if false)
**Date Changed:** March 2, 2026

---

### GET /api/hubs/{uuid}/inventory

**MODIFIED BEHAVIOR** (March 2, 2026):

**Changes:**
1. Items filtered by `CommodityAccessService::filterForPlayer()`
2. Black market items silently excluded if below threshold
3. Industrial items gated by reputation
4. Civilian items always visible

**NEW RESPONSE STRUCTURE:**
```json
{
  "inventory": [
    {
      "uuid": "850e8400-e29b-41d4-a716-446655440000",
      "mineral": {
        "name": "Iron Ore",
        "symbol": "FE",
        "base_value": 100,
        "category": "civilian",  // NEW
        "rarity": "common"
      },
      "quantity": 5000,
      "buy_price": 92,
      "sell_price": 108
    }
  ],
  "filters_applied": {
    "hidden_civilian": 0,
    "hidden_industrial": 2,
    "hidden_black_market": 5
  }
}
```

**Feature Flag:** `black_market`, `crew_profiles` (affects filtering logic)
**Date Changed:** March 2, 2026

---

### POST /api/hubs/{uuid}/buy-mineral

**MODIFIED BEHAVIOR** (March 2, 2026):

**Changes:**
1. Applies vendor effective markup
2. Uses `HubInventoryMutationService` for atomic updates
3. Records vendor interaction
4. Returns updated ship persona

**NEW RESPONSE FIELDS:**
```json
{
  "success": true,
  "trade": {
    // ... existing fields ...
    "vendor_markup_applied": 0.04,
    "final_price": 1040
  },
  "vendor_relationship_updated": {
    "goodwill": 35,
    "visit_count": 9
  },
  "ship_persona": {
    "overall_alignment": "neutral",
    "shady_score": 15
  }
}
```

**Date Changed:** March 2, 2026

---

### POST /api/hubs/{uuid}/sell-mineral

**MODIFIED BEHAVIOR** (March 2, 2026):

Same changes as buy-mineral endpoint.

**Date Changed:** March 2, 2026

---

## Data Models

### CrewMember (NEW)

```sql
CREATE TABLE crew_members (
    id BIGINT PRIMARY KEY,
    uuid VARCHAR(36) UNIQUE,
    galaxy_id BIGINT FK,
    name VARCHAR(100),
    role ENUM('science_officer', 'tactical_officer', ...),
    alignment ENUM('lawful', 'neutral', 'shady'),
    player_ship_id BIGINT FK NULLABLE,
    current_poi_id BIGINT FK,
    shady_actions INT DEFAULT 0,
    reputation INT DEFAULT 0,
    traits JSON,
    backstory VARCHAR(500),
    created_at TIMESTAMP,
    updated_at TIMESTAMP
);
```

**Added:** March 2, 2026

---

### VendorProfile (MODIFIED)

```sql
CREATE TABLE vendor_profiles (
    id BIGINT PRIMARY KEY,
    uuid VARCHAR(36) UNIQUE,
    galaxy_id BIGINT FK,
    poi_id BIGINT FK UNIQUE,
    trading_post_id BIGINT FK,
    service_type VARCHAR(50),
    criminality DECIMAL(5,4),
    personality JSON,
    dialogue_pool JSON,
    markup_base DECIMAL(5,4),
    created_at TIMESTAMP,
    updated_at TIMESTAMP
);
```

**Changed:** Redesigned to use template pattern with TradingPost FK
**Date:** March 2, 2026

---

### TradingPost (NEW)

```sql
CREATE TABLE trading_posts (
    id BIGINT PRIMARY KEY,
    uuid VARCHAR(36) UNIQUE,
    name VARCHAR(100),
    service_type VARCHAR(50),
    base_criminality DECIMAL(5,4),
    personality JSON,
    dialogue_pool JSON,
    markup_base DECIMAL(5,4),
    created_at TIMESTAMP
);
```

**Global templates:** 32 records (12 trading hubs, 8 salvage yards, 8 shipyards, 8 markets)
**Added:** March 2, 2026

---

### CustomsOfficial (NEW)

```sql
CREATE TABLE customs_officials (
    id BIGINT PRIMARY KEY,
    uuid VARCHAR(36) UNIQUE,
    poi_id BIGINT FK UNIQUE,
    name VARCHAR(100),
    honesty DECIMAL(3,2),
    severity DECIMAL(3,2),
    bribe_threshold INT,
    detection_skill DECIMAL(3,2),
    created_at TIMESTAMP
);
```

**One per inhabited POI**
**Added:** March 2, 2026

---

### PlayerVendorRelationship (NEW)

```sql
CREATE TABLE player_vendor_relationships (
    id BIGINT PRIMARY KEY,
    player_id BIGINT FK,
    vendor_profile_id BIGINT FK,
    goodwill INT DEFAULT 0,
    shady_dealings INT DEFAULT 0,
    visit_count INT DEFAULT 0,
    markup_modifier DECIMAL(5,4),
    is_locked_out BOOLEAN DEFAULT false,
    last_interaction_at TIMESTAMP,
    UNIQUE(player_id, vendor_profile_id)
);
```

**Added:** March 2, 2026

---

### Mineral (MODIFIED)

```sql
-- NEW COLUMNS
ALTER TABLE minerals ADD COLUMN category VARCHAR(50) DEFAULT 'civilian';
ALTER TABLE minerals ADD COLUMN is_illegal BOOLEAN DEFAULT false;
ALTER TABLE minerals ADD COLUMN min_reputation INT NULLABLE;
ALTER TABLE minerals ADD COLUMN min_sector_security INT NULLABLE;
```

**Enums Added:**
```php
CommodityCategory::CIVILIAN    // Always visible
CommodityCategory::INDUSTRIAL  // Reputation-gated
CommodityCategory::BLACK       // Threshold + multi-gate
```

**Date Changed:** March 2, 2026

---

## Error Handling

### New Error Codes

| Code | Scenario | Phase |
|------|----------|-------|
| 400 | Black market items below visibility threshold | 8 |
| 400 | Industrial items below reputation threshold | 2 |
| 403 | Insufficient funds for customs bribe | 7 |
| 404 | Crew member not found | 5 |
| 409 | Ship impounded by customs | 7 |
| 409 | Vendor locked out (too many shady dealings) | 6 |
| 410 | Illegal cargo detected and seized | 7 |

### Silent Failures (No Error Returned)

**Black Market Items (Phase 8):**
- Items below visibility threshold simply don't appear in inventory
- No 403 error or "forbidden" message
- Item count `total_items` decreases silently

**Example:**
```json
{
  "inventory": [
    { "mineral": "Iron Ore", ... },
    { "mineral": "Copper", ... }
    // Spice (black market) is NOT included, no error
  ],
  "hidden_by_access": {
    "black_market": 5,  // Shows count in metadata
    "industrial": 2
  }
}
```

---

## Feature Flags

### Rollout Control (config/features.php)

```php
'crew_profiles' => true,        // Phase 5 - Enable crew system
'vendor_profiles' => true,      // Phase 6 - Enable vendor system
'customs' => true,              // Phase 7 - Enable customs checks
'black_market' => true,         // Phase 8 - Enable black market
'npc_traders' => false,         // Phase 9 - Disabled by default
```

### Route Gating Pattern

```php
if (config('features.crew_profiles')) {
    Route::apiResource('crew', CrewController::class);
}
```

### Response Hygiene Pattern

```php
if (!config('features.crew_profiles')) {
    unset($response['crew']);
    unset($response['ship_persona']);
}
```

---

## Migration Guide

### For Existing Clients

**Step 1:** Update to handle new response fields
```php
// Old code
$crew = null;

// New code
$crew = $response['crew'] ?? [];
```

**Step 2:** Handle commodity category filtering
```php
// Old code
$inventory = $response['inventory'];

// New code
$inventory = $response['inventory'];  // Already filtered by server
$hiddenCount = $response['hidden_by_access']['black_market'] ?? 0;
```

**Step 3:** Implement crew hiring workflow
```javascript
// New workflow
const available = await fetch(`/api/galaxies/${galaxy_uuid}/crew/available?poi_uuid=${poi_uuid}`);
const crewList = available.json().available_crew;
// User selects crew
await fetch(`/api/ships/${ship_uuid}/crew/hire`, {
  method: 'POST',
  body: JSON.stringify({ crew_uuid: selectedCrew.uuid })
});
```

**Step 4:** Handle customs checks
```javascript
// New workflow
const result = await travel(destination);
// result.outcome in ['cleared', 'fined', 'cargo_seized', 'bribed', 'impounded']
if (result.outcome === 'fined') {
  await fetch(`/api/travel/${destination}/customs/accept`, {
    method: 'POST',
    body: JSON.stringify({ outcome: 'fined' })
  });
}
```

---

## Version History

| Version | Date | Changes |
|---------|------|---------|
| 2.0 | March 3, 2026 | Phases 5-9 complete |
| 1.9 | March 2, 2026 | Development branch |
| 1.8 | Feb 15, 2026 | Previous version |

---

## Testing Endpoints

### Quick Test Commands

```bash
# Get available crew
curl "http://localhost/api/galaxies/{uuid}/crew/available?poi_uuid={poi_uuid}" \
  -H "Authorization: Bearer {token}"

# Hire crew
curl -X POST "http://localhost/api/ships/{uuid}/crew/hire" \
  -H "Authorization: Bearer {token}" \
  -H "Content-Type: application/json" \
  -d '{"crew_uuid": "550e8400-e29b-41d4-a716-446655440000"}'

# Get vendor profile
curl "http://localhost/api/vendors/{uuid}" \
  -H "Authorization: Bearer {token}"

# Record vendor interaction
curl -X POST "http://localhost/api/vendors/{uuid}/interact" \
  -H "Authorization: Bearer {token}" \
  -H "Content-Type: application/json" \
  -d '{"interaction_type": "trade"}'
```

---

**Next Review:** Q2 2026
**Maintainer:** Architecture Team
**Status:** Production
