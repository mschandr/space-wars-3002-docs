# Template + Instance Architecture

**Date:** March 3, 2026
**Version:** 1.0
**Status:** Production Ready

## Overview

Phase 5-9 uses a **template + instance** pattern for crew, vendors, and customs officials. This allows the same permanent pool of characters to be reused across all galaxies while tracking galaxy-specific state changes.

## Architecture

### Permanent Global Templates (Never Flushed)

These tables contain reusable templates that persist across all galaxies:

| Table | Count | Purpose |
|-------|-------|---------|
| `trading_posts` | 36 | Vendor personality templates (12 trading hubs, 8 salvage yards, 8 shipyards, 8 markets) |
| `crew_members` | 626 | Crew pool with roles, alignment, traits, backstories |
| `vendor_profiles` | 73 | Vendor instances linked to trading post templates (1 per trading hub) |
| `customs_officials` | 401 | Customs officer templates (1 per inhabited POI) |

**Characteristics:**
- No `galaxy_id` field (they exist globally)
- Contain static personality/trait data
- Never deleted by `galaxy:flush`
- Reusable across all galaxies

### Galaxy-Specific State (Flushed with Galaxy)

These tables track how templates evolve per galaxy:

| Table | Purpose | Flushed | Relationships |
|-------|---------|---------|---------------|
| `crew_assignments` | Which crew stationed at which trading hub | ✓ Yes | crew_member_id → crew_members (permanent) |
| `galaxy_vendor_states` | Vendor markup/satisfaction changes per galaxy | ✓ Yes | vendor_profile_id → vendor_profiles (permanent) |
| `galaxy_customs_records` | Officer corruption/history per galaxy | ✓ Yes | customs_official_id → customs_officials (permanent) |

**Characteristics:**
- Have `galaxy_id` field
- Track mutable state (interactions, reputation, corruption)
- Deleted when `galaxy:flush` runs
- Reset fresh for new galaxy instances

## Example: How Crew Works

### Setup
```
Galaxy A (new game starts)
  └─ CrewAssignment.create(galaxy_id=A, crew_member_id=100, trading_hub_id=5)
     └─ Crew member #100 (template) stationed at trading hub #5
     └─ Player can hire this crew at that hub

Galaxy B (another game)
  └─ CrewAssignment.create(galaxy_id=B, crew_member_id=100, trading_hub_id=12)
     └─ Same crew member #100, different hub in different galaxy
     └─ Crew history (shady_actions, reputation) is unchanged
```

### When Game Runs
```
Player hires Crew #100 in Galaxy A
  → Ship.crew_members includes this crew
  → Services calculate alignment, vendor bonuses
  → crew_members.shady_actions increases (permanent)

Later: Player starts Galaxy B
  → Crew #100 still exists with same shady_actions
  → CrewAssignment reflects new hub assignment
  → Fresh game state, clean slate for vendor relationships
```

## Example: How Vendors Work

### Setup
```
Vendor Profile #1 (permanent template)
  - name: "Kovac's Emporium"
  - markup_base: 0.05
  - archetype: hard_bargainer

Galaxy A
  └─ GalaxyVendorState (galaxy_id=A, vendor_profile_id=1)
     - markup_modifier: 0 (fresh start)
     - interaction_count: 0

Galaxy B
  └─ GalaxyVendorState (galaxy_id=B, vendor_profile_id=1)
     - markup_modifier: 0 (fresh start, independent)
```

### When Trading Happens
```
Player trades with Vendor #1 in Galaxy A
  → GalaxyVendorState.interaction_count increases
  → GalaxyVendorState.markup_modifier changes (affects pricing)
  → Vendor_profiles (template) unchanged

Player switches to Galaxy B
  → Vendor #1 still exists
  → GalaxyVendorState in Galaxy B is fresh (0 interactions)
  → No carryover of reputation changes
```

## Example: How Customs Works

### Setup
```
Customs Official (permanent template)
  - name: "Captain Volkov"
  - honesty: 0.85
  - severity: 0.70

Galaxy A
  └─ GalaxyCustomsRecord (galaxy_id=A, customs_official_id=1)
     - total_checks: 0
     - times_bribed: 0
     - actual_honesty: null (uses template's 0.85)

Galaxy B
  └─ GalaxyCustomsRecord (galaxy_id=B, customs_official_id=1)
     - total_checks: 0
     - times_bribed: 0
     - actual_honesty: null (fresh)
```

### When Bribing Happens
```
Player bribes Volkov in Galaxy A
  → GalaxyCustomsRecord(A).times_bribed++
  → GalaxyCustomsRecord(A).actual_honesty -= 0.05 (now 0.80)
  → CustomsOfficial template unchanged

Player travels to Galaxy B
  → Volkov still exists with original honesty=0.85
  → GalaxyCustomsRecord(B).actual_honesty still null (uses 0.85)
  → No carryover, officer is not corrupted in new galaxy
```

## Benefits

| Benefit | Why It Matters |
|---------|----------------|
| **Reusability** | Same characters across unlimited galaxies, no duplication |
| **Immutability** | Template data never changes, consistency guaranteed |
| **Independence** | Galaxies don't affect each other's character states |
| **Clean Slates** | `galaxy:flush` resets player history without losing templates |
| **Efficiency** | Single pool of 600+ crew instead of 600+ per galaxy |

## Seeding

### Command
```bash
php artisan seed:test-data
```

### Seeding Order
1. **TradingPostSeeder** - 36 global templates (once only)
2. **CrewMemberSeeder** - 626 crew members (once only)
3. **VendorProfileSeeder** - 73 vendor instances (once only)
4. **CustomsOfficialSeeder** - 401 officials (once only)
5. **CrewAssignmentSeeder** - Assigns crew to active trading hubs per galaxy
6. **GalaxyVendorStateSeeder** - Creates state records per vendor per galaxy
7. **GalaxyCustomsRecordSeeder** - Creates record entries per official per galaxy

### Output
```
Permanent Global Templates:
  ✓ Trading post templates: 36
  ✓ Crew members (pool): 626
  ✓ Vendor profiles (templates): 73
  ✓ Customs officials (templates): 401

Galaxy-Specific State:
  ✓ Crew assignments: 626+ (varies by active hubs)
  ✓ Vendor states: 73+ per galaxy
  ✓ Customs records: 401+ per galaxy
```

## Flushing Behavior

### `php artisan galaxy:flush --galaxy=<id>`

| Table | Action |
|-------|--------|
| `galaxies` | ✓ Deleted |
| `crew_members` | ✗ **Preserved** (permanent template) |
| `crew_assignments` | ✓ Deleted (galaxy-specific state) |
| `vendor_profiles` | ✗ **Preserved** (permanent template) |
| `galaxy_vendor_states` | ✓ Deleted (galaxy-specific state) |
| `customs_officials` | ✗ **Preserved** (permanent template) |
| `galaxy_customs_records` | ✓ Deleted (galaxy-specific state) |

**Result:** Templates survive, galaxy state resets, next galaxy gets clean slate.

## Database Schema Differences

### Permanent Templates
```sql
-- No galaxy_id field
CREATE TABLE crew_members (
    id PRIMARY KEY,
    galaxy_id FK REFERENCES galaxies,  ← Used for initial POI assignment only
    name, role, alignment, traits, ...
);

-- Reused across galaxies
CREATE TABLE vendor_profiles (
    id PRIMARY KEY,
    galaxy_id FK,  ← Only because initially seeded per galaxy, not for isolation
    trading_post_id FK,
    service_type, criminality, ...
);
```

### Galaxy-Specific State
```sql
-- Explicitly galaxy-scoped
CREATE TABLE crew_assignments (
    id PRIMARY KEY,
    galaxy_id FK REFERENCES galaxies,  ← Always filtered by galaxy
    crew_member_id FK REFERENCES crew_members,  ← Template reference
    trading_hub_id FK,
    PRIMARY KEY (galaxy_id, crew_member_id)
);

CREATE TABLE galaxy_vendor_states (
    id PRIMARY KEY,
    galaxy_id FK REFERENCES galaxies,  ← Always filtered by galaxy
    vendor_profile_id FK,  ← Template reference
    markup_modifier, interaction_count, ...
    PRIMARY KEY (galaxy_id, vendor_profile_id)
);

CREATE TABLE galaxy_customs_records (
    id PRIMARY KEY,
    galaxy_id FK REFERENCES galaxies,  ← Always filtered by galaxy
    customs_official_id FK,  ← Template reference
    total_checks, times_bribed, actual_honesty, ...
    PRIMARY KEY (galaxy_id, customs_official_id)
);
```

## Service Layer Usage

When services query crew, vendors, or customs officials:

```php
// Always filter by galaxy for state queries
$vendorState = GalaxyVendorState::where('galaxy_id', $galaxyId)
    ->where('vendor_profile_id', $vendorId)
    ->first();

// Calculate effective markup combining template + state
$effectiveMarkup = $vendorState->vendorProfile->markup_base
                 + $vendorState->markup_modifier;

// Similar for crew assignments and customs
$crewHere = CrewAssignment::where('galaxy_id', $galaxyId)
    ->where('trading_hub_id', $hubId)
    ->with('crewMember')
    ->get();

$officerRecord = GalaxyCustomsRecord::where('galaxy_id', $galaxyId)
    ->where('customs_official_id', $officialId)
    ->first();
```

## Future Enhancements

- **Cross-galaxy crew transfer** - Allow players to bring favorite crew to new galaxies
- **Template versioning** - If crew/vendor templates change, old galaxies keep old versions
- **Persistent reputation** - Player reputation with a crew could follow across galaxies
- **Template customization** - Allow template modifications per game mode

---

**Related Files:**
- `app/Models/CrewAssignment.php`
- `app/Models/GalaxyVendorState.php`
- `app/Models/GalaxyCustomsRecord.php`
- `database/migrations/2026_03_03_000001_create_crew_assignments_table.php`
- `database/migrations/2026_03_03_000002_create_galaxy_vendor_states_table.php`
- `database/migrations/2026_03_03_000003_create_galaxy_customs_records_table.php`
